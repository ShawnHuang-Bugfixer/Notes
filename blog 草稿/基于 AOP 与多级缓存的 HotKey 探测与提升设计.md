在高并发应用中，部分缓存 key（通常称为 **热点 key**）访问频次远高于其他 key。如果这些热点 key 的数据主要存储在远程缓存（如 Redis），会导致频繁访问 Redis，增加网络和 Redis 的压力，甚至可能引发缓存击穿。为解决这一问题，可以设计 **热点 key 探测与提升机制**，将热点数据动态提升到本地缓存（如 Caffeine），降低远程访问压力。下面介绍一种基于 Spring AOP 与多级缓存的实现方案。

# 设计思路
利用 **AOP 拦截方法**，在方法执行前统计缓存 key 的访问次数，并通过**Spring EL 表达式** 解析方法参数，动态生成真实缓存 key。访问次数统计存储在 **Caffeine 本地缓存**中，设置 **定时任务**，固定周期扫描访问计数器缓存，每个 key 的访问次数超过阈值 `HOT_THRESHOLD` 时，认定为热点 key，对热点 key 执行 **提升操作**，将其从 Redis 拉取到本地缓存。

```Java
@Aspect  
@Component  
@Slf4j  
@Order(Ordered.HIGHEST_PRECEDENCE)  
public class HotKeyAspect {  
  
    @Value("${app.hotkey.detect.threshold}")  
    private long HOT_THRESHOLD;  
  
    @Value("${app.hotkey.detect.maxCapacity}")  
    private int MAX_CAPACITY;  
  
    @Value("${app.hotkey.detect.expireSec}")  
    private long EXPIRE_SECONDS;  
  
    private static final long SCAN_INTERVAL_TIME = 5000;  
  
    @Resource  
    private CacheManager caffeineCacheManager;  
  
    @Resource  
    private RedisTemplate<String, Object> redisTemplate;  
  
    // 用于统计访问次数的本地缓存（代替原来的 ConcurrentHashMap）  
    private Cache<String, Counter> accessCounterCache;  
  
    @PostConstruct  
    public void init() {  
        this.accessCounterCache = Caffeine.newBuilder()  
                .expireAfterAccess(EXPIRE_SECONDS, TimeUnit.SECONDS)  
                .maximumSize(MAX_CAPACITY)  
                .build();  
    }  
  
    @Scheduled(fixedRate = SCAN_INTERVAL_TIME)  
    public void detectHotKeys() {  
        accessCounterCache.asMap().forEach((key, counter) -> {  
            long count = counter.getAdder().sumThenReset();  
            if (count > HOT_THRESHOLD) {  
                log.debug("hotkey appear! key :{}, count: {} in 5 seconds", key, count);  
                promoteToCaffeine(counter.getCacheNames(), key);  
            }  
        });  
    }  
  
    @Async("cacheWarmupExecutor")  
    public void promoteToCaffeine(String[] cacheNames, String key) {  
        for (String cacheName : cacheNames) {  
            log.debug("put hot key: {} into caffeine: {}", key, cacheName);  
            org.springframework.cache.Cache cache = caffeineCacheManager.getCache(cacheName);  
            if (cache != null) {  
                org.springframework.cache.Cache.ValueWrapper valueWrapper = cache.get(key);  
                if (valueWrapper == null) {  
                    Object value = redisTemplate.opsForValue().get(key);  
                    if (value != null) cache.put(key, value);  
                }  
            }  
        }  
    }  
  
    @Around("@annotation(org.springframework.cache.annotation.Cacheable)")  
    public Object countAccess(ProceedingJoinPoint joinPoint) throws Throwable {  
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();  
        Method method = signature.getMethod();  
        Cacheable cacheable = method.getAnnotation(Cacheable.class);  
        String cacheManagerName = cacheable.cacheManager();  
  
        // 仅当使用自定义多级缓存时，才统计 key 的频率。  
        if (!"multiLevelCacheManger".equals(cacheManagerName)) {  
            return joinPoint.proceed();  
        }  
  
        // 要求 Spring Cache 的缓存名称 value 必须包含 hot 关键字。  
        String[] cacheNames = cacheable.value();  
        for (String cacheName : cacheNames) {  
            if (!cacheName.toLowerCase().contains("hot")) {  
                return joinPoint.proceed();  
            }  
        }  
  
        // 利用 Spring EPL 拼接真实的缓存 key (Redis 和 Caffeine 中保持一致)  
        // 此项目中对图片详情进行缓存，利用图片 id 构造缓存 key。  
        String keyExpression = cacheable.key();  
        SpelExpressionParser parser = new SpelExpressionParser();  
        Expression expression = parser.parseExpression(keyExpression);  
        StandardEvaluationContext context = new StandardEvaluationContext();  
        String[] parameterNames = signature.getParameterNames();  
  
        for (int i = 0; i < parameterNames.length; i++) {  
            if (parameterNames[i].toLowerCase().contains("id")) {  
                context.setVariable(parameterNames[i], joinPoint.getArgs()[i]);  
                break;  
            }  
        }  
  
        String realKey = expression.getValue(context, String.class);  
  
        // 使用 caffeine 缓存统计访问  
        accessCounterCache.asMap().compute(realKey, (k, existing) -> {  
            if (existing == null) {  
                Counter newCounter = new Counter(cacheNames);  
                newCounter.increment();  
                return newCounter;  
            } else {  
                existing.increment();  
                return existing;  
            }  
        });  
  
        return joinPoint.proceed();  
    }  
  
    @Getter  
    static class Counter {  
        private final LongAdder adder = new LongAdder();  
        private final String[] cacheNames;  
  
        public Counter(String[] cacheNames) {  
            this.cacheNames = cacheNames;  
        }  
  
        public void increment() {  
            adder.increment();  
        }  
    }  
}
```

## 累加器设计
由于热点 key 会被高并发访问，因此在累加 key 访问次数时，需要考虑到累加的线程安全性，故使用 LongAdder 来统计 key 的访问次数。

```Java
@Getter  
static class Counter {  
    private final LongAdder adder = new LongAdder();  
    private final String[] cacheNames;  
  
    public Counter(String[] cacheNames) {  
        this.cacheNames = cacheNames;  
    }  
  
    public void increment() {  
        adder.increment();  
    }  
}
```

## 定时任务设计
使用定时任务定期全量扫描 key 和 访问次数映射，当某键值的访问次数超过一定阈值后，将该 key 从 Redis 缓存拉取到本地 Caffeine 缓存中。

```Java
@Scheduled(fixedRate = SCAN_INTERVAL_TIME)
public void detectHotKeys() {
    accessCounterCache.asMap().forEach((key, counter) -> {
        long count = counter.getAdder().sumThenReset();
        if (count > HOT_THRESHOLD) {
            promoteToCaffeine(counter.getCacheNames(), key);
        }
    });
}

```

# 优点与缺点
1. 优点
	- 上述设计逻辑清晰，流程简单，在小规模情况下可以胜任热点 key 的探测与提升工作，并且利用原子累加器 LongAdder 支持多线程快速累加，
2. 缺点
	- 定时全量扫描，key 数量过多时可能影响性能，使用 LRU 或 top-N 策略并缩小 key 访问次数映射规模，只统计访问量高的 key。
	- 访问统计粒度有问题，尤其在高并发短周期访问下无法准确识别热点 Key，应该使用滑动窗口技术，提高热点识别准确性。
