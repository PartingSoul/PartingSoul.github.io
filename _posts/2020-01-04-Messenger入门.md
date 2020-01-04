---
layout:     post
title:      Messenger入门
subtitle:
date:       2020-01-04
author:     parting_soul
header-img: img/messenger入门.jpeg
catalog: true
tags:
    - Android
    - IPC
---

### 一. 概述

​	上篇文章介绍了AIDL的基本用法，AIDL主要用于进程间的通信，但是AIDL需要在服务端处理线程安全问题并且AIDL的使用方式比较复杂。

​	这里引出Messenger，Messenger同样可以用于进程间的通信，它的底层实现也是通过AIDL，但却是一个轻量级的实现方式，使用起来比AIDL简单。由于它一次只能处理一个请求，因此服务端不需要考虑线程安全问题。

### 二. 用法

Messenger是通过Handler和Message的方式进行进程间的通信。

使用Message传递的数据为arg1、arg2、what、Bundle。obj只能传递系统的Parcelable的对象，无法传递自定义的Parcelable对象。

例如现在需要将客户端的信息传送到服务端

#### 2.1 服务端代码编写

- 创建一个Service
- 创建一个Handler用于接收处理消息
- 创建一个Messenger用于通信

```java
public class MessengerService extends Service {
    private Messenger mMessenger = new Messenger(new H());

    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }

    static class H extends Handler {
        @Override
        public void handleMessage(@NonNull Message msg) {
            super.handleMessage(msg);
            Log.d("msg = " + msg.what);
        }
    }

}
```

#### 2.2 客户端代码编写

- 绑定远程服务，同时获取服务端的Messenger
- 使用服务端的Messenger将消息发送至服务端

```java
public class MessengerActivity extends AppCompatActivity {
    private boolean isBound;
    private Messenger mServiceMessenger;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.act_messenger);

        // 绑定服务
        Intent intent = new Intent();
        intent.setClassName("com.parting_soul.server", "com.parting_soul.server.MessengerService");
        bindService(intent, mServiceConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (isBound) {
            unbindService(mServiceConnection);
            isBound = false;
        }
    }

    public void onClick(View view) {
        if (!isBound) {
            return;
        }

        switch (view.getId()) {
            case R.id.bt_send_message:
                sendMessage();
                break;
            default:
                break;
        }
    }

    private void sendMessage() {
        try {
            Message message = Message.obtain();
            message.what = 123;
            mServiceMessenger.send(message);
            Log.d("service = ");
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    private ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.d("service connected");
            isBound = true;
            mServiceMessenger = new Messenger(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            isBound = false;
        }
    };
  
}
```

#### 2.3 服务端给客户端传送消息

如果在服务端接收客户端消息后，服务端需要给客户端应答，此时就需要利用Message的replyTo属性。

- 首先是需要创建处理服务端消息的Handler以及客户端对应的Messenger，服务端可以利用该Messenger给客户端传递消息
- 在客户端给服务端发送消息时，同时要在消息中添加自己的Messenger信息

客户端

```java
private Messenger mClientMessenger = new Messenger(new ClientHandler());

static class ClientHandler extends Handler {
  @Override
  public void handleMessage(@NonNull Message msg) {
    Log.d("接收到服务端的消息");
    switch (msg.what) {
      case 200:
        Bundle bundle = msg.getData();
        String result = bundle.getString("data");
        Log.d(result);
        break;
      default:
    }
  }
}
```

发送消息

```java
private void sendAndReceiveMessage() {
  try {
    Message message = Message.obtain();
    message.what = 2;
    //设置自己的Messenger
    message.replyTo = mClientMessenger;

    //传递序列化对象
    Bundle bundle = new Bundle();
    bundle.putParcelable("data", new Book("Android 书籍", "技术"));
    message.setData(bundle);

    mServiceMessenger.send(message);
  } catch (RemoteException e) {
    e.printStackTrace();
  }
}
```

服务端：接收消息后返回成功信息

```java
public void handleMessage(@NonNull Message msg) {
  String str = "what = " + msg.what + " 接收到消息";
  switch (msg.what) {
    case 1:
      break;
    case 2:
      Bundle bundle = msg.getData();
      //反序列化前需要设置ClassLoader
      bundle.setClassLoader(Book.class.getClassLoader());
      Book book = bundle.getParcelable("data");
      str += " " + book;
      
      //得到客户端的Messenger
      Messenger clientMessenger = msg.replyTo;
      replyToClient(clientMessenger);

      break;
    default:
  }
  Log.d(str);
}

//返回信息至客户端
private void replyToClient(Messenger clientMessenger) {
  try {
    Bundle bundle = new Bundle();
    bundle.putString("data", "接收成功");
    Message newMsg = Message.obtain();
    newMsg.what = 200;
    newMsg.setData(bundle);
    clientMessenger.send(newMsg);
  } catch (RemoteException e) {
    e.printStackTrace();
  }
}
```



