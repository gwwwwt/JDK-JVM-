# 在不同位置分配空间的基类

> 参考：src/hotspot/share/memory/allocation.hpp 中的注释及定义
>
> > ResourceObj：在resource area分配空间（参考resourceArea.hpp）
> >
> > CHeapObj：在C-Heap中分配；
> >
> > StackObj：在栈上分配；
> >
> > AllStatic：For classes used as name spaces；
> >
> > MetaspaceObj：在Metaspace中分配空间；

## 基类 AllocatedObj

> 本Note之后的源码大部分基于：src/hotspot/share/memory/allocation.hpp

```c++
class AllocatedObj { //定义了打印信息接口方法
 public:
  // Printing support
  void print() const;
  void print_value() const;

  virtual void print_on(outputStream* st) const;
  virtual void print_value_on(outputStream* st) const;
};
```

### 栈上分配对象 StackObj

```c++
//重载了new和new[]运算符, 如果使用 'new StackObj'会直接抛出异常
class StackObj : public AllocatedObj { 
 private:
  void* operator new(size_t size) throw();
  void* operator new [](size_t size) throw();
 public:
  void  operator delete(void* p);
  void  operator delete [](void* p);
};
```

### 堆分配对象 CHeapObj

> 涉及到了C++ STL模板，暂略吧

### Metaspace分配对象 MetaspaceObj

> **需要注意的是：MetaspaceObj 没有继承自 AllocatedObj，它也没有其它任何父类；**
>
> 源文件：src/hotspot/share/memory/allocation.hpp

```c++
class MetaspaceObj {
  friend class VMStructs;
  // When CDS is enabled, all shared metaspace objects are mapped
  // into a single contiguous memory block, so we can use these
  // two pointers to quickly determine if something is in the
  // shared metaspace.
  //
  // When CDS is not enabled, both pointers are set to NULL.
  static void* _shared_metaspace_base; // (inclusive) low address
  static void* _shared_metaspace_top;  // (exclusive) high address

 public:
  bool is_metaspace_object() const;
  bool is_shared() const {
    // If no shared metaspace regions are mapped, _shared_metaspace_{base,top} will
    // both be NULL and all values of p will be rejected quickly.
    return (((void*)this) < _shared_metaspace_top && ((void*)this) >= _shared_metaspace_base);
  }
  void print_address_on(outputStream* st) const;  // nonvirtual address printing

  static void set_shared_metaspace_range(void* base, void* top) {
    _shared_metaspace_base = base;
    _shared_metaspace_top = top;
  }
  static void* shared_metaspace_base() { return _shared_metaspace_base; }
  static void* shared_metaspace_top()  { return _shared_metaspace_top;  }

#define METASPACE_OBJ_TYPES_DO(f) \
  f(Class) \
  f(Symbol) \
  f(TypeArrayU1) \
  f(TypeArrayU2) \
  f(TypeArrayU4) \
  f(TypeArrayU8) \
  f(TypeArrayOther) \
  f(Method) \
  f(ConstMethod) \
  f(MethodData) \
  f(ConstantPool) \
  f(ConstantPoolCache) \
  f(Annotations) \
  f(MethodCounters)

#define METASPACE_OBJ_TYPE_DECLARE(name) name ## Type,
#define METASPACE_OBJ_TYPE_NAME_CASE(name) case name ## Type: return #name;

  enum Type {
    // Types are MetaspaceObj::ClassType, MetaspaceObj::SymbolType, etc
    METASPACE_OBJ_TYPES_DO(METASPACE_OBJ_TYPE_DECLARE)
    _number_of_types
  };

  static const char * type_name(Type type) {
    switch(type) {
    METASPACE_OBJ_TYPES_DO(METASPACE_OBJ_TYPE_NAME_CASE)
    default:
      ShouldNotReachHere();
      return NULL;
    }
  }

  static MetaspaceObj::Type array_type(size_t elem_size) {
    switch (elem_size) {
    case 1: return TypeArrayU1Type;
    case 2: return TypeArrayU2Type;
    case 4: return TypeArrayU4Type;
    case 8: return TypeArrayU8Type;
    default:
      return TypeArrayOtherType;
    }
  }

  void* operator new(size_t size, ClassLoaderData* loader_data,
                     size_t word_size,
                     Type type, Thread* thread) throw();
                     // can't use TRAPS from this header file.
  void operator delete(void* p) { ShouldNotCallThis(); }

  // Declare a *static* method with the same signature in any subclass of MetaspaceObj
  // that should be read-only by default. See symbol.hpp for an example. This function
  // is used by the templates in metaspaceClosure.hpp
  static bool is_read_only_by_default() { return false; }
};
```

### 不能实例化的类（即只包含static成员变量和函数）AllStatic

> **AllStatic 同样不是继承自AllocatedObj，也没有其它父类；**
>
> 源文件：src/hotspot/share/memory/allocation.hpp

```c++
class AllStatic { //默认构造函数和析构函数都进行报错, 所以所有继承自AllStatic的子类都不能调用构造函数进行实例化
 public:
  AllStatic()  { ShouldNotCallThis(); }
  ~AllStatic() { ShouldNotCallThis(); }
};
```

