---
title: Android Binder 浅析
date: 2016-07-09 23:33:41
tags: Android
---
##开头废话
先说一下为什么要写这篇博客吧，起源于我对主席的书进行二周目的时候，看到了第二章IPC，然后去试着手写了一遍AIDL,再然后就有点困惑这里面是啥玩意，再然后就发现这里面的坑有点大了。。从`AIDL`到`Binder`，然后看到了Android系统里面那么多的binder使用，又去翻了一点`activity`的启动，从中又看到了`Binder`在进程间通讯的影子，从`AMS`到`activityThread`的通信等等。我觉得是很有必要写一些我的认识，可能还是有些云里雾里的，就当了解吧。<!--more-->

###从AIDL开始
先说一下平时写AIDL都是怎么写的，用主席的那个例子

 1. 定义一个可序列化的实体类
这个就不多说了，这里选择实现`Parcelable`接口来进行序列化，

```
package com.demos.aidl;
public class Book implements Parcelable {
    public int bookId;
    public String bookName;
    ....
    }
```
这个类有两个字段，明白意思就好了。

2.定义`Book`的aidl文件
虽然是在同一个包里面，但是还是要去声明一个这样的文件

```
package com.demos.aidl;
parcelable Book;
```
 3.定义AIDL接口
 
```
package com.demos.aidl;
import com.demos.aidl.Book;
interface IBookManager {
     List<Book> getBookList();
     void addBook(in Book book);
}
```
这里注意`import`Book进来，虽然是一个包里面的。

 4.build一下，然后在`build/generated/source/aidl/debug/com.demos.aidl`x里面就有了个`IBookManager`这么一个JAVA文件了。这就是系统帮助我们生成的。
 5.使用
 
 在`client`端
```
  private ServiceConnection mConnection = new ServiceConnection() {
        public void onServiceConnected(ComponentName className, IBinder service) {
            IBookManager bookManager = IBookManager.Stub.asInterface(service);
             List<Book> list = bookManager.getBookList();
             .....
            }
  Intent intent = new Intent(this, BookManagerService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
```
然后在`Server`端

```
private Binder mBinder = new IBookManager.Stub() {
        @Override
        public List<Book> getBookList() throws RemoteException {
            return mBookList;
        }
        @Override
        public void addBook(Book book) throws RemoteException {
            mBookList.add(book);
        }
    }
```
这样就基本是一个很简单的进程间通信了。前面是系统给我们自动生成的一个`IBookManager`接口，这个接口里面都是些什么东西呢。进去看看的话，刚开始看不出什么东西，(因为它的排版很烂啊)，下面来排个版简单的分析一下这东西。

###分析aidl生成接口
首先它是一个接口`public interface IBookManager extends android.os.IInterface`，要去进行进程间通信，都要去遵循`IInterface`这个接口，然后里面还有一个很重要的类`public static abstract class Stub extends android.os.Binder implements com.demos.aidl.IBookManager`这个内部类继承自`Binder`还实现了我们定义的aidl接口。回过头看在客户端是怎么获得`IBookManager`的？`IBookManager.Stub.asInterface(service);`下面就是实现的代码：

```
public static com.demos.aidl.IBookManager asInterface(android.os.IBinder obj)
{
    if ((obj==null)) {
        return null;
    }
    android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    if (((iin!=null)&&(iin instanceof com.demos.aidl.IBookManager))) {
        //如果客户端和服务端在同一个进程，就直接返回服务端的IBookManager，不用跨进程调用了
        return ((com.demos.aidl.IBookManager)iin);
    }   //否则返回的是一个代理类
    return new com.demos.aidl.IBookManager.Stub.Proxy(obj);
}
```
下面去看看这个代理类

```
//这个是运行在客户端进程
private static class Proxy implements com.demos.aidl.IBookManager
{   //远程的IBinder对象
    private android.os.IBinder mRemote;
    Proxy(android.os.IBinder remote){
        mRemote = remote;
    }
   .....
@Override public java.util.List<com.demos.aidl.Book> getBookList() throws android.os.RemoteException
{
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    java.util.List<com.demos.aidl.Book> _result;
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        //这个是把参数等信息传递给远程的IBinder
        mRemote.transact(Stub.TRANSACTION_getBookList,_data, _reply, 0);
        _reply.readException();
        _result = _reply.createTypedArrayList(com.demos.aidl.Book.CREATOR);
    }
    finally {
    _reply.recycle();
    _data.recycle();
    }
return _result;
}
.....
}
```
其实这个东西什么正事没干，就是把参数什么的传递给远程真正干活的，真正干活的在哪？还是在前面的那个`Stub`类里面，有一个`onTransact`方法，参数和上面的` mRemote.transact`方法的参数相对应，看一下吧

```
//这个是运行在服务端进程
@Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
{   //根据code来进行判断执行哪个方法
    switch (code){
        case INTERFACE_TRANSACTION:{
            reply.writeString(DESCRIPTOR);
            return true;
    }
        case TRANSACTION_getBookList:{
            data.enforceInterface(DESCRIPTOR);
            //调用了getBookList方法，这个方法是要自己在service里面去实现的
            java.util.List<com.demos.aidl.Book> _result =               this.getBookList();
            reply.writeNoException();
            reply.writeTypedList(_result);
            return true;
    }
    ....
    return super.onTransact(code, data, reply, flags);
}
```
这样基本上就大概理清楚了，手写`AIDL`就不写了，写出来基本上是一样的。

##概述多进程
在Android中，打开一个APP可以认为是开启了一个进程，当然四大组件也可以再开新的进程，不同的进程之间是不同的虚拟机，所以可以认识他们互相不知道对方的存在，也就没法去直接的交流。那么如果想要跨进程通信，其实有很多的办法，比如管道，System V，Socket等，那么Android为什么还要去自己搞一个binder呢，既然自己费那么大劲搞了一个，肯定有无可比拟的优势，这里我只知道由于性能和安全的考虑，它就选择了binder😂。
###binder基本模型
既然两个进程之间无法直接通信，那么怎么办。在Linux中，内核是可以获取两个进程的全部信息的，那么就好办了，把进程a想说的话告诉内核，内核再告诉b就好了。。。其实实现起来还是很复杂的。好吧，正经点说，首先系统有一个`ServiceManager`的进程，简称SM,这个是负责管理`Service`的，当一个`Server`建立的时候，会去和SM通信，在SM里面进行注册，比如我是张三，我的地址是0x12138。当`Client`想要和`Server`进行通信的时候，先去询问SM,我怎么联系张三，SM告诉它这个号码，然后就去联系了。其实还是有点不恰当，，大概就这么理解吧。这里注意到，SM其实也是一个进程，一个`Server`想要和它进行通信就涉及到了跨进程通信的问题，这就成了鸡-蛋的问题，其实这两者之间也是使用了`Binder`,不过系统给我们预先造了一个"鸡",和普通的跨进程通信还不太一样，这里理解就好。
###Binder跨进程原理
这里还是简单的说，因为我也不是特别的理解。就拿上面举的那个book的例子开始说，可以看到`Stub`是继承自`Binder`并且实现了`IBookManager`接口，而`Binder`是实现了`IBinder`接口，其实在`Binder`里面有一个内部类`BinderProxy`，它也是实现了`IBinder`接口。这样在来看在`Client`里面的使用，` public void onServiceConnected(ComponentName className, IBinder service)`这里的`service`是`sub`呢还是`BinderProxy`呢，在前面说了，如果是在同一个进程，就返回`Server`里面的`Binder`本体，否在就返回一个`BinderProxy`对象，在`asInterface(android.os.IBinder obj)`这个`obj`如果是跨进程的，肯定是一个`BinderProxy`对象了，然后就去调用这个对象的`transact`方法，把需要调用方法的code，数据，返回值，还有一个是不是双向RPC的flag位传进去，我们看一下在`BinderProxy`里面，这个方法是怎么实现的

```
 public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
        return transactNative(code, data, reply, flags);
    }
```
可以看到是调用了`Native`的方法，后面就是一些cpp的东西，我也看不太懂，也就是进去了`Binder`驱动里面去折腾东西，然后`Binder`驱动把那些参数什么的给`Server`的`onTransact`方法，然后在根据code去调用`Server`中重写的`IBookManager`中的方法。然后在`onTransact`方法中最后再把写好的返回参数在返回给`Binder`驱动，可以在最后看到`return super.onTransact(code, data, reply, flags);`，再然后就是`Binder`驱动把结果告诉等待的`Client`端。这样就基本完成了一次跨进程通信。

这里稍微总结一下，可以看到，无论是`sub`还是`BinderProxy`都是实现了`IBinder`接口，以至于客户端根本无法分辨出是不是在进行跨进程通信，也就是说`Client`看起来就和同进程的`Server`进行通信一样，只需要轻轻松松的调用`getBookList`就能获得想要的结果，也不用去在乎`Server`是在什么地方，底层了一切`Binder`驱动都给我们封装好了。还有就是在跨进程通信的时候，并没有把方法传过去，仅仅是传递了一个方法的code,然后在`Server`中根据这个code来调用对应的方法.

###在Android中的提现
在Android的源码中，使用`Binder`来进行进程间通讯也是很常见的，来看一个关于启动`activity`的东西

```
class ActivityManagerProxy implements IActivityManager{}

public abstract class ActivityManagerNative extends Binder implements IActivityManager{}

public final class ActivityManagerService extends ActivityManagerNative{}
```
这个三个类的继承实现关系是不是有点眼熟，对，就和前面写的那个book例子一样的，`ActivityManagerProxy`就相当于那个内部类`Proxy`，`ActivityManagerNative`就相当于里面的`Stub`，而实现类`ActivityManagerService`简称AMS就是`Server`了。在细说一下，在执行`startActivity`的时候，会到`mInstrumentation.execStartActivity()`，就是凯子哥说的那个老板娘😂，然后就是`ActivityManagerNative.getDefault().startActivity`，这里`getDefault`返回一个`IActivityManager`对象，就是使用`asInterface(IBinder obj)`获得的，是不是更熟悉了，就是前面判断是不是同一个进程的地方，这里肯定是跨进程了，返回的是一个`ActivityManagerProxy`对象，然后在`ActivityManagerProxy`里面有`startActivity`方法，里面有这么一句` mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);`又熟悉了吧。好吧，就说到这里了。

##结尾
以上只是我自己了解的浅见，肯定有不合理的地方，也只是我自己的理解，以后还会继续学习这方面的东西，毕竟现在也只是一知半解的。

参考：

[《Binder学习指南》](http://weishu.me/2016/01/12/binder-index-for-newer/)

[【凯子哥带你学Framework】Activity启动过程全解析](http://blog.csdn.net/zhaokaiqiang1992/article/details/49428287)

[Android Bander设计与实现 - 设计篇](http://blog.csdn.net/universus/article/details/6211589)  没看太明白😂

[ Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)  老罗的也是经典

[还有我的手写AIDL😂](https://github.com/70kg/Demos/tree/master/app/src/main/java/com/demos/aidl)