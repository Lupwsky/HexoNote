---
title: Android-SpannableStringBuffer和SpannableString
date: 2018-05-21 16:15:08
categories: Android
---

这两个类用于更改文本的内容、样式或者标记。当需要显示一段文字的时候，通常的做法是在布局文件中使用 TextVeiw 来显示，如果需要改变这一段文字，在 Activity 中获取这个控件，然后调用 setText 方法将整端文字给替换掉，如果需要更改字体颜色，样式，也是调用的 TextView 中的相关方法去改变 TextView 的属性，例如 setColor 来改变颜色，setBackgroundColor 来设置背景色。而且这样的方式改变内容或者样式，都会影响到所有的文字。再来设想一个情景，一段话，总共 8 个字符长度，第 1 - 2 个字符颜色为黑色，第 3 - 4 个字符颜色为红色，第 5 - 6 个字符为蓝色，第 7 - 8 个字符为绿色，并且这四个小段的字符都有自己的监听方法，如果使用 TextView 来做，肯定得定义四个 TextView 来实现。使用 SpannableStringBuffer 和 SpannableString 可以轻松得使用一个 TextView 就可以实现，不仅仅是将指定得字符更改样式或者是内容，还可以将文字设置成图片，同样图片也可以设置监听方法。

<!-- more -->

# 方法和参数介绍

## 主要使用的方法

SpanaableStringBuffer 和 SpannableString 相关方法有三个，setSpan()，setSequence()，removeSpan()，主要使用 setSpan()

![IMAGE](Android-SpannableStringBuffer和SpannableString/5F244BD4F9948F0C84CB18FF34D61C9C.png)

## start 和 end 参数

start : Span 开始的位置 (包括这个字符)
end : Span 结束的位置 (不包括这个字符)

start 和 end 确定的是一个左闭右开的范围，例如 start=3，end=7，选中是第 3 个到第 6 个字符，虽然 end=7，但是不包含第 7 个字符，注意字符的计算是从0开始的。

## flag 参数

Spannable.SPAN_EXCLUSIVE_INCLUSIVE : 在 Span 前面输入的字符不应用 Span 的效果，在后面输入的字符应用 Span 效果
Spannable.SPAN_INCLUSIVE_EXCLUSIVE : 在 Span 前面输入的字符应用 Span 的效果，在后面输入的字符不应用 Span 效果
Spannable.SPAN_INCUJSIVE_INCLUSIVE : 在 Span 前后输入的字符都应用 Span 的效果
Spannable.SPAN_EXCLUSIVE_EXCLUSIVE : 前后都不包括

## what 参数

BackgroundColorSpan : 文本背景色
ForegroundColorSpan : 文本颜色
MaskFilterSpan : 修饰效果，如模糊 (BlurMaskFilter) 浮雕
RasterizerSpan : 光栅效果
StrikethroughSpan : 删除线
SuggestionSpan : 相当于占位符
UnderlineSpan : 下划线
AbsoluteSizeSpan : 文本字体 (绝对大小)
DynamicDrawableSpan : 设置图片，基于文本基线或底部对齐。
ImageSpan : 图片
RelativeSizeSpan : 相对大小 (文本字体)
ScaleXSpan : 基于 x 轴缩放
StyleSpan : 字体样式：粗体、斜体等
SubscriptSpan : 下标 (数学公式会用到)
SuperscriptSpan : 上标 (数学公式会用到)
TextAppearanceSpan : 文本外貌 (包括字体、大小、样式和颜色)
TypefaceSpan : 文本字体
URLSpan : 文本超链接
ClickableSpan : 点击事件

# 使用示例-更改颜色

定义一个TextView：

![IMAGE](Android-SpannableStringBuffer和SpannableString/7B2104FA9131D55C11539D13633C9C45.png)

相关代码：

![IMAGE](Android-SpannableStringBuffer和SpannableString/178F1C976B58DC7F3B6294DC91517077.png)

显示效果：

![IMAGE](Android-SpannableStringBuffer和SpannableString/E2398570EA5714F1FE63B4190E49EE4C.png)

# 使用示例-更改文字内容为图片

相关的代码：

![IMAGE](Android-SpannableStringBuffer和SpannableString/A02D5959E24B418BEEBEEE279005F7BB.png)

显示的效果，字符索引为 7 - 9 的字符会被替换成图片：

![IMAGE](Android-SpannableStringBuffer和SpannableString/9184F7817C5690DB8F3CDCA49D4D3E13.png)

# 使用示例-添加点击事件

相关的代码：

![IMAGE](Android-SpannableStringBuffer和SpannableString/CA3E51B7DD310555A5F25D82F33B2B4F.png)

# 点击事件冲突的解决方案

如果使用了 SpannableStringBuffer 或者 SpannableString 设置了点击事件，又再 TextView 上面设置了点击事件，就会产生冲突，问题如下：

1. 默认情况下，点击 ClickableSpan 的文本时会同时触发绑定在 TextView 的监听事件
2. 默认情况下，点击 ClickableSpan 的文本之外的文本时，TextView 会消费该事件，而不会传递给父 View

解决方案：`http://blog.csdn.net/zhaizu/article/details/51038113`