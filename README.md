---
title: 关于这个网站
date: 2017-09-16 16:20:03
categories:
- 个人
tags:
- 关于
- 作者
---

####  基本信息
这个网站属于个人网站，专注于分布式服务，分布式消息，以及业务架构

![](http://pic23.photophoto.cn/20120530/0020033092420808_b.jpg)

|  表格1 | 表格2 | 表格3 | 表格4 |
|:-------------|:-----------|:--------------|:------------------|
| 分类16666666666666 | 分类2 | 分类3 | 分类4 |
| 分类1 | 分类2 | 分类36666666666666666666 | 分类4777777777777777777 |
| 分类1 | 分类2666666666666666 | 分类3 | 分类4 |

[链接](http://www.baidu.com/s?wd=%E5%9B%BE%E7%89%87&rsv_spt=1&rsv_iqid=0xdc8f92b2000ae205&issp=1&f=3&rsv_bp=0&rsv_idx=2&ie=utf-8&tn=baiduhome_pg&rsv_enter=1&rsv_sug3=7&rsv_sug1=5&rsv_sug7=100&rsv_sug2=0&prefixsug=tupian&rsp=0&inputT=1803&rsv_sug4=2549)

```java
 public static void clear(File file){
     if(file.isDirectory()){
         File[] files = file.listFiles();
         for (File f : files) {
             clear(f);
         }
     }else{
         String name = file.getName();
         if(name.contains(".lastUpdated")){
             file.delete();
         }
     }
 }
```


