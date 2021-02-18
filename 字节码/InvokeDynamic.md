# InvokeDynamic字节码

> 注：植田佳奈 ==> 远坂凛

> **原文url：https://www.infoq.com/articles/Invokedynamic-Javas-secret-weapon/**
>
> **本note尝试翻译此篇blog，并可能会结合其它资料进行理解说明。**
>
> > 参考：
> >
> > [JVM中方法调用的实现机制](http://it.deepinmind.com/jvm/2019/07/19/jvm-method-invocation.html)
> >
> > [JVM之动态方法调用：invokedynamic](http://it.deepinmind.com/jvm/2019/07/19/jvm-method-invocation-invokedynamic.html)

## 原本的 invokevirtual、invokestatic、invokeinterface、invokespecial

> - **invokevirtual - the standard dispatch for instance methods**
> - **invokestatic - used to dispatch static methods**
> - **invokeinterface - used to dispatch a method call via an interface**
> - **invokespecial - used when non-virtual (i.e. "exact") dispatch is required**

### 为何会设计以上四种字节码

> **以如下代码为例进行说明（源码和javap反编译结果）：**

```java
public class InvokeExamples {
    public static void main(String[] args) {
        InvokeExamples sc = new InvokeExamples();
        sc.run();
    }

    private void run() {
        List<String> ls = new ArrayList<>();
        ls.add("Good Day");

        ArrayList<String> als = new ArrayList<>();
        als.add("Dydh Da");
    }
}
```

```java
javap -c InvokeExamples.class

public class kathik.InvokeExamples {
  public kathik.InvokeExamples();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class kathik/InvokeExamples
       3: dup
       4: invokespecial #3                  // Method "<init>":()V
       7: astore_1
       8: aload_1
       9: invokespecial #4                  // Method run:()V
      12: return

  private void run();
    Code:
       0: new           #5                  // class java/util/ArrayList
       3: dup
       4: invokespecial #6                  // Method java/util/ArrayList."<init>":()V
       7: astore_1
       8: aload_1
       9: ldc           #7                  // String Good Day
      11: invokeinterface #8,  2            // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z
      16: pop
      17: new           #5                  // class java/util/ArrayList
      20: dup
      21: invokespecial #6                  // Method java/util/ArrayList."<init>":()V
      24: astore_2
      25: aload_2
      26: ldc           #9                  // String Dydh Da
      28: invokevirtual #10                 // Method java/util/ArrayList.add:(Ljava/lang/Object;)Z
      31: pop
      32: return
}
```

#### 1. invokeinterface 和 invokevirtual

> **从上面的反编译结果中可以看到**
>
> > **`ls.add("Good Day")` 会生成 `invokeinterface`字节码。**
> >
> > **`als.add("Dydh Da")` 会生成 `invokevirtual`字节码。**
>
> **原因在于：**
>
> > **`List<String>`是一个接口，无法在编译期确定`add()方法在"vtable"中的位置（即索引或slot）`，所以javac为它生成了`invokeinterface字节码`。**
> >
> > **`ArrayList<String>`是具体的类型，可以在编译期确定`add()方法在"vtable"中的位置`，`即使可能存在子类重写add()方法，也需要到运行期才能确定具体调用方法的情况，但它是将"vtalbe"中对应位置的函数指针指向了子类定义的方法，位置（索引或slot）并不会发生变化`。所以javac为这种`可以在编译期确定slot`的情况生成`invokevirtual`字节码。**

#### 2. invokespecial

> **如果一个方法是通过invokespecial来调用的，那它一定没进行虚方法查找。JVM只会在虚表中的固定位置来查找请求的方法。invokespecial有三种使用场景：私有方法，父类方法，构造方法（在字节码中这个被转成一个叫的方法）。这三种情况下都绝对不会出现虚查找或者重写。**

#### 3. invokestatic

> **类静态方法调用。**

## InvokeDynamic

### Why invokedynamic

> **`invokedynamic字节码`的目标是：处理一种新的方法dispatch - 运行期允许应用层代码决定调用哪个方法。**

### Method Handles

> **每个`invokedynamic`字节都关联一个特殊的`bootstrap method or BSM`。解释器遇到invokedynamic后，会调用该BSM方法。而这个BSM方法会返回一个标识要调用方法的method call site对象实例（实例中包含一个method handle），而这个MethodHandle**

### Method Types

> **一个方法主要由以下四部分组成：**
>
> - Name
> - Signature (including return type)
> - Class where it is defined
> - Bytecode that implements the method
>
> > **在Method Handles API中，由`java.lang.invoke.MethodType`类来标识方法签名。可以调用MethodType.methodType()工厂方法来创建获取MethodType实例。**

#### Method.methodType()方法

> **方法参数：**
>
> > **第一个参数标识方法签名中的返回类型；其它参数标识方法参数类型；举例如下：**
> >
> > ```java
> > // Signature of toString()
> > MethodType mtToString = MethodType.methodType(String.class);
> > 
> > // Signature of a setter method
> > MethodType mtSetter = MethodType.methodType(void.class, Object.class);
> > 
> > // Signature of compare() from Comparator<String>
> > MethodType mtStringComparator = MethodType.methodType(int.class, String.class, String.class);
> > ```

#### MethodHandles.lookup()方法

> **MethodHandles.lookup()查询方法会返回一个"lookup context（查询上下文环境）"，该上下文和实际调用lookup()的方法拥有相同的访问权限。**

> **MethodHandles.lookup()返回的"lookup context对象"有多个以"find"开头的方法，如：findVirtual()、findConstructor()、findStatic等。这些方法会返回实际的method handle实例，但是前提是"look context"实例创建于有访问（调用）目标权限的方法中才行，`并且没有办法可以绕过这一限制，即method handle没有类似于反射中的setAccessible()方法`。**
>
> > **示例：**
> >
> > ```java
> > public MethodHandle getToStringMH() {
> >     MethodHandle mh = null;
> >     MethodType mt = MethodType.methodType(String.class);
> >     MethodHandles.Lookup lk = MethodHandles.lookup();
> > 
> >     try {
> >         mh = lk.findVirtual(getClass(), "toString", mt);
> >     } catch (NoSuchMethodException | IllegalAccessException mhx) {
> >         throw (AssertionError)new AssertionError().initCause(mhx);
> >     }
> > 
> >     return mh;
> > }
> > ```
>
> **上面获取到MethodType表示方法签名，再加上方法name和定义方法的类class，将这些参数传入"lookup context对象"的"findxxx()方法"，即可获取到MethodHandle实例。**

### MethodHandle invoke() & invokeExact()方法

> **invoke() 和 invokeExact() 的区别：**
>
> > **invokeExact()会严格按照传入的参数尝试进行调用。**
>
> > **invoke() 可能会对传入的参数进行处理后再调用方法，处理逻辑如下：**
> >
> > - Primitives will be boxed if required
> > - Boxed primitives will be unboxed if required
> > - Primitives will be widened if necessary
> > - A void return type will be converted to 0 (for primitive return types) or null for return types that expect a reference type
> > - null values are assumed to be correct and passed through regardless of static type

#### 示例

```java
Object rcvr = "a";
try {
    MethodType mt = MethodType.methodType(int.class);
    MethodHandles.Lookup l = MethodHandles.lookup();
    MethodHandle mh = l.findVirtual(rcvr.getClass(), "hashCode", mt);

    int ret;
    try {
        ret = (int)mh.invoke(rcvr);
        System.out.println(ret);
    } catch (Throwable t) {
        t.printStackTrace();
    }
} catch (IllegalArgumentException | NoSuchMethodException | SecurityException e) {
    e.printStackTrace();
} catch (IllegalAccessException x) {
    x.printStackTrace();
}
```



## Method Handles 和 invokedynamic

