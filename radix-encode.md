进制转换角度来实现 base64编码 以及任意 base-n编码

## 进制转换实现base64编码

## n进制的加法

## n进制的乘法

## n进制乘法优化

## 重新实现 base64编码

## 实现比特币 base58编码,base36编码

### The X/Open Portability Guide `l64a` 实现

These  functions provide a conversion between 32-bit long integers and little-endian base-64 ASCII strings (of length zero to six).  If the string used as argument for a64l() has length greater than six, only the first six bytes are used.  If the type long has more than 32 bits, then l64a() uses only the low order 32 bits of value, and a64l() sign-extends its 32-bit result.

https://code.woboq.org/userspace/glibc/stdlib/l64a.c.html

To store or transfer binary data in environments which only support text one has to encode the binary data by mapping the input bytes to characters in the range allowed for storing or transfering. SVID systems (and nowadays XPG compliant systems) provide minimal support for this task.

It is strange that the library does not provide the complete functionality needed but so be it.

http://www.delorie.com/gnu/docs/glibc/libc_82.html

This encoding scheme is not standard. There are some other encoding methods which are much more widely used (UU encoding, MIME encoding). Generally, it is better to use one of these encodings.

### 为什么被 base64 取代了？
uu encode 需要额外的字节。
glibc 并不能支持 任意二进制序列。

## 从进制转换的角度看 base64 编码

## 进制转换的可行性证明

## 进制转换的 base64编码 实现

## 进制转换编码应用：比特币的base58编码,base36,base85编码

## FAQ

1. 为什么要填充=，而非直接用进制转换？
1. 如何解码？看有没有=号，看位数有多少位，是几的倍数？


https://en.wikibooks.org/wiki/Algorithm_Implementation/Miscellaneous/Base64#C++

https://github.com/tkislan/base64/blob/master/base64.h

https://stackoverflow.com/questions/4080988/why-does-base64-encoding-require-padding-if-the-input-length-is-not-divisible-by

http://chimera.labs.oreilly.com/books/1234000001802/ch04.html#public_key_derivation


https://code.tutsplus.com/tutorials/base-what-a-practical-introduction-to-base-encoding--net-27590

https://en.wikipedia.org/wiki/Base36