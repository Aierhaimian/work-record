# 进程间通信——Binder机制

## 前言

binder一词在英文中为粘合剂、粘结剂的意思。那么究竟Binder是如何做两个进程之间的粘合剂的呢？让我们深入Binder源码，去探究一下其中的奥秘。

通常，同一程序中的两个函数之间能够直接调用的根本原因在于它们处于相同的内存空间中，其虚拟地址的映射规则完全一致，那么它们的调用关系很简单。反之，两个不同的进程被映射到不同的内存空间中，是无法通过内存地址来直接相互访问的。那么，在Android中就采取了曲线救国的间接访问方法——Binder。

经典的操作系统的进程间通信机制有信号量、管道、Socket等。那么，为什么Android独宠Binder一人呢？我觉得的Binder必有过人之处。接下来我们按照一下脉络展开分析。

	1. Binder Driver		-> 		路由器
	2. Service Manager	 	-> 		DNS
	3. Binder Client 		-> 		客户端
	4. Binder Server 		-> 		服务端
	
在进行分析之前大家请记住下图中典型的TCP/IP访问过程：

![ ](/home/dzx/workRecord/images/tcpip.png  "典型的TCP/IP访问过程")

## 智能指针

智能指针在整个Android工程中使用非常广泛，特别是在Binder的源码实现中更是比比皆是。所以，在深入了解Binder机制之前有必要了解一下这一基础知识，具体内容放在智能指针专题，请移步那里学习后再回来继续。

## 进程间的数据传递载体——Parcel

进程间应该如何传递数据呢？如果是int型的数据，通过层层赋值，也许可以实现进程间传递，但是如果传递的对象是一个对象呢？我们都知道对象的传递是靠引用实现的，而引用实际上就是内存地址，而进程间是不同的虚地址空间，无法通过内存地址进程传递，那么这种形式的数据传递又应该怎么办呢？Android中担负这一重任的是Parcel。

Parcel是进程间传递数据的载体，用于承载希望通过IBinder发送的相关信息。就是将对象在进程A中占据内存的信息打包起来，然后寄送到进程B，由进程B在自己的内存空间复制这个对象。Parcel详细接口说明文档见：

	https://developer.android.com/reference/android/os/Parcel
	
下面对这些接口进行一个分类：

1  Parcel设置相关

	dataSize()：获取当前已经存储的数据大小
	setDataCapacity(int size)：设置Parcel的空间大小，显然存储的数据不能大于这个值。
	setDataPosition(int pos)：改变Parcel中读写位置，必须介于0和dataSize()之间。
	dataAvail()：当前Parcel的存储能力。
	dataCapacity()：当前Parcel的存储能力。
	dataPosition()：数据的当前位置值，有点类似于游标。
	dataSize()：当前Parcel所包含的数据大小。

2 Primitives

	writeByte(byte)：写入一个byte。
	readByte()：读取一个byte。
	writeDouble(double)：写入一个double。
	readDouble(double)：读取一个double。

3 Primitive Arrays

	writeBooleanArray(coolean[])：写入布尔数组。
	readBooleanArray(boolean[])：读取布尔数组。
	readBooean[]createBooleanArray()：读取并返回一个布尔数组。
	writeByteArray(byte[])：写入字节数组。
	writeByteArray(byte[], int, int)：和上面几个不同的是，这个函数最后面的两个参数分别表示数组中需要被写入的数据起点以及需要写入多少。
	readByteArray(byte[])：读取字节数组。
	byte[]createByteArray()：读取并返回一个数组。

4 Parcelables

遵循Parcelable协议的对象可以通过Parcel来存取，如开发人员经常用到的bundle就是继承自Parceable的。与这类对象相关的Parcel操作包括：

	readParcelable(ClassLoader)：读取并返回一个新的Parcelable对象。
	writeParcelableArray(T[], int)：写入Parcelable对象数组。
	readParcelableArray(ClassLoader)：读取并返回一个Parcelable对象数组。

5 Bundles

Bundle继承自Parcelable，是一种特殊的type-safe的容器。Bundle的最大特点就是采用键值对的方式存储数据，并在一定程度上优化了读取效率。

	writeBundle(Bundle)：将Bundle写入Parcel。
	readBundle()：读取并返回一个新的Bundle对象。
	readBundle(ClassLoader)：读取并返回一个新的Bundle对象，ClassLoader用于Bundle获取对应的Parcelable对象。

6 Active Objects

Parcel的拎一个强大武器就是卡伊读写Active Objects。什么是Active Objects呢？通常我们存入Parcel的是对象的内容，而Active Objects写入的则是它们的特殊标志引用。所以在从Parcel中读取这些对象时，大家看到的并不是重新创建的对象实例，而是原来那个被写入的实例。可以猜想到，能够以这种方式传输的对象不会很多，目前主要有两类。

i. Binder。Binder一方面是Android系统IPC通信的核心机制之一，另一方面也是一个对象。利用Parcel将Binder对象写入，读取时就能得到原始的Binder对象，或者是它的特殊代理实现（最终操作的还是原始Binder对象）。

	writeStrongBinder(IBinder)
	writeStrongInterface(IInterface)
	readStrongBinder()

ii. FileDescriptor。FileDescriptor是Linux中的文件描述符，可以通过Parcel的如下方法进行传递。

	writeFileDescriptor(FileDescriptor), readFileDescriptor()
	
因为传递后的对象仍会基于和原对象相同的文件流进行操作，因而可以认为是Active Objects的一种。

7 Untyped Containers

它用于读写标准的任意类型的java容器。

	writeArray(Object[])
	readArray(ClassLoader)
	writeList(List)
	readList(List, ClassLoader)
	...

接下来，我们看看Parcel内部是如何实现的。应用程序可以通过Parcel.obtain()接口来获取一个Parcel对象。

	frameworks/base/core/java/android/os/Parcel.java
	    /**
	     * Retrieve a new Parcel object from the pool.
	     */
	    public static Parcel obtain() {
		final Parcel[] pool = sOwnedPool;  //系统预先产生了一个Parcel池，大小为6
		synchronized (pool) {
		    Parcel p;
		    for (int i=0; i<POOL_SIZE; i++) {
		        p = pool[i];
		        if (p != null) {
		            pool[i] = null; //引用置为空，这样下次就知道这个Parcel已经被占用了
		            if (DEBUG_RECYCLE) {
		                p.mStack = new RuntimeException();
		            }
		            return p;
		        }
		    }
		}
		return new Parcel(0); // 如果Parcel池中已空，就新建一个
	    }
	   
新生成的Parcel有何奥妙之处，我们来看看它的构造函数。

	    private Parcel(long nativePtr) {
		if (DEBUG_RECYCLE) {
		    mStack = new RuntimeException();
		}
		//Log.i(TAG, "Initializing obj=0x" + Integer.toHexString(obj), mStack);
		init(nativePtr);
	    }
	    
这里只调用了init()函数，注意传入的nativePtr为0.

	    private void init(long nativePtr) {
		if (nativePtr != 0) {
		    mNativePtr = nativePtr;
		    mOwnsNativeParcelObject = false;
		} else {
		    mNativePtr = nativeCreate();  // 为本地层代码准备的指针
		    mOwnsNativeParcelObject = true;
		}
	    }
	    
Parcel的JNI层实现在frameworks/base/core/jni中，实际上Parcel.java只是一个简单的中介，最终所有类型的读写操作都是通过本地代码实现的。

	frameworks/base/core/jni/android_os_Parcel.cpp
	static jlong android_os_Parcel_create(JNIEnv* env, jclass clazz)
	{
	    Parcel* parcel = new Parcel();
	    return reinterpret_cast<jlong>(parcel);
	}
	
所以上面的mNativePtr变量实际上是本地层的一个Parcel(C++)对象。那么

- Parcel 中如何存储数据.
- 我们选择其中两个代表性的数据类型（string 和 Binder）来分析Parcel的处理流程，对应的接口分别是：writeString()/readString()和writeStrongBinder()和readStrongBinder()。

先来看看Parcel(c++)类的构造过程。

	frameworks/native/libs/binder/Parcel.cpp
	Parcel::Parcel()
	{
	    LOG_ALLOC("Parcel %p: constructing", this);
	    initState();
	}
	
	void Parcel::initState()
	{
	    LOG_ALLOC("Parcel %p: initState", this);
	    mError = NO_ERROR;
	    mData = 0;
	    mDataSize = 0;
	    mDataCapacity = 0;
	    mDataPos = 0;
	    ALOGV("initState Setting data size of %p to %zu", this, mDataSize);
	    ALOGV("initState Setting data pos of %p to %zu", this, mDataPos);
	    mObjects = NULL;
	    mObjectsSize = 0;
	    mObjectsCapacity = 0;
	    mNextObjectHint = 0;
	    mHasFds = false;
	    mFdsKnown = true;
	    mAllowFds = true;
	    mOwner = NULL;
	}

