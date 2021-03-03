---
title: Looper解读
date: 2021-03-02 17:30:46
tags: android
---

## 类图

![类图](../resource/handler相关/Looper.dotuml.png)


## prepare

```java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}

/**
 * 构造方法中创建MessageQueue，并引用当前线程。
 * /
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

`Looper.prepare()`方法只是为了初始化`ThreadLocal`。

之前了解过，`ThreadLocal`可以包证多线程访问共享变量的线程安全问题。他不像`synchronized`靠阻塞实现线程安全，而是通过对变量拷贝的方式，使每一个线程都操作自己的拷贝，实现线程安全，所以效率要优于`synchronized`。[详情看这里](https://www.jianshu.com/p/6fc3bba12f38)

由于在Android中每个线程都有个Looper对象，所以采用ThreadLocal来保存和获取当前线程的Looper对象。

## loop

只看核心：

```java
public static void loop() {
    final Looper me = myLooper();//其实就是从threadLocal中取出looper
    //省略判空

    me.mInLoop = true; //改变状态

    final MessageQueue queue = me.mQueue;//拿到MessageQueue

    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();
    final int thresholdOverride =
            SystemProperties.getInt("log.looper."
                    + Process.myUid() + "."
                    + Thread.currentThread().getName()
                    + ".slow", 0);

    boolean slowDeliveryDetected = false;

    //接下来就是启动个死循环，为什么不用while而用for呢？
    for (;;) {
        Message msg = queue.next(); // 从MessageQueue中取出Message
        
        try {
            msg.target.dispatchMessage(msg);//调用Handler的dispatchMessage
        } catch (Exception exception) {
            //...
        } finally {
            //...
        }

        //无论如何都会回收消息
        msg.recycleUnchecked();
    }
```
