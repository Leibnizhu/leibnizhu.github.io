---
title: 《深入理解Java虚拟机》 学习笔记(一)——JVM内存结构
date: 2017-05-27T21:31:01+08:00
tags:
- JVM
- Java
- 笔记
image: java.png
---

最近一个月把经典Java书籍《深入理解Java虚拟机》读了一遍，受益匪浅，接下来几篇博客里将会总结一些学习笔记，或许会跟很多现有的博文重复，但主要是为了自己总结一下。  
**JVM笔记系列索引**  
[《深入理解Java虚拟机》 学习笔记(一)——JVM内存结构](/p/深入理解Java虚拟机-学习笔记一JVM内存结构/)  
[《深入理解Java虚拟机》 学习笔记(二)——垃圾回收](/p/深入理解Java虚拟机-学习笔记二垃圾回收/)  
[《深入理解Java虚拟机》 学习笔记(三)——类文件结构](/p/深入理解Java虚拟机-学习笔记三类文件结构/)  
[《深入理解Java虚拟机》 学习笔记(四)——类加载机制与JVM优化](/p/深入理解Java虚拟机-学习笔记四类加载机制与JVM优化/)  
[《深入理解Java虚拟机》 学习笔记(五.终章)——Java内存模型与线程安全/优化](/p/深入理解Java虚拟机-学习笔记五.终章Java内存模型与线程安全/优化/)  
## JVM内存结构
![](1.png)
JVM内存结构不光是只有堆内存和栈内存，实际情况要复杂很多，主要包含以下结构。
### 程序计数器
每个线程都有独立的程序计数器，各线程的互不影响，用于存储正在执行的虚拟机指令地址（对于Native方法则为空undefined）.
### JVM栈
JVM栈是线程私有的，每个方法执行的时候都会建立栈帧，栈帧包含以下内容：
1. 局部变量表：存放编译期可知的基本数据类型数据、对象引用和returnAddress，亦即运行期不会改变局部变量表大小;
2. 操作数栈；
3. 动态链接；
4. 方法出口，等等。

该区域可能抛出以下异常：
1. 当线程请求的栈深度超过最大值，会抛出StackOverflowError异常；
2. JVM栈动态扩展时无法申请导足够内存，抛出OutOfMemoryError异常。

### 本地方法栈
类似JVM栈，区别只在于本地方法栈用于执行本地(Native)方法。
### Java堆
所有线程共享的内存区域，用于存放对象实例（但现在不一定全部对象都在堆里了，栈上分配/标量替换等技术）。在GC的概念中还可以分为Eden区、FromSurvivor区及ToSurvivor区。也可能会划分出线程私有的分配缓冲区TLAB。
### 方法区
线程共享，用于存放已加载的类、常量、静态变量、JIT编译后的代码等数据。  
对于HotSpot虚拟机用户而言，经常将方法区称为永生代（Permanent Generation），是因为HotSpot虚拟机用永生代实现方法区，用GC管理方法区
#### 运行时常量池
运行时常量池是方法区的一部分，类文件被加载后，常量部分就会被放到运行时常量池里。运行期期间也可以将新的常量放入常量池，比如String.intern()方法。
### 直接内存
NIO里面引入直接内存的API，可以使用本地方法分配堆外内存，在某些情况下可以提高IO性能。

## 创建对象过程
1. 遇到new关键字的时候，检查对应类是否能在常量池定位到类的符号引用，并检查是否已加载、解析、初始化。没有的话线加载类；
2. 分配内存。加载类之后一个对象所需的内存大小就确定了；使用Serial、ParNew等收集器时，堆内存是整齐的，使用**指针碰撞**划分内存，即在空闲内存的分界点开始分配指定大小的内存空间；如果用CMS等给予Mark-Sweep算法的收集器时，使用**空闲列表**划分内存，即JVM维护了一个记录可用内存的表，从改变中找一块足够大小的内存空间用于分配。
3. 考虑到多线程同时创建对象的情况，会使用到前面说的TLAB，每个线程在自己的TLAB上分配内存，TLAB用完并重新分配新TLAB的时候才需要同步锁定。
4. 申请内存后，进行初始化零值（可以在TLAB分配时进行）；
5. 设置对象的对象头（Object Header）；
6. 执行&lt;init&gt;方法。
