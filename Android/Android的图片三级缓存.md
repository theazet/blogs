# Android中图片的三级缓存策略

> 从网络上加载图片是一种非常耗费流量的事情，而且网速较慢的时候加载会缓慢，另外Bitmap加载图片非常消耗时间和内存。

**三级缓存是什么？**

1. 内存：一级缓存，速度最快，缓存在内存中
2. 本地：二级缓存，速度比较快，缓存在本地文件中
3. 网络：三级缓存，速度慢，耗费流量，从网络中拉取

**三级缓存的优点**

使用三级缓存策略可以减少流量的消耗，加快图片的加载速度。

**三级缓存的实现方法**

1. 首次加载从网络加载，然后将其缓存到内存和本地文件中
2. 非首次加载的图片优先访问内存中的存储（一般使用LruCache）
3. 如果内存中没有缓存，那么访问本地文件中的缓存（一般使用DIskLruCache）

### LruCache原理分析

我们先看LruCache的构造方法：

```java
    public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
```

LruCache存储数据的结构是一个LinkedHashMap，它的accessOrder为true，所以它是按访问顺序排序的。具体内容可以看我的[散列与java中散列的应用]([https://www.theaze.cn/2019/04/16/%E6%95%A3%E5%88%97%E4%B8%8Ejava%E4%B8%AD%E6%95%A3%E5%88%97%E7%9A%84%E5%BA%94%E7%94%A8/](https://www.theaze.cn/2019/04/16/散列与java中散列的应用/))。

我们再看它的put方法

```java
    public final V put(K key, V value) {
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        synchronized (this) {
            putCount++;
            /**
            	safeSizeOf最终调用的是sizeOf(key, value)方法，如果不重写，那么其默认返回1，也就是
            	代表的缓存的个数。
            	如果存储图片时要在sizeOf方法中返回图片所占内存的大小
			*/
            size += safeSizeOf(key, value);
            //将创建的新元素添加进缓存队列，并添加成功后返回这个元素
            previous = map.put(key, value);
            if (previous != null) {
                //如果返回的是null，说明添加缓存失败，在已用缓存大小中减去这个元素的大小。
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            //无限循环删除最久未被使用的缓存，直到已使用的缓存小于size
            entryRemoved(false, key, previous, value);
        }

        trimToSize(maxSize);
        return previous;
    }
```

这说明了LruCache当可用缓存不足时，循环删除最久未被使用的缓存，直到新的缓存可以加入。

### DiskLruCache原理分析

DiskLruCache是巨佬JakeWharton编写的用于本地缓存的解决方案，且获得了Google官方的认证。

DiskLruCache的构造方法是私有的，须从其open()方法获取实例

```java
public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)
```

它需要四个参数：缓存的路径、app版本号、一个key对应多少个缓存文件、最大可以存储多少数据。

DiskLruCache写入缓存需要Editor类的edit()方法，这个方法需要传入一个key，它将会成为key的文件名，这个key需要与图片的url一致，但因为url中可能包含特殊字符，这样命名文件名是不合法的，所以需要进行将key进行MD5编码

```java
String imageUrl = "https://xxx.jpg";
String key = hashKey(imageUrl);
DiskLruCache.Editor editor = mDiskLruCache.edit(key);
```

DiskLruCache和LruCache类似，它也是用了一个accessOrder为true的LinkedHashMap存储数据。

```java
  private final LinkedHashMap<String, Entry> lruEntries =
      new LinkedHashMap<String, Entry>(0, 0.75f, true);
```

journal文件是DiskLruCache一个日志文件，DiskLruCache能够正常工作的前提就是要依赖于journal文件中的内容，当我们调用editor方法时会写入一条DIRTY记录，当DIRTY没有任何CLEAN或REMOVE记录时候该记录会被删除；，当调用commit方法时会写入一条CLEAN记录，并在记录最后写入该条缓存的大小；当我们调用remove方法删除数据后，就会写入一条REMOVE记录；当我们调用get()方法去读取一条缓存数据时，就会写入一条READ记录

### Bitmap的高效加载

Bitmap提供了四类方法：decodeFile、decodeResource、decodeStream、decodeByteArray，分别从文件、资源、输入流、字节流中加载Bitmap对象。

如果Bitmap尺寸过大，但是ImageView没有那么大，ImageView就无法显示原始图片，这样我们就可以通过BitmapFactory.Options来对图片进行采样，这样即降低了内存占用，一定程度上避免了OOM，又提高了加载的性能。

BitmapFactory.Options来采样使用的是iSampleSize参数，当它为1时是原样加载；当它小于1时和1是一样的效果，iSampleSize参数的值应为2的指数，如果不为2的指数就下降为比iSampleSize小且最近的2的指数。一个1024X1024像素的图片，设置iSampleSize为2时，采样后的大小为512X512像素。

当BitmapFactory的inJustDecodeBounds为true时只会解析图片的宽高信息，不会真正的加载图片，所以就可以利用这一特性计算采样率。

过程为：

1. 设置inJustDecodeBounds为true加载图片
2. 通过BitmapFactory.Options解析图片的宽高信息，计算采样率
3. 设置inJustDecodeBounds为false加载图片