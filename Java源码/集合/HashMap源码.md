## HashMap源码解析

> **`JDK version: 1.8`**
>
> **子类可扩展的回调方法:**
>
> > **`void afterNodeAccess(Node<K,V> p) { }`**
> > **`void afterNodeInsertion(boolean evict) { }`**
> > **`void afterNodeRemoval(Node<K,V> p) { }`**

### 常量定义

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // 默认容量

static final int MAXIMUM_CAPACITY = 1 << 30; //最大容量

static final float DEFAULT_LOAD_FACTOR = 0.75f; //装载因子

static final int TREEIFY_THRESHOLD = 8; //由链表转换成树的阈值

static final int UNTREEIFY_THRESHOLD = 6; //由树转换成链表的阈值

static final int MIN_TREEIFY_CAPACITY = 64; //链表转换成树的size阈值
```



### 成员变量

```java
transient Node<K,V>[] table;

transient Set<Map.Entry<K,V>> entrySet;

transient int size;

transient int modCount; //改变了KV映射数量的操作或修改了HashMap的内部结构(如 rehash); 用于fail-fast

int threshold; //resize阈值; threshold = capacity * loadFactor

final float loadFactor;
```



### 内部类 Node

```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash; //存储key对应的has值
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() { //获取Node实例的hash值
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) && Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
   
}
```



### 静态工具方法

#### 1. HashMap#hash()

> **之所以要将key.hashCode()结果的高16位异或低16位值, 目的是由于计算table下标时由key hash的低n位决定; 所以能过这个算法将高16位也参与下标计算; **

```java
static final int hash(Object key) {
  	int h;
  	return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```



#### 2. HashMap#tableSizeFor()

> **这是一个比较巧的算法, 由参数cap计算不小于它的最小的2的n次幂值;**
>
> > **算法说明: **
> >
> > > **算法目标: **
> > >
> > > **对于任意一个数, 用二进制表示大概格式为 `0x00100100` 32位; 那不小于它的最小的2的n次幂应该是 `0x00111111 + 1`, 即cap的二进制表示中最高位的1位置开始往后都置为1后再加1; **
> >
> > > 1. **先将 `cap - 1`, 是为了防止如果cap已经是2的n次幂的话, 如果直接按下面的步骤走, 结果会返回`cap*2`;**
> > >
> > > 2. **从 `n |= n >>> 1;` 开始,  到最后的 `n |= n >>> 16;` 为止, 只关心 n 的最高位1的右移和或操作, 低位的1在右移过程中其实没有太大意义;**
> > >
> > > 3. **可以以一个特定的值为例, 但这里不举例了, 参考资料中的内容即可, 这里只说结论;**
> > >
> > >    > **`n | n >>> 1`结果就是将最高位1的低一位也置为1; 同样的 `n | n >>> 2`的结果就是将最高位1的低二、低三位置1; `n | n>>> 4` 以及下面的操作; 最终无论最高位1位于哪一位上, 最终的结果都是会得到从该位开始到最后位都是1的结果; **
> > >
> > > 4. **最终 `n + 1`即为结果(当然还需要判断边界条件)**

```java
static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```



##### 2.1 tableSizeFor()方法应用

> **HashMap带参数构造方法中, 如果指定initialCapacity, 会调用tableSizeFor(initialCapacity)来设置threshold; **
>
> > **按照threshhold成员变量的含义，按道理应该是 `this.threshold = tableSizeFor(initialCapacity) * this.loadFactor;` 但是threshold在put()初始化时还有其它的含义，可以参考 put()方法源码；**
> >
> > **`table[]的初始化被推迟到了put()方法中, 在put()方法中会重新计算threshold;`**

```java
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    
}
```



### HashMap#get() 方法 & containsKey() 方法

> **需要注意的是: 如果get()返回值为null时, 并不一定是没有这种KV, 也可能该key对应的value是null;**
>
> **即: get()方法并不能判断这个Key是否存在, 只能通过containsKey()来判断;**
>
> > **其实 get() 和 containsKey() 都是通过 getNode(hash, key) 的返回值来进行判断的；**
>
> > **在比较key时, 先看hash值是否相同, 再看地址是否相等, 再看equals的返回值;**

```java
public V get(Object key) {
	  Node<K,V> e;
  	return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
      
  	Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    
  	if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
        // always check first node   
      	if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
          	return first;
           
      	if ((e = first.next) != null) {
               
          	if (first instanceof TreeNode)
              	return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                
          	do {
              	if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                  	return e;
            } while ((e = e.next) != null);
        }       
    }
  	return null;
}

public boolean containsKey(Object key) {
  	return getNode(hash(key), key) != null;
}
```



### HashMap#resize()方法

> **初始化或翻倍表大小; **
>
> **如果表为null, 则根据存放在`threshold`变量中的初始化`capacity`的值来分配`table`内存(根据原resize()注释, 在实例化HashMap时, capacity其实是存放在了成员变量`threshold`中, 注意: 并没有capacity成员变量); **
>
> **如果table为null 且 threshold 值为0，即调用 new HashMap()创建实例，则将table size设置为默认值16，对应的threshold 值为 （16 \* 0.75）**
>
> **如果表不为null, 则翻倍扩容;**
>
> > **关于数组长度翻倍之后, 元素e.hash的2个可能索引:**
> >
> > 例如我们从16扩展为32时，具体的变化如下所示：
> > ![这里写图片描述](https://img-blog.csdn.net/20160424144259226)
> > 其中n即表示容量capacity。resize之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：
> > ![这里写图片描述](https://img-blog.csdn.net/20160424144316071)
> > 因此，我们在扩充HashMap的时候，不需要重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”。可以看看下图为16扩充为32的resize示意图：
> > ![这里写图片描述](https://img-blog.csdn.net/20160424144334493)
> > 这个设计确实非常的巧妙，既省去了重新计算hash值的时间，而且同时，由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的bucket了。

```java
final Node<K,V>[] resize() {
       
  	Node<K,V>[] oldTab = table;
      
  	int oldCap = (oldTab == null) ? 0 : oldTab.length;    
    int oldThr = threshold;
  	int newCap, newThr = 0;
       
  	if (oldCap > 0) { //原table已初始化过, 则更新长度;
      	if (oldCap >= MAXIMUM_CAPACITY) { //如果table数组长度已达到最大限制, 更新threshold后直接返回;
          	threshold = Integer.MAX_VALUE;
          	return oldTab;
        }
      	else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; //将table长度和threshold翻倍
        }
		}    
  	else if (oldThr > 0) 
        // table未初始化但threshold大于0, 表示是通过 new HashMap(initialCapacity)构造的实例;
        // 则数组长度存储在构造方法中初始化的threshold成员变量中;
      	newCap = oldThr;

		else { //table未初始化且threshold为0, 表示是通过new HashMap()构造的实例; 则使用默认值;          
      	newCap = DEFAULT_INITIAL_CAPACITY;
      	newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
      
		if (newThr == 0) { //表示在上面的if(oldCap>0)执行块中的else if部分判断为false, 所有在这儿进行处理;
      	float ft = (float)newCap * loadFactor;
      	newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
       
    }
    threshold = newThr; //已确定新数大小newCap, 更新threshold值; 
        
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
      
    table = newTab;
      
    if (oldTab != null) {
      	for (int j = 0; j < oldCap; ++j) {
            
          	Node<K,V> e;
          	if ((e = oldTab[j]) != null) {
              	oldTab[j] = null;
                   
              	if (e.next == null) //如果oldTable[j]对应的链表只有一个节点, 直接将该节点移到newTab对应位置处;
                  	newTab[e.hash & (newCap - 1)] = e;
              	else if (e instanceof TreeNode)
                  	((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
              	else {
                    //对于位于oldTab[j] 处链表中的元素,它们在newTab中的索引值只有两个可能;e.hash或e.hash+oldCap
                    //所以针对oldTab[j]链表, 创建两个链表: low和high; 分别由loHead+loTail和hiHead+hiTail表示
                    //链表操作为: 尾插法;
                  	Node<K,V> loHead = null, loTail = null;
                  	Node<K,V> hiHead = null, hiTail = null;
                  	Node<K,V> next;
                  	do {
                      	next = e.next;
                      	if ((e.hash & oldCap) == 0) { //移到newTab后的索引不会变, 加入low链表中 
                          	if (loTail == null)
                              	loHead = e;
                          	else
                              	loTail.next = e;
                                
                          	loTail = e;
                        }
                      	else { //移到newTab后的索引会变为 next.hash + oldCap, 添加到high链表中;
                          if (hiTail == null)
                            	hiHead = e;
                          else
                            hiTail.next = e;
                          hiTail = e;
                        }
                        
                    } while ((e = next) != null);
                       
                  if (loTail != null) { //low链表放到newTab[j]位置处
                    	loTail.next = null;
                    	newTab[j] = loHead;
                  }
                       
                  if (hiTail != null) { //high链表放到newTab[j + oldCap]位置处
                    	hiTail.next = null;
                    	newTab[j + oldCap] = hiHead;
                  }
                   
                }
            }
        }
        
    }
    
	  return newTab;
}
```



### HashMap#put()方法

> **putVal()方法参数说明：**
>
> > 1. hash: key哈希值
> > 2. key/value
> > 3. **`onlyIfAbsent: 如果为true, 在key已存在的情况下, 不修改其对应的value值`**
> > 4. **`evict: 如果为false, 标识 table 处于 creation time`**
>
> **put() 方法返回值: **
>
> > **如果`key`存在, 返回key对应的旧值; 否则返回null;**
>
> **putVal() 方法基本流程:**
>
> > （1）通过hash值得到所在bucket的下标，如果为null，表示没有发生碰撞，则直接put
> > （2）如果发生了碰撞，则解决发生碰撞的实现方式：链表还是树。
> > （3）如果能够找到该key的结点，则执行更新操作，无需对modCount增1。
> > （4）如果没有找到该key的结点，则执行插入操作，需要对modCount增1。
> > （5）在执行插入操作时，如果bucket中bin的数量超过TREEIFY_THRESHOLD，则要树化。
> > （6）在执行插入操作之后，如果size超过了threshold，这要扩容。

```java
public V put(K key, V value) {
  	return putVal(hash(key), key, value, false, true);    
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
  	Node<K,V>[] tab; Node<K,V> p; int n, i;
    
  	if ((tab = table) == null || (n = tab.length) == 0) //table未初始化, 调用resize()初始化table; 
      	n = (tab = resize()).length;
  
  	if ((p = tab[i = (n - 1) & hash]) == null)
      	tab[i] = newNode(hash, key, value, null); //如果tab对应索引处为null, 则创建Node实例后放到索引处;
  	else {
      	Node<K,V> e; K k;
          
      	if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k)))) //存在对应的key
          	e = p;
      	else if (p instanceof TreeNode)
          	e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
      	else {
          	for (int binCount = 0; ; ++binCount) {
              	if ((e = p.next) == null) { //没有找到key对应的Entry, 则创建新Node并附加到p.next
                  	p.next = newNode(hash, key, value, null);
                  
                    //binCount标识当前链表有几个元素, 如果链表元素长度达到阈值, 则将链表转为红黑树
                  	if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                      	treeifyBin(tab, hash);
                  	break;
                }
                  
                //在链表中找到了key对应的Entry
              	if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                  	break;

              	p = e;
            }
        }
          
      	if (e != null) { //Map中存在key对应的Entry, 则更新value
          	V oldValue = e.value;
               
          	if (!onlyIfAbsent || oldValue == null)
              	e.value = value;
               
          	afterNodeAccess(e);
          	return oldValue;
            
        }
    }
       
  	++modCount;
  	if (++size > threshold)
      	resize();
       
  	afterNodeInsertion(evict);
  	return null;
}
```



### HashMap#treeifyBin() 方法

> **为tab的hash对应索引处的链表转换为红黑树;**
>
> > **涉及到两个阈值: **
> >
> > **`TREEIFY_THRESHOLD` : 索引处的链表长度;**
> >
> > **`MIN_TREEIFY_CAPACITY` : 链表转红黑树的size阈值**
>
> > **根据 put() 方法逻辑, 如果hash对应的索引处的链表长度超过8, 则会调用treeifyBin()方法; 但并不是一定会转为红黑树表示; 而是先判断HashMap中的size数量是否到达64阈值, 如果未达到, 会调用resize()方法重新hash 处理;**

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            //循环将Node链表转为由 hd 和 tl 标识的TreeNode双向链表
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null); //根据e节点信息构造TreeNode实例;
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
          
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
```



### 参考

> [HashMap源码注解(一)之常量定义](https://blog.csdn.net/fan2012huan/article/details/51094454)
>
> [HashMap源码注解(二)之成员变量](https://blog.csdn.net/fan2012huan/article/details/51094525)
>
> [HashMap源码注解(三)之内部数据结构 Node](https://blog.csdn.net/fan2012huan/article/list/2)
>
> [HashMap源码注解(四)之静态工具方法 hash()、tableSizeFor()](https://blog.csdn.net/fan2012huan/article/details/51097331)
>
> [HashMap源码注解(五)之get()方法](https://blog.csdn.net/fan2012huan/article/details/51130576)
>
> [HashMap源码注解(六)之put()方法](https://blog.csdn.net/fan2012huan/article/details/51233378)
>
> [HashMap源码注解(七)之resize()方法](https://blog.csdn.net/fan2012huan/article/details/51233540)

