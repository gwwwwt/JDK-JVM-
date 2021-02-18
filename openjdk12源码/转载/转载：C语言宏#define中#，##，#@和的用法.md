# 转载： C语言宏#define中#，##，#@和\的用法

> 参考：https://blog.csdn.net/yishizuofei/article/details/81022590

## 一、（#）字符串化操作符

**作用：将宏定义中的传入参数名转换成用一对双引号括起来参数名字符串。其只能用于有传入参数的宏定义中，且必须置于宏定义体中的参数名前。**

如：

```c
#define example( instr )  printf( "the input string is:\t%s\n", #instr )
#define example1( instr )  #instr
```

当使用该宏定义时：

```c
example( abc ); // 在编译时将会展开成：printf("the input string is:\t%s\n","abc")
string str = example1( abc );  // 将会展成：string str="abc"
```

**注意， 对空格的处理：**

a. 忽略传入参数名前面和后面的空格。

```c
如：str=example1(   abc );//将会被扩展成 str="abc"
```

b.当传入参数名间存在空格时，编译器将会自动连接各个子字符串，用每个子字符串之间以一个空格连接，忽略剩余空格。

```c
如：str=exapme( abc    def);//将会被扩展成 str="abc def"
```

## 二、 （##）符号连接操作符

**作用：将宏定义的多个形参转换成一个实际参数名。** 
如：

```c
#define exampleNum( n )  num##n
```

使用：

```c
int num9 = 9;
int num = exampleNum( 9 ); // 将会扩展成 int num = num9
```

**注意：**

a. 当用##连接形参时，##前后的空格可有可无。

```c
如：  #define exampleNum( n )       num ## n                 
// 相当于 #define exampleNum( n )      num##n
```

b. 连接后的实际参数名，必须为实际存在的参数名或是编译器已知的宏定义。

**c. 如果##后的参数本身也是一个宏的话，##会阻止这个宏的展开。**

```c
#include <stdio.h>
#include <string.h>

#define STRCPY(a, b)   strcpy(a ## _p, #b)
int main()
{
    char var1_p[20];
    char var2_p[30];
    strcpy(var1_p, "aaaa");
    strcpy(var2_p, "bbbb");
    STRCPY(var1, var2);
    STRCPY(var2, var1);
    printf("var1 = %s\n", var1_p);
    printf("var2 = %s\n", var2_p);

    //STRCPY(STRCPY(var1,var2),var2);
    //这里是否会展开为： strcpy(strcpy(var1_p,"var2")_p,"var2“）？答案是否定的：
    //展开结果将是：  strcpy(STRCPY(var1,var2)_p,"var2")
    //## 阻止了参数的宏展开!如果宏定义里没有用到 # 和 ##, 宏将会完全展开
    // 把注释打开的话，会报错:implicit declaration of function 'STRCPY'
    return 0;
}  
```

结果：

```c
var1 = var2
var2 = var1
```

## 三、 （#@）单字符化操作符

**作用：将传入单字符参数名转换成字符，以一对单引用括起来。**

如：

```c
#define makechar(x)    #@x
char a = makechar( b ); //展开后变成了：a = 'b';
```

## 四、（\） 续行操作符

**作用：当定义的宏不能用一行表达完整时，可以用”\”表示下一行继续此宏的定义。**

**注意 \ 前留空格。**

**当宏参数是另一个宏的时候，需要注意的是凡宏定义里有用’#’或’##’的地方宏参数是不会再展开。**

**1、非#和##的情况**

```c
#define TOW       (2) 
#define MUL(a,b) (a*b) 
printf("%d*%d=%d\n", TOW, TOW, MUL(TOW,TOW)); 
//这行的宏会被展开为： 
printf("%d*%d=%d\n", (2), (2), ((2)*(2))); 
//MUL里的参数TOW会被展开为(2). 
```

**2、当有#或##的时候**

```c
#define A           2
#define STR(s)      #s 
#define CONS(a,b)   (int)a##e##b
printf("int max: %s\n",   STR(INT_MAX));     // INT_MAX ＃include <limits>
//这行会被展开为： 
//printf("int max: %s\n", "INT_MAX"); 
printf("%s\n", CONS(A, A));                // compile error  
//这一行则是： 
//printf("%s\n", int(AeA)); 
//INT_MAX和A都不会再被展开, 然而解决这个问题的方法很简单. 加多一层中间转换宏.
//加这层宏的用意是把所有宏的参数在这层里全部展开, 那么在转换宏里的那一个宏(_STR)就能得到正确的宏参数. 
#define A            2
#define _STR(s)      #s 
#define STR(s)       _STR(s)           // 转换宏 
#define _CONS(a,b)   (int)a##e##b 
#define CONS(a,b)    _CONS(a,b)        // 转换宏 
printf("int max: %s\n", STR(INT_MAX)); // INT_MAX,int型的最大值,＃include <limits>
//输出为: int max: 2147483647 
//STR(INT_MAX) -->   _STR(2147483647) 然后再转换成字符串； 
printf("%d\n", CONS(A, A)); 
//输出为：200 
//CONS(A, A)  -->  _CONS(2, 2) --> (int)2e2 
```