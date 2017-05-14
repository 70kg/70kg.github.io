---
title: View的绘制过程
date: 2016-07-09 23:33:41
tags: Android
---
&emsp;&emsp;这次来说一说view的绘制过程，同样也算是个笔记的东西，梳理一下大致的过程，忽略了内部很多的条件判断等。上次说了`setcontentview`和`infalter`的一些简单过程，这个是用来把view从XML文件的格式解析和加载，但是具体view的怎么绘制到界面上的并没有提及，还有上次留下的问题，有时间学习了`activity`的启动流程再来说一下，现在开始吧。<!--more-->

&emsp;&emsp;网上有很多的文章说view的绘制，无非就是三个方法`onMeasure`,`onLayout`,`onDraw`，但是这仅仅是冰山一角，这三个方法的调用时机和内部的大致运行逻辑并没有说清楚，现在先从view开始的地方说起。

&emsp;&emsp;view从什么地方开始绘制呢，先说结论吧，因为没有一些铺垫即使从开始的地方说起也是云里雾里，说完整个的大致过程，再去看开始的地方也就顺其自然了。view的绘制是从`ViewRootImpl` 的`performTraversals`方法开始的，翻译就是递归调用，我们已经知道，view是嵌套存在的，所以要去绘制整个的view肯定要去嵌套调用一些方法，(ps:这个源码是在API22下看的，在23的时候又有一些改变，但是整个的逻辑是没有变化的)。我们找到这个方法，这是个非常长的方法，一共有782行，所以会去忽略很多的判断条件，只所我们关注的三个重点方法，首先找到这里：

###Measure
```
  int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
  int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
  // Ask host how big it wants to be
  performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
```
这里就是进行view测绘的地方，先去`getRootMeasureSpec`方法里面看看，

```
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
这是个组装参数的地方，像我们平常说的`MATCH_PARENT`对应`EXACTLY`，`WRAP_CONTENT`对应`AT_MOST`这里就看的很清楚了，而且这个根视图，之前提到过的，肯定是走`MATCH_PARENT`，这就是我们根视图总是全屏的原因。

&emsp;&emsp;回到前面还有个`performMeasure`方法，看注释也知道是干啥的，进去看看：

```
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```
看，来到了view的`measure`方法，也就是说`measure`方法的两个参数其实都是父view的，下面还可以看到，真正决定一个view的大小是这父view的两个参数和子view共同决定的。下面就进到了view里面。

```
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
     *        parent
     * @param heightMeasureSpec Vertical space requirements as imposed by the
     *        parent
     *
     * @see #onMeasure(int, int)
     */
    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ...各种不明白的东西。。😂
    
       onMeasure(widthMeasureSpec, heightMeasureSpec);
       
     ...继续各种不明白的东西。。😂
    }
```

这里主要还是看注释，四级水平理解一下，这个方法调用用来测量view应该有多大，父布局在宽高里面提供限制信息，也就是前面说的view的大小由爹和儿子一起决定，还有就是实际的测量工作是在子类的`onMeasure`方法里面搞，因为`measure`是final的。这就是平时自定义view的时候要去从写这个方法的原因，别急，还没完，进去看看。

```
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
     * @param widthMeasureSpec horizontal space requirements as imposed by the parent.
     *                         The requirements are encoded with
     *                         {@link android.view.View.MeasureSpec}.
     * @param heightMeasureSpec vertical space requirements as imposed by the parent.
     *                         The requirements are encoded with
     *                         {@link android.view.View.MeasureSpec}.
     *
     * @see #getMeasuredWidth()
     * @see #getMeasuredHeight()
     * @see #setMeasuredDimension(int, int)
     * @see #getSuggestedMinimumHeight()
     * @see #getSuggestedMinimumWidth()
     * @see android.view.View.MeasureSpec#getMode(int)
     * @see android.view.View.MeasureSpec#getSize(int)
     */
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
```
可以去读一读注释，简单说一下，这个方法是用来测量view和它的内容，而且一定要调用`setMeasuredDimension`把测量的结果穿进去，基类默认实现的背景的大小(这个在下面看到)。如果重写了这个方法，你得搞清楚这个view的大小至少比最小的大（后面会说）。好吧，蹩脚的英文。也就是说当我们测量完view之后，再`setMeasuredDimension`，我们的工作就结束了。那么这个方法是什么东西，去看看：

```
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
可以看到，默认的时候，当设置`wrap_content`和`march_parent`都是默认的父布局的大小，这里的size是进一步`getSuggestedMinimumWidth`传进来的,而这个的大小是

```
 protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
```
看有没有背景，这个默认是0，也可以在XML中设置,这样也说完了前面的两个问题。

&emsp;&emsp;到这里，似乎view的测量就完了，但是不要忘记，view是嵌套的，要去测量完整个的view，还得去`viewgroup`里面看看，`viewgroup`里面有三个方法：`measureChildren`, `measureChild`, `measureChildWithMargins`，一个个看。

```
/**
     * Ask all of the children of this view to measure themselves, taking into
     * account both the MeasureSpec requirements for this view and its padding.
     * We skip children that are in the GONE state The heavy lifting is done in
     * getChildMeasureSpec.
     *
     * @param widthMeasureSpec The width requirements for this view
     * @param heightMeasureSpec The height requirements for this view
     */
    protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }
```
遍历所以的子view，去测量，如果是gone的，就跳过。然后进入`measureChild`，提醒一下，这里传进去的`widthMeasureSpec`和`heightMeasureSpec`都是父view的，用来在子view测量的时候共同决定大小用的。这里的`measureChild`只包含`Padding`，`measureChildWithMargins`还包含`Margin`，调复杂的看：

```
protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
            //获取子的LayoutParams
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
//看，子view的大小是由谁决定的，没错吧，这里竟然还直接获取到了lp.width，这个看下面解释吧。提示一下：这个参数是：How big the child wants to be in the current dimension
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);
       //调用view的measure进而调用onmeasure
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
看注释，然后看重要的`getChildMeasureSpec`：

```
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;
        //父view的模式
        switch (specMode) {
        // Parent has imposed an exact size on us
        //父view是具体数值或者march_parent
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {//子view为具体数值
               //直接赋值具体数值
                resultSize = childDimension;
                //返回子view的模式设置为EXACTLY
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {//子view是MATCH_PARENT，数值为-1
                // Child wants to be our size. So be it.
                //直接赋值父view的大小
                resultSize = size;
                 //返回子view的模式设置为EXACTLY
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {//子view是WRAP_CONTENT，数值为-2
                // Child wants to determine its own size. It can't be
                // bigger than us.
                //设置子view的最大值不能超过父view
                //这个一般还会在子view的onmeasure里面去重写，是吧
                resultSize = size;
                //返回子view的模式设置为AT_MOST
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

     //。。。。
     //过程差不多的两个模式AT_MOST和UNSPECIFIED
       
        }
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```
注释已经很详细了，这就是我们见到的父view和子view的在某些关系下面的布局样式。这次明白了吧。整个逻辑大概说一下，rootview获得宽高和模式，通过`MeasureSpec`传给下一级，一般是`viewgroup`，然后`viewgroup`主要调用`getChildMeasureSpec`根据上面的`MeasureSpec`子view的`childDimension`来确定自己`viewgroup`的大小，如果还有子view，把计算出来的自己的`MeasureSpec`再传下去.在viewgroup中我们并没有看到调用`setMeasuredDimension`方法去把测量的值传递进去，这个是因为viewgroup是个抽象类，具体的测量方式需要我们自己去实现，比如可以去看看`LinearLayout`的实现，就在测量完成后调用了`setMeasuredDimension`方法。所以在一般自定义viewgroup的时候，可以先去调用`measureChildren`去测量完全部的子view，这样就可以去获取子view的宽高，然后再干什么就随你了，最后不要忘记去调用`setMeasuredDimension`方法。

###layout
&emsp;&emsp;现在整个view的测绘就差不多结束了，下一步就是开始layout的步骤。先去看viewgroup里面的：

```
 @Override
    public final void layout(int l, int t, int r, int b) {
        if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
            if (mTransition != null) {
                mTransition.layoutChange(this);
            }
            super.layout(l, t, r, b);
        } else {
            // record the fact that we noop'd it; request layout when transition finishes
            mLayoutCalledWhileSuppressed = true;
        }
    }
```
其实还是调用了view里面的`layout`方法，进去看看：

```
public void layout(int l, int t, int r, int b) {
        ......
        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            //回调onLayout
            onLayout(changed, l, t, r, b);
            ......
        }
        ......
    }
```
和`measure`的过程差不多，还是去调用`onLayout`方法，看一下

```
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    }
```
空的，那viewgroup的呢：

```
@Override
    protected abstract void onLayout(boolean changed,
            int l, int t, int r, int b);
```
也是空的，还变成抽象的了，那总结一下：view的layuout方法并没有像`measure`方法一下是final的，但是也是不推荐去重写的，如果有需要还是去重写`onLayout`方法，而在viewgroup中，`layout`变成了final,`onLayout`方法变成抽象的了，这就要求子类必须去实现这个过程，也就是要去子类自己去决定里面的view怎么去排布，所以去找个子类看看，还是拿`LinearLayout`看：

```
@Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        if (mOrientation == VERTICAL) {
            layoutVertical(l, t, r, b);
        } else {
            layoutHorizontal(l, t, r, b);
        }
    }
```
看名字就知道了，不解释，选个`layoutVertical`看看：

```
void layoutVertical(int left, int top, int right, int bottom) {
        final int paddingLeft = mPaddingLeft;

        int childTop;
        int childLeft;
        //理解为父view提供给子view的宽度
        // Where right end of child should go
        final int width = right - left;
        //子view在父view中的右边位置
        int childRight = width - mPaddingRight;
        
        //子view的可用大小
        // Space available for child
        int childSpace = width - paddingLeft - mPaddingRight;
        //子view的数量
        final int count = getVirtualChildCount();
        //主要的Gravity方式
        final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
        final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;

        switch (majorGravity) {
           case Gravity.BOTTOM://居底
               // mTotalLength contains the padding already
               childTop = mPaddingTop + bottom - top - mTotalLength;
               break;

               // mTotalLength contains the padding already
           case Gravity.CENTER_VERTICAL:
               childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
               break;

           case Gravity.TOP:
           default:
               childTop = mPaddingTop;
               break;
        }
//开始遍历了
        for (int i = 0; i < count; i++) {
            final View child = getVirtualChildAt(i);
            if (child == null) {
                childTop += measureNullChild(i);
                //忽略gone的
            } else if (child.getVisibility() != GONE) {
                final int childWidth = child.getMeasuredWidth();
                final int childHeight = child.getMeasuredHeight();
                
                final LinearLayout.LayoutParams lp =
                        (LinearLayout.LayoutParams) child.getLayoutParams();
                
                int gravity = lp.gravity;
                if (gravity < 0) {
                    gravity = minorGravity;
                }
                final int layoutDirection = getLayoutDirection();
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.CENTER_HORIZONTAL:
                        childLeft = paddingLeft + ((childSpace - childWidth) / 2)
                                + lp.leftMargin - lp.rightMargin;
                        break;

                    case Gravity.RIGHT:
                        childLeft = childRight - childWidth - lp.rightMargin;
                        break;

                    case Gravity.LEFT:
                    default:
                        childLeft = paddingLeft + lp.leftMargin;
                        break;
                }

                if (hasDividerBeforeChildAt(i)) {
                    childTop += mDividerHeight;
                }

                childTop += lp.topMargin;
                //其实还是调用view.layout
                setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                        childWidth, childHeight);
                childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

                i += getChildrenSkipCount(child, i);
            }
        }
    }
```
整个的layout过程比measure好理解，还要提一下，平时使用的时候会遇到`getMeasuredWidth`和`getWidth`，这里就差不多可以理解了，第一个是在measure之后可以获取到，而第二个需要在layout之后才可以获取到，一般都是一样的。总结一下：
measure操作完成后得到的是对每个View经测量过的measuredWidth和`measuredHeight`，layout操作完成之后得到的是对每个View进行位置分配后的mLeft、mTop、mRight、mBottom，这些值都是相对于父View来说的。凡是`layout_XXX`的布局属性基本都针对的是包含子View的ViewGroup的，当对一个没有父容器的View设置相关`layout_XXX`属性是没有任何意义的，看前面的那篇。到这里，layout的过程也差不多了，该到了绘制的时候了。

###draw
view的绘制过程是比较复杂的，贴一张图，来自[工匠若水](http://blog.csdn.net/yanbober/article/details/46128379)
![](http://img.blog.csdn.net/20150530154328068)
因为viewgroup没有去重写draw方法，直接去看view里面的：

```
 public void draw(Canvas canvas) {
        ...

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
       ...
```
注释写的很清楚了，一共分为6步，其中2，5可以省略，一步步来吧
####第一步，对View的背景进行绘制。

```
private void drawBackground(Canvas canvas) {
//平时设置的setbackground,实质都转变为一个Drawable
        final Drawable background = mBackground;
        ...
        //设置绘制范围
        setBackgroundBounds();
        ...
        final int scrollX = mScrollX;
        final int scrollY = mScrollY;
        if ((scrollX | scrollY) == 0) {//没有滚动
            background.draw(canvas);
        } else {
            canvas.translate(scrollX, scrollY);
            //最终使用了Drawable的draw
            background.draw(canvas);
            canvas.translate(-scrollX, -scrollY);
        }
    }
```
####第二步，保存图层，执行渐变操作
主要调用` canvas.saveLayer`方法，不多说
####第三步，绘制内容

```
 // Step 3, draw the content
        if (!dirtyOpaque) 
        onDraw(canvas);
```
而`onDraw`又是个空方法，因为具体绘制什么东西还得具体的view说了算

####第四步，绘制所有的子view

```
 // Step 4, draw the children
        dispatchDraw(canvas);
        
/**
     * Called by draw to draw the child views. This may be overridden
     * by derived classes to gain control just before its children are drawn
     * (but after its own view has been drawn).
     * @param canvas the canvas on which to draw the view
     */
    protected void dispatchDraw(Canvas canvas) {

    }
```
这里看到`dispatchDraw`是个空方法，注释里面说，如果包含子view就需要去重写这个方法，去viewgroup里面看看。

```
 @Override
    protected void dispatchDraw(Canvas canvas) {
        ......
        final int childrenCount = mChildrenCount;
        final View[] children = mChildren;
        ......
        for (int i = 0; i < childrenCount; i++) {
            ......
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                more |= drawChild(canvas, child, drawingTime);
            }
        }
        ......
        // Draw any disappearing views that have animations
        if (mDisappearingChildren != null) {
            ......
            for (int i = disappearingCount; i >= 0; i--) {
                ......
                more |= drawChild(canvas, child, drawingTime);
            }
        }
        ......
    }
```
还是去遍历所有的子view,然后去调用view.draw方法

####第五步，绘制渐变效果并且恢复图层
这个和第二部相对应，不说了

###第六步，绘制view的滚动条
也不分析了，所以其实所有的 view都是有滚动条的

到这里，整个view就差不多绘制完成了。

###View的invalidate
如果去查一查view的源码，你会发现这个方法出现的特别多，我们这个方法是用来在主线程中调用去重绘view的，但是为什么会出现这么多次，而且它是怎么去实现重绘的呢，而且开头的时候还有一个问题没有解决，简单的看一下。
view有多个重载的`invalidate`，但是最终都走到了`invalidateInternal`方法，看一下：

```
void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
            boolean fullInvalidate) {
        ......
            final AttachInfo ai = mAttachInfo;
            final ViewParent p = mParent;
            if (p != null && ai != null && l < r && t < b) {
                final Rect damage = ai.mTmpInvalRect;
                //设置刷新区域
                damage.set(l, t, r, b);
                //传递调运ViewParent， 一般是ViewGroup的invalidateChild方法
                p.invalidateChild(this, damage);
            }
            ......
    }
```
这里主要是`  p.invalidateChild(this, damage);`这个方法，
> View的invalidate（invalidateInternal）方法实质是将要刷新区域直接传递给了父ViewGroup的invalidateChild方法，在invalidate中，调用父View的invalidateChild，这是一个从当前向上级父View回溯的过程，每一层的父View都将自己的显示区域与传入的刷新Rect做交集 。

而`ViewParent`是个接口，viewgroup和`ViewRootImpl`都实现了这个接口，所以先去viewgroup看看：

```
 public final void invalidateChild(View child, final Rect dirty) {
        ViewParent parent = this;
        final AttachInfo attachInfo = mAttachInfo;
        ......
        do {
            ......
            //循环层层上级调运，直到ViewRootImpl会返回null
            parent = parent.invalidateChildInParent(location, dirty);
            ......
        } while (parent != null);
    }
```
这里进行了了循环，最终还是去了`ViewRootImpl`，看看

```@Override
    public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
        ......
        //View调运invalidate最终层层上传到ViewRootImpl后最终触发了该方法
        scheduleTraversals();
        ......
        return null;
    }
```
这里直接返回Null，退出上面的循环，而`scheduleTraversals`这个方法，最终会调用最初我们看到的`performTraversals`方法进行重新绘制。但是肯定是不会走最初的流程，经过一些判断之后只会去重绘需要重绘的地方。还是贴张图(来自[工匠若水]())
![](http://img.blog.csdn.net/20150531111928069),

> 常见的引起invalidate方法操作的原因主要有：
直接调用invalidate方法.请求重新draw，但只会绘制调用者本身。
触发setSelection方法。请求重新draw，但只会绘制调用者本身。
触发setVisibility方法。 当View可视状态在INVISIBLE转换VISIBLE时会间接调用invalidate方法，继而绘制该View。当View的可视状态在INVISIBLE\VISIBLE 转换为GONE状态时会间接调用requestLayout和invalidate方法，同时由于View树大小发生了变化，所以会请求measure过程以及draw过程，同样只绘制需要“重新绘制”的视图。
触发setEnabled方法。请求重新draw，但不会重新绘制任何View包括该调用者本身。
触发requestFocus方法。请求View树的draw过程，只绘制“需要重绘”的View。

最后再来去解释一下最初的那个view绘制入口的问题，还是接上一篇，在`setContentView`方法里面有这个` mContentParent.addView(view, params);`而在`addView`方法里面有这么一句话：

```
public void addView(View child, int index, LayoutParams params) {
        ......
        invalidate(true);
        ......
    }
```
这下明白了吧。

参考：

[ Android应用层View绘制流程与源码分析](http://blog.csdn.net/yanbober/article/details/46128379)






