---
title: Picasso源码分析
date: 2016-12-12 17:25:20
categories: 
- Android开源库源码分析
tags: 
- android
---


在Android中基本山所有的app都会涉及到网路图片的加载的需求，在Github上也有各式各样的开源的图片加载框架，从最初的UniversalImageLoader到现在的Gilde，Picasso和Fresco，对于图片加载框架的设计也趋于完善，现在主流也就是Gilde，Picasso和Fresco这个三个核心的框架了，那么这篇文章的目的就是从源码的角度来探究Picasso实现的过程。（本身适合了解基本api的童鞋阅读）

<!---more-->

----

<h3>基本使用</h3>

关于Picasso的基本的使用该方法如下所示：

```java
Picasso.with(context).load(url).into(imageView);
```

就只需要一句话就能够轻松将图片设置到你的ImageView当中，如果你有更多丰富的需求，诸如失败图片的正在加载图片的设置，只需要一句话，因为Picasso的图片展示基于Builder设计模式，我们可以任意添加所需要的功能，统统一句话：

```java
picasso.with(this).load(url).config(Bitmap.Config.ARGB_8888).error(R.mipmap.ic_launcher).into(img);  
```

用法非常简单，更多用法请仔细阅读api，本文的核心是对于源码的探究，下面我们基于picasso.with(context)为初始入口进行源码层级的阅读分析



----

<h3>源码分析</h3>

<h4>picasso.with(context)</h4>

with()方法的代码如下所示：

```java
public static Picasso with(Context context) {
    if (singleton == null) {
      synchronized (Picasso.class) {
        if (singleton == null) {
          singleton = new Builder(context).build();
        }
      }
    }
    return singleton;
  }

public Picasso build() {
      Context context = this.context;
	  //初始化图片下载的loader
      if (downloader == null) {
        downloader = Utils.createDefaultDownloader(context);
      }
  	  //缓存初始化
      if (cache == null) {
        cache = new LruCache(context);
      }
  	  //初始化线程池
      if (service == null) {
        service = new PicassoExecutorService();
      }
  	  //初始化一个请求转换器
      if (transformer == null) {
        transformer = RequestTransformer.IDENTITY;
      }
	  //缓存统计
      Stats stats = new Stats(cache);
	  //调度器
      Dispatcher dispatcher = new Dispatcher(context, service, HANDLER, downloader, cache, stats);

      return new Picasso(context, dispatcher, cache, listener, transformer, requestHandlers, stats,
          defaultBitmapConfig, indicatorsEnabled, loggingEnabled);
    }
  }
```

使用单例模式构造一个Picasso实现—singleton，初始化一个Builder，这个Builder的作用就是串联所有使用者设置的参数,并且调用其build()方法，在build的方法中做了一系列的初始化的操作：

1. 下载的loader的初始化
2. 图片缓存初始化
3. 线程池初始化
4. 请求转换器的初始化
5. 缓存统计类的初始化
6. 调度器的初始化

这些一些凭名字可以看出作用，诸如下载loader和图片缓存的，其他的只能一步步探究其作用，暂且搁置，后面遇到了我们再分析，接下来就到了Picasso的构造方法中了：

```java
Picasso.java 
Picasso(Context context, Dispatcher dispatcher, Cache cache, Listener listener,
      RequestTransformer requestTransformer, List<RequestHandler> extraRequestHandlers, Stats stats,
      Bitmap.Config defaultBitmapConfig, boolean indicatorsEnabled, boolean loggingEnabled) {
    this.context = context;
    this.dispatcher = dispatcher;
    this.cache = cache;
    this.listener = listener;
    this.requestTransformer = requestTransformer;
    this.defaultBitmapConfig = defaultBitmapConfig;

    int builtInHandlers = 7; // Adjust this as internal handlers are added or removed.
    int extraCount = (extraRequestHandlers != null ? extraRequestHandlers.size() : 0);
    List<RequestHandler> allRequestHandlers =
        new ArrayList<RequestHandler>(builtInHandlers + extraCount);

    //添加各种加载不同图片地址的handler，如网络中的，resoure中的，相册，conteneprovider....
    allRequestHandlers.add(new ResourceRequestHandler(context));

    if (extraRequestHandlers != null) {
      allRequestHandlers.addAll(extraRequestHandlers);
    }
   	//
    allRequestHandlers.add(new ContactsPhotoRequestHandler(context));
    allRequestHandlers.add(new MediaStoreRequestHandler(context));
    allRequestHandlers.add(new ContentStreamRequestHandler(context));
    allRequestHandlers.add(new AssetRequestHandler(context));
    allRequestHandlers.add(new FileRequestHandler(context));
    allRequestHandlers.add(new NetworkRequestHandler(dispatcher.downloader, stats));
    requestHandlers = Collections.unmodifiableList(allRequestHandlers);

    this.stats = stats;
    this.targetToAction = new WeakHashMap<Object, Action>();
    this.targetToDeferredRequestCreator = new WeakHashMap<ImageView, DeferredRequestCreator>();
    this.indicatorsEnabled = indicatorsEnabled;
    this.loggingEnabled = loggingEnabled;
    this.referenceQueue = new ReferenceQueue<Object>();
   	//初始化一个清除request的Thread
    this.cleanupThread = new CleanupThread(referenceQueue, HANDLER);
    this.cleanupThread.start();
  }
```

Picasso的构造函数中从Buidler中获取到使用者设置的额外参数设置添加到配置当中，并且赋值给各个变量。在allRequestHandlers维护着所有可能获取到图片的路径，包括网上获取，Drawable文件获取，Asset文件中的获取等等的取图方式，最后初始化一个清除request的Thread。在构造函数中初始化的内容是这些，接下来，我们在查看一下load()方法，我们挑选load(uri)即加载网络图片的来进行源码的探究：

```java
Picasso.java
public RequestCreator load(Uri uri) {
    return new RequestCreator(this, uri, 0);
  }
```

load(uri)的方法非常简单，即返回一个RequestCreator()，这个RequestCreator是所有load(...)方法的返回参数，我们得查看一下这个类的作用:

```java
RequestCreator.java 
RequestCreator(Picasso picasso, Uri uri, int resourceId) {
    if (picasso.shutdown) {
      throw new IllegalStateException(
          "Picasso instance already shut down. Cannot submit new requests.");
    }
    this.picasso = picasso;
    //初始化Reques类
    this.data = new Request.Builder(uri, resourceId, picasso.defaultBitmapConfig);
  }
public RequestCreator error(int errorResId){...}
public RequestCreator placeholder(int placeholderResId) {...}
public RequestCreator fit() {...}
public RequestCreator resize(int targetWidth, int targetHeight){...}
public RequestCreator centerInside(){...}
public RequestCreator clearCenterCrop(){...}
....
```

熟悉API的朋友就知道这些方法的具体作用，那么RequestCreator的作用在于在给target设置图片之前，读取用户一切对于目标target的操作，如：error图片的设置，加载过程中图片的设置，设置图片宽高等等，相当于一个预处理类的作用，然后把所有的参传递给Request。所以我们可以在调用into()方法之前将我们所有的要求设置到RequestCreator(Builder设计模式)。

我们再看构造方法中初始化了一个Request.Builder()，进入源码中查看：

```java
Request.java
  Builder(Uri uri, int resourceId, Bitmap.Config bitmapConfig) {
      this.uri = uri;
      this.resourceId = resourceId;
      this.config = bitmapConfig;
    }

```

Request.Builder所做的操作就是接收从RequestCreator传递过来参数



into()方法

```java
RequestCreator.java
public void into(ImageView target, Callback callback) {
    long started = System.nanoTime();
  	//是否为主线程
    checkMain();

    if (target == null) {
      throw new IllegalArgumentException("Target must not be null.");
    }
	//是否有图片资源，如果rui为空或者ResId为空，则取消request
  	//uri != null || resourceId != 0
    if (!data.hasImage()) {
      picasso.cancelRequest(target);
      //显示默认图片
      if (setPlaceholder) {
        setPlaceholder(target, getPlaceholderDrawable());
      }
      return;
    }
	//deferred参数在设置fit()方法时候为true，fit()方法的作用是让图片的宽高恰好等于imageView的宽高
  	//掉用过这个方法后不能再调用resize()操作
    if (deferred) {
      if (data.hasSize()) {
        throw new IllegalStateException("Fit cannot be used with resize.");
      }
      int width = target.getWidth();
      int height = target.getHeight();
      if (width == 0 || height == 0) {
        if (setPlaceholder) {
          setPlaceholder(target, getPlaceholderDrawable());
        }
        picasso.defer(target, new DeferredRequestCreator(this, target, callback));
        return;
      }
      //设置图片宽高
      data.resize(width, height);
    }
	//新建request
    Request request = createRequest(started);
  	//新建requestKey，即得到一个唯一标识request的key
    String requestKey = createKey(request);
	//是否从缓存中读取
    if (shouldReadFromMemoryCache(memoryPolicy)) {
      Bitmap bitmap = picasso.quickMemoryCacheCheck(requestKey);
      //得到缓存中的图片并且取消请求
      if (bitmap != null) {
        picasso.cancelRequest(target);
        setBitmap(target, picasso.context, bitmap, MEMORY, noFade, picasso.indicatorsEnabled);
        if (picasso.loggingEnabled) {
          log(OWNER_MAIN, VERB_COMPLETED, request.plainId(), "from " + MEMORY);
        }
        //回调接口
        if (callback != null) {
          callback.onSuccess();
        }
        return;
      }
    }

    if (setPlaceholder) {
      setPlaceholder(target, getPlaceholderDrawable());
    }
	//构造一个图片请求的动作
    Action action =
        new ImageViewAction(picasso, target, request, memoryPolicy, networkPolicy, errorResId,
            errorDrawable, requestKey, tag, callback, noFade);
	//添加到请求队列当中
    picasso.enqueueAndSubmit(action);
  }
```

在into()方法中主要的执行过程有如下几个步骤：

- 检查运行的线程是否为主线程，如果不是抛异常，结束
- 检查传入的uri或者resid等是否有图片资源，如果rui为空或者ResId为空，则取消request，结束
- 检查deferred参数，deferred参数在设置fit()方法时候为true，fit()方法的作用是让图片的宽高恰好等于imageView的宽高，调用过这个方法后不能再调用resize()操作。
- 新建Request，判断是是否拥有缓存，如果有则直接获取并且取消请求，调用setBitMap()为target设置图片，结束
- 如果没有缓存，则会构造一个Action来处理请求，然后添加至线程池当中

**小提示：**

在其中 createRequest(long started)方法中，会有一个请求转换的过程，这个过程一句使用者是否使用而决定，for  example:如果使用者在其中某一个ImageView的展示中想使用一个baseUrl不同的地址，那么我们就可以实现RequestTransformer接口，并且重写transformRequest(request)来自主构建Request实例。



```java
Piscasso.java
void enqueueAndSubmit(Action action) {
    Object target = action.getTarget();
    if (target != null && targetToAction.get(target) != action) {
      // This will also check we are on the main thread.
      cancelExistingRequest(target);
      targetToAction.put(target, action);
    }
    submit(action);
  }

  void submit(Action action) {
    dispatcher.dispatchSubmit(action);
  }
```

跟踪enqueueAndSubmit()最终调用到Dispatcher类当中，Dispatcher类所有的action操作(submit,cancel,pause.etc)，相关对应的逻辑处理完后会使用BitmapHunter来获取图片，BitmapHunter会最终回调Dispatcher的图片或是否成功的相关方法。

简单看下Dispatcher的构造函数：

Dispatcher.java

```java
Dispatcher.java
Dispatcher(Context context, ExecutorService service, Handler mainThreadHandler,
      Downloader downloader, Cache cache, Stats stats) {
    this.dispatcherThread = new DispatcherThread();
    this.dispatcherThread.start();
    Utils.flushStackLocalLeaks(dispatcherThread.getLooper());
    this.context = context;
    this.service = service;
  	//hunter的映射
    this.hunterMap = new LinkedHashMap<String, BitmapHunter>();
  	//失败Action的映射
    this.failedActions = new WeakHashMap<Object, Action>();
  	//暂停Action的映射
    this.pausedActions = new WeakHashMap<Object, Action>();
    this.pausedTags = new HashSet<Object>();
  	//DispatcherHandler初始化
    this.handler = new DispatcherHandler(dispatcherThread.getLooper(), this);
  	//下载类
    this.downloader = downloader;
    this.mainThreadHandler = mainThreadHandler;
    this.cache = cache;
    this.stats = stats;
    this.batch = new ArrayList<BitmapHunter>(4);
    this.airplaneMode = Utils.isAirplaneModeOn(this.context);
    this.scansNetworkChanges = hasPermission(context, Manifest.permission.ACCESS_NETWORK_STATE);
    this.receiver = new NetworkBroadcastReceiver(this);
    receiver.register();
  }

void dispatchSubmit(Action action) {
    handler.sendMessage(handler.obtainMessage(REQUEST_SUBMIT, action));
  }

 private static class DispatcherHandler extends Handler {
    private final Dispatcher dispatcher;

    public DispatcherHandler(Looper looper, Dispatcher dispatcher) {
      super(looper);
      this.dispatcher = dispatcher;
    }

    @Override public void handleMessage(final Message msg) {
      switch (msg.what) {
        case REQUEST_SUBMIT: {
          Action action = (Action) msg.obj;
          dispatcher.performSubmit(action);
          break;
        }
        case REQUEST_CANCEL: {
          Action action = (Action) msg.obj;
          dispatcher.performCancel(action);
          break;
        }
        case TAG_PAUSE: {
          Object tag = msg.obj;
          dispatcher.performPauseTag(tag);
          break;
        }
        case TAG_RESUME: {
          Object tag = msg.obj;
          dispatcher.performResumeTag(tag);
          break;
        }
        case HUNTER_COMPLETE: {
          BitmapHunter hunter = (BitmapHunter) msg.obj;
          dispatcher.performComplete(hunter);
          break;
        }
        case HUNTER_RETRY: {
          BitmapHunter hunter = (BitmapHunter) msg.obj;
          dispatcher.performRetry(hunter);
          break;
        }
        case HUNTER_DECODE_FAILED: {
          BitmapHunter hunter = (BitmapHunter) msg.obj;
          dispatcher.performError(hunter, false);
          break;
        }
        case HUNTER_DELAY_NEXT_BATCH: {
          dispatcher.performBatchComplete();
          break;
        }
        case NETWORK_STATE_CHANGE: {
          NetworkInfo info = (NetworkInfo) msg.obj;
          dispatcher.performNetworkStateChange(info);
          break;
        }
        case AIRPLANE_MODE_CHANGE: {
          dispatcher.performAirplaneModeChange(msg.arg1 == AIRPLANE_MODE_ON);
          break;
        }
        default:
          Picasso.HANDLER.post(new Runnable() {
            @Override public void run() {
              throw new AssertionError("Unknown handler message received: " + msg.what);
            }
          });
      }
    }
  }
```

在初始化方法中，核心就是DispatcherHandler初始化，DispatcherHandler负责将消息进行传递调用diapatcher相应的方法，以Msg=REQUEST_SUBMIT为例，那么就会调用dispatcher.performSubmit(action)，dispatcher.performSubmit(action)方法如下所示：

```java
Dispatcher.java
void performSubmit(Action action, boolean dismissFailed) {
  	//暂停标识
    if (pausedTags.contains(action.getTag())) {
      pausedActions.put(action.getTarget(), action);
      if (action.getPicasso().loggingEnabled) {
        log(OWNER_DISPATCHER, VERB_PAUSED, action.request.logId(),
            "because tag '" + action.getTag() + "' is paused");
      }
      return;
    }
	
    BitmapHunter hunter = hunterMap.get(action.getKey());
    if (hunter != null) {
      hunter.attach(action);
      return;
    }

    if (service.isShutdown()) {
      if (action.getPicasso().loggingEnabled) {
        log(OWNER_DISPATCHER, VERB_IGNORED, action.request.logId(), "because shut down");
      }
      return;
    }

    hunter = forRequest(action.getPicasso(), this, cache, stats, action);
    hunter.future = service.submit(hunter);
    hunterMap.put(action.getKey(), hunter);
    if (dismissFailed) {
      failedActions.remove(action.getTarget());
    }

    if (action.getPicasso().loggingEnabled) {
      log(OWNER_DISPATCHER, VERB_ENQUEUED, action.request.logId());
    }
  }
```

这个方法的逻辑如下:

1. 暂停的tags中是否包含当前action的tag，有的话则return，否则继续下一步
2. hunterMap中是否已经包含了相同的hunter，如果有则attach()到hunter中，return；否则继续下一步
3. 线程池是否shutDown，有则return，没有继续下一步
4. 通过forRequest()方法获得一个request，并且添加到线程池service中


在forRequest()方法中主要获取对应的RequestHandler处理类，这里类似于责任链的设计模式，通过依次传递request给各个handler来找到对应的匹配的RequestHandler。

BitmapHunter是主要的图片处理类，在BitmapHunter中会判断是从网络中还是缓存中获取图片,由于BitmapHunter本身是线程类，且无论从缓存中取数据还是从网络中取数据都是耗时操作，所以才引申除了线程池service这个类来进行执行runnable，查看下代码：

```java
BitmapHunter.java
static BitmapHunter forRequest(Picasso picasso, Dispatcher dispatcher, Cache cache, Stats stats,
      Action action) {
    Request request = action.getRequest();
    List<RequestHandler> requestHandlers = picasso.getRequestHandlers();

    // Index-based loop to avoid allocating an iterator.
    //noinspection ForLoopReplaceableByForEach
    for (int i = 0, count = requestHandlers.size(); i < count; i++) {
      RequestHandler requestHandler = requestHandlers.get(i);
      if (requestHandler.canHandleRequest(request)) {
        return new BitmapHunter(picasso, dispatcher, cache, stats, action, requestHandler);
      }
    }

    return new BitmapHunter(picasso, dispatcher, cache, stats, action, ERRORING_HANDLER);
  }
....
  @Override public void run() {
    try {
      //更新县城名字
      updateThreadName(data);
      //核心方法
      result = hunt();

      if (result == null) {
        dispatcher.dispatchFailed(this);
      } else {
        dispatcher.dispatchComplete(this);
      }
      //各种异常的捕获处理
    } catch (Downloader.ResponseException e) {
      if (!e.localCacheOnly || e.responseCode != 504) {
        exception = e;
      }
      dispatcher.dispatchFailed(this);
    } catch (NetworkRequestHandler.ContentLengthException e) {
      exception = e;
      dispatcher.dispatchRetry(this);
    } catch (IOException e) {
      exception = e;
      dispatcher.dispatchRetry(this);
    } catch (OutOfMemoryError e) {
      StringWriter writer = new StringWriter();
      stats.createSnapshot().dump(new PrintWriter(writer));
      exception = new RuntimeException(writer.toString(), e);
      dispatcher.dispatchFailed(this);
    } catch (Exception e) {
      exception = e;
      dispatcher.dispatchFailed(this);
    } finally {
      Thread.currentThread().setName(Utils.THREAD_IDLE_NAME);
    }
  }
```

BitmapHunter.run()方法中主要通过调用hunt()方法获取到Bitmap，此时会对result进行判断的操作，如果为空则回调dispatcher.dispatchFailed()，不为空则调用dispatcher.dispatchComplete()方法，并且捕获异常，根据相关的异常来对应回调到dispatcher不同的方法中。

BitmapHunter.hunt()方法详解：

```java
Bitmap hunt() throws IOException {
    Bitmap bitmap = null;
	//是否从缓存读取
    if (shouldReadFromMemoryCache(memoryPolicy)) {
      bitmap = cache.get(key);
      if (bitmap != null) {
        stats.dispatchCacheHit();
        loadedFrom = MEMORY;
        if (picasso.loggingEnabled) {
          log(OWNER_HUNTER, VERB_DECODED, data.logId(), "from cache");
        }
        return bitmap;
      }
    }
	//网络读取
    data.networkPolicy = retryCount == 0 ? NetworkPolicy.OFFLINE.index : networkPolicy;
    RequestHandler.Result result = requestHandler.load(data, networkPolicy);
    if (result != null) {
      loadedFrom = result.getLoadedFrom();
      exifRotation = result.getExifOrientation();

      bitmap = result.getBitmap();

      // If there was no Bitmap then we need to decode it from the stream.
      if (bitmap == null) {
        InputStream is = result.getStream();
        try {
          bitmap = decodeStream(is, data);
        } finally {
          Utils.closeQuietly(is);
        }
      }
    }

    if (bitmap != null) {
      if (picasso.loggingEnabled) {
        log(OWNER_HUNTER, VERB_DECODED, data.logId());
      }
      stats.dispatchBitmapDecoded(bitmap);
      if (data.needsTransformation() || exifRotation != 0) {
        synchronized (DECODE_LOCK) {
          if (data.needsMatrixTransform() || exifRotation != 0) {
            bitmap = transformResult(data, bitmap, exifRotation);
            if (picasso.loggingEnabled) {
              log(OWNER_HUNTER, VERB_TRANSFORMED, data.logId());
            }
          }
          if (data.hasCustomTransformations()) {
            bitmap = applyCustomTransformations(data.transformations, bitmap);
            if (picasso.loggingEnabled) {
              log(OWNER_HUNTER, VERB_TRANSFORMED, data.logId(), "from custom transformations");
            }
          }
        }
        if (bitmap != null) {
          stats.dispatchBitmapTransformed(bitmap);
        }
      }
    }

    return bitmap;
  }
```

hunt()方法过程如下：

1. 从缓存里面找到对应的图片，有则返回，没有则玩下走；

2. 将Request交付给对应的RequestHandler.load()来获取到图片，并且判断是否需要加工(needsTransformation

   ()),需要则转换图片后再返回Bitmap

回到BitmapHunter.createRequest()，最终过程都会调用到dispatcher.dispatchComplete(this)和dispatcher.dispatchFailed(this)，而这两个方法会调用到dispatcher.performBatchComplete()中。

附：

- dispatcher.dispatchRetry(this)：主要对requset进行重新请求操作，会再次执行一次获取图片的过程
- dispatcher.dispatchComplete(this)：最终调用performBatchComplete()对主线程发送一个承载着List<BitmapHunter>的msg
- dispatcher.dispatchFailed(this)：最终调用performBatchComplete()对主线程发送一个承载着List<BitmapHunter>的msg

在其中又会将HUNTER_BATCH_COMPLETE这个msg发送给主线程的handler，即在Picasso.java中初始化的HANDLER:

```java
Dispatcher.java
void performBatchComplete() {
    List<BitmapHunter> copy = new ArrayList<BitmapHunter>(batch);
    batch.clear();
    mainThreadHandler.sendMessage(mainThreadHandler.obtainMessage(HUNTER_BATCH_COMPLETE, copy));
    logBatch(copy);
  }
void performError(BitmapHunter hunter, boolean willReplay) {
    if (hunter.getPicasso().loggingEnabled) {
      log(OWNER_DISPATCHER, VERB_BATCHED, getLogIdsForHunter(hunter),
          "for error" + (willReplay ? " (will replay)" : ""));
    }
    hunterMap.remove(hunter.getKey());
    batch(hunter);
  }


Picasso.java
static final Handler HANDLER = new Handler(Looper.getMainLooper()) {
    @Override public void handleMessage(Message msg) {
      switch (msg.what) {
        case HUNTER_BATCH_COMPLETE: {
          @SuppressWarnings("unchecked") List<BitmapHunter> batch = (List<BitmapHunter>) msg.obj;
          //noinspection ForLoopReplaceableByForEach
          for (int i = 0, n = batch.size(); i < n; i++) {
            BitmapHunter hunter = batch.get(i);
            hunter.picasso.complete(hunter);
          }
          break;
        }
        case REQUEST_GCED: {
          Action action = (Action) msg.obj;
          if (action.getPicasso().loggingEnabled) {
            log(OWNER_MAIN, VERB_CANCELED, action.request.logId(), "target got garbage collected");
          }
          action.picasso.cancelExistingRequest(action.getTarget());
          break;
        }
        case REQUEST_BATCH_RESUME:
          @SuppressWarnings("unchecked") List<Action> batch = (List<Action>) msg.obj;
          //noinspection ForLoopReplaceableByForEach
          for (int i = 0, n = batch.size(); i < n; i++) {
            Action action = batch.get(i);
            action.picasso.resumeAction(action);
          }
          break;
        default:
          throw new AssertionError("Unknown handler message received: " + msg.what);
      }
    }
  };
```



hunter.picasso.complete(hunter)方法：

```java
Picasso.java 
void complete(BitmapHunter hunter) {
    Action single = hunter.getAction();
    List<Action> joined = hunter.getActions();

    boolean hasMultiple = joined != null && !joined.isEmpty();
    boolean shouldDeliver = single != null || hasMultiple;

    if (!shouldDeliver) {
      return;
    }

    Uri uri = hunter.getData().uri;
    Exception exception = hunter.getException();
    Bitmap result = hunter.getResult();
    LoadedFrom from = hunter.getLoadedFrom();

    if (single != null) {
      deliverAction(result, from, single);
    }

    if (hasMultiple) {
      //noinspection ForLoopReplaceableByForEach
      for (int i = 0, n = joined.size(); i < n; i++) {
        Action join = joined.get(i);
        deliverAction(result, from, join);
      }
    }

    if (listener != null && exception != null) {
      listener.onImageLoadFailed(this, uri, exception);
    }
  }

private void deliverAction(Bitmap result, LoadedFrom from, Action action) {
    if (action.isCancelled()) {
      return;
    }
    if (!action.willReplay()) {
      targetToAction.remove(action.getTarget());
    }
    if (result != null) {
      if (from == null) {
        throw new AssertionError("LoadedFrom cannot be null.");
      }
      action.complete(result, from);
      if (loggingEnabled) {
        log(OWNER_MAIN, VERB_COMPLETED, action.request.logId(), "from " + from);
      }
    } else {
      action.error();
      if (loggingEnabled) {
        log(OWNER_MAIN, VERB_ERRORED, action.request.logId());
      }
    }
  }
```

complete()方法中最终会调用action.complete(result, from)和action.error()来传递给对应的Action实现类，在这里我们选择的是ImageAction来查看对应方法

```java
@Override public void complete(Bitmap result, Picasso.LoadedFrom from) {
    if (result == null) {
      throw new AssertionError(
          String.format("Attempted to complete action with no result!\n%s", this));
    }

    ImageView target = this.target.get();
    if (target == null) {
      return;
    }

    Context context = picasso.context;
    boolean indicatorsEnabled = picasso.indicatorsEnabled;
  	//设置图片
    PicassoDrawable.setBitmap(target, context, result, from, noFade, indicatorsEnabled);

    if (callback != null) {
      callback.onSuccess();
    }
  }

  @Override public void error() {
    ImageView target = this.target.get();
    if (target == null) {
      return;
    }
    //设置错误图片
    if (errorResId != 0) {
      target.setImageResource(errorResId);
    } else if (errorDrawable != null) {
      target.setImageDrawable(errorDrawable);
    }

    if (callback != null) {
      callback.onError();
    }
  }
```

到这里，基本上也结束了，对应的图片也展示到了Target中，基本的流程也就完成了