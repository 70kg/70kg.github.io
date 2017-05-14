---
title: Volley源码解析
date: 2016-07-09 23:26:17
tags: Android
---
&emsp;&emsp;这篇博客想写挺久的了，因为之前接触到网络请求的第一个流行的库就是`Volley`，在使用的时候也感觉到它的功能和拓展性的强大，但是一直没有去探究他内部的流程，同时可以发现网上对`Volley`这个框架设计的评价都非常好，所以去了解内部的实现还是很有必要的。下面开始吧。<!--more-->

&emsp;&emsp;先从最开始怎么去使用说起，最常见的使用方式就是这样的：

```
RequestQueue queue = Volley.newRequestQueue(this);

        StringRequest request = new StringRequest("http://70kg.info", new Response.Listener<String>() {

            @Override
            public void onResponse(String response) {

            }
        }, new Response.ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError error) {

            }
        });

        queue.add(request);
```
先去创建一个`RequestQueue`，然后在去创建一个`request`，最后加入到队列里面就可以了。使用也是相当的方便清晰。那就从`RequestQueue`入手，看看这个队列是什么样的。

###RequestQueue
这个类是在`toolbox`包里面，都是一下已经给我们实现一些功能的类，方便我们去使用。`Volley.newRequestQueue(this);`最终走到了这里：

```
public static RequestQueue newRequestQueue(Context context, HttpStack stack) {
        //缓存目录
        File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);
        String userAgent = "volley/0";
        try {
            String packageName = context.getPackageName();
            PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
            userAgent = packageName + "/" + info.versionCode;
        } catch (NameNotFoundException e) {
        }

        if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        Network network = new BasicNetwork(stack);

        RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        queue.start();

        return queue;
    }
```
这里可以看到还有个`userAgent`，在使用使用`HttpClientStack`也就是API<9,使用`AndroidHttpClient`的时候使用，具体是什么会在下面的参考中给出，和本文关系不是很大，只要明白在>=9的时候使用了`HurlStack`，内部是`HttpURLConnection`就差不多了。下面还有个`Network`，是根据传进去的`stack`来选择不同的处理网络方式，后面会说。最重要的就是下面两行，真正的`new RequestQueue`，然后`star`,进去看看：

经过各种重载构造函数，走到了这个：

```
 public RequestQueue(Cache cache, Network network, int threadPoolSize,
            ResponseDelivery delivery) {
        //默认缓存
        mCache = cache;
        //处理网络的
        mNetwork = network;
        //一个NetworkDispatcher数组，默认大小4
        mDispatchers = new NetworkDispatcher[threadPoolSize];
        //一个ExecutorDelivery对象
        mDelivery = delivery;
    }
```
然后就是去看`star`方法了：

```
public void start() {
        stop();  
        //一个缓存调度线程
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();
        //默认4个网络调度线程
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }
```
所以当我们新建了一个`RequestQueue`之后，其实是启动了5个线程在跑。接下来就是` queue.add(request);`的时候了。

```
public Request add(Request request) {
        //相当于设个tag为当前的queue
        request.setRequestQueue(this);
        synchronized (mCurrentRequests) {
        //private final Set<Request<?>> mCurrentRequests = new HashSet<Request<?>>();正在进行，尚未完成请求的集合
            mCurrentRequests.add(request);
        }
       //序列号 可以理解为加入的顺序
        request.setSequence(getSequenceNumber());
        //添加个标记
        request.addMarker("add-to-queue");

        //不缓存，就直接加入网络队列去请求网络
        if (!request.shouldCache()) {
            mNetworkQueue.add(request);
            return request;
        }
        synchronized (mWaitingRequests) {
            String cacheKey = request.getCacheKey();
            //维护了一个等待请求的集合，如果一个请求正在被处理并且可以被缓存，后续的相同 url 的请求，将进入此等待队列。
            if (mWaitingRequests.containsKey(cacheKey)) {
            //前面有相同的了 等待
                Queue<Request> stagedRequests = mWaitingRequests.get(cacheKey);
                if (stagedRequests == null) {
                    stagedRequests = new LinkedList<Request>();
                }
                stagedRequests.add(request);
                mWaitingRequests.put(cacheKey, stagedRequests);
            } else {
                //否则加入缓存队列
                mWaitingRequests.put(cacheKey, null);
                mCacheQueue.add(request);
            }
            return request;
        }
    }
```

###CacheDispatcher
Volley默认都是可以缓存的，而处理缓存的是`CacheDispatcher`，这是一个线程，那就可以去`Run`方法看看：

```
@Override
    public void run() {
        //线程优先级
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        //初始化缓存
        mCache.initialize();
         //开启无限循环
        while (true) {
            try {
                //从缓存队列中获取第一个
                final Request request = mCacheQueue.take();
                request.addMarker("cache-queue-take");
               //取消了  不去处理
                if (request.isCanceled()) {
             request.finish("cache-discard-canceled");
                    continue;
                }

               //从缓存中检所
                Cache.Entry entry = mCache.get(request.getCacheKey());
                if (entry == null) {
                    request.addMarker("cache-miss");
                    //没找到  加入网络队列去请求
                    mNetworkQueue.put(request);
                    continue;
                }
                //缓存过期了 也是网络请求
                if (entry.isExpired()) {
                    request.addMarker("cache-hit-expired");
                    request.setCacheEntry(entry);
                    mNetworkQueue.put(request);
                    continue;
                }

                //获取到正常可用的缓存的request
                request.addMarker("cache-hit");
                //转换成一个NetworkResponse
                Response<?> response = request.parseNetworkResponse(
                        new NetworkResponse(entry.data, entry.responseHeaders));
                request.addMarker("cache-hit-parsed");
                if (!entry.refreshNeeded()) {
                    //mDelivery使用handler分发到主线程
                    mDelivery.postResponse(request, response);
                } else {
                    //检验新鲜度 request.addMarker("cache-hit-refresh-needed");
                    request.setCacheEntry(entry);
                    response.intermediate = true;
                   //分发  再去加入网络队列检验新鲜度
                    mDelivery.postResponse(request, response, new Runnable() {
                        @Override
                        public void run() {
                            try {
                                mNetworkQueue.put(request);
                            } catch (InterruptedException e) {
                            }
                        }
                    });
                }

            } catch (InterruptedException e) {
        
                if (mQuit) {
                    return;
                }
                continue;
            }
        }
    }
```
整个过程差不多都注释了，再来个大招，非常棒的流程图：

![](https://raw.githubusercontent.com/android-cn/android-open-project-analysis/master/tool-lib/network/volley/image/CacheDispatcher-run-flow-chart.png)

###CacheDispatcher
说完缓存分发，还有网络分发，还是看run方法：

```
@Override
    public void run() {
       //线程优先级
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        Request request;
        //循环取
        while (true) {
            try {
              //取一个
                request = mQueue.take();
            } catch (InterruptedException e) {
                if (mQuit) {
                    return;
                }
                continue;
            }
            try {
                request.addMarker("network-queue-take");
                if (request.isCanceled()) {                  request.finish("network-discard-cancelled");
                    continue;
                }

                // Tag the request (if API >= 14)
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                    TrafficStats.setThreadStatsTag(request.getTrafficStatsTag());
                }

                // 执行网络请求
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                request.addMarker("network-http-complete");

                //返回304并且已经处理过这个，跳过
                if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                    request.finish("not-modified");
                    continue;
                }
               //解析成Response
                Response<?> response = request.parseNetworkResponse(networkResponse);
                request.addMarker("network-parse-complete");
                //请求到之后缓存
                if (request.shouldCache() && response.cacheEntry != null) {
                    mCache.put(request.getCacheKey(), response.cacheEntry);
               request.addMarker("network-cache-written");
                }
                request.markDelivered();
                //分发到主线程
                mDelivery.postResponse(request, response);
            } catch (VolleyError volleyError) {
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e) {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
                mDelivery.postError(request, new VolleyError(e));
            }
        }
    }
```
整个流程和缓存差不多，再来一个图：
![](https://raw.githubusercontent.com/android-cn/android-open-project-analysis/master/tool-lib/network/volley/image/NetworkDispatcher-run-flow-chart.png)


###Network
在上面的网络分发中看到了主要就是`mNetwork.performRequest(request);`这句话执行了真正的网络请求操作，而`Network`是一个接口，实现类是在创建`RequestQueue`时候创建的`BasicNetwork`，看看这个里面的实现：

```
public class BasicNetwork implements Network {
       //...省略看不懂的网络操作
         @Override
    public NetworkResponse performRequest(Request<?> request) throws VolleyError {
     //...
     //这就是前面根据版本不同选择不同的网络处理方式
     httpResponse = mHttpStack.performRequest(request, headers);
    
    //....
    //组装返回值
     networkResponse = new NetworkResponse(statusCode, responseContents,responseHeaders, false);
    
    }
       
       //省略
}
```
到现在请求的结果也回来了，那么具体是怎么去分发`Response`的呢？主要是这句话`mDelivery.postResponse(request, response);`
而`ResponseDelivery`是一个接口，实现类是创建`RequestQueue`的时候创建的`public class ExecutorDelivery implements ResponseDelivery`，看一下它的`postResponse`方法，主要就是` mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, null));`可以理解为向主线程发送了一个`Runnable`而在这个`Runnable`里面呢：

```
  if (mResponse.isSuccess()) {
                mRequest.deliverResponse(mResponse.result);
            } else {
                mRequest.deliverError(mResponse.error);
            }
```
第一个方法是不是就比较熟悉了，就是自定义`Request`的时候要重写的两个方法之一，一般就直接` mListener.onResponse(response);`就可以了，而错误的分发`Request`基类已经实现完了。
到这，基本的过程差不多了，还有重试策略和缓存的具体没写，大概就这样吧。

参考：

[详细解读Volley（五）—— 通过源码来分析业务流程](http://www.cnblogs.com/tianzhijiexian/p/4264468.html)

[Volley 源码解析](http://a.codekk.com/detail/Android/grumoon/Volley%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90)

[Volley源码分析](http://bxbxbai.github.io/2014/12/24/read-volley-source-code/)









