# 编译第一步--编译预处理

## 曾经的 hello world 程序

```c
#include <stdio.h>
int main()
{
    printf("hello world");
    return 0;
}
```

使用gcc7.3.0(其他版本命令一样)如下命令编译并运行hello world 程序:

```bash
giant@gcc:~$ gcc helloworld.c -o helloworld && ./helloworld
hello world
```

从 helloworld.c 到 helloworld 到底发生了什么呢?

```bash
giant@gcc:~$ gcc -v --save-temps helloworld.c -o helloworld
/usr/local/libexec/gcc/x86_64-pc-linux-gnu/7.3.0/cc1 -E -quiet -v -imultiarch x86_64-linux-gnu helloworld.c -mtune=generic -march=x86-64 -fpch-preprocess -o helloworld.i
/usr/local/libexec/gcc/x86_64-pc-linux-gnu/7.3.0/cc1 -fpreprocessed helloworld.i -quiet -dumpbase helloworld.c -mtune=generic -march=x86-64 -auxbase helloworld -version -o helloworld.s
/usr/local/libexec/gcc/x86_64-pc-linux-gnu/7.3.0/collect2 -plugin /usr/local/libexec/gcc/x86_64-pc-linux-gnu/7.3.0/liblto_plugin.so -plugin-opt=/usr/local/libexec/gcc/x86_64-pc-linux-gnu/7.3.0/lto-wrapper -plugin-opt=-fresolution=helloworld.res -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s -plugin-opt=-pass-through=-lc -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s --eh-frame-hdr -m elf_x86_64 -dynamic-linker /lib64/ld-linux-x86-64.so.2 -o helloworld /usr/lib/x86_64-linux-gnu/crt1.o /usr/lib/x86_64-linux-gnu/crti.o /usr/local/lib/gcc/x86_64-pc-linux-gnu/7.3.0/crtbegin.o -L/usr/local/lib/gcc/x86_64-pc-linux-gnu/7.3.0 -L/usr/local/lib/gcc/x86_64-pc-linux-gnu/7.3.0/../../../../lib64 -L/lib/x86_64-linux-gnu -L/lib/../lib64 -L/usr/lib/x86_64-linux-gnu -L/usr/local/lib/gcc/x86_64-pc-linux-gnu/7.3.0/../../.. helloworld.o -lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed /usr/local/lib/gcc/x86_64-pc-linux-gnu/7.3.0/crtend.o /usr/lib/x86_64-linux-gnu/crtn.o

giant@gcc:~$ ls helloworld*
helloworld  helloworld.c  helloworld.i  helloworld.o  helloworld.s
```

由上述命令可以看到，编译 helloworld.c 生成了 helloworld.i 等中间文件.接下来我们就来看看 helloworld.i 文件的内容以及生成过程.

## The C Preprocessor(cpp)

C预处理器是c/c++编程语言使用的宏预处理器，提供的功能有：头文件的包含，宏定义展开，条件编译，行控制(不常用,暂且忽略)等。使用gcc编译helloworld.c时,gcc调用cc1程序来进行预处理,生成中间文件 helloworld.i.

C语言标准规定，预处理是指前4个编译阶段:

1. 三字符组与双字符组的替换
1. 行拼接: 将用反斜线+换行符连接的代码行拼接成一行
1. token化: 处理每行的空白、注释等，使每行成为token的顺序集。
1. 宏定义展开与预处理指令处理.

## cpp 生成中间文件(helloworld.i)的含义

打开 helloworld.i,我们可以看到如下内容:

```bash
giant@gcc:~$ head helloworld.i 
# 1 "helloworld.c"
# 1 "<built-in>"
# 1 "<command-line>"
# 31 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 32 "<command-line>" 2
# 1 "helloworld.c"
# 1 "/usr/include/stdio.h" 1 3 4
# 27 "/usr/include/stdio.h" 3 4
# 1 "/usr/include/x86_64-linux-gnu/bits/libc-header-start.h" 1 3 4
```

其中以符号#开头的行含义是什么呢?

>\# linenum filename flags

根据gnu的官方文档, 上述格式以#开头的行被称为行标记符，行标记符只会单独一行出现，不会被插入到字符串中.行标记符的含义是:接下来一行来源于*filename*的第*linenum*行，其中*filename*不包含任何不能打印的字符,因为不可打印字符会被八进制转义序列替代。

*flags*有4种可能，分别是 '1', '2', '3', '4'. 如果有多个 flag,那么这多个flag就**从小到大，从左到右排列，中间用空格分隔**。4种*flags*的含义分别是:

* '1' 表示新文件的开始
* '2' 表示返回到某文件
* '3' 表示接下来的代码来自系统头文件，因此某些警告可以被忽视或者不提示
* '4' 表示接下来的代码应当被当做包含在 extern "C" 块中

### built-in 和 command-line 的含义

`<built-in>` 和 `<command-line>` 是虚拟文件(pesudo-files),分别表示内置宏定义的集合，命令行宏定义的集合。比如宏 `_WIN32` 或 `__unix__`。使用命令 `gcc -E -dD helloworld.c` 可以查看 built-in 头文件中所包含的宏定义；使用命令 `gcc -E -dD -DTEST=1 helloworld.c` 可以看到当 command-line 头文件下多了 `#define TEST 1` 这条宏定义.

## 常见的预编译指令

```c
#include <iostream>

#define MUL(x,y) ((x) * (y))

// if, else
#ifdef TEST
#define HELLO "ifdef hello"
#else
#define HELLO "else hello"
#endif

// if,else if , else
#if defined(WORLD)
#define WORLD "def-world"
#warning "def-world"
#elif defined(WORLD1)
#define WORLD "def-world1"
#warning "def-world1"
#else
#define WORLD "def-world-else"
#warning "def-world-else"
#endif

// 宏定义一段代码
#define SAY_HELLO(str) \
do {\
std::cout << str << std::endl;\
} while (0)

// 将常量转换为字符串
#define TOSTR(var) #var
// m is macro-expanded before MACRO_TOSTR is macro-expanded
// https://gcc.gnu.org/onlinedocs/cpp/Stringizing.html#Stringizing
// https://gcc.gnu.org/onlinedocs/cpp/index.html#SEC_Contents
#define MACRO_TOSTR(m) TOSTR(m)
#define num 4

// 将2个标识符拼接在一起
#define CONCAT(s1, s2) s1 ## s2

int main()
{
    SAY_HELLO(CONCAT(HE, LLO));
    SAY_HELLO(WORLD);
    SAY_HELLO(__FILE__);
    SAY_HELLO(MACRO_TOSTR(num));
    return 0;
}

```

## References

1. [Wikipedia, the c preprocessor](https://en.wikipedia.org/wiki/C_preprocessor)
2. [gnu cpp preprocessor output](https://gcc.gnu.org/onlinedocs/cpp/Preprocessor-Output.html#Preprocessor-Output)
3. [cpp first four lines](https://gcc.gnu.org/ml/gcc-help/2011-10/msg00006.html)