# JVM Oop

## 工具方法

获取klass类中field在类中的偏移量：

> 源文件：src/hotspot/share/utilities/globalDefinitions_gcc.hpp

```c++

#define offset_of(klass,field) (size_t)((intx)&(((klass*)16)->field) - 16)
```



## oopDesc

> 源文件：src/hotspot/share/oops/oop.hpp

```c++
class oopDesc {
  friend class VMStructs;
  friend class JVMCIVMStructs;
 private:
  volatile markOop _mark;
  union _metadata {
    Klass*      _klass;
    narrowKlass _compressed_klass;
  } _metadata;
  
 
 protected:
  inline oop        as_oop() const { return const_cast<oopDesc*>(this); }
  
 public:
  static int mark_offset_in_bytes()      { return offset_of(oopDesc, _mark); }
  static int klass_offset_in_bytes()     { return offset_of(oopDesc, _metadata._klass); }
  static int klass_gap_offset_in_bytes() {
    assert(has_klass_gap(), "only applicable to compressed klass pointers");
    return klass_offset_in_bytes() + sizeof(narrowKlass);
  }
}
```

### const_cast<oopDesc*>

> **const_cast作用：this返回结果的类型是 const oopDesc*，所以不能直接使用this调用非const成员函数或修改成员变量。const_cast可以将常量指针转为非常量指针，结果仍指向原对象；**

