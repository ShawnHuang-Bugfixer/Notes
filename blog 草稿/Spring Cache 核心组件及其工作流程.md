# Spring Cache 工作流程 
Spring Cache 旨在将缓存逻辑从业务逻辑中剥离，实现无侵入的缓存集成。其核心接口包含 CacheManager 缓存管理器和 Cache 缓存抽象。可以提前通过提前配置 CacheManager 和 Cache 并结合注解将缓存逻辑集成到业务中。

Spring Cache 的工作流程可以大致概括为：
1. 解析注解信息
2. 根据注解查找对应缓存管理器实例
3. 在该缓存管理器示例中找对应指定缓存实例
4. 根据注解类型调用该缓存实例中实现的具体方法。

## CacheManager 缓存管理器接口
缓存管理器接口中包含两个方法，getCache(String name)；通过缓存名称获取缓存以及通过获取所有缓存名称。参考 RedisCachemanager，缓存管理器中一般包含了名称和缓存的映射。

```Java
public interface CacheManager {  
    @Nullable  
    Cache getCache(String name);  
  
    Collection<String> getCacheNames();  
}
```

![[attachments/Pasted image 20250901130856.png]]

## Cache 缓存抽象
缓存抽象接口中定义了与注解功能相关的接口方法，使用特定注解后由系统自动调用。

```Java
public interface Cache { 
	/**  
	 * 获取缓存名称  
	 * 调用时机：缓存管理、日志打印等需要标识缓存实例时  
	 */ 
    String getName();  
  
	/**  
	* 返回底层原生缓存（如Caffeine或Redis的对象）,这里只返回 CaffeineCache  
	* 调用时机：需要直接操作底层缓存的高级功能时  
	*/
    Object getNativeCache();  
  
	/**  
	* 实现缓存查询逻辑  
	*/
    @Nullable  
    ValueWrapper get(Object key);  
  
	/**    
	* 调用时机：@Cacheable注解中明确指定返回值类型时  
	*/
    @Nullable  
    <T> T get(Object key, @Nullable Class<T> type);  
  
	/**  
	* 缓存加载逻辑 
	* 调用时机：@Cacheable(sync=true)启用同步加载时, valueLoacder 
	*/
    @Nullable  
    <T> T get(Object key, Callable<T> valueLoader);  
  
	/**  
	* 写入缓存  
	* 调用时机：@CachePut注解方法执行后或显式更新缓存时  
	*/
    void put(Object key, @Nullable Object value);  
  
	/**  
	* 删除指定键的缓存  
	* 调用时机：@CacheEvict注解触发或显式删除时  
	*/
    void evict(Object key);  
  
	/**  
	* 清空所有缓存  
	* 调用时机：@CacheEvict(allEntries=true)时  
	*/
    void clear();   
}
```

![[attachments/Pasted image 20250901131858.png]]
## Spring Cache 注解
利用注解可以指定缓存管理器、缓存名称和缓存键等其他内容。以下述 @Cacheable 为例，该注解指定了缓存管理器、缓存名称、缓存键以及调用的具体缓存接口方法。

```Java
@Cacheable(cacheManager = "multiLevelCacheManger", value = "pictureHotKey", key = "'picture:pictureVO:' + #id", sync = true)
```

![[attachments/@Cacheable.png]]
