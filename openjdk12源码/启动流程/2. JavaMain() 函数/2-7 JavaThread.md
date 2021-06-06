# JavaThread



## 1. ThreadShadow

> **源码: exceptions.hpp**

```c++
// ThreadShadow类是一个用于访问 _pending_exception字段 的 helper class;
class ThreadShadow: public CHeapObj<mtThread> { // ThreadShadow 继承自 CHeapObj, 即对象在堆中分配空间

 protected:
  oop  _pending_exception;                       // Thread has gc actions.
  const char* _exception_file;                   // file information for exception (debugging only)
  int         _exception_line;                   // line information for exception (debugging only)

  // 本虚拟方法存在的唯一原因是保证c++编译器为每个ThreadShadow及其子类实例都创建一个 vtable;
  // 并且这个 vtable 在Thread对象布局的最开始处; 
  // 当然有一些C++编译器会在ThreadShadow没有虚拟方法的情况下, 自动将ThreadShadow放在子类的vtable之后,
  // 但并不是所有的c++编译器都会生成这样的布局, 所以为了统一, 在ThreadShdow中定义了这个虚拟方法
  virtual void unused_initial_virtual() { }

 public:
  ThreadShadow() : _pending_exception(NULL), _exception_file(NULL), _exception_line(0) {} //构造方法
  
  // --- _pending_exception在ThreadShadow中的偏移量
  static ByteSize pending_exception_offset()     { return byte_offset_of(ThreadShadow, _pending_exception); }

  // use THROW whenever possible!
  void set_pending_exception(oop exception, const char* file, int line);

  // use CLEAR_PENDING_EXCEPTION whenever possible!
  void clear_pending_exception();
};
```



## 2. Thread

> **源码: thread.hpp**
>
> > 1. **线程本地存储TLS来存储当前线程 _thr_current**
> > 2. **ObjectMonitor，支持synchronized关键字**
> > 3. **封装了平台相关线程信息，保存在 _os_thread字段中 **
> > 4. **添加了 stack overflow 检查支持，由 _stack_base、_stack_size实现**
> > 5. **Park event 以及 自旋锁支持**

```c++
// Class hierarchy
// - Thread
//   - JavaThread
//     - various subclasses eg CompilerThread, ServiceThread
//   - NonJavaThread
//     - NamedThread
//       - VMThread
//       - ConcurrentGCThread
//       - WorkerThread
//         - GangWorker
//         - GCTaskThread
//     - WatcherThread
//     - JfrThreadSampler
//
// All Thread subclasses must be either JavaThread or NonJavaThread.
// This means !t->is_Java_thread() iff t is a NonJavaThread, or t is
// a partially constructed/destroyed Thread.

class Thread: public ThreadShadow {
	private:

	// ------------ 看来如果定义了 USE_LIBRARY_BASED_TLS_ONLY, 会使用pthread_key来实现TLS; 
	// ------ 由于默认未定义这个宏, 所以这里使用 _thread GUN C 编译指令来实现TLS
	#ifndef USE_LIBRARY_BASED_TLS_ONLY
  		// Current thread is maintained as a thread-local variable
  		static THREAD_LOCAL_DECL Thread* _thr_current;
	#endif

	protected:
  		ThreadsList* volatile _threads_hazard_ptr;
  		SafeThreadsListPtr*   _threads_list_ptr;

 	private:
  		ThreadLocalAllocBuffer _tlab;                 // ------ 重要: Thread-local eden
  		jlong _allocated_bytes;        // Cumulative number of bytes allocated on the Java heap

  		// ------------------- 《《《《《《Java Thread中定义的 ObjectMonitor》》》》》》》-------
  		ObjectMonitor* _current_pending_monitor;      // ObjectMonitor this thread is waiting to lock
  		bool _current_pending_monitor_is_from_java;   // locking is from Java code

  		ObjectMonitor* _current_waiting_monitor; // Object.wait() 方法调用时申请的哪个ObjectMonitor

	public:
  		ObjectMonitor* omFreeList;
  		int omFreeCount;                              // length of omFreeList
  		int omFreeProvision;                          // reload chunk size
  		ObjectMonitor* omInUseList;                   // SLL to track monitors in circulation
  		int omInUseCount;                             // length of omInUseList

	public:
  		Thread();
  		virtual ~Thread() = 0;        // Thread is abstract.

	protected:
  		OSThread* _osthread;  // ---- 平台相关的线程信息

  		// Support for stack overflow handling, get_thread, etc.
  		address          _stack_base;
  		size_t           _stack_size;

	private:
  		// Deadlock detection support for Mutex locks. List of locks own by thread.
  		Monitor* _owned_locks;


 	public:
  		volatile intptr_t _Stalled;
  		volatile int _TypeTag;
  		ParkEvent * _ParkEvent;                     // for synchronized()
  		ParkEvent * _SleepEvent;                    // for Thread.sleep
  		ParkEvent * _MutexEvent;                    // for native internal Mutex/Monitor
  		ParkEvent * _MuxEvent;                      // for low-level muxAcquire-muxRelease
  		int NativeSyncRecursion;                    // diagnostic

  		volatile int _OnTrap;                       // Resume-at IP delta
  		jint _hashStateW;                           // Marsaglia Shift-XOR thread-local RNG
  		jint _hashStateX;                           // thread-specific hashCode generator state
  		jint _hashStateY;
  		jint _hashStateZ;

  		volatile jint rng[4];                      // RNG for spin loop

		  static void SpinAcquire(volatile int * Lock, const char * Name);
  		static void SpinRelease(volatile int * Lock);
  		static void muxAcquire(volatile intptr_t * Lock, const char * Name);
  		static void muxAcquireW(volatile intptr_t * Lock, ParkEvent * ev);
  		static void muxRelease(volatile intptr_t * Lock);
};
```



## 3. JavaThread

> **源码: thread.hpp**
>
> > **相比 Thread父类的关注线程整体状态与实现，JavaThread更侧重于对栈帧，运行时栈帧结构这些线程执行时的具体细节**

```c++
 // --- JavaThread类定义太长了, 下面会省略一些友元、不太重要的字段和方法等
class JavaThread: public Thread {

 	private:
  		JavaThread*    _next;       //Threads 单向链表
  		bool           _on_thread_list;  //JavaThread对象被添加到Threads链表后, 本字段会更新为true
  		oop            _threadObj;       //Java语言层面的thread object; ---- 也就是Thread类???

			int _java_call_counter; //Java语言调用次数计数器??
 
  private:  
  		JavaFrameAnchor _anchor;  //当前java 栈帧以及其state 封装

      // typedef void (*ThreadFunction)(JavaThread*, Thread*);
  		ThreadFunction _entry_point; //Java语言层面Thread启动函数??

  		JNIEnv        _jni_environment; // JNIEnv后面会具体介绍
  		Method*       _callee_target; //被调用的方法

  		// 用于将结果传回给Java代码
  		oop           _vm_result;    // oop result is GC-preserved
  		Metadata*     _vm_result_2;  // non-oop result
  
  		//异步请求处理相关
 		 	enum AsyncRequests {  
    			_no_async_condition = 0,
    			_async_exception,
    			_async_unsafe_access_error
  		};
  		
  		AsyncRequests _special_runtime_exit_condition; // Enum indicating pending async. request
  		oop           _pending_async_exception;

 			// --- Safepoint支持	 
 	public:                   
  		volatile JavaThreadState _thread_state;
 	
  private:
  		ThreadSafepointState* _safepoint_state;        // 处于safepoint状态的thread的相关信息
  		address               _saved_exception_pc;     // 当异常发生时保存的指令pc
 
  		// ---- JavaThread 终止支持
      // 正常情况下, JavaThread对象的 _terminated 字段状态转换过程如下: 
      //  _not_terminated => _thread_exiting => _thread_terminated
      // _vm_exited 是一个用来表示JavaThread还在执行native code, 而VM自己即已经中止的特殊状态
  		enum TerminatedTypes {
    			_not_terminated = 0xDEAD - 2,
    			_thread_exiting,          // 当前Thread已经调用过JavaThread::exit()
    			_thread_terminated,       // JavaThread已经从thread 链表中移除
    			_vm_exited                // 当前Thread还在执行native code, 但VM已经中止; 只有VM_Exit可以设置 _vm_exited
  		};
     
  		volatile TerminatedTypes _terminated;

  		// ----- JNI attach states:
      // 通常的JavaThread的 _jni_attach_state 是 _not_attaching_via_jni. 
      // 一个native thread that is attaching via JNI 是 _attaching_via_jni 并且
      // 会转换到 _attached_via_jni;
  		enum JNIAttachStates {
    			_not_attaching_via_jni = 1,  // thread is not attaching via JNI
    			_attaching_via_jni,          // thread is attaching via JNI
    			_attached_via_jni            // thread has attached via JNI
  		};
  
		  volatile JNIAttachStates _jni_attach_state;

      // --- Thread 的stack guard pages state
 	public: 
  		enum StackGuardState {
    			stack_guard_unused,         // not needed
    			stack_guard_reserved_disabled,
    			stack_guard_yellow_reserved_disabled,// disabled (temporarily) after stack overflow
    			stack_guard_enabled         // enabled
  		};

 	private:
  		StackGuardState  _stack_guard_state;

			// --- Stack overflow (栈帧溢出) 的限制
  		address          _stack_overflow_limit;
  		address          _reserved_stack_activation;

  		// --- 编译exception, 不同于_pending_exception; _pending_exception 是运行时异常
  		volatile oop     _exception_oop;               // Exception thrown in compiled code
  		volatile address _exception_pc;                // PC where exception happened
  		volatile address _exception_handler_pc;        // PC for handler of exception
  		volatile int     _is_method_handle_return;// true if the current exception PC is a MethodHandle call site.

      // --- JNI临界区支持
 	private:
  		jint    _jni_active_critical;    // count of entries into JNI critical region
  		char* _pending_jni_exception_check_fn; // Checked JNI: function name requires exception check
		
  		void initialize(); // ---- 初始化对象字段

 	public:
  		// 构造/析构方法
  		JavaThread(bool is_attaching_via_jni = false); // for main thread and JNI attached threads
  		JavaThread(ThreadFunction entry_point, size_t stack_size = 0);
  		~JavaThread();
 

  		// --- 对于JavaThread, is_Java_thread 和 can_call_java() 两个抽象方法都会返回true
  		virtual bool is_Java_thread() const            { return true;  }
  		virtual bool can_call_java() const             { return true; }

  		// 分配一个新Java语言层面的thread对象, thread_name 允许为 NULL.
  		void allocate_threadObj(Handle thread_group, const char* thread_name, bool daemon, TRAPS);

  		// 使用JavaThread中最后一个栈帧 _anchor 来获取相关信息, 如sp寄存器值, pc寄存器值
  		bool has_last_Java_frame() const               { return _anchor.has_last_Java_frame(); }
  		intptr_t* last_Java_sp() const                 { return _anchor.last_Java_sp(); }
  		address last_Java_pc(void)                     { return _anchor.last_Java_pc(); }

  		bool is_exiting() const; //线程执行过JavaThread::exit() 或者已中止时返回true

      //---- thread 握手操作支持
 	private:
  		HandshakeState _handshake;
 	public:
  		void set_handshake_operation(HandshakeOperation* op) { _handshake.set_operation(this, op);}
  		bool has_handshake() const { return _handshake.has_operation();}
 		 	void handshake_process_by_self() {_handshake.process_by_self(this);}
  		void handshake_process_by_vmthread() { _handshake.process_by_vmthread(this);}

  		// Stack overflow support
  		//
  		//  (small addresses)
  		//
  		//  --  <-- stack_end()                   ---
  		//  |                                      |
  		//  |  red pages                           |
  		//  |                                      |
  		//  --  <-- stack_red_zone_base()          |
  		//  |                                      |
  		//  |                                     guard
  		//  |  yellow pages                       zone
  		//  |                                      |
  		//  |                                      |
  		//  --  <-- stack_yellow_zone_base()       |
  		//  |                                      |
  		//  |                                      |
  		//  |  reserved pages                      |
  		//  |                                      |
  		//  --  <-- stack_reserved_zone_base()    ---      ---
  		//                                                 /|\  shadow <-- stack_overflow_limit() (
  		//                                                  |   zone
  		//                                                 \|/  size
  		//  some untouched memory                          ---
  		//
  		//
  		//  --
  		//  |
  		//  |  shadow zone
  		//  |
  		//  --
  		//  x    frame n
  		//  --
  		//  x    frame n-1
  		//  x
  		//  --
  		//  ...
  		//
  		//  --
  		//  x    frame 0
  		//  --  <-- stack_base()
  		//
  		//  (large addresses)
  		//
 	private:
      // 这些字段值来源于 StackRedPages, StackYelloPages, StackReservedPages, StackShadowPages 值;;
      // 如果page_size大于4K, zone size 会自适应调整
  		static size_t _stack_red_zone_size;
  		static size_t _stack_yellow_zone_size;
  		static size_t _stack_reserved_zone_size;
  		static size_t _stack_shadow_zone_size;


  // JSR166 per-thread parker
	private:
  		Parker*    _parker;
 	public:
  		Parker*     parker() { return _parker; }
};
```

