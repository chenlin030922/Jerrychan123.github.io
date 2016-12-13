---
title: volley源码学习
date: 2016-12-09 12:52:19
categories: 
- Android开源库源码分析
tags: 
- android
---

----

Volley 是 Google 推出的 Android 异步网络请求框架和图片加载框架。在 Google I/O 2013 大会上发布的，现在连android系统中内部也是使用了Volley作为其网络的请求框架，谷歌出品，必属精品，所以有必要进行一次梳理其实现的总体过程，学习其设计框架的思路也是必要的(文章适合知道基本使用Volley的同学观看)。
<!--more-->
-------

<h3>Volley</h3>

一般我们都会在app初始化的时候调用如下代码进行初始化Volley，下面就从该方法当做阅读入口进行逐一跟踪：

```java
mRequestQueue =  Volley.newRequestQueue(this); 
```

在调用Volley.newRequestQueue(this)时，最终调用到如下代码:

```java
 /**
     * Creates a default instance of the worker pool and calls {@link RequestQueue#start()} on it.
     * You may set a maximum size of the disk cache in bytes.
     *HttpStack是用来进行网络请求封装的类，传入null代表默认
     *maxDiskCacheBytes为最大磁盘缓存，传入-1使用默认大小
     */
    public static RequestQueue newRequestQueue(Context context, HttpStack stack, int maxDiskCacheBytes) {
      //构造缓存区域
        File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);

        String userAgent = "volley/0";
        try {
            String packageName = context.getPackageName();
            PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
            userAgent = packageName + "/" + info.versionCode;
        } catch (NameNotFoundException e) {
        }
		//我们可以传入okttp来充当我们的httpStack
        if (stack == null) {
          //使用Volley默认封装的stack，依靠android版本来决定使用不同的connection来进行网络请求
            if (Build.VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        Network network = new BasicNetwork(stack);
        
        RequestQueue queue;
        if (maxDiskCacheBytes <= -1)
        {
        	// No maximum size specified
        	queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        }
        else
        {
        	// Disk cache size specified
        	queue = new RequestQueue(new DiskBasedCache(cacheDir, maxDiskCacheBytes), network);
        }

        queue.start();

        return queue;
    }
```

在newRequestQueue方法中进行了相关必要参数的初始化的工作，其中包括:

- 缓存区域的初始化
- HttpStack的初始化
- Network的初始化
- RequestQueue的初始化，最终调用其start()方法

------

#### Stack 详解

在Volley中如果没有传递任何HttpStack则会使用了两种stack来进行不同android版本(SDK_INT >= 9?HurlStack:HttpClientStack)的适配工作：HurlStack和HttpClientStack，两种stack都实现了HttpStack接口，接口定义如下：

```java
public interface HttpStack {
    //请求处理方法
    public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders)
        throws IOException, AuthFailureError;

}
```

在接口中定义performRequest方法，在其子类中实现了对于http请求的封装的操作，以此来查看对应实现子类的performRequest方法。

在HurlStack中关于performRequest(…)的实现：

```java
@Override
    public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders)
            throws IOException, AuthFailureError {
        String url = request.getUrl();
      //存放请求头
        HashMap<String, String> map = new HashMap<String, String>();
        map.putAll(request.getHeaders());
        map.putAll(additionalHeaders);
      //是否重写url
        if (mUrlRewriter != null) {
            String rewritten = mUrlRewriter.rewriteUrl(url);
            if (rewritten == null) {
                throw new IOException("URL blocked by rewriter: " + url);
            }
            url = rewritten;
        }
        URL parsedUrl = new URL(url);
      //使用后HttpURLConnection进行网络请求
        HttpURLConnection connection = openConnection(parsedUrl, request);
        for (String headerName : map.keySet()) {
            connection.addRequestProperty(headerName, map.get(headerName));
        }
      //设置请求方法：GET ,POST, ect
        setConnectionParametersForRequest(connection, request);
        // Initialize HttpResponse with data from the HttpURLConnection.
        ProtocolVersion protocolVersion = new ProtocolVersion("HTTP", 1, 1);
        int responseCode = connection.getResponseCode();
        if (responseCode == -1) {
            // -1 is returned by getResponseCode() if the response code could not be retrieved.
            // Signal to the caller that something was wrong with the connection.
            throw new IOException("Could not retrieve response code from HttpUrlConnection.");
        }
        StatusLine responseStatus = new BasicStatusLine(protocolVersion,
                connection.getResponseCode(), connection.getResponseMessage());
        BasicHttpResponse response = new BasicHttpResponse(responseStatus);
      	//将返回的数据流写入response
        response.setEntity(entityFromConnection(connection));
      	//获取返回的请求头
        for (Entry<String, List<String>> header : connection.getHeaderFields().entrySet()) {
            if (header.getKey() != null) {
                Header h = new BasicHeader(header.getKey(), header.getValue().get(0));
                response.addHeader(h);
            }
        }
        return response;
    }
```

HurlStack使用HttpURLConnection进行Http请求的过程，步骤是基本的网络请求的过程，设置请求头->包装Url->打开连接->设置请求方法->获取数据已经请求状态->将数据流存入HttpResponse，最终返回一个HttpResponse。

HttpClientStack中关于performRequest(…)的实现：

```java
@Override
    public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders)
            throws IOException, AuthFailureError {
      	//构造请求参数
        HttpUriRequest httpRequest = createHttpRequest(request, additionalHeaders);
      	//添加请求头
        addHeaders(httpRequest, additionalHeaders);
        addHeaders(httpRequest, request.getHeaders());
        onPrepareRequest(httpRequest);
        HttpParams httpParams = httpRequest.getParams();
        int timeoutMs = request.getTimeoutMs();
        // TODO: Reevaluate this connection timeout based on more wide-scale
        // data collection and possibly different for wifi vs. 3G.
        HttpConnectionParams.setConnectionTimeout(httpParams, 5000);
        HttpConnectionParams.setSoTimeout(httpParams, timeoutMs);
        return mClient.execute(httpRequest);
    }
```

HttpClientStack执行的大体过程与HurlStack基本上一致，通过一系列操作最终调用AndroidHttpClient.execute(httpRequest)返回一个HttpResponse。



HttpStack封装了http请求的过程，并且只管http请求，满足设计模式中单一责任的原则，并且向外提供接口HttpStack方便使用者定制自身的HttpStack类来进行请求的过程，如：可以用OkhttpClient来进行http请求的过程，这对于Volley的整个流程是不受影响。

-----

#### Network

```java
public interface Network {
    /**
     * Performs the specified request.
     * @param request Request to process
     * @return A {@link NetworkResponse} with data and caching metadata; will never be null
     * @throws VolleyError on errors
     */
    public NetworkResponse performRequest(Request<?> request) throws VolleyError;
}
```

Network接口中定义了performRequest用于接收一个具体的Request请求，其实现类是BasicNetwork，我们直接查看performRequest方法：

```java
@Override
    public NetworkResponse performRequest(Request<?> request) throws VolleyError {
        long requestStart = SystemClock.elapsedRealtime();
      	//死循环
        while (true) {
            HttpResponse httpResponse = null;
            byte[] responseContents = null;
            Map<String, String> responseHeaders = Collections.emptyMap();
            try {
                // Gather headers.
                Map<String, String> headers = new HashMap<String, String>();
                addCacheHeaders(headers, request.getCacheEntry());
              	//通过HttpStack获得HttpResponse
                httpResponse = mHttpStack.performRequest(request, headers);
                StatusLine statusLine = httpResponse.getStatusLine();
                int statusCode = statusLine.getStatusCode();
				//返回的请求头
                responseHeaders = convertHeaders(httpResponse.getAllHeaders());
                //请求返回的状态码处理
                if (statusCode == HttpStatus.SC_NOT_MODIFIED) {
                    Entry entry = request.getCacheEntry();
                    if (entry == null) {
                        return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, null,
                                responseHeaders, true,
                                SystemClock.elapsedRealtime() - requestStart);
                    }

                    entry.responseHeaders.putAll(responseHeaders);
                    return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, entry.data,
                            entry.responseHeaders, true,
                            SystemClock.elapsedRealtime() - requestStart);
                }
                
                // Handle moved resources
                if (statusCode == HttpStatus.SC_MOVED_PERMANENTLY || statusCode == HttpStatus.SC_MOVED_TEMPORARILY) {
                	String newUrl = responseHeaders.get("Location");
                	request.setRedirectUrl(newUrl);
                }

                // Some responses such as 204s do not have content.  We must check.
                if (httpResponse.getEntity() != null) {
                  responseContents = entityToBytes(httpResponse.getEntity());
                } else {
                  // Add 0 byte response as a way of honestly representing a
                  // no-content request.
                  responseContents = new byte[0];
                }

                // if the request is slow, log it.
                long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
                logSlowRequests(requestLifetime, request, responseContents, statusLine);

                if (statusCode < 200 || statusCode > 299) {
                    throw new IOException();
                }
                return new NetworkResponse(statusCode, responseContents, responseHeaders, false,
                        SystemClock.elapsedRealtime() - requestStart);
            	...
                int statusCode = 0;
                NetworkResponse networkResponse = null;
                if (httpResponse != null) {
                    statusCode = httpResponse.getStatusLine().getStatusCode();
                } else {
                    throw new NoConnectionError(e);
                }
                if (statusCode == HttpStatus.SC_MOVED_PERMANENTLY || 
                		statusCode == HttpStatus.SC_MOVED_TEMPORARILY) {
                	VolleyLog.e("Request at %s has been redirected to %s", request.getOriginUrl(), request.getUrl());
                } else {
                	VolleyLog.e("Unexpected response code %d for %s", statusCode, request.getUrl());
                }
                if (responseContents != null) {
                    networkResponse = new NetworkResponse(statusCode, responseContents,
                            responseHeaders, false, SystemClock.elapsedRealtime() - requestStart);
                    ...
            }
        }
    }
```

在BasicNetwork中，我们可以看到其通过写了一个死循环来保证重复请求的操作直到报错或者返回结果，在其中他会调用mHttpStack.performRequest(request, headers)来获得一个HttpResponse，调用entityToBytes()将entity转换成一个字节数组，并且获取相关参数(header,statusCode .etc)去进一步的构造出一个NetworkResponse，并且返回：

```java
    public NetworkResponse(int statusCode, byte[] data, Map<String, String> headers,
            boolean notModified, long networkTimeMs) {
        this.statusCode = statusCode;
        this.data = data;
        this.headers = headers;
        this.notModified = notModified;
        this.networkTimeMs = networkTimeMs;
    }

    public NetworkResponse(int statusCode, byte[] data, Map<String, String> headers,
            boolean notModified) {
        this(statusCode, data, headers, notModified, 0);
    }

    public NetworkResponse(byte[] data) {
        this(HttpStatus.SC_OK, data, Collections.<String, String>emptyMap(), false, 0);
    }

    public NetworkResponse(byte[] data, Map<String, String> headers) {
        this(HttpStatus.SC_OK, data, headers, false, 0);
    }

    /**Http状态码 */
    public final int statusCode;

    /**网络请求返回的字节数据 */
    public final byte[] data;

    /**返回的请求头 */
    public final Map<String, String> headers;

    /** 是否返回的304的状态码*/
    public final boolean notModified;

    /** 请求耗时. */
    public final long networkTimeMs;
```



------

#### RequestQueue

RequestQueue是一个请求队列，Volley将所有Request请求维护在此请求对列当中，内部有两个对应缓存和网络的请求阻塞队列来对Request进行维护，并且通过不同的Dispatcher(extends Thread)进行进一步的请求结果的处理，主要代码如下：

```java
public class RequestQueue {

    /** 所有请求完成后的回调 */
    public static interface RequestFinishedListener<T> {
        /** Called when a request has finished processing. */
        public void onRequestFinished(Request<T> request);
    }
	...
  	//等待请求的Request，如果一个请求可以进行缓存，则后续的相同CacheKey的请求，将进入此等待队列。
    private final Map<String, Queue<Request<?>>> mWaitingRequests =
            new HashMap<String, Queue<Request<?>>>();
  	//当前请求以及未完成的请求
    private final Set<Request<?>> mCurrentRequests = new HashSet<Request<?>>();
    /** 缓存请求阻塞队列 */
    private final PriorityBlockingQueue<Request<?>> mCacheQueue =
        new PriorityBlockingQueue<Request<?>>();
    /** 网络请求阻塞队列 */
    private final PriorityBlockingQueue<Request<?>> mNetworkQueue =
        new PriorityBlockingQueue<Request<?>>();
  	//网络请求线程池大小
    private static final int DEFAULT_NETWORK_THREAD_POOL_SIZE = 4;
	...
      
  	//网络dispatcher数组
    private NetworkDispatcher[] mDispatchers;

    //缓存dispatcher
    private CacheDispatcher mCacheDispatcher;
	//Request完成listener
    private List<RequestFinishedListener> mFinishedListeners =
            new ArrayList<RequestFinishedListener>();

    public RequestQueue(Cache cache, Network network, int threadPoolSize,
            ResponseDelivery delivery) {
        mCache = cache;
        mNetwork = network;
      	//初始化网络Dispatcher
        mDispatchers = new NetworkDispatcher[threadPoolSize];
        mDelivery = delivery;
    }
    /**
     * start()方法中启动了缓存dispatcher跟网络请求dispatcher，用来处理数据请求，
     * 如果缓存中有数据，则直接在缓存中获取数据，缓存中没有数据则从网络中获取数据。
     */
    public void start() {
        stop();  // Make sure any currently running dispatchers are stopped.
        // 启动缓存请求队列
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        // Create network dispatchers (and corresponding threads) up to the pool size.
      	//创造网络请求队列
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }
	...

    /**
     * 添加新的请求到请求对列当中
     */
    public <T> Request<T> add(Request<T> request) {
        ...
    }

    ...
    <T> void finish(Request<T> request) {
        // Remove from the set of requests currently being processed.
        synchronized (mCurrentRequests) {
            mCurrentRequests.remove(request);
        }
        synchronized (mFinishedListeners) {
          for (RequestFinishedListener<T> listener : mFinishedListeners) {
            listener.onRequestFinished(request);
          }
        }

        if (request.shouldCache()) {
            synchronized (mWaitingRequests) {
                String cacheKey = request.getCacheKey();
                Queue<Request<?>> waitingRequests = mWaitingRequests.remove(cacheKey);
                if (waitingRequests != null) {
                    mCacheQueue.addAll(waitingRequests);
                }
            }
        }
    }

  	....
}
```

RequestQueue是Volley中最核心的类，掌管着所有request的请求和调度的过程，可以说是一个中转器的作用，我们从刚开始的start()方法看起：

```java
/**
     * start()方法中启动了缓存dispatcher跟网络请求dispatcher，用来处理数据请求，
     * 如果缓存中有数据，则直接在缓存中获取数据，缓存中没有数据则从网络中获取数据。
     */
    public void start() {
        stop();  // Make sure any currently running dispatchers are stopped.
        // 启动缓存请求队列
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

      	//创造网络请求队列
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }

```

在调用start()时候，主要做了两件事情:

- 调用stop()终止所有的dispatcher
- 初始化所有的dispatcher，并且start(),dispatcher都是继承于Thread的类

一般我们进行请求时候，都是使用的add(Request)方法来进行，所以接下来就查看一下这个方法的实现过程：

```java
 /**
     * 添加新的请求到请求对列当中
     */
    public <T> Request<T> add(Request<T> request) {
        // 设置对应关联
        request.setRequestQueue(this);
      	//添加到CurrentRequests中
        synchronized (mCurrentRequests) {
            mCurrentRequests.add(request);
        }

        // 按添加到请求队列的顺序请求数据
        request.setSequence(getSequenceNumber());
      	//添加标志
        request.addMarker("add-to-queue");
      	//当前request是否有缓存过，没有直接加入到网络请求队列中
        if (!request.shouldCache()) {
            mNetworkQueue.add(request);
            return request;
        }

        synchronized (mWaitingRequests) {
          	//得到缓存key(该key为medthod+url的单一key)
            String cacheKey = request.getCacheKey();
          	//是否包含该缓存key
            if (mWaitingRequests.containsKey(cacheKey)) {
                // There is already a request in flight. Queue up.
                Queue<Request<?>> stagedRequests = mWaitingRequests.get(cacheKey);
                if (stagedRequests == null) {
                    stagedRequests = new LinkedList<Request<?>>();
                }
                stagedRequests.add(request);
                mWaitingRequests.put(cacheKey, stagedRequests);
                if (VolleyLog.DEBUG) {
                    VolleyLog.v("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
                }
            } else {
                // Insert 'null' queue for this cacheKey, indicating there is now a request in
                // flight.
                mWaitingRequests.put(cacheKey, null);
                mCacheQueue.add(request);
            }
            return request;
        }
    }
```

在add()方法中主要的过程如图所示:

{% asset_img reuqestqueue_add.png reuqestqueue.add() %}

主要实现思路如下：

1. 添加到mCurrentRequests中，mCurrentRequests当前请求以及未完成的请求的set，所有通过add()进来的都会加入到此set的当中。
2. 通过request.shouldCache()来判断是否需要缓存request，如果不需要则直接加入到网络请求队列当中，return；如果需要缓存当前request则判断mWaitingRequests是否包含了该request的CacheKey的setKey，如果有，则加入相同的队列当中，则put到mWaitingRequests当中，如果没有包含了该request的CacheKey的setKey，置空与当前request的相同的queue(mWaitingRequests.put(cacheKey, null))，然后把request加入到缓存请求对列当中。

add()方法中主要是为了找到request的存放地，是网络请求队列还是缓存请求队列，在判断是否加入到缓存请求队列时维护了一个mWaitingRequests集合来管理相同的请求，如果cacheKey相同，则不会加入到任何请对队列当中，然后加入到mWaitingRequests存有相同SetKey的SetValue中，挂起。



-------

<h4>Dispatcher</h4>

返回RequestQueue.start()中，这里会将CacheDispatcher和NetworkDispatchers启动，两种Dispatcher共同继承于Thread来处理延时操作--网络请求和缓存请求的过程，这里只详细讲述CacheDispatcher的run()过程，NetworkDispatchers只以简述的形式讲述。

CacheDispatcher在run方法中写死一个无限循环，可以类比成handler机制中的Looper.loop()的设计思路，不断的轮训缓存队列(mCacheQueue)，从中取出request将其传递给mDelivery来进行处理。

```java
@Override
    public void run() {
		//设置线程优先级
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

        // 初始化缓存相关参数
        mCache.initialize();

        while (true) {
            try {
                //从mCacheQueue取缓存
                final Request<?> request = mCacheQueue.take();
              	//添加缓存过的标记
                request.addMarker("cache-queue-take");

                // 当前request是否取消了
                if (request.isCanceled()) {
                    request.finish("cache-discard-canceled");
                    continue;
                }

                // Attempt to retrieve this item from cache.
                Cache.Entry entry = mCache.get(request.getCacheKey());
                if (entry == null) {
                    request.addMarker("cache-miss");
                    // Cache miss; send off to the network dispatcher.
                    mNetworkQueue.put(request);
                    continue;
                }

                // If it is completely expired, just send it to the network.
                if (entry.isExpired()) {
                    request.addMarker("cache-hit-expired");
                    request.setCacheEntry(entry);
                    mNetworkQueue.put(request);
                    continue;
                }

                // We have a cache hit; parse its data for delivery back to the request.
                request.addMarker("cache-hit");
                Response<?> response = request.parseNetworkResponse(
                        new NetworkResponse(entry.data, entry.responseHeaders));
                request.addMarker("cache-hit-parsed");
				//缓存数据是否需要更新
                if (!entry.refreshNeeded()) {
                    // Completely unexpired cache hit. Just deliver the response.
                    mDelivery.postResponse(request, response);
                } else {
                    // Soft-expired cache hit. We can deliver the cached response,
                    // but we need to also send the request to the network for
                    // refreshing.
                    request.addMarker("cache-hit-refresh-needed");
                    request.setCacheEntry(entry);

                    // Mark the response as intermediate.
                    response.intermediate = true;

                    // Post the intermediate response back to the user and have
                    // the delivery then forward the request along to the network.
                    mDelivery.postResponse(request, response, new Runnable() {
                        @Override
                        public void run() {
                            try {
                                mNetworkQueue.put(request);
                            } catch (InterruptedException e) {
                                // Not much we can do about this.
                            }
                        }
                    });
                }

            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
                continue;
            }
        }
    }
```

run方法内循环的过程中每次从mCacheQueue拿出一个request请求，对其添加已经访问过的marker，如果在期间request被取消了，最终会调用RequestQueue.finish()方法从mCurrentRequests中remove掉，一次循环完成；如果request没被取消则从缓存中获取数据，并且判断是否为空，如果为空则加入网络队列中重新请求，一次循环完成；如果不为空判断是否过期，过期了则加入网络队列中重新请求，一次循环完成；如果不过期判断是否需要更新缓存，需要则加入网络队列中重新请求，否则调用mDelivery.postResponse()完成数据传递的过程，执行图如下：

{% asset_img CacheDispatcher_run.png CacheDispatcher.run() %}

NetworkDispatcher也继承自Thread接口，run方法实现代码如下：

```java
@Override
    public void run() {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        while (true) {
            long startTimeMs = SystemClock.elapsedRealtime();
            Request<?> request;
            try {
                // Take a request from the queue.
                request = mQueue.take();
            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
                continue;
            }

            try {
                request.addMarker("network-queue-take");

                // If the request was cancelled already, do not perform the
                // network request.
                if (request.isCanceled()) {
                    request.finish("network-discard-cancelled");
                    continue;
                }

                addTrafficStatsTag(request);

                //获取到请求结果
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                request.addMarker("network-http-complete");

                // If the server returned 304 AND we delivered a response already,
                // we're done -- don't deliver a second identical response.
                if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                    request.finish("not-modified");
                    continue;
                }

                // Parse the response here on the worker thread.
                Response<?> response = request.parseNetworkResponse(networkResponse);
                request.addMarker("network-parse-complete");
              	//是否缓存结果
                if (request.shouldCache() && response.cacheEntry != null) {
                    mCache.put(request.getCacheKey(), response.cacheEntry);
                    request.addMarker("network-cache-written");
                }

                // Post the response back.
                request.markDelivered();
              	//交付给mDelivery
                mDelivery.postResponse(request, response);
            } catch (VolleyError volleyError) {
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e) {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
                VolleyError volleyError = new VolleyError(e);
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                mDelivery.postError(request, volleyError);
            }
        }
    }
```

在NetworkDispatcher中主要通过Network接口performRequest()方法获得一个NetworkResponse,进而在转换成Delivery.postResponse()可接受的Request对象进行传递，其中会判断是否需要缓存该request以及相关error的捕获并且传递给Delivery.postError()。

-----

<h3>ResponseDelivery</h3>

无论是CacheDispatcher还是NetworkDispatcher，最终的结果都是交付由ResponseDelivery接口实现类来进行实现:

```java
public interface ResponseDelivery {
    /**
     * Parses a response from the network or cache and delivers it.
     */
    public void postResponse(Request<?> request, Response<?> response);

    /**
     * 处理从缓存和网络上获取的数据
     */
    public void postResponse(Request<?> request, Response<?> response, Runnable runnable);

    /**
     *处理错误的情况
     */
    public void postError(Request<?> request, VolleyError error);
}
```

其具体实现类为ExecutorDelivery中，其代码如下：

```java
/**
 * 将请求结果传递给回调接口
 */
public class ExecutorDelivery implements ResponseDelivery {
    /** Used for posting responses, typically to the main thread. */
    private final Executor mResponsePoster;

    /**
     * 传入Handler的原因，目的是为了与主线程进行交互
     * @param handler {@link Handler} to post responses on
     */
    public ExecutorDelivery(final Handler handler) {
        // Make an Executor that just wraps the handler.
        mResponsePoster = new Executor() {
            @Override
            public void execute(Runnable command) {
                handler.post(command);
            }
        };
    }
	...
      
    @Override
    public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
      	//request传递标记
        request.markDelivered();
        request.addMarker("post-response");
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
    }

    @Override
    public void postError(Request<?> request, VolleyError error) {
        request.addMarker("post-error");
        Response<?> response = Response.error(error);
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, null));
    }

    /**
     * A Runnable used for delivering network responses to a listener on the
     * main thread.
     */
    @SuppressWarnings("rawtypes")
    private class ResponseDeliveryRunnable implements Runnable {
        private final Request mRequest;
        private final Response mResponse;
        private final Runnable mRunnable;

        public ResponseDeliveryRunnable(Request request, Response response, Runnable runnable) {
            mRequest = request;
            mResponse = response;
            mRunnable = runnable;
        }

        @SuppressWarnings("unchecked")
        @Override
        public void run() {
            // If this request has canceled, finish it and don't deliver.
            if (mRequest.isCanceled()) {
                mRequest.finish("canceled-at-delivery");
                return;
            }

            // 传递请求的结果，对应调用相应的回调接口，看mRequest对应实现类
          	//如果是StringRequest，则回调对应接口。
            if (mResponse.isSuccess()) {
                mRequest.deliverResponse(mResponse.result);
            } else {
                mRequest.deliverError(mResponse.error);
            }

            // If this is an intermediate response, add a marker, otherwise we're done
            // and the request can be finished.
            if (mResponse.intermediate) {
                mRequest.addMarker("intermediate-response");
            } else {
                mRequest.finish("done");
            }

            // If we have been provided a post-delivery runnable, run it.
            if (mRunnable != null) {
                mRunnable.run();
            }
       }
    }
}
```

从Dispatcher那边获取到request和response最终传递到内部类ResponseDeliveryRunnable中进行处理，如果request是被finish的则丢弃传递，否则调用Request的对应的deliverResponse或者deliverError方法。这里要注意以下，该方法中传递了handler变量进来，这个变量是主线程的handler，也就保证了request的回调从工作线程切换回了主线程，其初始化代码如下：

```java
public RequestQueue(Cache cache, Network network, int threadPoolSize) {
        this(cache, network, threadPoolSize,
                new ExecutorDelivery(new Handler(Looper.getMainLooper())));
    }
```



-------

<h4>Request</h4>

Request类是一个抽象方法，其子类实现主要有：StringRequest，JsonRequest，ImageRequest .etc ，为了简单查看后续步骤，我们拿StringRequest来查看：

```java
 /**
     * Delivers error message to the ErrorListener that the Request was
     * initialized with.
     *
     * @param error Error details
     */
    public void deliverError(VolleyError error) {
        if (mErrorListener != null) {
            mErrorListener.onErrorResponse(error);
        }
    }
  @Override
    protected void deliverResponse(String response) {
        if (mListener != null) {
            mListener.onResponse(response);
        }
    }
```

到这里就执行到了最终的回调，所有的过程也就完成了。

------

<h4>个人总结</h4>

个人认为volley的精髓在于面向接口编程，具有非常广阔的拓展性，开发者完全可以自主写一套自己的volley逻辑，充分解耦了各个模块，使用组合的方式进行编程，是我们学习设计代码的一个非常好的库吧。



附上Volley过程图：
{% asset_img volley_flowchart.png volley流程图 %}
