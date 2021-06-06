# Handle 类体系

> **类注释: **
>
> ```c++
> //----------------------------------------------------------------------------------------------------------
> // In order to preserve oops during garbage collection, they should be
> // allocated and passed around via Handles within the VM. A handle is
> // simply an extra indirection allocated in a thread local handle area.
> //
> // A handle is a value object, so it can be passed around as a value, can
> // be used as a parameter w/o using &-passing, and can be returned as a
> // return value.
> //
> // oop parameters and return types should be Handles whenever feasible.
> //
> // Handles are declared in a straight-forward manner, e.g.
> //
> //   oop obj = ...;
> //   Handle h2(thread, obj);      // allocate a new handle in thread
> //   Handle h3;                   // declare handle only, no allocation occurs
> //   ...
> //   h3 = h1;                     // make h3 refer to same indirection as h1
> //   oop obj2 = h2();             // get handle value
> //   h1->print();                 // invoking operation on oop
> //
> // Handles are specialized for different oop types to provide extra type
> // information and avoid unnecessary casting. For each oop type xxxOop
> // there is a corresponding handle called xxxHandle.
> ```
>
> 