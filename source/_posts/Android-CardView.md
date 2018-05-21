---
title: Android-CardView
date: 2018-05-21 15:47:39
categories: Android
---

CardView 是 Android5.0 之后引入的，用它可以实现卡片布局圆角效果、阴影效果。

# 引入依赖包

```ini
// gradle导入CardView支持包
compile 'com.android.support:cardview-v7:25.0.1'
```

<!-- more -->

# 布局文件中使用

```xml
<android.support.v7.widget.CardView
    android:layout_width="match_parent"
    android:layout_height="150dp"
    android:layout_marginLeft="10dp"
    android:layout_marginRight="10dp"
    android:layout_marginTop="10dp"
    app:cardBackgroundColor="#FF8800"
    app:cardCornerRadius="10dp"
    app:cardElevation="8dp">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_gravity="center"
        android:gravity="center"
        android:text="测试文本"
        android:textColor="@android:color/white"
        android:textSize="72sp" />
</android.support.v7.widget.CardView>
```

# 其他的属性

cardBackgroundColor 设置背景颜色，此处直接设置background是不生效的
cardCornerRadius 设置圆角边大小
cardElevation 阴影大小
cardMaxElevation 最大的阴影大小
cardPreventCornerOverlap 在v20和之前的版本中添加内边距，这个属性是为了防止卡片内容和边角的重叠
cardUseCompatPadding 设置内边距，v21+的版本和之前的版本仍旧具有一样的计算方式
contentPadding 内边距
contentPaddingBottom
contentPaddingLeft
contentPaddingRight
contentPaddingTop

# 需要注意的地方

设置了卡片阴影效果之后，在 API21 之前的机型上，CardView 会在整体设置大小之内预留出阴影部分的位置，因此实际上的显示出来的效果会比想要的大小会小一圈。而 API21 之后的阴影是绘制在指定View大小之外的。为了兼容低版本机型，可以设置 app:cardUseCompatPadding="true" ，这样，在高版本机型上显示效果就会与低版本机型保持一直，此时你在布局时，设置View大小应该考虑到阴影部分的大小