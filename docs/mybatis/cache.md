# Mybatis缓存构建过程解析
## CacheBuilder
关于[建造者模式](design-pattern/builder)
```java
/**
  Cache cache = new CacheBuilder(currentNamespace)
          .implementation(valueOrDefault(typeClass, PerpetualCache.class))
          .addDecorator(valueOrDefault(evictionClass, LruCache.class))
          .clearInterval(flushInterval)
          .size(size)
          .readWrite(readWrite)
          .blocking(blocking)
          .properties(props)
          .build();
 */
// 使用建造者模式(简单版) 
public class CacheBuilder {
  // id
  private final String id;
  // 要初始化的缓存类型
  private Class<? extends Cache> implementation;
  // 使用的装饰者数组
  private final List<Class<? extends Cache>> decorators;
  // 缓存长度
  private Integer size;
  // 缓存清理间隔
  private Long clearInterval;
  // 是否只读
  private boolean readWrite;
  // 缓存的属性
  private Properties properties;
  // 是否使用锁
  private boolean blocking;
  // 使用 命名空间名称作为缓存的id
  public CacheBuilder(String id) {
    this.id = id;
    // 初始化装饰者数组
    this.decorators = new ArrayList<>();
  }
  // 构建过程
  public Cache build() {
    // 设置默认的缓存类
    setDefaultImplementations();
    // 构建缓存
    Cache cache = newBaseCacheInstance(implementation, id);
    // 设置缓存的属性
    setCacheProperties(cache);
    // issue #352, do not apply decorators to custom caches
    if (PerpetualCache.class.equals(cache.getClass())) {
      // 如果使用的是默认缓存
      for (Class<? extends Cache> decorator : decorators) {
        // 使用装饰者模式对cache进行装饰(添加新的功能)
        cache = newCacheDecoratorInstance(decorator, cache);
        // 设置缓存属性
        setCacheProperties(cache);
      }
      // 根据配置添加装饰者
      cache = setStandardDecorators(cache);
    } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
      // 自定义缓存,且没有日志功能,添加一个日志功能
      cache = new LoggingCache(cache);
    }
    // 将构建好的缓存返回
    return cache;
  }

  // 设置默认的缓存类型为(如果用户未配置使用PerpetualCache ,设置默认的装饰器使用最近未使用的装饰器)
  private void setDefaultImplementations() {
    if (implementation == null) {
      implementation = PerpetualCache.class;
      if (decorators.isEmpty()) {
        decorators.add(LruCache.class);
      }
    }
  }
  
  // 反射通过构造器创建 初始缓存
  private Cache newBaseCacheInstance(Class<? extends Cache> cacheClass, String id) {
    Constructor<? extends Cache> cacheConstructor = getBaseCacheConstructor(cacheClass);
    try {
      return cacheConstructor.newInstance(id);
    } catch (Exception e) {
      throw new CacheException("Could not instantiate cache implementation (" + cacheClass + "). Cause: " + e, e);
    }
  }
  // 设置标准的装饰器
  private Cache setStandardDecorators(Cache cache) {
    try {
      // 解析cache的元信息
      MetaObject metaCache = SystemMetaObject.forObject(cache);
      // 如果有size 设置size
      if (size != null && metaCache.hasSetter("size")) {
        metaCache.setValue("size", size);
      }
      if (clearInterval != null) {
        // 如果设置了清理间隔(默认不清理缓存)
        // 使用定时缓存装饰(具有定时清理缓存的功能)
        cache = new ScheduledCache(cache);
        ((ScheduledCache) cache).setClearInterval(clearInterval);
      }
      // 默认true
      if (readWrite) {
        // 默认使用的是多线程下安全的可读写返回对象(通过序列化和反序列化保证写入不改变缓存种对象)
        cache = new SerializedCache(cache);
      }
      // 添加log功能
      cache = new LoggingCache(cache);
      // 缓存可在多线程下使用
      cache = new SynchronizedCache(cache);
  
      // 默认false
      if (blocking) {
        // 缓存使用block(默认没有)
        cache = new BlockingCache(cache);
      }
      return cache;
    } catch (Exception e) {
      throw new CacheException("Error building standard cache decorators.  Cause: " + e, e);
    }
  }



}
```


## Cache
```java
// 待实现缓存接口
public interface Cache {

  String getId();

  void putObject(Object key, Object value);
  Object getObject(Object key);

  Object removeObject(Object key);

  void clear();

  int getSize();
  default ReadWriteLock getReadWriteLock() {
    return null;
  }
}

// 基础缓存类 只使用map作为进行缓存
public class PerpetualCache implements Cache {

  private final String id;

  private final Map<Object, Object> cache = new HashMap<>();

  public PerpetualCache(String id) {
    this.id = id;
  }

  @Override
  public String getId() {
    return id;
  }

  @Override
  public int getSize() {
    return cache.size();
  }

  @Override
  public void putObject(Object key, Object value) {
    cache.put(key, value);
  }

  @Override
  public Object getObject(Object key) {
    return cache.get(key);
  }

  @Override
  public Object removeObject(Object key) {
    return cache.remove(key);
  }

  @Override
  public void clear() {
    cache.clear();
  }

  @Override
  public boolean equals(Object o) {
    if (getId() == null) {
      throw new CacheException("Cache instances require an ID.");
    }
    if (this == o) {
      return true;
    }
    if (!(o instanceof Cache)) {
      return false;
    }

    Cache otherCache = (Cache) o;
    return getId().equals(otherCache.getId());
  }

  @Override
  public int hashCode() {
    if (getId() == null) {
      throw new CacheException("Cache instances require an ID.");
    }
    return getId().hashCode();
  }

}

// least recently used
public class LruCache implements Cache {

  // 待装饰的缓存
  private final Cache delegate;
  private Map<Object, Object> keyMap;
  private Object eldestKey;

  // 被注册对象引用; 设置缓存的大小
  public LruCache(Cache delegate) {
    this.delegate = delegate;
    setSize(1024);
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    return delegate.getSize();
  }

  public void setSize(final int size) {
    // 扩展LinkedHashMap的功能,返回true移除最近未使用的
    keyMap = new LinkedHashMap<Object, Object>(size, .75F, true) {
      private static final long serialVersionUID = 4267176411845948333L;

      @Override
      protected boolean removeEldestEntry(Map.Entry<Object, Object> eldest) {
        // 如果容量超出了最大容量,设置被移除的key,并在容器中移除
        boolean tooBig = size() > size;
        if (tooBig) {
          eldestKey = eldest.getKey();
        }
        return tooBig;
      }
    };
  }

  @Override
  public void putObject(Object key, Object value) {
    delegate.putObject(key, value);
    cycleKeyList(key);
  }

  @Override
  public Object getObject(Object key) {
    keyMap.get(key); // touch
    return delegate.getObject(key);
  }

  @Override
  public Object removeObject(Object key) {
    return delegate.removeObject(key);
  }

  @Override
  public void clear() {
    delegate.clear();
    keyMap.clear();
  }
  
  // put后判断是否移除了最近最少使用的,如果是,则在原缓存中移除,同时设置标记eldestKey
  private void cycleKeyList(Object key) {
    keyMap.put(key, key);
    if (eldestKey != null) {
      delegate.removeObject(eldestKey);
      eldestKey = null;
    }
  }

}

// 定时清理缓存(默认一小时)
public class ScheduledCache implements Cache {

  private final Cache delegate;
  protected long clearInterval;
  protected long lastClear;

  public ScheduledCache(Cache delegate) {
    this.delegate = delegate;
    this.clearInterval = TimeUnit.HOURS.toMillis(1);
    this.lastClear = System.currentTimeMillis();
  }

  public void setClearInterval(long clearInterval) {
    this.clearInterval = clearInterval;
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    clearWhenStale();
    return delegate.getSize();
  }

  @Override
  public void putObject(Object key, Object object) {
    clearWhenStale();
    delegate.putObject(key, object);
  }

  @Override
  public Object getObject(Object key) {
    return clearWhenStale() ? null : delegate.getObject(key);
  }

  @Override
  public Object removeObject(Object key) {
    clearWhenStale();
    return delegate.removeObject(key);
  }

  @Override
  public void clear() {
    lastClear = System.currentTimeMillis();
    delegate.clear();
  }

  @Override
  public int hashCode() {
    return delegate.hashCode();
  }

  @Override
  public boolean equals(Object obj) {
    return delegate.equals(obj);
  }

  private boolean clearWhenStale() {
    if (System.currentTimeMillis() - lastClear > clearInterval) {
      clear();
      return true;
    }
    return false;
  }

}
// 同步处理

public class SynchronizedCache implements Cache {

  private final Cache delegate;

  public SynchronizedCache(Cache delegate) {
    this.delegate = delegate;
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public synchronized int getSize() {
    return delegate.getSize();
  }

  @Override
  public synchronized void putObject(Object key, Object object) {
    delegate.putObject(key, object);
  }

  @Override
  public synchronized Object getObject(Object key) {
    return delegate.getObject(key);
  }

  @Override
  public synchronized Object removeObject(Object key) {
    return delegate.removeObject(key);
  }

  @Override
  public synchronized void clear() {
    delegate.clear();
  }

  @Override
  public int hashCode() {
    return delegate.hashCode();
  }

  @Override
  public boolean equals(Object obj) {
    return delegate.equals(obj);
  }

}
```
> Cache的实现类中构造函数是 String id的为基础缓存, 参数类型为Cache的是装饰器装饰器通过new 的方式添加装饰

