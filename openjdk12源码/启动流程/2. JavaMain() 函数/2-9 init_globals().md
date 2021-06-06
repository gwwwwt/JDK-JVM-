# init_globals() 函数

> **源码: init.cpp**

```c++
	jint init_globals() {
  	HandleMark hm;
  	management_init(); //略
  	bytecodes_init();  //<1> 《2. JavaMain函数/2-9-1 bytecodes_init().md》
  	classLoader_init1(); //<2> 《2. JavaMain函数/2-9-2 classLoader_init1().md》 
  	compilationPolicy_init(); //略
  	codeCache_init();         //略
  	VM_Version_init();				//<3>《2. JavaMain函数/2-9-3 VM_Version_init.md》; ----明天最好把这个整理好
  	os_init_globals();				//略
  	stubRoutines_init1();     //<4. 生成其它Stub>  
  	jint status = universe_init();  // dependent on codeCache_init and
                                  // stubRoutines_init1 and metaspace_init.
  	gc_barrier_stubs_init();   // depends on universe_init, must be before interpreter_init
  	interpreter_init();        // before any methods loaded
  	invocationCounter_init();  // before any methods loaded
  	accessFlags_init();
  	templateTable_init();
  	InterfaceSupport_init();
  	SharedRuntime::generate_stubs();
  	universe2_init();  // dependent on codeCache_init and stubRoutines_init1
  	javaClasses_init();// must happen after vtable initialization, before referenceProcessor_init
  	referenceProcessor_init();
  	jni_handles_init();
		
  	#if INCLUDE_VM_STRUCTS
  			vmStructs_init();
		#endif // INCLUDE_VM_STRUCTS

  	vtableStubs_init();
  	InlineCacheBuffer_init();
  	compilerOracle_init();
  	dependencyContext_init();

  	compileBroker_init();

  	VMRegImpl::set_regName();

  	universe_post_init()

  	stubRoutines_init2(); // note: StubRoutines need 2-phase init
  	MethodHandles::generate_adapters();

  	return JNI_OK;
}`
```
