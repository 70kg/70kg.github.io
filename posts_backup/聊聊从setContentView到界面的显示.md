
title: 聊聊从setContentView到界面的显示
date: 2016-07-09 23:34:45
tags: Android
---
###背景
&emsp;&emsp;为什么要写这篇呢，其实想写一篇比较深入的文章很久了，只是一直比较懒和各种接口没去花精力去实施，正好有个元旦假期，花了点时间看了一些博客再加上自己的分析，然后想记录下来，不敢说多么的精彩和深入，就当个笔记。也算是`frameWork`层的初步探索吧。开始吧。<!--more-->

###setContentView
&emsp;&emsp;在Android里面，去设置布局最常见的就是在`onCreate`方法里面使用`setContentView(int layoutResID)`这个函数把定义的XML文件设置进去，跟进去看看，发现其实有三个重载的函数：

```
public void setContentView(int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }

    public void setContentView(View view) {
        getWindow().setContentView(view);
        initWindowDecorActionBar();
    }

    public void setContentView(View view, ViewGroup.LayoutParams params) {
        getWindow().setContentView(view, params);
        initWindowDecorActionBar();
    }
```
但是最终都是调用了` getWindow().setContentView`，这个` getWindow()`又是啥？

```
public Window getWindow() {
        return mWindow;
    }
```
返回的是一个`Window`对象，这个`Window`是个抽象类(不那么啰嗦了),真正作用的是唯一的一个实现类`PhoneWindow`，在`attach`里面可以看到` mWindow = new PhoneWindow(this);`再回到前面还有个方法`initWindowDecorActionBar();`看名字是初始化Window装饰ActionBar，进去看看

```
/**
     * Creates a new ActionBar, locates the inflated ActionBarView,
     * initializes the ActionBar with the view, and sets mActionBar.
     */
    private void initWindowDecorActionBar() {
        Window window = getWindow();

        // Initializing the window decor can change window feature flags.
        // Make sure that we have the correct set before performing the test below.
        window.getDecorView();

        if (isChild() || !window.hasFeature(Window.FEATURE_ACTION_BAR) || mActionBar != null) {
            return;
        }

        mActionBar = new WindowDecorActionBar(this);
        mActionBar.setDefaultDisplayHomeAsUpEnabled(mEnableDefaultActionBarUp);

        mWindow.setDefaultIcon(mActivityInfo.getIconResource());
        mWindow.setDefaultLogo(mActivityInfo.getLogoResource());
    }
```
这里看到又调用了`getWindow`而且下面还有个`window.getDecorView();`上面的注释看到，这个函数执行之后就初始化了`window`的各种`feature flags`所以这就是为什么我们常见的设置`feature`一定要在`setContentView`之前了，因为在执行`setContentView`之后，就把Window的相关特征标志给初始化了，再去设置也没什么卵用。这个`window.getDecorView()`在`PhoneWindow`是这个样子的：
```
 @Override
  public final View getDecorView() {
     if (mDecor == null) {
          installDecor();
      }
       return mDecor;
   }
```
这个`installDecor`在`setContentView`里面也有使用，还是回去吧，看看`setContentView`里面是什么样子的

```
public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();//这个见过了，后面会再说
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        //这个在共享元素里面用过
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
        //这个就是把布局文件加入到mContentParent
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        final Callback cb = getCallback();
        //就是回调喽，可以发现我们可以多次调用setContentView，因为会removeAllViews，并且会回调onContentChanged，在这个方法里面就可以执行想要的操作了。
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
```
这里有两个疑问，`mContentParent`是啥？`installDecor();`里面到底干了啥？下面再细说一下。

###installDecor()

直接进入这个方法看看：

```
//只看关键的
private void installDecor() {
     if (mDecor == null) {
            mDecor = generateDecor();
            ...
        }
         if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
          }
}
```
这里主要关注`generateDecor`和`generateLayout(mDecor)`，这里顺便把上面的`mContentParent`的问题提到了一点，`mContentParent`就是在这里初始化的，先看上面的`   mDecor = generateDecor();`进去看到这个：

```
protected DecorView generateDecor() {
      return new DecorView(getContext(), -1);
   }
```
这个`DecorView`是啥？他是`PhoneWindow`的一个静态内部类：

     private final class DecorView extends FrameLayout implements RootViewSurfaceTaker
    
看到吧，其实就是个继承`FrameLayout`的`ViewGroup`，看一张图,转自[工匠若水](http://blog.csdn.net/yanbober/article/details/45970721)

![](http://img.blog.csdn.net/20150604144532934)
先说个结论：`mContentParent`就是`DecorView`里面的那个`content`,我们平时写的布局都是扔到了这个里面，所以叫做`setContentView()`，而`DecorView`就是包裹在外面的一层更根的布局。。不信可以进去`generateLayout(mDecor)`方法里面看看，

```
protected ViewGroup generateLayout(DecorView decor) {
        // Apply data from current theme.

        TypedArray a = getWindowStyle();

        //......
        //依据主题style设置一堆值进行设置

        // Inflate the window decor.

        int layoutResource;
        int features = getLocalFeatures();
        //......
        //根据设定好的features值选择不同的窗口修饰布局文件,得到layoutResource值

        //把选中的窗口修饰布局文件添加到DecorView对象里，并且指定contentParent值
        View in = mLayoutInflater.inflate(layoutResource, null);//加载了我们平时写的XML文件
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));//把我们写的文件加到DecorView中，并且设置为MATCH_PARENT，看吧，为什么默认都是MATCH_PARENT。
        mContentRoot = (ViewGroup) in;

        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);//这就是那个content
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }

        //......
        //继续一堆属性设置，完事返回contentParent
        return contentParent;
    }
```
总结一下`setContentView`的过程：

 1. 创建一个DecorView的对象mDecor，该mDecor对象将作为整个应用窗口的根视图。
 2. 调用`generateLayout(DecorView decor)`依据Feature等style theme创建不同的窗口修饰布局文件，然后

```
View in = mLayoutInflater.inflate(layoutResource, null);

decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT)); 

ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
```
 
到这，`setContentView`就差不多走完了，这个工作我们一般是在`onCreate()`里面进行，但是还远没有到显示的地方，下面来简单说一下显示的过程，一个Activity的开始实际是`ActivityThread`的main方法(这个还没分析过Activity的启动过程，先就这么认为吧)，当启动Activity调运完`ActivityThread`的main方法之后，接着调用`ActivityThread`类`performLaunchActivity`来创建要启动的Activity组件，在创建Activity组件的过程中，还会为该Activity组件创建窗口对象和视图对象；接着Activity组件创建完成之后，通过调用ActivityThread类的`handleResumeActivity`将它激活。看一下这个方法：

```
final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume) {

            //这个时候，Activity.onResume()已经调用了，但是现在界面还是不可见的
            ActivityClientRecord r = performResumeActivity(token, clearHide);

            if (r != null) {
                final Activity a = r.activity;
                  if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                //decor对用户不可见
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                //这里记住这个WindowManager.LayoutParams的type为TYPE_BASE_APPLICATION，后面介绍Window的时候会见到
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;

                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
                    //终于被添加进WindowManager了，但是这个时候，还是不可见的
                    wm.addView(decor, l);
                }

                if (!r.activity.mFinished && willBeVisible
                    && r.activity.mDecor != null && !r.hideForNow) {
                     //在这里，执行了重要的操作！
                     if (r.activity.mVisibleFromClient) {
                            r.activity.makeVisible();
                        }
                    }
            }
```
其中的`makeVisible`方法：

```
void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }
```
到现在，整个界面才真正的显示出来，所以当调用了`onResume`方法，界面也不一定是显示的。这里还有几个问题，`mLayoutInflater.inflate(layoutResID, mContentParent)`这个到底是怎么把XML解析成view的，`findViewById`是怎么找view的，还有就是`WindowManager`是啥？他和前面提到的`Window`，`PhoneWindow`，`DecorView`啥关系？下面会再来扯一扯。

###findViewById
我们平时在`Activity`里面去使用，跟进去看看

```
public View findViewById(@IdRes int id) {
        return getWindow().findViewById(id);
    }
```
好吧，熟悉吧，再进去看看

```
 @Nullable
    public View findViewById(@IdRes int id) {
        return getDecorView().findViewById(id);
    }
```
看，到了view的地方，再进去

```
public final View findViewById(@IdRes int id) {
        if (id < 0) {
            return null;
        }
        return findViewTraversal(id);
    }
```
再进去看看这个递归函数是怎么回事：

```
protected View findViewTraversal(@IdRes int id) {
        if (id == mID) {
            return this;
        }
        return null;
    }
```
就是返回自己啊，但是仔细想想，还是得去`ViewGroup`去看，具体原因就不扯了，这样的：

```
@Override
    protected View findViewTraversal(@IdRes int id) {
        if (id == mID) {
            return this;
        }

        final View[] where = mChildren;
        final int len = mChildrenCount;

        for (int i = 0; i < len; i++) {
            View v = where[i];

            if ((v.mPrivateFlags & PFLAG_IS_ROOT_NAMESPACE) == 0) {
                v = v.findViewById(id);

                if (v != null) {
                    return v;
                }
            }
        }

        return null;
    }
```
就是去遍历`ViewGroup`里面的`View`，然后去调用`View`的`findViewById`，所以我们的`findViewById`都是在`DecorView`这个`ViewGroup`里面去查找的。

###mLayoutInflater.inflate
当需要加载个XML布局文件的时候，一般这样使用` LayoutInflater.from(this).inflate()`，里面有一些函数的重载，但是最终都是走到了这里：

```
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }

        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
```
使用XML解析器去解析XML文件，关键是第10行，进去看看：

```
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        ...
        //返回值
            View result = root;
        ...
                //merge的布局优化，root必须非空且attachToRoot为true，
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }
                    //递归解析
                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml根据节点创建view
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;
。。。
                        // Create layout params that match root, if supplied  生成合适的layout params
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }
                    // Inflate all children under temp against its context. 继续递归
                    rInflateChildren(parser, temp, attrs, true);

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                //root非空且attachToRoot=true则将xml文件的root view加到形参提供的root里
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                    //这里就直接返回解析的view
                        result = temp;
                    }
                }

        。。。

            return result;
        }
    }
```
所以从上面的代码可以总结一下：

* inflate(xmlId, null); 只创建temp的View，然后直接返回temp。

* inflate(xmlId, parent); 创建temp的View，然后执行root.addView(temp, params);最后返回root。

* inflate(xmlId, parent, false); 创建temp的View，然后执行temp.setLayoutParams(params);然后再返回temp。

* inflate(xmlId, parent, true); 创建temp的View，然后执行root.addView(temp, params);最后返回root。

* inflate(xmlId, null, false); 只创建temp的View，然后直接返回temp。

* inflate(xmlId, null, true); 只创建temp的View，然后直接返回temp。

这里也引出了一个问题，就是有时候我们通过View的`layout_width`和`layout_height`来设置view的大小，然后通过`inflate`解析之后发现并不管用，这里的代码就可以说明问题，其实这两个属性并不是用来设置view的大小的，而是用来设置view在ViewGroup中的大小，(有点晕？后面会有专门的一篇来说这个)，所以叫做`layout_width`而不是直接叫做`width`。简单举个例子说一下：

* `mInflater.inflate(R.layout.textview_layout, null)`不能正确处理我们设置的宽和高是因为layout_width，layout_height是相对了父级设置的，而此temp的getLayoutParams为null。
* `mInflater.inflate(R.layout.textview_layout, parent)`能正确显示我们设置的宽高是因为我们的View在设置setLayoutParams时`params = root.generateLayoutParams(attrs)`不为空。 

其他就不多说了，也就是说只有这是了父布局才能正确的显示宽和高，同时可以注意到，在`Activity`中指定布局宽高的时候是可以正确显示的，这不正好说明了还存在更底层的一层id为content的FrameLayout吗。



参考：

[【凯子哥带你学Framework】Activity界面显示全解析](http://blog.csdn.net/zhaokaiqiang1992/article/details/49681321)

[ Android应用setContentView与LayoutInflater加载解析机制源码分析](http://blog.csdn.net/yanbober/article/details/45970721)

[PhoneWindow源码](http://grepcode.com/file/repo1.maven.org/maven2/org.robolectric/android-all/5.0.0_r2-robolectric-1/com/android/internal/policy/impl/PhoneWindow.java#PhoneWindow.generateLayout%28com.android.internal.policy.impl.PhoneWindow.DecorView%29)