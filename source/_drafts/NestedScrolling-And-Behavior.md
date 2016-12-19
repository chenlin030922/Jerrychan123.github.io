---
title: NestedScrolling And Behavior
tags: android
categories: Android使用zhengli
---

 从 Android 5.0 Lollipop 开始提供一套 API 来支持嵌入的滑动效果。同样在最新的 Support V4 包中也提供了前向的兼容。有了嵌入滑动机制，就能实现很多很复杂的滑动效果。在 Android Design Support 库中非常总要的 CoordinatorLayout 组件就是使用了这套机制，实现了 Toolbar 的收起和展开功能



 NestedScrolling提供了一套父 View 和子 View 滑动交互机制。要完成这样的交互，**父 View 需要实现 NestedScrollingParent 接口，而子 View 需要实现 NestedScrollingChild 接口**。



----



### **实现 NestedScrollingChild**

       首先来说NestedScrollingChild。**如果你有一个可以滑动的 View，需要被用来作为嵌入滑动的子 View**，就必须实现本接口。在此 View 中，包含一个 NestedScrollingChildHelper 辅助类。NestedScrollingChild接口的实现，基本上就是调用本 Helper 类的对应的函数即可，因为 Helper 类中已经实现好了 Child 和 Parent 交互的逻辑。原来的 View 的处理 Touch 事件，并实现滑动的逻辑大体上不需要改变。

       需要做的就是，如果要准备开始滑动了，需要告诉 Parent，**你要准备进入滑动状态了，调用startNestedScroll()。你在滑动之前，先问一下你的 Parent 是否需要滑动，也就是调用dispatchNestedPreScroll()**。如果父类滑动了一定距离，你需要重新计算一下父类滑动后剩下给你的滑动距 离余量。然后，你自己进行余下的滑动(比如先把导航栏推上去，那么父类就要先消耗掉滑动时导航栏的距离，然后子View再进行剩余滑动距离的移动)。最后，如果滑动距离还有剩余，你就再问一下，Parent 是否需要在继续滑动你剩下的距离，也就是调用dispatchNestedScroll()。



#### 一、startNestedScroll

首先子view需要开启整个流程（内部主要是找到合适的能接受nestedScroll的parent），通知父View，我要和你配合处理TouchEvent

#### 二、dispatchNestedPreScroll

在子View的`onInterceptTouchEvent`或者`onTouch`中(一般在MontionEvent.ACTION_MOVE事件里)，调用该方法通知父View滑动的距离。该方法的第三第四个参数返回父view消费掉的scroll长度和子View的窗体偏移量。如果这个scroll没有被消费完，则子view进行处理剩下的一些距离，**由于窗体进行了移动，如果你记录了手指最后的位置，需要根据第四个参数offsetInWindow计算偏移量，才能保证下一次的touch事件的计算是正确的。**
如果父view接受了它的滚动参数，进行了部分消费，则这个函数返回true，否则为false。
这个函数一般在子view处理scroll前调用。

#### 三、dispatchNestedScroll

向父view汇报滚动情况，包括子view消费的部分和子view没有消费的部分。
如果父view接受了它的滚动参数，进行了部分消费，则这个函数返回true，否则为false。
这个函数一般在子view处理scroll后调用。

我们最需要关注的是`dispatchNestedPreScroll`中的`consumed`参数。

```java
public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow) ;
```

它是一个`int`型的数组，长度为2，第一个元素是父view消费的`x`方向的滚动距离；第二个元素是父view消费的`y`方向的滚动距离，如果这两个值不为0，则子view需要对滚动的量进行一些修正。正因为有了这个参数，使得我们处理滚动事件的时候，思路更加清晰，不会像以前一样被一堆的滚动参数搞混。



#### 四、stopNestedScroll



-----



### **实现 NestedScrollingParent**

       作为一个可以嵌入 NestedScrollingChild 的父 View，需要实现NestedScrollingParent，这个接口方法和NestedScrollingChild大致有一一对应的关系。同样， 也有一个 NestedScrollingParentHelper 辅助类来默默的帮助你实现和 Child 交互的逻辑。滑动动作是 Child 主动发起，Parent 就收滑动回调并作出响应。

       从上面的 Child 分析可知，滑动开始的调用startNestedScroll()，Parent 收到onStartNestedScroll()回调，决定是否需要配合 Child 一起进行处理滑动，如果需要配合，还会回调onNestedScrollAccepted()。

       **每次滑动前，Child 先询问 Parent 是否需要滑动，即dispatchNestedPreScroll()**，这就回调到 Parent 的onNestedPreScroll()，Parent 可以在这个回调中“劫持”掉 Child 的滑动，也就是先于 Child 滑动。

       Child 滑动以后，会调用onNestedScroll()，回调到 Parent 的onNestedScroll()，这里就是 Child 滑动后，剩下的给 Parent 处理，也就是 后于 Child 滑动。

       最后，滑动结束，调用onStopNestedScroll()表示本次处理结束。

       其实，除了上面的 Scroll 相关的调用和回调，还有 Fling 相关的调用和回调，处理逻辑基本一致。





-----



### Behavior



官网对于 `CoordinatorLayout.Behavior` 的介绍已经将它的作用说明得很清楚了，就是用来协调 `CoordinatorLayout` 的Child Views之间的交互行为：

Interaction behavior plugin for child views of CoordinatorLayout.

A Behavior implements one or more interactions that a user can take on a child view. These interactions may include drags, swipes, flings, or any other gestures.

之前学习 `CoordinatorLayout` 的使用案例时，用的都是系统的特定控件，比如design包中的 `FloatingActionButton` 、 `AppBarLayout` 等，而不是普通的控件，如 `ImageButton` 之类的，就是因为design包中的这些特定控件已经被系统默认定义了继承自 `CoordinatorLayout.Behavior` 的各种 `Behavior` ，比如 `FloatingActionButton.Behavior` 和

`AppBarLayout.Behavior` 。而像系统的 `ToolBar` 控件就没有自己的 `Behavior`，所以只能将其搁置到 `AppBarLayout` 容器里才能产生相应的交互效果。

看到这里就能清楚一点了，如果我们想实现控件之间任意的交互效果，完全可以通过自定义 `Behavior` 的方式达到。看到这里大家可能会有一个疑惑，就是 `CoordinatorLayout` 如何获取Child Views的 `Behavior` 的呢，为什么在布局中，有些滑动型控件定义了 `app:layout_behavior` 属性而系统类似 `FloatingActionButton` 的控件则不需要明确定义该属性呢？看完 `CoordinatorLayout.Behavior` 的构造函数就明白了。

```java
/**
* Default constructor for instantiating Behaviors.
*/
public Behavior() {
}

/**
* Default constructor for inflating Behaviors from layout. The Behavior will have
* the opportunity to parse specially defined layout parameters. These parameters will
* appear on the child view tag.
*
* @param context
* @param attrs
*/
public Behavior(Context context, AttributeSet attrs) {
}
```

`CoordinatorLayout.Behavior` 有两个构造函数，注意看第二个带参数的构造函数的注释，里面提到，在这个构造函数中， `Behavior` 会解析控件的特殊布局属性，也就是通过 `parseBehavior` 方法获取对应的 `Behavior` ，从而协调Child Views之间的交互行为，可以在 `CoordinatorLayout` 类中查看，具体源码如下：

```java
static Behavior parseBehavior(Context context, AttributeSet attrs, String name) {
if (TextUtils.isEmpty(name)) {
return null;
}

final String fullName;
if (name.startsWith(".")) {
// Relative to the app package. Prepend the app package name.
fullName = context.getPackageName() + name;
} else if (name.indexOf('.') >= 0) {
// Fully qualified package name.
fullName = name;
} else {
// Assume stock behavior in this package (if we have one)
fullName = !TextUtils.isEmpty(WIDGET_PACKAGE_NAME)
? (WIDGET_PACKAGE_NAME + '.' + name)
: name;
}

try {
Map<String, Constructor<Behavior>> constructors = sConstructors.get();
if (constructors == null) {
constructors = new HashMap<>();
sConstructors.set(constructors);
}
Constructor<Behavior> c = constructors.get(fullName);
if (c == null) {
final Class<Behavior> clazz = (Class<Behavior>) Class.forName(fullName, true,
context.getClassLoader());
c = clazz.getConstructor(CONSTRUCTOR_PARAMS);
c.setAccessible(true);
constructors.put(fullName, c);
}
return c.newInstance(context, attrs);
} catch (Exception e) {
throw new RuntimeException("Could not inflate Behavior subclass " + fullName, e);
}
}
```

`parseBehavior` 方法告诉我们，给Child Views设置 `Behavior` 有两种方式：

1. `app:layout_behavior` 布局属性

在布局中设置，值为自定义 `Behavior` 类的名字字符串（包含路径），类似在 `AndroidManifest.xml` 中定义四大组件的名字一样，有两种写法，包含包名的全路径和以”.”开头的省略项目包名的路径。

2. `@CoordinatorLayout.DefaultBehavior` 类注解

在需要使用 `Behavior` 的控件源码定义中添加该注解，然后通过反射机制获取。这个方式就解决了我们前面产生的疑惑，系统的 `AppBarLayout` 、 `FloatingActionButton` 都采用了这种方式，所以无需在布局中重复设置。

看到这里，也告诉我们一点，在自定义 `Behavior` 时，一定要重写第二个带参数的构造函数，否则这个 `Behavior` 是不会起作用的。

根据 `CoordinatorLayout.Behavior` 提供的方法，这里将自定义 `Behavior` 分为两类来讲解，一种是 `dependent` 机制，一种是 `nested` 机制，对应着不同的使用场景。

## `dependent` 机制

这种机制描述的是两个Child Views之间的绑定依赖关系，设置 `Behavior` 属性的Child View跟随依赖对象Dependency View的大小位置改变而发生变化，对应需要实现的方法常见有两个：

```java
/**
* Determine whether the supplied child view has another specific sibling view as a
* layout dependency.
*
* <p>This method will be called at least once in response to a layout request. If it
* returns true for a given child and dependency view pair, the parent CoordinatorLayout
* will:</p>
* <ol>
*     <li>Always lay out this child after the dependent child is laid out, regardless
*     of child order.</li>
*     <li>Call {@link #onDependentViewChanged} when the dependency view's layout or
*     position changes.</li>
* </ol>
*
* @param parent the parent view of the given child
* @param child the child view to test
* @param dependency the proposed dependency of child
* @return true if child's layout depends on the proposed dependency's layout,
*         false otherwise
*
* @see #onDependentViewChanged(CoordinatorLayout, android.view.View, android.view.View)
*/
public boolean layoutDependsOn(CoordinatorLayout parent, V child, View dependency) {
return false;
}
```

和

```java
/**
* Respond to a change in a child's dependent view
*
* <p>This method is called whenever a dependent view changes in size or position outside
* of the standard layout flow. A Behavior may use this method to appropriately update
* the child view in response.</p>
*
* <p>A view's dependency is determined by
* {@link #layoutDependsOn(CoordinatorLayout, android.view.View, android.view.View)} or
* if {@code child} has set another view as it's anchor.</p>
*
* <p>Note that if a Behavior changes the layout of a child via this method, it should
* also be able to reconstruct the correct position in
* {@link #onLayoutChild(CoordinatorLayout, android.view.View, int) onLayoutChild}.
* <code>onDependentViewChanged</code> will not be called during normal layout since
* the layout of each child view will always happen in dependency order.</p>
*
* <p>If the Behavior changes the child view's size or position, it should return true.
* The default implementation returns false.</p>
*
* @param parent the parent view of the given child
* @param child the child view to manipulate
* @param dependency the dependent view that changed
* @return true if the Behavior changed the child view's size or position, false otherwise
*/
public boolean onDependentViewChanged(CoordinatorLayout parent, V child, View dependency) {
return false;
}
```

具体含义在注释中已经很清楚了， `layoutDependsOn()` 方法用于决定是否产生依赖行为， `onDependentViewChanged()` 方法在依赖的控件发生大小或者位置变化时产生回调。 `dependent` 机制最常见的案例就是 `FloatingActionButton` 和 `SnackBar`的交互行为，效果如下：

![img](http://img0.tuicool.com/FrEBji2.gif)

系统的 `FloatingActionButton` 已经默认定义了一个 `Behavior` 来协调交互，如果不用系统的FAB控件，比如改用GitHub上的一个库 [`futuresimple/android-floating-action-button` ](https://github.com/futuresimple/android-floating-action-button)，再通过自定义一个 `Behavior` ，也能很简单的实现与 `SnackBar`的协调效果：

```java
package com.yifeng.mdstudysamples;

import android.content.Context;
import android.support.design.widget.CoordinatorLayout;
import android.support.design.widget.Snackbar;
import android.util.AttributeSet;
import android.view.View;

/**
* Created by yifeng on 16/9/20.
*
*/
public class DependentFABBehavior extends CoordinatorLayout.Behavior {

public DependentFABBehavior(Context context, AttributeSet attrs) {
super(context, attrs);
}

/**
* 判断依赖对象
* @param parent
* @param child
* @param dependency
* @return
*/
@Override
public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
return dependency instanceof Snackbar.SnackbarLayout;
}

/**
* 当依赖对象发生变化时,产生回调,自定义改变child view
* @param parent
* @param child
* @param dependency
* @return
*/
@Override
public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) {
float translationY = Math.min(0, dependency.getTranslationY() - dependency.getHeight());
child.setTranslationY(translationY);
return true;
}
}
```

很简单的一个自定义 `Behavior` 处理，然后再为对应的Child View设置该属性即可。由于这里我们用的是第三方库，采用远程依赖的形式引入的，无法修改源码，所以不方便使用注解的方式为其设置 `Behavior` ，所以在布局中为其设置，并且使用了省略包名的方式：

```java
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
xmlns:app="http://schemas.android.com/apk/res-auto"
android:layout_width="match_parent"
android:layout_height="match_parent">

<android.support.design.widget.AppBarLayout
android:id="@+id/appbar"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">

<include
layout="@layout/include_toolbar"/>

</android.support.design.widget.AppBarLayout>

<com.getbase.floatingactionbutton.FloatingActionButton
xmlns:fab="http://schemas.android.com/apk/res-auto"
android:id="@+id/fab_add"
android:layout_width="wrap_content"
android:layout_height="wrap_content"
android:layout_margin="@dimen/dp_16"
android:layout_gravity="bottom|right"
android:onClick="onClickFab"
fab:fab_icon="@mipmap/ic_toolbar_add"
fab:fab_colorNormal="?attr/colorPrimary"
fab:fab_colorPressed="?attr/colorPrimaryDark"
app:layout_behavior=".DependentFABBehavior"/>

</android.support.design.widget.CoordinatorLayout>
```

这样，采用 `dependent` 机制自定义 `Behavior` ，与使用系统FAB按钮一样，即可与 `SnackBar` 控件产生如上图所示的协调交互效果。

比如我们再看一下这样一个效果：

![img](http://img1.tuicool.com/YJr6NjV.gif)

列表上下滑动式，底部评论区域随着顶部 `Toolbar` 的移动而移动，这里我们就可以自定义一个 `Dependent` 机制的 `Behavior` ，设置给底部视图，让其依赖于包裹 `Toolbar` 的 `AppBarLayout` 控件：

```java
package com.yifeng.mdstudysamples;

import android.content.Context;
import android.support.design.widget.AppBarLayout;
import android.support.design.widget.CoordinatorLayout;
import android.util.AttributeSet;
import android.view.View;

/**
* Created by yifeng on 16/9/23.
*
*/

public class CustomExpandBehavior extends CoordinatorLayout.Behavior {

public CustomExpandBehavior(Context context, AttributeSet attrs) {
super(context, attrs);
}

@Override
public boolean layoutDependsOn(CoordinatorLayout parent, View child, View dependency) {
return dependency instanceof AppBarLayout;
}

@Override
public boolean onDependentViewChanged(CoordinatorLayout parent, View child, View dependency) {
int delta = dependency.getTop();
child.setTranslationY(-delta);
return true;
}
}
```

布局内容如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
xmlns:app="http://schemas.android.com/apk/res-auto"
android:layout_width="match_parent"
android:layout_height="match_parent">

<android.support.design.widget.AppBarLayout
android:id="@+id/appbar"
android:layout_width="match_parent"
android:layout_height="@dimen/dp_56"
android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">

<include
layout="@layout/include_toolbar"/>

</android.support.design.widget.AppBarLayout>

<android.support.v7.widget.RecyclerView
android:id="@+id/rv_content"
android:layout_width="match_parent"
android:layout_height="match_parent"
app:layout_behavior="@string/appbar_scrolling_view_behavior"/>

<RelativeLayout
android:layout_width="match_parent"
android:layout_height="@dimen/dp_56"
android:layout_gravity="bottom"
app:layout_behavior=".CustomExpandBehavior"
android:padding="8dp"
android:background="@color/blue">

<Button
android:id="@+id/btn_send"
android:layout_width="wrap_content"
android:layout_height="match_parent"
android:text="Send"
android:layout_alignParentRight="true"
android:background="@color/white"/>

<EditText
android:layout_width="match_parent"
android:layout_height="match_parent"
android:layout_toLeftOf="@id/btn_send"
android:layout_marginRight="4dp"
android:padding="4dp"
android:hint="Please input the comment"
android:background="@color/white"/>

</RelativeLayout>

</android.support.design.widget.CoordinatorLayout>
```

注意，这里将自定义的 `Behavior` 设置给了底部内容的外层容器 `RelativeLayout`，即可实现上述效果。

## `Nested` 机制

`Nested` 机制要求 `CoordinatorLayout` 包含了一个实现了 `NestedScrollingChild` 接口的滚动视图控件，比如v7包中的 `RecyclerView` ，设置 `Behavior` 属性的Child View会随着这个控件的滚动而发生变化，涉及到的方法有：

```java
onStartNestedScroll(View child, View target, int nestedScrollAxes)

onNestedPreScroll(View target, int dx, int dy, int[] consumed)

onNestedPreFling(View target, float velocityX, float velocityY)

onNestedScroll(View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed)

onNestedFling(View target, float velocityX, float velocityY, boolean consumed)

onStopNestedScroll(View target)
```

其中， `onStartNestedScroll` 方法返回一个boolean类型的值，只有返回true时才能让自定义的 `Behavior` 接受滑动事件。同样的，举例说明一下。

通过查看系统FAB控件的源码可以知道，系统FAB定义的 `Behavior` 能够处理两个交互，一个是与 `SnackBar` 的位置交互，效果如上面的图示一样，另一个就是与 `AppBarLayout` 的展示交互，都是使用的 `Dependent` 机制，效果在之前的文章 – [Android CoordinatorLayout 实战案例学习《二》](http://www.jianshu.com/p/360fd368936d) 中可以查看，也就是 `AppBarLayout` 滚动到一定程度时，FAB控件的动画隐藏与展示。下面我们使用 `Nested` 机制自定义一个 `Behavior` ，实现如下与列表协调交互的效果：

![img](http://img1.tuicool.com/MfEVz2I.gif)

为了能够使用系统FAB控件提供的隐藏与显示的动画效果，这里直接继承了系统FAB控件的 `Behavior` ：

```java
package com.yifeng.mdstudysamples;

import android.content.Context;
import android.support.design.widget.CoordinatorLayout;
import android.support.design.widget.FloatingActionButton;
import android.support.v4.view.ViewCompat;
import android.util.AttributeSet;
import android.view.View;

/**
* Created by yifeng on 16/8/23.
*
*/
public class NestedFABBehavior extends FloatingActionButton.Behavior {

public NestedFABBehavior(Context context, AttributeSet attrs) {
super();
}

@Override
public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, FloatingActionButton child, View directTargetChild, View target, int nestedScrollAxes) {
return nestedScrollAxes == ViewCompat.SCROLL_AXIS_VERTICAL ||
super.onStartNestedScroll(coordinatorLayout, child, directTargetChild, target,
nestedScrollAxes);
}

@Override
public void onNestedScroll(CoordinatorLayout coordinatorLayout, FloatingActionButton child, View target, int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed) {
super.onNestedScroll(coordinatorLayout, child, target, dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed);
if (dyConsumed > 0 && child.getVisibility() == View.VISIBLE) {
//系统FAB控件提供的隐藏动画
child.hide();
} else if (dyConsumed < 0 && child.getVisibility() != View.VISIBLE) {
//系统FAB控件提供的显示动画
child.show();
}
}
}
```

然后在布局中添加 `RecyclerView` ，并为系统FAB控件设置自定义的 `Behavior`，内容如下：

```java
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
xmlns:app="http://schemas.android.com/apk/res-auto"
android:layout_width="match_parent"
android:layout_height="match_parent">

<android.support.design.widget.AppBarLayout
android:id="@+id/appbar"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">

<include
layout="@layout/include_toolbar"/>

</android.support.design.widget.AppBarLayout>

<android.support.v7.widget.RecyclerView
android:id="@+id/rv_content"
android:layout_width="match_parent"
android:layout_height="match_parent"/>

<android.support.design.widget.FloatingActionButton
android:id="@+id/fab_add"
android:layout_width="wrap_content"
android:layout_height="wrap_content"
android:layout_margin="@dimen/dp_16"
android:src="@mipmap/ic_toolbar_add"
app:layout_anchor="@id/rv_content"
app:layout_anchorGravity="bottom|right"
app:backgroundTint="@color/fab_ripple"
app:layout_behavior="com.yifeng.mdstudysamples.NestedFABBehavior"/>

</android.support.design.widget.CoordinatorLayout>
```

这样，即可实现系统FAB控件与列表滑动控件的交互效果。

## @string/appbar_scrolling_view_behavior

这是一个系统字符串，值为：

```java
android.support.design.widget.AppBarLayout$ScrollingViewBehavior
```

在 `CoordinatorLayout` 容器中，通常用在 `AppBarLayout` 视图下面（不是里面）的内容控件中，比如上面的 `RecyclerView` ，如果我们不给它添加这个 `Behavior` ， `Toolbar` 将覆盖在列表上面，出现重叠部分，如图

![img](http://img1.tuicool.com/2uUvQbB.png!web)

添加之后， `RecyclerView` 将位于 `Toolbar` 下面，类似在 `RelativeLayout` 中设置了 `below` 属性，如图：

![img](http://img2.tuicool.com/uaamMvr.png!web)







参考:

- https://lab.getbase.com/nested-scrolling-with-coordinatorlayout-on-android/
- Demo:https://github.com/ggajews/nestedscrollingchildviewdemo
-  [http://yifeng.studio/2016/09/23/behavior-analyzation/](http://yifeng.studio/2016/09/23/behavior-analyzation/?utm_source=tuicool&utm_medium=referral)