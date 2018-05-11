---
title: Java-Throw和Throws、检测型异常和非检测型异常
date: 2018-05-11 18:26:20
categories: Java
---

# throw 和 throws

## throw

- throw 总是出现在函数体中，用于抛出一个异常，如果没有做相应程序会立即停止运行
- 如果 throw 的是一个检测型异常(Exception是一个检测型异常，IO操作也是检测型异常)，必须使用 try-catch 处理或者使用 throws 交给上一层处理

## throws

- 总是出现在一个方法头中，表明该方法可能会抛出各种异常。
- 如果 throws 的是一个检测型异常，必须在调用这个方法的地方适应 try-catch 处理或者继续向上层抛出异常

总结：不论是 throw 或者是 throws ，看抛出的是否是检测型异常，如果是检测型异常，就必须要在调用的地方进行 try-catch 处理或者抛出给上一层，这个时候往往编译器也会编译不通过，提示你去处理这些异常。对于非检测型异常，不会提示要进行相应的处理，编译也是可以通过的，只是出现了异常时如果不作处理程序会停止运行。

<!-- more -->

# 检测型异常和非检测型异常

## 检测型异常 (Check Exception)

检测型异常，首先会在编译的时候就会被检测出来，必须明确的对这些异常进行相应的处理 (使用 try-catch 或者抛出给上一层处理)，常见的有 IOException，SQLException，InterruptedException 等。

## 非检测型异常 (UnCheck Exception)

非检测型异常，也称之为运行时异常 (RunTimeException)，他有如下的几点需要我们了了解的：

1. 非检测型异常继承于 Exception；
2. 可以不做任何处理，但是出现异常会导致程序终止运行 (出现异常会由 JVM 去处理)；
3. 常见的异常有 NullPointerException、NumberFormatException、ArrayIndexOuOfBoundsException、StringIndexOutOfBoundsException、ClassCastException、UnsupportedOperationException、ArithmeticException (算术错误，典型的是 0 作为除数)、IllegalArgumentException (非法参数，字符串转数字经常出现的异常) 等。