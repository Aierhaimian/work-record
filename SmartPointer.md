# 智能指针

## 前言

Java和C/C++的一个重大区别就是没有“指针”的概念。这并不代表Java不需要使用指针，而是这个“超级武器”被隐藏起来了。“水能载舟，亦能覆舟”，如果善用指针，它将给你的项目开发带来极大的便利，但是一旦出现问题，它将带来巨大的灾难。在C/C++项目中常见的指针问题可以归纳为：

- 指针没有初始化
- new了对象后没有delete
- 野指针

既然有以上问题，如果让我们自己来设计智能指针，应该如何设计一个能有效防止以上几个致命弱点的智能指针方案呢？

第一个没有初始化的问题很好解决，只要让智能指针在创建时置位null即可。
第二个问题是实现new和delete的配套。换句话说，new了一个对象就需要在某个地方及时delete它。既然是智能指针，那就意味着它应该是一个默默奉献的角色，尽可能自动地帮助程序员排忧解难，而不是所有的事情都需要程序员手工完成。

那么问题来了，要让智能指针自动判断是否需要回收一块内存空间，应该如何设计？

- SmartPointer 是个类

首先能想到的是，SmartPointer要能记录对象的内存地址，它的内部应该有一个指针变量指向对象。如：

	class SmartPointer{
		private:
			void *m_ptr; //指向object对象
	};
	
- SmartPointer 是个模板类

智能指针并不是针对某个特定类型的对象设计的，因而一定是模板类。

	template <typename T>
	class SmartPointer{
		private:
			T *m_ptr; //指向object对象
	};
	
- SmartPointer 的构造函数

根据第一个问题的解决思路，智能指针的构造函数应该将m_ptr置空。

	template <typename T>
	class SmartPointer{
		inline SmartPointer()
			: m_ptr(nullptr) {}
		private:
			T *m_ptr; //指向object对象
	};
	
- 引用计数
这是关键的问题点，智能指针怎么知道应该在什么时候释放一个内存对象呢？答案就是“当不需要的时候”释放，那究竟什么情况下才是不需要的时候呢？假设有一个指针指向这个object，那就必然代表后者是“被需要”的，直到这个指针解除了与内存对象的关系，我们就可以认为这个内存对象已经“不被需要”。

如果有两个以上的指针同时使用内存对象呢？只要有一个计数器记录着该内存对象“被需要”的个数即可。当这个计数器递减到零时，就说明该内存对象应该“寿终正寝”了。不过，又有一个新问题接踵而至，那就是由谁来管理这个计数器。

1. 计数器由智能指针拥有

大家应该都能想明白这种方案是不科学的。。。

2. 计数器由object自身持有

我们可以为有计数器需求的内存对象实现一个统一的“父类”——这样只要object继承于它就具备了计数的功能。

	template<class T>
	class LightRefBase{
		public:
			inline LightRefBase()
				: mCount(0) {}
			inline void incStrong() const { /*增加引用计数*/
				android_atomic_inc(&mCount);
			}
			inline void decStrong() const {/*减小引用计数*/
				if (android_atomic_dec(mCount) ==  1) {
					delete static_cast<const *T>(this); /*删除内存对象*/
				}
			}
		protected:
			inline ~LightRefBase() {}
			
		private:
			mutable volatile int32_t mCouont; /*引用计数*/
	};
	
以上代码段中的LightRefBase类主要提供了两个方法，即incStrong和decStrong，分别用于增加和减少引用计数值，而且如果当前已经没有人引用内存对象（计数值为0），他还需要自动释放自己。
那么，incstrong和decstrong这两个函数在什么情况下会被调用呢？既然是引用计数，当然是在“被引用时”，所以这个工作应该由SmartPointer完成。

	SmartPointer<TYPE>	smartP = new oject;
	
当一个智能指针引用了object时，其父类中的incstrong就会被调用。这也意味着SmartPointer必须要重载它的“=”运算符（多数情况下只重载“=”是不够的，应视具体情况需求而定）。

	template <typename T>
	class SmartPointer{
		inline SmartPointer()
			: m_ptr(nullptr) {}
		~wp();
		SmartPointer& operator = (T *other); //重载运算符
		
		private:
			T *m_ptr; //指向object对象
	};
	
	template<typename T>
	SmartPointer<T>& SmartPointer<T>::operator = (T *other)
	{
		if(other != null)
		{
			m_ptr = other; //指向内存对象
			other -> incstrong(); //主动增加计数值
		}
		return *this;
	}

当SmartPointer析构时，也应该及时调用decStrong来释放引用。

	template<typename T>
	wp<T>::~wp()
	{
		if (m_ptr)
			m_ptr -> decstrong();
	}
	
解决了第二个问题，实际上第三个问题也就迎刃而解了。

有了这些基础知识后，我们来分析下Android中的智能指针实现。

## 强指针sp

这里sp并不是SmartPointer的缩写，而是StrongPointer的缩写，看了题目你应该能想到吧，对应的还有wp，就是weakPointer的缩写，这一切让人猝不及防2333333
Android源码中对应实现在./system/core/include/utils/StrongPointer.h

	template<typename T>
	class sp {
	public:
	    inline sp() : m_ptr(0) { }

	    sp(T* other);
	    sp(const sp<T>& other);
	    template<typename U> sp(U* other);
	    template<typename U> sp(const sp<U>& other);

	    ~sp();

	    // Assignment

	    sp& operator = (T* other);
	    sp& operator = (const sp<T>& other);

	    template<typename U> sp& operator = (const sp<U>& other);
	    template<typename U> sp& operator = (U* other);

	    //! Special optimization for use by ProcessState (and nobody else).
	    void force_set(T* other);

	    // Reset

	    void clear();

	    // Accessors

	    inline  T&      operator* () const  { return *m_ptr; }
	    inline  T*      operator-> () const { return m_ptr;  }
	    inline  T*      get() const         { return m_ptr; }

	    // Operators

	    COMPARE(==)
	    COMPARE(!=)
	    COMPARE(>)
	    COMPARE(<)
	    COMPARE(<=)
	    COMPARE(>=)

	private:
	    template<typename Y> friend class sp;
	    template<typename Y> friend class wp;
	    void set_pointer(T* ptr);
	    T* m_ptr;
	};
	
可以看到sp与之前SmartPointer类的实现基本上是一致的。比如“=”重载实现为：

	template<typename T>
	sp<T>& sp<T>::operator =(T* other) {
	    if (other)
		other->incStrong(this);
	    if (m_ptr)
		m_ptr->decStrong(this);
	    m_ptr = other;
	    return *this;
	}
上面这段代码考虑了对一个智能指针重复赋值的情况。即当m_ptr不为空时，先要撤销它之前指向的内存对象，然后才能赋予其新值。另外，为sp分配一个内存对象并不一定要通过操作运算符，它的赋值函数也是可以的。

	template<typename T>
	sp<T>::sp(T* other)
		: m_ptr(other) {
	    if (other)
		other->incStrong(this); // 因为是构造函数，不用担心m_ptr之前已经赋过值
	}

析构函数也与预想的一致。

	template<typename T>
	sp<T>::~sp() {
	    if (m_ptr)
		m_ptr->decStrong(this);
	}
总的来说，强指针的实现与之前基本一致，这里不再赘述。

## 弱指针wp

既然有了强指针，那么为什么又要设计弱指针呢？我们想象这样一个场景：父对象parent指向子对象child，然后子对象又指向父对象，这就存在了循环引用的现象。

	struct CDad
	{
		CChild *myChild;
	};
	
	struct CChild
	{
		CDad *myChild;
	};
那么，它们很可能会产生对象间相互引用的关系。如果不考虑智能指针，这样的现象也不会导致什么问题，但是在智能指针的场景下，会有什么问题呢？
由于对象间的相互引用，内存回收时发现双方都是“被需要”的状态，出现了死锁，不能释放，从而引发不良后果。而解决这个矛盾的有效方法是采用“弱引用”。

CDad使用强指针来引用CChild，而CChild只使用弱引用来指向父类。双方规定当强引用计数为0时，不论弱引用是否为0都可以delete自己（Android系统中这个规则可以调整），这样只要有一方得到了释放，就可以成功避免死锁。那会不会引发野指针的问题呢？比如CDas因为强指针引用计数为0而被delete，但此时CChild还持有父类的弱引用，显然如果CChild此时用这个指针来访问CDad将引发致命的问题。因此，我们特别规定：

**弱指针必须先升级为强指针，才能访问它所指向的目标对象。**

源码路径：system/core/include/utils/RefBase.h

	template <typename T>
	class wp
	{
	public:
	    typedef typename RefBase::weakref_type weakref_type;

	    inline wp() : m_ptr(0) { }

	    wp(T* other);
	    wp(const wp<T>& other);
	    wp(const sp<T>& other);
	    template<typename U> wp(U* other);
	    template<typename U> wp(const sp<U>& other);
	    template<typename U> wp(const wp<U>& other);

	    ~wp();

	    // Assignment

	    wp& operator = (T* other);
	    wp& operator = (const wp<T>& other);
	    wp& operator = (const sp<T>& other);

	    template<typename U> wp& operator = (U* other);
	    template<typename U> wp& operator = (const wp<U>& other);
	    template<typename U> wp& operator = (const sp<U>& other);

	    void set_object_and_refs(T* other, weakref_type* refs);

	    // promotion to sp

	    sp<T> promote() const;

	    // Reset

	    void clear();

	    // Accessors

	    inline  weakref_type* get_refs() const { return m_refs; }

	    inline  T* unsafe_get() const { return m_ptr; }

	    // Operators

	    COMPARE_WEAK(==)
	    COMPARE_WEAK(!=)
	    COMPARE_WEAK(>)
	    COMPARE_WEAK(<)
	    COMPARE_WEAK(<=)
	    COMPARE_WEAK(>=)

	    inline bool operator == (const wp<T>& o) const {
		return (m_ptr == o.m_ptr) && (m_refs == o.m_refs);
	    }
	    template<typename U>
	    inline bool operator == (const wp<U>& o) const {
		return m_ptr == o.m_ptr;
	    }

	    inline bool operator > (const wp<T>& o) const {
		return (m_ptr == o.m_ptr) ? (m_refs > o.m_refs) : (m_ptr > o.m_ptr);
	    }
	    template<typename U>
	    inline bool operator > (const wp<U>& o) const {
		return (m_ptr == o.m_ptr) ? (m_refs > o.m_refs) : (m_ptr > o.m_ptr);
	    }

	    inline bool operator < (const wp<T>& o) const {
		return (m_ptr == o.m_ptr) ? (m_refs < o.m_refs) : (m_ptr < o.m_ptr);
	    }
	    template<typename U>
	    inline bool operator < (const wp<U>& o) const {
		return (m_ptr == o.m_ptr) ? (m_refs < o.m_refs) : (m_ptr < o.m_ptr);
	    }
		                 inline bool operator != (const wp<T>& o) const { return m_refs != o.m_refs; }
	    template<typename U> inline bool operator != (const wp<U>& o) const { return !operator == (o); }
		                 inline bool operator <= (const wp<T>& o) const { return !operator > (o); }
	    template<typename U> inline bool operator <= (const wp<U>& o) const { return !operator > (o); }
		                 inline bool operator >= (const wp<T>& o) const { return !operator < (o); }
	    template<typename U> inline bool operator >= (const wp<U>& o) const { return !operator < (o); }

	private:
	    template<typename Y> friend class sp;
	    template<typename Y> friend class wp;

	    T*              m_ptr;
	    weakref_type*   m_refs;
	};

和sp 相比，wp 在类定义上有如下重要区别：

- 除了指向目标对象的m_ptr外，wp另外有一个m_refs指针，类型为weakref_type。
- 没有重载->、*等运算符。
- 有一个prmote方法来将wp提升为sp。
- 目标对象的父类不是LightRefBase，而是RefBase。


		template<typename T>
		wp<T>::wp(T* other)
		    : m_ptr(other)
		{
		    if (other) m_refs = other->createWeak(this);
		}
由构造函数可见wp 并没有直接增加目标对象的引用计数值，而是调用了createWeak方法。这个函数属于RefBase类。

		class RefBase
		{
		public:
			    void            incStrong(const void* id) const;
			    void            decStrong(const void* id) const;

			    void            forceIncStrong(const void* id) const;

			    //! DEBUGGING ONLY: Get current strong ref count.
			    int32_t         getStrongCount() const;

		    class weakref_type
		    {
		    public:
			RefBase*            refBase() const;

			void                incWeak(const void* id);
			void                decWeak(const void* id);

			// acquires a strong reference if there is already one.
			bool                attemptIncStrong(const void* id);

			// acquires a weak reference if there is already one.
			// This is not always safe. see ProcessState.cpp and BpBinder.cpp
			// for proper use.
			bool                attemptIncWeak(const void* id);

			//! DEBUGGING ONLY: Get current weak ref count.
			int32_t             getWeakCount() const;

			//! DEBUGGING ONLY: Print references held on object.
			void                printRefs() const;

			//! DEBUGGING ONLY: Enable tracking for this object.
			// enable -- enable/disable tracking
			// retain -- when tracking is enable, if true, then we save a stack trace
			//           for each reference and dereference; when retain == false, we
			//           match up references and dereferences and keep only the 
			//           outstanding ones.

			void                trackMe(bool enable, bool retain);
		    };

			    weakref_type*   createWeak(const void* id) const;

			    weakref_type*   getWeakRefs() const;

			    //! DEBUGGING ONLY: Print references held on object.
		    inline  void            printRefs() const { getWeakRefs()->printRefs(); }

			    //! DEBUGGING ONLY: Enable tracking of object.
		    inline  void            trackMe(bool enable, bool retain)
		    {
			getWeakRefs()->trackMe(enable, retain);
		    }

		    typedef RefBase basetype;

		protected:
					    RefBase();
		    virtual                 ~RefBase();

		    //! Flags for extendObjectLifetime()
		    enum {
			OBJECT_LIFETIME_STRONG  = 0x0000,
			OBJECT_LIFETIME_WEAK    = 0x0001,
			OBJECT_LIFETIME_MASK    = 0x0001
		    };

			    void            extendObjectLifetime(int32_t mode);

		    //! Flags for onIncStrongAttempted()
		    enum {
			FIRST_INC_STRONG = 0x0001
		    };

		    virtual void            onFirstRef();
		    virtual void            onLastStrongRef(const void* id);
		    virtual bool            onIncStrongAttempted(uint32_t flags, const void* id);
		    virtual void            onLastWeakRef(const void* id);

		private:
		    friend class weakref_type;
		    class weakref_impl;

					    RefBase(const RefBase& o);
			    RefBase&        operator=(const RefBase& o);

		private:
		    friend class ReferenceMover;

		    static void renameRefs(size_t n, const ReferenceRenamer& renamer);

		    static void renameRefId(weakref_type* ref,
			    const void* old_id, const void* new_id);

		    static void renameRefId(RefBase* ref,
			    const void* old_id, const void* new_id);

			weakref_impl* const mRefs;
		};
RefBase嵌套了一个重要的类weakref_type，也就是前面m_refs指针所属的类型。RefBase中，还有一个mRefs的成员变量，类型为weakref_impl。从名称上来看，它应该是weakref_type的实现类。

		class RefBase::weakref_impl : public RefBase::weakref_type
		{
		public:
		    volatile int32_t    mStrong;
		    volatile int32_t    mWeak;
		    RefBase* const      mBase;
		    volatile int32_t    mFlags;

		#if !DEBUG_REFS

		    weakref_impl(RefBase* base)
			: mStrong(INITIAL_STRONG_VALUE)
			, mWeak(0)
			, mBase(base)
			, mFlags(0)
		    {
		    }

		    void addStrongRef(const void* /*id*/) { }
		    void removeStrongRef(const void* /*id*/) { }
		    void renameStrongRefId(const void* /*old_id*/, const void* /*new_id*/) { }
		    void addWeakRef(const void* /*id*/) { }
		    void removeWeakRef(const void* /*id*/) { }
		    void renameWeakRefId(const void* /*old_id*/, const void* /*new_id*/) { }
		    void printRefs() const { }
		    void trackMe(bool, bool) { }

		#else

		    weakref_impl(RefBase* base)
			: mStrong(INITIAL_STRONG_VALUE)
			, mWeak(0)
			, mBase(base)
			, mFlags(0)
			, mStrongRefs(NULL)
			, mWeakRefs(NULL)
			, mTrackEnabled(!!DEBUG_REFS_ENABLED_BY_DEFAULT)
			, mRetain(false)
		    {
		    }
		    
		    ~weakref_impl()
		    {
			bool dumpStack = false;
			if (!mRetain && mStrongRefs != NULL) {
			    dumpStack = true;
			    ALOGE("Strong references remain:");
			    ref_entry* refs = mStrongRefs;
			    while (refs) {
				char inc = refs->ref >= 0 ? '+' : '-';
				ALOGD("\t%c ID %p (ref %d):", inc, refs->id, refs->ref);
		#if DEBUG_REFS_CALLSTACK_ENABLED
				refs->stack.log(LOG_TAG);
		#endif
				refs = refs->next;
			    }
			}

			if (!mRetain && mWeakRefs != NULL) {
			    dumpStack = true;
			    ALOGE("Weak references remain!");
			    ref_entry* refs = mWeakRefs;
			    while (refs) {
				char inc = refs->ref >= 0 ? '+' : '-';
				ALOGD("\t%c ID %p (ref %d):", inc, refs->id, refs->ref);
		#if DEBUG_REFS_CALLSTACK_ENABLED
				refs->stack.log(LOG_TAG);
		#endif
				refs = refs->next;
			    }
			}
			if (dumpStack) {
			    ALOGE("above errors at:");
			    CallStack stack(LOG_TAG);
			}
		    }

		    void addStrongRef(const void* id) {
			//ALOGD_IF(mTrackEnabled,
			//        "addStrongRef: RefBase=%p, id=%p", mBase, id);
			addRef(&mStrongRefs, id, mStrong);
		    }

		    void removeStrongRef(const void* id) {
			//ALOGD_IF(mTrackEnabled,
			//        "removeStrongRef: RefBase=%p, id=%p", mBase, id);
			if (!mRetain) {
			    removeRef(&mStrongRefs, id);
			} else {
			    addRef(&mStrongRefs, id, -mStrong);
			}
		    }

		    void renameStrongRefId(const void* old_id, const void* new_id) {
			//ALOGD_IF(mTrackEnabled,
			//        "renameStrongRefId: RefBase=%p, oid=%p, nid=%p",
			//        mBase, old_id, new_id);
			renameRefsId(mStrongRefs, old_id, new_id);
		    }

		    void addWeakRef(const void* id) {
			addRef(&mWeakRefs, id, mWeak);
		    }

		    void removeWeakRef(const void* id) {
			if (!mRetain) {
			    removeRef(&mWeakRefs, id);
			} else {
			    addRef(&mWeakRefs, id, -mWeak);
			}
		    }

		    void renameWeakRefId(const void* old_id, const void* new_id) {
			renameRefsId(mWeakRefs, old_id, new_id);
		    }

		    void trackMe(bool track, bool retain)
		    { 
			mTrackEnabled = track;
			mRetain = retain;
		    }

		    void printRefs() const
		    {
			String8 text;

			{
			    Mutex::Autolock _l(mMutex);
			    char buf[128];
			    sprintf(buf, "Strong references on RefBase %p (weakref_type %p):\n", mBase, this);
			    text.append(buf);
			    printRefsLocked(&text, mStrongRefs);
			    sprintf(buf, "Weak references on RefBase %p (weakref_type %p):\n", mBase, this);
			    text.append(buf);
			    printRefsLocked(&text, mWeakRefs);
			}

			{
			    char name[100];
			    snprintf(name, 100, DEBUG_REFS_CALLSTACK_PATH "/%p.stack", this);
			    int rc = open(name, O_RDWR | O_CREAT | O_APPEND, 644);
			    if (rc >= 0) {
				write(rc, text.string(), text.length());
				close(rc);
				ALOGD("STACK TRACE for %p saved in %s", this, name);
			    }
			    else ALOGE("FAILED TO PRINT STACK TRACE for %p in %s: %s", this,
				      name, strerror(errno));
			}
		    }

		private:
		    struct ref_entry
		    {
			ref_entry* next;
			const void* id;
		#if DEBUG_REFS_CALLSTACK_ENABLED
			CallStack stack;
		#endif
			int32_t ref;
		    };

		    void addRef(ref_entry** refs, const void* id, int32_t mRef)
		    {
			if (mTrackEnabled) {
			    AutoMutex _l(mMutex);

			    ref_entry* ref = new ref_entry;
			    // Reference count at the time of the snapshot, but before the
			    // update.  Positive value means we increment, negative--we
			    // decrement the reference count.
			    ref->ref = mRef;
			    ref->id = id;
		#if DEBUG_REFS_CALLSTACK_ENABLED
			    ref->stack.update(2);
		#endif
			    ref->next = *refs;
			    *refs = ref;
			}
		    }

		    void removeRef(ref_entry** refs, const void* id)
		    {
			if (mTrackEnabled) {
			    AutoMutex _l(mMutex);
			    
			    ref_entry* const head = *refs;
			    ref_entry* ref = head;
			    while (ref != NULL) {
				if (ref->id == id) {
				    *refs = ref->next;
				    delete ref;
				    return;
				}
				refs = &ref->next;
				ref = *refs;
			    }

			    ALOGE("RefBase: removing id %p on RefBase %p"
				    "(weakref_type %p) that doesn't exist!",
				    id, mBase, this);

			    ref = head;
			    while (ref) {
				char inc = ref->ref >= 0 ? '+' : '-';
				ALOGD("\t%c ID %p (ref %d):", inc, ref->id, ref->ref);
				ref = ref->next;
			    }

			    CallStack stack(LOG_TAG);
			}
		    }

		    void renameRefsId(ref_entry* r, const void* old_id, const void* new_id)
		    {
			if (mTrackEnabled) {
			    AutoMutex _l(mMutex);
			    ref_entry* ref = r;
			    while (ref != NULL) {
				if (ref->id == old_id) {
				    ref->id = new_id;
				}
				ref = ref->next;
			    }
			}
		    }

		    void printRefsLocked(String8* out, const ref_entry* refs) const
		    {
			char buf[128];
			while (refs) {
			    char inc = refs->ref >= 0 ? '+' : '-';
			    sprintf(buf, "\t%c ID %p (ref %d):\n", 
				    inc, refs->id, refs->ref);
			    out->append(buf);
		#if DEBUG_REFS_CALLSTACK_ENABLED
			    out->append(refs->stack.toString("\t\t"));
		#else
			    out->append("\t\t(call stacks disabled)");
		#endif
			    refs = refs->next;
			}
		    }

		    mutable Mutex mMutex;
		    ref_entry* mStrongRefs;
		    ref_entry* mWeakRefs;

		    bool mTrackEnabled;
		    // Collect stack traces on addref and removeref, instead of deleting the stack references
		    // on removeref that match the address ones.
		    bool mRetain;

		#endif
		};
其中#else到#endif之间为dbug代码，不用看，剩余部分未实现addStrongRef等方法，上述代码中并没有引用计数器相关控制的实现，真正有用的代码在类声明的外面。

![](/home/dzx/workRecord/images/wp.png) 

看了这个图就不迷糊了。。。。

增加弱引用计数

	void RefBase::weakref_type::incWeak(const void* id)
	{
	    weakref_impl* const impl = static_cast<weakref_impl*>(this);
	    impl->addWeakRef(id);
	    const int32_t c __unused = android_atomic_inc(&impl->mWeak);
	    ALOG_ASSERT(c >= 0, "incWeak called on %p after last weak ref", this);
	}
	
增加强引用计数：同时增加强弱引用计数

	void RefBase::incStrong(const void* id) const
	{
	    weakref_impl* const refs = mRefs;
	    refs->incWeak(id);
	    
	    refs->addStrongRef(id);
	    const int32_t c = android_atomic_inc(&refs->mStrong);
	    ALOG_ASSERT(c > 0, "incStrong() called on %p after last strong ref", refs);
	#if PRINT_REFS
	    ALOGD("incStrong of %p from %p: cnt=%d\n", this, id, c);
	#endif
	    if (c != INITIAL_STRONG_VALUE)  {
		return;
	    }

	    android_atomic_add(-INITIAL_STRONG_VALUE, &refs->mStrong);
	    refs->mBase->onFirstRef();
	}	

	void RefBase::decStrong(const void* id) const
	{
	    weakref_impl* const refs = mRefs;
	    refs->removeStrongRef(id);
	    const int32_t c = android_atomic_dec(&refs->mStrong);
	#if PRINT_REFS
	    ALOGD("decStrong of %p from %p: cnt=%d\n", this, id, c);
	#endif
	    ALOG_ASSERT(c >= 1, "decStrong() called on %p too many times", refs);
	    if (c == 1) {
		refs->mBase->onLastStrongRef(id);
		if ((refs->mFlags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
		    delete this;
		}
	    }
	    refs->decWeak(id);
	}
	
	void RefBase::weakref_type::decWeak(const void* id)
	{
	    weakref_impl* const impl = static_cast<weakref_impl*>(this);
	    impl->removeWeakRef(id);
	    const int32_t c = android_atomic_dec(&impl->mWeak);
	    ALOG_ASSERT(c >= 1, "decWeak called on %p too many times", this);
	    if (c != 1) return;

	    if ((impl->mFlags&OBJECT_LIFETIME_WEAK) == OBJECT_LIFETIME_STRONG) {
		// This is the regular lifetime case. The object is destroyed
		// when the last strong reference goes away. Since weakref_impl
		// outlive the object, it is not destroyed in the dtor, and
		// we'll have to do it here.
		if (impl->mStrong == INITIAL_STRONG_VALUE) {
		    // Special case: we never had a strong reference, so we need to
		    // destroy the object now.
		    delete impl->mBase;
		} else {
		    // ALOGV("Freeing refs %p of old RefBase %p\n", this, impl->mBase);
		    delete impl;
		}
	    } else {
		// less common case: lifetime is OBJECT_LIFETIME_{WEAK|FOREVER}
		impl->mBase->onLastWeakRef(id);
		if ((impl->mFlags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_WEAK) {
		    // this is the OBJECT_LIFETIME_WEAK case. The last weak-reference
		    // is gone, we can destroy the object.
		    delete impl->mBase;
		}
	    }
	}
