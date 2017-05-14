---
title: 《Android开发艺术探索》笔记
date: 2016-03-04 18:36:38
tags:
---
#### 第一章 Activity的生命周期和启动模式

 - 在启动新activity时，必须在当前activity`onPause()`方法执行完成才执行新activity的`onCreate()`，所以在`onPause()`方中不能进行重量级操作，可以进行比如短时间的存储数据和停止动画等工作。
 - 当打开新的activity或者回到桌面时，会调用`onPause()`->`onStop()`,但是当新的activity采用了透明的主题，那么不会调用`onStop()`<!--more-->
 - 从activity是否可见，`onStart()`和`onStop()`是配对的，从activity是否在前台来说，`onResume()`和`onPause()`是配对的
 - 当启动一个activity，当前`onPause()`->新`onCreate()`->`onStart()`->`onResume()`->当前`onStop()`,所以耗时操作尽量在`onStop()`方法中进行操作
 - 当系统配置发生改变后，会调用`onSaveInstanceState`方法来进行保存当前状态，是在`onStop()`之前，和`onPause()`没有固定的顺序，当重新创建时，调用`onRestoreInstanceState`方法，在`onStart()`之后调用。
 - 关于屏幕旋转，配置了`android:configChanges = "orientation|screenSize"`是不会进行重新创建的，在API13之后，单独设置一个都会进行重新创建。这时会去调用`onConfigurationChanged`方法，而不走正常流程。
 
#### Activity的LaunchMode
 - standard:这是默认的启动模式，会一直创建新的对象并加入任务栈。谁启动了这个activity，这个activity就运行在启动它的那个activity的所在栈中。当使用`ApplicationContext`去启动activity时会报错误，请求新建任务栈。这是因为这种模式的activity会默认进入启动它的activity任务栈，但是非activity的context并没有所谓的任务栈，这就出问题了，所以可以指定`FLAG_ACTIVITY_NEW_TASK`标记位去创建一个新的任务栈。
 - singleTop:栈顶复用模式，如果新的activity位于栈顶，不会去创建新的，同时去回调`onNewIntent`方法去获取启动intent，不会去走正常的流程。
 - singleTask:栈内复用模式，这是一种单实例模式，只要栈内存在activity，就不会去创建，也是回调`onNewIntent`方法，同时，singleTask还默认拥有`cleanTop`效果。
 - singleInstance:实单例模式，这是加强型的singleTask，除了singleTask的特点，它会为新activity启用一个新的任务栈。

#### 任务栈

 - TaskAffinity,任务相关性，这个参数标识了一个activity所需任务栈的名字，默认情况下，所以activity所需的任务栈的名字为其应用的包名。当然我们可以自己设置，`TaskAffinity`属性主要和`singleTask`启动模式或者`allowTaskReqarenting`属性配对使用。在其他情况下没意义。
 - 任务栈分为前台任务栈和后台任务栈，后台任务栈中的activity处于暂停状态，用户可以通过切换将后台任务栈再次调用到前台。
 - 在activity下面定义`android:taskAffinity = "包名.自定义"`,一般还有`android:launchMode = "singleTask"`

#### Activity的Flags

 - FLAG_ACTIVITY_NEW_TASK: 就是singletask模式
 - FLAG_ACTIVITY_SINGLE_TOP:就是singletop模式
 - FLAG_ACTIVITY_CLEAN_TOP:清除当前activity上面的所以activity，一般和`FLAG_ACTIVITY_NEW_TASK`配合使用，`singletask`默认拥有这个属性。
 - FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS:拥有这个标志的activity不会出现在历史activity列表中。等同于XML设置`android:excludeFromRecents = "true"`

#### intentFilter的匹配规则

 - action：可以有多个，要求intent中的action 存在 且 必须和过滤规则中的其中一个相同 区分大小写；
 - category：系统会默认加上一个`android.intent.category.DEAFAULT`，所以intent中可以不存在category，但如果存在就必须匹配其中一个，一般都会加上默认的；
 - data:data由两部分组成，mimeType和URI，要求和action相似。如果没有指定URI，URI默认值为content和file（schema）;
 [Android入门：隐式Intent](http://blog.csdn.net/xiazdong/article/details/7764865)

#### IPC机制

 - IPC就是进程间通信或者跨进程通信，每个操作系统都有对应的IPC机制，Android上有特色的Binder。
 - 使用多进程最简单就是给四大组件指定`android:process`,而且Android使用多进程只有这一种方法（特殊的通过JNI去fork一个新进程不算）。
 - 在使用`android:process:remote`是默认在前面加上了包名。当然也可以自己去指定进程名称，其中以：开头的进程是属于当前应用的私有进程，和其他应用的组件不可以和它泡在同一个进程中国，而进程名不以：开头的属于全局进程，其他应用通过shareUID方式可以和它泡在同一个进程中。
 - Android为每一个应用分配了一个独立的虚拟机，或者说为每一个进程分配了一个虚拟机，不同的虚拟机在内存分配桑有不同的地址空间，这就导致了在不同虚拟机中访问同一个类的对象会产生多份副本。
 - 所有运行在不同进程中的四大组件，只要他们之间需要通过内存来共享数据，都会共享失败。所以多进程会造成下面问题：
    1：静态成员和单例模式完全失效
    2：线程同步机制完全失效
    3：Sharedpreference可靠性下降
    4：application会多次创建
 - 不同进程的组件会拥有独立的虚拟机，application以及内存空间。

#### Serializable接口

 - 这是JAVA提供的序列化接口，它是一个空接口，只要实现了这个接口，就可以实现序列化。其实还需要提供一个`SerialVersionUID`,这个在反序列化的时候去检测是否是同一个版本，比如当成员变量的数量，类型改变时就无法反序列化。所以还是尽量去指定。还有，静态成员变量属于类不属于对象，所以不会参加序列化过程，其实使用`tranient`关键字标记的成员变量不会参加序列化。

#### Paracelable接口

 - 这个Android提供的一种序列化接口，实现这个接口就可以完成序列化并可以通过`Intent`和`Binder`传递。这个写法稍微复杂点。
 - 这两个的区别就是`Serializable`是JAVA中的序列化接口，简单但是开销大，需要大量的`I/O`操作，`Paracelable`效率高，但是稍微复杂，取决与自己吧。

#### Binder

 - Binder就是Android中的一个类，继承了`IBinder`接口。主要用于`Service`中，包括`AIDL`和`Messenger`，其中`Messenger`底层就是`AIDL`，这个后面有，所以还在主要看`AIDL`。
 - 对应介绍`AIDL`，推荐看一下洪洋的[Android aidl Binder框架浅析](http://blog.csdn.net/lmj623565791/article/details/38461079)。其中在`Eclipse`中创建正确的`aidl`文件会自动在gen下面生成对应的JAVA文件，但是在Android studio中即使按提示创建了正确的AIDL文件，在debug下面还是不会出现，这时make project就可以了。（也是坑死我了）。
 - 说一下使用`aidl`的简单流程：
    1：创建aidl文件，就是一个接口
    2：在service中使用接口去获取Binder
    3：就是在客户端调用喽
 - 我们去定义一个`aidl`文件只是去简化工作，让系统去帮助我们生成一个对应的接口类（当然我们可以手写这个类）。这个接口类是继承了`IInterface`这个接口，同时自己也是一个接口，所有可以在`Binder`中传输的接口都需要继承`IInterface`接口。要分析这个类没有具体的案例也不好说，还是去看上面那篇博客吧。我自己也写了一遍[我的GitHub](https://github.com/70kg/Android-Studio-Project/tree/master/app/src/main/java/com/com/mr_wrong/AIDL).里面还有对Binder的`linkToDeath`和`unlinkToDeath`方法的介绍，这两个主要就是在服务端异常终止的时候进行重新绑定。（我自己测试好像没什么用。。估计姿势不对😂）。

#### Android中的IPC方式

 - 使用Bundle,它是实现了`Paracelable`接口的，我们可以往里面put一些支持的序列化对象，它有个父类`BaseBundle`,里面维护了一个`ArrayMap`。
 - 使用文件共享，这个在对读写没有并发的多进程情况下也是一种比较简单的方式。然而`SharedPreference`是个特例，虽然它也是去读取XML文件，但是它在内存中会有缓存啊。。so....
 - 使用Messenger,这个前面说过了，还是推荐看一下洪洋的博客[Android 基于Message的进程间通信 Messenger完全解析](http://blog.csdn.net/lmj623565791/article/details/47017485).这就是对`aidl`的封装。
 - 还有就是`AIDL`了。
#### AIDL
 - Messenger是以串行的方式处理客户端发来的信息，所以只适用于低并发的情况。
 - AIDL只支持特定的数据类型
    1：基本数据类型
    2：String和CharSequeue
    3:只支持ArrayList 并且里面的元素也要能被AIDL支持
    4：只支持HashMap  并且里面的元素也要能被AIDL支持
    5：所以实现了`Paracelable`接口的对象
    6：AIDL AILD本身也能在AIDL文件中使用
 - 其中自定义的`Paracelable`对象和AIDL对象要显式的import进来，不管他们是否和当前的aidl文件位于同一个包内
 - 同时 使用自定义的`Paracelable`对象时，要为他独立建一个同名的AIDL文件，并且声明为`Paracelable`类型
 - 服务端可以使用`CopyOnWriteArrayList`和`ConcurrentHashMap`来进行自动线程同步，支持并发读/写,客户端拿到的依然是ArrayList和HashMap
 - 可以使用`RemoteCallbackList`来进行解注册，他是系统专门提供的用于删除夸进程`listener`的接口，但是我没无法像list一样去操作它，要遍历要使用`listener.beginBroadcast()`和`listener.finishBroadcast()`。
 - 客户端通过IBinder.DeathRecipient来监听Binder死亡，也可以在onServiceDisconnected中监听并重连服务端。区别在于前者是在binder线程池中，访问UI需要用Handler，后者则是UI线程。  
 - 可以在`onBind`中进行权限的验证，还可以在服务端的`onTransact()`方法中采用uid和pid进行验证

#### 使用ContentProvider

 - `ContentProvider`是Android提供的专门用于不同应用间的数据共享方式，底层也是使用`Binder`实现的。
 - `ContentProvider`主要以表格的形式组织数据
 - 这里有一个使用`ContentProvider`和`SQLite`数据库的例子，就不贴代码了[我的GitHub](https://github.com/70kg/Android-Studio-Project/tree/master/app/src/main/java/com/com/mr_wrong/ContentProvider)。

#### 使用Socket

 - `Socket`也可以进行进程间通信，虽然更多的时候用来进行网络数据交换。这样也写了一个使用`TCP`实现`Socket`通信的例子，里面有个坑坑死我了，总是接受不到服务器信息，最后发现是`out.println(msg)` 写成`print`就完蛋。[我的GitHub](https://github.com/70kg/Android-Studio-Project/tree/master/app/src/main/java/com/com/mr_wrong/Socket)

#### 使用Binder连接池

 - 这个就是用于当一个项目用到很多的AIDL,总不能每个AIDL都去搞个`service`，可以把所以的AIDL都扔到一个`service`里面去管理，根据不同的业务模块拿到不同的`Binder`对象。Binder连接池的主要作用就是将每个业务模块的`Binder`请求统一转发到远程`service`上去执行。（其实感觉这个有点像个工厂）。
 - 具体我只是看了一下，并没有去写完这个代码，一是因为我的socket浪费了太多的时间，二是因为要使用到Binder连接池就是很大很大的项目了，目前还接触不到。原理并不复杂，再去写一个`BinderPool`的aidl接口，去管理里面的其他aidl，然后根据不同的业务需求返回不同的`IBinder`。用到再好好研究吧。

***
  下面就是开始`View`的部分，更"赤几"的来啦😁
***

#### View的事件分发机制

 - 主席的书对这个的讲解还是很精巧的。让人很容易的去理解这个过程。
 - 这个事件分发机制我们只需要去关心三个方法
   1：`public boolean dispatchTouchEvent(MotionEvent ev)`
用来进行事件的分发，如果事件能够传递给当前的view，那么这个方式肯定会调用。返回结果受当前view的`onTouchEvent`和下级view的`dispatchTouchEvent`影响，表示是否消耗当前事件。
   2：`public boolean onInterceptTouchEvent(MotionEvent ev)`
用来判断是否拦截某个事件，如果拦截了，那么在同一个事件序列中不会再次调用。返回结果表示是否拦截这个事件。
   3: `public boolean onTouchEvent(MotionEvent event)`
在`dispatchTouchEvent`中调用，用来处理点击事件。返回结果表示是否消耗当前事件。如果不消耗，在同一个事件序列中不会再次接受事件。

 - 下面就是精巧的伪代码表示三者关系时间啦
 
```
 public boolean dispatchTouchEvent(MotionEvent ev) {
        boolean consume = false;
        if (onInterceptTouchEvent(ev)){
            consume = onTouchEvent(ev);
        }else {
            consume = child.dispatchTouchEvent(ev);
        }
        return consume;
    }
```
解释一下：对于一个根`viewgroup`来说，点击事件产生后，首先调用`dispatchTouchEvent`方法，如果它的`onInterceptTouchEvent`返回true，表示它想拦截这个事件，然后就交给它的`onTouchEvent`来进行处理，否则就交给下一级view的`dispatchTouchEvent`再进行循环。

 - 如果view设置了`setOnTouchListener`,那么优先调用`setOnTouchListener`，如果返回false,那么再去调用`onTouchEvent`，返回true，说明已经消耗了，`onTouchEvent`不会调用了。还有常用的`setOnClickListener`它的优先级是最低的，没错，是最低的。。
 - 正常情况下，一个事件序列只能被一个view拦截和消耗。除非强行传递给其他的view。。
 - 某个view一旦开始处理事件，如果不消耗`ACTION_DOWN`即返回false,那么同一实践序列中的其他事件都不会交给他处理，并且将事件交给他爹处理。也就是说事件一旦交给你了，你就必须消耗掉，不然以后的都不给你了。😂
 - 如果view不消耗掉除`ACTION_DOWN`以外的其他事件，那么这个点击事件就会消失。此时父元素的`onTouchEvent`不会调用，最终会传递给`activity`处理。
 
#### view的滑动冲突

 - 这里主席总结的也很好，主要有两种方式去解决滑动冲突。
    1：外部拦截法
这个是在父布局中的`onInterceptTouchEvent`方法中进行判断，根据具体的业务逻辑是否去拦截掉事件。主要还是在`MotionEvent.ACTION_MOVE`中判断，然后返回值。
    2：内部拦截法
这个方法是指父容器不拦截任何事件，全部交给子元素。如果子元素需要就消耗掉，如果不需要再返回给父元素。这个主要配合`requestDisallowInterceptTouchEvent`来对父布局的拦截事件操作。这个也不好用，也比较麻烦，和整个事件传递的思想也不怎么搭。就不写代码了。

#### View的工作原理

 - `ViewRoot`对用于`ViewRootImpl`类，它是连接`WindowManager`和`DecorView`的纽带，view的三大流程均是通过`ViewRoot`来完成的。
 - 在`ActivityThread`中，当`Activity`对象被创建完毕之后，会讲`DecorView`添加到`Window`中，同时创建`ViewRootImpl`类，并将`ViewRootImpl`和`DecorView`建立关联。
 - view的绘制流程是冲`ViewRoot`的`performTraversals`方法开始的，它经过`measure`，`layout`,`draw`三个过程才能最终将一个view绘制出来。
 - `DecorView`作为顶级view，内部有标题栏和一个名为`content`的`Framelayout`，这个就是我们常用的`setcontentview`的地方，这也是为什么叫做`setcontentview`而不是`setview`。在实际使用的时候也会发现这个现象。

#### MeasureSpec

 - 在测量过程中，系统会讲view的`LayoutParams`根据父容器所施加的规则转换成对应的`MeasureSpec`，然后根据这个`MeasureSpec`来测量view的宽和高。
 - 对于`DecorView`，其`MeasureSpec`是由窗口的尺寸和其自身的`LayoutParams`来共同决定的。对于普通的view，其`MeasureSpec`由其父容器的`MeasureSpec`和其自身的`LayoutParams`来共同决定的。这个想想实际使用中遇到的，也是挺好理解的。
 - 对于测量中的`mode`就不写了，这个每本书都会去介绍，也说不出什么别的花样来，就是那么回事。
 - view的`measure`过程是由其`measure`方法完成。`measure`方法又是个`final`的，但是在`measure`方法中会去调用`onMeasure`方法，这就是我们常见的了。
 - view的默认在`AT_MOST`也就是`wrap_content`下是去使用`EXACTLY`(match_parent)模式的。所以当我们是直接继承view，还是乖乖去写`onmeasure`吧（大部分情况下。。😂）。
 - 在`activity`的`oncreate`，`onstart`,`onresume`中都没法得到正确的view宽高。这是因为view的measure过程和`activity`的生命周期不是同步的，so.. 解决办法
 
    1：`onWindowFocusChanged`这个是view和activity都有的方法，调用这个方法意味着view已经初始化完毕，可以去获取宽高。但是这个方法会在activity的窗口获得焦点和失去焦点的时候调用。

    2：`Veiw.post(runnable)`这个是将一个runnable对象加入到当前的`messagequeue`中，等到`measure`和`layout`等方法调用完成之后再去调用。
    
    3：使用`ViewTreeObserver`这个有多个回调。比如使用
    
```
    ViewTreeObserver observer = mJellyTextView.getViewTreeObserver();
        observer.addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
            @Override
            public void onGlobalLayout() {
                
            }
        });
```
当view树的状态发生改变或者view树内部的view的可见性发生改变时，都会去回调这个方法。要注意的是，这个方法会多次调用。
    
   4：使用`view.measure(int widthMeasureSpec, int heightMeasureSpec) `这个我不是特别的会用，如果只是为了获取宽高，为啥不去使用post方法，简单又好用。
    

 - 在view的默认实现中`getMeasuredWidth()`和`getWidth()`的值是相等的。只不过测量值形成于measure过程，左中的宽高形成于layout过程。一般可以认为是一样的。但是可以在重写layout中强制改变，然而并没有什么卵用。。

#### Draw过程

  1:绘制背景`background.draw(canvas)`
  
  2: 绘制自己`ondraw`
  
  3: 绘制children`dispatchDraw`
  
  4: 绘制装饰`onDrawScrollBars`
 
 - 在继承viewgroup进行绘制的时候，去看tips里面的吧。
 - 不要在view中使用handler，因为view有post啊😂
 - view中有线程或者动画，需要及时停止，参考`onDetachedFromWindow() `（在查看大图的时候可以在这里设置为空，可以让gc更快的去回收）。

#### 理解RemoteViews

 - `RemoteViews`表示一个view结构，它可以在远程进程中显示，这就是说它提供了一组基础操作用于跨进程更新界面。（叼不叼😂）。这个东西主要用在通知栏和桌面小部件。
 - 在使用通知栏时，使用`NotificationManager`的`nofity`实现，使用小部件使用`AppWidgetManager`实现，他们都运行在其他的进程，确切说是`SystemServier`进程，所以没法像以前那么直接去更新view，要使用`RemoteViews`提供的一系列set方法。set支持的属性有限，同样`RemoteViews`支持的view也有限。
 - 在通知栏的使用看我之前写的[Android 5.0通知栏](https://github.com/70kg/Android-Studio-Project/blob/master/app/src/main/java/com/com/mr_wrong/Notification/NotificationActivity.java)
 - 在小部件的使用我还没写，因为我在麦当劳，这里的声音实在太吵了。[先看看主席的吧](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_5/src/com/ryg/chapter_5/MyAppWidgetProvider.java).(ps：我是很不喜欢使用小部件的，又丑又鸡肋还占地方，看起来很方便，其实体验跟屎一样。😤)
 - `PendingIntent`表示一种处于`pending`状态的intent，就是要有某些条件去触发它去执行。支持三种使用：启动activity,启动service和发送广播。分别就是`PendingIntent.getActivity(content, requestcode, intent, flags)`。。`requestcode`表示发送方的请求码，多数情况下设为0就可以。
 -  `PendingIntent`相同的定义：内部的Intent和requestCode都相同。Intent相同的定义：两个Intent的componentName和intent-filter相同（不包括extras）
 -  flag定义：
 1：FLAG_NO_CREATE,基本不使用；
 2：FLAG_ONE_SHOT，以第一个为准，后续的会全部和第一条保持一致，任意一条被触发，其他的都cancel；
 3：FLAG_CANCEL_CURRENT，前面的相同的PendingIntent都会被cancel，只有最新的可用；
 4：FLAG_UDPATE_CURRENT，前面的PendingIntent都会被更新（它们Intent中的extras都会被更新）

#### `RemoteViews`的内部机制

 - `RemoteViews`常用的构造方法`RemoteViews(getPackageName(), R.layout.notification);`第一个就是当前应用的包名，第二个就是布局ID。
 - `RemoteViews`只支持有限的view类型：
   Layout：FrameLayout, LinearLayout, RelativeLayout, GridLayout;

   View: AnalogClock, Button, Chronometer, ImageButton, ImageView, ProgressBar, TextView, ViewFlipper, ListView, GridView, StackView, AdapterViewFlipper, ViewStub。
   

 - 客户端的remoteViews通过action对象，由binder机制来更新服务端的remoteViews，所以RemoteViews本身也实现了Parcelable接口。
 - RemoteViews中真正操作View的方法apply和reapply，前者会加载布局并更新界面，后者则只更新界面。
 - `RemoteViews`只支持`PendingIntent`，不支持`OnClickListener`。可以使用`setOnClickPendingIntent`给普通的view设置点击事件，但是不能给集合(lisview,stackview)中的view设置点击事件。必须使用`setPendingIntentTemplate`和`setOnClickFillInIntent`组合使用。
 - [还有个例子](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_5/src/com/ryg/chapter_5/MainActivity.java)配个[这个](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_5/src/com/ryg/chapter_5/DemoActivity_2.java)食用

#### Android的Drawable

 - 这章总体比较简单，再加上之前也写过关于`Drawable`的一些东西。估计会写的少点。在麦当劳喝了凉的再加上吹空调搞得我肚子难受，又不放心电脑放这去厕所，所以。。😂
 - `Drawable`有很多种，他们都表示一种图像的概念。`Drawable`是一个抽象类，他是所有`Drawable`对象的基类。可以通过`getIntrinsicWidth`和`getIntrinsicHeight`来获取内部的宽高。但是并不是所有的`Drawable`都有宽高，一张图片形成的可以获取，但是一个颜色形成的就无法获取。同时`Drawable`的宽高并不等于他们的大小，`Drawable`一般是没有大小的概念，当用作view的背景时，`Drawable`会被拉伸至view的同等大小。
 
#### `Drawable`的分类
 - BitmapDrawable
 就是一张图片。可以设置tilemode进行镜像平铺什么的
 - ShapeDrawable
 可以理解为通过颜色来构造的图形。可以渐变
 - LayerDrawable
 他是一种层次化的Drawable集合，可以将不同的`Drawable`放置在不同的层上面来达到一种层叠的效果。
 - StateListDrawable
 这个用在view的不同状态改变的时候。有挺多属性的。
 - LevelListDrawable
 也是一个Drawable的集合，每个Drawable都有一个等级，根据等级的不同可以切换不同的Drawable
 - TransitionDrawable
 淡入淡出的效果
 - InsetDrawable
 可以将其他的Drawable内嵌到自己当中。
 - ScaleDrawable
 根据自己的等级将指定的Drawable缩放到一定的比例。
 - ClipDrawable
  做裁剪的  用过这个。

对于自定义的`Drawable`，我觉得可以再讲讲关于`Drawable`的`Callback`接口和关于`Animatable`接口的相关知识，这两个配合起来使用对自定义`Drawable`才更加丰富。


#### Android动画深入分析

 - 这个动画方面的之前就了解过。再看一遍吧。
 
#### view动画
 - view动画主要就是`Animation`的四个子类`TranslateAnimation`,`ScaleAnimation`,`RotateAnimation`,`AlphaAnimation`。对于view动画还是建议使用XML来进行定义，具体就不写了。
 - 自定义view动画只要继承`Animation`这个抽象类就可以了，主要涉及矩阵的变化，还有配合`camera`的使用，写过一个[我的GitHub](https://github.com/70kg/Android-Studio-Project/blob/master/app/src/main/java/com/com/mr_wrong/CustomAnimAndCamera/CustomAnim.java)
 - 帧动画就不说了，使用尺寸大图片容易oom
 - `layoutanimation`也不说了，也写过[我的GitHub](https://github.com/70kg/Android-Studio-Project/blob/master/app/src/main/java/com/com/mr_wrong/Property_Animation/myLayoutTransition.java)
 - `activity`的切换效果，这个其实说也没什么用，在5.0之后的效果要酷炫的多，而且不兼容前面的，要说也是独立开一篇说。
 - 属性动画看我之前写的吧[Android属性动画](http://70kg.info/2015/07/03/Android%E5%B1%9E%E6%80%A7%E5%8A%A8%E7%94%BB/)。还有相关代码[我的GitHub](https://github.com/70kg/Android-Studio-Project/tree/master/app/src/main/java/com/com/mr_wrong/Property_Animation)。属性动画的具体原理是利用了反射去设置属性值。

#### 理解Window和WindowManager

 - `Window`是一个抽象类，它的具体实现是`PhoneWindow`，创建一个`Window`需要通过`WindowManager`。`Window`的具体实现是在`WindowManagerSeivice`中，`WindowManager`和`WindowManagerSeivice`的交互是一个ipc过程。Android中的所有视图都是通`Window`来呈现，所以`Window`实际上是view的直接管理者。
 
#### `Window`和`WindowManager`
 - `Window`的简单使用
```
 WindowManager mWindowManager = (WindowManager) getSystemService(Context.WINDOW_SERVICE);
       WindowManager.LayoutParams layoutParams =
               new WindowManager.LayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT
               0,0, PixelFormat.TRANSPARENT);
        layoutParams.flags = WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL|
                WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE|
                WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED;
        layoutParams.gravity = Gravity.LEFT|Gravity.TOP;
        layoutParams.x = 100;
        layoutParams.y = 100;
        mWindowManager.addView(mHeadImg,layoutParams);
```
其中`WindowManager.LayoutParams`的flags属性比较重要，它可以控制`Window`的显示特性。

 - `FLAG_NOT_FOCUSABLE`
表示`Window`不需要获取焦点也不需要接收各种输入事件，此标记同时会启用`FLAG_NOT_TOUCH_MODAL`最终事件会直接传递给下层具有焦点的`Window`
 - `FLAG_NOT_TOUCH_MODAL`
 这个表示当前`Window`区域以外的单击事件传递给下层的`Window`，当前`Window`区域内的点击事件自己处理。一般都要写写个标记
 - `FLAG_SHOW_WHEN_LOCKED`
 这个可以显示在锁屏上
 - type参数表示`Window`的类型，有三中类型：应用`Window`，子`Window`，系统`Window`。`Window`是分级的，应用的在1~99，子在1000~1999，系统的在2000~2999，大的会层叠在小的上面。我们想要显示在上层，可以使用`TYPE_SYSTEM_ERROR`或者`TYPE_SYSTEM_OVERLAY`。同时还有添加相应的权限。
 - `WindowManager`常用的方法就三个添加view，更新view和删除view。具体的实现过程就不写了，我看了会也是云里雾里的。

### `Window`的创建过程

#### activity的`Window`创建过程
 1：如果没有`DecorView`就去创建它
 2：将view添加到`DecorView`的`mContentParent`中.
 3:回调activity的`onContentChanged`方法去通知activity视图已经发生改变。

#### Dailog的`Window`创建过程
  1：创建`Window`
  2：初始化`DecorView`并将dialog的视图添加到`DecorView`中
  3：将`DecorView`添加到`Window`中显示
 
 - 普通的dialog必须使用activity的context，因为application的没有token，一般只有activity才有。

#### `Toast`的`Window`创建过程

 - 这个比较复杂一点，由于具有定时取消功能，所以采用了Handler.这也意味着`Toast`无法在没有looper的线程中使用。
 - 内部有一个`mToastQueue`的队列。为了防止DOS,最多只能同时存在50个。

 
## 四大组件的工作过程
#### **我是真没怎么看懂**😂
主席说了很多东西，看起来是很有价值，然而我笨没办法。留着以后再看吧，或许到时候就看得懂了。。。
***
看到了别人写的笔记，真的是差距啊。
#### Activity的工作过程 
![](http://img.blog.csdn.net/20151002094304318)
#### 启动Service 
![](http://img.blog.csdn.net/20151002094641495)
#### 绑定Service 
![](http://img.blog.csdn.net/20151002094725602)
#### 动态注册BroadcastReceiver
![](http://img.blog.csdn.net/20151002094832822)
发送和接收
![](http://img.blog.csdn.net/20151002094847398)
#### ContentProvider
-  当ContentProvider所在的进程启动的时候，它会同时被启动并被发布到AMS中，这个时候它的onCreate要先去Application的onCreate执行。
-  启动方式请看page363，这里就不画了，书上画的非常清晰易懂，就写几个tips 吧。 
1. 应用启动的入口为ActivityThread的main方法，main方法会创建ActivityThread实例并创建主线程消息队列。 
2. attach方法中远程调用AMS的attachApplication方法，并提供ApplicationThread用于和AMS的通信。 
3. attachApplication方法会通过bindApplication方法和H来调回ActivityThread的handleBindApplication，这个方法会先创建Application，再加载ContentProvider，然后才会回调Application的onCreate方法。 
4. ContentProvider的multiprocess属性决定了ContentProvider是否是单例（false时），一般都用单例。 
5. ContentResolver的具体类是ApplicationContentResolver，当ContentProvider所在进程未启动时，第一次访问它会触发ContentProvider的创建以及进程启动。 
- query流程 
![](http://img.blog.csdn.net/20151002095010807)

### Android的消息机制
 
* 三大件：Hanlder，MessageQueue，Looper。
* MessageQueue内部的数据结构并非队列，而是单链表，它只是用来存储数据。
* Looper是真正的数据处理者，线程默认没有Looper，使用Handler必须为线程创建Looper，UI线程也就是ActivityThread创建时会初始化Looper，所以主线程中默认可以直接使用Handler。
* UI线程检查当前线程的操作在ViewRootImpl的checkThread方法中，我们常见的不能在子线程中访问view的异常就是在这里抛出的。
* 不允许子线程访问主线程的原因是UI控件不是线程安全的，而加锁又会导致UI的操作过于复杂。
* Handler的工作过程，我简单概括一下就是：假设创建Handler的线程是A，耗时操作的线程是B，B线程中拿到handler实例发送消息或者post一个Runnable，实际是调用了MessageQueue的enqueueMessage方法，进入了消息队列，线程A中的Looper发现了这个消息，就会处理这个消息，也就是消息中的Runnable或者handler的handleMessage方法会被调用，这样handler中的业务逻辑就被切换到线程A中去了。
* ThreadLocal我用一句大白话来讲解，就是看上去只new了一份，但在每个不同的线程中却可以拥有不同数据副本的神奇类。其本质是ThreadLocal中的Values类维护了一个Object[]，而每个Thread类中有一个ThreadLocal.Values成员，当调用ThreadLocal的set方法时，其实是根据一定规则把这个线程中对应的ThreadLocal值塞进了Values的Object[]数组中的某个index里。这个index总是为ThreadLocal的reference字段所标识的对象的下一个位置。
* MessageQueue的工作原理：主要方法为enqueueMessage和next。 
a. enqueueMessag主要就是一个单链表的插入操作， 
b. next方法是一个无限循环，如果消息队列中没有消息，next方法就阻塞，有新消息到来时，next方法会返回这条消息并将其从单链表中删除。
* Looper的工作原理： 
a. prepare方法，为当前没有Looper的线程创建Looper。 
b. prepareMainLooper和getMainLooper方法用于创建和获取ActivityThread的Looper。 
c. quit和quitSafely方法，前者立即退出，后者只是设定一个标记，当消息队列中的所有消息处理完毕后会才安全退出。子线程中创建的Looper建议不需要的时候都要手动终止。 
d. loop方法，死循环，阻塞获取msg并丢给msg.target.dispatchMessage方法去处理，这里的target就是handler。
* Handler的工作原理： 
a. 无论sendMessage还是post最终都是调用的sendMessageAtTime方法。 
b. 发送消息其实就是把一条消息通过MessageQueue的enqueueMessage方法加入消息队列，Looper收到消息就会调用handler的dispatchMessage方法。它的处理过程参考书page388的流程图，一看就懂～ 
c. 这里我补充一个东西，当我们直接Handler h = new Handler()时，本质调用的是Handler(Callback callback, Boolean async)构造方法，这个方法里会调用Looper.myLooper()方法，这个方法其实就是返回的ThreadLocal里保存的当前线程的Looper，这也就解释了为什么我们在主线程中这样new没有问题，子线程中如果不先Looper.prepare会抛出异常的原因，前面多次说了，因为ActivityThread会在初始化的时候创建自己的Looper。

#### 以上均来自@amurocrash的笔记，在此感谢。非常棒[Android开发艺术探索读书笔记（三）](http://blog.csdn.net/amurocrash/article/details/48858353)
***
### Android的线程和线程池
* `AsyncTask`封装了线程池和Handler。`HandlerThread`是一种具有消息循环的线程。在它的内部可以使用Handler.`IntentService` 是一个服务，系统对其进行封装可以更方便的执行后台任务，内部使用了`HandlerThread `和handler，当任务执行完成后会自动退出。
* `AsyncTask `有四个核心的方法：

  1:`onPreExecute`这个在主线程中执行，用于一些准备工作
  2:`doInBackground(Object[] params)`运行在线程池，在这个方法中可以通过`publishProgress`进行进度更新。
  3:`onProgressUpdate(Object[] values) `在主线程中，当进度改变时回去调用这个方法。
  4:`onPostExecute(Object o)`主线程中，用于处理返回结果
* 还有` onCancelled()`方法，主线程中调用，当任务被去取消的时候调用，这样就不会再去调用`onPostExecute `方法了。
* 使用`AsyncTask`的限制
  1：这个类只能在主线程中进行加载。
  2：这个类的对象只能在主线程中进行创建。
  3：`execute()`方法只能在主线程进行调用
  4：一个`AsyncTask `对象只能执行一次，也就是只能执行一次`execute()`方法，否则会报错。
  5：在1.6之前是串行的执行任务，1.6的时候采用了在线程池中并行的处理，但是在3.0开始，为了避免并发错误，又采用了一个新线程来串行执行任务。但是3.0以后的版本依然可以使用`executeOnExecutor()`来并行处理任务。但是不能在低版本上面使用。
* `HandlerThread`个人感觉就是封装了一下我们一起在子线程中的`Looper.prepare()`和`Looper.loop（）`,通过handler的构造`handler（looper）`来与子线程的messagequeue进行关联，也就是在handler的`handlermessage`方法中去处理耗时任务，还得去通过一个主线程的handler去更新UI。具体原理去看源码，我都看得懂😂。
* `IntentService`是一个抽象的特殊service，内部也是使用了`HandlerThread `和`Handler`。这个源码更少。。。😂不过很好使。
* 线程池的好处：
 1：重用线程，减少开销
 2：能控制线程池的最大并发度，避免线程阻塞
 3：能够对线程进行简单的管理
* `ThreadPoolExecutor`是线程池的真正实现。他常用的构造方法
```
ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) 
```
在Android中我只找到了两个构造函数，一个是上面的，一个是还有` ThreadFactory threadFactory,RejectedExecutionHandler handler`这两个参数的。没有找到主席说的，但是并不影响。主要就是这几个参数；

`corePoolSize `核心线程数，核心线程默认会一直存活

`maximumPoolSize `线程池的最大线程数，超过了会阻塞

`keepAliveTime `非核心线程闲置的超时时长，超过了就会回收

`unit `这个是时间单位，毫秒，分钟，秒等

`workQueue `任务队列，runnable对象会储存在这里

`threadFactory `为线程池提供创建新线程的功能

`handler `这个不常用

* `AsyncTask `内部的线程池配置：

 核心线程数是CPU核心数+1；
 
 最大线程数是CPU核心数*2+1；
 
 核心线程无超时机制，非核心线程在闲置时的超时为1秒；
 
 任务队列的容量128；
 
 ###线程池的分类
 
* `FixedThreadPool()`通过`Executors.newFixedThreadPool()`创建，只有核心线程且不会回收，可以更快的响应外界的请求。
* `CachedThreadPool()`数量不定，只有非核心线程。适合执行大量的耗时较少的任务
* `ScheduledThreadPool()`核心线程固定，非核心线程不固定，非核心线程闲置会立即回收，主要用来执行定时任务和具有固定周期的重复任务。
* `SingleThreadExecutor()`只有一个核心线程。所有任务都在同一个线程中顺序执行。好处是不需要处理线程间的同步问题。


#### Bitmap的加载和Cache
这个我就不写了，关于`Bitmap`就是内存缓存和硬盘缓存，使用`LruCache`和`DiskLruCache`之前就写过这个[我的GitHub](https://github.com/70kg/Android-Studio-Project/blob/master/app/src/main/java/com/com/mr_wrong/ImageLoaderWithCaches/ImageLoaderWithDoubleCaches.java)。不过我用的是`AsyncTask `，主席用的是线程池。我对线程池不怎么熟练。为什么不用`AsyncTask `还是之前说的兼容性问题。对应列表滑动的优化也是老生常谈了，还是早去使用`RecyclerView`吧。

### 综合技术
#### 使用`CrashHandler`来获取应用的crash信息
这个在项目上也就使用一次，还是看[主席写的吧](https://github.com/singwhatiwanna/android-art-res/tree/master/Chapter_13/CrashTest/src/com/ryg/crashtest)。我记得友盟等第三方的统计SDk都支持这个的，我也没怎么用过这些，不过看他们开源的都使用友盟的，集成倒是很方便。
#### 使用`multidex`来解决方法数越界
就是传说中的65536方法数越界，这个解决办法挺简单的，官方出品，我估计我暂时还用不到。😂
#### Android的动态加载技术
这个我就更用不上了。之前看过360开源的一个动态加载的库，主席好像也有个库，这种高大上的先稍微了解一下就可以了。当个科普文看了。
#### 反编译初步
这个也就用来看看别人的代码，用到了倒是很实用，但是混淆了你也没办法。

#### JNI和NDK编程
也是作为了解的部分，强行上手也没什么用
#### Android性能优化
这个详细的可以去看谷爹官方出的教程。都出三季了。YouTube上有视频，不过英语捉鸡没办法，貌似优酷有翻译，还是看博客吧。胡凯大大翻译的博客[地址](http://hukai.me/blog/categories/android/)



#### 最后的话
好了，主席的这本书算是看完了，一共写了18649个字。前前后后花了有两周的时间吧。中间还因为十一假期拖沓了几天。真的很有营养。但是这些都是理论，还是应该多去写代码去实践。话说我的meizi项目也是拖了有段时间了，thinking in java 说好的十一更新也没来得及。加油吧。

 
 
 
 
 
 
 
  

 
 
 
 
 
 
 
 
 
 


