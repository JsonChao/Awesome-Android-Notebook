---

		title:  Android主流三方库源码分析（二、深入理解Retrofit源码）
		date: 2018/12/09 22:19:00   
		tags: 
		- Android主流三方库源码分析
		categories: 安卓主流三方库源码分析
		thumbnail: https://www.google.com.hk/url?sa=i&source=images&cd=&cad=rja&uact=8&ved=2ahUKEwjQte_t-JLfAhWJu7wKHfbpDxwQjRx6BAgBEAU&url=https%3A%2F%2Fandroid.jlelse.eu%2Fconclusion-of-my-adventures-with-retrofit-2-a252f91da740&psig=AOvVaw0i0kLPBasGhWll3FiMAlod&ust=1544451630934348
---

---

## 前言

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

前篇我们详细地分析了OKHttp的核心源码，如果对OKHttp内部机制不了解的可以看看[Android主流三方库源码分析（一、深入理解OKHttp源码）](https://jsonchao.github.io/2018/12/01/Android%E4%B8%BB%E6%B5%81%E4%B8%89%E6%96%B9%E5%BA%93%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%80%E3%80%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3OKHttp%E6%BA%90%E7%A0%81%EF%BC%89/)。这篇，将会来深入地分析下目前Android最优秀的网络封装框架Retrofit的源码流程。

### 一、基本使用流程

#### 1、定义HTTP API，用于描述请求


    public interface GitHubService {
    
         @GET("users/{user}/repos")
         Call<List<Repo>> listRepos(@Path("user") String user);
    }
    
#### 2、创建Retrofit并生成API的实现（注意：方法上面的注解表示请求的接口部分，返回类型是请求的返回值类型，方法的参数即是请求的参数）

    // 1.Retrofit构建过程
    Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();

    // 2.创建网络请求接口类实例过程
    GitHubService service = retrofit.create(GitHubService.class);

#### 3、调用API方法，生成Call，执行请求

    // 3.生成并执行请求过程
    Call<List<Repo>> repos = service.listRepos("octocat");
    repos.execute() or repos.enqueue()
    
    
Retrofit的基本使用流程很简洁，但是简洁并不代表简单，Retrofit为了实现这种简洁的使用流程，内部使用了优秀的架构设计和大量的设计模式，在我分析过Retrofit最新版的源码和大量优秀的Retrofit源码分析文章后，我发现，要想真正理解Retrofit内部的核心源码流程和设计思想，首先，需要对这九大设计模式有一定的了解，如下：

    1.Retrofit构建过程 
    建造者模式、工厂方法模式
    
    2.创建网络请求接口实例过程
    外观模式、代理模式、单例模式、策略模式、装饰模式（建造者模式）
    
    3.生成并执行请求过程
    适配器模式（代理模式、装饰模式）
    
其次，需要对OKHttp源码有一定的了解，如果不了解的可以看看这篇[Android主流三方库源码分析（一、深入理解OKHttp源码）](https://jsonchao.github.io/2018/12/01/Android%E4%B8%BB%E6%B5%81%E4%B8%89%E6%96%B9%E5%BA%93%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%80%E3%80%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3OKHttp%E6%BA%90%E7%A0%81%EF%BC%89/)。最后，让我们按以上流程去深入Retrofit源码内部，领悟它带给我们的**设计之美**。
    
### 二、Retrofit构建过程 

#### 1、Retrofit核心对象解析

首先Retrofit中有一个全局变量非常关键，在V2.5之前的版本，使用的是LinkedHashMap()，它是一个网络请求配置对象，是由网络请求接口中方法注解进行解析后得到的。


    public final class Retrofit {
    
        // 网络请求配置对象，存储网络请求相关的配置，如网络请求的方法、数据转换器、网络请求适配器、网络请求工厂、基地址等
        private final Map<Method, ServiceMethod<?>> serviceMethodCache = new ConcurrentHashMap<>();


Retrofit使用了建造者模式通过内部类Builder类建立一个Retrofit实例，如下：

    

    public static final class Builder {
    
        // 平台类型对象（Platform -> Android)
        private final Platform platform;
        // 网络请求工厂，默认使用OkHttpCall（工厂方法模式）
        private @Nullable okhttp3.Call.Factory callFactory;
        // 网络请求的url地址
        private @Nullable HttpUrl baseUrl;
        // 数据转换器工厂的集合
        private final List<Converter.Factory> converterFactories = new ArrayList<>();
        // 网络请求适配器工厂的集合，默认是ExecutorCallAdapterFactory
        private final List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>();
        // 回调方法执行器，在 Android 上默认是封装了 handler 的 MainThreadExecutor, 默认作用是：切换线程（子线程 -> 主线程）
        private @Nullable Executor callbackExecutor;
        // 一个开关，为true则会缓存创建的ServiceMethod
        private boolean validateEagerly;


#### 2、Builder内部构造

下面看看Builder内部构造做了什么。
    
    
    public static final class Builder {
    
        ...
        
        Builder(Platform platform) {
            this.platform = platform;
        }

        
        public Builder() {
            this(Platform.get());
        }
        
        ...
        
    }
    
    
    class Platform {
    
        private static final Platform PLATFORM = findPlatform();
        
        static Platform get() {
          return PLATFORM;
        }
        
        private static Platform findPlatform() {
          try {
            // 使用JVM加载类的方式判断是否是Android平台
            Class.forName("android.os.Build");
            if (Build.VERSION.SDK_INT != 0) {
              return new Android();
            }
          } catch (ClassNotFoundException ignored) {
          }
          try {
            // 同时支持Java平台
            Class.forName("java.util.Optional");
            return new Java8();
          } catch (ClassNotFoundException ignored) {
          }
          return new Platform();
        }
    
    static class Android extends Platform {
    
        ...
    
        
        @Override public Executor defaultCallbackExecutor() {
            //切换线程（子线程 -> 主线程）
            return new MainThreadExecutor();
        }
    
        // 创建默认的网络请求适配器工厂，如果是Android7.0或Java8上，则使
        // 用了并发包中的CompletableFuture保证了回调的同步
        // 在Retrofit中提供了四种CallAdapterFactory(策略模式)：
        // ExecutorCallAdapterFactory（默认）、GuavaCallAdapterFactory、
        // va8CallAdapterFactory、RxJavaCallAdapterFactory
        @Override List<? extends CallAdapter.Factory> defaultCallAdapterFactories(
            @Nullable Executor callbackExecutor) {
          if (callbackExecutor == null) throw new AssertionError();
          ExecutorCallAdapterFactory executorFactory = new   ExecutorCallAdapterFactory(callbackExecutor);
          return Build.VERSION.SDK_INT >= 24
            ? asList(CompletableFutureCallAdapterFactory.INSTANCE, executorFactory)
            : singletonList(executorFactory);
        }
        
        ...
    
        @Override List<? extends Converter.Factory> defaultConverterFactories() {
          return Build.VERSION.SDK_INT >= 24
              ? singletonList(OptionalConverterFactory.INSTANCE)
              : Collections.<Converter.Factory>emptyList();
        }
    
        ...
        
        static class MainThreadExecutor implements Executor {
        
            // 获取Android 主线程的Handler 
            private final Handler handler = new Handler(Looper.getMainLooper());
    
            @Override public void execute(Runnable r) {
            
                // 在UI线程对网络请求返回数据处理
                handler.post(r);
            }
        }
    }
      
可以看到，在Builder内部构造时设置了默认Platform、callAdapterFactories和callbackExecutor。

#### 3、添加baseUrl

很简单，就是将String类型的url转换为OkHttp的HttpUrl过程如下：

    /**
     * Set the API base URL.
     *
     * @see #baseUrl(HttpUrl)
     */
    public Builder baseUrl(String baseUrl) {
        checkNotNull(baseUrl, "baseUrl == null");
        return baseUrl(HttpUrl.get(baseUrl));
    }

    public Builder baseUrl(HttpUrl baseUrl) {
        checkNotNull(baseUrl, "baseUrl == null");
        List<String> pathSegments = baseUrl.pathSegments();
        if (!"".equals(pathSegments.get(pathSegments.size() - 1))) {
          throw new IllegalArgumentException("baseUrl must end in /: " + baseUrl);
        }
        this.baseUrl = baseUrl;
        return this;
    }
        
#### 4、添加GsonConverterFactory

首先，看到GsonConverterFactory.creat()的源码。

    public final class GsonConverterFactory extends Converter.Factory {
     
        public static GsonConverterFactory create() {
            return create(new Gson());
        }
        
        
        public static GsonConverterFactory create(Gson gson) {
            if (gson == null) throw new NullPointerException("gson ==   null");
            return new GsonConverterFactory(gson);
        }
        
        private final Gson gson;
        
        // 创建了一个含有Gson对象实例的GsonConverterFactory
        private GsonConverterFactory(Gson gson) {
            this.gson = gson;
        }

然后，看看addConverterFactory()方法内部。

    public Builder addConverterFactory(Converter.Factory factory) {
        converterFactories.add(checkNotNull(factory, "factory null"));
        return this;
    }
    
可知，这一步是将一个含有Gson对象实例的GsonConverterFactory放入到了数据转换器工厂converterFactories里。


#### 5、build过程

    public Retrofit build() {
    
        if (baseUrl == null) {
          throw new IllegalStateException("Base URL required.");
        }
        
        okhttp3.Call.Factory callFactory = this.callFactory;
        if (callFactory == null) {
            // 默认使用okhttp
             callFactory = new OkHttpClient();
        }
        
        Executor callbackExecutor = this.callbackExecutor;
        if (callbackExecutor == null) {
            // Android默认的callbackExecutor
            callbackExecutor = platform.defaultCallbackExecutor();
        }
        
        // Make a defensive copy of the adapters and add the defaultCall adapter.
        List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
        // 添加默认适配器工厂在集合尾部
        callAdapterFactories.addAll(platform.defaultCallAdapterFactorisca  llbackExecutor));
        
        // Make a defensive copy of the converters.
        List<Converter.Factory> converterFactories = new ArrayList<>(
            1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());
        // Add the built-in converter factory first. This prevents overriding its behavior but also
        // ensures correct behavior when using converters thatconsumeall types.
        converterFactories.add(new BuiltInConverters());
        converterFactories.addAll(this.converterFactories);
        converterFactories.addAll(platform.defaultConverterFactories();
        
        return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
            unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
            
    }
    
可以看到，最终我们在Builder类中看到的6大核心对象都已经配置到Retrofit对象中了。


### 三、创建网络请求接口实例过程

retrofit.create()使用了外观模式和代理模式创建了网络请求的接口实例，我们分析下create方法。

    public <T> T create(final Class<T> service) {
        Utils.validateServiceInterface(service);
        if (validateEagerly) {
            // 判断是否需要提前缓存ServiceMethod对象
            eagerlyValidateMethods(service);
        }
        
        // 使用动态代理拿到请求接口所有注解配置后，创建网络请求接口实例
        return (T) Proxy.newProxyInstance(service.getClassLoader(), new  Class<?>[] { service },
            new InvocationHandler() {
              private final Platform platform = Platform.get();
              private final Object[] emptyArgs = new Object[0];
    
              @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
                  throws Throwable {
                // If the method is a method from Object then defer to normal invocation.
                if (method.getDeclaringClass() == Object.class) {
                  return method.invoke(this, args);
                }
                if (platform.isDefaultMethod(method)) {
                  return platform.invokeDefaultMethod(method, service, proxy, args);
                }
                return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
              }
        });
     }
    
    private void eagerlyValidateMethods(Class<?> service) {
    
      Platform platform = Platform.get();
      for (Method method : service.getDeclaredMethods()) {
        if (!platform.isDefaultMethod(method)) {
          loadServiceMethod(method);
        }
      }
    }
    
继续看看loadServiceMethod的内部流程
    
    ServiceMethod<?> loadServiceMethod(Method method) {
    
        ServiceMethod<?> result = serviceMethodCache.get(method);
        if (result != null) return result;
    
        synchronized (serviceMethodCache) {
          result = serviceMethodCache.get(method);
          if (result == null) {
                // 解析注解配置得到了ServiceMethod
                result = ServiceMethod.parseAnnotations(this, method);
                // 可以看到，最终加入到ConcurrentHashMap缓存中
                serviceMethodCache.put(method, result);
          }
        }
        return result;
    }
    
    
    abstract class ServiceMethod<T> {
      static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method   method) {
            // 通过RequestFactory解析注解配置（工厂模式、内部使用了建造者模式）
            RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
        
            Type returnType = method.getGenericReturnType();
            if (Utils.hasUnresolvableType(returnType)) {
              throw methodError(method,
                  "Method return type must not include a type variable or wildcard: %s", returnType);
            }
            if (returnType == void.class) {
              throw methodError(method, "Service methods cannot return void.");
            }
        
            // 最终是通过HttpServiceMethod构建的请求方法
            return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
        }

        abstract T invoke(Object[] args);
    }
    
#### 以下为请求构造核心流程 
    
根据RequestFactory#Builder构造方法和parseAnnotations方法的源码，可知的它的作用就是用来解析注解配置的。

    Builder(Retrofit retrofit, Method method) {
        this.retrofit = retrofit;
        this.method = method;
        // 获取网络请求接口方法里的注释
        this.methodAnnotations = method.getAnnotations();
        // 获取网络请求接口方法里的参数类型       
        this.parameterTypes = method.getGenericParameterTypes();
        // 获取网络请求接口方法里的注解内容    
        this.parameterAnnotationsArray = method.getParameterAnnotations();
    }
    
接着看HttpServiceMethod.parseAnnotations()的内部流程。

    static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
          Retrofit retrofit, Method method, RequestFactory requestFactory) {
            
        //1.根据网络请求接口方法的返回值和注解类型，
        // 从Retrofit对象中获取对应的网络请求适配器
        CallAdapter<ResponseT, ReturnT> callAdapter = createCallAdapter(retrofit,method);
        
        // 得到响应类型
        Type responseType = callAdapter.responseType();
        
        ...
    
        //2.根据网络请求接口方法的返回值和注解类型从Retrofit对象中获取对应的数据转换器 
        Converter<ResponseBody, ResponseT>responseConverter =
            createResponseConverter(retrofit,method, responseType);
    
        okhttp3.Call.Factory callFactory = retrofit.callFactory;
        
        return newHttpServiceMethod<>(requestFactory, callFactory, callAdapter,responseConverter);
    }
    
##### 1.createCallAdapter(retrofit, method)
      
    private static <ResponseT, ReturnT> CallAdapter<ResponseT, ReturnT>     createCallAdapter(
          Retrofit retrofit, Method method) {
          
        // 获取网络请求接口里方法的返回值类型
        Type returnType = method.getGenericReturnType();
        
        // 获取网络请求接口接口里的注解
        Annotation[] annotations = method.getAnnotations();
        try {
          //noinspection unchecked
          return (CallAdapter<ResponseT, ReturnT>)  retrofit.callAdapter(returnType, annotations);
        } catch (RuntimeException e) { // Wide exception range because factories are user code.
          throw methodError(method, e, "Unable to create call adapter for %s", returnType);
        }
    }
      
    public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
        return nextCallAdapter(null, returnType, annotations);
    }
  
    public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
        ...

        int start = callAdapterFactories.indexOf(skipPast) + 1;
        // 遍历 CallAdapter.Factory 集合寻找合适的工厂
        for (int i = start, count = callAdapterFactories.size(); i <count; i++) {
            CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
            if (adapter != null) {
              return adapter;
            }
        }
    }
      
##### 2.createResponseConverter(Retrofit retrofit, Method method, Type responseType)     
    
     private static <ResponseT> Converter<ResponseBody, ResponseT>  createResponseConverter(
         Retrofit retrofit, Method method, Type responseType) {
       Annotation[] annotations = method.getAnnotations();
       try {
         return retrofit.responseBodyConverter(responseType,annotations);
       } catch (RuntimeException e) { // Wide exception range because    factories are user code.
         throw methodError(method, e, "Unable to create converter for%s",   responseType);
       }
    }
    
    public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
        return nextResponseBodyConverter(null, type, annotations);
    }
    
    public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
      @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
    ...
    
    int start = converterFactories.indexOf(skipPast) + 1;
    // 遍历 Converter.Factory 集合并寻找合适的工厂, 这里是GsonResponseBodyConverter
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);
      if (converter != null) {
        //noinspection unchecked
        return (Converter<ResponseBody, T>) converter;
      }
    }

#### 最终，执行HttpServiceMethod的invoke方法

    @Override ReturnT invoke(Object[] args) {
        return callAdapter.adapt(
            new OkHttpCall<>(requestFactory, args, callFactory, responseConverter));
    }
    
最终在adapt中创建了一个ExecutorCallbackCall对象，它是一个装饰者，而在它内部真正去执行网络请求的还是OkHttpCall。


### 四、创建网络请求接口类实例并执行请求过程

#### 1、service.listRepos()

    1、Call<List<Repo>> repos = service.listRepos("octocat");

service对象是动态代理对象Proxy.newProxyInstance()，当调用getCall()时会被
它拦截，然后调用自身的InvocationHandler#invoke()，得到最终的Call对象。

#### 2、同步执行流程 repos.execute()
    

    @Override public Response<T> execute() throws IOException {
        okhttp3.Call call;
    
        synchronized (this) {
          if (executed) throw new IllegalStateException("Already executed.");
          executed = true;
    
          if (creationFailure != null) {
            if (creationFailure instanceof IOException) {
              throw (IOException) creationFailure;
            } else if (creationFailure instanceof RuntimeException) {
              throw (RuntimeException) creationFailure;
            } else {
              throw (Error) creationFailure;
            }
          }
    
          call = rawCall;
          if (call == null) {
            try {
              // 创建一个OkHttp的Request对象请求
              call = rawCall = createRawCall();
            } catch (IOException | RuntimeException | Error e) {
              throwIfFatal(e); //  Do not assign a fatal error to     creationFailure.
              creationFailure = e;
              throw e;
            }
          }
        }
    
        if (canceled) {
          call.cancel();
        }
    
        // 调用OkHttpCall的execute()发送网络请求（同步），
        // 并解析网络请求返回的数据
        return parseResponse(call.execute());
    }


    private okhttp3.Call createRawCall() throws IOException {
        // 创建 一个okhttp3.Request
        okhttp3.Call call =
        callFactory.newCall(requestFactory.create(args));
        if (call == null) {
          throw new NullPointerException("Call.Factory returned null.");
        }
        return call;
    }
    

    Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
        ResponseBody rawBody = rawResponse.body(); 
        
        // Remove the body's source (the only stateful object) so we can   pass the response along.
        rawResponse = rawResponse.newBuilder()
            .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
            .build();    
        
        // 根据响应返回的状态码进行处理    
        int code = rawResponse.code();
        if (code < 200 || code >= 300) {
          try {
            // Buffer the entire body to avoid future I/O.
            ResponseBody bufferedBody = Utils.buffer(rawBody);
            return Response.error(bufferedBody, rawResponse);
          } finally {
            rawBody.close();
          }
        }    
        if (code == 204 || code == 205) {
          rawBody.close();
          return Response.success(null, rawResponse);
        }    
        
        
        ExceptionCatchingResponseBody catchingBody = new ExceptionCatchingResponseBody(rawBody);
        try {
          // 将响应体转为Java对象
          T body = responseConverter.convert(catchingBody);
          
          return Response.success(body, rawResponse);
        } catch (RuntimeException e) {
          // If the underlying source threw an exception, propagate that     rather than indicating it was
          // a runtime exception.
          catchingBody.throwIfCaught();
          throw e;
        }
    }
    

#### 3、异步请求流程 reponse.enqueque


    @Override 
    public void enqueue(final Callback<T> callback) {
    
        // 使用静态代理 delegate进行异步请求 
        delegate.enqueue(new Callback<T>() {
    
          @Override 
          public void onResponse(Call<T> call, finalResponse<T>response) {
            // 线程切换，在主线程显示结果
            callbackExecutor.execute(new Runnable() {
                @Override 
                 public void run() {
                if (delegate.isCanceled()) {
                  callback.onFailure(ExecutorCallbackCall.this, newIOException("Canceled"));
                } else {
                  callback.onResponse(ExecutorCallbackCall.this,respons);
                }
              }
            });
          }
          @Override 
          public void onFailure(Call<T> call, final Throwable t) {
            callbackExecutor.execute(new Runnable() {
              @Override public void run() {
                callback.onFailure(ExecutorCallbackCall.this, t);
              }
            });
          }
        });
    }
    
看看 delegate.enqueue 内部流程。
 
    @Override 
    public void enqueue(final Callback<T> callback) {
       
        okhttp3.Call call;
        Throwable failure;
    
        synchronized (this) {
          if (executed) throw new IllegalStateException("Already executed.");
          executed = true;
    
          call = rawCall;
          failure = creationFailure;
          if (call == null && failure == null) {
            try {
              // 创建OkHttp的Request对象，再封装成OkHttp.call
              // 方法同发送同步请求，此处上面已分析
              call = rawCall = createRawCall(); 
            } catch (Throwable t) {
              failure = creationFailure = t;
            }
          }

    @Override public void enqueue(final Callback<T> callback) {
      checkNotNull(callback, "callback == null");
    
      okhttp3.Call call;
      Throwable failure;
    
      ...
    
      call.enqueue(new okhttp3.Callback() {
        @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
          Response<T> response;
          try {
            // 此处上面已分析
            response = parseResponse(rawResponse);
          } catch (Throwable e) {
            throwIfFatal(e);
            callFailure(e);
            return;
          }
    
          try {
            callback.onResponse(OkHttpCall.this, response);
          } catch (Throwable t) {
            t.printStackTrace();
          }
        }
    
        @Override public void onFailure(okhttp3.Call call, IOException e) {
          callFailure(e);
        }
    
        private void callFailure(Throwable e) {
          try {
            callback.onFailure(OkHttpCall.this, e);
          } catch (Throwable t) {
            t.printStackTrace();
          }
        }
      });
    }

如果你看到这里的话，恭喜你，你已经对Retrofit已经有一个比较深入的了解了，但是，笔者还是要建议大家自己主动配合着Retrofit最新版的源码一步步去彻底地认识它，只有这样，你才能看到它真实的内心，最后附上一张Stay的Retrofit源码流程图，要注意的是，这是V2.5之前版本的流程，但是，在看完上面的源码分析后，我们知道，主体流程是没有变化的。

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7ab58d6242cc4fae96cd73eb198e325a~tplv-k3u1fbpfcp-zoom-1.image)

### 五、总结

从本质上来说，Retrofit虽然只是一个RESTful 的HTTP 网络请求框架的封装库。但是，它内部通过 大量的设计模式 封装了 OkHttp，让使用者感到它非常简洁、易懂。它内部主要是用动态代理的方式，动态将网络请求接口的注解解析成HTTP请求，最后执行请求的过程。好了，至此，我们的Android主流三方库源码分析的网络库分析部分已经完毕。接下来，将为大家带来最流行的图片加载框架Glide的源码分析，敬请期待~


# 公众号

我的公众号 `JsonChao` 开通啦，如果您想第一时间获取最新文章和最新动态，欢迎扫描关注~

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/836b3b038d69418789e76ecce02f6041~tplv-k3u1fbpfcp-zoom-1.image)


##### 参考链接：
---
1、Retrofit V2.5.0源码

2、Android进阶之光

3、[Android：手把手带你 深入读懂 Retrofit 2.0 源码](https://www.jianshu.com/p/0c055ad46b6c)

4、[Retrofit分析-漂亮的解耦套路](https://www.jianshu.com/p/45cb536be2f4)


## 赞赏

如果这个库对您有很大帮助，您愿意支持这个项目的进一步开发和这个项目的持续维护。你可以扫描下面的二维码，让我喝一杯咖啡或啤酒。非常感谢您的捐赠。谢谢！

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9fc0ff8f71614ff1b8a35b7be13d916f~tplv-k3u1fbpfcp-zoom-1.image" width=20%><img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2964f267f3734915a365fef33680dbea~tplv-k3u1fbpfcp-zoom-1.image" width=20%>
</div>


----

## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d60a8690b8594a9c8ff7c920035c6ef7~tplv-k3u1fbpfcp-zoom-1.image" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/4318537403878167](https://juejin.im/user/4318537403878167)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/4318537403878167)上一起分享知识。
