---

		title:  Android主流三方库源码分析（七、深入理解ButterKnife源码）
		date: 2019/01/13 22:57:00   
		tags: 
		- Android主流三方库源码分析
		categories: 安卓主流三方库源码分析
		thumbnail: https://ws1.sinaimg.cn/large/a62a5cffly1fv1dx3qazmj20mi0ciabd.jpg
---

---

## 前言

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

不知不觉，笔者已经对Android主流三方库中的网络框架OkHttp、Retrofit，图片加载框架Glide、数据库框架GreenDao、响应式编程框架RxJava、内存泄露框架LeakCanary进行了详细的分析，如果有朋友对这些开源框架的内部实现机制感兴趣的话，可以在笔者的个人主页选择相应的文章阅读。这篇，我将会对Android中的依赖注入框架ButterKnife的源码实现机制进行详细地讲解。

# 一、简单示例

首先，我们先来看一下ButterKnife的基本使用（取自[Awesome-WanAndroid](https://github.com/JsonChao/Awesome-WanAndroid)），如下所示：

    public class CollectFragment extends BaseRootFragment<CollectPresenter> implements CollectContract.View {

        @BindView(R.id.normal_view)
        SmartRefreshLayout mRefreshLayout;
        @BindView(R.id.collect_recycler_view)
        RecyclerView mRecyclerView;
        @BindView(R.id.collect_floating_action_btn)
        FloatingActionButton mFloatingActionButton;
        
        @Nullable
        @Override
        public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
            View view = inflater.inflate(getLayoutId(), container, false);
            unBinder = ButterKnife.bind(this, view);
            initView();
            return view;
        }
        
        @OnClick({R.id.collect_floating_action_btn})
        void onClick(View view) {
            switch (view.getId()) {
                case R.id.collect_floating_action_btn:
                    mRecyclerView.smoothScrollToPosition(0);
                    break;
                default:
                    break;
            }
        }
        
        
        @Override
        public void onDestroyView() {
            super.onDestroyView();
            if (unBinder != null && unBinder != Unbinder.EMPTY) {
                unBinder.unbind();
                unBinder = null;
            }
        }


可以看到，我们使用了@BindView()替代了findViewById()方法，然后使用了@OnClick替代了setOnClickListener()方法。ButterKnife的初期版本是通过使用注解+反射这样的运行时解析的方式实现上述功能的，后面，为了改善性能，便使用了**注解+APT编译时解析技术并从中生成配套模板代码的方式**来实现。

在开始分析之前，可能有同学对APT不是很了解，我这里普及一下，APT是Annotation Processing Tool的缩写，即注解处理工具。它的使用步骤通常为如下三个步骤：

- 1、**首先，声明注解的生命周期为CLASS，即@Retention(CLASS)**。
- 2、**然后，通过继承AbstractProcessor自定义一个注解处理器**。
- 3、**最后，在编译的时候，编译器会扫描所有带有你要处理的注解的类，最后再调用AbstractProcessor的process方法，对注解进行处理**。


下面，我们正式来解剖一下ButterKnife的心脏。


# 二、源码分析

## 1、模板代码解析

首先，在我们编写好上述的示例代码之后，调用 gradle build 命令，在app/build/generated/source/apt下将可以找到APT为我们生产的配套模板代码CollectFragment_ViewBinding，如下所示：

    public class CollectFragment_ViewBinding implements Unbinder {
        private CollectFragment target;
        
        private View view2131230812;
        
        @UiThread
        public CollectFragment_ViewBinding(final CollectFragment target, View source) {
          this.target = target;
        
          View view;
          // 1
          target.mRefreshLayout = Utils.findRequiredViewAsType(source, R.id.normal_view, "field 'mRefreshLayout'", SmartRefreshLayout.class);
          target.mRecyclerView = Utils.findRequiredViewAsType(source, R.id.collect_recycler_view, "field 'mRecyclerView'", RecyclerView.class);
          view = Utils.findRequiredView(source, R.id.collect_floating_action_btn, "field 'mFloatingActionButton' and method 'onClick'");
          target.mFloatingActionButton = Utils.castView(view, R.id.collect_floating_action_btn, "field 'mFloatingActionButton'", FloatingActionButton.class);
          view2131230812 = view;
          // 2
          view.setOnClickListener(new DebouncingOnClickListener() {
            @Override
            public void doClick(View p0) {
              target.onClick(p0);
            }
          });
        }
        
        @Override
        @CallSuper
        public void unbind() {
          CollectFragment target = this.target;
          if (target == null) throw newIllegalStateException("Bindings already     cleared.");
          this.target = null;
        
          target.mRefreshLayout = null;
          target.mRecyclerView = null;
          target.mFloatingActionButton = null;
        
          view2131230812.setOnClickListener(null);
          view2131230812 = null;
        }
    }

生成的配套模板CollectFragment_ViewBinding中，在注释1处，使用了ButterKnife内部的工具类Utils的findRequiredViewAsType()方法来寻找控件。在注释2处，使用了view的setOnClickListener()方法来添加了一个去抖动的DebouncingOnClickListener，这样便可以防止重复点击，在重写的doClick()方法内部，直接调用了CollectFragment的onClick方法。最后，我们在深入看下Utils的findRequiredViewAsType()方法内部的实现。

    public static <T> T findRequiredViewAsType(View source, @IdRes int id, String who,
      Class<T> cls) {
        // 1
        View view = findRequiredView(source, id, who);
        // 2
        return castView(view, id, who, cls);
    }

    public static View findRequiredView(View source, @IdRes int id, String who) {
        View view = source.findViewById(id);
        if (view != null) {
            return view;
        }
        
        ...
    }

    public static <T> T castView(View view, @IdRes int id, String who, Class<T> cls) {
        try {
            return cls.cast(view);
        } catch (ClassCastException e) {
            ...
        }
    }

在注释1处，**最终也是通过View的findViewById()方法找到相应的控件**，在注释2处，**通过相应Class对象的cast方法强转成对应的控件类型**。


## 2、ButterKnife 是怎样实现代码注入的

接下来，为了使用这套模板代码，我们必须调用ButterKnife的bind()方法实现代码注入，即自动帮我们执行重复繁琐的findViewById和setOnClicklistener操作。下面我们来分析下bind()方法是如何实现注入的。

    @NonNull @UiThread
    public static Unbinder bind(@NonNull Object target, @NonNull View source) {
        return createBinding(target, source);
    }
    
在bind()方法中调用了createBinding()，

    @NonNull @UiThread
    public static Unbinder bind(@NonNull Object target, @NonNull View source) {
        Class<?> targetClass = target.getClass();
        // 1
        Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

        if (constructor == null) {
            return Unbinder.EMPTY;
        }

        
        try {
            // 2
            return constructor.newInstance(target, source);
        // 3
        } catch (IllegalAccessException e) {
        ...
    }

首先，在注释1处，通过 findBindingConstructorForClass() 方法从 Class 中查找 constructor，这里constructor即上文生成的CollectFragment_ViewBinding类。然后，在注释2处，**利用反射来新建 constructor 对象**。最后，如果新建 constructor 对象失败，则会在注释3后面捕获一系列对应的异常进行自定义异常抛出处理。

下面，我们来详细分析下
findBindingConstructorForClass() 方法的实现逻辑。

    @VisibleForTesting
    static final Map<Class<?>, Constructor<? extends Unbinder>> BINDINGS = new LinkedHashMap<>();

    @Nullable @CheckResult @UiThread
    private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
        // 1
        Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
        if (bindingCtor != null || BINDINGS.containsKey(cls)) {
            return bindingCtor;
        }
        
        // 2
        String clsName = cls.getName();
        if (clsName.startsWith("android.") || clsName.startsWith("java.")
        || clsName.startsWith("androidx.")) {
            return null;
        }
        
        try {
            // 3
            Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
            bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
        } catch (ClassNotFoundException e) {
            // 4
            bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
        } catch (NoSuchMethodException e) {
            throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
        }
        
        // 5
        BINDINGS.put(cls, bindingCtor);
        return bindingCtor;
    }

这里，我把多余的log代码删除并把代码格式优化了一下，可以看到，findBindingConstructorForClass() 这个方法中的逻辑瞬间清晰不少，这里**建议以后大家自己在分析源码的时候可以进行这样的优化重整**，会带来不少好处。

重新看到 findBindingConstructorForClass() 方法，在注释1处，我们首先从缓存BINDINGS中获取CollectFragment类对象对应的模块类CollectFragment_ViewBinding的构造器对象，这里的BINDINGS是一个LinkedHashMap对象，它保存了上述两者的映射关系。在注释2处，如果是 android，androidx，java 原生的文件，不进行处理。在注释3处，先**通过CollectFragment类对象的类加载器加载出对应的模块类CollectFragment_ViewBinding的类对象**，再通过自身的getConstructor()方法获得相应的构造对象。如果在步骤3中加载不出对应的模板类对象，则会在注释4处使用类似递归的方法重新执行findBindingConstructorForClass()方法。最后，如果找到了bindingCtor模板构造对象，则将它保存在BINDINGS这个LinkedHashMap对象中。

**这里总结一下findBindingConstructorForClass()方法的处理：**

- 1、**首先从缓存BINDINGS中获取CollectFragment类对象对应的模块类CollectFragment_ViewBinding的构造器对象，获取不到，则继续执行下面的操作**。
- 2、**如果不是android，androidx，java 原生的文件，再进行后面的处理**。
- 3、**通过CollectFragment类对象的类加载器加载出对应的模块类CollectFragment_ViewBinding的类对象，再通过自身的getConstructor()方法获得相应的构造对象，如果获取不到，会抛出异常，在异常的处理中，我们会从当前 class 文件的父类中再去查找。如果找到了，最后会将bindingCtor对象缓存进在BINDINGS对象中**。


## 3、ButterKnife是如何在编译时生成代码的？

在编译的时候，ButterKnife会通过自定义的注解处理器ButterKnifeProcessor的process方法，对编译器扫描到的要处理的类中的注解进行处理，然后，**通过javapoet这个库来动态生成绑定事件或者控件的模板代码**，最后在运行的时候，直接调用bind方法完成绑定即可。

首先，我们先来分析下ButterKnifeProcessor的重写的入口方法init()。

    @Override public synchronized void init(ProcessingEnvironment env) {
        super.init(env);

        String sdk = env.getOptions().get(OPTION_SDK_INT);
        if (sdk != null) {
            try {
                this.sdk = Integer.parseInt(sdk);
            } catch (NumberFormatException e) {
               ...
            }
        }

        typeUtils = env.getTypeUtils();
        filer = env.getFiler();
        ...
    }

可以看到，**ProcessingEnviroment对象提供了两大工具类 typeUtils和filer。typeUtils的作用是用来处理TypeMirror，而Filer则是用来创建生成辅助文件**。


接着，我们再来看看被重写的getSupportedAnnotationTypes()方法，这个方法的作用主要是用于指定ButterknifeProcessor注册了哪些注解的。

    @Override public Set<String> getSupportedAnnotationTypes() {
        Set<String> types = new LinkedHashSet<>();
        for (Class<? extends Annotation> annotation : getSupportedAnnotations()) {
        types.add(annotation.getCanonicalName());
        }
        return types;
    }
    
这里面首先创建了一个LinkedHashSet对象，然后将getSupportedAnnotations()方法返回的支持注解集合进行遍历一一并添加到types中返回。

接着我们看下getSupportedAnnotations()方法，

    private Set<Class<? extends Annotation>> getSupportedAnnotations() {
        Set<Class<? extends Annotation>> annotations = new LinkedHashSet<>();

        annotations.add(BindAnim.class);
        annotations.add(BindArray.class);
        annotations.add(BindBitmap.class);
        annotations.add(BindBool.class);
        annotations.add(BindColor.class);
        annotations.add(BindDimen.class);
        annotations.add(BindDrawable.class);
        annotations.add(BindFloat.class);
        annotations.add(BindFont.class);
        annotations.add(BindInt.class);
        annotations.add(BindString.class);
        annotations.add(BindView.class);
        annotations.add(BindViews.class);
        annotations.addAll(LISTENERS);
    
        return annotations;
    }

可以看到，这里注册了一系列的Bindxxx注解类和监听列表LISTENERS，接着看一下LISTENERS中包含的监听方法：

    private static final List<Class<? extends Annotation>> LISTENERS = Arrays.asList(
        OnCheckedChanged.class, 
        OnClick.class, 
        OnEditorAction.class, 
        OnFocusChange.class, 
        OnItemClick.class, 
        OnItemLongClick.class, 
        OnItemSelected.class, 
        OnLongClick.class, 
        OnPageChange.class, 
        OnTextChanged.class, 
        OnTouch.class 
    );

最后，我们来分析下整个ButterKnifeProcessor中最关键的方法process()。

    @Override public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
        // 1
        Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);

        for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
            TypeElement typeElement = entry.getKey();
            BindingSet binding = entry.getValue();

            // 2
            JavaFile javaFile = binding.brewJava(sdk, debuggable);
            try {
                javaFile.writeTo(filer);
            } catch (IOException e) {
               ...
            }
        }

        return false;
    }

首先，在注释1处通过**findAndParseTargets()方法**，知名见义，它应该就是**找到并解析注解目标的关键方法**了，继续看看它内部的处理：

    private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
        Map<TypeElement, BindingSet.Builder> builderMap = new LinkedHashMap<>();
        Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();

        // 1、一系列处理每一个@Bindxxx元素的for循环代码块
        ...

        // Process each @BindView element.
        for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
            try {
            // 2
            parseBindView(element, builderMap, erasedTargetNames);
            } catch (Exception e) {
                logParsingError(element, BindView.class, e);
            }
        }

        // Process each @BindViews element.
        ...

        // Process each annotation that corresponds to a listener.
        for (Class<? extends Annotation> listener : LISTENERS) {
            findAndParseListener(env, listener, builderMap, erasedTargetNames);
        }

        // 2
        Deque<Map.Entry<TypeElement, BindingSet.Builder>> entries =
            new ArrayDeque<>(builderMap.entrySet());
        Map<TypeElement, BindingSet> bindingMap = new LinkedHashMap<>();
        while (!entries.isEmpty()) {
            Map.Entry<TypeElement, BindingSet.Builder> entry = entries.removeFirst();

            TypeElement type = entry.getKey();
            BindingSet.Builder builder = entry.getValue();

            TypeElement parentType = findParentType(type, erasedTargetNames);
            if (parentType == null) {
                bindingMap.put(type, builder.build());
            } else {
                BindingSet parentBinding = bindingMap.get(parentType);
                if (parentBinding != null) {
                    builder.setParent(parentBinding);
                    bindingMap.put(type, builder.build());
                } else {
                entries.addLast(entry);
                }
            }
        }
        return bindingMap;
    }

findAndParseTargets()方法的代码非常多，我这里尽可能做了精简。首先，在注释1处，**扫描并处理所有具有@Bindxxx注解和符合LISTENERS监听方法集合的代码，然后在每一个@Bindxxx对应的for循环代码中的parseBindxxx()或findAndParseListener()方法中将解析出的信息放入builderMap这个LinkedHashMap对象中**，其中builderMap是一个key为TypeElement，value为BindingSet.Builder的映射集合，这个 BindSet 是指**的一个类型请求的所有绑定的集合**。在注释3处，首先使用上面的builderMap对象去构建了一个entries对象，它是一个双向队列，能实现两端存取的操作。接着，又新建了一个key为TypeElement，value为BindingSet的LinkedHashMap对象，最后使用了一个while循环从entries的第一个元素开始，这里会判断当前元素类型是否有父类，如果没有，直接构建builder放入bindingMap中，如果有，则将parentBinding添加到BindingSet.Builder这个建造者对象中，然后再创建BindingSet再添加到bindingMap中。

接着，我们分析下注释2处parseBindView是如何对每一个@BindView注解的元素进行处理。

    private void parseBindView(Element element, Map<TypeElement, BindingSet.Builder> builderMap,
      Set<TypeElement> erasedTargetNames) {
        TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

        // 1、首先验证生成的常见代码限制
        ...

        // 2、验证目标类型是否继承自View。
        ...
        
        // 3
        int id = element.getAnnotation(BindView.class).value();
        BindingSet.Builder builder = builderMap.get(enclosingElement);
        Id resourceId = elementToId(element, BindView.class, id);
        if (builder != null) {
            String existingBindingName = builder.findExistingBindingName(resourceId);
            if (existingBindingName != null) {
                ...
                return;
            }
        } else {
            // 4
            builder = getOrCreateBindingBuilder(builderMap, enclosingElement);
        }

        String name = simpleName.toString();
        TypeName type = TypeName.get(elementType);
        boolean required = isFieldRequired(element);
    
        // 5
        builder.addField(resourceId, new     FieldViewBinding(name, type, required));

        // Add the type-erased version to the valid binding targets set.
        erasedTargetNames.add(enclosingElement);
    }

首先，在注释1、2处均是一些验证处理操作，如果不符合则会return。然后，我们看到注释3处，这里获取了BindView要绑定的View的id，然后先从builderMap中获取BindingSet.Builder对象，如果存在，直接return。如果不存在，则会在注释4处的
getOrCreateBindingBuilder()方法生成一个。我们看一下getOrCreateBindingBuilder()方法:

    private BindingSet.Builder getOrCreateBindingBuilder(
      Map<TypeElement, BindingSet.Builder> builderMap, TypeElement enclosingElement) {
        BindingSet.Builder builder = builderMap.get(enclosingElement);
        if (builder == null) {
            builder = BindingSet.newBuilder(enclosingElement);
            builderMap.put(enclosingElement, builder);
        }
        return builder;
    }

可以看到，这里会再次从buildMap中获取BindingSet.Builder对象，如果没有则直接调用BindingSet的newBuilder()方法新建一个BindingSet.Builder对象并保存在builderMap中，然后，再将新建的builder对象返回。

回到parseBindView()方法的注释5处，这里根据view的信息生成一个FieldViewBinding，最后添加到上边生成的builder对象中。

最后，再回到我们的process()方法中，现在**所有的绑定的集合数据都放在了bindingMap对象中，这里使用for循环取出每一个BindingSet对象，调用它的brewJava()方法**，看看它内部的处理：

    JavaFile brewJava(int sdk, boolean debuggable) {
        TypeSpec bindingConfiguration = createType(sdk, debuggable);
        return JavaFile.builder(bindingClassName.packageName(), bindingConfiguration)
        .addFileComment("Generated code from Butter Knife. Do not modify!")
        .build();
    }
    
    private TypeSpec createType(int sdk, boolean debuggable) {
        TypeSpec.Builder result = TypeSpec.classBuilder(bindingClassName.simpleName())
        .addModifiers(PUBLIC);
        if (isFinal) {
            result.addModifiers(FINAL);
        }

        if (parentBinding != null) {
            result.superclass(parentBinding.bindingClassName);
        } else {
            result.addSuperinterface(UNBINDER);
        }

        if (hasTargetField()) {
            result.addField(targetTypeName, "target", PRIVATE);
        }

        if (isView) {
            result.addMethod(createBindingConstructorForView());
        } else if (isActivity) {
            result.addMethod(createBindingConstructorForActivity());
        } else if (isDialog) {
            result.addMethod(createBindingConstructorForDialog());
        }
        if (!constructorNeedsView()) {
            // Add a delegating constructor with a target type + view signature for reflective use.
            result.addMethod(createBindingViewDelegateConstructor());
        }
        result.addMethod(createBindingConstructor(sdk, debuggable));

        if (hasViewBindings() || parentBinding == null) {
            result.addMethod(createBindingUnbindMethod(result));
        }

        return result.build();
    }

在createType()方法里面使用了java中的[javapoet](https://github.com/square/javapoet)技术生成了一个bindingConfiguration对象，很显然，它里面**保存了所有的绑定配置信息。然后，通过javapoet的builder构造器将上面得到的bindingConfiguration对象构建生成一个JavaFile对象，最终，通过javaFile.writeTo(filer)生成了java源文件**。

至此，ButterKnife的源码分析就结束了。


# 三、总结

从上面的源码分析来看，ButterKnife的执行流程总体可以分为如下两步：


-  1、**在编译的时候扫描注解，并通过自定义的ButterKnifeProcessor做相应的处理解析得到bindingMap对象，最后，调用 javapoet 库生成java模板代码**。
-  2、**当我们调用 ButterKnife的bind()方法的时候，它会根据类的全限定类型，找到相应的模板代码，并在其中完成 findViewById 和 setOnClick ，setOnLongClick 等操作**。


接下来，笔者会对Android中的依赖注入框架Dagger2的源码实现流程进行详细的讲解，敬请期待~


# 公众号

我的公众号 `JsonChao` 开通啦，如果您想第一时间获取最新文章和最新动态，欢迎扫描关注~

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f554fcc1cae47088f4d3ba3e04ac626~tplv-k3u1fbpfcp-zoom-1.image)



##### 参考链接：
---
1、ButterKnife V10.0.0 源码

2、Android进阶之光

3、[ButterKnife源码分析](https://www.jianshu.com/p/0f3f4f7ca505)

4、[butterknife 源码分析](https://blog.csdn.net/gdutxiaoxu/article/details/71512754)


## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f719f80268c4bb2b5a3faff9c043c3e~tplv-k3u1fbpfcp-zoom-1.image" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/4318537403878167](https://juejin.im/user/4318537403878167)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/4318537403878167)上一起分享知识。