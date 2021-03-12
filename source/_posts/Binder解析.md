---
title: Binder解析
date: 2021-03-12 11:06:25
tags: Android binder
---

## AIDL

### 定义

AIDL的全称是：Android接口定义语言(即 Android Interface Definition Language)，他定义的只不过是一套模板，实际起作用的是AS通过解释这个模板自动生成的`android.os.IInterface`的子类接口。作用是依靠`Service`中转，通过Binder机制来做IPC。

### 数据类型

AIDL支持的数据类型：

- byte
- char
- short
- int
- long
- float
- double
- boolean
- Parcelable
- List
- Map

### 定向Tag

```java
interface BookController {

    List<Book> getBookList();

    void addBookInOut(inout Book book);

    void addBookIn(in Book book);

    void addBookOut(out Book book);

}
```

- in 数据只能由客户端流向服务端
- out 数据只能由服务端流向客户端
- inout 数据可在服务端与客户端之间双向流通

如果AIDL方法接口的参数值类型是：基本数据类型、String、CharSequence或者其他AIDL文件定义的方法接口，那么这些参数值的定向 Tag 默认是且只能是 in，所以除了这些类型外，其他参数值都需要明确标注使用哪种定向Tag。

>引自https://www.jianshu.com/p/29999c1a93cd

### 源码

```java
//包名与项目包名一致
package com.vee.aidltest;

/**
*   自动生成的接口
**/
public interface IMyAidlInterface extends android.os.IInterface{

    /**
    *    我们自己声明的业务函数
    **/
    public java.lang.String getName() throws android.os.RemoteException;
  
    
    /**
    * 本接口的默认实现，其中业务方法`getName`返回null;
    **/
    public static class Default implements com.vee.aidltest.IMyAidlInterface{

        @Override public java.lang.String getName() throws android.os.RemoteException
        {
        return null;
        }
        @Override
        public android.os.IBinder asBinder() {
        return null;
        }
    }


    /**
    *   静态内部类，继承自`android.os.Binder.Stub`类型，也遵循AIDL接口
    **/
  public static abstract class Stub extends android.os.Binder implements com.vee.aidltest.IMyAidlInterface
  {

    private static final java.lang.String DESCRIPTOR = "com.vee.aidltest.IMyAidlInterface";
    
    /**
    *   构造方法中调用对象方法。
    **/
    public Stub(){
      this.attachInterface(this, DESCRIPTOR);
    }
    /**
     *  把IBinder对象转换成AIDL接口，如果需要的话会生成代理。
     */
    public static com.vee.aidltest.IMyAidlInterface asInterface(android.os.IBinder obj){
      if ((obj==null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof com.vee.aidltest.IMyAidlInterface))) {
        return ((com.vee.aidltest.IMyAidlInterface)iin);
      }
      return new com.vee.aidltest.IMyAidlInterface.Stub.Proxy(obj);
    }

    @Override public android.os.IBinder asBinder(){
      return this;
    }

    @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException{
      java.lang.String descriptor = DESCRIPTOR;
      
      switch (code){
        case INTERFACE_TRANSACTION:
        {
          reply.writeString(descriptor);
          return true;
        }
        case TRANSACTION_getName:
        {
          data.enforceInterface(descriptor);
          java.lang.String _result = this.getName();
          reply.writeNoException();
          reply.writeString(_result);
          return true;
        }
        default:
        {
          return super.onTransact(code, data, reply, flags);
        }
      }
    }

    /**
    *   自动生成的静态代理
    **/
    private static class Proxy implements com.vee.aidltest.IMyAidlInterface{
      private android.os.IBinder mRemote;
      Proxy(android.os.IBinder remote){
        mRemote = remote;
      }
      @Override public android.os.IBinder asBinder(){
        return mRemote;
      }
      public java.lang.String getInterfaceDescriptor(){
        return DESCRIPTOR;
      }

      @Override public java.lang.String getName() throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        java.lang.String _result;
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          boolean _status = mRemote.transact(Stub.TRANSACTION_getName, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            return getDefaultImpl().getName();
          }
          _reply.readException();
          _result = _reply.readString();
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
        return _result;
      }
      public static com.vee.aidltest.IMyAidlInterface sDefaultImpl;
    }
    
    static final int TRANSACTION_getName = (android.os.IBinderFIRST_CALL_TRANSACTION + 0);
    
    public static boolean setDefaultImpl(com.vee.aidltest.IMyAidlInterface impl) {
      // Only one user of this interface can use this function
      // at a time. This is a heuristic to detect if two different
      // users in the same process use this function.
      if (Stub.Proxy.sDefaultImpl != null) {
        throw new IllegalStateException("setDefaultImpl() called twice");
      }
      if (impl != null) {
        Stub.Proxy.sDefaultImpl = impl;
        return true;
      }
      return false;
    }
    public static com.vee.aidltest.IMyAidlInterface getDefaultImpl() {
      return Stub.Proxy.sDefaultImpl;
    }
  }
  
```