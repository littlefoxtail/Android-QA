Picasso是一个小而雅的框架，聚焦于图片加载的主要问题：
1. 异步过程管理与调度
2. 资源与显示容器多对多关系
3. 多级缓存
4. 图片转换支持ScaleType

Glide着眼于解决这样一个问题：
> 构建一个系统，对指定的标识媒体资源的实体，其封装形式可以自行扩展，执行资源的获取，并应用内存、磁盘进行缓存，封装图片的解码过程，按指定要求进行裁切、缩放等图片转换

# 工作原理


- 缓存与池
	- MemorySizeCalculator
	- LruBitmapPool
	- BitmapPoolAdapter
	- LruArrayPool
	- LruResourceCache
	- InternalCacheDiskCacheFactory
- 资源获取
	- RequestManagerRetriever
	- DefaultConnectivityMonitorFactory
- 核心流程管理
	- Engine
- 线程池
	- GlideExecutor
# 分支流程
## Glide如何加载内存资源
loadFromeMemory
要从load方法开始
1. Engin.load
```java
//EnginKey如何唯一的标识某一个图片资源key
public boolean equals(Object o) {
	if (o instanceof EngineKey) {
	  EngineKey other = (EngineKey) o;
	  return model.equals(other.model)
		  && signature.equals(other.signature)
		  && height == other.height
		  && width == other.width
		  && transformations.equals(other.transformations)
		  && resourceClass.equals(other.resourceClass)
		  && transcodeClass.equals(other.transcodeClass)
		  && options.equals(other.options);
	}
	return false;
}
```
2. Engin.loadFromMemory方法加载内存缓存数据
```java
  private EngineResource<?> loadFromMemory(
      EngineKey key, boolean isMemoryCacheable, long startTime) {
    if (!isMemoryCacheable) {
      return null;
    }
	//尝试从正在请求的资源中获取一个请求，这个正在请求的资源单纯的被放在一个HashMap中
    EngineResource<?> active = loadFromActiveResources(key);
    if (active != null) {
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      }
      return active;
    }
//去内存缓存中查找，这个实际会在LruCache中获取
    EngineResource<?> cached = loadFromCache(key);
    if (cached != null) {
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
      return cached;
    }

    return null;
  }

```
- loadFromActiveResources
从正在请求的资源中查找图片资源
- loadFromCache方法
```java
  private EngineResource<?> loadFromCache(Key key) {
    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
      cached.acquire();
      activeResources.activate(key, cached);
    }
    return cached;
  }

```
## Glide是如何去磁盘中加载缓存的
### ResourceCacheGenerator
```java
public boolean startNext() {
    GlideTrace.beginSection("ResourceCacheGenerator.startNext");
    try {
      List<Key> sourceIds = helper.getCacheKeys();
      if (sourceIds.isEmpty()) {
        return false;
      }
      //我们需要的资源类型
      List<Class<?>> resourceClasses = helper.getRegisteredResourceClasses();
      if (resourceClasses.isEmpty()) {
        if (File.class.equals(helper.getTranscodeClass())) {
          return false;
        }

}
```

```java
//转换类，可以直接将资源类型转换为Glide需要的资源类型
        Transformation<?> transformation = helper.getTransformation(resourceClass);
        // PMD.AvoidInstantiatingObjectsInLoops Each iteration is comparatively expensive anyway,
        // we only run until the first one succeeds, the loop runs for only a limited
        // number of iterations on the order of 10-20 in the worst case.
        //获取disk缓存的key
        currentKey =
            new ResourceCacheKey( // NOPMD AvoidInstantiatingObjectsInLoops
                helper.getArrayPool(),
                sourceId,
                helper.getSignature(),
                helper.getWidth(),
                helper.getHeight(),
                transformation,
                resourceClass,
                helper.getOptions());
                //获取缓存文件
        cacheFile = helper.getDiskCache().get(currentKey);
        if (cacheFile != null) {
          sourceKey = sourceId;
          //如果缓存文件不为空，则返回资源加载器
          modelLoaders = helper.getModelLoaders(cacheFile);
          modelLoaderIndex = 0;
        }
      }


```