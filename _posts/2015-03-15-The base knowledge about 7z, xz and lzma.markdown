---
layout: post
title: 关于lzma、xz和7z的一些概念
categories:
- 总结
tags:
- [7z,lzma,xz,compress,file format]
---


关于lzma、xz和7z的一些概念
========================


算法：
---
1. LZMA（Lempel-Ziv-Markov chain-Algorithm）是Igor Pavlov为7-Zip发明的压缩算法。
2. LZMA2算法是LZMA算法的升级版，修正了一些问题。


文件格式：
---
.lzma、.7z和.xz分别是三种文件格式的后缀名，它们对应三种不同的文件结构。文件结构相当于容器，把LZMA算法压缩后的数据包装起来，然后添加上魔术字、校验码、压缩元信息、文件夹结构等信息。

.xz和.lzma一样，只能压缩一个文件。它们需要和打包工具tar一起使用才能把多个文件压缩成一个文件。
而.7z这种更复杂的文件结构可以包含多个文件或文件夹的压缩数据。

由于.xz的压缩元信息存储在头部，而压缩数据存储在元信息后面，所以.xz格式可以支持流式解压缩。
相反，.7z把压缩元信息存储在尾部，而压缩数据在元信息的前面，所以.7z不适合流式解压缩。


应用程序：
---
1. **7-Zip**是最早的项目，它是windows上类似winrar的一个软件。作者把7-Zip的部分代码开源出来成为[LZMA SDK](http://www.7-zip.org/sdk.html)。7-Zip软件支持的文件格式较多，包括默认的.7z以及.lzma和.xz。
2. **p7zip**是7-Zip移植到 POSIX-like 系统（比如Linux）上的命令行软件。
3. **LZMA Utils**是 POSIX-like 系统上的另一个命令行软件。它基于LZMA SDK开发，只支持.lzma格式。
4. **XZ Utils**是LZMA Utils的继承者。支持.xz和.lzma格式。


文件格式详解：
---
### 1. lzma
**.lzma**是历史遗留的老文件格式，它正在被.xz格式取代。。lzma文件对LZMA压缩数据进行简单的封装，加上13个字节的头部信息。头部结构如下图。注意到它不包含magic number和CRC完整性校验等信息，所以.lzma是个不完备的文件格式。

	![lzma-alone header](/pic/lzma-alone header.png "lzma-alone文件头")

### 2. xz
**.xz**作为.lzma的替代，它的文件结构更复杂，包含的元信息更多。<br> .xz文件可以由多个Stream和Stream Padding组成，但通常只有一个Stream。   
	![xz multi-stream](/pic/xz multi-stream.png "xz文件格式")
<br> 每个Stream的格式如下图：  
	![xz stream](/pic/xz stream.png “xz stream”)
<br> Stream Header包含magic number（FD 37 7A 58 5A 00）、Stream Flags以及CRC校验。其中Stream Flgas是CRC校验的类型。  
	![xz stream header](/pic/xz stream header.png "xz stream header")
<br> Block包含Block Header和压缩数据。Block Header包含了压缩算法（filter）的数目和属性、数据长度以及CRC校验。
	![xz block](/pic/xz block.png “xz block”)
	![xz block header](/pic/xz block header.png "xz block header")
<br> Index作用和结构如下：
	![xz index](/pic/xz index.png “xz index”)
<br> Stream Footer的结构如下。其中Backward Size是Index模块的大小，用来在从后向前处理时快速定位到Index的位置。
	![xz stream footer](/pic/xz stream footer.png "xz stream footer")

### 3. 7z
 **.7z**是一种复杂的文件格式。它支持对多个文件/文件夹的压缩、打包。 对7z文件格式的理解可以参考[https://blog.byneil.com/category/zip-7z/](https://blog.byneil.com/category/zip-7z/)和[http://www.romvault.com/Understanding7z.pdf](http://www.romvault.com/Understanding7z.pdf) 
	![7z file format](/pic/7z.png "7z file format")


参考文献：
---
* 7z
	1. [http://www.fmtz.com/formats/7z--7-zip--file-format/links](http://www.fmtz.com/formats/7z--7-zip--file-format/links)
	2. [http://en.wikipedia.org/wiki/7z](http://en.wikipedia.org/wiki/7z)
	3. [http://www.romvault.com/Understanding7z.pdf](http://www.romvault.com/Understanding7z.pdf)
	4. [http://www.7-zip.org/7z.html](http://www.7-zip.org/7z.html)
	5. [https://blog.byneil.com/category/zip-7z/](https://blog.byneil.com/category/zip-7z/)
* xz
	1. [http://en.wikipedia.org/wiki/XZ_Utils](http://en.wikipedia.org/wiki/XZ_Utils)
	2. [http://en.wikipedia.org/wiki/Xz](http://en.wikipedia.org/wiki/Xz)
	3. [http://tukaani.org/xz/xz-file-format.txt](http://tukaani.org/xz/xz-file-format.txt)
