---

		title:  Android主流三方库源码分析（九、深入理解EventBus源码）
		date: 2019/01/28 00:02:00   
		tags: 
		- Android主流三方库源码分析
		categories: 安卓主流三方库源码分析
		thumbnail: https://miro.medium.com/max/978/1*9SLf18DZkJd0POT9JmzrgQ.png
---

---

## 前言

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

不知不觉，Android主流三方库源码分析系列已经要接近尾声了。这一次，笔者将会对Android中的事件总线框架EventBus源码进行详细地解析，一起来和大家揭开它背后的面纱。

# 一、简单示例

## 1、首先，定义要传递的事件实体


    public class CollectEvent { ... }
    
    
## 2、准备订阅者：声明并注解你的订阅方法


    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onMessageEvent(CollectEvent event) {
        LogHelper.d("OK");
    }
    

## 3、在2中，也就是订阅中所在的类中，注册和解注册你的订阅者


    @Override
    public void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
    }
    
    @Override
    public void onStop() {
        super.onStop();
        EventBus.getDefault().unregister(this);
    }
    

## 4、发送事件


    EventBus.getDefault().post(new CollectEvent());
    
    
在正式讲解之前，笔者觉得需要对一些基础性的概念进行详细的讲解。众所周知，EventBus没出现之前，那时候的开发者一般是使用Android四大组件中的广播进行组件间的消息传递，那么我们**为什么要使用事件总线机制来替代广播呢**？主要是因为：


- 广播：耗时、容易被捕获（不安全）。
- 事件总线：更节省资源、更高效，能将信息传递给原生以外的各种对象。


那么，话又说回来了，**事件总线又是什么呢？**

如下图所示，事件总线机制通过记录对象、使用观察者模式来通知对象各种事件。（当然，你也可以发送基本数据类型如 int，String 等作为一个事件）


![image](https://upload-images.jianshu.io/upload_images/2276275-c20610cdd44f4c5f.png?imageMogr2/auto-orient/)


对于**事件总线EventBus**而言，它的**优缺点**又是如何？这里简单总结下：


- 优点：开销小，代码更优雅、简洁，解耦发送者和接收者，可动态设置事件处理线程和优先级。
- 缺点：每个事件必须自定义一个事件类，增加了维护成本。


EventBus是基于观察者模式扩展而来的，我们先了解一下观察者模式是什么？

观察者模式又可称为**发布 - 订阅模式**，它定义了对象间的一种1对多的依赖关系，每当这个对象的状态改变时，其它的对象都会接收到通知并被自动更新。

观察者模式有以下角色：

- 抽象被观察者：将所有已注册的观察者对象保存在一个集合中。
- 具体被观察者：当内部状态发生变化时，将会通知所有已注册的观察者。
- 抽象观察者：定义了一个更新接口，当被观察者状态改变时更新自己。
- 具体被观察者：实现抽象观察者的更新接口。


这里笔者给出一个简单的示例来让大家更深一步理解观察者模式的思想：


1、首先，创建抽象观察者


    public interface observer {
        
        public void update(String message);
    }
    
    
2、接着，创建具体观察者


    public class WeXinUser implements observer {
        private String name;
        
        public WeXinUser(String name) {
            this.name = name;
        }
        
        @Override
        public void update(String message) {
            ...
        }
    }

3、然后，创建抽象被观察者


    public interface observable {
        
        public void addWeXinUser(WeXinUser weXinUser);
        
        public void removeWeXinUser(WeXinUser weXinUser);
        
        public void notify(String message);
    }
    

4、最后，创建具体被观察者


    public class Subscription implements observable {
        private List<WeXinUser> mUserList = new ArrayList();
        
        @Override
        public void addWeXinUser(WeXinUser weXinUser) {
            mUserList.add(weXinUser);
        }
        
        @Override
        public void removeWeXinUser(WeXinUser weXinUser) {
            mUserList.remove(weXinUser);
        }
        
        @Override
        public void notify(String message) {
            for(WeXinUser weXinUser : mUserList) {
                weXinUser.update(message);
            }
        }
    }
    

在具体使用时，我们便可以这样使用，如下所示：


    Subscription subscription = new Subscription();
    
    WeXinUser hongYang = new WeXinUser("HongYang");
    WeXinUser rengYuGang = new WeXinUser("RengYuGang");
    WeXinUser liuWangShu = new WeXinUser("LiuWangShu");
    
    subscription.addWeiXinUser(hongYang);
    subscription.addWeiXinUser(rengYuGang);
    subscription.addWeiXinUser(liuWangShu);
    subscription.notify("New article coming");
    
    
在这里，hongYang、rengYuGang、liuWangShu等大神都订阅了我的微信公众号，每当我的公众号发表文章时（subscription.notify())，他们就会接收到最新的文章信息（weXinUser.update()）。（ps：当然，这一切都是YY，事实是，我并没有微信公众号-0v0-）


当然，EventBus的观察者模式和一般的观察者模式不同，它使用了**扩展的观察者模式对事件进行订阅和分发，其实这里的扩展就是指的使用了EventBus来作为中介者，抽离了许多职责**，如下是它的官方原理图：


![image](https://raw.githubusercontent.com/greenrobot/EventBus/master/EventBus-Publish-Subscribe.png)


在得知了EventBus的原理之后，我们注意到，每次我们在register之后，都必须进行一次unregister，这是为什么呢？

**因为register是强引用，它会让对象无法得到内存回收，导致内存泄露。所以必须在unregister方法中释放对象所占的内存**。


有些同学可能之前使用的是EventBus2.x的版本，那么它又与EventBus3.x的版本有哪些区别呢？


- 1、EventBus2.x使用的是**运行时注解，它采用了反射的方式对整个注册的类的所有方法进行扫描来完成注册，因而会对性能有一定影响**。
- 2、EventBus3.x使用的是**编译时注解，Java文件会编译成.class文件，再对class文件进行打包等一系列处理。在编译成.class文件时，EventBus会使用EventBusAnnotationProcessor注解处理器读取@Subscribe()注解并解析、处理其中的信息，然后生成Java类来保存所有订阅者的订阅信息。这样就创建出了对文件或类的索引关系，并将其编入到apk中**。
- 3、从EventBus3.0开始**使用了对象池缓存减少了创建对象的开销**。


除了EventBus，其实现在比较流行的事件总线还有RxBus，那么，它与EventBus相比又如何呢？


- 1、**RxJava的Observable有onError、onComplete等状态回调**。
- 2、**Rxjava使用组合而非嵌套的方式，避免了回调地狱**。
- 3、**Rxjava的线程调度设计的更加优秀，更简单易用**。
- 4、**Rxjava可使用多种操作符来进行链式调用来实现复杂的逻辑**。
- 5、**Rxjava的信息效率高于EventBus2.x，低于EventBus3.x**。


在了解了EventBus和RxBus的区别之后，那么，对待新项目的事件总线选型时，我们该如何考量？


很简单，**如果项目中使用了RxJava，则使用RxBus，否则使用EventBus3.x**。


接下来将按以下顺序来进行EventBus的源码分析：

- 1、订阅者：EventBus.getDefault().register(this)；
- 2、发布者：EventBus.getDefault().post(new CollectEvent())；
- 3、订阅者：EventBus.getDefault().unregister(this)。

下面，我们正式开始EventBus的探索之旅~


### 二、EventBus.getDefault().register(this)


首先，我们从获取EventBus实例的方法getDefault()开始分析：


    public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }


在getDefault()中使用了双重校验并加锁的单例模式来创建EventBus实例。

接着，我们看到EventBus的默认构造方法中做了什么:


    private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();

    public EventBus() {
        this(DEFAULT_BUILDER);
    }


在EventBus的默认构造方法中又调用了它的另一个有参构造方法，将一个类型为EventBusBuilder的DEFAULT_BUILDER对象传递进去了。这里的EventBusBuilder很明显是一个EventBus的建造器，以便于EventBus能够添加自定义的参数和安装一个自定义的默认EventBus实例。

我们再看一下EventBusBuilder的构造方法：


    public class EventBusBuilder {
    
        ...

        EventBusBuilder() {
        }
        
        ...
        
    }
    
    
EventBusBuilder的构造方法中什么也没有做，那我么继续查看EventBus的这个有参构造方法：

    
    private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
    private final Map<Object, List<Class<?>>> typesBySubscriber;
    private final Map<Class<?>, Object> stickyEvents;

    EventBus(EventBusBuilder builder) {
        ...
        
        // 1
        subscriptionsByEventType = new HashMap<>();
        
        // 2
        typesBySubscriber = new HashMap<>();
        
        // 3
        stickyEvents = new ConcurrentHashMap<>();
        
        // 4
        mainThreadSupport = builder.getMainThreadSupport();
        mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
        backgroundPoster = new BackgroundPoster(this);
        asyncPoster = new AsyncPoster(this);
        
        ...
        
        // 5
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);
       
        // 从builder取中一些列订阅相关信息进行赋值
        ...
       
        // 6
        executorService = builder.executorService;
    }


在注释1处，创建了一个subscriptionsByEventType对象，可以看到它是一个类型为HashMap的subscriptionsByEventType对象，并且其key为 Event 类型，value为 Subscription链表。这里的Subscription是一个订阅信息对象，它里面保存了两个重要的字段，一个是类型为 Object 的 subscriber，该字段即为注册的对象（在 Android 中时通常是 Activity对象）；另一个是 类型为SubscriberMethod 的 subscriberMethod，它就是被@Subscribe注解的那个订阅方法，里面保存了一个重要的字段eventType，它是 Class<?> 类型的，代表了 Event 的类型。在注释2处，新建了一个类型为 Map 的typesBySubscriber对象，它的key为subscriber对象，value为subscriber对象中所有的 Event 类型链表，日常使用中仅用于判断某个对象是否注册过。在注释3处新建了一个类型为ConcurrentHashMap的stickyEvents对象，它是专用于粘性事件处理的一个字段，key为事件的Class对象，value为当前的事件。可能有的同学不了解sticky event，这里解释下：

- 我们都知道**普通事件是先注册，然后发送事件才能收到；而粘性事件，在发送事件之后再订阅该事件也能收到。并且，粘性事件会保存在内存中，每次进入都会去内存中查找获取最新的粘性事件，除非你手动解除注册**。


在注释4处，新建了三个不同类型的事件发送器，这里总结下：

- mainThreadPoster：主线程事件发送器，通过它的mainThreadPoster.enqueue(subscription, event)方法可以将订阅信息和对应的事件进行入队，然后通过 handler 去发送一个消息，在 handler 的 handleMessage 中去执行方法。
- backgroundPoster：后台事件发送器，通过它的enqueue() 将方法加入到后台的一个队列，最后通过线程池去执行，注意，它在 Executor的execute()方法 上添加了 synchronized关键字 并设立 了控制标记flag，保证任一时间只且仅能有一个任务会被线程池执行。
- asyncPoster：实现逻辑类似于backgroundPoster，不同于backgroundPoster的保证任一时间只且仅能有一个任务会被线程池执行的特性，asyncPoster则是异步运行的，可以同时接收多个任务。

我们再回到注释5这行代码，这里新建了一个subscriberMethodFinder对象，这是从EventBus中抽离出的订阅方法查询的一个对象，在优秀的源码中，我们经常能看到**组合优于继承**的这种实现思想。在注释6处，从builder中取出了一个默认的线程池对象，它由**Executors的newCachedThreadPool()**方法创建，它是一个**有则用、无则创建、无数量上限**的线程池。

分析完这些核心的字段之后，后面的讲解就比较轻松了，接着我们查看EventBus的regist()方法：


    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        
        // 1
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                // 2
                subscribe(subscriber, subscriberMethod);
            }
        }
    }


在注释1处，根据当前注册类获取 subscriberMethods这个订阅方法列表 。在注释2处，使用了增强for循环令subsciber对象 对 subscriberMethods 中每个 SubscriberMethod 进行订阅。

接着我们查看SubscriberMethodFinder的findSubscriberMethods()方法：


    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        // 1
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        // 2
        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }


在注释1处，如果缓存中有subscriberClass对象对应 的订阅方法列表，则直接返回。注释2处，先详细说说这个**ignoreGeneratedIndex**字段， 它用来**判断是否使用生成的 APT 代码去优化寻找接收事件的过程，如果开启了的话，那么将会通过 subscriberInfoIndexes 来快速得到接收事件方法的相关信息**。如果我们没有在项目中接入 EventBus 的 APT，那么可以将 ignoreGeneratedIndex 字段设为 false 以提高性能。这里ignoreGeneratedIndex 默认为false，所以会执行findUsingInfo()方法，后面生成 subscriberMethods 成功的话会加入到缓存中，失败的话会 抛出异常。

接着我们查看SubscriberMethodFinder的findUsingInfo()方法：


    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        // 1
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        // 2
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod: array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                 // 3
                 findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        // 4
        return getMethodsAndRelease(findState);
    }


在注释1处，调用了SubscriberMethodFinder的prepareFindState()方法创建了一个新的 FindState 类，我们来看看这个方法：


    private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];
    private FindState prepareFindState() {
        // 1
        synchronized(FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                FindState state = FIND_STATE_POOL[i];
                if (state != null) {
                    FIND_STATE_POOL[i] = null;
                    return state;
                }
            }
        }
        // 2
        return new FindState();
    }
    

在注释1处，会先从 FIND_STATE_POOL 即 FindState 池中取出可用的 FindState（这里的POOL_SIZE为4），如果没有的话，则通过注释2处的代码直接新建 一个新的 FindState 对象。


接着我们来分析下FindState这个类：


    static class FindState {
        ....
        void initForSubscriber(Class<?> subscriberClass) {
            this.subscriberClass = clazz = subscriberClass;
            skipSuperClasses = false;
            subscriberInfo = null;
        }
        ...
    }


它是 SubscriberMethodFinder 的内部类，这个方法主要做一个初始化、回收对象等工作。

我们接着回到SubscriberMethodFinder的注释2处的SubscriberMethodFinder()方法：


    private SubscriberInfo getSubscriberInfo(FindState findState) {
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }
        if (subscriberInfoIndexes != null) {
            for (SubscriberInfoIndex index: subscriberInfoIndexes) {
                SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
                if (info != null) {
                    return info;
                }
            }
        }
        return null;
    }
    
    
在前面初始化的时候，findState的subscriberInfo和subscriberInfoIndexes 这两个字段为空，所以这里直接返回 null。

接着我们查看注释3处的findUsingReflectionInSingleClass()方法：


    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method: methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?> [] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        // 重点
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode, subscribeAnnotation.priority(),  subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification &&     method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException("@Subscribe method " + methodName + "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName + " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }


这个方法很长，大概做的事情是：

- 1、**通过反射的方式获取订阅者类中的所有声明方法，然后在这些方法里面寻找以 @Subscribe作为注解的方法进行处理**。
- 2、**在经过经过一轮检查，看看 findState.subscriberMethods是否存在，如果没有，将方法名，threadMode，优先级，是否为 sticky 方法等信息封装到 SubscriberMethod 对象中，最后添加到 subscriberMethods 列表中**。


最后，我们继续查看注释4处的getMethodsAndRelease()方法：


    private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
        // 1
        List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
        // 2
        findState.recycle();
        // 3
        synchronized(FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                if (FIND_STATE_POOL[i] == null) {
                    FIND_STATE_POOL[i] = findState;
                    break;
                }
            }
        }
        // 4
        return subscriberMethods;
    }
    
    
在这里，首先在注释1处，从findState中取出了保存的subscriberMethods。在注释2处，将findState里的保存的所有对象进行回收。在注释3处，把findState存储在 FindState 池中方便下一次使用，以提高性能。最后，在注释4处，返回subscriberMethods。接着，**在EventBus的 register() 方法的最后会调用 subscribe 方法**：


    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }


我们继续看看这个subscribe()方法做的事情：


    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        
        // 1
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList <> ();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event " + eventType);
            }
        }
        int size = subscriptions.size();
        
        // 2
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }
        
        // 3
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);
        // 4
        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if(eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }


首先，在注释1处，会根据 subscriberMethod的eventType，在 subscriptionsByEventType 去查找一个 CopyOnWriteArrayList ，如果没有则创建一个新的 CopyOnWriteArrayList，然后将这个 CopyOnWriteArrayList 放入 subscriptionsByEventType 中。在注释2处，**添加 newSubscription对象，它是一个 Subscription 类，里面包含着 subscriber 和 subscriberMethod 等信息，并且这里有一个优先级的判断，说明它是按照优先级添加的。优先级越高，会插到在当前 List 靠前面的位置**。在注释3处，对typesBySubscriber 进行添加，这主要是在EventBus的isRegister()方法中去使用的，目的是用来判断这个 Subscriber对象 是否已被注册过。最后，在注释4处，会判断是否是 sticky事件。如果是sticky事件的话，会调用 checkPostStickyEventToSubscription() 方法。

我们接着查看这个checkPostStickyEventToSubscription()方法：


    private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
        if (stickyEvent != null) {
            postToSubscription(newSubscription, stickyEvent, isMainThread());
        }
    }
    
    
可以看到最终是**调用了postToSubscription()这个方法来进行粘性事件的发送**，对于粘性事件的处理，我们最后再分析，接下来我们看看事件是如何post的。


### 三、EventBus.getDefault().post(new CollectEvent())


    public void post(Object event) {
        // 1
        PostingThreadState postingState = currentPostingThreadState.get();
        List <Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);
        // 2
        if (!postingState.isPosting) {
            postingState.isMainThread = isMainThread();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }


注释1处，这里的currentPostingThreadState 是一个 ThreadLocal 类型的对象，里面存储了 PostingThreadState，而 PostingThreadState 中包含了一个 eventQueue 和其他一些标志位，相关的源码如下：


    private final ThreadLocal <PostingThreadState> currentPostingThreadState = new ThreadLocal <PostingThreadState> () {
    @Override
    protected PostingThreadState initialValue() {
        return new PostingThreadState();
    }
    };

    final static class PostingThreadState {
        final List <Object> eventQueue = new ArrayList<>();
        boolean isPosting;
        boolean isMainThread;
        Subscription subscription;
        Object event;
        boolean canceled;
    }
    
    
接着把传入的 event，保存到了当前线程中的一个变量 PostingThreadState 的 eventQueue 中。在注释2处，最后调用了 postSingleEvent() 方法，我们继续查看这个方法：


    private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        // 1
        if (eventInheritance) {
            // 2
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |=
                // 3
                postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            ...
        }
    }
    
    
首先，在注释1处，首先取出 Event 的 class 类型，接着**会对 eventInheritance 标志位 判断，它默认为true，如果设为 true 的话，它会在发射事件的时候判断是否需要发射父类事件，设为 false，能够提高一些性能**。接着，在注释2处，会调用lookupAllEventTypes() 方法，它的作用就是取出 Event 及其父类和接口的 class 列表，当然重复取的话会影响性能，所以它也做了一个 eventTypesCache 的缓存，这样就不用重复调用 getSuperclass() 方法。最后，在注释3处会调用postSingleEventForEventType()方法，我们看下这个方法：


    private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class <?> eventClass) {
        CopyOnWriteArrayList <Subscription> subscriptions;
        synchronized(this) {
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription: subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
    }
    
    
可以看到，这里直接根据 Event 类型从 subscriptionsByEventType 中取出对应的 subscriptions对象，最后调用了 postToSubscription() 方法。

这个时候我们再看看这个postToSubscription()方法：


    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case MAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknow thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
    

从上面可以看出，这里通过threadMode 来判断在哪个线程中去执行方法：

- 1、POSTING：执行 invokeSubscriber() 方法，内部**直接采用反射调用**。
- 2、MAIN：**首先去判断当前是否在 UI 线程，如果是的话则直接反射调用，否则调用mainThreadPoster的enqueue()方法，即把当前的方法加入到队列之中，然后通过 handler 去发送一个消息，在 handler 的 handleMessage 中去执行方法**。
- 3、MAIN_ORDERED：**与MAIN类似，不过是确保是顺序执行的**。
- 4、BACKGROUND：**判断当前是否在 UI 线程，如果不是的话则直接反射调用，是的话通过backgroundPoster的enqueue()方法 将方法加入到后台的一个队列，最后通过线程池去执行。注意，backgroundPoster在 Executor的execute()方法 上添加了 synchronized关键字 并设立 了控制标记flag，保证任一时间只且仅能有一个任务会被线程池执行**。
- 5、ASYNC：**逻辑实现类似于BACKGROUND，将任务加入到后台的一个队列，最终由Eventbus 中的一个线程池去调用，这里的线程池与 BACKGROUND 逻辑中的线程池用的是同一个，即使用Executors的newCachedThreadPool()方法创建的线程池，它是一个有则用、无则创建、无数量上限的线程池。不同于backgroundPoster的保证任一时间只且仅能有一个任务会被线程池执行的特性，这里asyncPoster则是异步运行的，可以同时接收多个任务**。


分析完EventBus的post()方法值，我们接着看看它的unregister()。


### 四、EventBus.getDefault().unregister(this)


它的核心源码如下所示：


    public synchronized void unregister(Object subscriber) {
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        if (subscribedTypes != null) {
            for (Class<?> eventType : subscribedTypes) {
                //1
                unsubscribeByEventType(subscriber, eventType);
            }
            // 2
            typesBySubscriber.remove(subscriber);
        }
    }
    
    
首先，在注释1处，**unsubscribeByEventType() 方法中对 subscriptionsByEventType 移除了该 subscriber 的所有订阅信息**。最后，在注释2处，**移除了注册对象和其对应的所有 Event 事件链表**。


最后，我们在来分析下EventBus中对粘性事件的处理。


### 五、EventBus.getDefault.postSticky(new CollectEvent())


如果想要发射 sticky 事件需要通过 EventBus的postSticky() 方法，内部源码如下所示：

    
    public void postSticky(Object event) {
        synchronized (stickyEvents) {
            // 1
            stickyEvents.put(event.getClass(), event);
        }
        // 2
        post(event);
    }
    

在注释1处，先将该事件放入 stickyEvents 中，接着在注释2处使用post()发送事件。前面我们在分析register()方法的最后部分时，其中有关粘性事件的源码如下：


    if (subscriberMethod.sticky) {
        Object stickyEvent = stickyEvents.get(eventType);
        if (stickyEvent != null) {
            postToSubscription(newSubscription, stickyEvent, isMainThread());
        }
    }


可以看到，在这里**会判断当前事件是否是 sticky 事件，如果 是，则从 stickyEvents 中拿出该事件并执行 postToSubscription() 方法**。

至此，EventBus源码分析完毕。


### 六、总结


EventBus 的源码在Android主流三方库源码分析系列中可以说是除了ButterKnife之外，算是比较简单的了。但是，它其中的一些思想和设计是值得借鉴的。比如**它使用 FindState 复用池来复用 FindState 对象，在各处使用了 synchronized 关键字进行代码块同步的一些优化操作**。其中上面分析了这么多，**EventBus最核心的逻辑就是利用了 subscriptionsByEventType 这个重要的列表，将订阅对象，即接收事件的方法存储在这个列表，发布事件的时候在列表中查询出相对应的方法并执行**。至此，Android主流三方库源码分析系列到此完结~


    想象一个来自未来的自己，他非常自信，非常成功，
    拥有你现在所希望的一切，他会对现在的你说些什么？
    他怎么说，你就怎么去做，10年之后，你就变成了他。



##### 参考链接：
---
1、EventBus V3.1.1 源码

2、Android进阶之光

3、Android组件化架构

3、[EventBus设计之禅](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650243033&idx=1&sn=28709941e6f8821c6f5dd3fa047b1393&chksm=88638eb6bf1407a034ba2184cac9e31c24f43e3ac395b1b23b18243f64220c244f2edc7dc22a&scene=38#wechat_redirect)

4、[从源码入手来学习EventBus 3事件总线机制](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650242621&idx=1&sn=cc9c31aba5ff33b20fb9fecc3b0404b2&chksm=88638f52bf14064453fc09e6e53798ac5a1299d84df490b15764f9a85292732879eb4a1c0665&scene=38#wechat_redirect)


## 赞赏

如果这个库对您有很大帮助，您愿意支持这个项目的进一步开发和这个项目的持续维护。你可以扫描下面的二维码，让我喝一杯咖啡或啤酒。非常感谢您的捐赠。谢谢！

<div align="center">
<img src="https://raw.githubusercontent.com/JsonChao/Awesome-Android-Interview/master/screenshot/wexin_play.jpg" width=20%><img src="https://raw.githubusercontent.com/JsonChao/Awesome-Android-Interview/master/screenshot/Apaliy.jpg" width=20%>
</div>


----

## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="https://raw.githubusercontent.com/JsonChao/Awesome-Android-Interview/master/screenshot/wexin_qrcode.jpg" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/5a3ba9375188252bca050ade](https://juejin.im/user/5a3ba9375188252bca050ade)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/5a3ba9375188252bca050ade)上一起分享知识。