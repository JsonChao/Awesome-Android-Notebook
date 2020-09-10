---

		title:   Android触摸事件传递机制
		date: 2018/10/17 22:38:00   
		tags: 
		- Android进阶
		categories: 安卓进阶
		thumbnail: https://i.ytimg.com/vi/YoJCu8IUSwk/maxresdefault.jpg
---

---

## 前言

#### 成为一名优秀的Android开发，需要一份完备的[知识体系](https://github.com/JsonChao/Awesome-Android-Exercise)，在这里，让我们一起成长为自己所想的那样~。

### 一、了解Activity的构成

一个Activity包含了一个Window对象，这个对象是由PhoneWindow来实现的。PhoneWindow将DecorView作为整个应用窗口的根View，而这个DecorView又将屏幕划分为两个区域：一个是TitleView，另一个是ContentView，而我们平时所写的就是展示在ContentView中的，下图表示Activity的构成。

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/51530d267964446497d4b2543a244967~tplv-k3u1fbpfcp-zoom-1.image)

### 二、触摸事件的类型

触摸事件对应的是MotionEvent类，事件的类型主要有如下三种：

- ACTION_DOWN
- ACTION_MOVE(移动的距离超过一定的阈值会被判定为ACTION_MOVE操作)
- ACTION_UP

### 三、事件传递的三个阶段

- 分发（dispatchTouchEvent）：方法返回值为true表示事件被当前视图消费掉；返回为super.dispatchTouchEvent表示继续分发该事件。
- 拦截（onInterceptTouchEvent）：方法返回值为true表示拦截这个事件并交由自身的onTouchEvent方法进行消费；返回false表示不拦截，需要继续传递给子视图。
如果return super.onInterceptTouchEvent(ev)， 事件拦截分两种情况: 　


    1.如果该View(ViewGroup)存在子View且点击到了该子View, 则不拦截, 继续分发
    给子View 处理, 此时相当于return false。
    2.如果该View(ViewGroup)没有子View或者有子View但是没有点击中子View(此时ViewGroup
    相当于普通View), 则交由该View的onTouchEvent响应，此时相当于return true。 
    注意：一般的LinearLayout、 RelativeLayout、FrameLayout等ViewGroup默认不拦截， 而
    ScrollView、ListView等ViewGroup则可能拦截，得看具体情况。
- 消费（onTouchEvent）：方法返回值为true表示当前视图可以处理对应的事件；返回值为false表示当前视图不处理这个事件，它会被传递给父视图的onTouchEvent方法进行处理。如果return super.onTouchEvent(ev)，事件处理分为两种情况：


    1.如果该View是clickable或者longclickable的,则会返回true, 表示消费
    了该事件, 与返回true一样;
    2.如果该View不是clickable或者longclickable的,则会返回false, 表示不
    消费该事件,将会向上传递,与返回false一样.

注意：在Android系统中，拥有事件传递处理能力的类有以下三种。
- Activity：拥有分发和消费两个方法。
- ViewGroup：拥有分发、拦截和消费三个方法。
- View：拥有分发、消费两个方法。

### 四、Activity对点击事件的分发过程

我们对触摸屏进行操作时，Linux就会收到相应的硬件中断，然后将中断加工成原始的输入事件并写入相应的设备节点中。而我们的Android 输入系统所做的事情概括起来说就是监控这些设备节点，当某个设备节点有数据可读时，将数据读出并进行一系列的翻译加工，然后在所有的窗口中找到合适的事件接收者，并派发给它。
当点击事件产生后，事件会传递给当前的Activity，由Activity中的PhoneWindow完成，PhoneWindow再把事件处理工作交给DecorView，之后再有DecorView将事件处理工作交给ViewGroup。源码流程如下所示：

##### 1.Activity#dispatchTouchEvent

    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        // 由Activity所附属的Window分发，返回true，事件循环结束
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        // 返回false意味着事件没人处理，所有View的onTouchEvent都
        // 返回了false，那么Activity的onTouchEvent就会被调用
        return onTouchEvent(ev);
    }
    
##### 2.抽象类Window#superDispatchTouchEvent

    public abstract boolean superDispatchTouchEvent(MotionEvent event);
    
##### 3.唯一实现类PhoneWindow#superDispatchTouchEvent

    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }

### 五、View的事件分发机制

事件分发到ViewGroup的dispatchTouchEvent方法，如果它的onInterceptTouchEvent返回true，则由自己处理，这时如果它的mOnTouchListener被设置，则onTouch会被调用，否则onTouchEvent会被调用。在onTouchEvent中，如果设置了mOnCLickListener，则onClick会被调用。如果它的onInterceptTouchEvent返回false，则交给点击事件链上的子View处理，如此循环，完成分发。ViewGroup#dispatchTouchEvent关键源码如下所示：

##### 1.ViewGroup会在ACTION_DOWN事件到来时做重置状态操作

    // Handle an initial down.
    if (actionMasked === MotionEvent.ACTION_DOWN) {
        // Throw away all previous state when starting a new touch gesture.
        // The framework may have dropped the up or cancel event for the
        // previous gesture due to an app switch, ANR, or some other stae change.
        cancelAndClearTouchTarget(ev);
        // 在此方法中会重置FLAG_DISALLOW_INTERCEPT
        resetTouchState();
    }

##### 2.处理当前View是否拦截点击事件

    final boolean interception；
    // 当事件由ViewGorup的子元素成功处理时，mFirstTouchTarget会被赋值
    // 并指向子元素，反之，被ViewGroup拦截时，mFirstTouchTarget则为null。
    if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
        // 在子View中通过requestDisallowInterceptTouchEvent方法来设置
        // FLAG_DISALLOW_INTERCEPT,此时ViewGroup将无法拦截除ACTION_DOWN以外的其他事件 
        final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
        if (!disallowintercept) {
            intercepted = onInterceptTouchEvent(ev);
            //re store action in case it was changed
            ev.setAction(action);
        } else {
            intercepted = false;
        } else {
            // There are no touch targets and this action is not an initial down so this 
            // view group continues to intercept touches（ACTION_MOVE、ACTION_UP.eg).
            intercepted = true;
        }
    }
    
##### 3.dispatchTouchEvent()方法剩余的部分源码

    public boolean dispatchTouchEvent(MotionEvent ev) {
        ...
        final View[] children = mChildren;
        // 遍历ViewGroup的子元素，如果子元素能够接受到点击事件，则交给子元素处理。
        for (int i = childrenCount - 1;i >= 0;i--) {
            final int childIndex = customOrder ? getChildDrawingOrder(childrenCount, i) : i;
            final View child = (preorderedList == null) ? children[childIndex] : preorderedList.get(childIndex);
            if (childWithAccessibilityFocus != null) {
                if (childWithAccessibilityFocus != child) {
                    continue;
                }
                childWithAccessibilityFocus = null;
                i = childrenCount - 1;
                }
                // 判断触摸点的位置是否在子View的范围内或者子View是否在播放动画，有一项
                // 不符合则开始遍历下一个子View。
                if (!canViewReceivePointerEvents(child) || !isTransformedTouchPointInView(x, y, child, null)) {
                    ev.setTargetAccessibilityFocus(false);
                    continue;
                }
                newTouchTarget == getTouchTarget(child);
                if (newTouchTarget != null) {
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                    break;
                }
                resetCancelNextUpFlag(child);
                if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                    mLastTouchDownTime = ev.getDownTime();
                    if (preorderedList != null) {
                        for (int j = 0;j < childrenCOunt;j++) {
                            if (children[childIndex] == mChildren[j]) {
                                mLastTouchDownIndex = j;
                                break;
                            }
                        }
                    } else {
                        mLastTouchDownIndex = childIndex;
                    }
                    mLastTouchDownX == ev.getX();
                    mLastTouchDownY = ev.getY();
                    newTouchTarget = addTouchTarget(child, idBitsToAssign);
                    alreadyDispatchedToNewTouchTarget == true;
                    break
                }
                ev.setTargetAccessibilityFocus(false);
            }
        ...
    }

##### 4.在dispathcTransformedTouchEvent方法中执行真正的分发逻辑

    private boolean dispatchTransformedTouchEvent(MotionEvent event,boolean cancel,View child,int desiredPointerIdBits) {
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            // 有子View，则调用子View的dispatchTouchEvent(event)方法，如果没有子View，
            // 则调用super.dispatchTouchEvent(event)方法。
            if (child == null) {
                handled == super..dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }
        ...
    }
    
##### 5.事件传递到View的dispatchTouchEvent()

    public boolean dispatchTouchEvent(MotionEvent event) {
        ...
        boolean result = false;
        if (onFilterTouchEventForSecurity(event)) {
            ListenerInfo li = mListenerInfo;
            // onTouch方法优先级要高于onTouchEvent(event)方法
            if (li != null && li.mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }
            if (!result && onTouchEvent(event)) {
                result == true;
            }
        }
        ...
        return result;
    }

##### 6.事件传递到View的onTouchEvent()

    public boolean onTouchEvent(MotionEvent event) {
        ...
        final int action = event.getAction();
        // 只要View的CLICKABLE和LONG_CLICKABLE有一个为true，onTouchEvent()就会
        // 返回true消耗这个事件。
        if ((viewFlags & CLICKABLE) == CLICKABLE || （viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
            switch(action) {
                case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivatFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        boolean focusTaken = false;
                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            removeLongPressCallback();
                            if (!focusTaken) {
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }
                    }
                    ...
                }
                return true;
        }
        return true;
    }
    
##### 7.在ACTION_UP事件中会调用performCLick()方法

    public boolean performClick() {
        final boolean result;
        final Listenerinfo li = mListenerInfo;
        // 如果View设置了点击事件，onClick方法就会执行。
        if (li != null && li.mOnClickListener !== null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            li.mOnClickListener.onClick(this);
            result = true;
        } else {
            result = false;
        }
        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
        return result;
    }

由以上源码分析可得出View完整的点击事件传递流程如下图所示。

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6154da8a924448ec99fa37f5b599646e~tplv-k3u1fbpfcp-zoom-1.image)

### 六、总结：点击事件分发的传递规则

由事件分发的源码分析可知点击事件分发的3个重要方法的关系，用伪代码表示为：

    public boolean diapatchTouchEvent(MotionEvent ev) {
        boolean consume = false;
        if (onInterceptTouchEvent(ev)) {
            consume = onTouchEvent(ev);
        } else {
            consume = child.dispatchTouchEvent(ev);
        }
        return consume;
    }
    
一些重要的结论：

1.事件传递优先级：onTouchListener.onTouch > onTouchEvent > onClickListener.onClick。

2.正常情况下，一个时间序列只能被一个View拦截且消耗。因为一旦一个元素拦截了此事件，那么同一个事件序列内的所有事件都会直接交给它处理（即不会再调用这个View的拦截方法去询问它是否要拦截了，而是把剩余的ACTION_MOVE、ACTION_DOWN等事件直接交给它来处理）。特例：通过将重写View的onTouchEvent返回false可强行将事件转交给其他View处理。

3.如果View不消耗除ACTION_DOWN以外的其他事件，那么这个点击事件会消失，此时父元素的onTouchEvent并不会被调用，并且当前View可以持续收到后续的事件，最终这些消失的点击事件会传递给Activity处理。

4.ViewGroup默认不拦截任何事件（返回false）。

5.View的onTouchEvent默认都会消耗事件（返回true），除非它是不可点击的（clickable和longClickable同时为false）。View的longClickable属性默认都为false，clickable属性要分情况，比如Button的clickable属性默认为true，而TextView的clickable默认为false。

6.View的enable属性不影响onTouchEvent的默认返回值。

7.通过requestDisallowInterceptTouchEvent方法可以在子元素中干预父元素的事件分发过程，但是ACTION_DOWN事件除外。

最终完整的事件分发流程图如下所示：

![image](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/722474ded4664c868befde04808119b0~tplv-k3u1fbpfcp-zoom-1.image)


# 公众号

我的公众号 `JsonChao` 开通啦，如果您想第一时间获取最新文章和最新动态，欢迎扫描关注~

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bce509ff5dfc4b24870d5ce08604ab31~tplv-k3u1fbpfcp-zoom-1.image)


##### 参考链接：
---
1、Android开发艺术探索

2、Android进阶之光

3、Android高级进阶

4、[Gityuan Android事件分发机制](http://gityuan.com/2015/09/19/android-touch/)

5、[通俗理解Android的事件分发机制](https://mp.weixin.qq.com/s?__biz=MzIwMTAzMTMxMg==&mid=2649492294&idx=1&sn=1645fa7730dbb1c627bd374e917cd557&chksm=8eec80b9b99b09af1938836b2f19afc60b46c284050b9f17ceb5018867cd86df1c6a40fe56eb&scene=38#wechat_redirect)

6、[Android事件分发机制详解：史上最全面、最易懂](http://www.jianshu.com/p/38015afcdb58)

7、[Android开发之漫漫长途 Ⅵ——图解Android事件分发机制（深入底层源码）](http://mp.weixin.qq.com/s?__biz=MzIxNzU1Nzk3OQ==&mid=2247486486&idx=1&sn=7acc1c9dd8c600ad0ec2db7d32f82f1f&chksm=97f6b2a2a0813bb425cf8bf329bf0e856d3769ac8e21ed5a9a6cb7c57b1097c41f94afe4202d&scene=38#wechat_redirect)



## 赞赏

如果这个库对您有很大帮助，您愿意支持这个项目的进一步开发和这个项目的持续维护。你可以扫描下面的二维码，让我喝一杯咖啡或啤酒。非常感谢您的捐赠。谢谢！

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/534068ff08d44c9097bbbc45c0b0252a~tplv-k3u1fbpfcp-zoom-1.image" width=20%><img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ffc93060e0bd4c5bad2d66682f55af4f~tplv-k3u1fbpfcp-zoom-1.image" width=20%>
</div>


----

## Contanct Me

###  ●  微信：

> 欢迎关注我的微信：`bcce5360`  

###  ●  微信群：

> **微信群如果不能扫码加入，麻烦大家想进微信群的朋友们，加我微信拉你进群。**

<div align="center">
<img src="//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/83711ccc9a0947d597edeefeecd32119~tplv-k3u1fbpfcp-zoom-1.image" width=35%>
</div>
        

###  ●  QQ群：

> 2千人QQ群，**Awesome-Android学习交流群，QQ群号：959936182**， 欢迎大家加入~


### About me

- #### Email: [chao.qu521@gmail.com]()
- #### Blog: [https://jsonchao.github.io/](https://jsonchao.github.io/)
- #### 掘金: [https://juejin.im/user/4318537403878167](https://juejin.im/user/4318537403878167)
    


#### 很感谢您阅读这篇文章，希望您能将它分享给您的朋友或技术群，这对我意义重大。

#### 希望我们能成为朋友，在 [Github](https://github.com/JsonChao)、[掘金](https://juejin.im/user/4318537403878167)上一起分享知识。
