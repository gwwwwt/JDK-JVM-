# SymbolTable & Symbol 



## 1. SymbolTable类

```c++
//源码: symbolTable.hpp
class SymbolTable : public CHeapObj<mtSymbol> {

  private:
  		static SymbolTable* _the_table; //SymbolTable只有一个对象, 由_the_table指示
  
  		static volatile bool _lookup_shared_first;
  		static volatile bool _alt_hash;

  		// 用于统计信息
  		volatile size_t _symbols_removed;
  		volatile size_t _symbols_counted;

  		SymbolTableHash* _local_table;
  		size_t _current_size;
  		volatile bool _has_work;
  		// Set if one bucket is out of balance due to hash algorithm deficiency
  		volatile bool _needs_rehashing;

  		volatile size_t _items_count;
  		volatile size_t _uncleaned_items_count;

  		SymbolTable(); //默认构造函数是 private的, 所以外部用户不可以使用SymbolTable()来构建对象

	public:
  		// The symbol table
  		static SymbolTable* the_table() { return _the_table; }

  		enum {
    			symbol_alloc_batch_size = 8,
    			symbol_alloc_arena_size = 360*K // TODO (revisit)
  		};

      // ---- 创建SymbolTable的静态方法
  		static void create_table() {
    			_the_table = new SymbolTable();
    			initialize_symbols(symbol_alloc_arena_size);
  		}
};
```





## 2. Symbol类

> **关于Symbol，参考《1. 类的加载.md》**
>
> > ```c++
> > // 参考Symbol::_length_and_refcount字段, 它的低16位记录了Symbol的引用计数;
> > // 而定义 PERM_REFCOUNT 常量的作用: 将字段的低16位设置为本值, 表示本Symbol需要常驻内存, 不需要释放
> > #define PERM_REFCOUNT ((1 << 16) - 1)
> > ```

```c++
// ---- Symbol 类; 源码: symbol.hpp
class Symbol : public MetaspaceObj {
  
  public:
  		static size_t _total_count; //静态字段
  		Symbol() { } //空的默认构造函数只用于在栈上创建一个dummy symbol对象, 来获取它的vtable指针

 	private:
  
  		// 这里是在Symbol类中定义的引用计数; 
  		// 这是一个32位int值, 记录了2个值: 1. 高16位表示本Symbol表示的字符串长度; 2. 低16位表示引用计数 refcount
  		volatile uint32_t _length_and_refcount;
  
  		short _identity_hash; //identity hash值
  		u1 _body[2];

  		enum {
    			// 上面 _length_and_refcount 字段的说明, 字符串长度是16位, 所以字符串长度最大值就是 (1<<16)-1 
    			max_symbol_length = (1 << 16) -1
  		};

  		static int byte_size(int length) {
    			return (int)(sizeof(Symbol) + (length > 2 ? length - 2 : 0));
  		}

 public:
  		// Low-level access (used with care, since not GC-safe)
  		const u1* base() const { return &_body[0]; }

  		unsigned identity_hash() const {
    			unsigned addr_bits = (unsigned)((uintptr_t)this >> (LogMinObjAlignmentInBytes + 3));
    			return ((unsigned)_identity_hash & 0xffff) |
           		((addr_bits ^ (length() << 8) ^ (( _body[0] << 8) | _body[1])) << 16);
  		}

  		char char_at(int index) const {
    			assert(index >=0 && index < length(), "symbol index overflow");
    			return (char)base()[index];
  		}

  		const u1* bytes() const { return base(); }
};
```