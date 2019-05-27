---
title: Java-JavaAgent技术的基础使用
date: 2019-05-27 14:38:05
categories: Java
---

Java Agent 技术可以在加载类的时候进行拦截并对字节码进行修改, Java Agent 技术可以实现动态的修改代码和替换类的定义而不影响原有程序的逻辑, 例如实现对代码的监控, 捕获代码的执行时间, 添加事务控制等 AOP 功能, 实际的实现方式主要使用在 JDK 1.5 的时候添加的 java.lang.instrument 包中的 API  来实现, 修改字节码通常使用 Javassit 和 ASM 技术实现 

# 参考资料

* [Java Instrumentation](https://blog.csdn.net/productshop/article/details/50623626)
* [Java Agent 简析及开发示例](https://blog.csdn.net/u010039929/article/details/62881743)
* [谈谈 Java Intrumentation 和相关应用](http://fanyilun.me/2017/07/18/%E8%B0%88%E8%B0%88Java%20Intrumentation%E5%92%8C%E7%9B%B8%E5%85%B3%E5%BA%94%E7%94%A8/)
* [Javassist 使用指南(一)](http://erhu.party/2016/11/17/javassist-turorial-1/)
* [Javassist 使用指南(二)](http://erhu.party/2016/11/23/javassist-turorial-2/)
* [Javassist 使用指南(三)](http://erhu.party/2016/11/23/javassist-tutorial-3/)