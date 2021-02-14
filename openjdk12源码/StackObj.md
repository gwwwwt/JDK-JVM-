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