---

		title:   Android View的绘制流程
		date: 2018/10/28 20:41:00   
		tags: 
		- Android进阶
		categories: 安卓进阶
		thumbnail: http://www.chinadaily.com.cn/dfpd/whly/attachement/jpg/site1/20120831/0013729e46ae11aaa1e32c.jpg
---

---

## 前言

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

上一篇中我们讲到了[Android的触摸事件传递机制](https://jsonchao.github.io/2018/10/17/Android%E8%A7%A6%E6%91%B8%E4%BA%8B%E4%BB%B6%E4%BC%A0%E9%80%92%E6%9C%BA%E5%88%B6/)，除此之外，关于Android View的绘制流程这一块也是View相关的核心知识点。我们都知道，PhoneWindow是Android系统中最基本的窗口系统，每个Activity会创建一个。同时，PhoneWindow也是Activity和View系统交互的接口。DecorView本质上是一个FrameLayout，是Activity中所有View的祖先。

### 一、开始：DecorView被加载到Window中

从Activity的startActivity开始，最终调用到ActivityThread的handleLaunchActivity方法来创建Activity，相关核心代码如下：

    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    
        ....
        // 创建Activity，会调用Activity的onCreate方法
        // 从而完成DecorView的创建
        Activity a = performLaunchActivity(r, customIntent);
        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            Bundle oldState = r.state;
            handleResumeActivity(r.tolen, false, r.isForward, !r.activity..mFinished && !r.startsNotResumed);
        }
    }
    
    final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume) {
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;
        // 调用Activity的onResume方法
        ActivityClientRecord r = performResumeActivity(token, clearHide);
        if (r != null) {
            final Activity a = r.activity;
            ...
            if (r.window == null &&& !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                // 得到DecorView
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                // 得到了WindowManager，WindowManager是一个接口
                // 并且继承了接口ViewManager
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
                    // WindowManager的实现类是WindowManagerImpl，
                    // 所以实际调用的是WindowManagerImpl的addView方法
                    wm.addView(decor, l);
                }
            }
        }
    }
    
    public final class WindowManagerImpl implements WindowManager {
        private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
        ...
        
        @Override
        public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
            applyDefaultToken(params);
            mGlobal.addView(view, params, mDisplay, mParentWindow);
        }
        ...
    }
    
在了解View绘制的整体流程之前，我们必须先了解下ViewRoot和DecorView的概念。ViewRoot对应于ViewRootImpl类，它是连接WindowManager和DecorView的纽带，View的三大流程均是通过ViewRoot来完成的。在ActivityThread中，当Activity对象被创建完毕后，会将DecorView添加到Window中，同时会创建ViewRootImpl对象，并将ViewRootImpl对象和DecorView建立关联，相关源码如下所示：
    
    // WindowManagerGlobal的addView方法
    public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
        ...
        ViewRootImpl root;
        View pannelParentView = null;
        synchronized (mLock) {
            ...
            // 创建ViewRootImpl实例
            root = new ViewRootImpl(view..getContext(), display);
            view.setLayoutParams(wparams);
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
        }
        try {
            // 把DecorView加载到Window中
            root.setView(view, wparams, panelParentView);
        } catch (RuntimeException e) {
            synchronized (mLock) {
                final int index = findViewLocked(view, false);
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
            }
            throw e;
        }
    }

### 二、了解绘制的整体流程


绘制会从根视图ViewRoot的performTraversals()方法开始，从上到下遍历整个视图树，每个View控件负责绘制自己，而ViewGroup还需要负责通知自己的子View进行绘制操作。performTraversals()的核心代码如下。

    private void performTraversals() {
        ...
        int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
        int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
        ...
        //执行测量流程
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        ...
        //执行布局流程
        performLayout(lp, desiredWindowWidth, desiredWindowHeight);
        ...
        //执行绘制流程
        performDraw();
    }
    
performTraversals的大致工作流程图如下所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2056d6cd12fc48fab03821dccc7e30a3~tplv-k3u1fbpfcp-zoom-1.image)

[显示不出来的可点击这里查看](https://user-gold-cdn.xitu.io/2020/1/10/16f8d5450ad1333a?w=1240&h=697&f=png&s=347384)

注意：
- preformLayout和performDraw的传递流程和performMeasure是类似的，唯一不同的是，performDraw的传递过程是在draw方法中通过dispatchDraw来实现的，不过这并没有本质区别。
- 获取content：


    ViewGroup content = (ViewGroup)findViewById(android.R.id.content);

- 获取设置的View：


    content.getChildAt(0);

    
### 三、理解MeasureSpec

##### 1.MeasureSpec源码解析

MeasureSpec表示的是一个32位的整形值，它的高2位表示测量模式SpecMode，低30位表示某种测量模式下的规格大小SpecSize。MeasureSpec是View类的一个静态内部类，用来说明应该如何测量这个View。MeasureSpec的核心代码如下。

    public static class MeasureSpec {
        private static final int MODE_SHIFT = 30;
        private static final int MODE_MASK = 0X3 << MODE_SHIFT;
        
        // 不指定测量模式, 父视图没有限制子视图的大小，子视图可以是想要
        // 的任何尺寸，通常用于系统内部，应用开发中很少用到。
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;
        
        // 精确测量模式，视图宽高指定为match_parent或具体数值时生效，
        // 表示父视图已经决定了子视图的精确大小，这种模式下View的测量
        // 值就是SpecSize的值。
        public static final int EXACTLY = 1 << MODE_SHIFT;
        
        // 最大值测量模式，当视图的宽高指定为wrap_content时生效，此时
        // 子视图的尺寸可以是不超过父视图允许的最大尺寸的任何尺寸。
        public static final int AT_MOST = 2 << MODE_SHIFT;
        
        // 根据指定的大小和模式创建一个MeasureSpec
        public static int makeMeasureSpec(int size, int mode) {
            if (sUseBrokenMakeMeasureSpec) {
                return size + mode;
            } else {
                return (size & ~MODE_MASK) | (mode & MODE_MASK);
            }
        }
        
        // 微调某个MeasureSpec的大小
        static int adjust(int measureSpec, int delta) {
            final int mode = getMode(measureSpec);
            if (mode == UNSPECIFIED) {
                // No need to adjust size for UNSPECIFIED mode.
                return make MeasureSpec(0, UNSPECIFIED);
            }
            int size = getSize(measureSpec) + delta;
            if (size < 0) {
                size = 0;
            }
            return makeMeasureSpec(size, mode);
        }
    }
    
MeasureSpec通过将SpecMode和SpecSize打包成一个int值来避免过多的对象内存分配，为了方便操作，其提供了打包和解包的方法，打包方法为上述源码中的makeMeasureSpec，解包方法源码如下：

    public static int getMode(int measureSpec) {
        return (measureSpec & MODE_MASK);
    }
    
    public static int getSize(int measureSpec) {
        return (measureSpec & ~MODE_MASK);
    }
    
##### 2.DecorView的MeasureSpec的创建过程：

    //desiredWindowWidth和desiredWindowHeight是屏幕的尺寸
    childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
    childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    
    private static int getRootMeaureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {
            case ViewGroup.LayoutParams.MATRCH_PARENT:
                // Window can't resize. Force root view to be windowSize.
                measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
                break;
            case ViewGroup.LayoutParams.WRAP_CONTENT：
                // Window can resize. Set max size for root view.
                measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
                break
            default:
                // Window wants to be an exact size. Force root view to be that size.
                measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
                break;
        }
        return measureSpec;
    }
    
##### 3.子元素的MeasureSpec的创建过程

    // ViewGroup的measureChildWithMargins方法
    protected void measureChildWithMargins(View child,
    int parentWidthMeasureSpec, int widthUsed,
    int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
        
        // 子元素的MeasureSpec的创建与父容器的MeasureSpec和子元素本身
        // 的LayoutParams有关，此外还和View的margin及padding有关
        final int childWidthMeasureSpec = getChildMeasureSpec(
        parentWidthMeasureSpec,
        mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin + widthUsed, 
        lp.width);
        
        final int childHeightMeasureSpec = getChildMeasureSpec(
        parentHeightMeasureSpec,
        mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin + heightUsed, 
        lp.height);
        
        child..measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
    
    public static int getChildMeasureSpec(int spec, int padding, int childDimesion) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);
        
        // padding是指父容器中已占用的空间大小，因此子元素可用的
        // 大小为父容器的尺寸减去padding
        int size = Math.max(0, specSize - padding);
        
        int resultSize = 0;
        int resultMode = 0;
        
        switch (sepcMode) {
            // Parent has imposed an exact size on us
            case MeasureSpec.EXACTLY:
                if (childDimension >= 0) {
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                    // Child wants to be our size. So be it.
                    resultSize = size;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimesion == LayoutParams.WRAP_CONTENT) {
                    // Child wants to determine its own size. It can't be
                    // bigger than us.
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                }
                break;
            
            // Parent has imposed a maximum size on us 
            case MeasureSpec.AT_MOST:
                if (childDimension >= 0) {
                    // Child wants a specific size... so be it
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                    // Child wants to be our size, but our size is not fixed.
                    // Constrain child to not be bigger than us.
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                    // Child wants to determine its own size. It can't be
                    // bigger than us.
                    resultSize = size;
                    resultMode = MeasureSpec.AT_MOST;
                }
                break;
                
            // Parent asked to see how big we want to be
            case MeasureSpec.UNSPECIFIED:
                if (childDimension >= 0) {
                    // Child wants a specific size... let him have it
                    resultSize = childDimension;
                    resultMode = MeasureSpec.EXACTLY;
                } else if (childDimension == LayoutParams.MATCH_PARENT) {
                    // Child wants to be our size... find out how big it should be
                    resultSize = 0;
                    resultMode = MeasureSpec.UNSPECIFIED;
                } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                    // Child wants to determine its own size....
                    // find out how big it should be
                    resultSize = 0;
                    resultMode == MeasureSpec.UNSPECIFIED;
                }
                break;
            }
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
    
普通View的MeasureSpec的创建规则如下：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e68713041770444a8b7e445cbe04db0a~tplv-k3u1fbpfcp-zoom-1.image)

注意：UNSPECIFIED模式主要用于系统内部多次Measure的情形，一般不需关注。
    
结论：对于DecorView而言，它的MeasureSpec由窗口尺寸和其自身的LayoutParams共同决定；对于普通的View，它的MeasureSpec由父视图的MeasureSpec和其自身的LayoutParams共同决定。

### 四、View绘制流程之Measure

##### 1.Measure的基本流程

由前面的分析可知，页面的测量流程是从performMeasure方法开始的，相关的核心代码流程如下。

    private void perormMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        ...
        // 具体的测量操作分发给ViewGroup
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        ...
    }
    
    // 在ViewGroup中的measureChildren()方法中遍历测量ViewGroup中所有的View
    protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            // 当View的可见性处于GONE状态时，不对其进行测量
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }
    
    // 测量某个指定的View
    protected void measureChild(View child, int parentWidthMeasureSpec, int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();
        
        // 根据父容器的MeasureSpec和子View的LayoutParams等信息计算
        // 子View的MeasureSpec
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec, mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec, mPaddingTop + mPaddingBottom, lp.height);
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
    
    // View的measure方法
    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        ...
        // ViewGroup没有定义测量的具体过程，因为ViewGroup是一个
        // 抽象类，其测量过程的onMeasure方法需要各个子类去实现
        onMeasure(widthMeasureSpec, heightMeasureSpec);
        ...
    }
    
    // 不同的ViewGroup子类有不同的布局特性，这导致它们的测量细节各不相同，如果需要自定义测量过程，则子类可以重写这个方法
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // setMeasureDimension方法用于设置View的测量宽高
        setMeasureDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec), 
        getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
    
    // 如果View没有重写onMeasure方法，则会默认调用getDefaultSize来获得View的宽高
    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);
        
        switch (specMode) {
            case MeasureSpec.UNSPECIFIED:
                result = size;
                break;
            case MeasureSpec.AT_MOST:
            case MeasureSpec.EXACTLY:
                result = sepcSize;
                break;
        }
        return result;
    }
    
##### 2.对getSuggestMinimumWidth的分析

    protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinmumWidth());
    }
    
    protected int getSuggestedMinimumHeight() {
        return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());
    }
    
    public int getMinimumWidth() {
        final int intrinsicWidth = getIntrinsicWidth();
        return intrinsicWidth > 0 ? intrinsicWidth : 0;
    }
    
如果View没有设置背景，那么返回android:minWidth这个属性所指定的值，这个值可以为0；如果View设置了背景，则返回android:minWidth和背景的最小宽度这两者中的最大值。

##### 3.自定义View时手动处理wrap_content时的情形

直接继承View的控件需要重写onMeasure方法并设置wrap_content时的自身大小，否则在布局中使用wrap_content就相当于使用match_parent。解决方式如下：

    protected void onMeasure(int widthMeasureSpec, 
    int height MeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
        int widtuhSpecSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
        // 在wrap_content的情况下指定内部宽/高(mWidth和mHeight)
        int heightSpecSize = MeasureSpec.AT_MOST && heightSpecMode == MeasureSpec.AT_MOST) {
            setMeasuredDimension(mWidth, mHeight);
        } else if (widthSpecMode == MeasureSpec.AT_MOST) {
            setMeasureDimension(mWidth, heightSpecSize);
        } else if (heightSpecMode == MeasureSpec.AT_MOST) {
            setMeasureDimension(widthSpecSize, mHeight);
        }
    }
    
##### 4.LinearLayout的onMeasure方法实现解析

    protected void onMeasure(int widthMeasureSpec, int hegithMeasureSpec) {
        if (mOrientation == VERTICAL) {
            measureVertical(widthMeasureSpec, heightMeasureSpec);
        } else {
            measureHorizontal(widthMeasureSpec, heightMeasureSpec);
        }
    }
    
    // measureVertical核心源码
    // See how tall everyone is. Also remember max width.
    for (int i = 0; i < count; ++i) {
        final View child = getVirtualChildAt(i);
        ...
        // Determine how big this child would like to be. If this or 
        // previous children have given a weight, then we allow it to 
        // use all available space (and we will shrink things later 
        // if need)
        measureChildBeforeLayout(
                child, i, widthMeasureSpec, 0, heightMeasureSpec,
                totalWeight == 0 ? mTotalLength : 0);
                
        if (oldHeight != Integer.MIN_VALUE) {
            lp.height = oldHeight;
        }
        
        final int childHeight = child.getMeasuredHeight();
        final int totalLength = mTotalLength;
        mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin + 
        lp.bottomMargin + getNextLocationOffset(child));
    }
    
系统会遍历子元素并对每个子元素执行measureChildBeforeLayout方法，这个方法内部会调用子元素的measure方法，这样各个子元素就开始依次进入measure过程，并且系统会通过mTotalLength这个变量来存储LinearLayout在竖直方向的初步高度。每测量一个子元素，mTotalLength就会增加，增加的部分主要包括了子元素的高度以及子元素在竖直方向上的margin等。

    // LinearLayout测量自己大小的核心源码
    // Add in our padding
    mTotalLength += mPaddingTop + mPaddingBottom;
    int heightSize = mTotalLength;
    // Check against our minimum height
    heightSize = Math.max(heightSize, getSuggestedMinimumHeight());
    // Reconcile our calculated size with the heightMeasureSpec
    int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
    heightSize = heightSizeAndState & MEASURED_SIZE_MASK；
    ...
    setMeasuredDimension(resolveSizeAndSize(maxWidth, widthMeasureSpec, childState),
    heightSizeAndState);
    
    public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);
        switch (specMode) {
            case MeasureSpec.UNSPECIFIED:
                result = size;
                break;
            case MeasureSpec.AT_MOST:
                // 高度不能超过父容器的剩余空间
                if (specSize < size) {
                    result = specSize | MEASURED_STATE_TOO_SMALL；
                } else {
                    result = size;
                }
                break;
            case MeasureSpec.EXACTLY:
                result = specSize;
                break;
        }
        return result | (childMeasuredState & MEASURED_STATE_MASK);
    }
    
##### 5.在Activity中获取某个View的宽高

由于View的measure过程和Activity的生命周期方法不是同步执行的，如果View还没有测量完毕，那么获得的宽/高就是0。所以在onCreate、onStart、onResume中均无法正确得到某个View的宽高信息。解决方式如下：
    
- Activity/View#onWindowFocusChanged

    
    // 此时View已经初始化完毕
    // 当Activity的窗口得到焦点和失去焦点时均会被调用一次
    // 如果频繁地进行onResume和onPause，那么onWindowFocusChanged也会被频繁地调用
    public void onWindowFocusChanged(boolean hasFocus) {
        super.onWindowFocusChanged(hasFocus);
        if (hasFocus) {
            int width = view.getMeasureWidth();
            int height = view.getMeasuredHeight();
        }
    }
    
- view.post(runnable)


    // 通过post可以将一个runnable投递到消息队列的尾部，// 然后等待Looper调用次runnable的时候，View也已经初
    // 始化好了
    protected void onStart() {
        super.onStart();
        view.post(new Runnable() {
            
            @Override
            public void run() {
                int width = view.getMeasuredWidth();
                int height = view.getMeasuredHeight();
            }
        });
    }

- ViewTreeObserver


    // 当View树的状态发生改变或者View树内部的View的可见// 性发生改变时，onGlobalLayout方法将被回调
    protected void onStart() {
        super.onStart();
        
        ViewTreeObserver observer = view.getViewTreeObserver();
        observer.addOnGlobalLayoutListener(new OnGlobalLayoutListener() {
            
            @SuppressWarnings("deprecation")
            @Override
            public void onGlobalLayout() {
                view.getViewTreeObserver().removeGlobalOnLayoutListener(this);
                int width = view.getMeasuredWidth();
                int height = view.getMeasuredHeight();
            }
        });
    }
    
- View.measure(int widthMeasureSpec, int heightMeasureSpec)
    
### 五、View的绘制流程之Layout
    
##### 1.Layout的基本流程

    // ViewRootImpl.java
    private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth, int desiredWindowHeight) {
        ...
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
        ...
    }
    
    // View.java
    public void layout(int l, int t, int r, int b) {
        ...
        // 通过setFrame方法来设定View的四个顶点的位置，即View在父容器中的位置
        boolean changed = isLayoutModeOptical(mParent) ? 
        set OpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
        
        ...
        onLayout(changed, l, t, r, b);
        ...
    }
    
    // 空方法，子类如果是ViewGroup类型，则重写这个方法，实现ViewGroup
    // 中所有View控件布局流程
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        
    }

##### 2.LinearLayout的onLayout方法实现解析

    protected void onlayout(boolean changed, int l, int t, int r, int b) {
        if (mOrientation == VERTICAL) {
            layoutVertical(l, t, r, b);
        } else {
            layoutHorizontal(l,)
        }
    }
    
    // layoutVertical核心源码
    void layoutVertical(int left, int top, int right, int bottom) {
        ...
        final int count = getVirtualChildCount();
        for (int i = 0; i < count; i++) {
            final View child = getVirtualChildAt(i);
            if (child == null) {
                childTop += measureNullChild(i);
            } else if (child.getVisibility() != GONE) {
                final int childWidth = child.getMeasureWidth();
                final int childHeight = child.getMeasuredHeight();
                
                final LinearLayout.LayoutParams lp = 
                        (LinearLayout.LayoutParams) child.getLayoutParams();
                ...
                if (hasDividerBeforeChildAt(i)) {
                    childTop += mDividerHeight;
                }
                
                childTop += lp.topMargin;
                // 为子元素确定对应的位置
                setChildFrame(child, childLeft, childTop + getLocationOffset(child), childWidth, childHeight);
                // childTop会逐渐增大，意味着后面的子元素会被
                // 放置在靠下的位置
                childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);
                
                i += getChildrenSkipCount(child,i)
            }
        }
    }
    
    private void setChildFrame(View child, int left, int top, int width, int height) {
        child.layout(left, top, left + width, top + height);
    }
    
注意：在View的默认实现中，View的测量宽/高和最终宽/高是相等的，只不过测量宽/高形成于View的measure过程，而最终宽/高形成于View的layout过程，即两者的赋值时机不同，测量宽/高的赋值时机稍微早一些。在一些特殊的情况下则两者不相等：
- 重写View的layout方法,使最终宽度总是比测量宽/高大100px


    public void layout(int l, int t, int r, int b) {
        super.layout(l, t, r + 100, b + 100);
    }
    
- View需要多长measure才能确定自己的测量宽/高,在前几次测量的过程中，其得出的测量宽/高有可能和最终宽/高不一致，但最终来说，测量宽/高还是和最终宽/高相同
    
### 六、View的绘制流程之Draw

##### 1.Draw的基本流程

    private void performDraw() {
        ...
        draw(fullRefrawNeeded);
        ...
    }
    
    private void draw(boolean fullRedrawNeeded) {
        ...
        if (!drawSoftware(surface, mAttachInfo, xOffest, yOffset, 
        scalingRequired, dirty)) {
            return;
        }
        ...
    }
    
    private boolean drawSoftware(Surface surface, AttachInfo attachInfo, 
    int xoff, int yoff, boolean scallingRequired, Rect dirty) {
        ...
        mView.draw(canvas);
        ...
    }
    
    // 绘制基本上可以分为六个步骤
    public void draw(Canvas canvas) {
        ...
        // 步骤一：绘制View的背景
        drawBackground(canvas);
        
        ...
        // 步骤二：如果需要的话，保持canvas的图层，为fading做准备
        saveCount = canvas.getSaveCount();
        ...
        canvas.saveLayer(left, top, right, top + length, null, flags);
        
        ...
        // 步骤三：绘制View的内容
        onDraw(canvas);
        
        ...
        // 步骤四：绘制View的子View
        dispatchDraw(canvas);
        
        ...
        // 步骤五：如果需要的话，绘制View的fading边缘并恢复图层
        canvas.drawRect(left, top, right, top + length, p);
        ...
        canvas.restoreToCount(saveCount);
        
        ...
        // 步骤六：绘制View的装饰(例如滚动条等等)
        onDrawForeground(canvas)
    }
    
##### 2.setWillNotDraw的作用

    // 如果一个View不需要绘制任何内容，那么设置这个标记位为true以后，
    // 系统会进行相应的优化。
    public void setWillNotDraw(boolean willNotDraw) {
        setFlags(willNotDraw ? WILL_NOT_DRAW : 0, DRAW_MASK);
    }
    
- 默认情况下，View没有启用这个优化标记位，但是ViewGroup会默认启用这个优化标记位。
- 当我们的自定义控件继承于ViewGroup并且本身不具备绘制功能时，就可以开启这个标记位从而便于系统进行后续的优化。
- 当明确知道一个ViewGroup需要通过onDraw来绘制内容时，我们需要显示地关闭WILL_NOT_DRAW这个标记位。

### 七、总结

View的绘制流程和事件分发机制都是Android开发中的核心知识点，也是自定义View高手的内功心法。对于一名优秀的Android开发来说，主流三方源码分析和Android核心源码分析可以说是必修课，下一篇，将会带领大家更进一步深入Android。


# 公众号

我的公众号 `JsonChao` 开通啦，如果您想第一时间获取最新文章和最新动态，欢迎扫描关注~

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/079f700ae4e04526adc56fe556adaf48~tplv-k3u1fbpfcp-zoom-1.image)

##### 参考链接：
---
1、Android开发艺术探索

2、Android进阶之光

3、Android高级进阶

4、[Android应用层View绘制流程与源码分析](https://blog.csdn.net/yanbober/article/details/46128379)

5、[Android中View绘制流程浅析](https://blog.csdn.net/sinat_35938012/article/details/81055380)


## 赞赏

如果这个库对您有很大帮助，您愿意支持这个项目的进一步开发和这个项目的持续维护。你可以扫描下面的二维码，让我喝一杯咖啡或啤酒。非常感谢您的捐赠。谢谢！

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0dd30b939fa7460bab6caa0a48001eba~tplv-k3u1fbpfcp-zoom-1.image" width=20%><img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3703697af9784318995c58439b212592~tplv-k3u1fbpfcp-zoom-1.image" width=20%>
</div>


----

## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5b0bc2fab6724721a3ab3e4367de6d2c~tplv-k3u1fbpfcp-zoom-1.image" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/4318537403878167](https://juejin.im/user/4318537403878167)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/4318537403878167)上一起分享知识。

