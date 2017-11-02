---
title: Charset and Character Encoding
tag: 'encoding, ascii, unicode, utf-8, utf-16, utf-32'
date: 2017-08-03 21:25:32
tags:
---


Opening a document downloaded from Internet usually displays garbled code, most of which are caused by  opening with an incorrect encoding. You may hear many encoding character sets, such as ASCII, Unicode, UTF-8, UTF-16 and so on. Most of us will be confused by these encoding and this article will introduce the details of some encodings and their differences between them.

<!--more-->
## ASCII

ASCII  abbreviated **A**merican **S**tandard **C**ode for **I**nformation **I**nterchange is a character encoding standard.

![asciifull](/images/character-encoding/asciifull.gif)

It uses 7 bits to devote a character and thus there are 128 (2^7) characters from 0x00 to 0x7F. The standard ASCII contains only the lowercase letters a-z, uppercase letters A-Z,  numbers 0-9, basic punctuations and some control codes.

## GB2312, GBK

Ascii can meet most of demands when the computer only widely used in the America and the some western countries. But how to represent the Chinese characters became a big problem when Chinese want to use computer too. And thus some encoding stardard methods were developed. GB2312 is backward compatible with ASCII and it uses one byte (8 bits with leading zero) is same as ASCII but two bytes greater than 127 (each byte with leading 1) to represent Chinese characters. It totally contains 7445 characters including 6763 Chinese characters and 682 other characters. Though GB2312 can meet 99.75% usages of Chinese characters, it doesn't support some uncommon characters and GBK was developed for supporting more characters.

## Unicode

With the populariry of computer in the world, each country developed a individual encoding method for supporting their language like gb2312. But there are not compatible and it can be a big problem when it need exchange information on Internet.

Therefore, an character set including all languages is need. Then Unciode came which assigns each character a unique number called **code point**. The latest version of Unicode (Unicode 10.0) contains 136,755 characters.

Unicode can be implemented by different character encodings such as UTF-8, UTF-16, UTF-32, UCS-2 and so on.

### UTF-8

UTF-8 is a variable-length character encoding capable of encoding all possible Unicode code points. It was designed to be backward compatible with ASCII and to avoid the complications of endiness (Big-end or Little-end).

| Number of bytes | Code points range |        Bytes representation         |
| :-------------: | :---------------: | :---------------------------------: |
|        1        |  U+0000 ~ U+007F  |               0XXXXXX               |
|        2        |  U+0080 ~ U+07FF  |          110XXXXX 10XXXXXX          |
|        3        |  U+0800 ~ U+FFFF  |     1110XXXX 10XXXXXX 10XXXXXX      |
|        4        | U+10000 ~ U+10FFF | 11110XXX 10XXXXXX 10XXXXXX 10XXXXXX |

### UCS-2

UCS-2 simply use two bytes for each character and thus it can only encode 65536 code points, the so-called Basic Multilingual Plane (BMP). And thus many Unicode characters are beyond the reach of UCS-2.

### UTF-16

UTF=16 extends UCS-2 by using the same 16 bits as UCS-2 and use 32 bits for the others planes. Since UTF-16 use 2 bytes as an unit and thus the byte order matters. To recoginize the byte order of code unit, UTF-16 allows a **Byte Order Mark (BOM)** to precede the first actual coded value. The BOM with code point value U+FEFF is an invisible zero-width non-breaking space character. The BOM in UTF-16BE (UTF-16 Big Endian) is 0xFEFF but 0xFFFE in UTF-16LE (UTF-16 Little Endian). Older Windows NT system use UCS-2 and UTF-16 is used for text in OS API in Microsoft Windows 2000/XP/2003/7/8.

### UTF-32

UTF-32 is a fixed length encoding which uses 4 bytes to represente a character. The main advantage of UTF-82 is that the Unicode character is indexable since it is all 4 bytes aligned but it is NOT space efficient since characters beyond BMP are rarely used. And same as UTF-16, UTF-32 do NOT have endianness defined.
