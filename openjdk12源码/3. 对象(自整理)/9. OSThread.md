# OSThread

> **源码: osThread.hpp**

```c++
// Note: the ThreadState is legacy code and is not correctly implemented.
// Uses of ThreadState need to be replaced by the state in the JavaThread.

enum ThreadState { //OSThread的可能状态
  ALLOCATED,                    // Memory has been allocated but not initialized
  INITIALIZED,                  // The thread has been initialized but yet started
  RUNNABLE,                     // Has been started and is runnable, but not necessarily running
  MONITOR_WAIT,                 // Waiting on a contended monitor lock
  CONDVAR_WAIT,                 // Waiting on a condition variable
  OBJECT_WAIT,                  // Waiting on an Object.wait() call
  BREAKPOINTED,                 // Suspended at breakpoint
  SLEEPING,                     // Thread.sleep()
  ZOMBIE                        // All done, but not reclaimed yet
};


class OSThread: public CHeapObj<mtThread> {
	private:
  		OSThreadStartFunc _start_proc;  // Thread start routine
  		void* _start_parm;              // Thread start routine parameter
  		volatile ThreadState _state;    // Thread state *hint*
 		 	volatile jint _interrupted;     // Thread.isInterrupted state
  
      thread_id_t _thread_id;  //操作系统内核thread id
  
  public:
	  	OSThread(OSThreadStartFunc start_proc, void* start_parm);
  		~OSThread();	
}