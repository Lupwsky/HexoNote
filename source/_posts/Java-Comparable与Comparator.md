---
title: Java-Comparable与Comparator
date: 2018-05-11 18:44:15
categories: Java
---

有时候需要对集合中的对象进行排序，例如 Android 开发中经常就需要按照某个条件进行排序，这个时候就需要使用比较器来实现了。实现一个比较器只需要实现Comparable 接口就可以了，实现了接口后就可以调用下面两个方法来排序了。Comparable 和 Comparator不同的是在集合对象的内部实现的，一个是在集合对象的外部实现的。

```java
(1) Collections.sort(List<T> list);
(2) Collections.sort(List<T> list, Comparator<? super T> c);
```

<!-- more -->

两个方法，第一个方法是没有比较器的，没比较器怎么排序？所以很显然，需要比较的对象的类需要实现Comparable接口。另外一个需要传递一个比较器就可以了。

在集合对象的内部实现Comparable接口完成排序

```java
(1) 创建一个JavaBean类，实现Comparable接口，需要重写compareTo，equals和hashCode方法
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

(2) 调用Collections.sort(dataList)就可排序了，其中dataList里面存放的对象就是实现了Comparable接口的实体类
```

在集合对象的外部实现

```java
(1) 创建实体类，仅需实现equals和hashCode方法
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

(2) 创建Comparator对象，封装成一个类
public class CommodityComparator  {
   
    /**
     * 按时间戳排序-顺序
     *
     */
    public static final Comparator<CommodityBean> ORDER_BY_TIME = (o1, o2) -> {
        if (o1.getUpdate_date() < o2.getUpdate_date()) return 1;
        if (o1.getUpdate_date() > o2.getUpdate_date()) return -1;
        return 0;
    };

    /**
     * 按时间戳排序-逆序
     *
     */
    public static final Comparator<CommodityBean> ORDER_BY_TIME_REVERSE = (o1, o2) -> {
        if (o1.getUpdate_date() < o2.getUpdate_date()) return -1;
        if (o1.getUpdate_date() > o2.getUpdate_date()) return 1;
        return 0;
    };


    /**
     * 按销量排序-顺序
     *
     */
    public static final Comparator<CommodityBean> ORDER_BY_PAY_NUM = (o1, o2) -> {
        if (o1.getPay_num() > o2.getPay_num()) return 1;
        if (o1.getPay_num() < o2.getPay_num()) return -1;
        return 0;
    };


    /**
     * 按销量排序-逆序
     *
     */
    public static final Comparator<CommodityBean> ORDER_BY_PAY_NUM_REVERSE = (o1, o2) -> {
        if (o1.getPay_num() > o2.getPay_num()) return -1;
        if (o1.getPay_num() < o2.getPay_num()) return 1;
        return 0;
    };
}

(3) 调用Collections.sort(dataList, CommodityComparator.ORDER_BY_PAY_NUM_REVERSE)就可以排序了 
```