## java.nio.Buffer



#### 1. 作用

> A container for data of a specific **primitive type**.
>
> 重要的三个属性: 
>
> > 1. capacity: 元素数据; capacity值不能为负数, 且初始化后不能修改; 
> > 2. limit: Buffer中首个不能读或写的元素索引; limit值不能为负数, 且不能大于capacity值; 
> > 3. position: 下个读或写的元素索引; position值不能为负, 且不能大于limit值; 
>
>
> java.nio.Buffer类是一个抽象类; JDK中为每种**非boolean primitive type**实现了一个对应的子类Buffer;
>
> 关于Buffer以及其子类方法中, 主要定义了两种操作, 即**"get/put"**; 
>
> > 1. 相对操作(Relative operations): 从当前position开始, 读/写一个或多个元素并相应增加position值;
> >
> >    对于出现(position>limit)情况时, 读操作抛出**BufferUnderflowException**; 而写操作抛出**BufferOverflowException**; 
> >
> >    对于从Channel对Buffer进行操作时, 通常都是使用相对操作来处理数据; 
> >
> > 2. 绝对操作(Absolute operations): 通过显式传入元素索引进行操作, 此类操作不会影响position值; 
> >
> >    若(传入的索引>limit)情况时, 读/写都会抛出**IndexOutOfBoundsException**; 
> >
> > 3. Marking and resetting: mark用于标识执行reset操作时position会被恢复的索引值; **mark初始值为-1, 若设置mark后, mark值不能为负, 且mark值不能大于position**; 
> >
> >    此外, 若由于position或limit更新为新值, 若新值小于原mark值, 则mark值将被丢弃; 而若mark未设置的情况下执行reset操作, 则会导致抛出**InvalidMarkException**;
> >
> > 4. 其它操作:
> >
> >    > 4.1 clear: 使Buffer重新准备可channel-read或相对写操作; 它会使 limit=capacity 且 position=0;
> >    >
> >    > 4.2 flip: 使Buffer准备为可channel-write或相对读操作; 它会使 limit=position 且 position=0;
> >    >
> >    > 4.3 rewind: 使Buffer重新可读; 它会使position=0, 且不会影响limit值;
>
> >  ***0 <= mark <= position <= limit <= capacity***



#### 2. 实现

```java
//java.nio.Buffer
public abstract class Buffer {  //抽象类;
  // Invariants: mark <= position <= limit <= capacity
    private int mark = -1;
    private int position = 0;
    private int limit;
    private int capacity;

    // Used only by direct buffers
    // NOTE: hoisted here for speed in JNI GetDirectBufferAddress
    long address;
  
    // Creates a new buffer with the given mark, position, limit, and capacity,
    // after checking invariants.
    Buffer(int mark, int pos, int lim, int cap) {       // package-private
        if (cap < 0) throw new IllegalArgumentException("Negative capacity: " + cap);
      
        this.capacity = cap;
        limit(lim);
        position(pos);
        if (mark >= 0) {
            if (mark > pos) //检查mark不能大于postion值;
                throw new IllegalArgumentException("mark > position: ("
                                                   + mark + " > " + pos + ")");
            this.mark = mark;
        }
    }
  
    //...省略成员变量getter
  
    //==== 下面关于Absolute operations
    // 关于更新position/limit 值, 除一般限制check外, 若新值小于mark当前值, 则取消mark状态;
    public final Buffer position(int newPosition) {
        if ((newPosition > limit) || (newPosition < 0))
            throw new IllegalArgumentException();
        position = newPosition;
        if (mark > position) mark = -1;
        return this;
    }
  
    final Buffer limit(int newLimit) {
        if ((newLimit > capacity) || (newLimit < 0))
            throw new IllegalArgumentException();
        limit = newLimit;
        if (position > limit) position = limit;
        if (mark > limit) mark = -1;
        return this;
    }
    //==== Absolute operations end
  
    //mark & reset实现
    public final Buffer mark() {
        mark = position;
        return this;
    }
  
    public final Buffer reset() {
        int m = mark;
        if (m < 0)
            throw new InvalidMarkException(); //非mark状态下执行reset, 则抛出异常;
        position = m;
        return this;
    }
  
  
    //clear & flip & rewind 实现
    //这三个操作均会取消mark状态;
    public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }
  
    public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
  
    public final Buffer rewind() {
        position = 0;
        mark = -1;
        return this;
    }
  
    //check
    public final int remaining() {return limit - position;}
    public final boolean hasRemaining() {return position < limit;}

    //abstract method
    public abstract boolean isReadOnly();
    public abstract boolean hasArray();
    public abstract Object array(); //调用本方法前, 最好先调用hasArray()检查是否包含底层数组;
    public abstract int arrayOffset();
    public abstract boolean isDirect();
  
    //工具方法
    final int nextPutIndex() {                          // package-private
        if (position >= limit)
            throw new BufferOverflowException();
        return position++;
    }
  
    final int nextPutIndex(int nb) {                    // package-private
        if (limit - position < nb)
            throw new BufferOverflowException();
        int p = position;
        position += nb;
        return p;
    }
  
    final int checkIndex(int i) {                       // package-private
        if ((i < 0) || (i >= limit))
            throw new IndexOutOfBoundsException();
        return i;
    }
  
    final int checkIndex(int i, int nb) {               // package-private
        if ((i < 0) || (nb > limit - i))
            throw new IndexOutOfBoundsException();
        return i;
    }
  
    final int markValue() {                             // package-private
        return mark;
    }
    
    final void truncate() {                             // package-private
        mark = -1;
        position = 0;
        limit = 0;
        capacity = 0;
    }
    
    final void discardMark() {                          // package-private
        mark = -1;
    }

    static void checkBounds(int off, int len, int size) { // package-private
        if ((off | len | (off + len) | (size - (off + len))) < 0)
            throw new IndexOutOfBoundsException();
    }
}
```

