---
title: Java-编写高质量代码-改善程序员的151个建议读书笔记
date: 2019-04-04 17:40:21
categories: Java
---

[编写高质量代码-改善程序员的151个建议]() 读书笔记, 记录一下平常写代码时没有注意的知识点, 书籍后面提到了关于反射, 线程相关的建议, 反射和多线程之前有专门的笔记记录过就不在记录了

# 变长参数基本规则

* 变长参数必须是方法中的最后一个参数
* 同一个方法不能定义多个变长参数

# 自增的陷阱

下面的这段代码最终 count 的值为 0, 而不是 10

```java
public static void main(String[] args) {
    int count = 0;
    for (int i = 0; i < 10; i++) {
        count = count++;
    }
}
```

<!-- more -->

count++ 和 ++count, 先使用, 再加 1, 先加 1, 再使用, 这里的使用不是使用 count 值, 使用的是 count++ 这个表达式的值, count++ 和 ++count 表达式不仅仅是让 count 自增, 并且是有返回值的, count++ 返回的是 count 自增之前的值, ++count 返回的是 count 自增后的值, 对于 count = count++ 这个表达式, count 先自增 1, count 的值为 1, 然后将 count++ 表达式的返回的值赋值给 count, count++ 返回的是 count 自增前的值, 自增前值为 0, 因此 count 虽然在前面自增了 1 , 但是在这一步又被赋值为 0, 因此循环结束后 count 的值还是 0, 对于表达式 count++, JVM 做的操作是如下:

* JVM 把 count 值拷贝到临时变量中
* JVM 将 count 值加 1
* JVM 返回临时变量的值

不过, 现在 IDE 都很智能, 像 count = count++ 这种写法都会告警提示的, 我觉得实际项目中应该很少有人会这样写吧~~

# 奇偶判断

判断一个数的奇偶, 使用取余的方式, 判断偶数时使用偶数去做比较判断而不用奇数去判断, 正常的判断如下, 会正常的输出 `奇数`:

```java
int value = -1;
int result = value % 2;
log.info("{}", result == 0 ? "偶数" : "奇数");
```

但是如果使用 `value % 2 == 1` 去判断, 当需要判断的数为奇数的为负数时, 如 `-1`, 却错误的输出了 `偶数`:

```java
int value = -1;
int result = value % 2;
log.info("{}", result != 1 ? "偶数" : "奇数");
```

Java 中取余的模拟算法如下, 以上面的例子为例:

```java
// 当 value 为 -1 时, value / 2 * 2 = 0, 最终 a = -1
// -1 == 1 的的值为 false, 所以就被判定为偶数了
int re = value - value / 2 * 2;
```

`还有一种判断的方法 i&1 != 0 也能正确的判断`

# 不使用隐形类型转换

Java 中是先运算再做类型转换的, 如下代码:

```java
int a = getValue();
long result = a * 10000;
```

当 int 的 a 值获取到一个大的 int 类型的值时, 首先 a * 10000 计算可能得到一个对于 int 类型来说已经越界的值, 再被转成 long 类存放到 result 中, 此时得到的值任然是错误的值, 显示的声明类型转换就可以解决, 示例代码如下:

```java
long result = a * 10000L;
```

更加稳妥的方法是让第一个参与运算的参数直接使用长整型, 如下:

```java
long result = 10000L * a;
```

# List 中过滤掉 NULL 的值

```java
List<Integer> dataList = Lists.newArrayList(1, 2, null, 3);
int result = dataList.stream().mapToInt(value -> value).sum();
```

运行时就会出现空指针异常, 以后需要注意此类问题, 添加元素和参与运算的时候需要对 NULL 进行过滤, 示例如下:

```java
List<Integer> dataList = Lists.newArrayList(1, 2, null, 3);
int result = dataList.stream().filter(Objects::nonNull).mapToInt(value -> value).sum();
```

# 包装类型的比较

包装类型的比较一定不能直接使用 `==, >=, <` 等判断, 刚好最近团队的代码就出现这种情况, 记录下来, 伪代码如下:

```java
public static void main(String[] args) {
    Byte value1 = new Byte("1");
    Byte value2 = new Byte("1");
    if (value1 == value2) {
        log.info("do something 1 ...");
    } else {
        log.info("do something 2 ...");
    }
}
```

使用 equales() 方法或者使用 camareTo() 方法来判断, 示例代码如下:

```java
public static void main(String[] args) {
    Byte value1 = new Byte("1");
    Byte value2 = new Byte("1");
    if (value1.compareTo(value2) == 0) {
        log.info("do something 1 ...");
    } else {
        log.info("do something 2 ...");
    }
}
```

# 优先选择基本类型

包装类型在参与运算的时候需要进行拆包, 从性能, 安全和稳定方面应该优先使用基本类型

# 整型池

 new 一个对象的的方式创建的 Interger 的实例每次创建的都是新的实例, 即使封装的基本类型的值是相同的, value1 和 value2 的地址肯定是不相等的, 代码如下:

```java
Integer value1 = new Integer(100);
Integer value2 = new Integer(100);
if (value1 == value2) {
    log.info("true {}, {}", System.identityHashCode(value1), System.identityHashCode(value2));
} else {
    log.info("false {}, {}", System.identityHashCode(value1), System.identityHashCode(value2));
}
```

但是通过 Interget.valueOf() 的方式创建时, 每次创建实例前会去一个缓存中查找时候已经存在, 如果已经存在 (范围为 -128 至 127 之间) 就直接返回已经存在的实例, 否则才新建, valueOf() 源码如下:

```java
static final int low = -128;
static final int high;

static {
// high value may be configured by property
int h = 127;
// ...
}

public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

如下测试代码, 就会输出 false:

```java
public static void main(String[] args) {
    Integer value1 = 100;
    Integer value2 = 100;

    if (value1 == value2) {
        log.info("true {}, {}", System.identityHashCode(value1), System.identityHashCode(value2));
    } else {
        log.info("false {}, {}", System.identityHashCode(value1), System.identityHashCode(value2));
    }
}
```

`使用整型池可以显著提升空间和时间性能`

# 代码块

* 普通代码块, 方法后面 `{}` 括起来的代码片段
* 静态代码块, static 关键字修饰, 并使用 `{}` 括起来的代码片段, 用于静态变量初始化或者环境初始化
* 同步代码块, synchronized 关键字修饰, 并使用 `{}` 括起来的代码片段, 用于多线程同步
* 构造代码块, 类中直接使用`{}` 括起来的代码片段, 构造代码块会被插入到每个构造方法之前, 在通过 new 的方式创建实例调用构造方法的时候就会执行, 用于初始化实例变量和环境

# 字符串常量池

创建字符串时, 首先检测池中是否有字面相等的字符串, 如果有直接返回, 如果没有则添加到字符串常量池, 然后返回, 但是如果是使用 new String() 的方式创建的字符串, 则不会去字符串常量池检查, 也不会放入字符串常量池中

```java
String str1 = "111";
String str2 = "111";
String str3 = new String("111");
String str4 = str3.intern();
log.info("{}", str1 == str2);
log.info("{}", str1 == str3);
log.info("{}", str1 == str4);
```

分别输出, `true`, `false`, `true`, 前面两个就不用说了, 主要说说第三个, String.intern() 方法, 该方法会先到字符串常量池中先检测是否存在, 如果存在就直接返回, 如果不存在, 将新的值放入字符创常量池中, 返回创建的值

# String, StringBuffer, StringBuilder 的应用场景

* String 字符串不经常变化, 如常量的声明, 少量变量的运算
* StringBuffer 字符串经常变化, 且运行在多线程中
* StringBuilder 字符串经常变化, 且运行在单线程中

# 汉字字符串排序建议

* 排序汉字是经常使用的汉字, 使用 Collator 类可以满足要求
* 需要进行严格排序, 将汉字转换成拼音后再进行排序, 如使用开源项目 pinyin4j 等将汉字转换成拼音

# asList 获取的 List 是一个长度不可变的 List

```java
Integer[] dataArr = {1, 2, 3, 4}
List<Integer> dataList = Array.asList(dataArr);
// 添加的元素出现异常
dataList.add(5);
```

# 列表相等的判断

列表判断是否相等只关心元素的数据是否相同, 而不关心使用具体使用实现类, 如 ArrayList 类型和 Vector 类型都实现了 AbstractList, 如果里面存放的元素数量相等, 并且内容相同, 尽管不是同一个类型, 使用 equals() 判断时也会判断为相等

# Collections.shuffle() 打乱列表

```java
Clooections.shuffle(dataList);
```

# 枚举类

基础用法:

```java
public enum Clolr {
    RED(0, "红色", "红色的描述"),

    WHITE(1, "白色", "白色的描述");

    @Getter
    private final int index;

    @Getter
    private final String name;

    @Getter
    private final String desc;

    EnumTest(int index, String name, String desc) {
        this.index = index;
        this.name = name;
        this.desc = desc;
    }

    public static EnumTest getInstance(String name) {
        switch (name) {
            case "白色": return WHITE;
            case "红色": return RED;
            default: throw new RuntimeException("错误的枚举类型");
        }
    }
}
```

枚举类实现工厂方类, 可以避免错误调用发生, 性能好, 使用便捷, 还能降低类之间的耦合, 推荐使用:

```java
public enum CarFactory {
    FordCar, BomCar;

    public Car create() {
        switch(this) {
            case FordCar: return FordCar.newBuilder().build();
            case BomCar: return BomCar.newBuilder().build();
            default: throw new AssertionError("无效的参数");
        }
    }
}


// 使用工厂模式创建实例
Car car = CarFactory.BomCar.create();
```

# 异常捕获的 finally 中不要操作返回值

finally 块中的 return 返回后方法结束执行, 并且会重置 try 块中和 catch 中的 return 语句, 即出现先出现先返回, 再执行 finally, 再重置返回值的情况

```java
public static void main(String[] args) {
    // 先执行 try 中的 return 语句
    // 接着执行 finally 里面的 return 语句, 返回值被重置, 最终返回了 -1 而不是 1
    log.info("{}", doProcess(1));

    // 出现异常被捕获, 执行了 catch 的 return 语句
    // 接着执行 finally 里面的 return 语句, 返回值被重置, 最终返回了 -1 而不是 100
    log.info("{}", doProcess(-1));
}

public static int doProcess(int p) {
    try {
        if (p < 0) {
            throw new DataFormatException("格式错误");
        } else {
            return p;
        }
    } catch (Exception e) {
        log.info("error = {}", e);
        return 100;
    } finally {
        return -1;
    }
}
```

定义变量, 在 finally 中修改后再返回, 如下代码, 当 p 传值为 1 时, 按照上面的逻辑, 会不会返回 a 的值为 1000 ? 经测试实际上返回的值为 10, 最后一个 return 不会执行到, 因为在 try 块中方法就已经有返回值 1000 了, 在 finally 修改 a 的值已经不会影响返回值了, 同时 finally 里面的语句执行完方法表示方法也就执行完毕了, 最后的 return 语句也不会执行到, 最后的 return 语句也是一种错误的语法

```java
public static int doProcess(int p) {
    int a = 1;
    try {
        if (p < 0) {
            throw new DataFormatException("格式错误");
        } else {
            return a * 10;
        }
    } catch (Exception e) {
        log.info("error = {}", e);
        return a;
    } finally {
        a = 1000;
    }
    return a;
}
```

上面的代码或许会出现在面试题中, 实际上的代码也不允许去这样写, 至少在 IDEA 里面就明确的提示你最后的 return 语句是一个无效的语句, 需要删除, 否则编译也无法通过