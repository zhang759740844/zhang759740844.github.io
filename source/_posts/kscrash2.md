title: iOS Crash 收集框架 KSCrash 源码解析(中)
date: 2019/11/1 14:07:12  
categories: iOS
tags: 

 - 学习笔记
 - 源码解析
---

前一篇看了如何捕获各种类型的异常，这一篇将探究如何**解析异常**。这一篇的理解有一定的难度，你需要提前对 Mach-O 的结构有一个大体上的了解。其中涉及到如何将 crash 的地址 symbolicate 的过程，步骤类似于 fishhook，可以翻阅我之前对 fishhook 的解析。

<!--more-->

