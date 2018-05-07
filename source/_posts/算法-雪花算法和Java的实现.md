---
title: 算法-雪花算法和beyond大牛Java的实现方案学习笔记
date: 2018-04-26 09:41:23
categories: 算法
---

这篇文章内容均来源于网上beyond大牛的文章，仅仅作为一个笔记记录于此，方便日后使用方便。SnowFlake 算法是 Twitter 设计的一个可以在分布式系统中生成唯一的 ID 的算法，它可以满足每秒上万条消息 ID 分配的请求，这些消息 ID 是唯一的且有大致的递增顺序。SnowFlake 的优点是，整体上按照时间自增排序，并且整个分布式系统内不会产生 ID 碰撞 (由数据中心 ID 和机器 ID 作区分)，并且效率较高，经测试，SnowFlake 每秒能够产生 26 万 ID 左右。

# SnowFlake 的结构

SnowFlake的结构如下(每部分用-分开)，如下：
**0 - 0000000000 0000000000 0000000000 0000000000 0 - 00000 - 00000 - 000000000000**

(1) 1 位标识，由于 long 基本类型在 Java 中是带符号的，最高位是符号位，正数是 0，负数是 1，所以一般是正数，最高位是 0;

(2) 41 位时间截(毫秒级)，注意，41 位时间截不是存储当前时间的时间截，而是存储时间截的差值 (当前时间截 - 开始时间截) 得到的值，这里的的开始时间截，一般是我们的 ID 生成器开始使用的时间，由我们程序来指定的;

(3) 10 位的节点位，可以部署在 1024 个节点，包括 5 位 datacenterId (数据中心ID) 和 5 位 workerId (机器ID)，可以根据我们业务的需要，灵活分配节点部分;

(4) 12 位序列号，支持每个节点每毫秒 (同一机器，同一时间截) 产生 4096 个 ID 序号。

<!-- more -->

# SnowFlake 的 Java 实现方案

```Java
/**
 * twitter的snowflake算法 -- java实现
 * 
 * @author beyond
 * @date 2016/11/26
 */
public class SnowFlake {

    /**
     * 起始的时间戳
     */
    private final static long START_STMP = 1480166465631L;

    /**
     * 每一部分占用的位数
     */
    private final static long SEQUENCE_BIT = 12; //序列号占用的位数
    private final static long MACHINE_BIT = 5;   //机器标识占用的位数
    private final static long DATACENTER_BIT = 5;//数据中心占用的位数

    /**
     * 每一部分的最大值
     */
    private final static long MAX_DATACENTER_NUM = -1L ^ (-1L << DATACENTER_BIT);
    private final static long MAX_MACHINE_NUM = -1L ^ (-1L << MACHINE_BIT);
    private final static long MAX_SEQUENCE = -1L ^ (-1L << SEQUENCE_BIT);

    /**
     * 每一部分向左的位移
     */
    private final static long MACHINE_LEFT = SEQUENCE_BIT;
    private final static long DATACENTER_LEFT = SEQUENCE_BIT + MACHINE_BIT;
    private final static long TIMESTMP_LEFT = DATACENTER_LEFT + DATACENTER_BIT;

    private long datacenterId;  //数据中心
    private long machineId;     //机器标识
    private long sequence = 0L; //序列号
    private long lastStmp = -1L;//上一次时间戳

    public SnowFlake(long datacenterId, long machineId) {
        if (datacenterId > MAX_DATACENTER_NUM || datacenterId < 0) {
            throw new IllegalArgumentException("datacenterId can't be greater than MAX_DATACENTER_NUM or less than 0");
        }
        if (machineId > MAX_MACHINE_NUM || machineId < 0) {
            throw new IllegalArgumentException("machineId can't be greater than MAX_MACHINE_NUM or less than 0");
        }
        this.datacenterId = datacenterId;
        this.machineId = machineId;
    }

    /**
     * 产生下一个ID
     *
     * @return
     */
    public synchronized long nextId() {
        long currStmp = getNewstmp();
        if (currStmp < lastStmp) {
            throw new RuntimeException("Clock moved backwards.  Refusing to generate id");
        }

        if (currStmp == lastStmp) {
            //相同毫秒内，序列号自增
            sequence = (sequence + 1) & MAX_SEQUENCE;
            //同一毫秒的序列数已经达到最大
            if (sequence == 0L) {
                currStmp = getNextMill();
            }
        } else {
            //不同毫秒内，序列号置为0
            sequence = 0L;
        }

        lastStmp = currStmp;

        return (currStmp - START_STMP) << TIMESTMP_LEFT //时间戳部分
                | datacenterId << DATACENTER_LEFT       //数据中心部分
                | machineId << MACHINE_LEFT             //机器标识部分
                | sequence;                             //序列号部分
    }

    private long getNextMill() {
        long mill = getNewstmp();
        while (mill <= lastStmp) {
            mill = getNewstmp();
        }
        return mill;
    }

    private long getNewstmp() {
        return System.currentTimeMillis();
    }
}


public static void main(String[] args) {
    SnowFlake snowFlake = new SnowFlake(2, 3);

    for (int i = 0; i < (1 << 12); i++) {
        System.out.println(snowFlake.nextId());
    }
}
```

# 参考资料

感谢大牛beyond的无私分享：
`http://www.wolfbe.com/detail/201611/381.html`
`http://www.wolfbe.com/detail/201701/386.html`