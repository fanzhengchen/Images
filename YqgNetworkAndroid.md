# 一起逛HTTP请求部分结构示意图

### 网络模块结构示意图 包含 Volley 部分
![image](https://raw.githubusercontent.com/fanzhengchen/Images/master/NetworkArchitecture.png)



### 网络部分结构
###### 1. 业务逻辑部分 Presenter 与 YqgProtocol, 我们在Presenter处理业务, YqgProtocol包括对涉及到具体业务逻辑的请求封装和对请求的Response的最终处理
###### 2. YqgProtocolHall 里包含了请求发起的接口, 并构造好了Volley 里的 RequestQueue,RequestQueue是Volley 框架中进行网络请求的核心。
###### 3. RequestQueue, RequestQueue首先加入Request, 然后通过NetworkDispatcher 这个线程进行请求获取NetworkResponse 然后由ResponseDelivery这个接口执行 deliverResponse 或者 deliverError方法，通过一条很长的处理链条最终调用我们在YqgProtocol里传入的VolleyCallback。


### 核心部分
1. RequestQueue, 这是Volley的核心，包含了请求的发起和回调，而且还进行了HTTP缓存和线程池的维护
2. 从ResponseDelivery开始的回调链，这里包含了对各种网络错误的处理, 目前基于于业务需要设计的BasicProtocol与Volley感觉有着一定的耦合度。

##### RequestQueue
###### 关键方法主要为 addRequest、start、finish
###### 1. addRequest 首先将Request 加入 mCurrentRequests，然后执行缓存策略, 保存一个HashSet，若相同请求已经存在于HashSet, 则进行缓存， 缓存队列由 mCacheDispatcher 线程维护。
```Java
     * Adds a Request to the dispatch queue.
     * @param request The request to service
     * @return The passed-in request
     */
    public <T> Request<T> add(Request<T> request) {
        // Tag the request as belonging to this queue and add it to the set of current requests.
        request.setRequestQueue(this);
        synchronized (mCurrentRequests) {
            mCurrentRequests.add(request);
        }

        // Process requests in the order they are added.
        request.setSequence(getSequenceNumber());
        request.addMarker("add-to-queue");

        // If the request is uncacheable, skip the cache queue and go straight to the network.
        if (!request.shouldCache()) {
            mNetworkQueue.add(request);
            return request;
        }

        // Insert request into stage if there's already a request with the same cache key in flight.
        synchronized (mWaitingRequests) {
            String cacheKey = request.getCacheKey();
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
##### 2. start 方法负责启动所有的网络请求线程和缓存维护线程
```Java
/**
     * Starts the dispatchers in this queue.
     */
    public void start() {
        stop();  // Make sure any currently running dispatchers are stopped.
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        // Create network dispatchers (and corresponding threads) up to the pool size.
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }
```
##### 3. finish 方法负责终止所有正在执行的请求，没有执行的请求则全部加入缓存队列
```Java
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
                    if (VolleyLog.DEBUG) {
                        VolleyLog.v("Releasing %d waiting requests for cacheKey=%s.",
                                waitingRequests.size(), cacheKey);
                    }
                    // Process all queued up requests. They won't be considered as in flight, but
                    // that's not a problem as the cache has been primed by 'request'.
                    mCacheQueue.addAll(waitingRequests);
                }
            }
        }
    }
```

##### 回调链
###### 1. 在NetworkDispatcher 中，run 方法中执行请求，获得NetworkResponse, 由ResponseDelivery接口进行回调处理
```Java
try {
    request.addMarker("network-queue-take");
    if(request.isCanceled()) {
        request.finish("network-discard-cancelled");
    } else {
        this.addTrafficStatsTag(request);
        NetworkResponse e = this.mNetwork.performRequest(request);
        request.addMarker("network-http-complete");
        if(e.notModified && request.hasHadResponseDelivered()) {
            request.finish("not-modified");
        } else {
            Response volleyError1 = request.parseNetworkResponse(e);
            request.addMarker("network-parse-complete");
            if(request.shouldCache() && volleyError1.cacheEntry != null) {
                this.mCache.put(request.getCacheKey(), volleyError1.cacheEntry);
                request.addMarker("network-cache-written");
            }

            request.markDelivered();
            this.mDelivery.postResponse(request, volleyError1);
        }
    }
} catch (VolleyError var7) {
    var7.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
    this.parseAndDeliverNetworkError(request, var7);
} catch (Exception var8) {
    VolleyLog.e(var8, "Unhandled exception %s", new Object[]{var8.toString()});
    VolleyError volleyError = new VolleyError(var8);
    volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
    this.mDelivery.postError(request, volleyError);
}     
```
###### 2. Delivery postResponse 执行Request 中的deliverResponse方法, 执行 BasicRequest 构造方法中的ResponseListner 和 ErrorListner， 进而执行 BasicProtocol 中的onHttpSuccess 和 onHttpError, 最后执行 notifySuccess 和 notifyError 方法， 执行我们定义的Callback放法修改UI, 然后在notifySuccess或者notifyError中移除对应的请求。

### 接下来需要思考和改进的方向
###### 1. 考虑使用 Retrofit 替代当前的 Volley 提高效率， 并且尽量解耦, 目前代码结构仍有一定耦合度
###### 2. 考虑接入 RxJava， 将数据以流的形式进行处理简化回调， 增强可读性。
