---
title: Scala实现64位的MurmurHash函数
date: 2017-01-19 11:20:52
tags:
- scala
- hash函数
---

# 简介
最近使用Spark的GraphX进行一些图计算，GraphX要求每个节点都有唯一的ID；但我们的数据并没有包含唯一的ID，所以需要使用Hash函数将每条数据的信息进行摘要生成唯一ID。  
Hash中，String自己的hashCode()自然是很烂的(字符串每个字符乘以质数再与之前hash相加，如此迭代)，MD5/SHA1之类的算法开销又比较大，Spark图运算的时候节点较多，hash的开销还是蛮可观的。  
经过搜索，发现了MurmurHash算法，具体参考 [维基百科](https://zh.wikipedia.org/wiki/Murmur%E5%93%88%E5%B8%8C) ，总而言之就是高效低碰撞，hadoop/memcached之类都在用。  
Scala API自身是有MurmurHash算法的实现的（[scala.util.hashing.MurmurHash3](http://www.scala-lang.org/api/current/scala/util/hashing/MurmurHash3$.html)），但返回值是int，32位。对于海量数据而言，显然不够用，我们需要64位的。  
于是就写了个scala实现的64位MurmurHash函数，详见下文（没有测试过具体的碰撞性能）。  

# scala实现
```scala
object MurmurHash64 {
  def stringHash(str: String): Long = {
    val data = str.getBytes
    val length = data.length
    val seed = 0xe17a1465
    val m = 0xc6a4a7935bd1e995L
    val r = 47

    var h = (seed & 0xffffffffL) ^ (length * m)

    val length8 = length / 8

    for (i <- 0 until length8) {
      val i8 = i * 8
      var k = (data(i8 + 0) & 0xff).toLong + ((data(i8 + 1) & 0xff).toLong << 8) + ((data(i8 + 2) & 0xff).toLong << 16) + ((data(i8 + 3) & 0xff).toLong << 24) + ((data(i8 + 4) & 0xff).toLong << 32) + ((data(i8 + 5) & 0xff).toLong << 40) + ((data(i8 + 6) & 0xff).toLong << 48) + ((data(i8 + 7) & 0xff).toLong << 56)

      k *= m
      k ^= k >>> r
      k *= m

      h ^= k
      h *= m
    }

    if (length % 8 >= 7)
      h ^= (data((length & ~7) + 6) & 0xff).toLong << 48
    if (length % 8 >= 6)
      h ^= (data((length & ~7) + 5) & 0xff).toLong << 40
    if (length % 8 >= 5)
      h ^= (data((length & ~7) + 4) & 0xff).toLong << 32
    if (length % 8 >= 4)
      h ^= (data((length & ~7) + 3) & 0xff).toLong << 24
    if (length % 8 >= 3)
      h ^= (data((length & ~7) + 2) & 0xff).toLong << 16
    if (length % 8 >= 2)
      h ^= (data((length & ~7) + 1) & 0xff).toLong << 8
    if (length % 8 >= 1) {
      h ^= (data(length & ~7) & 0xff).toLong
      h *= m
    }

    h ^= h >>> r
    h *= m
    h ^= h >>> r

    h
  }
}
```
