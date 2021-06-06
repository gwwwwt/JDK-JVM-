# Threads::create_vm() -- 核心创建VM逻辑

> **源码: thread.cpp**

```c++
jint Threads::create_vm(JavaVMInitArgs* args, bool* canTryAgain) {
  
  // ------------------- 第1大步骤: 各种类的初始化; Arguments根据命令行参数设置对应字段值; 检查各项参数
  extern void JDK_Version_init(); //--- 参考 《2. JavaMain() 函数/2-1 JDK_Version_init()》
  

  //版本检查, 在InitializeJVM方法中构造args参数时, version传入的JNI_VERSION_1.2
  if (!is_supported_jni_version(args->version)) return JNI_EVERSION;

  // Initialize library-based TLS, --- 参考 《2. JavaMain() 函数/2-2 线程本地存储.md》
  ThreadLocalStorage::init(); 

  ostream_init();
  
  // Process java launcher properties. --- 参考第1节
  Arguments::process_sun_java_launcher_properties(args);

  // Initialize the os module. --- 参考《2. JavaMain() 函数/2-3 os::init().md》
  // 本方法主要初始化一些硬件、线程、条件/互斥对象的初始化
  // 后面还会调用另一个os初始化方法, os::init_2()
  os::init();

  Arguments::init_system_properties(); // 初始化 system properties --- 参考第2节

  JDK_Version_init(); // 上面是导入JDK_Version_init()函数, 这里是实际调用
  
  // --- 上面初始化jdk version后, 将jvm version信息加入Arguments#_system_properties中, 参考第3节
  Arguments::init_version_specific_system_properties();

  // --- 参考《2. JavaMain()函数/2-4 解析Options参数 Arguments::parse》 
  // --- 作用: 将多种来源的参数选项解析到Arguments的对应字段中
  // --- ***重要: 解析所有配置options***; 方法内部调用了os::init_container_support()
  jint parse_result = Arguments::parse(args);
  if (parse_result != JNI_OK) return parse_result;

  os::init_before_ergo(); //第4节

  jint ergo_result = Arguments::apply_ergo();//第5节

  // ------------------------------ 第2大步骤: 创建并设置JavaThread对象

  jint os_init_2_result = os::init_2(); // 参考《2. JavaMain函数/2-5 os::init2().md》

  SafepointMechanism::initialize(); // 安全点safepoint初始化, 参考《2-6 SafepointMechanism::initialize().md》

  
  // ----------- 第6节 启动时需要加载的lib处理 ------------
  
  // 对于 -Xrun 方式指定的参数, 转换成 -agentlib: 形式的参数, 之后再统一的执行 create_vm_init_agents() 函数;
  // 所以 convert_vm_init_libraries_to_agents() 必须在 create_vm_init_agents() 函数之前执行
  if (Arguments::init_libraries_at_startup()) {
    convert_vm_init_libraries_to_agents();
  }

  // Launch -agentlib/-agentpath and converted -Xrun agents
  if (Arguments::init_agents_at_startup()) {
    create_vm_init_agents();
  }
  // ------------ 第6节 end -----------------------------
  
  // --- 初始化线程状态
  _thread_list = NULL;
  _number_of_threads = 0;
  _number_of_non_daemon_threads = 0;

  
  vm_init_globals(); // ----- 第7节:  初始化全局数据结构, 在堆中创建system classes

  if (JVMCICounterSize > 0) {
    JavaThread::_jvmci_old_thread_counters = NEW_C_HEAP_ARRAY(jlong, JVMCICounterSize, mtInternal);
    memset(JavaThread::_jvmci_old_thread_counters, 0, sizeof(jlong) * JVMCICounterSize);
  } else {
    JavaThread::_jvmci_old_thread_counters = NULL; //默认会进入本分支
  }
  
  // ----- 参考《2. JavaMain()函数/2-7 JavaThread.md》
  JavaThread* main_thread = new JavaThread();    //创建JavaThread对象
  main_thread->set_thread_state(_thread_in_vm);  //更新 _thread_state 字段
  
  //设置JavaThread TLS: _thr_current或者ThreadLocalStorage::set_thread(this);
  main_thread->initialize_thread_current();
  
  // must do this before set_active_handles -- 关于具体线程栈的结构, 以后在相关note里面补
  main_thread->record_stack_base_and_size(); // 设置线程栈base和size
  main_thread->register_thread_stack_with_NMT();
  main_thread->set_active_handles(JNIHandleBlock::allocate_block()); /// ???

  main_thread->set_as_starting_thread(); //这里会创建OSThread, 见第8节

  main_thread->create_stack_guard_pages(); //关于stack guard page, 以后再整理

  ObjectMonitor::Initialize(); // 初始化Java语言层面的synchronization支持, 但Initialize()中没有什么太重要的逻辑

  jint status = init_globals(); // 差点把这儿放过去, 这里初始化了很多重要东西

  // Should be done after the heap is fully created
  main_thread->cache_global_variables();

  HandleMark hm;

  { MutexLocker mu(Threads_lock);
    Threads::add(main_thread);
  }

  // Any JVMTI raw monitors entered in onload will transition into
  // real raw monitor. VM is setup enough here for raw monitor enter.
  JvmtiExport::transition_pending_onload_raw_monitors();

  // Create the VMThread
  { TraceTime timer("Start VMThread", TRACETIME_LOG(Info, startuptime));

  VMThread::create();
    Thread* vmthread = VMThread::vm_thread();

    if (!os::create_thread(vmthread, os::vm_thread)) {
      vm_exit_during_initialization("Cannot create VM thread. "
                                    "Out of system resources.");
    }

    // Wait for the VM thread to become ready, and VMThread::run to initialize
    // Monitors can have spurious returns, must always check another state flag
    {
      MutexLocker ml(Notify_lock);
      os::start_thread(vmthread);
      while (vmthread->active_handles() == NULL) {
        Notify_lock->wait();
      }
    }
  }

  assert(Universe::is_fully_initialized(), "not initialized");
  if (VerifyDuringStartup) {
    // Make sure we're starting with a clean slate.
    VM_Verify verify_op;
    VMThread::execute(&verify_op);
  }

  // We need this to update the java.vm.info property in case any flags used
  // to initially define it have been changed. This is needed for both CDS and
  // AOT, since UseSharedSpaces and UseAOT may be changed after java.vm.info
  // is initially computed. See Abstract_VM_Version::vm_info_string().
  // This update must happen before we initialize the java classes, but
  // after any initialization logic that might modify the flags.
  Arguments::update_vm_info_property(VM_Version::vm_info_string());

  Thread* THREAD = Thread::current();

  // Always call even when there are not JVMTI environments yet, since environments
  // may be attached late and JVMTI must track phases of VM execution
  JvmtiExport::enter_early_start_phase();

  // Notify JVMTI agents that VM has started (JNI is up) - nop if no agents.
  JvmtiExport::post_early_vm_start();

  initialize_java_lang_classes(main_thread, CHECK_JNI_ERR);

  quicken_jni_functions();

  // No more stub generation allowed after that point.
  StubCodeDesc::freeze();

  // Set flag that basic initialization has completed. Used by exceptions and various
  // debug stuff, that does not work until all basic classes have been initialized.
  set_init_completed();

  LogConfiguration::post_initialize();
  Metaspace::post_initialize();

  HOTSPOT_VM_INIT_END();

  // record VM initialization completion time
#if INCLUDE_MANAGEMENT
  Management::record_vm_init_completed();
#endif // INCLUDE_MANAGEMENT

  // Signal Dispatcher needs to be started before VMInit event is posted
  os::initialize_jdk_signal_support(CHECK_JNI_ERR);

  // Start Attach Listener if +StartAttachListener or it can't be started lazily
  if (!DisableAttachMechanism) {
    AttachListener::vm_start();
    if (StartAttachListener || AttachListener::init_at_startup()) {
      AttachListener::init();
    }
  }

  // Launch -Xrun agents
  // Must be done in the JVMTI live phase so that for backward compatibility the JDWP
  // back-end can launch with -Xdebug -Xrunjdwp.
  if (!EagerXrunInit && Arguments::init_libraries_at_startup()) {
    create_vm_init_libraries();
  }

  if (CleanChunkPoolAsync) {
    Chunk::start_chunk_pool_cleaner_task();
  }

  // initialize compiler(s)
#if defined(COMPILER1) || COMPILER2_OR_JVMCI
#if INCLUDE_JVMCI
  bool force_JVMCI_intialization = false;
  if (EnableJVMCI) {
    // Initialize JVMCI eagerly when it is explicitly requested.
    // Or when JVMCIPrintProperties is enabled.
    // The JVMCI Java initialization code will read this flag and
    // do the printing if it's set.
    force_JVMCI_intialization = EagerJVMCI || JVMCIPrintProperties;

    if (!force_JVMCI_intialization) {
      // 8145270: Force initialization of JVMCI runtime otherwise requests for blocking
      // compilations via JVMCI will not actually block until JVMCI is initialized.
      force_JVMCI_intialization = UseJVMCICompiler && (!UseInterpreter || !BackgroundCompilation);
    }
  }
#endif
  CompileBroker::compilation_init_phase1(CHECK_JNI_ERR);
  // Postpone completion of compiler initialization to after JVMCI
  // is initialized to avoid timeouts of blocking compilations.
  if (JVMCI_ONLY(!force_JVMCI_intialization) NOT_JVMCI(true)) {
    CompileBroker::compilation_init_phase2();
  }
#endif

  // Pre-initialize some JSR292 core classes to avoid deadlock during class loading.
  // It is done after compilers are initialized, because otherwise compilations of
  // signature polymorphic MH intrinsics can be missed
  // (see SystemDictionary::find_method_handle_intrinsic).
  initialize_jsr292_core_classes(CHECK_JNI_ERR);

  // This will initialize the module system.  Only java.base classes can be
  // loaded until phase 2 completes
  call_initPhase2(CHECK_JNI_ERR);

  // Always call even when there are not JVMTI environments yet, since environments
  // may be attached late and JVMTI must track phases of VM execution
  JvmtiExport::enter_start_phase();

  // Notify JVMTI agents that VM has started (JNI is up) - nop if no agents.
  JvmtiExport::post_vm_start();

  // Final system initialization including security manager and system class loader
  call_initPhase3(CHECK_JNI_ERR);

  // cache the system and platform class loaders
  SystemDictionary::compute_java_loaders(CHECK_JNI_ERR);

#if INCLUDE_CDS
  if (DumpSharedSpaces) {
    // capture the module path info from the ModuleEntryTable
    ClassLoader::initialize_module_path(THREAD);
  }
#endif

#if INCLUDE_JVMCI
  if (force_JVMCI_intialization) {
    JVMCIRuntime::force_initialization(CHECK_JNI_ERR);
    CompileBroker::compilation_init_phase2();
  }
#endif

  // Always call even when there are not JVMTI environments yet, since environments
  // may be attached late and JVMTI must track phases of VM execution
  JvmtiExport::enter_live_phase();

  // Make perfmemory accessible
  PerfMemory::set_accessible(true);

  // Notify JVMTI agents that VM initialization is complete - nop if no agents.
  JvmtiExport::post_vm_initialized();

  JFR_ONLY(Jfr::on_vm_start();)

#if INCLUDE_MANAGEMENT
  Management::initialize(THREAD);

  if (HAS_PENDING_EXCEPTION) {
    // management agent fails to start possibly due to
    // configuration problem and is responsible for printing
    // stack trace if appropriate. Simply exit VM.
    vm_exit(1);
  }
#endif // INCLUDE_MANAGEMENT

  if (MemProfiling)                   MemProfiler::engage();
  StatSampler::engage();
  if (CheckJNICalls)                  JniPeriodicChecker::engage();

  BiasedLocking::init();

#if INCLUDE_RTM_OPT
  RTMLockingCounters::init();
#endif

  if (JDK_Version::current().post_vm_init_hook_enabled()) {
    call_postVMInitHook(THREAD);
    // The Java side of PostVMInitHook.run must deal with all
    // exceptions and provide means of diagnosis.
    if (HAS_PENDING_EXCEPTION) {
      CLEAR_PENDING_EXCEPTION;
    }
  }

  {
    MutexLocker ml(PeriodicTask_lock);
    // Make sure the WatcherThread can be started by WatcherThread::start()
    // or by dynamic enrollment.
    WatcherThread::make_startable();
    // Start up the WatcherThread if there are any periodic tasks
    // NOTE:  All PeriodicTasks should be registered by now. If they
    //   aren't, late joiners might appear to start slowly (we might
    //   take a while to process their first tick).
    if (PeriodicTask::num_tasks() > 0) {
      WatcherThread::start();
    }
  }

  create_vm_timer.end();
#ifdef ASSERT
  _vm_complete = true;
#endif

  if (DumpSharedSpaces) {
    MetaspaceShared::preload_and_dump(CHECK_JNI_ERR);
    ShouldNotReachHere();
  }

  return JNI_OK;
}
```



## 1. Arguments::process_sun_java_launcher_properties()方法

> **源文件：arguments.cpp**
>
> **凡是涉及到调用Arguments的处理options逻辑的话，大部分都是转化到Arguments的特定字段值 **
>
> **本方法遍历命令行options，主要处理Launcher相关参数，可能会影响到的字段为: **
>
> **`_sun_java_launcher`、`_sun_java_launcher_is_altjvm`、`_sun_java_launcher_pid`**

```c++
// 解析命令行获取 Launcher相关JVM options
void Arguments::process_sun_java_launcher_properties(JavaVMInitArgs* args) {
  
  //必须先处理Launcher option, 因为后面初始化VM时可能会用到这些option
  for (int index = 0; index < args->nOptions; index++) {
    const JavaVMOption* option = args->options + index;
    const char* tail;

    //参考《1. 入口.md》, tail = "SUN_STANDARD"
    if (match_option(option, "-Dsun.java.launcher=", &tail)) {
      	//设置 _sun_java_launcher = "SUN_STANDARD"
      	process_java_launcher_argument(tail, option->extraInfo);
      	continue;
    }
    
   	//--- 默认是没有下面两个option的
    if (match_option(option, "-Dsun.java.launcher.is_altjvm=", &tail)) {
    	  if (strcmp(tail, "true") == 0) {
      	  _sun_java_launcher_is_altjvm = true;
      	}
      	continue;
    }
    
    if (match_option(option, "-Dsun.java.launcher.pid=", &tail)) {
    	  _sun_java_launcher_pid = atoi(tail);
      	continue;
    }
  }
}
```





## 2. Arguments::init_system_properties()

> **函数作用: 初始化Arguments中的用于保存不同的path信息以及属性的成员变量，主要有如下几个字段**
>
> > **Arguments中定义了多个 (PathString\*) 或 (SystemProperty\*)类型的成员变量：**
> >
> > + **_system_boot_class_path：boot class path路径**
> > + **_vm_info：vm完整信息，这里只是初始化，需要等到解析参数后再补完**
> > + **\_sun_boot_library_path、\_java_library_path、\_java_home、\_java_class_path：需要等到调用os::init_system_properties_values()后补完**
> > + **_jdk_boot_class_path_append: 存储 -Xbootclasspath/a: 参数信息**
> > + **_system_properties：jvm版本、名字、调试级别等信息链表，上面的boot_class等也会append到这个链表上**

> **初始化上面的字段后，调用`os::init_system_properties_values()`填充上面的字段信息**

> **源码: arguments.cpp**
>
> > **key/value字符串由类SystemProperty包装，且SystemProperty除了包装key/value字符串外，还定义了`_internal` 和 `_writable`两个成员变量。类定义：**
> >
> > ```c++
> > class PathString : public CHeapObj<mtArguments> {
> > protected:
> > char* _value;
> > ...
> > }
> > 
> > class SystemProperty : public PathString {
> > private:
> > char*           _key;
> > SystemProperty* _next;
> > bool            _internal;
> > bool            _writeable;
> > ...
> > }
> > ```
> >
> > > **可以看到，\_value在父类PathString中定义，\_key在SystemProperty中定义。**
> > >
> > > **此外还有一个`_next`字段，所以多个SystemProperty可以构成一个单向链表。**

```c++
void Arguments::init_system_properties() {
  // --- 下面的XXX_path字段, _value使用NULL来初始化
  // --- 初始化_system_properies链表, 并先往链表中添加了VM版本信息SystemProperty属性

  //--- SystemProperty构造方法的第3个参数 false/true 设置的是 _writeable字段
  //--- 第4个参数 false/true 设置的是 _internal字段
  PropertyList_add(&_system_properties, 
                   new SystemProperty("java.vm.specification.name",
                   				"Java Virtual Machine Specification",  false));
  
  //"12-internal+0-adhoc.gwwwwt.jdk12"
  PropertyList_add(&_system_properties, 
                   new SystemProperty("java.vm.version", VM_Version::vm_release(),  false));
  
  //"OpenJDK 64-Bit Server VM"
  PropertyList_add(&_system_properties, 
                   new SystemProperty("java.vm.name", VM_Version::vm_name(),  false));
  //"fastdebug"
  PropertyList_add(&_system_properties,
                   new SystemProperty("jdk.debug", VM_Version::jdk_debug_level(),  false));

  //""mixed mode, aot, sharing""
  _vm_info = new SystemProperty("java.vm.info", VM_Version::vm_info_string(), true);
  
  _system_boot_class_path = new PathString(NULL);

  // 下面几个是 JVMTI agent writable properties, 并且是os specific;
  // 它们的 _value初始化为NULL或"". 
  // 而具体的值填充是在下面的os::init_system_properties_values() 方法中完成的
  _sun_boot_library_path = new SystemProperty("sun.boot.library.path", NULL,  true);
  _java_library_path = new SystemProperty("java.library.path", NULL,  true);
  _java_home =  new SystemProperty("java.home", NULL,  true);
  _java_class_path = new SystemProperty("java.class.path", "",  true);
  
  // jdk.boot.class.path.append is a non-writeable, internal property.
  // It can only be set by either:
  //    - -Xbootclasspath/a:
  //    - AddToBootstrapClassLoaderSearch during JVMTI OnLoad phase
  _jdk_boot_class_path_append = new SystemProperty("jdk.boot.class.path.append", "", false, true);

  // Add to System Property list.
  PropertyList_add(&_system_properties, _sun_boot_library_path);
  PropertyList_add(&_system_properties, _java_library_path);
  PropertyList_add(&_system_properties, _java_home);
  PropertyList_add(&_system_properties, _java_class_path);
  PropertyList_add(&_system_properties, _jdk_boot_class_path_append);
  PropertyList_add(&_system_properties, _vm_info);

  // 根据共享库位置设置上面的_java_home、_sun_boot_library_path、_ext_dirs等信息
  os::init_system_properties_values();
}
```



### 2.1 os::init_system_properties_values()

> **源码: os_bsd.cpp**
>
> **作用: os模块获取当前共享库的路径来填充Arguments中的相关字段信息**
>
> **经过调用后，影响到的Arguments字段值如下： **
>
> > **以本机MacOS为例，下面的路径前缀都是"/Users/gwwwwt/jvm/jdk12/build/macosx-x86_64-server-fastdebug/"**
> >
> > **Arguments::_sun_boot_library_path: `jdk/lib`**
> >
> > **Arguments::_java_home: `jdk`**
> >
> > **Arguments::_system_boot_class_path: `modules/java.base`**
> >
> > **Arguments::_has_jimage: false**
> >
> > **Arguments::_java_library_path:`/Users/gwwwwt/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.`**
> >
> > **Arguments::_ext_dirs: `/Users/gwwwwt/Library/Java/Extensions:/Users/gwwwwt/jvm/jdk12/build/macosx-x86_64-server-fastdebug/jdk/lib/ext:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java`**

```c++
void os::init_system_properties_values() {
  /*
  在MacOS上,下面的libjvm.so需要换成libjvm.dylib
  本方法在product version jdk中主要有如下几步:
  1. 从 libjvm.so的完整路径获取 JAVA_HOME 值; 下面的os::jvm_path();
  	libjvm.so的路径应该是 <JAVA_HOME>/jre/lib/<arch>/{client|server}/libjvm.so
  2. 如果在libjvm.so路径中包含"jre/lib/"子字符串, 则认为libjvm.so路径正确
  	如果不包含"jre/lib/"子字符串, 则报错返回"Could not create the Java virtual machine."
  
  而在debugging version jdk中则主要是如下几步:
  1. 接上面product version中的第2步, 如果libjvm.so路径不包含"jre/lib",
  	并不会退出, 而是检查"$JAVA_HOME"环境变量
  2. 如果定义了"$JAVA_HOME", 并且可以定位 $JAVAH_HOME/jre/lib/<arch>,
  	则附加 "hotspot/libjvm.so"到这个path, 则检查libjvm.so路径是否存在:
  	<JAVA_HOME>/jre/lib/<arch>/hotspot/libjvm.so
  3. 检查不通过直接退出
  */
#ifndef DEFAULT_LIBPATH
  #ifndef OVERRIDE_LIBPATH
    #define DEFAULT_LIBPATH "/lib:/usr/lib"
  #endif
#endif

// Base path of extensions installed on the system.
#define SYS_EXT_DIR     "/usr/java/packages"
#define EXTENSIONS_DIR  "/lib/ext"

#ifndef __APPLE__
// 非Mac环境逻辑略
#else // __APPLE__

  #define SYS_EXTENSIONS_DIR   "/Library/Java/Extensions"
  #define SYS_EXTENSIONS_DIRS  SYS_EXTENSIONS_DIR ":/Network" SYS_EXTENSIONS_DIR ":/System" SYS_EXTENSIONS_DIR ":/usr/lib/java"

  const char *user_home_dir = get_home(); //获取用户home目录, 在本机Mac上返回NULL, 

  size_t system_ext_size = strlen(user_home_dir) + sizeof(SYS_EXTENSIONS_DIR) +
    sizeof(SYS_EXTENSIONS_DIRS);

  const size_t bufsize =
    MAX2((size_t)MAXPATHLEN,  // for dll_dir & friends.
         (size_t)MAXPATHLEN + sizeof(EXTENSIONS_DIR) + system_ext_size); // extensions dir
  char *buf = (char *)NEW_C_HEAP_ARRAY(char, bufsize, mtInternal);

  { //--- 设置Arguments::_sun_boot_library_path
    //--- 设置Arguments::_java_home
    char *pslash;
    //参考下面3.5.1.1节, jvm_path返回后, buf数组中的内容为:
    ///Users/gwwwwt/jvm/jdk12/build/macosx-x86_64-server-fastdebug/jdk/lib/server/libjvm.dylib
    os::jvm_path(buf, bufsize);

    // 通过pslash将buf中的路径截断到如下字符串:
    //"/Users/gwwwwt/jvm/jdk12/build/macosx-x86_64-server-fastdebug/jdk/lib"
    // 即buf中的路径也是上面这个字符串
    *(strrchr(buf, '/')) = '\0'; // Get rid of /libjvm.so.
    pslash = strrchr(buf, '/');
    if (pslash != NULL) {
      *pslash = '\0';            // Get rid of /{client|server|hotspot}.
    }

    // --- 1. 设置 _sun_boot_library_path#_value 字段为".../jdk/lib"
    Arguments::set_dll_dir(buf);

    // --- 2. 设置 _java_home#_value 字段为 ".../jdk"
    if (pslash != NULL) {
      pslash = strrchr(buf, '/');
      if (pslash != NULL) {
        *pslash = '\0';          // Get rid of /lib.
      }
    }
    Arguments::set_java_home(buf);
    
    //设置Arguments::_system_boot_class_path: modules/java.base
    set_boot_path('/', ':'); 
  }

  { //--- 设置Arguments::_java_library_path
    const char *l = ::getenv("JAVA_LIBRARY_PATH");
    const char *l_colon = ":";
    if (l == NULL) { l = ""; l_colon = ""; }

    const char *v = ::getenv("DYLD_LIBRARY_PATH");
    const char *v_colon = ":";
    if (v == NULL) { v = ""; v_colon = ""; }

    char *ld_library_path = (char *)NEW_C_HEAP_ARRAY(char,
                                                     strlen(v) + 1 + strlen(l) + 1 +
                                                     system_ext_size + 3,
                                                     mtInternal);
    sprintf(ld_library_path, "%s%s%s%s%s" SYS_EXTENSIONS_DIR ":" SYS_EXTENSIONS_DIRS ":.",
            v, v_colon, l, l_colon, user_home_dir);
    /*
    设置_java_library_path#_value字段为:
"/Users/gwwwwt/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:."
    */
    Arguments::set_library_path(ld_library_path);
    FREE_C_HEAP_ARRAY(char, ld_library_path);
  }

  /*
  设置Arguments::_ext_dirs为: 
  "/Users/gwwwwt/Library/Java/Extensions:/Users/gwwwwt/jvm/jdk12/build/macosx-x86_64-server-fastdebug/jdk/lib/ext:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java"
  */
  sprintf(buf, "%s" SYS_EXTENSIONS_DIR ":%s" EXTENSIONS_DIR ":" SYS_EXTENSIONS_DIRS,
          user_home_dir, Arguments::get_java_home());
  Arguments::set_ext_dirs(buf);

  FREE_C_HEAP_ARRAY(char, buf);
#undef SYS_EXTENSIONS_DIR
#undef SYS_EXTENSIONS_DIRS
#endif // __APPLE__
#undef SYS_EXT_DIR
#undef EXTENSIONS_DIR
}
```



#### 2.1.1 os::jvm_path()方法

> **源码: os_bsd.cpp**
>
> **本机返回路径："/Users/gwwwwt/jvm/jdk12/build/macosx-x86_64-server-fastdebug/jdk/lib/server/libjvm.dylib"**

```c++
void os::jvm_path(char *buf, jint buflen) { // 获取libjvm.so或libjvm.dylib的路径
  // 如果全局变量 char saved_jvm_path[]的第一个字符非, 认为已初始化
  // 则直接复制 saved_jvm_path内容即可
  if (saved_jvm_path[0] != 0) {
    strcpy(buf, saved_jvm_path);
    return;
  }

  // dladdr系统调用, 获取os::jvm_path方法指针所在的共享库路径;
  // 也就是 libjvm.[so,dylib]的路径,保存在dli_fname数组中
  char dli_fname[MAXPATHLEN];
  bool ret = dll_address_to_library_name(
                                         CAST_FROM_FN_PTR(address, os::jvm_path),
                                         dli_fname, sizeof(dli_fname), NULL);
  assert(ret, "cannot locate libjvm");
  char *rp = NULL;
  if (ret && dli_fname[0] != '\0') {
    rp = os::Posix::realpath(dli_fname, buf, buflen);
  }
  if (rp == NULL) {
    return;
  }

  //--- 下面if条件默认false, 略过
  if (Arguments::sun_java_launcher_is_altjvm()) {
    // Support for the java launcher's '-XXaltjvm=<path>' option. Typical
    // value for buf is "<JAVA_HOME>/jre/lib/<arch>/<vmtype>/libjvm.so"
    // or "<JAVA_HOME>/jre/lib/<vmtype>/libjvm.dylib". If "/jre/lib/"
    // appears at the right place in the string, then assume we are
    // installed in a JDK and we're done. Otherwise, check for a
    // JAVA_HOME environment variable and construct a path to the JVM
    // being overridden.

    const char *p = buf + strlen(buf) - 1;
    for (int count = 0; p > buf && count < 5; ++count) {
      for (--p; p > buf && *p != '/'; --p)
        /* empty */ ;
    }
    if (strncmp(p, "/jre/lib/", 9) != 0) {
      char* java_home_var = ::getenv("JAVA_HOME");
      if (java_home_var != NULL && java_home_var[0] != 0) {
        char* jrelib_p;
        int len;

        p = strrchr(buf, '/');
        assert(strstr(p, "/libjvm") == p, "invalid library name");

        rp = os::Posix::realpath(java_home_var, buf, buflen);
        if (rp == NULL) {
          return;
        }

        len = strlen(buf);
        assert(len < buflen, "Ran out of buffer space");
        jrelib_p = buf + len;

        snprintf(jrelib_p, buflen-len, "/jre/lib");
        if (0 != access(buf, F_OK)) {
          snprintf(jrelib_p, buflen-len, "/lib");
        }

        len = strlen(buf);
        jrelib_p = buf + len;
        snprintf(jrelib_p, buflen-len, "/%s", COMPILER_VARIANT);
        if (0 != access(buf, F_OK)) {
          snprintf(jrelib_p, buflen-len, "%s", "");
        }

        if (0 == access(buf, F_OK)) {
          len = strlen(buf);
          snprintf(buf + len, buflen-len, "/libjvm%s", JNI_LIB_SUFFIX);
        } else {
          rp = os::Posix::realpath(dli_fname, buf, buflen);
          if (rp == NULL) {
            return;
          }
        }
      }
    }
  }

  //saved_jvm_path包含了libjvm共享库路径
  strncpy(saved_jvm_path, buf, MAXPATHLEN);
  saved_jvm_path[MAXPATHLEN - 1] = '\0';
}
```



## 3. Arguments::init_version_specific_system_properties() 方法

> **源码: arguments.cpp**
>
> **作用:  本方法的调用顺序排在`JDK_Version_init();`之后，所以本方法的作用就是将`JDK_Version::_current`中保存的JDK 版本信息也保存到`_system_properties链表`**

```c++
// Update/Initialize System properties after JDK version number is known
void Arguments::init_version_specific_system_properties() {
  enum { bufsz = 16 };
  char buffer[bufsz];
  const char* spec_vendor = "Oracle Corporation";
  uint32_t spec_version = JDK_Version::current().major_version(); //12

  jio_snprintf(buffer, bufsz, UINT32_FORMAT, spec_version); //12

  //"Oracle Corporation"
  PropertyList_add(&_system_properties,new SystemProperty("java.vm.specification.vendor",  spec_vendor, false));
  
  //"12"
  PropertyList_add(&_system_properties, new SystemProperty("java.vm.specification.version", buffer, false));
  
  //"Oracle Corporation"
  PropertyList_add(&_system_properties,new SystemProperty("java.vm.vendor", VM_Version::vm_vendor(),  false));
}
```





## 4. os::init_before_ergo() 方法

> **主要设置JavaThread 线程栈大小**

```c++
void os::init_before_ergo() {
  //os._initial_active_processor_count = os._processor_count
  //os._processor_count字段在 3.4节 os::init() 中初始化
  initialize_initial_active_processor_count();
  
  large_page_init(); //MacOS空方法体

  // 设置JavaThread的相关字段值; 它们需要在 os::init_2()中进行设置最小栈大小 之前设置
  // vm_age_size() 返回os::Bsd::page_size(); 它同样是在 3.4节 os::init()中初始化的, 一般为4k
  // StackRedPages: 1; StackYellowPages: 2; StackReservedPages: 1; StackShadowPages: 20
  JavaThread::set_stack_red_zone_size     (align_up(StackRedPages      * 4 * K, vm_page_size()));
  JavaThread::set_stack_yellow_zone_size  (align_up(StackYellowPages   * 4 * K, vm_page_size()));
  JavaThread::set_stack_reserved_zone_size(align_up(StackReservedPages * 4 * K, vm_page_size()));
  JavaThread::set_stack_shadow_zone_size  (align_up(StackShadowPages   * 4 * K, vm_page_size()));

  // VM version initialization identifies some characteristics of the
  // platform that are used during ergonomic decisions.
  VM_Version::init_before_ergo(); //空实现
}
```



## 5. Arguments::apply_ergo() 方法

> **垃圾收集器、可用堆大小、GCConfig初始化**

```c++
jint Arguments::apply_ergo() {
  // Set flags based on ergonomics.
  jint result = set_ergonomics_flags(); // 1): 其中的重要步骤是确定垃圾回收器, 使用compressedOops和compressed_obj_ptr
  if (result != JNI_OK) return result;

  // Set heap size based on available physical memory
  set_heap_size();

  GCConfig::arguments()->initialize(); //设置默认的垃圾收集器参数

  set_shared_spaces_flags(); //没啥用

  Metaspace::ergo_initialize(); // Initialize Metaspace flags and alignments

  // Set compiler flags after GC is selected and GC specific
  // flags (LoopStripMiningIter) are set.
  CompilerConfig::ergo_initialize();

  // Set bytecode rewriting flags
  set_bytecode_flags();

  // Set flags if aggressive optimization flags are enabled
  jint code = set_aggressive_opts_flags();
  if (code != JNI_OK) {
    return code;
  }

  // Turn off biased locking for locking debug mode flags,
  // which are subtly different from each other but neither works with
  // biased locking
  if (UseHeavyMonitors
#ifdef COMPILER1
      || !UseFastLocking
#endif // COMPILER1
#if INCLUDE_JVMCI
      || !JVMCIUseFastLocking
#endif
    ) {
    if (!FLAG_IS_DEFAULT(UseBiasedLocking) && UseBiasedLocking) {
      // flag set to true on command line; warn the user that they
      // can't enable biased locking here
      warning("Biased Locking is not supported with locking debug flags"
              "; ignoring UseBiasedLocking flag." );
    }
    UseBiasedLocking = false;
  }

  if (PrintAssembly && FLAG_IS_DEFAULT(DebugNonSafepoints)) {
    warning("PrintAssembly is enabled; turning on DebugNonSafepoints to gain additional output");
    DebugNonSafepoints = true;
  }

  if (FLAG_IS_CMDLINE(CompressedClassSpaceSize) && !UseCompressedClassPointers) {
    warning("Setting CompressedClassSpaceSize has no effect when compressed class pointers are not used");
  }

#ifndef PRODUCT
  if (!LogVMOutput && FLAG_IS_DEFAULT(LogVMOutput)) {
    if (use_vm_log()) {
      LogVMOutput = true;
    }
  }
#endif // PRODUCT

  if (PrintCommandLineFlags) {
    JVMFlag::printSetFlags(tty);
  }

  // Apply CPU specific policy for the BiasedLocking
  if (UseBiasedLocking) {
    if (!VM_Version::use_biased_locking() && !(FLAG_IS_CMDLINE(UseBiasedLocking))) {
      UseBiasedLocking = false;
    }
  }
#ifdef COMPILER2
  if (!UseBiasedLocking) {
    UseOptoBiasInlining = false;
  }
#endif

  // ThreadLocalHandshakesConstraintFunc handles the constraints.
  if (FLAG_IS_DEFAULT(ThreadLocalHandshakes) || !SafepointMechanism::supports_thread_local_poll()) {
    log_debug(ergo)("ThreadLocalHandshakes %s", ThreadLocalHandshakes ? "enabled." : "disabled.");
  } else {
    log_info(ergo)("ThreadLocalHandshakes %s", ThreadLocalHandshakes ? "enabled." : "disabled.");
  }

  return JNI_OK;
}
```



### 5.1 Arguments::set_ergonomics_flags() 方法

```c++
jint Arguments::set_ergonomics_flags() {
  //openjdk12 选的G1收集器
  GCConfig::initialize(); //1. 选择垃圾收集器 _arguments = select_gc(); 宏太多了,追不进去了, 丫的一堆人疯了嵌套宏, 有病

  set_conservative_max_heap_alignment(); // 2. ???

  set_use_compressed_oops(); //如果本机内存小于 max_heap_for_compressed_oops(), 则会使用压缩oops

  set_use_compressed_klass_ptrs(); //使用压缩klass_ptrs的前提是使用压缩oops, 所以本函数必须在上一个函数调用的后面

  return JNI_OK;
}

// ---GCConfig.cpp
//---- 初始化可用GC列表
void GCConfig::initialize() {
  assert(_arguments == NULL, "Already initialized");
  _arguments = select_gc();
}

GCArguments* GCConfig::select_gc() {
  // Fail immediately if an unsupported GC is selected
  fail_if_unsupported_gc_is_selected();

  /*
  is_no_gc_selected() 会检查GCConfig.cpp中定义的静态变量:
  static const SupportedGC SupportedGCs[] = {
   SupportedGC(UseConcMarkSweepGC, CollectedHeap::CMS, cmsArguments, "concurrent mark sweep gc"),
   SupportedGC(UseEpsilonGC, CollectedHeap::Epsilon, epsilonArguments, "epsilon gc"),
   SupportedGC(UseG1GC, CollectedHeap::G1, g1Arguments, "g1 gc"),
   SupportedGC(UseParallelGC, CollectedHeap::Parallel, parallelArguments, "parallel gc"),
   SupportedGC(UseParallelOldGC, CollectedHeap::Parallel, parallelArguments, "parallel gc"),
   SupportedGC(UseSerialGC, CollectedHeap::Serial, serialArguments, "serial gc"),
   SupportedGC(UseShenandoahGC, CollectedHeap::Shenandoah, shenandoahArguments, "shenandoah gc"),
   SupportedGC(UseZGC, CollectedHeap::Z, zArguments, "z gc") };

  数组中的每个 SupportedGC对象的 _flags字段, 即上面构造方法的第一个参数; 
  初始时所有变量值均为false
  */
  if (is_no_gc_selected()) {
    // 对于MacOS+openjdk12环境, 设置 UseG1GC = true;
    // 即采用 G1 收集器
    select_gc_ergonomically(); 

    // Succeeded to select GC ergonomically
    _gc_selected_ergonomically = true;
  }
  
  // Exactly one GC selected
  FOR_EACH_SUPPORTED_GC(gc) {
    if (gc->_flag) {
      return &gc->_arguments;
    }
  }

  fatal("Should have found the selected GC");

  return NULL;
}
```



## 6. convert_vm_init_libraries_to_agents() 方法

> **thread.cpp**

```c++
/**作用: 将 -Xrun 参数转换为 -agentlib: 形式，之后再统一的使用 -agentlib: 逻辑处理**/
// **由于见过的 -Xrun 的方式比较少，所以暂时先略过这个方法**
void Threads::convert_vm_init_libraries_to_agents() {
  AgentLibrary* agent;
  AgentLibrary* next;

  // --- -Xrun 参数保存在Arguments::_libraryList 中; 遍历这个list
  for (agent = Arguments::libraries(); agent != NULL; agent = next) {
    next = agent->next();  // cache the next agent now as this agent may get moved off this list
    OnLoadEntry_t on_load_entry = lookup_jvm_on_load(agent);

    // If there is an JVM_OnLoad function it will get called later,
    // otherwise see if there is an Agent_OnLoad
    if (on_load_entry == NULL) {
      on_load_entry = lookup_agent_on_load(agent);
      if (on_load_entry != NULL) {
        // switch it to the agent list -- so that Agent_OnLoad will be called,
        // JVM_OnLoad won't be attempted and Agent_OnUnload will
        Arguments::convert_library_to_agent(agent);
      } else {
        vm_exit_during_initialization("Could not find JVM_OnLoad or Agent_OnLoad function in the library", agent->name());
      }
    }
  }
}

// ---------- 重要方法
// Create agents for -agentlib:  -agentpath:  and converted -Xrun
// Invokes Agent_OnLoad
// Called very early -- before JavaThreads exist
void Threads::create_vm_init_agents() {
  extern struct JavaVM_ main_vm;
  AgentLibrary* agent;

  JvmtiExport::enter_onload_phase();

  for (agent = Arguments::agents(); agent != NULL; agent = agent->next()) {
    // CDS dumping does not support native JVMTI agent.
    // CDS dumping supports Java agent if the AllowArchivingWithJavaAgent diagnostic option is specified.
    if (DumpSharedSpaces) {
      if(!agent->is_instrument_lib()) {
        vm_exit_during_cds_dumping("CDS dumping does not support native JVMTI agent, name", agent->name());
      } else if (!AllowArchivingWithJavaAgent) {
        vm_exit_during_cds_dumping(
          "Must enable AllowArchivingWithJavaAgent in order to run Java agent during CDS dumping");
      }
    }

    OnLoadEntry_t  on_load_entry = lookup_agent_on_load(agent);

    if (on_load_entry != NULL) {
      // Invoke the Agent_OnLoad function
      jint err = (*on_load_entry)(&main_vm, agent->options(), NULL);
      if (err != JNI_OK) {
        vm_exit_during_initialization("agent library failed to init", agent->name());
      }
    } else {
      vm_exit_during_initialization("Could not find Agent_OnLoad function in the agent library", agent->name());
    }
  }

  JvmtiExport::enter_primordial_phase();
}
```



## 7. vm_init_globals() 

> **源码: init.cpp**

```c++
void vm_init_globals() {  
  /*   1. 初始化各基础类型、Oop对象在堆中的大小等; oop相关如下(在默认情况下, UseCompressedOops为true):
  	if (UseCompressedOops) {
    	heapOopSize        = jintSize;  					//4
    	LogBytesPerHeapOop = LogBytesPerInt;			//2
    	LogBitsPerHeapOop  = LogBitsPerInt;				//5
    	BytesPerHeapOop    = BytesPerInt;					//4
    	BitsPerHeapOop     = BitsPerInt;					//32
  	}
  */
  basic_types_init();  
  eventlog_init();						// log相关
  mutex_init();								// lock相关
  
  /* 2. 初始化 ChunkPool; 主要是初始化ChunkPool中的四个静态变量: 
	  static ChunkPool* _large_pool;
  	static ChunkPool* _medium_pool;
  	static ChunkPool* _small_pool;
  	static ChunkPool* _tiny_pool;
  */
  chunkpool_init();
  
  perfMemory_init(); //略
  SuspendibleThreadSet_init(); //略
}
```



### 7.1 BasicType

> **源码: globalDefinitions.hpp**
>
> ```c++
> enum BasicType {
>   T_BOOLEAN     =  4,
>   T_CHAR        =  5,
>   T_FLOAT       =  6,
>   T_DOUBLE      =  7,
>   T_BYTE        =  8,
>   T_SHORT       =  9,
>   T_INT         = 10,
>   T_LONG        = 11,
>   T_OBJECT      = 12,
>   T_ARRAY       = 13,
>   T_VOID        = 14,
>   T_ADDRESS     = 15,
>   T_NARROWOOP   = 16,
>   T_METADATA    = 17,
>   T_NARROWKLASS = 18,
>   T_CONFLICT    = 19, // for stack value type with conflicting contents
>   T_ILLEGAL     = 99
> };
> ```
>
> > **BasicType 到 char 之间的转换:**
> >
> > ```c++
> > char type2char_tab[T_CONFLICT+1]=
> > 				{ 0, 0, 0, 0, 'Z', 'C', 'F', 'D', 'B', 'S', 'I', 'J', 'L', '[', 'V', 0, 0, 0, 0, 0};
> > 
> > // --- Basic 转 char
> > inline char type2char(BasicType t) { return (uint)t < T_CONFLICT+1 ? type2char_tab[t] : 0; }
> > 
> > // --- char 转 BasicType
> > inline BasicType char2type(char c) {
> >   switch( c ) {
> >   case 'B': return T_BYTE;
> >   case 'C': return T_CHAR;
> >   case 'D': return T_DOUBLE;
> >   case 'F': return T_FLOAT;
> >   case 'I': return T_INT;
> >   case 'J': return T_LONG;
> >   case 'S': return T_SHORT;
> >   case 'Z': return T_BOOLEAN;
> >   case 'V': return T_VOID;
> >   case 'L': return T_OBJECT;
> >   case '[': return T_ARRAY;
> >   }
> >   return T_ILLEGAL;
> > }
> > ```



### 7.2 Chunk & ChunkPool

> **源码: arena.hpp arena.cpp**
>
> ```c++
> // --- arena.hpp --- Linked list of raw memory chunks
> class Chunk: CHeapObj<mtChunk> {
>  private:
>   Chunk*       _next;     // Next Chunk in list
>   const size_t _len;      // Size of this Chunk
>  public:
>   void* operator new(size_t size, AllocFailType alloc_failmode, size_t length) throw();
>   void  operator delete(void* p);
>   Chunk(size_t length);
> 	//.... 其它方法略
> };
> 
> // --- arena.cpp --- MT-safe pool of chunks to reduce malloc/free thrashing
> class ChunkPool: public CHeapObj<mtInternal> {
>   Chunk*       _first;        // first cached Chunk; its first word points to next chunk
>   size_t       _num_chunks;   // number of unused chunks in pool
>   size_t       _num_used;     // number of chunks currently checked out
>   const size_t _size;         // size of each chunk (must be uniform)
> 
>   // Our four static pools
>   static ChunkPool* _large_pool;
>   static ChunkPool* _medium_pool;
>   static ChunkPool* _small_pool;
>   static ChunkPool* _tiny_pool;
> }
> ```
>
> 



## 8. Thread::set_as_starting_thread(): 创建实际的OS线程

> **对于JVM中定义的Thread以及JavaThread子类，它们只是定义了相关的成员变量及方法，虽然类名中包含"Thread"这个字符串，但它们本身并没有启动新线程并执行代码的作用，`而是需要在操作系统中根据它们的成员变量信息来创建实际的线程来执行，即Thread/JavaThread只是启动实际操作系统线程的信息提供者`**

```c++
bool Thread::set_as_starting_thread() { //源码 thread.cpp
	  return os::create_main_thread((JavaThread*)this);
}

// ---- os_bsd.cpp
bool os::create_main_thread(JavaThread* thread) {
    //主要创建了一个JavaThread关联的OSThread, 而OSThread通过 _thread_id 字段关联了当前线程
	  return create_attached_thread(thread); 
}

bool os::create_attached_thread(JavaThread* thread) {

  OSThread* osthread = new OSThread(NULL, NULL); // 创建空OSThread对象
  osthread->set_thread_id(os::Bsd::gettid()); //设置OSThread对象对应的内核线程id, 也就是当前线程

  // Store pthread info into the OSThread
#ifdef __APPLE__
  uint64_t unique_thread_id = locate_unique_thread_id(osthread->thread_id());
  guarantee(unique_thread_id != 0, "just checking");
  osthread->set_unique_thread_id(unique_thread_id);
#endif
  osthread->set_pthread_id(::pthread_self());

  os::Bsd::init_thread_fpu_state(); //初始化浮点数寄存器

  osthread->set_state(RUNNABLE); //初始化osthread状态为RUNNABLE

  thread->set_osthread(osthread); //设置JavaThread对应的OSThread

  os::Bsd::hotspot_sigmask(thread); // 设置线程 signal mask

  return true;
}
```

