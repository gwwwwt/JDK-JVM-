# 字节码初始化: bytecodes_init() 函数

> **源码: bytecodes.cpp**

## 1. 相关类型说明

### 1.1 Bytecodes::Code

> **Java字节码值**
>
> ```c++
> class Bytecodes: AllStatic {
>  	public:
>   		enum Code {
>     			_illegal              =  -1,
> 
>     			// Java语言字节码
>     			_nop                  =   0, // 0x00
>     			_aconst_null          =   1, // 0x01
>     			_iconst_m1            =   2, // 0x02
>         	_iconst_0             =   3, // 0x03
> 					//.... 
>           number_of_java_codes, // java语言字节码数量
>         
>           // JVM层面的字节码
>     			_fast_agetfield       = number_of_java_codes,
>     			_fast_bgetfield       ,
> 					//...
>     			
>     			_fast_aldc            , // --- 处理oop常量的字节码
>     			_fast_aldc_w          ,
>     			_invokehandle         , // --- 处理多态方法调用
>         
>     			number_of_codes       // 总的字节码数量
>   	}
> }
> ```



### 1.2 BasicType

> **源码: globalDefinitions.hpp**
>
> **Java基础数据类型**
>
> ```c++
> enum BasicType {
>   T_BOOLEAN     =  4,
>   T_CHAR        =  5,
>   T_FLOAT       =  6,
>   T_DOUBLE      =  7,
>   T_BYTE        =  8,
>   T_SHORT       =  9,
>   T_INT         = 10,
>   T_LONG        = 11,
>   T_OBJECT      = 12,
>   T_ARRAY       = 13,
>   T_VOID        = 14,
>   T_ADDRESS     = 15,
>   T_NARROWOOP   = 16,
>   T_METADATA    = 17,
>   T_NARROWKLASS = 18,
>   T_CONFLICT    = 19, // for stack value type with conflicting contents
>   T_ILLEGAL     = 99
> };
> ```



## 2. bytecodes_init() 函数

### 2.1 相关成员变量

> **在 Bytecodes 类中定义了下面几个数组类型的静态成员变量:**

```c++
// 可以看到这些数组的长度都是 number_of_codes

static const char* _name          [number_of_codes];
static BasicType   _result_type   [number_of_codes];
static s_char      _depth         [number_of_codes];
static u_char      _lengths       [number_of_codes];
static Code        _java_code     [number_of_codes];
static jchar       _flags         [(1<<BitsPerByte)*2]; // all second page for wide formats
```



### 2.2 工具函数 def()

> **Bytecodes类中定义了重载的def()系方法，用于设置 上面说到的Bytecodes的静态数组成员变量**

```c++
void Bytecodes::def(Code code, const char* name, const char* format, const char* wide_format, 
                    BasicType result_type, int depth, bool can_trap) {
  	def(code, name, format, wide_format, result_type, depth, can_trap, code);
}

// 重载的核心def()方法
/* 参数说明:
code: Bytecodes::Code枚举类型, 即字节码类型
name: 字节码字符串
format: 在 wide_format 参数非NULL的情况下,  format参数也必须非NULL, 否则会报错;
wide_format: format的完整形式
result_type: 
depth:
can_trap:
java_code: 一般情况下与code参数相同
*/
void Bytecodes::def(Code code, const char* name, const char* format, const char* wide_format, 
                    BasicType result_type, int depth, bool can_trap, Code java_code) {
    // -- 获取format 和 wide_format 字符串长度
  	int len  = (format      != NULL ? (int) strlen(format)      : 0);
  	int wlen = (wide_format != NULL ? (int) strlen(wide_format) : 0);
  	_name          [code] = name;
  	_result_type   [code] = result_type;
  	_depth         [code] = depth;
  	_lengths       [code] = (wlen << 4) | (len & 0xF); //
  	_java_code     [code] = java_code;
  	int bc_flags = 0;
  	if (can_trap)           bc_flags |= _bc_can_trap;
  	if (java_code != code)  bc_flags |= _bc_can_rewrite;
  	_flags[(u1)code+0*(1<<BitsPerByte)] = compute_flags(format,      bc_flags);
  	_flags[(u1)code+1*(1<<BitsPerByte)] = compute_flags(wide_format, bc_flags);
}
```



```c++
void bytecodes_init() {
  	Bytecodes::initialize();
}

void Bytecodes::initialize() {
		// <1> : Java字节码命令只有一个字节, 所以命令数量不能超过一个字节的最大值范围
  	assert(number_of_codes <= 256, "too many bytecodes");

  	// initialize bytecode tables - didn't use static array initializers
  	// (such as {}) so we can do additional consistency checks and init-
  	// code is independent of actual bytecode numbering.
  	//
  	// Note 1: NULL for the format string means the bytecode doesn't exist
  	//         in that form.
  	//
  	// Note 2: The result type is T_ILLEGAL for bytecodes where the top of stack
  	//         type after execution is not only determined by the bytecode itself.

  	//  Java bytecodes
  	//  bytecode               bytecode name           format   wide f.   result tp  stk traps
  	def(_nop                 , "nop"                 , "b"    , NULL    , T_VOID   ,  0, false);
  	def(_aconst_null         , "aconst_null"         , "b"    , NULL    , T_OBJECT ,  1, false);
  	def(_iconst_m1           , "iconst_m1"           , "b"    , NULL    , T_INT    ,  1, false);
  	def(_iconst_0            , "iconst_0"            , "b"    , NULL    , T_INT    ,  1, false);
  	def(_iconst_1            , "iconst_1"            , "b"    , NULL    , T_INT    ,  1, false);
  	def(_iconst_2            , "iconst_2"            , "b"    , NULL    , T_INT    ,  1, false);
  	def(_iconst_3            , "iconst_3"            , "b"    , NULL    , T_INT    ,  1, false);
  	def(_iconst_4            , "iconst_4"            , "b"    , NULL    , T_INT    ,  1, false);
  	def(_iconst_5            , "iconst_5"            , "b"    , NULL    , T_INT    ,  1, false);
  	def(_lconst_0            , "lconst_0"            , "b"    , NULL    , T_LONG   ,  2, false);
  	def(_lconst_1            , "lconst_1"            , "b"    , NULL    , T_LONG   ,  2, false);
  	def(_fconst_0            , "fconst_0"            , "b"    , NULL    , T_FLOAT  ,  1, false);
  	def(_fconst_1            , "fconst_1"            , "b"    , NULL    , T_FLOAT  ,  1, false);
  	def(_fconst_2            , "fconst_2"            , "b"    , NULL    , T_FLOAT  ,  1, false);
  	def(_dconst_0            , "dconst_0"            , "b"    , NULL    , T_DOUBLE ,  2, false);
  	def(_dconst_1            , "dconst_1"            , "b"    , NULL    , T_DOUBLE ,  2, false);
  	def(_bipush              , "bipush"              , "bc"   , NULL    , T_INT    ,  1, false);
  	def(_sipush              , "sipush"              , "bcc"  , NULL    , T_INT    ,  1, false);
  	def(_ldc                 , "ldc"                 , "bk"   , NULL    , T_ILLEGAL,  1, true );
		
    // 还有很多def, 这里略去了它们

 	 	_is_initialized = true;
}
```

