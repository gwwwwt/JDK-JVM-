# os::init_2()

> **源码: os_bsd.cpp**

> 1. **设置信号处理函数**
> 2. **设置FD资源限制**
> 3. **设置退出时回调函数**

```c++
jint os::init_2(void) { //本方法在全局arguments解析完成后调用

  os::Posix::init_2(); //记日志

  // 初始化suspend/resume (SR) 信号; 信号number由环境变量'_JAVA_SR_SIGNUM'指定; 
  // 并且指定的这个信号number必须大于 SIGSEGV 和 SIGBUS;
  // 对应的信号处理函数为: os_bsd.cpp中定义的SR_handler()函数
  if (SR_initialize() != 0) {
    perror("SR_initialize failed");
    return JNI_ERR;
  }

  // 初始化两个全局的sigset: static sigset_t unblocked_sigs, vm_sigs; 
  // 根据不同的条件往这两个信号集中添加相应的信息
  Bsd::signal_sets_init();
  
  //注册信息处理函数, 貌似 SIGSEGV, SIGPIPE, SIGBUS, SIGILL, SIGFPE, SIGXFSZ 这几个信息都设置使用默认处理函数
  Bsd::install_signal_handlers();
  
  // Initialize data for jdk.internal.misc.Signal; 略
  if (!ReduceSignalUsage) {
    jdk_misc_signal_init();
  }

  // Check and sets minimum stack sizes against command line options... 略
  if (Posix::set_minimum_stack_sizes() == JNI_ERR) {
    return JNI_ERR;
  }

  // ---- 设置FD数量限制, 对于MacOS来说, 这个限制会被设置为系统默认限制和10240 两个数的较小值
  if (MaxFDLimit) {
    struct rlimit nbr_files;
    int status = getrlimit(RLIMIT_NOFILE, &nbr_files);
    if (status = 0) {
      nbr_files.rlim_cur = nbr_files.rlim_max;

#ifdef __APPLE__
      nbr_files.rlim_cur = MIN(OPEN_MAX, nbr_files.rlim_cur);
#endif
      status = setrlimit(RLIMIT_NOFILE, &nbr_files);

    }
  }

  if (PerfAllowAtExitRegistration) { //atexit() 设置退出时回调函数, 这里设置了 perMemory_exit_helper 函数
    if (atexit(perfMemory_exit_helper) != 0) {
      warning("os::init_2 atexit(perfMemory_exit_helper) failed");
    }
  }

  prio_init(); // --- 初始化线程优先级策略

#ifdef __APPLE__ //----- 略
  // dynamically link to objective c gc registration
  void *handleLibObjc = dlopen(OBJC_LIB, RTLD_LAZY);
  if (handleLibObjc != NULL) {
    objc_registerThreadWithCollectorFunction = (objc_registerThreadWithCollector_t) dlsym(handleLibObjc, OBJC_GCREGISTER);
  }
#endif

  return JNI_OK;
}
```
