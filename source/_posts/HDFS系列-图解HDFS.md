---
title: 'HDFS系列: 图解HDFS'
date: 2023-02-09 19:54:49
categories: HDFS
tags: [大数据, 分布式, 存储, hdfs]
---

# 前言

最近在学习`HDFS`，看到一篇很有意思的文章，通过漫画的形式来讲解`HDFS`的原理，可以很好地帮助初学者入门。

为什么市面上大多数的教材和技术书籍，要讲内容将的晦涩难懂，入门体验非常差。希望能多一些这种漫画的风趣幽默，并且通俗易懂的书出现，寓教于乐才是学习的高境界。

<!-- more -->
<!-- markdownlint-disable MD041 MD002--> 

# 漫画

## 1 写数据

![img](HDFS%E7%B3%BB%E5%88%97-%E5%9B%BE%E8%A7%A3HDFS/SouthEast.jpeg)

![img](HDFS%E7%B3%BB%E5%88%97-%E5%9B%BE%E8%A7%A3HDFS/SouthEast-20230209220809400.jpeg)

![img](HDFS%E7%B3%BB%E5%88%97-%E5%9B%BE%E8%A7%A3HDFS/SouthEast-20230209220903614.jpeg)

## 2 读数据

![img](HDFS%E7%B3%BB%E5%88%97-%E5%9B%BE%E8%A7%A3HDFS/SouthEast-20230209221028578.jpeg)

## 3 容错

![img](HDFS%E7%B3%BB%E5%88%97-%E5%9B%BE%E8%A7%A3HDFS/SouthEast-20230209221103836.jpeg)

![img](HDFS%E7%B3%BB%E5%88%97-%E5%9B%BE%E8%A7%A3HDFS/SouthEast-20230209221137547.jpeg)

![img](HDFS%E7%B3%BB%E5%88%97-%E5%9B%BE%E8%A7%A3HDFS/SouthEast-20230209221235394.jpeg)

![img](HDFS%E7%B3%BB%E5%88%97-%E5%9B%BE%E8%A7%A3HDFS/SouthEast-20230209221255039.jpeg)

## 4 副本布局

![img](HDFS%E7%B3%BB%E5%88%97-%E5%9B%BE%E8%A7%A3HDFS/SouthEast-20230209221313685.jpeg)

![img](HDFS%E7%B3%BB%E5%88%97-%E5%9B%BE%E8%A7%A3HDFS/SouthEast-20230209221339122.jpeg)

# 引用

- [HDFS Explained as Comics](https://www.mail-archive.com/common-user@hadoop.apache.org/msg15171.html)
- [HDFS 原理讲解漫画 之一 系统构成和写数据过程](https://blog.csdn.net/hudiefenmu/article/details/37655491)
- [HDFS 原理讲解漫画 之二 读数据和容错](https://blog.csdn.net/hudiefenmu/article/details/37694503)
- [HDFS 原理讲解漫画 之三 容错和副本布局策略](https://blog.csdn.net/hudiefenmu/article/details/37820789)
