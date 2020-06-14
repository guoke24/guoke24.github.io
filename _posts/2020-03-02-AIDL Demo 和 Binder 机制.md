---
layout:     post  
title:      "AIDL Demo 和 Binder 机制"  
subtitle:   "主要是 Demo，顺便看看 Binder 机制的皮毛"  
date:       2020-03-01 12:00:00  
author:     "GuoHao"  
header-img: "img/waittouse.jpg"  
catalog: true  
tags:  
    - Android  
    - 源码  
    - AIDL  
    - Binder  

---


# 概述

[AidlDemo 项目参考](https://github.com/guoke24/AidlDemo)
该 Github 项目有两个独立的项目，
一个是 AidlClient，代表客户端；
另一个是 AidlServer，代表服务端。  

## aidl 使用的三步曲：
- client，server 端都是采用相同的 aidl
- server 端，new 一个 Stub 实例，并实现 aidl 中定义的函数，然后在 onBind 函数中返回 IBinder 类型的该 Stub 实例。
- client 端，实现发起绑定逻辑，并在绑定完成的回调中拿到 IBinder 类型的参数，并通过 Stub.asInterfac 函数转变为代理类实例，用 aidl 接口依赖。


下面以 [AidlDemo 项目参考](https://github.com/guoke24/AidlDemo) 为例进行分析。



# 两端都用同一个 Aidl

Client 端的 aidl 路径：
/AidlDemo/AidlClient/app/src/main/aidl/com/guohao/aidl/server/IBookManager.aidl

Server 端的 aidl 路径：
/AidlDemo/AidlServer/app/src/main/aidl/com/guohao/aidl/server/IBookManager.aidl

aidl 源码：

```
// IBookManager.aidl

package com.guohao.aidl.server;
interface IBookManager {

    int add(int num1,int num2);

    int wrong_add(int num1,int num2);

    int more_fun();
}
```

<br>
定义好的 aidl 接口，编译后会生成 java 文件：{工程根目录}/app/build/generated/source/aidl/debug/com/guohao/aidl/server/IBookManager.java

源码如下：
有点长，先看 IBookManager.java 接口的结构：
- add 接口函数
- wrong_add 接口函数 
- more_fun 接口函数
- Stub 类
  - asBinder 函数
  - asInterface 函数，client 使用，把 server 的 IBinder 转成 Proxy
  - onTransact 函数，server 使用，接收 client 通信的回调
  - Proxy 类，client 使用
     - asBinder 函数
     - add 函数，向 Binder 驱动发送通信，继而转到 server 的 onTransact 函数
     - wrong_add 函数，同上
     - more_fun 函数，同上
  


具体源码：
```
// IBookManager.java

/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: /Users/guohao/AS_Proj/AidlDemo/AidlServer/app/src/main/aidl/com/guohao/aidl/server/IBookManager.aidl
 */
package com.guohao.aidl.server;

public interface IBookManager extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.guohao.aidl.server.IBookManager {
    
        // 用于告诉 Binder 驱动，向哪个进程发起通信
        private static final java.lang.String DESCRIPTOR = "com.guohao.aidl.server.IBookManager";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.guohao.aidl.server.IBookManager interface,
         * generating a proxy if needed.
         */
        public static com.guohao.aidl.server.IBookManager asInterface(android.os.IBinder obj) {
            ...
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.guohao.aidl.server.IBookManager))) {
                return ((com.guohao.aidl.server.IBookManager) iin);
            }
            return new com.guohao.aidl.server.IBookManager.Stub.Proxy(obj);
        }

        //
        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        // 在 server 端被调用，接收 client 通信的回调
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_add: {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    int _arg1;
                    _arg1 = data.readInt();
                    int _result = this.add(_arg0, _arg1);
                    reply.writeNoException();
                    reply.writeInt(_result);
                    return true;
                }
                case TRANSACTION_wrong_add: {
                    ...
                    int _result = this.wrong_add(_arg0, _arg1);
                    ...
                    return true;
                }
                case TRANSACTION_more_fun: {
                    data.enforceInterface(descriptor);
                    int _result = this.more_fun();
                    ...
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }

        // 在 client 端持有实例，通过 mRemote.transact 函数向 Binder 驱动发起跨进程通信
        private static class Proxy implements com.guohao.aidl.server.IBookManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public int add(int num1, int num2) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                int _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(num1);
                    _data.writeInt(num2);
                    mRemote.transact(Stub.TRANSACTION_add, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readInt();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            @Override
            public int wrong_add(int num1, int num2) throws android.os.RemoteException {
                ...
                try {
                    ...
                    mRemote.transact(Stub.TRANSACTION_wrong_add, _data, _reply, 0);
                    ...
                } finally {
                    ...
                }
                return _result;
            }

            @Override
            public int more_fun() throws android.os.RemoteException {
                ...
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_more_fun, _data, _reply, 0);
                    ...
                } finally {
                    ...
                }
                return _result;
            }
        }

        // mRemote.transact 函数时会使用的标志位，用于识别调用哪个函数
        static final int TRANSACTION_add = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_wrong_add = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
        static final int TRANSACTION_more_fun = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
    }

    public int add(int num1, int num2) throws android.os.RemoteException;

    public int wrong_add(int num1, int num2) throws android.os.RemoteException;

    public int more_fun() throws android.os.RemoteException;
}

```

# server端的实现

server 端的工作：
> new 一个 Stub 实例，并实现 aidl 中定义的函数，然后在 onBind 函数中返回 IBinder 类型的该 Stub 实例

源码：
```
// AidlService.java

public class AidlService extends Service {
    public AidlService() {
    }

    @Override
    public IBinder onBind(Intent intent) {
        // TODO: Return the communication channel to the service.
        
        // 返回接口 IBookManager.Stub 实例
        return aidlServer;
    }

    // 声明并实现 IBookManager 接口的 stub 类实例
    IBookManager.Stub aidlServer = new IBookManager.Stub(){

        @Override
        public int add(int num1, int num2) throws RemoteException {
            Log.d("guohao","add...num1 = " + num1 + " ,num2 = " + num2 );
            return ( num1 + num2 );
        }

        @Override
        public int wrong_add(int num1, int num2) throws RemoteException {
            Log.d("guohao","wrong_add...num1 = " + num1 + " ,num2 = " + num2 );
            return ( num1 + num2 );
        }

        @Override
        public int more_fun() throws RemoteException {
            Log.d("guohao","more_fun..." );
            return 0;
        }

    };
    // end 声明并实现 接口的stub对象
}
```

# client端的实现

client 端工作
> 实现发起绑定逻辑，并在绑定完成的回调中拿到 IBinder 类型的参数，并通过 Stub.asInterfac 函数转变为代理类实例，用 aidl 接口依赖。

源码：

```
// MainActivity.java

    private IBookManager mIBookManager;

    // 创建跟服务端的连接
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            // 获取远程服务Binder的代理：service
            Log.d("guohao","onServiceConnected");
            // service 转化为 本地的 mIBookManager 接口
            // asInterface 返回的是代理类 IBookManager.Stub.Proxy
            // 通过 IBookManager 接口依赖
            mIBookManager = IBookManager.Stub.asInterface(service);
            // 注意：本地的 IBookManager 接口，只定义了两个函数，
            // 而服务端的  IBookManager 接口，定义了三个函数，
            // 但是，这里不会因为函数不一致而报错
        }
        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.d("guohao","onServiceDisconnected");
        }
    };
    // end 创建跟服务端的连接

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    // 发起绑定服务
    private boolean bindBookService(){
        Intent intent = new Intent();
        // 指定 action，要绑定的服务类，在服务端的清单文件里定义
        intent.setAction("com.guohao.aidl.server.aidlService");
        // 指定 包名，服务端的包名
        intent.setPackage("com.guohao.aidlserver");
        startService(intent);
        return bindService(intent, mConnection, BIND_AUTO_CREATE);
    }
    // end 发起绑定服务

    public void test(View v){
        Log.d("guohao","test2");
        // 调用 绑定服务
        boolean b = bindBookService();
        if(b){
            Toast.makeText(this,"bind succ",Toast.LENGTH_SHORT).show();
        }else{
            Toast.makeText(this,"bind fail",Toast.LENGTH_SHORT).show();
        }
    }

    public void test2(View v){
        Log.d("guohao","test2");
        if(mIBookManager == null) {
            Log.d("guohao","未绑定服务");
            Toast.makeText(this,"未绑定服务" ,Toast.LENGTH_SHORT).show();
            return;
        }
        try {
            // 发起 远程调用
            int res = mIBookManager.add(1,1);
            Log.d("guohao","res = " + res);
            Toast.makeText(this,"mIBookManager.add(1,1) = " + res,Toast.LENGTH_SHORT).show();

        } catch (RemoteException e) {
            Log.d("guohao","e = " + e.toString());
            e.printStackTrace();
        }

    }
```

# 时序图小结
简单的绑定过程：  

![](/img/简单的绑定过程.png)


<br>



其中 bindService 函数的 [老罗的源码分析](https://blog.csdn.net/luoshengyang/article/details/6745181)，[另一篇源码分析（图清楚）](https://www.cnblogs.com/zhchoutai/p/8681312.html)

小结：
conn 在 ServiceDispatcher 实例中，
ServiceDispatcher 实例在 InnerConnection 实例中，
AMS 中持有 InnerConnection 实例，
在 AMS 中通过 InnerConnection - ServiceDispatcher - RunConnection.run - conn.onServiceConnected 层层调用，完成绑定通知。

<br>
跨进程通信过程：  

![](/img/跨进程通信过程.png)



# 测试了接口不一致，不会报错
* server 端的 IBookManager接口，函数声明为：
```
package com.guohao.aidl.server;
interface IBookManager {

    int add(int num1,int num2);

    int wrong_add(int num1,int num2);

    int more_fun();
}
```

* client 端的 IBookManager接口，函数声明为：
```

interface IBookManager {

    int add(int num1,int num2);

    // 服务端定义了该函数会多一个参数 wrong_add(int num1,int num2)
    int wrong_add(int num1);
}
```

* 结果：绑定服务不会报错，调用 wrong_add 函数也不会报错，缺失的参数 num2 默认为 0