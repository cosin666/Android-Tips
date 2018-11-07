## Glide基础 ##

**Glide**是一个快速高效的Android图片加载库，注重于平滑的滚动。

### 特点 ###

- 提供了极易用的API

- 性能
	- 图片解码速度
	- 解码时掉帧
	- 自动 downsampling 和 缓存，减小存储开销和解码次数
	- 资源复用，减小垃圾回收和碎片问题
	- 生命周期绑定
	
- 其他
    - 支持gif图
    - 本地视频解码

### 应用场景 ###

列表中加载大量网络图片

### 整体设计 ###

![image](https://raw.githubusercontent.com/newsky2012/Android-Tips/master/Glide%E7%AE%80%E6%9E%90/imgs/2.jpg)

1. Glide 收到加载及显示资源的任务，创建**Request**并将它交给**RequestManager**（任务管理器），
2. Request启动**Engine**（数据获取引擎）去数据源获取资源，
3. 获取到后Transformation（图片处理） 处理后交给**Target**（目标）。

### 使用方法 ###

    Glide.with(fragment)                    // 绑定生命周期
        .load(url)                          // 通过url加载图片
        .placeholder(R.drawable.default)    // 显示默认图
        .error(R.drawable.error)            // 显示加载错误图
        .crossFade()                        // 设置动画
        .override(600, 200)                 // 设置图片大小
        .into(imageView);                   // 将图片显示到imageView

### 源码分析 ###

- with方法

with方法实现了生命周期绑定，首先来看with方法，可以接收多种参数

    public class Glide {
    
        ...
    
        public static RequestManager with(Context context) {
            RequestManagerRetriever retriever = RequestManagerRetriever.get();
            return retriever.get(context);
        }
    
        public static RequestManager with(Activity activity) {
            RequestManagerRetriever retriever = RequestManagerRetriever.get();
            return retriever.get(activity);
        }
    
        public static RequestManager with(FragmentActivity activity) {
            RequestManagerRetriever retriever = RequestManagerRetriever.get();
            return retriever.get(activity);
        }
    
        @TargetApi(Build.VERSION_CODES.HONEYCOMB)
        public static RequestManager with(android.app.Fragment fragment) {
            RequestManagerRetriever retriever = RequestManagerRetriever.get();
            return retriever.get(fragment);
        }
    
        public static RequestManager with(Fragment fragment) {
            RequestManagerRetriever retriever = RequestManagerRetriever.get();
            return retriever.get(fragment);
        }
    }

with方法，创建单例对象RequestManagerRetriever，通过get方法获取RequestManager

    public class RequestManagerRetriever implements Handler.Callback {
    
        private static final RequestManagerRetriever INSTANCE = new RequestManagerRetriever();
    
        private volatile RequestManager applicationManager;
    
        ...
    
        /**
         * Retrieves and returns the RequestManagerRetriever singleton.
         */
        public static RequestManagerRetriever get() {
            return INSTANCE;
        }
    
        private RequestManager getApplicationManager(Context context) {
            // Either an application context or we're on a background thread.
            if (applicationManager == null) {
                synchronized (this) {
                    if (applicationManager == null) {
                        // Normally pause/resume is taken care of by the fragment we add to the fragment or activity.
                        // However, in this case since the manager attached to the application will not receive lifecycle
                        // events, we must force the manager to start resumed using ApplicationLifecycle.
                        applicationManager = new RequestManager(context.getApplicationContext(),
                                new ApplicationLifecycle(), new EmptyRequestManagerTreeNode());
                    }
                }
            }
            return applicationManager;
        }
    
        public RequestManager get(Context context) {                        // 最终对应到FragmentActivity、Activity和Application
            if (context == null) {
                throw new IllegalArgumentException("You cannot start a load on a null Context");
            } else if (Util.isOnMainThread() && !(context instanceof Application)) {
                if (context instanceof FragmentActivity) {
                    return get((FragmentActivity) context);
                } else if (context instanceof Activity) {
                    return get((Activity) context);
                } else if (context instanceof ContextWrapper) {
                    return get(((ContextWrapper) context).getBaseContext());
                }
            }
            return getApplicationManager(context);
        }
    
        public RequestManager get(FragmentActivity activity) {
            if (Util.isOnBackgroundThread()) {
                return get(activity.getApplicationContext());
            } else {
                assertNotDestroyed(activity);
                FragmentManager fm = activity.getSupportFragmentManager();
                return supportFragmentGet(activity, fm);                     // supportFragmentGet
            }
        }
    
        public RequestManager get(Fragment fragment) {
            if (fragment.getActivity() == null) {
                throw new IllegalArgumentException("You cannot start a load on a fragment before it is attached");
            }
            if (Util.isOnBackgroundThread()) {
                return get(fragment.getActivity().getApplicationContext());
            } else {
                FragmentManager fm = fragment.getChildFragmentManager();
                return supportFragmentGet(fragment.getActivity(), fm);        // supportFragmentGet
            }
        }
    
        @TargetApi(Build.VERSION_CODES.JELLY_BEAN_MR1)
        private static void assertNotDestroyed(Activity activity) {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1 && activity.isDestroyed()) {
                throw new IllegalArgumentException("You cannot start a load for a destroyed activity");
            }
        }

        @TargetApi(Build.VERSION_CODES.HONEYCOMB)
        public RequestManager get(Activity activity) {
            if (Util.isOnBackgroundThread() || Build.VERSION.SDK_INT < Build.VERSION_CODES.HONEYCOMB) {
                return get(activity.getApplicationContext());
            } else {
                assertNotDestroyed(activity);
                android.app.FragmentManager fm = activity.getFragmentManager();
                return fragmentGet(activity, fm);                           // fragmentGet
            }
        }
    
        @TargetApi(Build.VERSION_CODES.JELLY_BEAN_MR1)
        public RequestManager get(android.app.Fragment fragment) {          // fragmentGet
            if (fragment.getActivity() == null) {
                throw new IllegalArgumentException("You cannot start a load on a fragment before it is attached");
            }
            if (Util.isOnBackgroundThread() || Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR1) {
                return get(fragment.getActivity().getApplicationContext());
            } else {
                android.app.FragmentManager fm = fragment.getChildFragmentManager();
                return fragmentGet(fragment.getActivity(), fm);
            }
        }
    
        @TargetApi(Build.VERSION_CODES.JELLY_BEAN_MR1)
        RequestManagerFragment getRequestManagerFragment(final android.app.FragmentManager fm) {
            RequestManagerFragment current = (RequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
            if (current == null) {
                current = pendingRequestManagerFragments.get(fm);
                if (current == null) {
                    current = new RequestManagerFragment();
                    pendingRequestManagerFragments.put(fm, current);
                    fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
                    handler.obtainMessage(ID_REMOVE_FRAGMENT_MANAGER, fm).sendToTarget();
                }
            }
            return current;
        }
    
        @TargetApi(Build.VERSION_CODES.HONEYCOMB)
        RequestManager fragmentGet(Context context, android.app.FragmentManager fm) {
            RequestManagerFragment current = getRequestManagerFragment(fm);         // 获取fragment
            RequestManager requestManager = current.getRequestManager();
            if (requestManager == null) {                           // 创建RequestManager，将fragment的生命周期管理对象传入
                requestManager = new RequestManager(context, current.getLifecycle(), current.getRequestManagerTreeNode());
                current.setRequestManager(requestManager);          // 绑定生命周期                       
            }
            return requestManager;
        }
    
        SupportRequestManagerFragment getSupportRequestManagerFragment(final FragmentManager fm) {
            SupportRequestManagerFragment current = (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
            if (current == null) {
                current = pendingSupportRequestManagerFragments.get(fm);
                if (current == null) {
                    current = new SupportRequestManagerFragment();
                    pendingSupportRequestManagerFragments.put(fm, current);
                    fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
                    handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
                }
            }
            return current;
        }
    
        RequestManager supportFragmentGet(Context context, FragmentManager fm) {
            SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm);
            RequestManager requestManager = current.getRequestManager();
            if (requestManager == null) {
                requestManager = new RequestManager(context, current.getLifecycle(), current.getRequestManagerTreeNode());
                current.setRequestManager(requestManager);
            }
            return requestManager;
        }
    
        ...
    }


实际上Glide是创建了一个不可见的fragment，将图片加载请求绑定到这个不可见的fragment上，接下来看fragment中到底干了什么

    public class RequestManagerFragment extends Fragment {
        private final ActivityFragmentLifecycle lifecycle;
        private final RequestManagerTreeNode requestManagerTreeNode = new FragmentRequestManagerTreeNode();
        private RequestManager requestManager;
        private final HashSet<RequestManagerFragment> childRequestManagerFragments
            = new HashSet<RequestManagerFragment>();
        private RequestManagerFragment rootRequestManagerFragment;
    
        public RequestManagerFragment() {
            this(new ActivityFragmentLifecycle());
        }
    
        ...
    
        @Override
        public void onStart() {
            super.onStart();
            lifecycle.onStart();
        }
    
        @Override
        public void onStop() {
            super.onStop();
            lifecycle.onStop();
        }
    
        @Override
        public void onDestroy() {
            super.onDestroy();
            lifecycle.onDestroy();
        }
    
        @Override
        public void onTrimMemory(int level) {
            // If an activity is re-created, onTrimMemory may be called before a manager is ever set.
            // See #329.
            if (requestManager != null) {
                requestManager.onTrimMemory(level);
            }
        }
    
        @Override
        public void onLowMemory() {
            // If an activity is re-created, onLowMemory may be called before a manager is ever set.
            // See #329.
            if (requestManager != null) {
                requestManager.onLowMemory();
            }
        }
    
        ...
    
    }
    
RequestManager实现了LifecycleListener接口，在初始化的时候会将自己注册到lifecycle的监听中

    public class RequestManager implements LifecycleListener {
        private final Context context;
        private final Lifecycle lifecycle;
        private final RequestManagerTreeNode treeNode;
        private final RequestTracker requestTracker;
        
        ...
    
        /**
         * Lifecycle callback that registers for connectivity events (if the android.permission.ACCESS_NETWORK_STATE
         * permission is present) and restarts failed or paused requests.
         */
        @Override
        public void onStart() {
            // onStart might not be called because this object may be created after the fragment/activity's onStart method.
            resumeRequests();           // 重新开启执行一些未成功的请求
        }
    
        /**
         * Lifecycle callback that unregisters for connectivity events (if the android.permission.ACCESS_NETWORK_STATE
         * permission is present) and pauses in progress loads.
         */
        @Override
        public void onStop() {
            pauseRequests();            // 停止正在执行的任务
        }
    
        /**
         * Lifecycle callback that cancels all in progress requests and clears and recycles resources for all completed
         * requests.
         */
        @Override
        public void onDestroy() {
            requestTracker.clearRequests(); // 移除所有请求
        }
        
        ...
    
    }

一张图总结
![avatar](\imgs\2.jpg)

- load方法

load方法中，主要是生成一个builder，将杂七杂八的一些参数配配置信息传进去

    public class RequestManager implements LifecycleListener {
    
        ...
    
        /**
         * Returns a request builder to load the given {@link String}.
         * signature.
         *
         * @see #fromString()
         * @see #load(Object)
         *
         * @param string A file path, or a uri or url handled by {@link com.bumptech.glide.load.model.UriLoader}.
         */
        public DrawableTypeRequest<String> load(String string) {
            return (DrawableTypeRequest<String>) fromString().load(string);
        }
    
        /**
         * Returns a request builder that loads data from {@link String}s using an empty signature.
         *
         * <p>
         *     Note - this method caches data using only the given String as the cache key. If the data is a Uri outside of
         *     your control, or you otherwise expect the data represented by the given String to change without the String
         *     identifier changing, Consider using
         *     {@link GenericRequestBuilder#signature(Key)} to mixin a signature
         *     you create that identifies the data currently at the given String that will invalidate the cache if that data
         *     changes. Alternatively, using {@link DiskCacheStrategy#NONE} and/or
         *     {@link DrawableRequestBuilder#skipMemoryCache(boolean)} may be appropriate.
         * </p>
         *
         * @see #from(Class)
         * @see #load(String)
         */
        public DrawableTypeRequest<String> fromString() {
            return loadGeneric(String.class);           // 返回RequestBuilder
        }
    
        private <T> DrawableTypeRequest<T> loadGeneric(Class<T> modelClass) {
            // ModelLoader 将传入的数据转化成具体的可以解码的数据类型
            ModelLoader<T, InputStream> streamModelLoader = Glide.buildStreamModelLoader(modelClass, context);
            ModelLoader<T, ParcelFileDescriptor> fileDescriptorModelLoader =
                    Glide.buildFileDescriptorModelLoader(modelClass, context);
            if (modelClass != null && streamModelLoader == null && fileDescriptorModelLoader == null) {
                throw new IllegalArgumentException("Unknown type " + modelClass + ". You must provide a Model of a type for"
                        + " which there is a registered ModelLoader, if you are using a custom model, you must first call"
                        + " Glide#register with a ModelLoaderFactory for your custom model class");
            }
            return optionsApplier.apply(
                    new DrawableTypeRequest<T>(modelClass, streamModelLoader, fileDescriptorModelLoader, context,
                            glide, requestTracker, lifecycle, optionsApplier));
        }
    
        ...
    
    }

- into方法

into方法设置Target，Glide会将资源加载到Target中，加载时会取消Target之前已经存在的加载请求

    public Target<TranscodeType> into(ImageView view) {
        Util.assertMainThread();
        if (view == null) {
            throw new IllegalArgumentException("You must pass in a non null View");
        }

        if (!isTransformationSet && view.getScaleType() != null) {
            switch (view.getScaleType()) {
                case CENTER_CROP:
                    applyCenterCrop();
                    break;
                case FIT_CENTER:
                case FIT_START:
                case FIT_END:
                    applyFitCenter();
                    break;
                //$CASES-OMITTED$
                default:
                    // Do nothing.
            }
        }

        return into(glide.buildImageViewTarget(view, transcodeClass));          // GlideDrawableImageViewTarget
    }

    public <Y extends Target<TranscodeType>> Y into(Y target) {
        Util.assertMainThread();
        if (target == null) {
            throw new IllegalArgumentException("You must pass in a non null Target");
        }
        if (!isModelSet) {
            throw new IllegalArgumentException("You must first set a model (try #load())");
        }

        Request previous = target.getRequest();

        if (previous != null) {                                         // 如果之前有请求，取消之前的请求并回收资源
            previous.clear();
            requestTracker.removeRequest(previous);
            previous.recycle();
        }

        Request request = buildRequest(target);
        target.setRequest(request);
        lifecycle.addListener(target);
        requestTracker.runRequest(request);

        return target;
    }

Request请求是如何管理的？  RequestManager

    public final class GenericRequest<A, T, Z, R> implements Request, SizeReadyCallback,
        ResourceCallback {
        private static final String TAG = "GenericRequest";
        private static final Queue<GenericRequest<?, ?, ?, ?>> REQUEST_POOL = Util.createQueue(0);
        private static final double TO_MEGABYTE = 1d / (1024d * 1024d);
    
        private enum Status {
            /** Created but not yet running. */
            PENDING,
            /** In the process of fetching media. */
            RUNNING,
            /** Waiting for a callback given to the Target to be called to determine target dimensions. */
            WAITING_FOR_SIZE,
            /** Finished loading media successfully. */
            COMPLETE,
            /** Failed to load media, may be restarted. */
            FAILED,
            /** Cancelled by the user, may not be restarted. */
            CANCELLED,
            /** Cleared by the user with a placeholder set, may not be restarted. */
            CLEARED,
            /** Temporarily paused by the system, may be restarted. */
            PAUSED,
        }
    
        ...
    
        @Override
        public void begin() {
            startTime = LogTime.getLogTime();
            if (model == null) {
                onException(null);
                return;
            }
    
            status = Status.WAITING_FOR_SIZE;
            if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
                onSizeReady(overrideWidth, overrideHeight);
            } else {
                target.getSize(this);               //  测量完后调用onSizeReady↖
            }
    
            if (!isComplete() && !isFailed() && canNotifyStatusChanged()) {
                target.onLoadStarted(getPlaceholderDrawable());         // 显示默认图
            }
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logV("finished run method in " + LogTime.getElapsedMillis(startTime));
            }
        }
    
        ...
        
        /**
         * A callback method that should never be invoked directly.
         */
        @Override
        public void onSizeReady(int width, int height) {
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logV("Got onSizeReady in " + LogTime.getElapsedMillis(startTime));
            }
            if (status != Status.WAITING_FOR_SIZE) {
                return;
            }
            status = Status.RUNNING;
    
            width = Math.round(sizeMultiplier * width);
            height = Math.round(sizeMultiplier * height);
    
            ModelLoader<A, T> modelLoader = loadProvider.getModelLoader();
            final DataFetcher<T> dataFetcher = modelLoader.getResourceFetcher(model, width, height);
    
            if (dataFetcher == null) {
                onException(new Exception("Failed to load model: \'" + model + "\'"));
                return;
            }
            ResourceTranscoder<Z, R> transcoder = loadProvider.getTranscoder();
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logV("finished setup for calling load in " + LogTime.getElapsedMillis(startTime));
            }
            loadedFromMemoryCache = true;
            // 加载
            loadStatus = engine.load(signature, width, height, dataFetcher, loadProvider, transformation, transcoder,
                    priority, isMemoryCacheable, diskCacheStrategy, this);
            loadedFromMemoryCache = resource != null;
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logV("finished onSizeReady in " + LogTime.getElapsedMillis(startTime));
            }
        }

        ...

    }

创建线程后台加载图片

    public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
            DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
            Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
        Util.assertMainThread();
        long startTime = LogTime.getLogTime();

        final String id = fetcher.getId();
        EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),
                loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),
                transcoder, loadProvider.getSourceEncoder());

        EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
        if (cached != null) {
            cb.onResourceReady(cached);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logWithTimeAndKey("Loaded resource from cache", startTime, key);
            }
            return null;
        }

        EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
        if (active != null) {
            cb.onResourceReady(active);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logWithTimeAndKey("Loaded resource from active resources", startTime, key);
            }
            return null;
        }

        EngineJob current = jobs.get(key);
        if (current != null) {
            current.addCallback(cb);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logWithTimeAndKey("Added to existing load", startTime, key);
            }
            return new LoadStatus(cb, current);
        }

        // 创建runnable
        EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
        DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(key, width, height, fetcher, loadProvider, transformation,
                transcoder, diskCacheProvider, diskCacheStrategy, priority);
        EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
        jobs.put(key, engineJob);
        engineJob.addCallback(cb);           // 加载成功或者错误
        engineJob.start(runnable);

        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Started new load", startTime, key);
        }
        return new LoadStatus(cb, engineJob);
    }