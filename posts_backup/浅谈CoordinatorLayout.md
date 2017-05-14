---
title: 浅谈CoordinatorLayout
date: 2016-08-01 22:24:32
tags: Android
---

#### 写在前面  
这是一篇重新写的东西，之前写过一点，但是烂尾了，我也很久没有去写点东西了，也是很惭愧，之前打算的事情也没有坚持下来。最近这一周空余的时间比较多，然后去重构了一个公司项目里的一个个人中心的页面，原来使用了`ListView`再加上`addHead`的方式，然后动态的去控制`Head`的高度去实现嵌套滑动的效果，因为我的模拟器没有跑起来，所以也没有去录下个GIF来，因为这篇主要说一下`CoordinatorLayout`的处理嵌套滑动的原理，没有效果图也影响不大。<!--more-->
#### 开始吧
感觉好久没有写点东西，写起来很是生疏，不知道该怎么去写一篇博客了，那就先写出来点大纲吧，说到哪是哪。这篇主要分这几部分吧

 - 为什么要有`CoordinatorLayout` 
 - `CoordinatorLayout`是怎么实现嵌套滑动的事件分发的
 - `AppBarLayout`的一些东西

#### 为什么要有`CoordinatorLayout` 
先来回想一下`Android`系统的事件分发有什么不足的地方，当然我觉得`Android`的事件分发写的也是十分的巧妙。这里就不去再多说事件分发的东西，说一下这样的一个场景，就是下面一个类似`list`的可滚动区域，上面是一个头，这种场景在很多的个人中心的地方比较常见。如果要去实现这样的效果，传统的做法就是外面包一层parent，当然直接去处理list的滑动事件应该也是可以的，我觉得包一层比较好写点，让这个parent去分发事件和处理事件。好吧，我有点不知道说什么了，还是直接说事件分发的不足的地方吧，就是一个view消费了事件，与此同时，其他的view的没有机会去接触到这个事件了，这样当遇到上面说的情况，就是嵌套滑动的时候，我们需要上下两个view都能同时接触到这个事件，处理不处理就让他们自己去决定了。这样我觉得实现嵌套滑动的情况就好多了，因为都有机会去消费掉这个事件，无论事件是在哪个view上面，其实就是类似再外面包了一层parent的效果，这就是`CoordinatorLayout` 帮我们做的事件，靠，我写的好烂啊。。真的说不清楚这些。

#### `CoordinatorLayout`是怎么实现嵌套滑动的事件分发的
先说一下两个接口吧，`NestedScrollingParent`和`NestedScrollingChild`，这两个接口看名字就比较容易理解他们的作用，就是一个给嵌套滑动的parent实现的，一个是给child实现，在选择的系统中，只有`CoordinatorLayout`实现了parent，所以下面说到parent就理解成`CoordinatorLayout`就OK啦，而child这个接口其实方法已经在`View`这个类里面帮我们实现好了，也就是说想要去成为一个可以嵌套滑动的child，仅仅去实现这个child接口就OK了，其他的`View`已经帮你写完了。下面还是以`RecycleView`为例子，它是实现了这个child的接口，还是回到情景里面说吧，说一下到底是怎么传递事件的。

 - 在`ReclceView`上down -> `parent.onStartNestedScroll` ->遍历有`Behavior`的直接子view ->child的`Behavior.onStartNestedScroll`。
这里说一下`onStartNestedScroll`这个方法是有boolean的返回值，`true`的意思是要去处理这个事件，同时还会去调用`parent.onNestedScrollAccepted` -> 遍历有`Behavior`的直接子view  ->child的`Behavior.onNestedScrollAccepted`.这个方法里面可以做一些view配置的初始化。如果不需要当然可以不去重写。
 - 在`RecycleView`上move -> `dispatchNestedPreScroll`(也是child这个接口里面的)->`parent.onNestedPreScroll` -> 遍历有`Behavior`的直接子view  ->child的`Behavior.onNestedPreScroll`,同时之后还有一个流程也是在move的时候触发的 ->`dispatchNestedScroll`(也是child这个接口里面的) ->`parent.onNestedScroll` -> 遍历有`Behavior`的直接子view  ->child的`Behavior.onNestedScroll`.。这两个的主要区别的pre那个方法里面可以去处理消费了多少的滑动距离，比如手指滑动了12px,你可以选择把consumed[1]赋值为2，那么head就移动(消费)了2px，剩下的10px就给了下面的recycleview，而在`onNestedScroll`方法中还有机会去已经消费的和没有消费的距离再进行一次处理。
 - 在`RecycleView`上面up -> `stopNestedScroll`(也是child这个接口里面的) ->`parent.onStopNestedScroll` -> 遍历有`Behavior`的直接子view  ->child的`Behavior.onStopNestedScroll`。
 - 还有两个关于`Fling`的方法，就不说了，一个套路。

还有就是再说一下`CoordinatorLayout`的`onInterceptTouchEvent`和`onTouchEvent`，这里贴一下代码吧

```
     for (int i = 0; i < childCount; i++) {
            final View child = topmostChildList.get(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final Behavior b = lp.getBehavior();

            if ((intercepted || newBlock) && action != MotionEvent.ACTION_DOWN) {
                // Cancel all behaviors beneath the one that intercepted.
                // If the event is "down" then we don't have anything to cancel yet.
                if (b != null) {
                    if (cancelEvent == null) {
                        final long now = SystemClock.uptimeMillis();
                        cancelEvent = MotionEvent.obtain(now, now,
                                MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
                    }
                    switch (type) {
                        case TYPE_ON_INTERCEPT:
                            b.onInterceptTouchEvent(this, child, cancelEvent);
                            break;
                        case TYPE_ON_TOUCH:
                            b.onTouchEvent(this, child, cancelEvent);
                            break;
                    }
                }
                continue;
            }

            if (!intercepted && b != null) {
                switch (type) {
                    case TYPE_ON_INTERCEPT:
                        intercepted = b.onInterceptTouchEvent(this, child, ev);
                        break;
                    case TYPE_ON_TOUCH:
                        intercepted = b.onTouchEvent(this, child, ev);
                        break;
                }
                if (intercepted) {
                    mBehaviorTouchView = child;
                }
            }
```
可以看出，每个直接的childview都会接受到拦截事件，即使你的手势不是在这个child上面触发的。

 - 还有个方法就是`layoutDependsOn`这个方法是在哪里调用的，这个是在onpredraw的时候就调用了，也就是没显示之前就已经开始准备各个view直接的依赖关系。当然内部实现也是在parent里面去便利child来实现的，也就只有parent可以获取到所有的child。

#### `AppBarLayout`的一些东西
我觉得`AppBarLayout`这个组件还是十分重要的，特别是里面的两个`Behavior`的实现，给了我们很好的参考的例子。具体的还是去看源码吧，太多了要说也是再开一篇。
 
