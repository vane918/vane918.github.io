---
layout: post
title:  "安卓UI系列一:UI绘制流程"
date:   2019-11-01 00:00:00
catalog:  true
tags:
    - android
    - view绘制
---

## 1. View是如何被添加到屏幕窗口上

启动app界面，首先在Activity的`onCreate`方法中调用`setContentView（R.layout.activity）`方法进行布局加载，下面从`setContentView`开始分析源码View是如何被添加到屏幕窗口上的。

### 1.1创建顶层布局容器DecorView

app的activity首先在Activity.java中调用`setContentView`方法，如下：

```java
public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
```

`getWindow`方法返回到是一个Window类型的对象，Window是一个抽象类，它唯一的实现类是PhoneWindow，它的`setContentView`实现如下：

```java
@Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```

主要做了两件事：

1. 调用`installDecor()`创建DecorView
2. 通过LayoutInflater对象调用了`inflate(layoutResID, mContentParent)`解析了当前传进来的layoutResID布局资源ID.

首先看下`installDecor()`主要做了什么事情。

```java
private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
            ...
```

首先判断DecorView是否为空，为空就调用`generateDecor(-1)`创建一个DecorView。`generateDecor()` 直接new一个DecorView对象，DecorView继承了FrameLayout，也就是说DecorView是一个容器。

```java
protected DecorView generateDecor(int featureId) {
        // System process doesn't have application context and in that case we need to directly use
        // the context we have. Otherwise we want the application context, so we don't cling to the
        // activity.
        Context context;
        if (mUseDecorContext) {
            Context applicationContext = getContext().getApplicationContext();
            if (applicationContext == null) {
                context = getContext();
            } else {
                context = new DecorContext(applicationContext, getContext().getResources());
                if (mTheme != -1) {
                    context.setTheme(mTheme);
                }
            }
        } else {
            context = getContext();
        }
        return new DecorView(context, featureId, this, getAttributes());
    }
```

### 1.2 在顶层布局中加载基础布局ViewGroup

创建完DecorView之后，如果mContentParent为空，则将创建的DecorView对象传入`generateLayout(mDecor)`。

```java
protected ViewGroup generateLayout(DecorView decor) {
        // Apply data from current theme.

        TypedArray a = getWindowStyle();

        mIsFloating = a.getBoolean(R.styleable.Window_windowIsFloating, false);
        int flagsToUpdate = (FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR)
                & (~getForcedWindowFlags());
        if (mIsFloating) {
            setLayout(WRAP_CONTENT, WRAP_CONTENT);
            setFlags(0, flagsToUpdate);
        } else {
            setFlags(FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR, flagsToUpdate);
        }
        ...
        
        // Inflate the window decor.

        int layoutResource;
        int features = getLocalFeatures();
        // System.out.println("Features: 0x" + Integer.toHexString(features));
        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            layoutResource = R.layout.screen_swipe_dismiss;
            setCloseOnSwipeEnabled(true);
        } 
        ...
        
        else {
            // Embedded, so no decoration is needed.
            layoutResource = R.layout.screen_simple;
            // System.out.println("Simple!");
        }

        mDecor.startChanging();
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);

        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        ...

        return contentParent;
```

该方法很长，前面主要是做了根据系统主题的属性设置了相对应的特性。关键的地方在于Inflate the window decor。主要做了以下几个事情。

1. 根据不同的features对layoutResource进行了初始化。
2. 调用`onResourcesLoaded()`将view添加到DecorView里面。

```java
void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
        ...
        
        final View root = inflater.inflate(layoutResource, null);
        if (mDecorCaptionView != null) {
            if (mDecorCaptionView.getParent() == null) {
                addView(mDecorCaptionView,
                        new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
            }
            mDecorCaptionView.addView(root,
                    new ViewGroup.MarginLayoutParams(MATCH_PARENT, MATCH_PARENT));
        } else {

            // Put it below the color views.
            addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        }
        mContentRoot = (ViewGroup) root;
        initializeElevation();
    }
```

`onResourcesLoaded`做的事情很简单，首先将layoutResource进行解析，创建了一个View对象。然后将创建的View对象root通过addView()方法添加到DecorView对象里面。

接着往下分析`generateLayout()`，通过`findViewById(ID_ANDROID_CONTENT)`创建了一个ViewGroup对象contentParent，ID_ANDROID_CONTENT是主容器的id。得到ViewGroup对象后，直接返回ViewGroup对象。

小结：首先根据系统主题设置相应的特性，然后通过不同的features获取了layoutResource。接着将获取到的layoutResource解析成一个view，添加到decorview中。最后通过`findViewById(ID_ANDROID_CONTENT)`创建了一个ViewGroup对象。

![ui-class-structure](/images/ui/ui-class-structure.png)

<center>图1 类关系图</center>
![view-structure](/images/ui/view-structure.png)

<center>图2 视图结构图</center>
解析一下上面的视图结构图。`generateDecor()`创建了一个DecorView对象，然后调用`generateLayout(mDecor)`根据不同的系统主题初始化了view，如上图中的screen_simple.xml布局。该布局是一个LinearLayout布局，上面是一个ViewStub,包含ActionBar,Title，下面是一个FrameLayout,它的id就是上面分析的ID_ANDROID_CONTENT。

### 1.3 将ContentView添加到基础布局中的FrameLayout

继续分析`setContentView()`。分析完`installDecor()`后，接着分析`mLayoutInflater.inflate(layoutResID, mContentParent)`。

根据上面的分析，mContentParent其实就是图2中的FrameLayout,它的id是content。然后将setContentView方法传进来的id经过解析，最终是添加到基础容器mContentParent中的。

### 1.4 总结

View被添加到屏幕窗口的流程总结如下：

首先系统会创建一个系统顶层容器：DecorView。DecorView是一个ViewGroup容器，继承FrameLayout，是PhoneWindow持有的一个实例，它是所有程序的顶层View，在系统内部进行初始化。当DecorView初始化完成之后，系统会根据系统的主题特性，去加载一个基础容器，不如说，NoActionBar，或者是DarkActionBar，不同的主题加载的基础容器也不一样。但无论如何，这样的基础容器中一定包含一个id为content的FrameLayout。而我们开发者调用`setContentView()`设置的xml布局文件是经过解析之后被添加到基础布局中的FrameLayout。

## 2.View的绘制流程

### 2.1绘制流程

```
ActivityThread.handleResumeActivity
-->WIndowManagerImpl.addView(decorView, layoutParams)
-->WIndowManagerGlobal.addView()
-->ViewRootImpl.setView(decorVIew, layoutParams, parentVIew)
-->performTraversals()
```

### 2.2绘制三大步骤

测量：ViewRootImpl.performMeasure

布局：ViewRootImpl.performLayout

绘制：ViewRootImpl.performdraw

#### 2.2.1 测量

```
ViewRootImpl.performMeasure
-->view.measure
-->view.onMeasure
-->view.setMeasuredDimension
-->setMeasuredDimensionRaw
```

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        if (mView == null) {
            return;
        }
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
           //mView是decorview
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```

[->View.java]

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int oWidth  = insets.left + insets.right;
            int oHeight = insets.top  + insets.bottom;
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }

        // Suppress sign extension for the low bytes
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
        //取缓存
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);

        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;

        // Optimize layout by avoiding an extra EXACTLY pass when the view is
        // already measured as the correct size. In API 23 and below, this
        // extra pass is required to make LinearLayout re-distribute weight.
        final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
                || heightMeasureSpec != mOldHeightMeasureSpec;
        final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
                && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;
        final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
                && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);
        final boolean needsLayout = specChanged
                && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);

        if (forceLayout || needsLayout) {
            // first clears the measured dimension flag
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

            resolveRtlPropertiesIfNeeded();

            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }

            // flag not set, setMeasuredDimension() was not invoked, we raise
            // an exception to warn the developer
            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("View with id " + getId() + ": "
                        + getClass().getName() + "#onMeasure() did not set the"
                        + " measured dimension by calling"
                        + " setMeasuredDimension()");
            }

            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }
        
        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;

        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }
```

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
```

```java
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;

            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }
```

```java
private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;

        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }
```

`setMeasuredDimensionRaw`保存了measuredWidth和measuredHeight，实际上是确定了测量时到宽和高。那么宽和高时如何确定的呢？我们回溯到之前的`measure()`方法。

我们首先了解下MeasureSpec这个类。

测量一个view实际上是确定它的模式和尺寸，这两个属性是封装到MeasureSpec类里面的。利用一个32位的int值表示view的模式和尺寸。如下所示：

00000000000000000000000000000000

前2位表示view的模式SpaceMode，后30位表示尺寸SpaceSize。

MeasureSpec类中定义了下面的三个静态常量：

```java
//父容器不对view做任何限制，系统内部使用
public static final int UNSPECIFIED = 0 << MODE_SHIFT;//00000000000000000000000000000000
//父容器检测出view大小，view的大小就是SpaceSize LayoutPamras match_parent固定大小
public static final int EXACTLY     = 1 << MODE_SHIFT;//01000000000000000000000000000000
//父容器指定一个可用大小，View的大小不能超过这个值，LayoutPamras wrap_content
public static final int AT_MOST     = 2 << MODE_SHIFT;//10000000000000000000000000000000
```

view的MeasureSpec如何确定的呢。回溯到`performTraversals()`

```java
private void performTraversals() {
  ...
    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
  performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
```

mWidth表示我们窗口的宽，lp.width表示顶层容器decorView的宽。

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // Window can resize. Set max size for root view.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }
```

`getRootMeasureSpec()`可知，如果是MATCH_PARENT模式，那么窗口不能缩小，根view就是windowSize。

如果是WRAP_CONTENT模式，那么窗口可以缩小，但最大尺寸为窗口本身的尺寸。`makeMeasureSpec()`方法是将尺寸和模式转化为MeasureSpec类型。

DecorView的MeasureSpec由窗口大小和自身LayoutParams决定，遵守如下规则：

1. LayoutParams.MATCH_PARENT:精确模式，窗口大小
2. LayoutParams.WRAP_CONTENT:最大模式，最大为窗口大小
3. 固定大小：精确模式，大小为LayoutParams大小

[->FrameLayout.java]

DecorView实际上是继承FrameLayout,下面看下FrameLayout的`onMeasure()`方法。

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();
  
  final boolean measureMatchParentChildren =
                MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
                MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
        mMatchParentChildren.clear();

        int maxHeight = 0;
        int maxWidth = 0;
        int childState = 0;
        //遍历子控件，确定子控件MeasureSpec
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
                childState = combineMeasuredStates(childState, child.getMeasuredState());
                if (measureMatchParentChildren) {
                    if (lp.width == LayoutParams.MATCH_PARENT ||
                            lp.height == LayoutParams.MATCH_PARENT) {
                        mMatchParentChildren.add(child);
                    }
                }
            }
        }
    ...
    //决定自身的宽高，依赖于子控件
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));
```

我门看下measureChildWithMargins()方法

```java
protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

上面的for循环是遍历我们的子控件，对子控件进行测量。对子控件测量的方式是调用measureChildWithMargins()方法来完成的：

1. 获取子控件的MeasureSpec
2. 调用子控件的`measure()`方法进行测量

View的MeasureSpec由父容器的MeasureSpec和自身的LayoutParams决定。

| chlidLayoutParams\parentSpecMode |      EXACTLY       |      AT_MOST       |    UNSPECIFIED    |
| :------------------------------: | :----------------: | :----------------: | :---------------: |
|              dp/px               | EXACTLY/childSize  | EXACTLY/childSize  | EXACTLY/childSize |
|           match_parent           | EXACTLY/parentSize | AT_MOST/parentSize |   UNSPECIFIED/0   |
|           wrap_content           | AT_MOST/parentSize | AT_MOST/parentSize |   UNSPECIFIED/0   |

小结：

ViewGroup测量流程：

ViewGroup measure --> onMeasure(测量子控件的宽高) --> setMeasureDimension --> setMeasuredDimensionRaw(保存自身的宽高) 

View测量流程：

View measure --> onMeasure --> setMeasureDimension --> setMeasuredDimensionRaw(保存自身的宽高) 

有一点需要注意的是：

[->View.java]

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
```

```java
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
            result = specSize;
            break;
        }
        return result;
    }
```

view的`onMeasure()`在调用`setMeasuredDimension()`前，调用了`getDefaultSize()`方法，AT_MOST和EXACTLY都返回了specSize。也就是说，在view的测量过程中，它的尺寸都赋值为测量规格里面的size，而这个size，根据上面的表，AT_MOST和EXACTLY不管是match_parent还是wrap_content，都是父容器的剩余空间parentSize。

我们自定义view的时候，不重写`onMeasure()`方法的话，那么在布局文件里面，控件的match_parent和wrap_content效果是一样的。这就是为什么我们在自定义view的时候一定要重写`onMeasure()`方法。

#### 2.2.2 布局

```
ViewRootImpl.performLayout
-->view.layout
-->view.onLayout
```

```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        ...
        final View host = mView;
        try {
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
```

```java
public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;
        //确定上下左右间距
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            //onLayout是一个空方法，由子类实现。如果是一个容器，那么需要确定子控件的摆放；
            //如果是子控件，只需要确定自身的摆放
            onLayout(changed, l, t, r, b);

```

小结：

1 调用view.layout确定自身的位置，即确定mLeft，mTop,mRight,mBottom的值。

2 如果是ViewGroup类型，需要调用onLayout确定子view的位置。

#### 2.2.3 绘制

```
ViewRootImpl.performdraw
-->ViewRootImpl.draw(fullRedrawNeeded)
-->ViewRootImpl.drawSoftware
-->view.draw(Canvas)
```

[->View.java]

```java
@CallSuper
    public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

        // Step 1, draw the background, if needed
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);
        }

        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            drawAutofilledHighlight(canvas);

            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);

            // Step 7, draw the default focus highlight
            drawDefaultFocusHighlight(canvas);

            if (debugDraw()) {
                debugDrawFocus(canvas);
            }

            // we're done...
            return;
        }
```

1. 绘制背景 drawBackground(canvas)
2. 绘制自己 onDraw(canvas)
3. 绘制子View dispatchDraw(canvas)
4. 绘制前景，滚动条等装饰  onDrawFroreground(canvas)