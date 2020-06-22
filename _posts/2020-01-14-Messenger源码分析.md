---
layout:     post
title:      Messenger源码分析
subtitle:
date:       2020-01-14
author:     parting_soul
header-img: img/messenger源码分析.jpg
catalog: true
tags:
    - Android
    - IPC
---

### 一.  概述

​		前面我们介绍了Messenger的基本用法，相对于AIDL而言，Messenger的用法更简单，但是Messenger不具有并发处理任务的能力，这是两者主要的区别。但是Messenger同样是基于AIDL实现的，这边我们来讲讲它的实现原理。

### 二. 源码分析

#### 2.1 服务端

​		在使用Messenger时，我们会在服务端的Service中创建一个Messenger，通过Messenger获得一个IBinder对象，在onBind方法中返回。

```java
public class MessengerService extends Service {
    private Messenger mMessenger = new Messenger(new H());

    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }
	...
}
```

从创建Messenger开始，参数是一个Handler

```java
public final class Messenger implements Parcelable {
    private final IMessenger mTarget;

    public Messenger(Handler target) {
        mTarget = target.getIMessenger();
    }
    ...
}
```

​		可以看到，在构造方法中通过去获取IMessenger对象，我们来看看Handler代码，可以看到getIMessenger方法中若Messenger不存在，则创建一个Messenger实现类MessengerImpl对象，然后返回。

Handler.java

```java
final IMessenger getIMessenger() {
  synchronized (mQueue) {
    if (mMessenger != null) {
      return mMessenger;
    }
    mMessenger = new MessengerImpl();
    return mMessenger;
  }
}
```

​		我们来看下MessengerImpl类，原来IMessenger是一个aidl接口，该接口中定义了一个send方法，供客户端调用。send方法是服务端的具体逻辑实现，可以看到这里将客户端发送过来的消息交给了Handler，这也就是Messenger只能够串行处理客户端请求的原因。

```java
private final class MessengerImpl extends IMessenger.Stub {
  public void send(Message msg) {
    msg.sendingUid = Binder.getCallingUid();
    Handler.this.sendMessage(msg);
  }
}
```

IMessenger.aidl定义

```java
package android.os;

/** @hide */
oneway interface IMessenger {
    void send(in Message msg);
}
```

#### 2.2 客户端

​	在客户端，服务端绑定成功后需要将服务端返回的IBinder对象封装成Messenger对象。

```java
@Override
public void onServiceConnected(ComponentName name, IBinder service) {
    Log.d("service connected");
    isBound = true;
    mServiceMessenger = new Messenger(service);
}
```

我们来看这个形参为IBinder的构造方法，可以看到在构造方法中，通过生成的aidl类IMessenger.Stub把IBinder对象转化成了IMessenger对象，供客户端调用。

```java
public final class Messenger implements Parcelable {
    private final IMessenger mTarget;
 		
  	public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
    }
  	....
}
```

客户端通过封装的Messenger发送消息

```java
Message message = Message.obtain();
message.what = 1;
mServiceMessenger.send(message);
```

我们来看下Messenger的send方法，可以看到最终还是调用了IMessenger的send方法

```java
public void send(Message message) throws RemoteException {
    mTarget.send(message);
}
```

关于AIDL部分这边不再做分析，见[AIDL进阶](https://partingsoul.github.io/2020/01/13/AIDL%E8%BF%9B%E9%98%B6/)

#### 2.3 小结

![Messenger](http://img.partingsoul.cn/Messenger.png)

- Messenger内部实现还是通过AIDL，只是Messenger封装了AIDL
- Messenger的AIDL方法中定义了一个用于客户端向服务端发送消息的send方法
- 服务端send方法的具体实现是直接把客户端发送过来的消息发送给了Handler，交给Handler处理