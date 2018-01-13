# base64编码和他的前辈们

## 什么是编码

什么是码？下面的文字摘自《Code the hidden language of computer hardware and software》

```text
code
3.a. A system of signals used to represent letters or numbers in transmitting messages.
b. A system of symbols, letters, or words given certain arbitrary meanings, used for transmitting messages requiring secrecy or brevity.
4. A system of symbols and rules used to represent instructions to a computer.
—The American Heritage Dictionary of the English Language

In this book, the word code usually means a system for transferring information among people and
machines. In other words, a code lets you communicate.
```

码(code)是用来在人和计算机之间传输信息的系统。人们用码来和计算机交流。

什么是编码?下面的文字摘自 [techtarget](http://searchnetworking.techtarget.com/definition/encoding-and-decoding)

```text
In computers, encoding is the process of putting a sequence of characters (letters, numbers, punctuation, and certain symbols) into a specialized format for efficient transmission or storage. Decoding is the opposite process -- the conversion of an encoded format back into the original sequence of characters.
```

由上面的文字可以提炼出，编码是将一系列字符处理为另一系列字符的过程。下面就来介绍3种和base64有关的编码方式。

## l64a 编码函数

### l64a 描述

The X/Open Portability Guide(XPG Standard) 中定义了方法 `l64a` 和 `a64l`, 但由于年代久远，google 没查到 定义 `l64a` 的XPG Standard文档，只能在 Linux 下输入 `man l64a` 来看下 `l64a` 究竟是做什么的。

```bash
man l64a
NAME
       a64l, l64a - convert between long and base-64

SYNOPSIS
       #include <stdlib.h>

       long a64l(char *str64);

       char *l64a(long value);

   Feature Test Macro Requirements for glibc (see feature_test_macros(7)):

       a64l(), l64a():
           _SVID_SOURCE || _XOPEN_SOURCE >= 500 || _XOPEN_SOURCE && _XOPEN_SOURCE_EXTENDED

DESCRIPTION
       These  functions provide a conversion between 32-bit long integers and little-endian base-64 ASCII strings (of length zero to six).  If the string used as argument for a64l() has length greater than six, only the first six bytes are used.  If the type long
       has more than 32 bits, then l64a() uses only the low order 32 bits of value, and a64l() sign-extends its 32-bit result.

       The 64 digits in the base-64 system are:

              '.'  represents a 0
              '/'  represents a 1
              0-9  represent  2-11
              A-Z  represent 12-37
              a-z  represent 38-63

       So 123 = 59*64^0 + 1*64^1 = "v/".
```

通过上面的描述可以看到 `l64a` 主要作用是：**将32位整形转为 little-endian base-64 ASCII string**. 如果 `long int` 的位数大于32，`l64a` 仅使用低32位。 同理，`a64l`遇到长度大于6的字符串，只用前6个字符。 这种转换可以理解为一种记法：通过该记法，`long int` 可以由至多6个字符的字符串来表示，其中每个字符代表64进制中该位的值。 

通过上面的描述还可以看出， base-64 ASCII string 所使用的字符集(一共64个字符)为: `./ 0-9 a-z A-Z`. 将 `long int` 转为 base-64 其实就将 `long int` 转为64进制数，按从低位到高位的顺序，在字符集找到每位对应的字符，将这些字符拼接起来，即为最终要返回的字符串。 

### l64a glibc 实现

下面我们可以看一下 glibc2.25中 `l64a` 函数的实现方式(`cat stdlib/l64a.c`)：

```c
#include <stdlib.h>

/* Conversion table.  */
static const char conv_table[64] =
{
  '.', '/', '0', '1', '2', '3', '4', '5',
  '6', '7', '8', '9', 'A', 'B', 'C', 'D',
  'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L',
  'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T',
  'U', 'V', 'W', 'X', 'Y', 'Z', 'a', 'b',
  'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j',
  'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r',
  's', 't', 'u', 'v', 'w', 'x', 'y', 'z'
};

char *
l64a (long int n)
{
  unsigned long int m = (unsigned long int) n;
  static char result[7];
  int cnt;

  /* The standard says that only 32 bits are used.  */
  m &= 0xffffffff;

  if (m == 0ul)
    /* The value for N == 0 is defined to be the empty string. */
    return (char *) "";

  for (cnt = 0; m > 0ul; ++cnt)
    {
      result[cnt] = conv_table[m & 0x3f];
      m >>= 6;
    }
  result[cnt] = '\0';

  return result;
}
```

为什么要叫**little-endian base-64 ASCII string**呢？ 从上面的代码也可以看出，低位先计算，先存放在数组中，高位后计算。在最后得到的 ASCII string 中，低位在前，高位在后，所以叫做 **little-endian base-64 ASCII string**.

### 使用 l64a 编码连续二进制流

`l64a` 只能编码32位的输入值。若想编码字符串，或者任意长度的二进制流，我们只能每4个字节处理一次，预先分配好缓冲区，并将处理结果拷贝到缓冲区中。

```c
char *
encode (const void *buf, size_t len)
{
  /* We know in advance how long the buffer has to be. */
  unsigned char *in = (unsigned char *) buf;
  char *out = malloc (6 + ((len + 3) / 4) * 6 + 1);
  char *cp = out;

  /* Encode the length. */
  /* Using `htonl' is necessary so that the data can be
     decoded even on machines with different byte order. */

  cp = mempcpy (cp, l64a (htonl (len)), 6);

  while (len > 3)
    {
      unsigned long int n = *in++;
      n = (n << 8) | *in++;
      n = (n << 8) | *in++;
      n = (n << 8) | *in++;
      len -= 4;
      if (n)
        cp = mempcpy (cp, l64a (htonl (n)), 6);
      else
            /* `l64a' returns the empty string for n==0, so we 
               must generate its encoding ("......") by hand. */
        cp = stpcpy (cp, "......");
    }
  if (len > 0)
    {
      unsigned long int n = *in++;
      if (--len > 0)
        {
          n = (n << 8) | *in++;
          if (--len > 0)
            n = (n << 8) | *in;
        }
      memcpy (cp, l64a (htonl (n)), 6);
      cp += 6;
    }
  *cp = '\0';
  return out;
}
```

http://www.delorie.com/gnu/docs/glibc/libc_82.html

### l64a 缺点分析

1. 容易出错，XPG 标准规定：若`long int` 多于32位， `l64a` 只使用低32位。现代64系统`long int`大多是8个字节，每次遍历4个字节，直接给`l64a`传参 `*(long int*)&buffer[0]` 就错了，只能写做 `*(int*)&buffer[0]`. 另外，如果直接将每4个字节看做一个int,那么在 little-endian 机器上编码出的结果，无法被 big-endian机器解码.
1. 位的利用率不高,假设要编码 12n 个字节，即 96n 位，rfc 4648 编码结果有 16n个字符， l64a 编码结果有 18n个字符。
1. API使用不便，并且也无人维护了。比如XPG标准规定n=0时候返回空字符串，但在连续的字节流中返回空字符串，会导致解码时候n=0的信息丢失。比如没有提供 能编码任意长度二进制流 的方法，连 [glibc的网站](http://www.delorie.com/gnu/docs/glibc/libc_82.html)上都说：那就这样吧。。

>It is strange that the library does not provide the complete functionality needed but so be it.

## base64的前辈：uuencode 编码方法

### uuencode 描述

[Uuencoding](https://en.wikipedia.org/wiki/Uuencoding) 起源于1980年 Mary Ann Horton 写的Unix程序 `uuencode` 和 `uudecode`，是一种 binary-to-text 编码，用来在邮件传输中编码解码二进制数据。Uuencoding 的叫法来自于 Unix-to-Unix encoding，一种能在Unix系统之间安全传输文件的编码方式，并且该编码方式并不依赖于传输中间节点是否为Unix系统。

`uuencode`主要用来将二进制文件编码为“对大多数字符集来说常用的”字符子集，该子集在传输过程几乎不可能被更改或者损坏。 而 `uudecode` 主要用来根据 `uuencode` 编码出的字符来重建二进制文件。

### uuencode 编码规则

一个 uuencode 编码后的文件的格式为：

```uuencode
begin <mode> <filename><newline>
<length character><formatted characters><newline>
<length character><formatted characters><newline>
...
<length character><formatted characters><newline>
`<newline>
end<newline>
```

编码后的文件第一行为 `begin <mode> <filename><newline>`,其中 `<mode>` 为3位八进制数，代表 Unix 文件权限(比如 777,644)，`<filename>`为重建二进制文件的文件名，`<newline>` 表示换行符。

每个数据行的格式都为 `<length character><formatted characters><newline>`，其中 `<length character>` 为长度字符，表示本数据行编码了多少个字节，该长度字符为 `32+实际编码字节` 所对应的 ASCII 字符。**"\`"** 为特殊的长度字符，表示该数据行编码了 0 个字节。 除最后一行外的所有数据行(在整个文件的字节数不能被45整除的情况下，若能整除，最后一行也是45字节)，都只编码 45 字节的数据，即长度字符大多数情况下为 32+45=77 所对应的 ASCII字符 'M'。在每个数据行中，`<formatted characters>` 为二进制数据按编码规则编码出的字符。

二进制数据的编码规则如下：

1. 每3个字符一组，一共24位
1. 将24位分为 4个 6-bit 组，每组代表0-63间的一个数
1. 将每个数的值都加上32
1. 输出每个数加上32后对应的 ASCII字符

若最后一行的字符数不能被3整除，则填充0使其能被3整除，但填充的字节并不计算 `<length character>`,省得解码时候文件末尾多了不必要的空格。

举例，按上述编码规则，单词 love 的编码过程：

<table border="1">
   <tbody>
      <tr>
         <td style="padding:0px;">待编码字符</td>
         <td style="padding:0px;" colspan="8" align="center"><b>l</b></td>
         <td style="padding:0px;" colspan="8" align="center"><b>o</b></td>
         <td style="padding:0px;" colspan="8" align="center"><b>v</b></td>
         <td style="padding:0px;" colspan="8" align="center"><b>e</b></td>
      </tr>
      <tr>
         <td style="padding:0px;">ASCII十进制值</td>
         <td style="padding:0px;" colspan="8" align="center">108</td>
         <td style="padding:0px;" colspan="8" align="center">111</td>
         <td style="padding:0px;" colspan="8" align="center">118</td>
         <td style="padding:0px;" colspan="8" align="center">101</td>
      </tr>
      <tr>
         <td style="padding:1px;">二进制表示</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">0</td>
      </tr>
      <tr>
         <td style="padding:0px;">每组数值</td>
         <td style="padding:0px;" colspan="6" align="center">27</td>
         <td style="padding:0px;" colspan="6" align="center">6</td>
         <td style="padding:0px;" colspan="6" align="center">61</td>
         <td style="padding:0px;" colspan="6" align="center">54</td>
         <td style="padding:0px;" colspan="6" align="center">25</td>
         <td style="padding:0px;" colspan="6" align="center">16</td>
      </tr>
      <tr>
         <td style="padding:0px;">+32后的数值</td>
         <td style="padding:0px;" colspan="6" align="center"><b>59</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>38</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>93</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>86</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>57</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>48</b></td>
      </tr>
      <tr>
         <td style="padding:0px;">对应的ASCII字符</td>
         <td style="padding:0px;" colspan="6" align="center"><b>;</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>&</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>]</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>V</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>9</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>0</b></td>
      </tr>
   </tbody>
</table>

由上表看出，love 被编码为 `;&]V90`，该数据行 length=4, length character 为 ASCII 4+32 对应的字符：**$**。最终 love 的编码结果为： `$;&]V90`

编码后的文件最后2行都为固定格式：

```uuencode
`<newline>
end<newline>
```

我们也可以使用 Python2(2020年不维护了) 验证下 uuencode 的结果是否正确(对比下发现结果正确)：

```bash
python -c 'print "love".encode("uu")'
begin 666 <data>
$;&]V90

end
```

### uuencode 编码实现

下面是 uuencode 编码实现的关键函数(完整版见github [uuencode.c]()):

```c
#define ENC(c) ((c) ? ((c) & 077) + 32: '`')

void encode(FILE *input, FILE *output, char *name, mode_t mode)
{
  int ch, n;
  char *p;
  char buf[80] = {0};
  bool needBreak;

  fprintf(output, "begin %o %s\n", mode & 0777, name);
  while ((n = fread(buf, 1, 45, input))) {
    ch = ENC(n);
    if (fputc(ch, output) == EOF)
      break;
    for (p = buf; n > 0; n -= 3, p += 3) {
            needBreak = false;
      /* Pad with nulls if not a multiple of 3. */
      if (n < 3) {
              needBreak = true;
        p[2] = '\0';
        if (n < 2)
          p[1] = '\0';
      }
      ch = *p >> 2;
      ch = ENC(ch);
      if (fputc(ch, output) == EOF)
        break;
      ch = ((*p << 4) & 060) | ((p[1] >> 4) & 017);
      ch = ENC(ch);
      if (fputc(ch, output) == EOF)
        break;
      ch = ((p[1] << 2) & 074) | ((p[2] >> 6) & 03);
      if (needBreak && ch == 0 )
      {
              break;
      }
      ch = ENC(ch);
      if (fputc(ch, output) == EOF)
        break;
      ch = p[2] & 077;
      if (needBreak && ch == 0 ) break;
      ch = ENC(ch);
      if (fputc(ch, output) == EOF)
        break;
    }
    if (fputc('\n', output) == EOF)
      break;
  }
  if (ferror(input))
    errx(1, "read error");
  fprintf(output, "%c\nend\n", ENC('\0'));
}
```

使用命令 `gcc -g uuencode.c -o uuencode`, 测试下编码结果是否正确：

```bash
[root@localhost]$ printf "love" > love.txt
[root@localhost]$ ./uuencode love.txt doyouknow
begin 664 doyouknow
$;&]V90
`
end

[root@localhost]$ python -c 'print "love".encode("uu")'
begin 666 <data>
$;&]V90空空
空
end

[root@localhost]$ 
```

对比 Python2 的实现，发现 Python2 的倒数第2行以及对于 非3倍数的字符数量的处理并不标准。

### uuencode 缺点

1. 致命缺点，发送的文件经过 [EBCDIC](https://en.wikipedia.org/wiki/EBCDIC) 编码的机器可能会损坏，因为 EBCDIC 有不同的代码页，在不同的代码页(code page)中，同一字符的码位(code point)不同。比如对于字符 **$**，在 [EBCDIC代码页037](https://en.wikipedia.org/wiki/EBCDIC_037) 中码位为91，但在 [EBCDIC代码页285](https://en.wikipedia.org/wiki/EBCDIC_285) 中码位为74。若 `uuencode` 编码结果中包含字符 **$**,先将字符发送到 EBDIC037 的机器上,码位为91，该机器再将码位91发送到 EBCDIC285 的机器上时(该机器为啥不转换码位91位码位74呢？个人猜测 EBCDIC 的年代，有很多粗制滥造的不支持ASCII和SMTP的邮件软件和邮件服务器)，EBCDIC285 机器再将 EBCDIC 转为 ASCII 发送到目标机器，此时目标机器上收到的数据已经是错的。 SO上也有个相关的问题: [why did base64 win against uuencode?](https://retrocomputing.stackexchange.com/questions/3019/why-did-base64-win-against-uuencode)

1. 编码后的文件增大约 **37.7%**, 比base64的 **33%** 要高,测试如下:

    ```bash
    [root@localhost]$ dd if=/dev/zero of=output.dat  bs=1M  count=24
    [root@localhost]$ ./uuencode output.dat test > output_uu
    [root@localhost]$ ll output*
    -rw-rw-r-- 1 root root 25165824 Jan 13 18:58 output.dat
    -rw-rw-r-- 1 root root 34672936 Jan 13 18:52 output_uu
    [root@localhost]$ python -c "print 34672936/25165824.0"
    1.37777868907
    ```

## 主角：base64编码([rfc4648](https://tools.ietf.org/html/rfc4648#ref-3))

### base64编码描述

Base64编码是一种 将二进制数据转为可印刷ASCII字符 的binary-to-text 编码方法。Base64的叫法源于MIME 内容传输编码。Base编码的主要作用是在某些只允许US-ASCII字符的环境下存储或传输数据。

### base64编码规则

**核心思想**：将二进制字节流按输入顺序从左到右排列，每6个二进制位(2 ^ 6 = 64,既不能是5位也不能是7位)处理一次，最后不足6位的补到6位并填充相应数量的填充符号。

**base64字符集**: base64一共用到了65个字符，字母**A-Z**，字母**a-z**，数字**0-9**，2个符号**+/**, 以及填充字符 "="。

二进制字节流是每8位1个字节，base64编码是每6位处理一次，6和8的最小公倍数为24(3个字节)，因此对于输入的二进制序列，按照输入顺序从左到右，**每24位处理一次**，将24个二进制位看作是由4组，每组6个二进制位构成，从左到右计算出每组6个二进制位代表的数值，根据该数值从base64字符集中找到对应的字符，依次输出即可。 

**填充字符情况分析**: 若输入的二进制总位数不能被24整除，则需要补位补到能被24整除，并填充相应数量的填充符号。

1. 余数为8，即多出1个字节。此时需要补充16个字节，构成一个24字节组。该24字节组前12个字节是有意义字节，后12个字节用2个填充符号"="代替
1. 余数为16，即多出2个字节。此时需要补充8个字节。该24字节组前18个字节是有意义字节，后6个字节用1个填充符号"="代替

**编码举例**:假设要对单词 love 进行编码,总位数为32位，前24位一组编码为 **bG92**,还剩下8位，补充16位后，前面12位编码为 **ZQ**, 后面12个字节用 **==** 代替。具体编码过程如下：

<table border="1">
   <tbody>
      <tr>
         <td style="padding:0px;">待编码字符</td>
         <td style="padding:0px;" colspan="8" align="center"><b>l</b></td>
         <td style="padding:0px;" colspan="8" align="center"><b>o</b></td>
         <td style="padding:0px;" colspan="8" align="center"><b>v</b></td>
         <td style="padding:0px;" colspan="8" align="center"><b>e</b></td>
      </tr>
      <tr>
         <td style="padding:0px;">ASCII十进制值</td>
         <td style="padding:0px;" colspan="8" align="center">108</td>
         <td style="padding:0px;" colspan="8" align="center">111</td>
         <td style="padding:0px;" colspan="8" align="center">118</td>
         <td style="padding:0px;" colspan="8" align="center">101</td>
      </tr>
      <tr>
         <td style="padding:1px;">二进制表示</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">1</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">0</td>
         <td style="padding:1px;">0</td>
      </tr>
      <tr>
         <td style="padding:0px;">每组数值</td>
         <td style="padding:0px;" colspan="6" align="center">27</td>
         <td style="padding:0px;" colspan="6" align="center">6</td>
         <td style="padding:0px;" colspan="6" align="center">61</td>
         <td style="padding:0px;" colspan="6" align="center">54</td>
         <td style="padding:0px;" colspan="6" align="center">25</td>
         <td style="padding:0px;" colspan="6" align="center">16</td>
      </tr>
      <tr>
         <td style="padding:0px;">对应的base64字符</td>
         <td style="padding:0px;" colspan="6" align="center"><b>b</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>G</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>9</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>2</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>Z</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>Q</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>=</b></td>
         <td style="padding:0px;" colspan="6" align="center"><b>=</b></td>
      </tr>
   </tbody>
</table>

### base64编码实现

由于base64是每6位一组进行编码，因此只需构造出**位数组**即可，然后遍历该数组，每6个元素计算出对应二进制的值，再去字符集中查找对应的字符。若位数组长度除以24余8，那么只需要往**位数组**末尾插入4个0，在最终结果后面加上**==**即可；若余数为16，那么只需要往**位数组**末尾插入2个0，在最终结果后面加上**=**即可。下面为 Python3 的实现代码：

```python
#!/usr/bin/python3
import sys

alphabet = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'
padding = '='

# 将每个数字转为8个 0,1 列表
def numberList2bitList(numList):
    lst = []
    for num in numList:
        lst.extend([num >> i & 1 for i in range(7,-1,-1)])

    return lst

def mybase64(lst):
    if len(lst) == 0:
        return ''
    paddingStr = ''
    retstr = ''
    # 要补上4个0到lst末尾,并且填充2个=符号
    if len(lst) % 24 == 8:
        lst.extend([0,0,0,0])
        paddingStr = '=='
    # 要补上2个0到lst末尾,并且填充1个=符号
    elif len(lst) % 24 == 16:
        lst.extend([0,0])
        paddingStr = '='
    else:
        pass

    for i in range(int(len(lst)/6)):
        tmp = lst[i * 6 : (i+1)*6]
        num = sum([2 ** x * tmp[5-x] for x in [0,1,2,3,4,5]])
        print(num, end=',')
        retstr += alphabet[num]

    retstr += paddingStr
    return retstr

if __name__=="__main__":
    if len(sys.argv) != 2:
        print("请输入要编码的字符串")
    else:
        print(mybase64(numberList2bitList([ord(x) for x in sys.argv[1]])))
```

### base32编码和base16编码

**base32编码**和**base16编码**其实就是 **base64编码** 的变种，只不过每次处理的位数分别为**5位**和**4位**,字符集也相应减少到 32个 和 16个。填充字符没变，仍为 **"="**.

**base32字符集**: 字母**A-Z** 数字**2-7**
**base16字符集**: 字母**A-F** 数字**0-9**

4和8的最小公倍数为8，因此base16编码不会出现填充字符的情况。 5和8的最小公倍数为40，每40位处理一次，下面分析下**base32编码**出现填充字符的情况：

1. 总位数除以40余8,需要补32位凑成40位，其中前10位是有意义字符，后面30位用6个填充字符，即**======**代替
1. 余数为16，需要补24位，其中前20位是有意义字符，后面20位用4个填充字符，即**====**代替
1. 余数为24，需要补16位，其中前25位是有意义字符，后面15位用3个填充字符，即**===**代替
1. 余数为32，需要补8位，其中前35位是有意义字符，后面5位用**=**代替

### 为啥base64编码被广泛使用

1. 字符集选得好，65个字符都是常用字符，而且基本所有机器都能识别
1. 编码方法简单明了：每6个二进制位处理一次，不足6位的补到6位并填充相应数量的填充符号
1. 比 uuencode 节省空间，提供的接口比 l64a 更完善易用，被 MIME 使用

## References:

1. [glibc l64a 文档](http://www.delorie.com/gnu/docs/glibc/libc_82.html)
1. [rfc4648](https://tools.ietf.org/html/rfc4648#ref-3)
1. [uuencoding](https://en.wikipedia.org/wiki/Uuencoding)
1. [apple open source uuencode.c](https://opensource.apple.com/source/basic_cmds/basic_cmds-55/uuencode/uuencode.c)
1. [为什么Base64取代了 uuencode](https://retrocomputing.stackexchange.com/questions/3019/why-did-base64-win-against-uuencode)
1. [base64,wikipedia](https://en.wikipedia.org/wiki/Base64)