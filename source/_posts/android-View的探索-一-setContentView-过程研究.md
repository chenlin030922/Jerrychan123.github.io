---
title: 'android View的探索(一) : setContentView()过程研究'
date: 2016-12-09 15:56:36
tags: android
categories: Android源码研究
---
这里开启一轮关于View的研究，主要选择View开始研究的原因是我愿意(ง •̀_•́)ง,！在深入了解View的过程中，主要有两个目的，其一是整理一下以前对于View的认识且纠正一下错思维误区，其二是更深入了解其运行原理以备日后能够侃侃而谈骗基友。
<!--more-->
------
###下面进入正题：setContentView()
首先从最宏观的角度来说，我们在学习android的过程中第一次遇到关于View方法就是setContentView()，随着学习的加深，我们发现当代码中要实例化一个View的时候，很多时候是通过LayoutInflater来进行的，那为什么setContentView()不需要通过这种方式？

为啥不需要，就是我们这篇文章要研究的内容，so read the fucking code。

-----

```java
   ---- Acvtivity.java ----

   public void setContentView(@LayoutRes int layoutResID) {
       getWindow().setContentView(layoutResID);
       initWindowDecorActionBar();
   }
​
final void attach(Context context, ActivityThread aThread,
           Instrumentation instr, IBinder token, int ident,
           Application application, Intent intent, ActivityInfo info,
           CharSequence title, Activity parent, String id,
           NonConfigurationInstances lastNonConfigurationInstances,
           Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
       attachBaseContext(context);
       mFragments.attachHost(null /*parent*/);
       mWindow = new PhoneWindow(this);
       mWindow.setCallback(this);
       mWindow.setOnWindowDismissedCallback(this);
       mWindow.getLayoutInflater().setPrivateFactory(this);
       if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
           mWindow.setSoftInputMode(info.softInputMode);
       }
       if (info.uiOptions != 0) {
           mWindow.setUiOptions(info.uiOptions);
       }
     ....
   }
```
上述代码来源于最基础的Activity类，我们从最基本的看起即可，在代码可以看到主要做了两个操作：
> 获取window进行设置contentView

> 初始化ActionBar

从上方我们可以推测，初始化contentView的过程，其实是在window中进行的，而这个window的具体实现则是PhoneWindow,在Activity的attach方法中我们可以知道，因此需要进入的PhoneWindow中过去查看setContentView()：

```java
   ---- PhoneWindow.java ----

​
// This is the view in which the window contents are placed. It is either
   // mDecor itself, or a child of mDecor where the contents go.
   private ViewGroup mContentParent;
 // This is the top-level view of the window, containing the window decor.
   private DecorView mDecor;
@Override
   public void setContentView(int layoutResID) {
       if (mContentParent == null) {
         //初始化DecorView
           installDecor();
       } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
           mContentParent.removeAllViews();
       }
​
       if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
           final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                   getContext());
           transitionTo(newScene);
       } else {
         //调用inflate初始化
           mLayoutInflater.inflate(layoutResID, mContentParent);

       }
       mContentParent.requestApplyInsets();
       final Callback cb = getCallback();
       if (cb != null && !isDestroyed()) {
           cb.onContentChanged();
       }
   }
```
在上述代码中FEATURE_CONTENT_TRANSITIONS是5.0增加的一个标识来识别是否进行实现Activity或者Fragment切换时的异常复杂的动画效果，我们可以省略来进行代码分析:
> installDecor()，初始化DecorView以及mContentParent操作

> 调用LayoutInflater.inflate()方法初始化布局

```java
   ---- PhoneWindow.java ----

​
     private void installDecor() {
       if (mDecor == null) {
         //此方法new了一个DecorView
           mDecor = generateDecor();
           mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
           mDecor.setIsRootNamespace(true);
           if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
               mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
           }
       }
       if (mContentParent == null) {
         //进行mContentParent初始化的过程
           mContentParent = generateLayout(mDecor);
​
           // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
           mDecor.makeOptionalFitsSystemWindows();
           ...
   }
```
在DecorView和mContentParent初始化完成后，系统再调用inflate()方法来进行布局的初始化的过程，我们需要知道具体是哪个LayoutInflater来完成工作的:
```java
 // PhoneWindow.java ----

public PhoneWindow(Context context) {
       super(context);
       mLayoutInflater = LayoutInflater.from(context);
   }
 //LayoutInflater.java ----

 public static LayoutInflater from(Context context) {
       LayoutInflater LayoutInflater =
               (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);

       if (LayoutInflater == null) {
           throw new AssertionError("LayoutInflater not found.");
       }
       return LayoutInflater;
   }
//ContextImpl.java

@Override
   public Object getSystemService(String name) {
       return SystemServiceRegistry.getSystemService(this, name);

   }
//SystemServiceRegistry

registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
               new CachedServiceFetcher<LayoutInflater>() {
           @Override
           public LayoutInflater createService(ContextImpl ctx) {
               return new PhoneLayoutInflater(ctx.getOuterContext());

           }});
```
我们看到inflater的初始化来自于from方法，这在我们撸代码的时候其实也经常会那么写，但是点到进去会发现是个抽象类，而inflater由context.getSystemService()获取，由于context的具体实现类是ContextImpl，我们再一次追溯，知道在SystemServiceRegistry中找到了具体实现类：PhoneLayoutInflater。
接着我们回溯到PhoneWindow的mLayoutInflater.inflate(layoutResID, mContentParent)，其实到这步就已经跟我们在布局中使用LayoutInflater的效果一样了。
but，我们要有好奇心，点进去看看我们跳回inflate()这个方法,跳回PhoneLayoutInflater中进行查看，发现里面并没有inflate方法，那只能重新回到LayoutInflater.java中了，发现找了半天并没有什么卵用，不过了解到LayoutInlfater的具体实现是PhoneLayoutInflater对我们也是很有帮助的=￣ω￣=。
````java
//LayoutInflater.java

public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
   //获取资源文件

       final Resources res = getContext().getResources();

       if (DEBUG) {
           Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                   + Integer.toHexString(resource) + ")");
       }
  //获取xml解析器XmlResourceParser

       final XmlResourceParser parser = res.getLayout(resource);

       try {
          //返回解析后得到的View

           return inflate(parser, root, attachToRoot);

       } finally {
           parser.close();
       }
   }
```
最终还是比较简单的，我们在inflate中看到，其实获得XmlResourceParser解析器，然后将布局文件添加到了根布局当中。

等等最后还有一个方法：
```java
//PhoneWindow.java
final Callback cb = getCallback();
if (cb != null && !isDestroyed()) {  
  cb.onContentChanged();
}
```
这个callback是干哈用的，点进去：
```java
 public void setCallback(Callback callback) {

       mCallback = callback;
   }
​
   /**
     * Return the current Callback interface for this window.
     */
   public final Callback getCallback() {
       return mCallback;
   }
```
我们只能找到setCallBack()这个方法的具体位置，我们在上面的activity找到了window=new PhoneWindow()这个实现:
```java
//activity
final void attach(Context context, ActivityThread aThread,
           Instrumentation instr, IBinder token, int ident,
           Application application, Intent intent, ActivityInfo info,
           CharSequence title, Activity parent, String id,
           NonConfigurationInstances lastNonConfigurationInstances,
           Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
       attachBaseContext(context);
       mFragments.attachHost(null /*parent*/);
       mWindow = new PhoneWindow(this);
       mWindow.setCallback(this);
       mWindow.setOnWindowDismissedCallback(this);
       mWindow.getLayoutInflater().setPrivateFactory(this);
       if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
           mWindow.setSoftInputMode(info.softInputMode);
       }
       if (info.uiOptions != 0) {
           mWindow.setUiOptions(info.uiOptions);
       }
     ....
   }
```
 mWindow.setCallback(this)这句话是不是够了！嗯哼！那么 cb.onContentChanged()这个方法其实就是调用了在activity中的onContentChanged方法，这是通知ContentView发生改变的方法，我们可以重写进行一些你喜欢的操作。

----

####总结
画张图吧，这样比较好理解：

{% asset_img setcontentView.png setContentView()过程图 %}


