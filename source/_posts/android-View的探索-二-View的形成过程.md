---
title: 'android View的探索(二) : View的形成过程'
date: 2016-12-09 16:35:46
tags: android
categories: Android源码研究
---

我们在设计项目的时候，经常会遇到一些奇奇怪怪的需求，在这些需求中，对于UI的需求更是占了大半部分，这时候我们就避免不了与View打交道，熟悉自定义View的旁友应该会将onMeasure()->onLayout->onDraw的调用过程烂熟于心。因此，为了在撸代码的时候更加的对VIew得心应手，我们有必要对其进行一个深入的了解，下面就从构造方法看起吧。
<!--more-->

View的构造方法如下：
```java
public View(Context context) {

       mContext = context;
       mResources = context != null ? context.getResources() : null;
       mViewFlags = SOUND_EFFECTS_ENABLED | HAPTIC_FEEDBACK_ENABLED;
       ....

           sCompatibilityDone = true;
       }
   }
​
 public View(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
       this(context);
​
       final TypedArray a = context.obtainStyledAttributes(
               attrs, com.android.internal.R.styleable.View, defStyleAttr, defStyleRes);
​
       if (mDebugViewAttributes) {
           saveAttributeData(attrs, a);
       }
​
       Drawable background = null;
​
       ...

       final int N = a.getIndexCount();
       for (int i = 0; i < N; i++) {
           int attr = a.getIndex(i);
           switch (attr) {
               //读取attr的参数
               ...
           }
  //设置View的滚动模式
       setOverScrollMode(overScrollMode);
​
       // Cache start/end user padding as we cannot fully resolve padding here (we dont have yet
       // the resolved layout direction). Those cached values will be used later during padding
       // resolution.
       mUserPaddingStart = startPadding;
       mUserPaddingEnd = endPadding;
  //设置背景
       if (background != null) {
           setBackground(background);
       }
​
       // setBackground above will record that padding is currently provided by the background.
       // If we have padding specified via xml, record that here instead and use it.
       mLeftPaddingDefined = leftPaddingDefined;
       mRightPaddingDefined = rightPaddingDefined;
​
       ...
   }

```

对于View的构造方法而言，在其中只是做了相关参数的初始化以及传入参数attr的数据的获取的操作，相对来说，并没有什么特别需要注意的地方，那我们直接查看到onMeasure中：

```java
 /**

     * <p>
     * Measure the view and its content to determine the measured width and the
     * measured height. This method is invoked by {@link #measure(int, int)} and
     * should be overridden by subclasses to provide accurate and efficient
     * measurement of their contents.
     * </p>
     *
     * <p>
     * <strong>CONTRACT:</strong> When overriding this method, you
     * <em>must</em> call {@link #setMeasuredDimension(int, int)} to store the
     * measured width and height of this view. Failure to do so will trigger an
     * <code>IllegalStateException</code>, thrown by
     * {@link #measure(int, int)}. Calling the superclass'
     * {@link #onMeasure(int, int)} is a valid use.
     * </p>
     *
     * <p>
     * The base class implementation of measure defaults to the background size,
     * unless a larger size is allowed by the MeasureSpec. Subclasses should
     * override {@link #onMeasure(int, int)} to provide better measurements of
     * their content.
     * </p>
     *
     * <p>
     * If this method is overridden, it is the subclass's responsibility to make
     * sure the measured height and width are at least the view's minimum height
     * and width ({@link #getSuggestedMinimumHeight()} and
     * {@link #getSuggestedMinimumWidth()}).
     * </p>
     *
   */
     
   protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
       setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
               getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
   }
   
```

在View注释中已经把onMeasure()解释得相当的完善了，我们可以在注释中获取一些信息：
* onMeasure()方法中会测量其本身View以及包含内容的宽高

* 当我们在子类中重写onMeasure()方法时，我们需要调用setMeasuredDimension(int, int)为这个View进行保存宽高

* View默认的是background的大小

```java

/**

     * Utility to return a default size. Uses the supplied size if the
     * MeasureSpec imposed no constraints. Will get larger if allowed
     * by the MeasureSpec.
     *
     * @param size Default size for this view
     * @param measureSpec Constraints imposed by the parent
     * @return The size this view should be.
     */
public static int getDefaultSize(int size, int measureSpec) {
       int result = size;
       int specMode = MeasureSpec.getMode(measureSpec);
       int specSize = MeasureSpec.getSize(measureSpec);
​
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
protected int getSuggestedMinimumWidth() {
       return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
   }
protected int getSuggestedMinimumHeight() {
       return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());
​
   }

```

在上述代码中，我们可以看到在getDefaultSize()方法中通过MeasureSpec来获得测量模式和大小(MeasureSpec代表一个32位的int值，高两位代表了specMode，低30位specSize)。specMode总共有三种()
* MeasureSpec.UNSPECIFIED:父容器不对View有任何的影响，要多大就给多大

* MeasureSpec.EXACTLY:父容器检测出View所需要的精确大小，这是View的最终大小就是specSize的值，其对应了match_parent模式

* MeasureSpec.AT_MOST:父容器制定了可用大小，即specSize，View的大小不能大于这个值，对应wrap_content模式

上述三种模式的说明来自于《android艺术开发探索》。在View的测量中，我们可以看到View的大小的测量跟设置的模式以及父布局是息息相关的，因此我们在进行自定义View的时候需要考虑到这两方面的因素。
我们重新回溯到setMeasuredDimension()方法中：

```java
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {

       boolean optical = isLayoutModeOptical(this);
       if (optical != isLayoutModeOptical(mParent)) {
           Insets insets = getOpticalInsets();
           int opticalWidth = insets.left + insets.right;
           int opticalHeight = insets.top + insets.bottom;
​
           measuredWidth += optical ? opticalWidth : -opticalWidth;
           measuredHeight += optical ? opticalHeight : -opticalHeight;
       }
       setMeasuredDimensionRaw(measuredWidth, measuredHeight);
   }
 private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
       mMeasuredWidth = measuredWidth;
       mMeasuredHeight = measuredHeight;
​
       mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
   }
```
在其中我们看到最终会将计算得到的宽高传给mMeasuredHeight和mMeasuredWidth。
接下来在看看onLayout方法：
```java
 /**

     * Called from layout when this view should
     * assign a size and position to each of its children.
     *
     * Derived classes with children should override
     * this method and call layout on each of
     * their children.
     * @param changed This is a new size or position for this view
     * @param left Left position, relative to parent
     * @param top Top position, relative to parent
     * @param right Right position, relative to parent
     * @param bottom Bottom position, relative to parent
     */
   protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
   }
```
onLayout()是一个空实现的方法，从注释当中我们可以看到此方法更适用于ViewGroup，只有View是一个ViewGroup才需要考虑到layout的位置的问题。

往下走，onDraw():
```java
/**

     * Implement this to do your drawing.
     *
     * @param canvas the canvas on which the background will be drawn
     */
   protected void onDraw(Canvas canvas) {
   }
```
同样是空实现，这里自定义View通过重写该方法来进行绘制的操作。

ViewRootIpml
分析完上面的三个方法，我们更多地是看到一些赋值和空实现的方法，那么到底View的怎么开始绘制的过程的呢？这时候涉及到ViewRootIpml这个类了，它是连接View和WindowManager的桥梁，具体的作用是什么呢，我们从构建开始说起，一步步往下学习，首先我们找到其初始化的地方：
```java
//WindowManagerGlobal

   public void addView(View view, ViewGroup.LayoutParams params,

           Display display, Window parentWindow) {
           ...

​

           ViewRootImpl root;
           View panelParentView = null;

           ...

​

           root = new ViewRootImpl(view.getContext(), display);
​
           view.setLayoutParams(wparams);
​
           mViews.add(view);
           mRoots.add(root);
           mParams.add(wparams);

       }
  try {

           root.setView(view, wparams, panelParentView);
       } catch (RuntimeException e) {
           // BadTokenException or InvalidDisplayException, clean up.
           synchronized (mLock) {
               final int index = findViewLocked(view, false);
               if (index >= 0) {
                   removeViewLocked(index, true);
               }
           }
           throw e;
       }

​
       ...

   }
​
```
我们在WindowManager的实现类WindowManagerGlobal找到了其ViewRootImpl初始化的地方(关于WindowManager在以后会进行深入探讨，现在主要知道它是window窗口的管理类即可)，从addView()方法中我们推测其是将View添加到window的一个过程(addView这个方法在ActivityThread的handleResumeActivity()中实现)，然后又调用了ViewRootImpl的setView()方法：
```java
//ViewRootImpl

public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {

       synchronized (this) {
           if (mView == null) {

                 ...

               requestLayout();

              ...

               }

       }
   }
  @Override

   public void requestLayout() {

       if (!mHandlingLayoutInLayoutRequest) {
           checkThread();
           mLayoutRequested = true;
           scheduleTraversals();

       }

   }

   void scheduleTraversals() {

           if (!mTraversalScheduled) {

               mTraversalScheduled = true;
               mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
               mChoreographer.postCallback(
                       Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
               if (!mUnbufferedInputDispatch) {
                   scheduleConsumeBatchedInput();
               }
               notifyRendererOfFramePending();
               pokeDrawLockIfNeeded();
           }
       }
   final class TraversalRunnable implements Runnable {

           @Override
           public void run() {
               doTraversal();

           }
       }
   void doTraversal() {
           if (mTraversalScheduled) {
               mTraversalScheduled = false;
               mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
​
               if (mProfile) {
                   Debug.startMethodTracing("ViewAncestor");
               }
​
               performTraversals();

​
               if (mProfile) {
                   Debug.stopMethodTracing();
                   mProfile = false;
               }
           }
       }
  private void performTraversals() {

        ...

        performMeasure()；
       ...
       performLayout();

        ...

        performDraw();

     }

​
```
在这里我们可以看到当第一次进来的时候会设置mView的值，在其中调用了requestLayout()进行布局，然后依次是scheduleTraversals()->doTraversal()->performTraversals()的过程，在performTraversals()方法中调用了performMeasure(),performLayout(),performDraw()方法：
```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {

       Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
       try {
           mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
       } finally {
           Trace.traceEnd(Trace.TRACE_TAG_VIEW);
       }
   }
  private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
           int desiredWindowHeight) {
         //当前mView是由WindowsManagerGlobal.addView传递过来，在ViewRootImpl中
         //通过setView()获得到的。
         final View host = mView;
         ...
           host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
   ...
           
        mInLayout = false;
   }
 private void performDraw() {
       if (mAttachInfo.mDisplayState == Display.STATE_OFF && !mReportNextDraw) {
           return;
       }
​
       final boolean fullRedrawNeeded = mFullRedrawNeeded;
       mFullRedrawNeeded = false;
​
       mIsDrawing = true;
       Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");
       try {
           draw(fullRedrawNeeded);
       } finally {
           mIsDrawing = false;
           Trace.traceEnd(Trace.TRACE_TAG_VIEW);
       }

           try {
               mWindowSession.finishDrawing(mWindow);
           } catch (RemoteException e) {
           }
       }
   }
 private void draw(boolean fullRedrawNeeded) {

       

       ....

         if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {

           return;
         }

               

       }

       if (animating) {
           mFullRedrawNeeded = true;
           scheduleTraversals();
       }
   }
​
   /**
     * @return true if drawing was successful, false if an error occurred
     */
   private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
           boolean scalingRequired, Rect dirty) {
​
     

       mView.draw(canvas);

​
         drawAccessibilityFocusedDrawableIfNeeded(canvas);

           

       return true;
   }
​
```
在上面我们最终可以看到，measure->layout->draw的过程最终回调到了View中的对应的是三个方法,所以我们重新回到View中研究一下这些方法,从measure()看起：
```java
/**

     * <p>
     * This is called to find out how big a view should be. The parent
     * supplies constraint information in the width and height parameters.
     * </p>
     *
     * <p>
     * The actual measurement work of a view is performed in
     * {@link #onMeasure(int, int)}, called by this method. Therefore, only
     * {@link #onMeasure(int, int)} can and must be overridden by subclasses.
     * </p>
     *
     *
     * @param widthMeasureSpec Horizontal space requirements as imposed by the
     *       parent
     * @param heightMeasureSpec Vertical space requirements as imposed by the
     *       parent
     *
     * @see #onMeasure(int, int)
     */

public final void measure(int widthMeasureSpec, int heightMeasureSpec) {

       boolean optical = isLayoutModeOptical(this);

   //是否采用视觉边界

       if (optical != isLayoutModeOptical(mParent)) {

           Insets insets = getOpticalInsets();

           int oWidth = insets.left + insets.right;

           int oHeight = insets.top + insets.bottom;
           widthMeasureSpec = MeasureSpec.adjust(widthMeasureSpec, optical ? -oWidth : oWidth);
           heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
       }

        ···

       onMeasure(widthMeasureSpec, heightMeasureSpec);

       ···     

   }
```
从注释中我们可以了解到，其实该方法是计算View树的宽高的最终方法，宽高由父布局和子布局共同决定，且该方法不允许子类进行相关操作，因为其是final的，子类想要设置宽高需要通过onMeasure()方法来进行。

接下来我们再看下layout()方法：
```java
/**

     * Assign a size and position to a view and all of its

     * descendants
     *
     * <p>This is the second phase of the layout mechanism.
     * (The first is measuring). In this phase, each parent calls
     * layout on all of its children to position them.
     * This is typically done using the child measurements
     * that were stored in the measure pass().</p>
     *
     * <p>Derived classes should not override this method.
     * Derived classes with children should override
     * onLayout. In that method, they should
     * call layout on each of their children.</p>
     *
     * @param l Left position, relative to parent
     * @param t Top position, relative to parent
     * @param r Right position, relative to parent
     * @param b Bottom position, relative to parent
     */
   @SuppressWarnings({"unchecked"})
   public void layout(int l, int t, int r, int b) {
       if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
           onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
           mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
       }
​
       int oldL = mLeft;
       int oldT = mTop;
       int oldB = mBottom;
       int oldR = mRight;
​
       boolean changed = isLayoutModeOptical(mParent) ?
               setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

  //setFrame会判断是否需要重新布局，即如果我们在完成一次View展示后，重新设置View的大小，那么
      //setFrame返回tru

       if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
           onLayout(changed, l, t, r, b);

           mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
​
           ListenerInfo li = mListenerInfo;
           if (li != null && li.mOnLayoutChangeListeners != null) {
               ArrayList<OnLayoutChangeListener> listenersCopy =
                       (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
               int numListeners = listenersCopy.size();
               for (int i = 0; i < numListeners; ++i) {
                   listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
               }
           }
       }
​
       mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
       mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
   }
```
确认过大小后，我们需要定位View最终位置，在layout中会调用到onLayout()来获得我们需要展示的childrenView的效果。
draw()方法：
```java
/**

     * Manually render this view (and all of its children) to the given Canvas.
     * The view must have already done a full layout before this function is
     * called. When implementing a view, implement
     * {@link #onDraw(android.graphics.Canvas)} instead of overriding this method.
     * If you do need to override this method, call the superclass version.
     *
     * @param canvas The Canvas to which the View is rendered.
     */
   @CallSuper
   public void draw(Canvas canvas) {
       final int privateFlags = mPrivateFlags;
       final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
               (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
       mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;
​
       /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *     1. Draw the background
         *     2. If necessary, save the canvas' layers to prepare for fading

         *     3. Draw view's content

         *     4. Draw children
         *     5. If necessary, draw the fading edges and restore layers
         *     6. Draw decorations (scrollbars for instance)
         */
​
       // Step 1, draw the background, if needed
       int saveCount;
​
       if (!dirtyOpaque) {
           drawBackground(canvas);
       }
​
       // skip step 2 & 5 if possible (common case)
       final int viewFlags = mViewFlags;
       boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
       boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
       if (!verticalEdges && !horizontalEdges) {
           // Step 3, draw the content
           if (!dirtyOpaque) onDraw(canvas);
​
           // Step 4, draw the children
          //递归draw

           dispatchDraw(canvas);
​
           // Overlay is part of the content and draws beneath Foreground
           if (mOverlay != null && !mOverlay.isEmpty()) {
               mOverlay.getOverlayView().dispatchDraw(canvas);
           }
​
           // Step 6, draw decorations (foreground, scrollbars)
           onDrawForeground(canvas);
​
           // we're done...
           return;
       }
​
```
看代码注释就可以很轻松知道draw中的步骤，这里就不详细解释了。

看到这里就对我们对View从哪里开始，并且如何一步步从测量绘制完成有了一个稍微深入的认识，画一个图先总结一下吧：

{% asset_img one.png one %}

但是我们只看到了绘制完成的部分，并没有看到View是怎么展示在我们面前的，想要知道View如何被调用再到绘制完成显示出来，这就牵扯到atcivity的启动的过程了，这里不对这方面进行讲解，详细的过程可以大概看下：
http://www.jianshu.com/p/93c66b3f08d6 
我写的这篇文章(写的比较乱，有时间整理一下=￣ω￣=)，本身我们就知道activity启动完成显示到手机上会执行到onResume()这个方法，这个方法可硬对应在ActivityThread中的handleResumeActivity():
```java
   final void handleResumeActivity(...){
          ....
           else if (!willBeVisible) {
               if (localLOGV) Slog.v(
                   TAG, "Launch " + r + " mStartedActivity set");
               r.hideForNow = true;
           }
​
           // Get rid of anything left hanging around.
           cleanUpPendingRemoveWindows(r);
​
           // The window is now visible if it has been added, we are not
           // simply finishing, and we are not starting another activity.
           if (!r.activity.mFinished && willBeVisible
                   && r.activity.mDecor != null && !r.hideForNow) {
               if (r.newConfig != null) {
                   r.tmpConfig.setTo(r.newConfig);
                   if (r.overrideConfig != null) {
                       r.tmpConfig.updateFrom(r.overrideConfig);
                   }
                   if (DEBUG_CONFIGURATION) Slog.v(TAG, "Resuming activity "
                           + r.activityInfo.name + " with newConfig " + r.tmpConfig);
                   performConfigurationChanged(r.activity, r.tmpConfig);
                   freeTextLayoutCachesIfNeeded(r.activity.mCurrentConfig.diff(r.tmpConfig));
                   r.newConfig = null;
               }
               if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward="
                       + isForward);
               WindowManager.LayoutParams l = r.window.getAttributes();
               if ((l.softInputMode
                       & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                       != forwardBit) {
                   l.softInputMode = (l.softInputMode
                           & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                           | forwardBit;
                   if (r.activity.mVisibleFromClient) {
                       ViewManager wm = a.getWindowManager();
                       View decor = r.window.getDecorView();
                       wm.updateViewLayout(decor, l);
                   }
               }
               r.activity.mVisibleFromServer = true;
               mNumVisibleActivities++;
               if (r.activity.mVisibleFromClient) {
                   r.activity.makeVisible();
               }
           }
​
           if (!r.onlyLocalRequest) {
               r.nextIdle = mNewActivities;
               mNewActivities = r;
               if (localLOGV) Slog.v(
                   TAG, "Scheduling idle handler for " + r);
               Looper.myQueue().addIdleHandler(new Idler());
           }
           r.onlyLocalRequest = false;
​
           // Tell the activity manager we have resumed.
           if (reallyResume) {
               try {
                   ActivityManagerNative.getDefault().activityResumed(token);
               } catch (RemoteException ex) {
               }
           }
​
       } else {
           // If an exception was thrown when trying to resume, then
           // just end this activity.
           try {
               ActivityManagerNative.getDefault()
                   .finishActivity(token, Activity.RESULT_CANCELED, null, false);
           } catch (RemoteException ex) {
           }
       }
   }
```
代码中有句 r.activity.makeVisible()就是展示的关键的代码，他会让DecorView变为Visibile状态，最终展示出来。

-------
####ViewGrop
上面通过一系列的讲解只是从最简单的角度上而言——当页面中只有一个View的时候，但是绝大部分情况下是不可能的，往往项目中VIew树的层级是很深的(**这里建议使用RelativeLayout作为初始布局容器**)，那就涉及到ViewGroup了，其实ViewGroup也很简单，只不过多了一些递归的过程而已。

这里选择FrameLayout来进行大概的讲述吧(因为好多文章用了LinearLayout，RelativeLayout我又嫌麻烦。。)，从onMeasure()开始看：
```java
  @Override

   protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
      //获取子View的数量

       int count = getChildCount();
  
      //当前布局宽高否是match_parent
       final boolean measureMatchParentChildren =
               MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
               MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
       mMatchParentChildren.clear();
​
       int maxHeight = 0;
       int maxWidth = 0;
       int childState = 0;
​
       for (int i = 0; i < count; i++) {
           final View child = getChildAt(i);
          //GONE的布局不会进行测量

           if (mMeasureAllChildren || child.getVisibility() != GONE) {
              //测量时加上margin参数

               measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
               final LayoutParams lp = (LayoutParams) child.getLayoutParams();
               maxWidth = Math.max(maxWidth,
                       child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
               maxHeight = Math.max(maxHeight,
                       child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
               childState = combineMeasuredStates(childState, child.getMeasuredState());
               if (measureMatchParentChildren) {
                   //子布局中是否由含match_parent的布局，有则添加到mMatchParentChildren中
                   if (lp.width == LayoutParams.MATCH_PARENT ||
                           lp.height == LayoutParams.MATCH_PARENT) {
                       mMatchParentChildren.add(child);
                   }
               }
           }
       }
​
       // Account for padding too
       maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
       maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();
​
       // Check against our minimum height and width
       maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
       maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());
​
       // Check against our foreground's minimum height and width
      //计算图片宽度，取图片宽高和最大宽高两者之间最大值

       final Drawable drawable = getForeground();
       if (drawable != null) {
           maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
           maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
       }
​
       setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
               resolveSizeAndState(maxHeight, heightMeasureSpec,
                       childState << MEASURED_HEIGHT_STATE_SHIFT));​
       count = mMatchParentChildren.size();

      //这里对于子布局是match_parent的进行重新测量得到准确值
       if (count > 1) {
           for (int i = 0; i < count; i++) {
               final View child = mMatchParentChildren.get(i);
               final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
​
               final int childWidthMeasureSpec;
               if (lp.width == LayoutParams.MATCH_PARENT) {
                   final int width = Math.max(0, getMeasuredWidth()
                           - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                           - lp.leftMargin - lp.rightMargin);
                   childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                           width, MeasureSpec.EXACTLY);
               } else {
                   childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                           getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                           lp.leftMargin + lp.rightMargin,
                           lp.width);
               }
​
               final int childHeightMeasureSpec;
               if (lp.height == LayoutParams.MATCH_PARENT) {
                   final int height = Math.max(0, getMeasuredHeight()
                           - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                           - lp.topMargin - lp.bottomMargin);
                   childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                           height, MeasureSpec.EXACTLY);
               } else {
                   childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                           getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                           lp.topMargin + lp.bottomMargin,
                           lp.height);
               }
                //调用子布局的measure()方法
               child.measure(childWidthMeasureSpec, childHeightMeasureSpec);

           }
       }
   }
​
protected void measureChildWithMargins(View child,
           int parentWidthMeasureSpec, int widthUsed,
           int parentHeightMeasureSpec, int heightUsed) {
       final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
​
       final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
               mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                       + widthUsed, lp.width);
       final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
               mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                       + heightUsed, lp.height);
​            //调用子布局的measure()方法
       child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
   }
```
从onMeasure()可以看到主要的方法就是child.measure()的调用，他会调用到View的measure()，measure()又会重新调用onMeasure()由此形成依次递归直到获取到正确的测量结果。

onLayout()方法：
```java
   @Override

   protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
       layoutChildren(left, top, right, bottom, false /* no force left gravity */);
   }
​
   void layoutChildren(int left, int top, int right, int bottom,
                                 boolean forceLeftGravity) {
       final int count = getChildCount();
​
       final int parentLeft = getPaddingLeftWithForeground();
       final int parentRight = right - left - getPaddingRightWithForeground();
​
       final int parentTop = getPaddingTopWithForeground();
       final int parentBottom = bottom - top - getPaddingBottomWithForeground();
​
       for (int i = 0; i < count; i++) {
           final View child = getChildAt(i);
           if (child.getVisibility() != GONE) {
               final LayoutParams lp = (LayoutParams) child.getLayoutParams();
​
               final int width = child.getMeasuredWidth();
               final int height = child.getMeasuredHeight();
​
               int childLeft;
               int childTop;
​
               int gravity = lp.gravity;
               if (gravity == -1) {
                   gravity = DEFAULT_CHILD_GRAVITY;
               }
​
               final int layoutDirection = getLayoutDirection();
               final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
               final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;
​
               switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                   case Gravity.CENTER_HORIZONTAL:
                       childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                       lp.leftMargin - lp.rightMargin;
                       break;
                   case Gravity.RIGHT:
                       if (!forceLeftGravity) {
                           childLeft = parentRight - width - lp.rightMargin;
                           break;
                       }
                   case Gravity.LEFT:
                   default:
                       childLeft = parentLeft + lp.leftMargin;
               }
​
               switch (verticalGravity) {
                   case Gravity.TOP:
                       childTop = parentTop + lp.topMargin;
                       break;
                   case Gravity.CENTER_VERTICAL:
                       childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                       lp.topMargin - lp.bottomMargin;
                       break;
                   case Gravity.BOTTOM:
                       childTop = parentBottom - height - lp.bottomMargin;
                       break;
                   default:
                       childTop = parentTop + lp.topMargin;
               }
​              //调用子View的layout
               child.layout(childLeft, childTop, childLeft + width, childTop + height);
           }
       }
   }
​
```
可以看到layout也是一个递归调用的过程，这里就不详细讲述了，而由于ViewGroup没有onDraw()方法，这里就不需要讲了。

附上一张View的生命周期:
{% asset_img two.jpg cyclelife %}


-----

终于写完了，脑袋都要爆炸了，View形成的过程大概就是这样的，谢谢阅读，如果觉得哪写的不对或者不好的地方欢迎留言交流，希望能够一起进步~