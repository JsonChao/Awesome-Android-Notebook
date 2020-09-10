---

		title:  Android主流三方库源码分析（三、深入理解Glide源码）
		date: 2018/12/16 20:19:00   
		tags: 
		- Android主流三方库源码分析
		categories: 安卓主流三方库源码分析
		thumbnail: http://www.glide.me/wp-content/uploads/2015/06/tweeter_1024x512_B.png
---

---

## 前言

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

    tips:文章太长可以先点赞收藏哦，后续再慢慢阅读~
    

前两篇我们详细地分析了Android的网络底层框架OKHttp和封装框架Retrofit的核心源码，如果对OKHttp或Retrofit内部机制不了解的可以看看[Android主流三方库源码分析（一、深入理解OKHttp源码）](https://juejin.im/post/6844903631909552135)和[Android主流三方库源码分析（二、深入理解Retrofit源码）](https://juejin.im/post/6844903972071981064)。本篇，我们将会来深入地分析下目前Android使用最广泛的图片加载框架框架Glide的源码加载流程。

### 一、基本使用流程

Glide最基本的使用流程就是下面这行代码，其它所有扩展的额外功能都是以其建造者链式调用的基础上增加的。

    GlideApp.with(context).load(url).into(iv);

其中的GlideApp是注解处理器自动生成的，要使用GlideApp，必须先配置应用的AppGlideModule模块，里面可以为空配置，也可以根据实际情况添加指定配置。

    @GlideModule
    public class MyAppGlideModule extends AppGlideModule {
    
        @Override
        public void applyOptions(Context context, GlideBuilder builder) {
            // 实际使用中根据情况可以添加如下配置
            <!--builder.setDefaultRequestOptions(new RequestOptions().format(DecodeFormat.PREFER_RGB_565));-->
            <!--int memoryCacheSizeBytes = 1024 * 1024 * 20;-->
            <!--builder.setMemoryCache(new LruResourceCache(memoryCacheSizeBytes));-->
            <!--int bitmapPoolSizeBytes = 1024 * 1024 * 30;-->
            <!--builder.setBitmapPool(new LruBitmapPool(bitmapPoolSizeBytes));-->
            <!--int diskCacheSizeBytes = 1024 * 1024 * 100;-->
            <!--builder.setDiskCache(new InternalCacheDiskCacheFactory(context, diskCacheSizeBytes));-->
        }
    }
    
接下来，本文将针对Glide的最新源码版本V4.8.0对Glide加载网络图片的流程进行详细地分析与讲解，力争做到让读者朋友们知其然也知其所以然。

### 二、GlideApp.with(context)源码详解

首先，用**艽野尘梦**绘制的这份Glide框架图让我们对Glide的总体框架有一个初步的了解。

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2478933d44534c2fabb6466049b9f3b7~tplv-k3u1fbpfcp-zoom-1.image)

从GlideApp.with这行代码开始，内部主线执行流程如下。

#### 1、GlideApp#with

    return (GlideRequests) Glide.with(context);

#### 2、Glide#with

    return getRetriever(context).get(context);
    
    return Glide.get(context).getRequestManagerRetriever();
    
    // 外部使用了双重检锁的同步方式确保同一时刻只执一次Glide的初始化
    checkAndInitializeGlide(context);
    
    initializeGlide(context);
    
    // 最终执行到Glide的另一个重载方法
    initializeGlide(context, new GlideBuilder());
    
    @SuppressWarnings("deprecation")
      private static void initializeGlide(@NonNull Context   context, @NonNull GlideBuilder builder) {
        Context applicationContext =     context.getApplicationContext();
        // 1、获取前面应用中带注解的GlideModule
        GeneratedAppGlideModule annotationGeneratedModule =     getAnnotationGeneratedGlideModules();
        // 2、如果GlideModule为空或者可配置manifest里面的标志为true，则获取manifest里面
        // 配置的GlideModule模块（manifestModules）。
        List<com.bumptech.glide.module.GlideModule>     manifestModules = Collections.emptyList();
        if (annotationGeneratedModule == null ||     annotationGeneratedModule.isManifestParsingEnabled(    )) {
          manifestModules = new   ManifestParser(applicationContext).parse();
        }
    
        ...
    
        RequestManagerRetriever.RequestManagerFactory     factory =
            annotationGeneratedModule != null
                ? annotationGeneratedModule.getRequestManag    erFactory() : null;
        builder.setRequestManagerFactory(factory);
        for (com.bumptech.glide.module.GlideModule module :     manifestModules) {
          module.applyOptions(applicationContext, builder);
        }
        if (annotationGeneratedModule != null) {
          annotationGeneratedModule.applyOptions(applicatio  nContext, builder);
        }
        // 3、初始化各种配置信息
        Glide glide = builder.build(applicationContext);
        // 4、把manifestModules以及annotationGeneratedModule里面的配置信息放到builder
        // 里面（applyOptions）替换glide默认组件（registerComponents）
        for (com.bumptech.glide.module.GlideModule module :     manifestModules) {
          module.registerComponents(applicationContext,   glide, glide.registry);
        }
        if (annotationGeneratedModule != null) {
          annotationGeneratedModule.registerComponents(appl  icationContext, glide, glide.registry);
        }
        applicationContext.registerComponentCallbacks(glide    );
        Glide.glide = glide;
    }
    
#### 3、GlideBuilder#build

    @NonNull
      Glide build(@NonNull Context context) {
        // 创建请求图片线程池sourceExecutor
        if (sourceExecutor == null) {
          sourceExecutor =   GlideExecutor.newSourceExecutor();
        }
    
        // 创建硬盘缓存线程池diskCacheExecutor
        if (diskCacheExecutor == null) {
          diskCacheExecutor =   GlideExecutor.newDiskCacheExecutor();
        }
    
        // 创建动画线程池animationExecutor
        if (animationExecutor == null) {
          animationExecutor =   GlideExecutor.newAnimationExecutor();
        }
    
        if (memorySizeCalculator == null) {
          memorySizeCalculator = new   MemorySizeCalculator.Builder(context).build();
        }
    
        if (connectivityMonitorFactory == null) {
          connectivityMonitorFactory = new   DefaultConnectivityMonitorFactory();
        }
    
        if (bitmapPool == null) {
          // 依据设备的屏幕密度和尺寸设置各种pool的size
          int size =   memorySizeCalculator.getBitmapPoolSize();
          if (size > 0) {
            // 创建图片线程池LruBitmapPool，缓存所有被释放的bitmap
            // 缓存策略在API大于19时，为SizeConfigStrategy，小于为AttributeStrategy。
            // 其中SizeConfigStrategy是以bitmap的size和config为key，value为bitmap的HashMap
            bitmapPool = new LruBitmapPool(size);
          } else {
            bitmapPool = new BitmapPoolAdapter();
          }
        }
    
        // 创建对象数组缓存池LruArrayPool，默认4M
        if (arrayPool == null) {
          arrayPool = new   LruArrayPool(memorySizeCalculator.getArrayPoolSiz  eInBytes());
        }
    
        // 创建LruResourceCache，内存缓存
        if (memoryCache == null) {
          memoryCache = new   LruResourceCache(memorySizeCalculator.getMemoryCa  cheSize());
        }
    
        if (diskCacheFactory == null) {
          diskCacheFactory = new   InternalCacheDiskCacheFactory(context);
        }
    
        // 创建任务和资源管理引擎（线程池，内存缓存和硬盘缓存对象）
        if (engine == null) {
          engine =
              new Engine(
                  memoryCache,
                  diskCacheFactory,
                  diskCacheExecutor,
                  sourceExecutor,
                  GlideExecutor.newUnlimitedSourceExecutor(  ),
                  GlideExecutor.newAnimationExecutor(),
                  isActiveResourceRetentionAllowed);
        }
        
        RequestManagerRetriever requestManagerRetriever =
        new RequestManagerRetriever(requestManagerFactory);

        return new Glide(
            context,
            engine,
            memoryCache,
            bitmapPool,
            arrayPool,
            requestManagerRetriever,
            connectivityMonitorFactory,
            logLevel,
            defaultRequestOptions.lock(),
            defaultTransitionOptions);
    }

#### 4、Glide#Glide构造方法

    Glide(...) {
        ...
        // 注册管理任务执行对象的类(Registry)
        // Registry是一个工厂，而其中所有注册的对象都是一个工厂员工，当任务分发时，
        // 根据当前任务的性质，分发给相应员工进行处理
        registry = new Registry();
        
        ...
        
        // 这里大概有60余次的append或register员工组件（解析器、编解码器、工厂类、转码类等等组件）
        registry
        .append(ByteBuffer.class, new ByteBufferEncoder())
        .append(InputStream.class, new StreamEncoder(arrayPool))
        
        // 根据给定子类产出对应类型的target（BitmapImageViewTarget / DrawableImageViewTarget)
        ImageViewTargetFactory imageViewTargetFactory = new ImageViewTargetFactory();
        
        glideContext =
            new GlideContext(
                context,
                arrayPool,
                registry,
                imageViewTargetFactory,
                defaultRequestOptions,
                defaultTransitionOptions,
                engine,
                logLevel);
    }
     
#### 5、RequestManagerRetriever#get

    @NonNull
    public RequestManager get(@NonNull Context context) {
      if (context == null) {
        throw new IllegalArgumentException("You cannot start a load on a null Context");
      } else if (Util.isOnMainThread() && !(context instanceof Application)) {
        // 如果当前线程是主线程且context不是Application走相应的get重载方法
        if (context instanceof FragmentActivity) {
          return get((FragmentActivity) context);
        } else if (context instanceof Activity) {
          return get((Activity) context);
        } else if (context instanceof ContextWrapper) {
          return get(((ContextWrapper) context).getBaseContext());
        }
      }
    
      // 否则直接将请求与ApplicationLifecycle关联
      return getApplicationManager(context);
    }
    
这里总结一下，对于当前传入的context是application或当前线程是子线程时，请求的生命周期和ApplicationLifecycle关联，否则，context是FragmentActivity或Fragment时，在当前组件添加一个SupportFragment（SupportRequestManagerFragment），context是Activity时，在当前组件添加一个Fragment(RequestManagerFragment)。
    
#### 6、GlideApp#with小结

##### 1、初始化各式各样的配置信息（包括缓存，请求线程池，大小，图片格式等等）以及glide对象。
##### 2、将glide请求和application/SupportFragment/Fragment的生命周期绑定在一块。

##### 这里我们再回顾一下with方法的执行流程。

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a307a034fea446a486c4f988849751a0~tplv-k3u1fbpfcp-zoom-1.image)
    
### 三、load(url)源码详解

#### 1、GlideRequest(RequestManager)#load

    return (GlideRequest<Drawable>) super.load(string);
    
    return asDrawable().load(string);
    
    // 1、asDrawable部分
    return (GlideRequest<Drawable>) super.asDrawable();
    
    return as(Drawable.class);
    
    // 最终返回了一个GlideRequest（RequestManager的子类）
    return new GlideRequest<>(glide, this, resourceClass, context);
    
    // 2、load部分
    return (GlideRequest<TranscodeType>) super.load(string);
    
    return loadGeneric(string);
    
    @NonNull
    private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
        // model则为设置的url
        this.model = model;
        // 记录url已设置
        isModelSet = true;
        return this;
    }
    
可以看到，load这部分的源码很简单，就是给GlideRequest（RequestManager）设置了要请求的mode（url），并记录了url已设置的状态。

##### 这里，我们再看看load方法的执行流程。

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/33114794d81a421ea4345806826919b7~tplv-k3u1fbpfcp-zoom-1.image)

### 四、into(iv)源码详解

前方预警，真正复杂的地方开始了。

#### 1、RequestBuilder.into

     @NonNull
    public ViewTarget<ImageView, TranscodeType>   into(@NonNull ImageView view) {
      Util.assertMainThread();
      Preconditions.checkNotNull(view);
    
      RequestOptions requestOptions =     this.requestOptions;
      if (!requestOptions.isTransformationSet()
          && requestOptions.isTransformationAllowed()
          && view.getScaleType() != null) {
        // Clone in this method so that if we use this   RequestBuilder to load into a View and then
        // into a different target, we don't retain the   transformation applied based on the previous
        // View's scale type.
        switch (view.getScaleType()) {
          // 这个RequestOptions里保存了要设置的scaleType，Glide自身封装了CenterCrop、CenterInside、
          // FitCenter、CenterInside四种规格。
          case CENTER_CROP:
            requestOptions =   requestOptions.clone().optionalCenterCrop();
            break;
          case CENTER_INSIDE:
            requestOptions =   requestOptions.clone().optionalCenterInside()  ;
            break;
          case FIT_CENTER:
          case FIT_START:
          case FIT_END:
            requestOptions =   requestOptions.clone().optionalFitCenter();
            break;
          case FIT_XY:
            requestOptions =   requestOptions.clone().optionalCenterInside()  ;
            break;
          case CENTER:
          case MATRIX:
          default:
            // Do nothing.
        }
      }
    
      // 注意，这个transcodeClass是指的drawable或bitmap
      return into(
          glideContext.buildImageViewTarget(view,     transcodeClass),
          /*targetListener=*/ null,
          requestOptions);
    }
    
#### 2、GlideContext#buildImageViewTarget

    return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
     
#### 3、ImageViewTargetFactory#buildTarget

    @NonNull
    @SuppressWarnings("unchecked")
    public <Z> ViewTarget<ImageView, Z>   buildTarget(@NonNull ImageView view,
        @NonNull Class<Z> clazz) {
      // 返回展示Bimtap/Drawable资源的目标对象
      if (Bitmap.class.equals(clazz)) {
        return (ViewTarget<ImageView, Z>) new   BitmapImageViewTarget(view);
      } else if (Drawable.class.isAssignableFrom(clazz))     {
        return (ViewTarget<ImageView, Z>) new   DrawableImageViewTarget(view);
      } else {
        throw new IllegalArgumentException(
            "Unhandled class: " + clazz + ", try   .as*(Class).transcode(ResourceTranscoder)");
      }
    }
    
可以看到，Glide内部只维护了两种target，一种是BitmapImageViewTarget，另一种则是DrawableImageViewTarget，接下来继续深入。
    
#### 4、RequestBuilder#into

    private <Y extends Target<TranscodeType>> Y into(
          @NonNull Y target,
          @Nullable RequestListener<TranscodeType>   targetListener,
          @NonNull RequestOptions options) {
        Util.assertMainThread();
        Preconditions.checkNotNull(target);
        if (!isModelSet) {
          throw new IllegalArgumentException("You must call   #load() before calling #into()");
        }
    
        options = options.autoClone();
        // 分析1.建立请求
        Request request = buildRequest(target,     targetListener, options);
    
        Request previous = target.getRequest();
        if (request.isEquivalentTo(previous)
            && !isSkipMemoryCacheWithCompletePreviousReques    t(options, previous)) {
          request.recycle();
          // If the request is completed, beginning again   will ensure the result is re-delivered,
          // triggering RequestListeners and Targets. If   the request is failed, beginning again will
          // restart the request, giving it another chance   to complete. If the request is already
          // running, we can let it continue running   without interruption.
          if (!Preconditions.checkNotNull(previous).isRunni  ng()) {
            // Use the previous request rather than the new     one to allow for optimizations like skipping
            // setting placeholders, tracking and     un-tracking Targets, and obtaining View     dimensions
            // that are done in the individual Request.
            previous.begin();
          }
          return target;
        }
        
        requestManager.clear(target);
        target.setRequest(request);
        // 分析2.真正追踪请求的地方
        requestManager.track(target, request);
    
        return target;
    }
    
    // 分析1
    private Request buildRequest(
          Target<TranscodeType> target,
          @Nullable RequestListener<TranscodeType>   targetListener,
          RequestOptions requestOptions) {
        return buildRequestRecursive(
            target,
            targetListener,
            /*parentCoordinator=*/ null,
            transitionOptions,
            requestOptions.getPriority(),
            requestOptions.getOverrideWidth(),
            requestOptions.getOverrideHeight(),
            requestOptions);
    }

    // 分析1
    private Request buildRequestRecursive(
          Target<TranscodeType> target,
          @Nullable RequestListener<TranscodeType>   targetListener,
          @Nullable RequestCoordinator parentCoordinator,
          TransitionOptions<?, ? super TranscodeType>   transitionOptions,
          Priority priority,
          int overrideWidth,
          int overrideHeight,
          RequestOptions requestOptions) {
    
        // Build the ErrorRequestCoordinator first if     necessary so we can update parentCoordinator.
        ErrorRequestCoordinator errorRequestCoordinator =     null;
        if (errorBuilder != null) {
          // 创建errorRequestCoordinator（异常处理对象）
          errorRequestCoordinator = new   ErrorRequestCoordinator(parentCoordinator);
          parentCoordinator = errorRequestCoordinator;
        }
    
        // 递归建立缩略图请求
        Request mainRequest =
            buildThumbnailRequestRecursive(
                target,
                targetListener,
                parentCoordinator,
                transitionOptions,
                priority,
                overrideWidth,
                overrideHeight,
                requestOptions);
    
        if (errorRequestCoordinator == null) {
          return mainRequest;
        }
    
        ...
        
        Request errorRequest =     errorBuilder.buildRequestRecursive(
            target,
            targetListener,
            errorRequestCoordinator,
            errorBuilder.transitionOptions,
            errorBuilder.requestOptions.getPriority(),
            errorOverrideWidth,
            errorOverrideHeight,
            errorBuilder.requestOptions);
        errorRequestCoordinator.setRequests(mainRequest,     errorRequest);
        return errorRequestCoordinator;
    }
    
    // 分析1
    private Request buildThumbnailRequestRecursive(
          Target<TranscodeType> target,
          RequestListener<TranscodeType> targetListener,
          @Nullable RequestCoordinator parentCoordinator,
          TransitionOptions<?, ? super TranscodeType> transitionOptions,
          Priority priority,
          int overrideWidth,
          int overrideHeight,
          RequestOptions requestOptions) {
        if (thumbnailBuilder != null) {
          // Recursive case: contains a potentially recursive thumbnail request builder.
          
          ...

          ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
          // 获取一个正常请求对象
          Request fullRequest =
              obtainRequest(
                  target,
                  targetListener,
                  requestOptions,
                  coordinator,
                  transitionOptions,
                  priority,
                  overrideWidth,
                  overrideHeight);
          isThumbnailBuilt = true;
          // Recursively generate thumbnail requests.
          // 使用递归的方式建立一个缩略图请求对象
          Request thumbRequest =
              thumbnailBuilder.buildRequestRecursive(
                  target,
                  targetListener,
                  coordinator,
                  thumbTransitionOptions,
                  thumbPriority,
                  thumbOverrideWidth,
                  thumbOverrideHeight,
                  thumbnailBuilder.requestOptions);
          isThumbnailBuilt = false;
          // coordinator（ThumbnailRequestCoordinator）是作为两者的协调者，
          // 能够同时加载缩略图和正常的图的请求
          coordinator.setRequests(fullRequest, thumbRequest);
          return coordinator;
        } else if (thumbSizeMultiplier != null) {
          // Base case: thumbnail multiplier generates a thumbnail request, but cannot recurse.
          // 当设置了缩略的比例thumbSizeMultiplier(0 ~  1)时，
          // 不需要递归建立缩略图请求
          ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
          Request fullRequest =
              obtainRequest(
                  target,
                  targetListener,
                  requestOptions,
                  coordinator,
                  transitionOptions,
                  priority,
                  overrideWidth,
                  overrideHeight);
          RequestOptions thumbnailOptions = requestOptions.clone()
              .sizeMultiplier(thumbSizeMultiplier);

          Request thumbnailRequest =
              obtainRequest(
                  target,
                  targetListener,
                  thumbnailOptions,
                  coordinator,
                  transitionOptions,
                  getThumbnailPriority(priority),
                  overrideWidth,
                  overrideHeight);

          coordinator.setRequests(fullRequest, thumbnailRequest);
          return coordinator;
        } else {
          // Base case: no thumbnail.
          // 没有缩略图请求时，直接获取一个正常图请求
          return obtainRequest(
              target,
              targetListener,
              requestOptions,
              parentCoordinator,
              transitionOptions,
              priority,
              overrideWidth,
              overrideHeight);
        }
    }
    
    private Request obtainRequest(
          Target<TranscodeType> target,
          RequestListener<TranscodeType> targetListener,
          RequestOptions requestOptions,
          RequestCoordinator requestCoordinator,
          TransitionOptions<?, ? super TranscodeType>   transitionOptions,
          Priority priority,
          int overrideWidth,
          int overrideHeight) {
        // 最终实际返回的是一个SingleRequest对象（将制定的资源加载进对应的Target
        return SingleRequest.obtain(
            context,
            glideContext,
            model,
            transcodeClass,
            requestOptions,
            overrideWidth,
            overrideHeight,
            priority,
            target,
            targetListener,
            requestListeners,
            requestCoordinator,
            glideContext.getEngine(),
            transitionOptions.getTransitionFactory());
    }
    
从上源码分析可知，我们在分析1处的buildRequest()方法里建立了请求，且最多可同时进行缩略图和正常图的请求，最后，调用了requestManager.track(target, request)方法，接着看看track里面做了什么。
    
#### 5、RequestManager#track

    // 分析2
    void track(@NonNull Target<?> target, @NonNull Request request) {
        // 加入一个target目标集合(Set)
        targetTracker.track(target);
        
        requestTracker.runRequest(request);
    }

#### 6、RequestTracker#runRequest
    
    /**
    * Starts tracking the given request.
    */
    // 分析2
    public void runRequest(@NonNull Request request) {
        requests.add(request);
        if (!isPaused) {
          // 如果不是暂停状态则开始请求
          request.begin();
        } else {
          request.clear();
          if (Log.isLoggable(TAG, Log.VERBOSE)) {
            Log.v(TAG, "Paused, delaying request");
          }
          // 否则清空请求，加入延迟请求队列（为了对这些请求维持一个强引用，使用了ArrayList实现）
          pendingRequests.add(request);
        }
    }
    
#### 7、SingleRequest#begin

    // 分析2
    @Override
    public void begin() {
      
      ...
      
      if (model == null) {
      
        ...
        // model（url）为空，回调加载失败
        onLoadFailed(new GlideException("Received null   model"), logLevel);
        return;
      }
    
      if (status == Status.RUNNING) {
        throw new IllegalArgumentException("Cannot   restart a running request");
      }
    
     
      if (status == Status.COMPLETE) {
        onResourceReady(resource,   DataSource.MEMORY_CACHE);
        return;
      }
    
      status = Status.WAITING_FOR_SIZE;
      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        // 当使用override() API为图片指定了一个固定的宽高时直接执行onSizeReady，
        // 最终的核心处理位于onSizeReady
        onSizeReady(overrideWidth, overrideHeight);
      } else {
        // 根据imageView的宽高算出图片的宽高，最终也会走到onSizeReady
        target.getSize(this);
      }
    
      if ((status == Status.RUNNING || status ==     Status.WAITING_FOR_SIZE)
          && canNotifyStatusChanged()) {
        // 预先加载设置的缩略图
        target.onLoadStarted(getPlaceholderDrawable());
      }
      if (IS_VERBOSE_LOGGABLE) {
        logV("finished run method in " +   LogTime.getElapsedMillis(startTime));
      }
    }
    
从requestManager.track(target, request)开始，最终会执行到SingleRequest#begin()方法的onSizeReady，可以猜到（因为后面只做了预加载缩略图的处理），真正的请求就是从这里开始的，咱们进去一探究竟~
    
#### 8、SingleRequest#onSizeReady

    // 分析2
    @Override
    public void onSizeReady(int width, int height) {
      stateVerifier.throwIfRecycled();
      
      ...
      
      status = Status.RUNNING;
    
      float sizeMultiplier =     requestOptions.getSizeMultiplier();
      this.width = maybeApplySizeMultiplier(width,     sizeMultiplier);
      this.height = maybeApplySizeMultiplier(height,     sizeMultiplier);
    
      ...
      
      // 根据给定的配置进行加载，engine是一个负责加载、管理活跃和缓存资源的引擎类
      loadStatus = engine.load(
          glideContext,
          model,
          requestOptions.getSignature(),
          this.width,
          this.height,
          requestOptions.getResourceClass(),
          transcodeClass,
          priority,
          requestOptions.getDiskCacheStrategy(),
          requestOptions.getTransformations(),
          requestOptions.isTransformationRequired(),
          requestOptions.isScaleOnlyOrNoTransform(),
          requestOptions.getOptions(),
          requestOptions.isMemoryCacheable(),
          requestOptions.getUseUnlimitedSourceGeneratorsP    ool(),
          requestOptions.getUseAnimationPool(),
          requestOptions.getOnlyRetrieveFromCache(),
          this);
    
      ...
    }

终于看到Engine类了，感觉距离成功不远了，继续~
    
#### 9、Engine#load

    public <R> LoadStatus load(
        GlideContext glideContext,
        Object model,
        Key signature,
        int width,
        int height,
        Class<?> resourceClass,
        Class<R> transcodeClass,
        Priority priority,
        DiskCacheStrategy diskCacheStrategy,
        Map<Class<?>, Transformation<?>> transformations,
        boolean isTransformationRequired,
        boolean isScaleOnlyOrNoTransform,
        Options options,
        boolean isMemoryCacheable,
        boolean useUnlimitedSourceExecutorPool,
        boolean useAnimationPool,
        boolean onlyRetrieveFromCache,
        ResourceCallback cb) {
      
      ...
    
      // 先从弱引用中查找，如果有的话回调onResourceReady并直接返回
      EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
      if (active != null) {
        cb.onResourceReady(active,   DataSource.MEMORY_CACHE);
        if (VERBOSE_IS_LOGGABLE) {
          logWithTimeAndKey("Loaded resource from active     resources", startTime, key);
        }
        return null;
      }
    
      // 没有再从内存中查找,有的话会取出并放到ActiveResources（内部维护的弱引用缓存map）里面
      EngineResource<?> cached = loadFromCache(key,     isMemoryCacheable);
      if (cached != null) {
        cb.onResourceReady(cached,   DataSource.MEMORY_CACHE);
        if (VERBOSE_IS_LOGGABLE) {
          logWithTimeAndKey("Loaded resource from cache",     startTime, key);
        }
        return null;
      }
    
      EngineJob<?> current = jobs.get(key,     onlyRetrieveFromCache);
      if (current != null) {
        current.addCallback(cb);
        if (VERBOSE_IS_LOGGABLE) {
          logWithTimeAndKey("Added to existing load",     startTime, key);
        }
        return new LoadStatus(cb, current);
      }
    
      // 如果内存中没有，则创建engineJob（decodejob的回调类，管理下载过程以及状态）
      EngineJob<R> engineJob =
          engineJobFactory.build(
              key,
              isMemoryCacheable,
              useUnlimitedSourceExecutorPool,
              useAnimationPool,
              onlyRetrieveFromCache);
    
      // 创建解析工作对象
      DecodeJob<R> decodeJob =
          decodeJobFactory.build(
              glideContext,
              model,
              key,
              signature,
              width,
              height,
              resourceClass,
              transcodeClass,
              priority,
              diskCacheStrategy,
              transformations,
              isTransformationRequired,
              isScaleOnlyOrNoTransform,
              onlyRetrieveFromCache,
              options,
              engineJob);
    
      // 放在Jobs内部维护的HashMap中
      jobs.put(key, engineJob);
    
      // 关注点8 后面分析会用到
      // 注册ResourceCallback接口
      engineJob.addCallback(cb);
      // 内部开启线程去请求
      engineJob.start(decodeJob);
    
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Started new load", startTime,   key);
      }
      return new LoadStatus(cb, engineJob);
    }
    
    public void start(DecodeJob<R> decodeJob) {
        this.decodeJob = decodeJob;
        // willDecodeFromCache方法内部根据不同的阶段stage，如果是RESOURCE_CACHE/DATA_CACHE则返回true，使用diskCacheExecutor，否则调用getActiveSourceExecutor，内部会根据相应的条件返回sourceUnlimitedExecutor/animationExecutor/sourceExecutor
        GlideExecutor executor =   
        decodeJob.willDecodeFromCache()
            ? diskCacheExecutor
            : getActiveSourceExecutor();
        executor.execute(decodeJob);
    }
    
可以看到，最终Engine(引擎)类内部会执行到自身的start方法，它会根据不同的配置采用不同的线程池使用diskCacheExecutor/sourceUnlimitedExecutor/animationExecutor/sourceExecutor来执行最终的解码任务decodeJob。

#### 10、DecodeJob#run
    
    runWrapped();
    
    private void runWrapped() {
        switch (runReason) {
          case INITIALIZE:
            stage = getNextStage(Stage.INITIALIZE);
            // 关注点1
            currentGenerator = getNextGenerator();
            // 关注点2 内部会调用相应Generator的startNext()
            runGenerators();
            break;
          case SWITCH_TO_SOURCE_SERVICE:
            runGenerators();
            break;
          case DECODE_DATA:
            // 关注点3 将获取的数据解码成对应的资源
            decodeFromRetrievedData();
            break;
          default:
            throw new IllegalStateException("Unrecognized     run reason: " + runReason);
        }
    }
    
    // 关注点1，完整情况下，会异步依次生成这里的ResourceCacheGenerator、DataCacheGenerator和SourceGenerator对象，并在之后执行其中的startNext()
    private DataFetcherGenerator getNextGenerator() {
        switch (stage) {
          case RESOURCE_CACHE:
            return new ResourceCacheGenerator(decodeHelper, this);
          case DATA_CACHE:
            return new DataCacheGenerator(decodeHelper, this);
          case SOURCE:
            return new SourceGenerator(decodeHelper, this);
          case FINISHED:
            return null;
          default:
            throw new IllegalStateException("Unrecognized     stage: " + stage);
        }
    }
    
#### 11、SourceGenerator#startNext

    // 关注点2
    @Override
    public boolean startNext() {
      // dataToCache数据不为空的话缓存到硬盘（第一执行该方法是不会调用的）
      if (dataToCache != null) {
        Object data = dataToCache;
        dataToCache = null;
        cacheData(data);
      }
    
      if (sourceCacheGenerator != null &&     sourceCacheGenerator.startNext()) {
        return true;
      }
      sourceCacheGenerator = null;
    
      loadData = null;
      boolean started = false;
      while (!started && hasNextModelLoader()) {
        // 关注点4 getLoadData()方法内部会在modelLoaders里面找到ModelLoder对象
        // （每个Generator对应一个ModelLoader），
        // 并使用modelLoader.buildLoadData方法返回一个loadData列表
        loadData =   helper.getLoadData().get(loadDataListIndex++);
        if (loadData != null
            && (helper.getDiskCacheStrategy().isDataCache  able(loadData.fetcher.getDataSource())
            || helper.hasLoadPath(loadData.fetcher.getDat  aClass()))) {
          started = true;
          // 关注点6 通过loadData对象的fetcher对象（有关注点3的分析可知其实现类为HttpUrlFetcher）的
          // loadData方法来获取图片数据
          loadData.fetcher.loadData(helper.getPriority(),     this);
        }
      }
      return started;
    }
    
#### 12、DecodeHelper#getLoadData

    List<LoadData<?>> getLoadData() {
        if (!isLoadDataSet) {
          isLoadDataSet = true;
          loadData.clear();
          List<ModelLoader<Object, ?>> modelLoaders =   glideContext.getRegistry().getModelLoaders(model)  ;
          //noinspection ForLoopReplaceableByForEach to   improve perf
          for (int i = 0, size = modelLoaders.size(); i <   size; i++) {
            ModelLoader<Object, ?> modelLoader =     modelLoaders.get(i);
            // 注意：这里最终是通过HttpGlideUrlLoader的buildLoadData获取到实际的loadData对象
            LoadData<?> current =
                modelLoader.buildLoadData(model, width,     height, options);
            if (current != null) {
              loadData.add(current);
            }
          }
        }
        return loadData;
    }
    
#### 13、HttpGlideUrlLoader#buildLoadData

    @Override
    public LoadData<InputStream> buildLoadData(@NonNull   GlideUrl model, int width, int height,
        @NonNull Options options) {
      // GlideUrls memoize parsed URLs so caching them     saves a few object instantiations and time
      // spent parsing urls.
      GlideUrl url = model;
      if (modelCache != null) {
        url = modelCache.get(model, 0, 0);
        if (url == null) {
          // 关注点5
          modelCache.put(model, 0, 0, model);
          url = model;
        }
      }
      int timeout = options.get(TIMEOUT);
      // 注意，这里创建了一个DataFetcher的实现类HttpUrlFetcher
      return new LoadData<>(url, new HttpUrlFetcher(url,     timeout));
    }
    
    // 关注点5
    public void put(A model, int width, int height, B value) {
        ModelKey<A> key = ModelKey.get(model, width,     height);
        // 最终是通过LruCache来缓存对应的值，key是一个ModelKey对象（由model、width、height三个属性组成）
        cache.put(key, value);
    }

从这里的分析，我们明白了HttpUrlFetcher实际上就是最终的请求执行者，而且，我们知道了Glide会使用LruCache来对解析后的url来进行缓存，以便后续可以省去解析url的时间。

#### 14、HttpUrlFetcher#loadData

    @Override
    public void loadData(@NonNull Priority priority,
        @NonNull DataCallback<? super InputStream>   callback) {
      long startTime = LogTime.getLogTime();
      try {
        // 关注点6
        // loadDataWithRedirects内部是通过HttpURLConnection网络请求数据
        InputStream result =   loadDataWithRedirects(glideUrl.toURL(), 0, null,   glideUrl.getHeaders());
        // 请求成功回调onDataReady()
        callback.onDataReady(result);
      } catch (IOException e) {
        if (Log.isLoggable(TAG, Log.DEBUG)) {
          Log.d(TAG, "Failed to load data for url", e);
        }
        callback.onLoadFailed(e);
      } finally {
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
          Log.v(TAG, "Finished http url fetcher fetch in     " + LogTime.getElapsedMillis(startTime));
        }
      }
    }
    
    private InputStream loadDataWithRedirects(URL url, int redirects, URL lastUrl,
      Map<String, String> headers) throws IOException {
        
        ...
    
        urlConnection.connect();
        // Set the stream so that it's closed in cleanup to avoid resource leaks. See #2352.
        stream = urlConnection.getInputStream();
        if (isCancelled) {
          return null;
        }
        final int statusCode = urlConnection.getResponseCode();
        // 只要是2xx形式的状态码则判断为成功
        if (isHttpOk(statusCode)) {
          // 从urlConnection中获取资源流
          return getStreamForSuccessfulRequest(urlConnection);
        } else if (isHttpRedirect(statusCode)) {
        
          ...
          
          // 重定向请求
          return loadDataWithRedirects(redirectUrl, redirects + 1, url,   headers);
        } else if (statusCode == INVALID_STATUS_CODE) {
          throw new HttpException(statusCode);
        } else {
          throw new HttpException(urlConnection.getResponseMessage(),   statusCode);
        }
    }
    
    private InputStream getStreamForSuccessfulRequest(HttpURLConnection urlConnection)
      throws IOException {
        if (TextUtils.isEmpty(urlConnection.getContentEncoding())) {
          int contentLength = urlConnection.getContentLength();
          stream = ContentLengthInputStream.obtain(urlConnection.getInputStr  eam(), contentLength);
        } else {
          if (Log.isLoggable(TAG, Log.DEBUG)) {
            Log.d(TAG, "Got non empty content encoding: " +     urlConnection.getContentEncoding());
          }
          stream = urlConnection.getInputStream();
        }
        return stream;
    }
    
在HttpUrlFetcher#loadData方法的loadDataWithRedirects里面，Glide通过原生的HttpURLConnection进行请求后，并调用getStreamForSuccessfulRequest()方法获取到了最终的图片流。

#### 15、DecodeJob#run

在我们通过HtttpUrlFetcher的loadData()方法请求得到对应的流之后，我们还必须对流进行处理得到最终我们想要的资源。这里我们回到第10步DecodeJob#run方法的关注点3处，这行代码将会对流进行解码。

    decodeFromRetrievedData();
    
接下来，继续看看他内部的处理。

    private void decodeFromRetrievedData() {
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
          logWithTimeAndKey("Retrieved data", startFetchTime,
              "data: " + currentData
                  + ", cache key: " + currentSourceKey
                  + ", fetcher: " + currentFetcher);
        }
        Resource<R> resource = null;
        try {
          //  核心代码 
          // 从数据中解码得到资源
          resource = decodeFromData(currentFetcher, currentData,   currentDataSource);
        } catch (GlideException e) {
          e.setLoggingDetails(currentAttemptingKey, currentDataSource);
          throwables.add(e);
        }
        if (resource != null) {
          // 关注点8 
          // 编码和发布最终得到的Resource<Bitmap>对象
          notifyEncodeAndRelease(resource, currentDataSource);
        } else {
          runGenerators();
        }
    }
    
     private <Data> Resource<R> decodeFromData(DataFetcher<?> fetcher, Data data,
      DataSource dataSource) throws GlideException {
        try {
          if (data == null) {
            return null;
          }
          long startTime = LogTime.getLogTime();
          // 核心代码
          // 进一步包装了解码方法
          Resource<R> result = decodeFromFetcher(data, dataSource);
          if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Decoded result " + result, startTime);
          }
          return result;
        } finally {
          fetcher.cleanup();
        }
    }
    
    @SuppressWarnings("unchecked")
    private <Data> Resource<R> decodeFromFetcher(Data data, DataSource dataSource)
      throws GlideException {
        LoadPath<Data, ?, R> path = decodeHelper.getLoadPath((Class<Data>) data.getClass());
        // 核心代码
        // 将解码任务分发给LoadPath
        return runLoadPath(data, dataSource, path);
    }

    private <Data, ResourceType> Resource<R> runLoadPath(Data data, DataSource dataSource,
      LoadPath<Data, ResourceType, R> path) throws GlideException {
        Options options = getOptionsWithHardwareConfig(dataSource);
        // 将数据进一步包装
        DataRewinder<Data> rewinder =     glideContext.getRegistry().getRewinder(data);
        try {
          // ResourceType in DecodeCallback below is required for   compilation to work with gradle.
          // 核心代码
          // 将解码任务分发给LoadPath
          return path.load(
              rewinder, options, width, height, new   DecodeCallback<ResourceType>(dataSource));
        } finally {
          rewinder.cleanup();
        }
    }
    
#### 16、LoadPath#load
    
    public Resource<Transcode> load(DataRewinder<Data> rewinder, @NonNull Options options, int width,
      int height, DecodePath.DecodeCallback<ResourceType> decodeCallback) throws GlideException {
    List<Throwable> throwables = Preconditions.checkNotNull(listPool.acquire());
    try {
      // 核心代码
      return loadWithExceptionList(rewinder, options, width, height, decodeCallback, throwables);
    } finally {
      listPool.release(throwables);
    }
  }

    private Resource<Transcode> loadWithExceptionList(DataRewinder<Data> rewinder,
          @NonNull Options options,
          int width, int height, DecodePath.DecodeCallback<ResourceType>   decodeCallback,
          List<Throwable> exceptions) throws GlideException {
        Resource<Transcode> result = null;
        //noinspection ForLoopReplaceableByForEach to improve perf
        for (int i = 0, size = decodePaths.size(); i < size; i++) {
          DecodePath<Data, ResourceType, Transcode> path =   decodePaths.get(i);
          try {
            // 核心代码
            // 将解码任务又进一步分发给DecodePath的decode方法去解码
            result = path.decode(rewinder, width, height, options,     decodeCallback);
          } catch (GlideException e) {
            exceptions.add(e);
          }
          if (result != null) {
            break;
          }
        }
    
        if (result == null) {
          throw new GlideException(failureMessage, new   ArrayList<>(exceptions));
        }
    
        return result;
    }

#### 17、DecodePath#decode

    public Resource<Transcode> decode(DataRewinder<DataType> rewinder,     int width, int height,
          @NonNull Options options, DecodeCallback<ResourceType> callback)   throws GlideException {
        // 核心代码
        // 继续调用DecodePath的decodeResource方法去解析出数据
        Resource<ResourceType> decoded = decodeResource(rewinder, width,     height, options);
        Resource<ResourceType> transformed =     callback.onResourceDecoded(decoded);
        return transcoder.transcode(transformed, options);
    }

    @NonNull
    private Resource<ResourceType> decodeResource(DataRewinder<DataType>   rewinder, int width,
        int height, @NonNull Options options) throws GlideException {
      List<Throwable> exceptions =     Preconditions.checkNotNull(listPool.acquire());
      try {
        // 核心代码
        return decodeResourceWithList(rewinder, width, height, options,   exceptions);
      } finally {
        listPool.release(exceptions);
      }
    }
    
    @NonNull
    private Resource<ResourceType>   decodeResourceWithList(DataRewinder<DataType> rewinder, int width,
        int height, @NonNull Options options, List<Throwable> exceptions)   throws GlideException {
      Resource<ResourceType> result = null;
      //noinspection ForLoopReplaceableByForEach to improve perf
      for (int i = 0, size = decoders.size(); i < size; i++) {
        ResourceDecoder<DataType, ResourceType> decoder = decoders.get(i);
        try {
          DataType data = rewinder.rewindAndGet();
          if (decoder.handles(data, options)) {
            // 获取包装的数据
            data = rewinder.rewindAndGet();
            // 核心代码 
            // 根据DataType和ResourceType的类型分发给不同的解码器Decoder
            result = decoder.decode(data, width, height, options);
          }
        } catch (IOException | RuntimeException | OutOfMemoryError e) {
          if (Log.isLoggable(TAG, Log.VERBOSE)) {
            Log.v(TAG, "Failed to decode data for " + decoder, e);
          }
          exceptions.add(e);
        }
    
        if (result != null) {
          break;
        }
      }
    
      if (result == null) {
        throw new GlideException(failureMessage, new   ArrayList<>(exceptions));
      }
      return result;
    }
    
可以看到，经过一连串的嵌套调用，最终执行到了decoder.decode()这行代码，decode是一个ResourceDecoder<DataType, ResourceType>接口（资源解码器），根据不同的DataType和ResourceType它会有不同的实现类，这里的实现类是ByteBufferBitmapDecoder，接下来让我们来看看这个解码器内部的解码流程。

#### 18、ByteBufferBitmapDecoder#decode

    /**
     * Decodes {@link android.graphics.Bitmap Bitmaps} from {@link    java.nio.ByteBuffer ByteBuffers}.
     */
    public class ByteBufferBitmapDecoder implements     ResourceDecoder<ByteBuffer, Bitmap> {
      
      ...
    
      @Override
      public Resource<Bitmap> decode(@NonNull ByteBuffer source, int width,   int height,
          @NonNull Options options)
          throws IOException {
        InputStream is = ByteBufferUtil.toStream(source);
        // 核心代码
        return downsampler.decode(is, width, height, options);
      }
    }
    
可以看到，最终是使用了一个downsampler，它是一个压缩器，主要是对流进行解码，压缩，圆角等处理。

#### 19、DownSampler#decode

    public Resource<Bitmap> decode(InputStream is, int outWidth, int outHeight,
      Options options) throws IOException {
        return decode(is, outWidth, outHeight, options, EMPTY_CALLBACKS);
    }
    
     @SuppressWarnings({"resource", "deprecation"})
    public Resource<Bitmap> decode(InputStream is, int requestedWidth, int requestedHeight,
          Options options, DecodeCallbacks callbacks) throws IOException {
        Preconditions.checkArgument(is.markSupported(), "You must provide an     InputStream that supports"
            + " mark()");
    
        ...
    
        try {
          // 核心代码
          Bitmap result = decodeFromWrappedStreams(is, bitmapFactoryOptions,
              downsampleStrategy, decodeFormat, isHardwareConfigAllowed,   requestedWidth,
              requestedHeight, fixBitmapToRequestedDimensions, callbacks);
          // 关注点7   
          // 解码得到Bitmap对象后，包装成BitmapResource对象返回，
          // 通过内部的get方法得到Resource<Bitmap>对象
          return BitmapResource.obtain(result, bitmapPool);
        } finally {
          releaseOptions(bitmapFactoryOptions);
          byteArrayPool.put(bytesForOptions);
        }
    }
    
    private Bitmap decodeFromWrappedStreams(InputStream is,
          BitmapFactory.Options options, DownsampleStrategy downsampleStrategy,
          DecodeFormat decodeFormat, boolean isHardwareConfigAllowed, int requestedWidth,
          int requestedHeight, boolean fixBitmapToRequestedDimensions,
          DecodeCallbacks callbacks) throws IOException {
        
        // 省去计算压缩比例等一系列非核心逻辑
        ...
        
        // 核心代码
        Bitmap downsampled = decodeStream(is, options, callbacks, bitmapPool);
        callbacks.onDecodeComplete(bitmapPool, downsampled);

        ...

        // Bimtap旋转处理
        ...
        
        return rotated;
    }

    private static Bitmap decodeStream(InputStream is,     BitmapFactory.Options options,
          DecodeCallbacks callbacks, BitmapPool bitmapPool) throws   IOException {
        
        ...
        
        TransformationUtils.getBitmapDrawableLock().lock();
        try {
          // 核心代码
          result = BitmapFactory.decodeStream(is, null, options);
        } catch (IllegalArgumentException e) {
          ...
        } finally {
          TransformationUtils.getBitmapDrawableLock().unlock();
        }
    
        if (options.inJustDecodeBounds) {
          is.reset();
        }
        return result;
    }
    
从以上源码流程我们知道，最后是在DownSampler的decodeStream()方法中使用了BitmapFactory.decodeStream()来得到Bitmap对象。然后，我们来分析下图片时如何显示的，我们回到步骤19的DownSampler#decode方法，看到关注点7，这里是将Bitmap包装成BitmapResource对象返回，通过内部的get方法可以得到Resource<Bitmap>对象，再回到步骤15的DecodeJob#run方法，这是使用了notifyEncodeAndRelease()方法对Resource<Bitmap>对象进行了发布。

#### 20、DecodeJob#notifyEncodeAndRelease

    private void notifyEncodeAndRelease(Resource<R> resource, DataSource     dataSource) {
     
        ...
    
        notifyComplete(result, dataSource);
    
        ...
        
    }
    
    private void notifyComplete(Resource<R> resource, DataSource     dataSource) {
        setNotifiedOrThrow();
        callback.onResourceReady(resource, dataSource);
    }

从以上EngineJob的源码可知，它实现了DecodeJob.CallBack<R>这个接口。

    class EngineJob<R> implements DecodeJob.Callback<R>,
        Poolable {
        ...
    }
    

#### 21、EngineJob#onResourceReady

    @Override
    public void onResourceReady(Resource<R> resource, DataSource   dataSource) {
      this.resource = resource;
      this.dataSource = dataSource;
      MAIN_THREAD_HANDLER.obtainMessage(MSG_COMPLETE, this).sendToTarget();
    }
    
    private static class MainThreadCallback implements Handler.Callback{
    
        ...
    
        @Override
        public boolean handleMessage(Message message) {
          EngineJob<?> job = (EngineJob<?>) message.obj;
          switch (message.what) {
            case MSG_COMPLETE:
              // 核心代码
              job.handleResultOnMainThread();
              break;
            ...
          }
          return true;
        }
    }
    
从以上源码可知，通过主线程Handler对象进行切换线程，然后在主线程调用了handleResultOnMainThread这个方法。

    @Synthetic
    void handleResultOnMainThread() {
      ...
    
      //noinspection ForLoopReplaceableByForEach to improve perf
      for (int i = 0, size = cbs.size(); i < size; i++) {
        ResourceCallback cb = cbs.get(i);
        if (!isInIgnoredCallbacks(cb)) {
          engineResource.acquire();
          cb.onResourceReady(engineResource, dataSource);
        }
      }
     
      ...
    }
    
这里又通过一个循环调用了所有ResourceCallback的方法，让我们回到步骤9处Engine#load方法的关注点8这行代码，这里对ResourceCallback进行了注册，在步骤8出SingleRequest#onSizeReady方法里的engine.load中，我们看到最后一个参数，传入的是this，可以明白，engineJob.addCallback(cb)这里的cb的实现类就是SingleRequest。接下来，让我们看看SingleRequest的onResourceReady方法。

#### 22、SingleRequest#onResourceReady

    /**
     * A callback method that should never be invoked directly.
     */
    @SuppressWarnings("unchecked")
    @Override
    public void onResourceReady(Resource<?> resource, DataSource   dataSource) {
      ...
      
      // 从Resource<Bitmap>中得到Bitmap对象
      Object received = resource.get();
      
      ...
      
      onResourceReady((Resource<R>) resource, (R) received, dataSource);
    }
    
    private void onResourceReady(Resource<R> resource, R resultDataSource dataSource) {
    
        ...
    
        try {
          ...
    
          if (!anyListenerHandledUpdatingTarget) {
            Transition<? super R> animation =
                animationFactory.build(dataSource, isFirstResource);
            // 核心代码
            target.onResourceReady(result, animation);
          }
        } finally {
          isCallingCallbacks = false;
        }
    
        notifyLoadSuccess();
    }
    
在SingleRequest#onResourceReady方法中又调用了target.onResourceReady(result, animation)方法，这里的target其实就是我们在into方法中建立的那个BitmapImageViewTarget，看到BitmapImageViewTarget类，我们并没有发现onResourceReady方法，但是我们从它的子类ImageViewTarget中发现了onResourceReady方法，从这里我们继续往下看。

#### 23、ImageViewTarget#onResourceReady

    public abstract class ImageViewTarget<Z> extends ViewTarget<ImageView, Z>
    implements Transition.ViewAdapter {
    
        ...

        @Override
        public void onResourceReady(@NonNull Z resource, @Nullable       Transition<? super Z> transition) {
          if (transition == null || !transition.transition(resource, this))   {
            // 核心代码
            setResourceInternal(resource);
          } else {
            maybeUpdateAnimatable(resource);
          }
        }
     
        ...
        
        private void setResourceInternal(@Nullable Z resource) {
            // Order matters here. Set the resource first to make sure that the         Drawable has a valid and
            // non-null Callback before starting it.
            // 核心代码
            setResource(resource);
            maybeUpdateAnimatable(resource);
        }
        
        // 核心代码
        protected abstract void setResource(@Nullable Z resource);
    }

这里我们在回到BitmapImageViewTarget的setResource方法中，我们终于看到Bitmap被设置到了当前的imageView上了。

    public class BitmapImageViewTarget extends ImageViewTarget<Bitmap> {
    
        ...
        
        
        @Override
        protected void setResource(Bitmap resource) {
          view.setImageBitmap(resource);
        }
    }
    
到这里，我们的分析就结束了，从以上的分析可知，Glide将大部分的逻辑处理都放在了最后一个into方法中，里面经过了20多个分析步骤才将请求图片流、解码出图片，到最终设置到对应的imageView上。

##### 最后，这里给出一份我花费了数个小时绘制的完整Glide加载流程图，非常珍贵，大家可以仔仔细细再把Glide的主体流程在梳理一遍。

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9da2924eeab4ef79249b5836fd916da~tplv-k3u1fbpfcp-zoom-1.image)


### 五、总结

到此，Glide整个的加载流程分析就结束了，可以看到，Glide最核心的逻辑都聚集在into()方法中，它里面的设计精巧而复杂，这部分的源码分析非常耗时，但是，如果你真真正正地去一步步去深入其中，你也许在Android进阶之路上将会有顿悟的感觉。目前，Android主流三方库源码分析系列已经对网络库（OkHttp、Retrofit）和图片加载库（Glide）进行了详细的源码分析，接下来，将会对数据库框架GreenDao的核心源码进行深入的分析，敬请期待~


# 公众号

我的公众号 `JsonChao` 开通啦，如果您想第一时间获取最新文章和最新动态，欢迎扫描关注~

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/67f603675080457ba5961556d443a128~tplv-k3u1fbpfcp-zoom-1.image)


##### 参考链接：
---
1、Glide V4.8.0源码

2、[从源码的角度理解Glide的执行流程](https://blog.csdn.net/guolin_blog/article/details/53939176)

3、[glide源码分析](https://zhuanlan.zhihu.com/p/37297719)


## 赞赏

如果这个库对您有很大帮助，您愿意支持这个项目的进一步开发和这个项目的持续维护。你可以扫描下面的二维码，让我喝一杯咖啡或啤酒。非常感谢您的捐赠。谢谢！

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/74d2307a89094c3b9fa5389b103a2450~tplv-k3u1fbpfcp-zoom-1.image" width=20%><img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/178fedd47d544cacafd48b9135aa386c~tplv-k3u1fbpfcp-zoom-1.image" width=20%>
</div>


----

## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ad9bfa18881454a989500a40218f82d~tplv-k3u1fbpfcp-zoom-1.image" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/4318537403878167](https://juejin.im/user/4318537403878167)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/4318537403878167)上一起分享知识。