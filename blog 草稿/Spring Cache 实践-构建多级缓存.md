Spring Cache 核心工作流程本文不再梳理。主要梳理如何基于 Spring Cache 搭建多级缓存。
>[Spring Cache 核心工作原理]([Spring Cache 核心组件及其工作流程 - ShawnHuang-Bugfixer](http://blog.afishingcat.xin/index.php/2025/08/04/58.html))

# 多级缓存设计
先设计多级缓存，以热门图片为例：将热门图片的相关信息缓存在本地（Caffeine）与远程（Redis）两层。查询图片信息时，优先存储到 Redis，当某张图片访问频率达到设定阈值后，再将其缓存数据提升到 Caffeine，以获得更低延迟的访问速度。

有了这个基础思路，现在来看如何利用 Spring Cache 实现上述逻辑。Spring Cache 并未提供多级缓存管理器的具体实现，更没有多级缓存 Cache，因此需要手动实现。但是 SpringCache 已经包含了 CaffeineCacheManager 和 RedisCacheManager 两个现成的缓存管理器，可以通过这两个缓存管理器实现自己的 MultiCacheManager。在 MultiCacheManager 中包含一个缓存名和自定义多级缓存 MultiCache 的 Map 映射。

在 MultiCache 中整合 Caffeine 和 Redis，实现多级缓存相关构建和查询逻辑。

# MultiCache 多级缓存
在多级缓存中整合 CaffeineCache 和 RedisCache，实现自定义多级缓存逻辑。

```Java
/**  
 * 实现 Cache 接口，封装 CaffeineCache 和 RedisCache 以实现热点 key 多重缓存逻辑。  
 */  
@Slf4j  
public class MultiLevelCache implements Cache {  
    private final String name;  
    private final CaffeineCache caffeineCache;  
    private final RedisCache redisCache;  
    private final RedissonClient redissonClient;  
    private static final long lockWaitTime = 3;                 // 锁等待时间（秒）  
    private static final long lockLeaseTime = 10;               // 锁自动释放时间（秒）  
  
    public MultiLevelCache(String cacheName, CaffeineCache caffeineCache, RedisCache redisCache, RedissonClient redissonClient) {  
        this.name = cacheName;  
        this.caffeineCache = caffeineCache;  
        this.redisCache = redisCache;  
        this.redissonClient = redissonClient;  
    }  
  
    /**  
     * 实现 hotkey 多级缓存查询逻辑  
     *  
     * @param key 缓存键  
     * @return 返回 缓存值  
     */  
    @Override  
    public ValueWrapper get(@NonNull Object key) {  
        // 1. 查询Caffeine  
        ValueWrapper value = caffeineCache.get(key);  
        if (value != null) {  
            log.debug("get key:{} from caffeine", key);  
            return value;  
        }  
  
        // 2. 查询Redis  
        ValueWrapper redisValue = redisCache.get(key);  
        if (redisValue != null) {  
            log.debug("get key:{} from redis", key);  
            return redisValue;  
        }  
        return null;  
    }  
  
    /**  
     * 获取缓存名称  
     * 调用时机：缓存管理、日志打印等需要标识缓存实例时  
     */  
    @Override  
    public String getName() {  
        return this.name;  
    }  
  
    /**  
     * 返回底层原生缓存（如Caffeine或Redis的对象）,这里只返回 CaffeineCache  
     * 调用时机：需要直接操作底层缓存的高级功能时  
     */  
    @Override  
    public Object getNativeCache() {  
        return caffeineCache.getNativeCache();  
    }  
  
    /**  
     * 多级缓存查询（带类型检查）  
     * 调用时机：@Cacheable注解中明确指定返回值类型时  
     */  
    @Override  
    public <T> T get(@NonNull Object key, Class<T> type) {  
        // 1. 查询Caffeine  
        T caffeineValue = caffeineCache.get(key, type);  
        if (caffeineValue != null) {  
            log.debug("get data from caffeine, key : {}", key);  
            return caffeineValue;  
        }  
  
        // 2. 查询Redis并检查类型  
        ValueWrapper wrapper = redisCache.get(key);  
        if (wrapper != null && type.isInstance(wrapper.get())) {  
            //            asyncExecutor.execute(() -> caffeineCache.put(key, redisValue)); // 异步回填  
            return type.cast(wrapper.get());  
        }  
        return null;  
    }  
  
    /**  
     * 缓存加载逻辑（防止缓存穿透）  
     * 调用时机：@Cacheable(sync=true)启用同步加载时  
     */  
    @Override  
    public <T> T get(@NonNull Object key, @NonNull Callable<T> valueLoader) {  
        // 1. 先尝试从本地缓存获取  
        ValueWrapper value =  caffeineCache.get(key);  
        if (value != null) {  
            log.debug("get data from caffeine, key : {}", key);  
            return (T)value.get();  
        }  
  
        // 2. 尝试从 Redis 缓存获取  
        value = redisCache.get(key);  
        if (value != null) {  
            return (T)value.get();  
        }  
  
        // 3. 获取分布式锁（按 Key 加锁）  
        RLock lock = redissonClient.getLock("lock:" + key.toString());  
        try {  
            // 3.1 尝试加锁（等待 lockWaitTime 秒，锁持有时间 lockLeaseTime 秒）  
            if (lock.tryLock(lockWaitTime, lockLeaseTime, TimeUnit.SECONDS)) {  
                try {  
                    // 3.2 双重检查 Redis 缓存（防止其他线程已加载）  
                    value = redisCache.get(key);  
                    if (value != null) {  
                        return (T)value.get();  
                    }  
                    log.debug("rebuild redis cache----------------------------------------");  
                    // 3.3 执行加载逻辑（如数据库查询）  
                    T data = valueLoader.call();  
  
                    // 3.4 更新 Redis 和本地缓存  
                    redisCache.put(key, data);  
                    return data;  
                } finally {  
                    lock.unlock();  
                }  
            } else {  
                // 3.5 锁获取失败，等待并重试从 Redis 获取  
                while (true) {  
                    Thread.sleep(50); // 避免 CPU 忙等  
                    value = redisCache.get(key);  
                    if (value != null) {  
                        return (T)value.get();  
                    }  
                }  
            }  
        } catch (InterruptedException e) {  
            Thread.currentThread().interrupt();  
            throw new ValueRetrievalException(key, valueLoader, e);  
        } catch (Exception e) {  
            throw new ValueRetrievalException(key, valueLoader, e);  
        }  
    }  
  
    /**  
     * 写入多级缓存  
     * 调用时机：@CachePut注解方法执行后或显式更新缓存时  
     */  
    @Override  
    public void put(@NonNull Object key, Object value) {  
        redisCache.put(key, value);           // 同步写Redis保证持久化  
    }  
  
    /**  
     * 删除指定键的缓存  
     * 调用时机：@CacheEvict注解触发或显式删除时  
     */  
    @Override  
    public void evict(@NonNull Object key) {  
        caffeineCache.evict(key);  // 立即移除本地缓存  
        redisCache.evict(key);     // 同步移除Redis缓存  
    }  
  
    /**  
     * 清空所有缓存  
     * 调用时机：@CacheEvict(allEntries=true)时  
     */  
    @Override  
    public void clear() {  
        caffeineCache.clear();  // 清空本地缓存  
        redisCache.clear();     // 清空Redis缓存  
    }  
}
```

# MultiCacheManager
多级缓存管理器用于整合 CaffeineCacheManager 和 RedisCacheManager，并维护缓存名和缓存实例的映射。

```Java
/**  
 * 自定义多级缓存管理。构建 value = ‘customCache’ 和自定义多级缓存 MultiLevelCache 的一对一映射。  
 * <p>  
 * 接收 CaffeineCacheManger 和 RedisCacheManger，从两个 cacheManager 中获取统一的配置信息。  
 * 为了避免动态配置 value 值导致使用默认配置对 cache 进行配置，建议 hotkey 先在 RedisCacheManger  
 * 中配置缓存信息。  
 */  
public class MultiLevelCacheManager implements CacheManager {  
    private final CaffeineCacheManager caffeineCacheManager;  
    private final RedisCacheManager redisCacheManager;  
    private final RedissonClient redissonClient;  
    private final ConcurrentMap<String, MultiLevelCache> caches = new ConcurrentHashMap<>();  
  
    public MultiLevelCacheManager(CaffeineCacheManager caffeineCacheManager,  
                                  RedisCacheManager redisCacheManager, RedissonClient redissonClient) {  
        this.caffeineCacheManager = caffeineCacheManager;  
        this.redisCacheManager = redisCacheManager;  
        this.redissonClient = redissonClient;  
    }  
  
    @Override  
    public Cache getCache(@NonNull String name) {  
        return caches.computeIfAbsent(name, cacheName ->  
                new MultiLevelCache(  
                        cacheName,  
                        (CaffeineCache) caffeineCacheManager.getCache(cacheName),  
                        (RedisCache) redisCacheManager.getCache(cacheName),  
                        redissonClient  
                )  
        );  
    }  
  
    @Override  
    @NonNull    public Collection<String> getCacheNames() {  
        return caches.keySet();  
    }  
}
```

# Spring Cache 配置类

在配置类中声明 CacheManager 各种实现类的基础配置。

```Java
/**  
 * 配置不同的 CacheManager 实例  
 *  
 * @author 黄兴鑫  
 * @since 2025/3/13 11:53  
 */@Configuration  
@EnableCaching  
public class CacheConfig {  
    // caffeine 默认配置  
    private static final int CAFFEINE_INT_NUM = 100;  
    private static final int CAFFEINE_MAX_SIZE = 1000;  
    private static final int TTL_SECONDS = 10 * 60;  
  
    // Caffeine 配置 可以通过 yml 配置代替  
    @Bean("caffeineCacheManager")  
    public CacheManager caffeineCacheManager() {  
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();  
        cacheManager.setCaffeine(Caffeine.newBuilder()  
                .initialCapacity(CAFFEINE_INT_NUM)  
                .maximumSize(CAFFEINE_MAX_SIZE)  
                .expireAfterWrite(TTL_SECONDS, TimeUnit.SECONDS)  
                .recordStats());  
        return cacheManager;  
    }  
  
    // Redis 缓存配置  
    @Bean("redisCacheManager")  
    @Primary  
    public CacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory) {  
        return RedisCacheManager.builder(redisConnectionFactory)  
                .cacheDefaults(RedisCacheTypeEnum.DEFAULT.getCacheConfig())  
                .withInitialCacheConfigurations(RedisCacheTypeEnum.getCacheConfigurations())  
                .build();  
    }  
  
    @Bean("multiLevelCacheManger")  
    public CacheManager multiLevelCacheManger(@Qualifier("caffeineCacheManager") CacheManager caffeineCacheManager,  
                                              @Qualifier("redisCacheManager") CacheManager redisCacheManager,  
                                              RedissonClient redissonClient) {  
        return new MultiLevelCacheManager((CaffeineCacheManager) caffeineCacheManager, (RedisCacheManager) redisCacheManager, redissonClient);  
    }  
}
```

**注意**：配置 RedisCacheManager 时使用了配置枚举，在枚举中提前声明了 cacheName 和对应的 RedisCache 相关配置。这样在通过缓存名查询缓存时直接返回提前配置的缓存，如果通过缓存名查询结果为 null，则会按默认配置创建缓存实例。

## RedisCacheManager 配置枚举
```Java
/**  
 * 配置不同缓存命名对应的缓存规则  
 */  
@Getter  
public enum RedisCacheTypeEnum {  
    // 默认缓存，过期时间 1 小时，JSON 序列化，启用前缀，不缓存 null    DEFAULT("defaultCache",  
            Duration.ofHours(1),  
            false,  
            true,  
            RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()),  
            RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer())),  
  
    HOT_PICTURE_KEY("pictureHotKey",  
            Duration.ofMinutes(60),  
            false,  
            true,  
            RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()),  
            RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer())),  
  
    HOT_PICTUREVO_List("pictureHotVOList",  
            Duration.ofMinutes(10), // 600  
            false,  
            true,  
            RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()),  
            RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer())),  
  
    COLD_PICTUREVO_List("pictureColdVOList",  
            Duration.ofMinutes(2), // 120  
            false,  
            true,  
            RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()),  
            RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));  
  
    private final String cacheName;  
    private final RedisCacheConfiguration cacheConfig;  
  
    /**  
     * 枚举构造方法  
     *  
     * @param cacheName       缓存名称  
     * @param ttl             过期时间  
     * @param usePrefix       是否启用 Key 前缀  
     * @param cacheNull       是否缓存 null 值  
     * @param keySerializer   Key 的序列化方式  
     * @param valueSerializer Value 的序列化方式  
     */  
    RedisCacheTypeEnum(String cacheName, Duration ttl, boolean usePrefix, boolean cacheNull,  
                       RedisSerializationContext.SerializationPair<String> keySerializer,  
                       RedisSerializationContext.SerializationPair<?> valueSerializer) {  
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()  
                .entryTtl(ttl)  
                .serializeKeysWith(keySerializer)  
                .serializeValuesWith(valueSerializer);  
  
        if (!usePrefix) {  
            config = config.disableKeyPrefix(); // 禁用 Key 前缀  
        } else {  
            config = config.computePrefixWith(CacheKeyPrefix.simple()); // 默认前缀：cacheName::key  
        }  
  
        if (!cacheNull) {  
            config = config.disableCachingNullValues(); // 不缓存 null        }  
  
        this.cacheName = cacheName;  
        this.cacheConfig = config;  
    }  
  
    /**  
     * 生成 Map<String, RedisCacheConfiguration>，用于 RedisCacheManager  
     */    public static Map<String, RedisCacheConfiguration> getCacheConfigurations() {  
        return Arrays.stream(values())  
                .collect(Collectors.toMap(RedisCacheTypeEnum::getCacheName, RedisCacheTypeEnum::getCacheConfig));  
    }  
  
    /**  
     * 获取所有缓存名称数组  
     */  
    public static String[] getCacheNames() {  
        return Arrays.stream(values())  
                .map(RedisCacheTypeEnum::getCacheName)  
                .toArray(String[]::new);  
    }  
}
```

# 注解调用自定义多级缓存

```Java
@Cacheable(cacheManager = "multiLevelCacheManger", value = "pictureHotKey", key = "'picture:pictureVO:' + #id", sync = true)  
public PictureVO getPictureVOById(long id) {  
    log.info("缓存失效！");  
    if (!pictureBloomFilter.contains(String.valueOf(id))) {  
        return null;  
    }  
    Picture picture = this.getById(id);  
    if (picture == null) {  
        return null;  
    }  
    Long spaceId = picture.getSpaceId();  
    Space dbSpace = spaceService.getById(spaceId);  
    PictureVO pictureVO = getPictureVO(picture);  
    if (dbSpace != null) {  
        if (dbSpace.getSpaceType() == 0) {  
            // 私有空间，仅私有空间拥有者可操作  
            StpUtil.checkPermission(PermissionConstants.PRIVATE_VIEW_IMAGE);  
        } else {  
            StpUtil.checkPermission(PermissionConstants.TEAM_VIEW_IMAGE);  
        }  
        pictureVO.setSpaceType(dbSpace.getSpaceType());  
    }  
    return pictureVO;  
}
```

指定了多级缓存管理器，并指定了缓存名，执行注解声明的方法前，通过 Spring AOP 将自定义的缓存逻辑织入目标方法前后，实现无侵入集成多级缓存。

## 多级缓存在 MultiCache 中的实现

valueLoader 函数接口，调用 valueLoader.call() 实际上就是调用了目标方法并获得返回值。查询时先查 Caffeine 在查 Redis，都无数据则通过目标方法获取返回值。然后通过加锁的方式控制线程写回数据到缓存。

下述逻辑还整合了缓存穿透、缓存击穿的解决方案。

```Java
    /**  
     * 缓存加载逻辑（防止缓存穿透）  
     * 调用时机：@Cacheable(sync=true)启用同步加载时  
     */  
    @Override  
    public <T> T get(@NonNull Object key, @NonNull Callable<T> valueLoader) {  
        // 1. 先尝试从本地缓存获取  
        ValueWrapper value =  caffeineCache.get(key);  
        if (value != null) {  
            log.debug("get data from caffeine, key : {}", key);  
            return (T)value.get();  
        }  
  
        // 2. 尝试从 Redis 缓存获取  
        value = redisCache.get(key);  
        if (value != null) {  
            return (T)value.get();  
        }  
  
        // 3. 获取分布式锁（按 Key 加锁）  
        RLock lock = redissonClient.getLock("lock:" + key.toString());  
        try {  
            // 3.1 尝试加锁（等待 lockWaitTime 秒，锁持有时间 lockLeaseTime 秒）  
            if (lock.tryLock(lockWaitTime, lockLeaseTime, TimeUnit.SECONDS)) {  
                try {  
                    // 3.2 双重检查 Redis 缓存（防止其他线程已加载）  
                    value = redisCache.get(key);  
                    if (value != null) {  
                        return (T)value.get();  
                    }  
                    log.debug("rebuild redis cache----------------------------------------");  
                    // 3.3 执行加载逻辑（如数据库查询）  
                    T data = valueLoader.call();  
  
                    // 3.4 更新 Redis 和本地缓存  
                    redisCache.put(key, data);  
                    return data;  
                } finally {  
                    lock.unlock();  
                }  
            } else {  
                // 3.5 锁获取失败，等待并重试从 Redis 获取  
                while (true) {  
                    Thread.sleep(50); // 避免 CPU 忙等  
                    value = redisCache.get(key);  
                    if (value != null) {  
                        return (T)value.get();  
                    }  
                }  
            }  
        } catch (InterruptedException e) {  
            Thread.currentThread().interrupt();  
            throw new ValueRetrievalException(key, valueLoader, e);  
        } catch (Exception e) {  
            throw new ValueRetrievalException(key, valueLoader, e);  
        }  
    }  
```