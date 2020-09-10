---

		title:  Android主流三方库源码分析（八、深入理解Dagger2源码）
		date: 2019/01/20 23:17:00   
		tags: 
		- Android主流三方库源码分析
		categories: 安卓主流三方库源码分析
		thumbnail: https://cdn-images-1.medium.com/max/1200/1*hlFGgtG2Aa1qsxtDRb0PlA.jpeg
---

---

## 前言

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

上一篇，笔者详细地分析了Android中的依赖注入框架ButterKnife，使用它帮助我们解决了重复编写findViewById和setOnclickListener的繁琐。众所周知，当项目越来越大时，类之间的调用层次会越来越深，并且有些类是Activity/Fragment，有些是单例，而且**它们的生命周期也不是一致的**，所以创建这些对象时要处理的各个对象的依赖关系和生命周期时的任务会很繁重，因此，为了解决这个问题Dagger2应运而生。**相比ButterKnife的轻量级使用，Dagger2会显得更重量级和锋利一些，它能够掌控全局，对项目中几乎所有的依赖进行集成管理**。如果有对Binder架构体系比较了解的朋友应该知道，其中的服务大管家ServiceManager负责所有的服务（引导服务、核心服务、其它服务）的管理，而Dagger2其实就是将项目中的依赖进行了集成管理。下面，笔者来跟大家一起探索Dagger2的内部实现机制，看看它是如何进行依赖管理的。

Dagger2其实同RxJava一样，是一种多平台通用的库。由于Dagger2的通用写法比较繁琐，因此，Google推出了适用于Android平台的Dagger.Android用法。本文，将基于Dagger.Android的源码对Dagger2内部的实现机制进行探索。


# 一、预备知识


鉴于Dagger有一定的上手成本，这里首先带大家复习一下本篇源码分析可能会涉及到的相关基础知识点，以此降低阅读难度。


## 1、@Inject


告诉dagger这个字段或类需要依赖注入，然后在需要依赖的地方使用这个注解，dagger会自动生成这个构造器的实例。

### 获取所需依赖：

- 全局变量注入
- 方法注入

### 提供所需实例：

- 构造器注入（如果有多个构造函数，只能注解一个，否则编译报错）


## 2、@Module


类注解，表示此类的方法是提供依赖的，它告诉dagger在哪可以找到依赖。用于不能用@Inject提供依赖的地方，如第三方库提供的类，基本数据类型等不能修改源码的情况。

注意：**Dagger2会优先在@Module注解的类上查找依赖，没有的情况才会去查询类的@Inject构造方法**


## 3、@Singleton


声明这是一个单例，**在确保只有一个Component并且不再重新build()之后，对象只会被初始化一次，之后的每次都会被注入相同的对象**，它就是一个内置的作用域。

对于@Singleton，大家可能会产生一些误解，这里详细阐述下：

- Singleton容易给人造成一种误解就是用Singleton注解后在整个Java代码中都是单例，但**实际上他和Scope一样，只是在同一个Component是单例**。也就是说，如果重新调用了component的build（）方法，即使使用了Singleton注解了，但仍然获取的是不同的对象。
- 它表明了**@Singleton注解只是声明了这是一个单例，为的只是提高代码可读性，其实真正控制对象生命周期的还是Component**。同理，自定义的@ActivityScope 、@ApplicationScope也仅仅是一个声明的作用，**真正控制对象生命周期的还是Component**。


## 4、@Providers


只在@Module中使用，用于提供构造好的实例。一般与@Singleton搭配，用单例方法的形式对外提供依赖,是一种替代@Inject注解构造方法的方式。

注意：

- 使用了@Providers的方法应使用provide作为前缀，使用了@Module的类应使用Module作为后缀。
- **如果@Providers方法或@Inject构造方法有参数，要保证它能够被dagger获取到，比如通过其它@Providers方法或者@Inject注解构造器的形式得到**。


## 5、@Component


**@Component作为Dagger2的容器总管，它拥有着@Inject与@Module的所有依赖。同时，它也是一枚注射器，用于获取所需依赖和提供所需依赖的桥梁**。这里的桥梁即指@Inject和@Module（或@Inject构造方法）之间的桥梁。定义时需要列出响应的Module组成，此外，还可以使用dependencies继承父Component。


### Component与Module的区别：

Component既是注射器也是一个容器总管，而module则是作为容器总管Component的子容器，实质是一个用于提供依赖的模块。


## 6、@Scope


注解作用域，通过自定义注解**限定对象作用范围，增强可读性**。


@Scope有两种常用的使用场景：

- **模拟Singleton代表全局单例，与Component生命周期关联**。
- **模拟局部单例，如登录到退出登录期间**。 


## 7、@Qualifier


限定符，利用它**定义注解类以用于区分类的不同实例**。例如：2个方法返回不同的Person对象，比如说小明和小华，为了区分，使用@Qualifier定义的注解类。


## 8、dependencies 


使用它表示ChildComponent依赖于FatherComponent，如下所示：

    @Component(modules = ChildModule.class, dependencies = FatherComponent.class)
    public interface ChildComponent {
        ...
    }
    
    
## 9、@SubComponent


表示是一个子@Component，它能**将应用的不同部分封装起来，用来替代@Dependencies**。


回顾完Dagger2的基础知识，下面我们要启动发动机了。

# 二、简单示例（取自[AwesomeWanAndroid](https://github.com/JsonChao/Awesome-WanAndroid)）

## 1、首先，创建一个BaseActivityComponent的Subcomponent：


    @Subcomponent(modules = {AndroidInjectionModule.class})
    public interface BaseActivityComponent extends AndroidInjector<BaseActivity> {
    
        @Subcomponent.Builder
        abstract class BaseBuilder extends AndroidInjector.Builder<BaseActivity>{
        }
    }
    
    
这里必须要注解成@Subcomponent.Builder表示是顶级@Subcomponent的内部类。AndroidInjector.Builder的泛型指定了BaseActivity，即表示每一个继承于BaseActivity的Activity都继承于同一个子组件（BaseActivityComponent）。


## 2、然后，创建一个将会导入Subcomponent的公有Module。

    // 1
    @Module(subcomponents = {BaseActivityComponent.class})
    public abstract class AbstractAllActivityModule {
    
        @ContributesAndroidInjector(modules = MainActivityModule.class)
        abstract MainActivity contributesMainActivityInjector();
    
        @ContributesAndroidInjector(modules = SplashActivityModule.class)
        abstract SplashActivity contributesSplashActivityInjector();
        
        // 一系列的对应Activity的contributesxxxActivityInjector
        ...
        
    }
    
在注释1处用subcomponents来表示开放全部依赖给AbstractAllActivityModule，使用Subcomponent的重要原因是它将应用的不同部分封装起来了。**@AppComponent负责维护共享的数据和对象，而不同处则由各自的@Subcomponent维护**。

    
## 3、接着，配置项目的Application。


    public class WanAndroidApp extends Application implements HasActivityInjector {

        // 3
        @Inject
        DispatchingAndroidInjector<Activity> mAndroidInjector;

        private static volatile AppComponent appComponent;
        
        @Override
        public void onCreate() {
            super.onCreate();
            
            ...
            // 1
            appComponent = DaggerAppComponent.builder()
                .build();
            // 2
            appComponent.inject(this);
            
            ...
            
        }
        
        ...
        
        // 4
        @Override
        public AndroidInjector<Activity> activityInjector() {
            return mAndroidInjector;
        }
    }
    
        
首先，在注释1处，使用AppModule模块和httpModule模块构建出AppComponent的实现类DaggerAppComponent。这里看一下AppComponent的配置代码：


    @Singleton
    @Component(modules = {AndroidInjectionModule.class,
            AndroidSupportInjectionModule.class,
            AbstractAllActivityModule.class,
            AbstractAllFragmentModule.class,
            AbstractAllDialogFragmentModule.class}
        )
    public interface AppComponent {
    
        /**
         * 注入WanAndroidApp实例
         *
         * @param wanAndroidApp WanAndroidApp
         */
        void inject(WanAndroidApp wanAndroidApp);
        
        ...
        
    }
    
        
可以看到，AppComponent依赖了AndroidInjectionModule模块，它包含了一些基础配置的绑定设置，如activityInjectorFactories、fragmentInjectorFactories等等，而AndroidSupportInjectionModule模块显然就是多了一个supportFragmentInjectorFactories的绑定设置，activityInjectorFactories的内容如所示：


    @Beta
    @Module
    public abstract class AndroidInjectionModule {
        @Multibinds
        abstract Map<Class<? extends Activity>, AndroidInjector.Factory<? extends Activity>>
            activityInjectorFactories();
        
        @Multibinds
        abstract Map<Class<? extends Fragment>, AndroidInjector.Factory<? extends Fragment>>
            fragmentInjectorFactories();
        
        ...
    
    }
    

接着，下面依赖的AbstractAllActivityModule、
AbstractAllFragmentModule、AbstractAllDialogFragmentModule则是为项目的所有Activity、Fragment、DialogFragment提供的统一基类抽象Module，这里看下AbstractAllActivityModule的配置：


    @Module(subcomponents = {BaseActivityComponent.class})
    public abstract class AbstractAllActivityModule {
    
        @ContributesAndroidInjector(modules = MainActivityModule.class)
        abstract MainActivity contributesMainActivityInjector();
    
        @ContributesAndroidInjector(modules = SplashActivityModule.class)
        abstract SplashActivity contributesSplashActivityInjector();
        
        ...
        
    }


可以看到，项目下的所有xxxActiviity都有对应的contributesxxxActivityInjector()方法提供实例注入。并且，注意到AbstractAllActivityModule这个模块依赖的
subcomponents为BaseActivityComponent，前面说过了，每一个继承于BaseActivity的Activity都继承于BaseActivityComponent这一个subcomponents。同理，AbstractAllFragmentModule与AbstractAllDialogFragmentModule也是类似的实现模式，如下所示：


    // 1
    @Module(c = BaseFragmentComponent.class)
    public abstract class AbstractAllFragmentModule {
    
        @ContributesAndroidInjector(modules = CollectFragmentModule.class)
        abstract CollectFragment contributesCollectFragmentInject();
    
        @ContributesAndroidInjector(modules = KnowledgeFragmentModule.class)
        abstract KnowledgeHierarchyFragment contributesKnowledgeHierarchyFragmentInject();
        
        ...
        
    }
    
    
    // 2
    @Module(subcomponents = BaseDialogFragmentComponent.class)
    public abstract class AbstractAllDialogFragmentModule {
    
        @ContributesAndroidInjector(modules = SearchDialogFragmentModule.class)
        abstract SearchDialogFragment contributesSearchDialogFragmentInject();
    
        @ContributesAndroidInjector(modules = UsageDialogFragmentModule.class)
        abstract UsageDialogFragment contributesUsageDialogFragmentInject();
    
    }

注意到注释1和注释2处的代码，AbstractAllFragmentModule和AbstractAllDialogFragmentModule的subcomponents为BaseFragmentComponent、BaseDialogFragmentComponent，很显然，同AbstractAllActivityModule的子组件BaseActivityComponent一样，它们都是作为一个通用的子组件。
    
然后，回到我们配置项目下的Application下面的注释2处的代码，在这里使用了第一步Dagger为我们构建的DaggerAppComponent对象将当期的Application实例注入了进去，交给了Dagger这个依赖大管家去管理。最终，**Dagger2内部创建的mAndroidInjector对象会在注释3处的地方进行实例赋值。在注释4处，实现HasActivityInjector接口，重写activityInjector()方法，将我们上面得到的mAndroidInjector对象返回**。这里的mAndroidInjector是一个类型为DispatchingAndroidInjector<Activity>的对象，可以这样理解它：它能够执行Android框架下的核心成员如Activity、Fragment的成员注入，在我们项目下的Application中将DispatchingAndroidInjector的泛型指定为Activity就说明它承担起了所有Activity成员依赖的注入。那么，如何指定某一个Activity能被纳入DispatchingAndroidInjector这个所有Activity的依赖总管的口袋中呢？接着看使用步骤4。


## 4、最后，将目标Activity纳入Activity依赖分配总管DispatchingAndroidInjector的囊中。


很简单，只需在目标Activity的onCreate()方法前的super.onCreate(savedInstanceState)前配置一行代码 AndroidInjection.inject(this)，如下所示：

    public abstract class BaseActivity<T extends AbstractPresenter> extends AbstractSimpleActivity implements
        AbstractView {

        ...
        @Inject
        protected T mPresenter;
        
        
        @Override
        protected void onCreate(@Nullable Bundle savedInstanceState) {
            AndroidInjection.inject(this);
            super.onCreate(savedInstanceState);
        }
    
        ...
        
    }
    

这里使用了@Inject表明了需要注入mPresenter实例，然后，我们需要在具体的Presenter类的构造方法上使用@Inject提供基于当前构造方法的mPresenter实例，如下所示：


    public class MainPresenter extends BasePresenter<MainContract.View> implements MainContract.Presenter {

        ...

        @Inject
        MainPresenter(DataManager dataManager) {
            super(dataManager);
            this.mDataManager = dataManager;
        }

        ...
        
    }
    

从上面的使用流程中，有三个关键的核心实现是我们需要了解的，如下所示：


- 1、appComponent = DaggerAppComponent.builder().build()这句代码如何构建出DaggerAPPComponent的？

- 2、appComponent.inject(this)是如何将mAndroidInjector实例赋值给当前的Application的？

- 3、在目标Activity下的AndroidInjection.inject(this)这句代码是如何将当前Activity对象纳入依赖分配总管DispatchingAndroidInjector囊中的呢？


下面，让我们来逐个一一地来探索其中的奥妙吧~


# 三、DaggerAppComponent.builder().build()是如何构建出DaggerAPPComponent的？


首先，我们看到DaggerAppComponent的builder()方法：


    public static Builder builder() {
        return new Builder();
    }
    
    
里面直接返回了一个新建的Builder静态内部类对象，看看它的构造方法中做了什么：


    public static final class Builder {

        private Builder() {}
        
        ...
        
    }
    

看来，Builder的默认构造方法什么也没有做，那么，真正的实现肯定在Builder对象的build()方法中，接着看到build()方法。


    public static final class Builder {
    
        ...
     
        public AppComponent build() {
             return new DaggerAppComponent(this);
        }
    
        ...

    }


在Builder的build()方法中直接返回了新建的DaggerAppComponent对象。下面，看看DaggerAppComponent的构造方法:

    
    private DaggerAppComponent(Builder builder) {
        initialize(builder);
    }
    
在DaggerAppComponent的构造方法中调用了initialize方法，顾名思义，它就是真正初始化项目全局依赖配置的地方了，下面，来看看它内部的实现：


    private void initialize(final Builder builder) {
        // 1
        this.mainActivitySubcomponentBuilderProvider =
            new Provider<
                AbstractAllActivityModule_ContributesMainActivityInjector.MainActivitySubcomponent
                    .Builder>() {
            @Override
            public AbstractAllActivityModule_ContributesMainActivityInjector.MainActivitySubcomponent
                    .Builder
                get() {
                    // 2
                    return new MainActivitySubcomponentBuilder();
                }
            };

        // 一系列xxxActivitySubcomponentBuilderProvider的创建赋值代码块
        ...

    }
    
    
在注释1处，新建了一个mainActivit的子组件构造器实例提供者Provider。在注释2处，使用匿名内部类的方式重写了该Provider的get()方法，返回一个新创建好的MainActivitySubcomponentBuilder对象。很显然，它就是负责创建管理MAinActivity中所需依赖的Subcomponent建造者。接下来我们重点来分析下MainActivitySubcomponentBuilder这个类的作用。

    // 1
    private final class MainActivitySubcomponentBuilder
      extends AbstractAllActivityModule_ContributesMainActivityInjector.MainActivitySubcomponent
          .Builder {
        private MainActivity seedInstance;
        
        @Override
        public AbstractAllActivityModule_ContributesMainActivityInjector.MainActivitySubcomponent
            build() {
          if (seedInstance == null) {
            throw new IllegalStateException(MainActivity.class.getCanonicalName() + " must be set");
          }
          // 2
          return new MainActivitySubcomponentImpl(this);
        }
        
        @Override
        public void seedInstance(MainActivity arg0) {
          // 3
          this.seedInstance = Preconditions.checkNotNull(arg0);
        }
    }


首先，在注释1处，MainActivitySubcomponentBuilder继承了AbstractAllActivityModule_ContributesMainActivityInjector内部的子组件MainActivitySubcomponent的内部的子组件建造者类Builder，如下所示：


    @Subcomponent(modules = MainActivityModule.class)
    public interface MainActivitySubcomponent extends AndroidInjector<MainActivity> {
        @Subcomponent.Builder
        abstract class Builder extends
        AndroidInjector.Builder<MainActivity> {}
    }


可以看到，这个子组件建造者Builder又继承了AndroidInjector的抽象内部类Builder<MAinActivity>，那么，这个AndroidInjector到底是什么呢？

顾名思义，**AndroidInjector**是一个Android注射器，它**为每一个具体的子类型，即核心Android类型Activity和Fragment执行成员注入。**

接下来我们便来分析下AndroidInjector的内部实现，源码如下所示：


    public interface AndroidInjector<T> {
    
        void inject(T instance);
        
        // 1
        interface Factory<T> {
            AndroidInjector<T> create(T instance);
        }
        
        // 2
        abstract class Builder<T> implements AndroidInjector.Factory<T> {
            @Override
            public final AndroidInjector<T> create(T instance) {
                seedInstance(instance);
                return build();
            }
        
            @BindsInstance
            public abstract void seedInstance(T instance);
        
            public abstract AndroidInjector<T> build();
        }
    }

在注释1处，使用了抽象工厂模式，用来创建一个具体的Activity或Fragment类型的AndroidInjector实例。注释2处，Builder<T>实现了AndroidInjector.Factory<T>，它是一种Subcomponent.Builder的通用实现模式，在重写的create()方法中，进行了实例保存seedInstance()和具体Android核心类型的构建。


接着，我们回到MainActivitySubcomponentBuilder类，可以看到，它实现了AndroidInjector.Builder的seedInstance()和build()方法。在注释3处首先播种了MainActivity的实例，然后
在注释2处新建了一个MainActivitySubcomponentImpl对象返回。我们看看MainActivitySubcomponentImpl这个类是如何将mPresenter依赖注入的，相关源码如下：


    private final class MainActivitySubcomponentImpl
        implements AbstractAllActivityModule_ContributesMainActivityInjector
        .MainActivitySubcomponent {
          
        private MainPresenter getMainPresenter() {
            // 2
            return MainPresenter_Factory.newMainPresenter(
            DaggerAppComponent.this.provideDataManagerProvider.get());
        }

        @Override
        public void inject(MainActivity arg0) {
            // 1
            injectMainActivity(arg0);
        }

        private MainActivity injectMainActivity(MainActivity instance) {
            // 3
            BaseActivity_MembersInjector
            .injectMPresenter(instance, getMainPresenter());
            return instance;
        }


在注释1处，MainActivitySubcomponentImpl实现了AndroidInjector接口的inject()方法，**在injectMainActivity()首先调用getMainPresenter()方法从MainPresenter_Factory工厂类中新建了一个MainPresenter对象**。我们看看MainPresenter的newMainPresenter()方法：

    
    public static MainPresenter newMainPresenter(DataManager dataManager) {
        return new MainPresenter(dataManager);
    }


这里直接新建了一个MainPresenter。然后我们回到MainActivitySubcomponentImpl类的注释3处，继续调用了**BaseActivity_MembersInjector的injectMPresenter()方法**，顾名思义，可以猜到，它是BaseActivity的成员注射器，继续看看injectMPresenter()内部：


    public static <T extends AbstractPresenter> void injectMPresenter(
      BaseActivity<T> instance, T mPresenter) {
        instance.mPresenter = mPresenter;
    }
    
    
可以看到，这里直接将需要的mPresenter实例赋值给了BaseActivity的mPresenter，当然，这里其实是指的BaseActivity的子类MainActivity，其它的xxxActivity的依赖管理机制都是如此。


# 四、appComponent.inject(this)是如何将mAndroidInjector实例赋值给当前的Application的？


我们继续查看appComponent的inject()方法：


    @Override
    public void inject(WanAndroidApp wanAndroidApp) {
      injectWanAndroidApp(wanAndroidApp);
    }


在inject()方法里调用了injectWanAndroidApp()，继续查看injectWanAndroidApp()方法：


    private WanAndroidApp injectWanAndroidApp(WanAndroidApp instance) {
        WanAndroidApp_MembersInjector.injectMAndroidInjector(
            instance,
            getDispatchingAndroidInjectorOfActivity());
        return instance;
    }
    

首先，执行getDispatchingAndroidInjectorOfActivity()方法得到了一个Activity类型的DispatchingAndroidInjector对象，继续查看getDispatchingAndroidInjectorOfActivity()方法：


    private DispatchingAndroidInjector<Activity> getDispatchingAndroidInjectorOfActivity() {
        return DispatchingAndroidInjector_Factory.newDispatchingAndroidInjector(
        getMapOfClassOfAndProviderOfFactoryOf());
    }


在getDispatchingAndroidInjectorOfActivity()方法里面，首先调用了getMapOfClassOfAndProviderOfFactoryOf()方法，我们看到这个方法：


    private Map<Class<? extends Activity>, Provider<AndroidInjector.Factory<? extends Activity>>>
      getMapOfClassOfAndProviderOfFactoryOf() {
        return MapBuilder
            .<Class<? extends Activity>, Provider<AndroidInjector.Factory<? extends Activity>>>
            newMapBuilder(8)
            .put(MainActivity.class, (Provider) mainActivitySubcomponentBuilderProvider)
            .put(SplashActivity.class, (Provider) splashActivitySubcomponentBuilderProvider)
            .put(ArticleDetailActivity.class,
                (Provider) articleDetailActivitySubcomponentBuilderProvider)
            .put(KnowledgeHierarchyDetailActivity.class,
                (Provider) knowledgeHierarchyDetailActivitySubcomponentBuilderProvider)
            .put(LoginActivity.class, (Provider) loginActivitySubcomponentBuilderProvider)
            .put(RegisterActivity.class, (Provider) registerActivitySubcomponentBuilderProvider)
            .put(AboutUsActivity.class, (Provider) aboutUsActivitySubcomponentBuilderProvider)
            .put(SearchListActivity.class, (Provider) searchListActivitySubcomponentBuilderProvider)
            .build();
    }


可以看到，这里新建了一个建造者模式实现的MapBuilder，并且同时制定了固定容量为8，将项目下使用了AndroidInjection.inject(mActivity)方法的8个Activity对应的xxxActivitySubcomponentBuilderProvider保存起来。

我们再回到getDispatchingAndroidInjectorOfActivity()方法，这里将上面得到的Map容器传入了DispatchingAndroidInjector_Factory的newDispatchingAndroidInjector()方法中，这里应该就是新建DispatchingAndroidInjector的地方了。我们点进去看看：


    public static <T> DispatchingAndroidInjector<T> newDispatchingAndroidInjector(
      Map<Class<? extends T>, Provider<AndroidInjector.Factory<? extends T>>> injectorFactories) {
        return new DispatchingAndroidInjector<T>(injectorFactories);
    }
    
    
在这里，果然新建了一个DispatchingAndroidInjector对象。继续看看DispatchingAndroidInjector的构造方法：


    @Inject
    DispatchingAndroidInjector(
      Map<Class<? extends T>, Provider<AndroidInjector.Factory<? extends T>>> injectorFactories) {
        this.injectorFactories = injectorFactories;
    }
    
    
这里仅仅是将传进来的Map容器保存起来了。

我们再回到WanAndroidApp_MembersInjector的injectMAndroidInjector()方法，将上面得到的DispatchingAndroidInjector实例传入，继续查看injectMAndroidInjector()这个方法：


    public static void injectMAndroidInjector(
      WanAndroidApp instance, DispatchingAndroidInjector<Activity> mAndroidInjector) {
        instance.mAndroidInjector = mAndroidInjector;
    }
    
    
可以看到，最后在WanAndroidApp_MembersInjector的injectMAndroidInjector()方法中，直接将新建好的DispatchingAndroidInjector实例赋值给了WanAndroidApp的mAndroidInjector。


# 五、在目标Activity下的AndroidInjection.inject(this)这句代码是如何将当前Activity对象纳入依赖分配总管DispatchingAndroidInjector囊中的呢？


首先，我们看到AndroidInjection.inject(this)这个方法：


    public static void inject(Activity activity) {
        checkNotNull(activity, "activity");
        
        // 1
        Application application = activity.getApplication();
        if (!(application instanceof HasActivityInjector)) {
        throw new RuntimeException(
            String.format(
                "%s does not implement %s",
                application.getClass().getCanonicalName(), 
                HasActivityInjector.class.getCanonicalName()));
        }

        // 2
        AndroidInjector<Activity> activityInjector =
            ((HasActivityInjector) application).activityInjector();
            
        checkNotNull(activityInjector, "%s.activityInjector() returned null", application.getClass());

        // 3
        activityInjector.inject(activity);
  }


在注释1处，会先判断当前的application是否实现了HasActivityInjector这个接口，如果没有，则抛出RuntimeException。如果有，会继续在注释2处调用application的activityInjector()方法得到DispatchingAndroidInjector实例。最后，在注释3处，会将当前的activity实例传入activityInjector的inject()方法中。我们继续查看inject()方法：


    @Override
    public void inject(T instance) {
        boolean wasInjected = maybeInject(instance);
        if (!wasInjected) {
            throw new IllegalArgumentException(errorMessageSuggestions(instance));
        }
    }
    
    
**DispatchingAndroidInjector的inject()方法，它的作用就是给传入的instance实例执行成员注入**。具体在这个案例中，其实就是负责将创建好的Presenter实例赋值给BaseActivity对象
的mPresenter全局变量。在inject()方法中，又调用了maybeInject()方法，我们继续查看它：


    @CanIgnoreReturnValue
    public boolean maybeInject(T instance) {
        // 1
        Provider<AndroidInjector.Factory<? extends T>> factoryProvider =
        injectorFactories.get(instance.getClass());
        if (factoryProvider == null) {
        return false;
        }

        @SuppressWarnings("unchecked")
        // 2
        AndroidInjector.Factory<T> factory = (AndroidInjector.Factory<T>) factoryProvider.get();
        try {
            // 3
            AndroidInjector<T> injector =
                checkNotNull(
                    factory.create(instance), "%s.create(I) should not return null.", factory.getClass());
            // 4
            injector.inject(instance);
            return true;
        } catch (ClassCastException e) {
            ...
        }
    }


在注释1处，我们从injectorFactories（前面得到的Map容器）中根据当前Activity实例拿到了factoryProvider对象，这里我们具体一点，看到MainActivity对应的factoryProvider，也就是我们研究的第一个问题中的mainActivitySubcomponentBuilderProvider：


    private void initialize(final Builder builder) {
        this.mainActivitySubcomponentBuilderProvider =
            new Provider<
                AbstractAllActivityModule_ContributesMainActivityInjector.MainActivitySubcomponent
                .Builder>() {
            @Override
            public AbstractAllActivityModule_ContributesMainActivityInjector.MainActivitySubcomponent
                    .Builder
                get() {
                    return new MainActivitySubcomponentBuilder();
                }
            };

        ...
        
    }
    
    
在maybeInject()方法的注释2处，调用了mainActivitySubcomponentBuilderProvider的get()方法得到了一个新建的MainActivitySubcomponentBuilder对象。在注释3处执行了它的create方法，create()方法的具体实现在AndroidInjector的内部类Builder中：


    abstract class Builder<T> implements AndroidInjector.Factory<T> {
        @Override
        public final AndroidInjector<T> create(T instance) {
            seedInstance(instance);
            return build();
        }


看到这里，我相信看过第一个问题的同学已经明白后面是怎么回事了。在create()方法中，我们首先MainActivitySubcomponentBuilder的seedInstance()将MainActivity实例注入，然后再调用它的build()方法新建了一个MainActivitySubcomponentImpl实例返回。


最后，在注释4处，执行了MainActivitySubcomponentImpl的inject()方法：


    private final class MainActivitySubcomponentImpl
        implements AbstractAllActivityModule_ContributesMainActivityInjector
        .MainActivitySubcomponent {
          
        private MainPresenter getMainPresenter() {
            // 2
            return MainPresenter_Factory.newMainPresenter(
            DaggerAppComponent.this.provideDataManagerProvider.get());
        }

        @Override
        public void inject(MainActivity arg0) {
            // 1
            injectMainActivity(arg0);
        }

        private MainActivity injectMainActivity(MainActivity instance) {
            // 3
            BaseActivity_MembersInjector
            .injectMPresenter(instance, getMainPresenter());
            return instance;
        }


这里的逻辑已经在问题一的最后部分详细讲解了，最后，会在注释3处调用BaseActivity_MembersInjector的injectMPresenter()方法：


    public static <T extends AbstractPresenter> void injectMPresenter(
      BaseActivity<T> instance, T mPresenter) {
        instance.mPresenter = mPresenter;
    }


这样，就将mPresenter对象赋值给了当前Activity对象的mPresenter全局变量中了。至此，Dagger.Android的核心源码分析就结束了。


### 五、总结


相比于ButterKnife，Dagger是一个**锋利的全局依赖注入管理框架**，它主要用来**管理对象的依赖关系和生命周期**，当项目越来越大时，类之间的调用层次会越来越深，并且有些类是Activity或Fragment，有些是单例，而且它们的生命周期不一致，所以创建所需对象时需要处理的各个对象的依赖关系和生命周期时的任务会很繁重。因此，使用Dagger会大大减轻这方面的工作量。虽然它的学习成本比较高，而且需要写一定的模板类，但是，**对于越大的项目来说，Dagger越值得被需要**。下一篇，便是Android主流三方库源码分析系列的终结篇了，笔者将会对Android中的事件总线框架EventBus源码进行深入的分析，敬请期待~
    

# 公众号

我的公众号 `JsonChao` 开通啦，如果您想第一时间获取最新文章和最新动态，欢迎扫描关注~

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b134e9db02e414aaaa3c59a440cf036~tplv-k3u1fbpfcp-zoom-1.image)


##### 参考链接：
---
1、Dagger V2.1.5 源码

2、Android进阶之光

3、[告别Dagger2模板代码：DaggerAndroid原理解析](https://blog.csdn.net/mq2553299/article/details/77725912)

4、[Android Dagger2 从零单排](https://www.jianshu.com/p/7ee1a1100fab)


## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e882b5fe28f14dd2b381c0e190f52a47~tplv-k3u1fbpfcp-zoom-1.image" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/4318537403878167](https://juejin.im/user/4318537403878167)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/4318537403878167)上一起分享知识。