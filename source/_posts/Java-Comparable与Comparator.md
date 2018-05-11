---
title: Java-Comparable与Comparator
date: 2018-05-11 18:44:15
categories: Java
---

有时候需要对集合中的对象进行排序，例如 Android 开发中经常就需要按照某个条件进行排序，这个时候就需要使用比较器来实现了。实现一个比较器只需要实现 Comparable 接口或者实现 Comparator 比较器就可以了，实现了接口后就可以调用下面两个方法来排序了。Comparable 和 Comparator不同的是在集合对象的内部实现的，一个是在集合对象的外部实现的，其次使用 Comparable， 由于 JavaBean 实现 Comparable，JavaBean 里面只能有一种排序方法，所以也只能使用一种排序方式，而使用 Comparator 则可以 JavaBena 列表的的多种排序。

# 排序方法

两个使用比较器排序的方法：

```java
// 方法一
Collections.sort(List<T> list);

// 方法二
Collections.sort(List<T> list, Comparator<? super T> c);
```

<!-- more -->

# Comparable例子

第一个方法是没有比较器的，没比较器怎么排序？所以很显然，需要比较的对象的类需要实现 Comparable 接口。看一个例子， 创建一个 JavaBean 类，实现 Comparable 接口，需要重写 compareTo，equals 和 hashCode 方法，如下：

```java
public class CommodityBean implements Comparable<CommodityBean>, Serializable {
    private String origin;
    private long update_date;
    private int total_num;
    // 其他...

    /**
     * 按时间戳排序-顺序 (默认的排序方式)
     *
     * @param o 商品对象
     * @return  比较结果
     */
    @Override
    public int compareTo(@NonNull CommodityBean o) {
        if (this.getUpdate_date() < o.getUpdate_date()) return 1;
        if (this.getUpdate_date() > o.getUpdate_date()) return -1;
        return 0;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof CommodityBean)) return false;
        CommodityBean that = (CommodityBean) o;
        return getArticle_id().equals(that.getArticle_id());
    }

    @Override
    public int hashCode() {
        return getArticle_id().hashCode();
    }
}
```

调用 Collections.sort(dataList) 就可排序了，其中 dataList 里面存放的对象就是实现了 Comparable 接口的实体类。

# Comparator例子

第二个方法需要传递一个比较器对象，其实现是在实体类的外面实现，首先创建一个实体类，重写 equals 和 hashCode 方法，如下：

```java
public class CommodityBean implements Serializable {
    private String origin;
    private long update_date;
    private int total_num;
    // 其他...

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof CommodityBean)) return false;
        CommodityBean that = (CommodityBean) o;
        return getArticle_id().equals(that.getArticle_id());
    }

    @Override
    public int hashCode() {
        return getArticle_id().hashCode();
    }
}

创建 Comparator 对象，这里的例子创建多个 Comparator，可以根据条件使用不同的比较器来排序，并封装成一个类，如下：
public class CommodityComparator  {
    /**
     * 按时间戳排序-顺序
     */
    public static final Comparator<CommodityBean> ORDER_BY_TIME = (o1, o2) -> {
        if (o1.getUpdate_date() < o2.getUpdate_date()) return 1;
        if (o1.getUpdate_date() > o2.getUpdate_date()) return -1;
        return 0;
    };

    /**
     * 按时间戳排序-逆序
     */
    public static final Comparator<CommodityBean> ORDER_BY_TIME_REVERSE = (o1, o2) -> {
        if (o1.getUpdate_date() < o2.getUpdate_date()) return -1;
        if (o1.getUpdate_date() > o2.getUpdate_date()) return 1;
        return 0;
    };

    /**
     * 按销量排序-顺序
     */
    public static final Comparator<CommodityBean> ORDER_BY_PAY_NUM = (o1, o2) -> {
        if (o1.getPay_num() > o2.getPay_num()) return 1;
        if (o1.getPay_num() < o2.getPay_num()) return -1;
        return 0;
    };

    /**
     * 按销量排序-逆序
     */
    public static final Comparator<CommodityBean> ORDER_BY_PAY_NUM_REVERSE = (o1, o2) -> {
        if (o1.getPay_num() > o2.getPay_num()) return -1;
        if (o1.getPay_num() < o2.getPay_num()) return 1;
        return 0;
    };
}
```

最后调用 Collections.sort(dataList, CommodityComparator.ORDER_BY_PAY_NUM_REVERSE) 就可以排序了。