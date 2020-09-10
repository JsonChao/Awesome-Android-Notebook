---

		title:  深入探索Android布局优化（中）
		date: 2020/1/14 22:02:00   
		tags: 
		- 性能优化
		categories: 性能优化
		thumbnail: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1557665970516&di=b58d306a0db07efca58f8c9b655f5c13&imgtype=0&src=http%3A%2F%2Fimg02.tooopen.com%2Fimages%2F20160520%2Ftooopen_sl_055418231108.jpg
---

---

# 前言

### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

Android的绘制优化其实可以分为两个部分，即**布局(UI)优化和卡顿优化**，而布局优化的核心问题就是要解决因布局渲染性能不佳而导致应用卡顿的问题，所以它可以认为是卡顿优化的一个子集。对于Android开发来说，写布局可以说是一个比较简单的工作，但是如果想将写的每一个布局的渲染性能提升到比较好的程度，要付出的努力是要远远超过写布局所付出的。由于布局优化这一主题包含的内容太多，因此，笔者将它分为了上、下两篇，本篇，即为深入探索Android布局优化的上篇。本篇包含的主要内容如下所示：

- 4、布局加载原理
- 5、获取界面布局耗时


# 四、布局加载原理

## 1、为什么要了解Android布局加载原理？

知其然知其所以然，不仅要明白在平时开发过程中是怎样对布局API进行调用，还要知道它内部的实现原理是什么。明白具体的实现原理与流程之后，我们可能会发现更多可优化的点。

## 2、布局加载源码分析

我们都知道，Android的布局都是通过setContentView()这个方法进行设置的，那么它的内部肯定实现了布局的加载，接下来，我们就详细分析下它内部的实现原理与流程。

以[Awesome-WanAndroid](https://github.com/JsonChao/Awesome-WanAndroid)项目为例，我们在通用Activity基类的onCreate方法中进行了布局的设置：

    setContentView(getLayoutId());


点进去，发现是调用了AppCompatActivity的setContentView方法：

    @Override
    public void setContentView(@LayoutRes int layoutResID) {
        getDelegate().setContentView(layoutResID);
    }
    
    
这里的setContentView其实是AppCompatDelegate这个代理类的抽象方法：

     /**
     * Should be called instead of {@link Activity#setContentView(int)}}
     */
    public abstract void setContentView(@LayoutRes int resId);
    
    
**在这个抽象方法的左边，会有一个绿色的小圆圈，点击它就可以查看到对应的实现类与方法**，这里的实现类是AppCompatDelegateImplV9，实现方法如下所示：

     @Override
    public void setContentView(int resId) {
        ensureSubDecor();
        ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);
        contentParent.removeAllViews();
        LayoutInflater.from(mContext).inflate(resId, contentParent);
        mOriginalWindowCallback.onContentChanged();
    }
    
    
setContentView方法中主要是获取到了content父布局，移除其内部所有视图之后并**最终调用了LayoutInflater对象的inflate去加载对应的布局**。接下来，我们关注inflate内部的实现：

    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
        return inflate(resource, root, root != null);
    }
    

这里只是调用了inflate另一个的重载方法：

    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }

        // 1
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            // 2
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
    

在注释1处，通过Resources的getLayout方法获取到了一个XmlResourceParser对象，继续跟踪下getLayout方法：

    public XmlResourceParser getLayout(@LayoutRes int id) throws NotFoundException {
        return loadXmlResourceParser(id, "layout");
    }
    
    
这里继续调用了loadXmlResourceParser方法，**注意第二个参数传入的为layout，说明此时加载的是一个Xml资源布局解析器**。我们继续跟踪loadXmlResourceParse方法：


    @NonNull
    XmlResourceParser loadXmlResourceParser(@AnyRes int id, @NonNull String type)
            throws NotFoundException {
        final TypedValue value = obtainTempTypedValue();
        try {
            final ResourcesImpl impl = mResourcesImpl;
            impl.getValue(id, value, true);
            if (value.type == TypedValue.TYPE_STRING) {
                // 1
                return impl.loadXmlResourceParser(value.string.toString(), id,
                        value.assetCookie, type);
            }
            throw new NotFoundException("Resource ID #0x" + Integer.toHexString(id)
                    + " type #0x" + Integer.toHexString(value.type) + " is not valid");
        } finally {
            releaseTempTypedValue(value);
        }
    }
    

在注释1处，如果值类型为字符串的话，则调用了ResourcesImpl实例的loadXmlResourceParser方法。我们首先看看这个方法的注释：

    /**
     * Loads an XML parser for the specified file.
     *
     * @param file the path for the XML file to parse
     * @param id the resource identifier for the file
     * @param assetCookie the asset cookie for the file
     * @param type the type of resource (used for logging)
     * @return a parser for the specified XML file
     * @throws NotFoundException if the file could not be loaded
     */
    @NonNull
    XmlResourceParser loadXmlResourceParser(@NonNull String file, @AnyRes int id, int assetCookie,
            @NonNull String type)
            throws NotFoundException {
            
            ...
            
            final XmlBlock block = mAssets.openXmlBlockAsset(assetCookie, file);
            
            ...
            
            return block.newParser();
            
            ...
    }
    

注释的意思说明了这个方法是用于**加载指定文件的Xml解析器**，这里我们之间查看关键的mAssets.openXmlBlockAsset方法，这里的mAssets对象是AssetManager类型的，看看AssetManager实例的openXmlBlockAsset方法做了什么处理：

    /**
     * {@hide}
     * Retrieve a non-asset as a compiled XML file.  Not for use by
     * applications.
     * 
     * @param cookie Identifier of the package to be opened.
     * @param fileName Name of the asset to retrieve.
     */
    /*package*/ final XmlBlock openXmlBlockAsset(int cookie, String fileName)
        throws IOException {
        synchronized (this) {
            if (!mOpen) {
                throw new RuntimeException("Assetmanager has been closed");
            }
            // 1
            long xmlBlock = openXmlAssetNative(cookie, fileName);
            if (xmlBlock != 0) {
                XmlBlock res = new XmlBlock(this, xmlBlock);
                incRefsLocked(res.hashCode());
                return res;
            }
        }
        throw new FileNotFoundException("Asset XML file: " + fileName);
    }


可以看到，最终是调用了注释1处的openXmlAssetNative方法，这是定义在AssetManager中的一个Native方法：

        private native final long openXmlAssetNative(int cookie, String fileName);


与此同时，**我们可以猜到读取Xml文件肯定是通过IO流的方式进行的，而openXmlBlockAsset方法后抛出的IOException异常也验证了我们的想法**。因为涉及到IO流的读取，所以这里是Android布局加载流程一个耗时点
，也有可能是我们后续优化的一个方向。

分析完Resources实例的getLayout方法的实现之后，我们继续跟踪inflate方法的注释2处：

    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }

        // 1
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            // 2
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
    
    
infalte的实现代码如下：

    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            
            ...

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                final String name = parser.getName();

                ...
                
                // 1
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    // 2
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                    
                    ...
                }
                ...
            }
            ...
        }
        ...
    }


可以看到，infalte内部是通过XmlPull解析的方式对布局的每一个节点进行创建对应的视图的。首先，在注释1处会判断节点是否是merge标签，如果是，则对merge标签进行校验，如果merge节点不是当前布局的父节点，则抛出异常。然后，在注释2处，**通过createViewFromTag方法去根据每一个标签创建对应的View视图**。我们继续跟踪下createViewFromTag方法的实现：

    private View createViewFromTag(View parent, String name, Context context, AttributeSet attrs) {
        return createViewFromTag(parent, name, context, attrs, false);
    }
    
     View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
            boolean ignoreThemeAttr) {
       
        ...
       
        try {
            View view;
            if (mFactory2 != null) {
                view = mFactory2.onCreateView(parent, name, context, attrs);
            } else if (mFactory != null) {
                view = mFactory.onCreateView(name, context, attrs);
            } else {
                view = null;
            }

            if (view == null && mPrivateFactory != null) {
                view = mPrivateFactory.onCreateView(parent, name, context, attrs);
            }

            if (view == null) {
                final Object lastContext = mConstructorArgs[0];
                mConstructorArgs[0] = context;
                try {
                    if (-1 == name.indexOf('.')) {
                        view = onCreateView(parent, name, attrs);
                    } else {
                        view = createView(name, null, attrs);
                    }
                } finally {
                    mConstructorArgs[0] = lastContext;
                }
            }

            return view;
        } 
        ...
    }
    

在createViewFromTag方法中，首先会判断mFactory2是否存在，存在就会使用mFactory2的onCreateView方法区创建视图，否则就会调用mFactory的onCreateView方法，接下来，如果此时的tag是一个Fragment，则会调用mPrivateFactory的onCreateView方法，否则的话，最终都会调用LayoutInflater实例的createView方法：

     public final View createView(String name, String prefix, AttributeSet attrs)
            throws ClassNotFoundException, InflateException {
       
       ...

        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);

            if (constructor == null) {
                // Class not found in the cache, see if it's real, and try to add it
                // 1
                clazz = mContext.getClassLoader().loadClass(
                        prefix != null ? (prefix + name) : name).asSubclass(View.class);

                if (mFilter != null && clazz != null) {
                    boolean allowed = mFilter.onLoadClass(clazz);
                    if (!allowed) {
                        failNotAllowed(name, prefix, attrs);
                    }
                }
                // 2
                constructor = clazz.getConstructor(mConstructorSignature);
                constructor.setAccessible(true);
                sConstructorMap.put(name, constructor);
            } else {
                ...
            }

            ...

            // 3
            final View view = constructor.newInstance(args);
            if (view instanceof ViewStub) {
                // Use the same context when inflating ViewStub later.
                final ViewStub viewStub = (ViewStub) view;
                viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
            }
            mConstructorArgs[0] = lastContext;
            return view;

        } 
       ...
    }


LayoutInflater的createView方法中，首先，在注释1处，使用类加载器创建了对应的Class实例，然后在注释2处根据Class实例获取到了对应的构造器实例，并最终在注释3处通过构造器实例constructor的newInstance方法创建了对应的View对象。可以看到，**在视图节点的创建过程中采用到了反射**，我们都知道反射是比较耗性能的，**过多的反射可能会导致布局加载过程变慢**，这个点可能是后续优化的一个方向。

最后，我们来总结下Android中的布局加载流程：

- 1、在setContentView方法中，会通过LayoutInflater的inflate方法去加载对应的布局。
- 2、inflate方法中首先会调用Resources的getLayout方法去通过IO的方式去加载对应的Xml布局解析器到内存中。
- 3、接着，会通过createViewFromTag根据每一个tag创建具体的View对象。
- 4、它内部主要是按优先顺序为Factory2和Factory的onCreatView、createView方法进行View的创建，而createView方法内部采用了构造器反射的方式实现。

从以上分析可知，在Android的布局加载流程中，性能瓶颈主要存在两个地方：

- 1、布局文件解析中的IO过程。
- 2、创建View对象时的反射过程。


## 3、LayoutInflater.Factory分析

**在前面分析的View的创建过程中，我们明白系统会优先使用Factory2和Factory去创建对应的View，那么它们究竟是干什么的呢？**

其实LayoutInflater.Factory是layoutInflater中创建View的一个Hook，Hook即挂钩，我们可以利用它在创建View的过程中加入一些日志或进行其它更高级的定制化处理：比如可以**全局替换自定义的TextView等等**。

接下来，我们查看下Factory2的实现：

     public interface Factory2 extends Factory {
        /**
         * Version of {@link #onCreateView(String, Context, AttributeSet)}
         * that also supplies the parent that the view created view will be
         * placed in.
         *
         * @param parent The parent that the created view will be placed
         * in; <em>note that this may be null</em>.
         * @param name Tag name to be inflated.
         * @param context The context the view is being created in.
         * @param attrs Inflation attributes as specified in XML file.
         *
         * @return View Newly created view. Return null for the default
         *         behavior.
         */
        public View onCreateView(View parent, String name, Context context, AttributeSet attrs);
    }
    

可以看到，Factory2是直接继承于Factory,继续跟踪下Factory的源码：

     public interface Factory {
        /**
         * Hook you can supply that is called when inflating from a LayoutInflater.
         * You can use this to customize the tag names available in your XML
         * layout files.
         *
         * <p>
         * Note that it is good practice to prefix these custom names with your
         * package (i.e., com.coolcompany.apps) to avoid conflicts with system
         * names.
         *
         * @param name Tag name to be inflated.
         * @param context The context the view is being created in.
         * @param attrs Inflation attributes as specified in XML file.
         *
         * @return View Newly created view. Return null for the default
         *         behavior.
         */
        public View onCreateView(String name, Context context, AttributeSet attrs);
    }
    

onCreateView方法中的第一个参数就是指的tag名字，比如TextView等等，我们还注意到Factory2比Factory的onCreateView方法多一个parent的参数，这是当前创建的View的父View。看来，Factory2比Factory功能要更强大一些。

最后，我们总结下Factory与Factory2的区别：

- 1、Factory2继承与Factory。
- 2、Factory2比Factory的onCreateView方法多一个parent的参数，即当前创建View的父View。


# 五、获取界面布局耗时

## 1、常规方式

如果要获取每个界面的加载耗时，我们就必需在setContentView方法前后进行手动埋点。但是它有如下缺点：

- 1、不够优雅。
- 2、代码有侵入性。


## 2、AOP

关于AOP的使用，我在[《深入探索Android启动速度优化》](https://jsonchao.github.io/2019/11/10/%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%B4%A2Android%E5%90%AF%E5%8A%A8%E9%80%9F%E5%BA%A6%E4%BC%98%E5%8C%96/)一文的**AOP(Aspect Oriented Programming)打点**部分已经详细讲解过了，这里就不再赘述，还不了解的同学可以点击上面的链接先去学习下AOP的使用。

我们要使用AOP去获取界面布局的耗时，那么我们的切入点就是setContentView方法，声明一个@Aspect注解的PerformanceAop类，然后，我们就可以在里面实现对setContentView进行切面的方法，如下所示：

    @Around("execution(* android.app.Activity.setContentView(..))")
    public void getSetContentViewTime(ProceedingJoinPoint joinPoint) {
        Signature signature = joinPoint.getSignature();
        String name = signature.toShortString();
        long time = System.currentTimeMillis();
        try {
            joinPoint.proceed();
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        LogHelper.i(name + " cost " + (System.currentTimeMillis() - time));
    }
    

为了获取方法的耗时，我们必须使用@Around注解，这样第一个参数ProceedingJoinPoint就可以提供proceed方法去执行我们的setContentView方法，在此方法的前后就可以获取setContentView方法的耗时。后面的execution表明了在setContentView方法执行内部去调用我们写好的getSetContentViewTime方法，后面括号内的*是通配符，表示匹配任何Activity的setContentView方法，并且方法参数的个数和类型不做限定。

完成AOP获取界面布局耗时的方法之后，重装应用，打开几个Activity界面，就可以看到如下的界面布局加载耗时日志：

    2020-01-01 12:20:17.605 12297-12297/json.chao.com.wanandroid I/WanAndroid-PEGASILOG: │ [PerformanceAop.java | 36 | getSetContentViewTime] AppCompatActivity.setContentView(..) cost 174
    2020-01-01 12:20:58.010 12297-12297/json.chao.com.wanandroid I/WanAndroid-PEGASILOG: │ [PerformanceAop.java | 36 | getSetContentViewTime] AppCompatActivity.setContentView(..) cost 13
    2020-01-01 12:21:27.058 12297-12297/json.chao.com.wanandroid I/WanAndroid-PEGASILOG: │ [PerformanceAop.java | 36 | getSetContentViewTime] AppCompatActivity.setContentView(..) cost 44
    2020-01-01 12:21:31.128 12297-12297/json.chao.com.wanandroid I/WanAndroid-PEGASILOG: │ [PerformanceAop.java | 36 | getSetContentViewTime] AppCompatActivity.setContentView(..) cost 61
    2020-01-01 12:23:09.805 12297-12297/json.chao.com.wanandroid I/WanAndroid-PEGASILOG: │ [PerformanceAop.java | 36 | getSetContentViewTime] AppCompatActivity.setContentView(..) cost 22


可以看到，Awesome-WanAndroid项目里面各个界面的加载耗时一般都在几十毫秒作用，加载慢的界面可能会达到100多ms，当然，不同手机的配置不一样，但是，**这足够让我们发现哪些界面布局的加载比较慢**。


## 3、LayoutInflaterCompat.setFactory2

上面我们使用了AOP的方式监控了Activity的布局加载耗时，那么，**如果我们需要监控每一个控件的加载耗时，该怎么实现呢？**

答案是使用LayoutInflater.Factory2，我们**在基类Activity的onCreate方法中直接使用LayoutInflaterCompat.setFactory2方法对Factory2的onCreateView方法进行重写**，代码如下所示：

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {

        // 使用LayoutInflaterCompat.Factory2全局监控Activity界面每一个控件的加载耗时，
        // 也可以做全局的自定义控件替换处理，比如：将TextView全局替换为自定义的TextView。
        LayoutInflaterCompat.setFactory2(getLayoutInflater(), new LayoutInflater.Factory2() {
            @Override
            public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {

                if (TextUtils.equals(name, "TextView")) {
                    // 生成自定义TextView
                }
                long time = System.currentTimeMillis();
                // 1
                View view = getDelegate().createView(parent, name, context, attrs);
                LogHelper.i(name + " cost " + (System.currentTimeMillis() - time));
                return view;
            }

            @Override
            public View onCreateView(String name, Context context, AttributeSet attrs) {
                return null;
            }
        });

        // 2、setFactory2方法需在super.onCreate方法前调用，否则无效        
        super.onCreate(savedInstanceState);
        setContentView(getLayoutId());
        unBinder = ButterKnife.bind(this);
        mActivity = this;
        ActivityCollector.getInstance().addActivity(this);
        onViewCreated();
        initToolbar();
        initEventAndData();
    }
    

这样我们就实现了利用LayoutInflaterCompat.Factory2**全局监控Activity界面每一个控件加载耗时的处理，后续我们可以将这些数据上传到我们自己的APM服务端，作为监控数据可以分析出哪些控件加载比较耗时**。当然，这里我们也可以做全局的自定义控件替换处理，比如在上述代码中，我们可以将TextView全局替换为自定义的TextView。

然后，我们注意到这里我们使用getDelegate().createView方法来创建对应的View实例，跟踪进去发现这里的createView是一个抽象方法：

     public abstract View createView(@Nullable View parent, String name, @NonNull Context context,
            @NonNull AttributeSet attrs);
            

它对应的实现方法为AppCompatDelegateImplV9对象的createView方法，代码如下所示：

    @Override
    public View createView(View parent, final String name, @NonNull Context context,
            @NonNull AttributeSet attrs) {
        
        ...

        return mAppCompatViewInflater.createView(parent, name, context, attrs, inheritContext,
                IS_PRE_LOLLIPOP, /* Only read android:theme pre-L (L+ handles this anyway) */
                true, /* Read read app:theme as a fallback at all times for legacy reasons */
                VectorEnabledTintResources.shouldBeUsed() /* Only tint wrap the context if enabled */
        );
    }
    

这里最终又调用了AppCompatViewInflater对象的createView方法：

     public final View createView(View parent, final String name, @NonNull Context context,
            @NonNull AttributeSet attrs, boolean inheritContext,
            boolean readAndroidTheme, boolean readAppTheme, boolean wrapContext) {
        
        ...

        // We need to 'inject' our tint aware Views in place of the standard framework versions
        switch (name) {
            case "TextView":
                view = new AppCompatTextView(context, attrs);
                break;
            case "ImageView":
                view = new AppCompatImageView(context, attrs);
                break;
            case "Button":
                view = new AppCompatButton(context, attrs);
                break;
            case "EditText":
                view = new AppCompatEditText(context, attrs);
                break;
            case "Spinner":
                view = new AppCompatSpinner(context, attrs);
                break;
            case "ImageButton":
                view = new AppCompatImageButton(context, attrs);
                break;
            case "CheckBox":
                view = new AppCompatCheckBox(context, attrs);
                break;
            case "RadioButton":
                view = new AppCompatRadioButton(context, attrs);
                break;
            case "CheckedTextView":
                view = new AppCompatCheckedTextView(context, attrs);
                break;
            case "AutoCompleteTextView":
                view = new AppCompatAutoCompleteTextView(context, attrs);
                break;
            case "MultiAutoCompleteTextView":
                view = new AppCompatMultiAutoCompleteTextView(context, attrs);
                break;
            case "RatingBar":
                view = new AppCompatRatingBar(context, attrs);
                break;
            case "SeekBar":
                view = new AppCompatSeekBar(context, attrs);
                break;
        }

        if (view == null && originalContext != context) {
            // If the original context does not equal our themed context, then we need to manually
            // inflate it using the name so that android:theme takes effect.
            view = createViewFromTag(context, name, attrs);
        }

        if (view != null) {
            // If we have created a view, check its android:onClick
            checkOnClickListener(view, attrs);
        }

        return view;
    }
    

在AppCompatViewInflater对象的createView方法中系统根据不同的tag名字创建出了对应的AppCompat兼容控件。看到这里，我们明白了**Android系统是使用了LayoutInflater的Factor2/Factory结合了AppCompat兼容类来进行高级版本控件的兼容适配的。**

接下来，我们注意到注释1处，**setFactory2方法需在super.onCreate方法前调用，否则无效，这是为什么呢？**

这里可以先大胆猜测一下，可能是因为在super.onCreate()方法中就需要将Factory2实例存储到内存中以便后续使用。下面，我们就跟踪一下super.onCreate()的源码，看看是否如我们所假设的一样。AppCompatActivity的onCreate方法如下所示：

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        final AppCompatDelegate delegate = getDelegate();
        delegate.installViewFactory();
        delegate.onCreate(savedInstanceState);
        if (delegate.applyDayNight() && mThemeId != 0) {
            // If DayNight has been applied, we need to re-apply the theme for
            // the changes to take effect. On API 23+, we should bypass
            // setTheme(), which will no-op if the theme ID is identical to the
            // current theme ID.
            if (Build.VERSION.SDK_INT >= 23) {
                onApplyThemeResource(getTheme(), mThemeId, false);
            } else {
                setTheme(mThemeId);
            }
        }
        super.onCreate(savedInstanceState);
    }
    

第一行的delegate实例的installViewFactory()方法就吸引了我们的注意，因为它包含了一个敏感的关键字“Factory“，这里我们继续跟踪进installViewFactory()方法：

    public abstract void installViewFactory();
    
这里一个是抽象方法，点击左边绿色圆圈，可以看到这里具体的实现类为AppCompatDelegateImplV9，其实现的installViewFactory()方法如下所示：

    @Override
    public void installViewFactory() {
        LayoutInflater layoutInflater = LayoutInflater.from(mContext);
        if (layoutInflater.getFactory() == null) {
            LayoutInflaterCompat.setFactory2(layoutInflater, this);
        } else {
            if (!(layoutInflater.getFactory2() instanceof AppCompatDelegateImplV9)) {
                Log.i(TAG, "The Activity's LayoutInflater already has a Factory installed"
                        + " so we can not install AppCompat's");
            }
        }
    }


可以看到，如果我们在super.onCreate()方法前没有设置LayoutInflater的Factory2实例的话，这里就会设置一个默认的Factory2。最后，我们再来看下默认Factory2的onCreateView方法的实现：

    /**
     * From {@link LayoutInflater.Factory2}.
     */
    @Override
    public final View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
        // 1、First let the Activity's Factory try and inflate the view
        final View view = callActivityOnCreateView(parent, name, context, attrs);
        if (view != null) {
            return view;
        }

        // 2、If the Factory didn't handle it, let our createView() method try
        return createView(parent, name, context, attrs);
    }
    

在注释1处，我们首先会尝试让Activity的Facotry实例去加载对应的View实例，如果Factory不能够处理它，在注释2处，就会调用createView方法去创建对应的View，AppCompatDelegateImplV9类的createView方法的实现上面我们已经分析过了，此处就不再赘述了。


# 三、总结（中）

在本篇文章中，我们主要对Android的全局监控布局和控件的加载耗时进行了全面的讲解，这为大家学习《深入探索Android布局优化（下）》打下了良好的基础。下面，总结一下本篇文章涉及的两大大主题：

- 4、布局加载原理：布局加载源码分析、LayoutInflater.Factory分析。
- 5、获取界面布局耗时：使用AOP的方式去获取界面加载的耗时、利用LayoutInflaterCompat.setFactory2去监控每一个控件加载的耗时。

下篇，我们将进入布局优化的实战环节，敬请期待~


# 公众号

我的公众号 `JsonChao` 开通啦，如果您想第一时间获取最新文章和最新动态，欢迎扫描关注~

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b91ceda8770b45719e155c84ddaad9a6~tplv-k3u1fbpfcp-zoom-1.image)


##### 参考链接：
---
1、[国内Top团队大牛带你玩转Android性能分析与优化 第五章 布局优化](https://coding.imooc.com/class/308.html)

2、[极客时间之Android开发高手课 UI优化](https://time.geekbang.org/column/article/80921)

3、[手机屏幕的前世今生 可能比你想的还精彩](http://mobile.zol.com.cn/680/6809348.html)

4、[OLED 和 LCD 什么区别？](https://www.zhihu.com/question/22263252)

5、[Android 目前稳定高效的UI适配方案](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650826034&idx=1&sn=5e86768d7abc1850b057941cdd003927&chksm=80b7b1acb7c038ba8912b9a09f7e0d41eef13ec0cea19462e47c4e4fe6a08ab760fec864c777&scene=21#wechat_redirect)

6、[骚年你的屏幕适配方式该升级了!-smallestWidth 限定符适配方案](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650826381&idx=1&sn=5b71b7f1654b04a55fca25b0e90a4433&chksm=80b7b213b7c03b0598f6014bfa2f7de12e1f32ca9f7b7fc49a2cf0f96440e4a7897d45c788fb&scene=21#wechat_redirect)

7、[dimens_sw github](https://github.com/ladingwu/dimens_sw)

8、[一种极低成本的Android屏幕适配方式](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247484502&idx=2&sn=a60ea223de4171dd2022bc2c71e09351&scene=21#wechat_redirect)

9、[骚年你的屏幕适配方式该升级了!-今日头条适配方案](https://mp.weixin.qq.com/s/oSBUA7QKMWZURm1AHMyubA)

10、[今日头条屏幕适配方案终极版正式发布!](https://juejin.im/post/6844903697000972295#heading-0)

11、[使用Systrace分析UI性能](https://www.jianshu.com/p/b492140a555f)

12、[GAPID-Graphics API Debugger](https://gapid.dev/about/)

13、[Android性能优化之渲染篇](http://hukai.me/android-performance-render/)

14、[Android 屏幕绘制机制及硬件加速](https://blog.csdn.net/qian520ao/article/details/81144167)

15、[Android 图形处理官方教程](https://blog.csdn.net/qian520ao/article/details/81144167)

16、[Vulkan - 高性能渲染](https://zhuanlan.zhihu.com/p/20712354)

17、[Android Vulkan Tutorial](https://source.android.com/devices/graphics/arch-vulkan)

18、[Test UI performance-gfxinfo](https://developer.android.com/training/testing/performance#top_of_page)

19、[使用dumpsys gfxinfo 测UI性能（适用于Android6.0以后）](https://www.jianshu.com/p/7477e381a7ea)

20、[TextureView API](https://developer.android.com/reference/android/view/TextureView)

21、[PrecomputedText API](https://developer.android.com/reference/android/text/PrecomputedText)

22、[Litho Tutorial](https://fblitho.com/docs/tutorial)

23、[基本功 | Litho的使用及原理剖析](https://tech.meituan.com/2019/03/14/litho-use-and-principle-analysis.html)

24、[Flutter官方文档中文版](https://flutter.cn/docs/resources/technical-overview)

25、[[Google Flutter 团队出品] 深入了解 Flutter 的高性能图形渲染](https://www.bilibili.com/video/av48772383/?spm_id_from=333.788.videocard.0)

26、[Flutter渲染机制—UI线程](http://gityuan.com/2019/06/15/flutter_ui_draw/)

27、[RenderThread:异步渲染动画](https://mp.weixin.qq.com/s?__biz=MzUyMDAxMjQ3Ng==&mid=2247489230&amp;idx=1&amp;sn=adc193e35903ab90a4c966059933a35a&source=41#wechat_redirect)

28、[RenderScript官方文档](https://developer.android.com/guide/topics/renderscript/compute)

29、[RenderScript :简单而快速的图像处理](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2016/0504/4205.html?utm_source=itdadao&utm_medium=referral)

30、[RenderScript渲染利器](https://www.jianshu.com/p/b72da42e1463)


## 赞赏

如果这个库对您有很大帮助，您愿意支持这个项目的进一步开发和这个项目的持续维护。你可以扫描下面的二维码，让我喝一杯咖啡或啤酒。非常感谢您的捐赠。谢谢！

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ac9e50387674040860cdfd50d67fa80~tplv-k3u1fbpfcp-zoom-1.image" width=20%><img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02cfec0590ce4466b980e3ab72334651~tplv-k3u1fbpfcp-zoom-1.image" width=20%>
</div>


----

## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/415f15ac5d784f5d974f3ca9abe9d05c~tplv-k3u1fbpfcp-zoom-1.image" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/4318537403878167](https://juejin.im/user/4318537403878167)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/4318537403878167)上一起分享知识。