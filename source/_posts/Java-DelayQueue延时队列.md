---
title: Java-DelayQueue延时队列
date: 2019-02-28 22:17:12
categories: Java
---

# DelayQueue

* DelayQueue 内部使用的是 PriorityQueue 实现的, 同时实现了 BlockingQueue, BlockingQueue 是一个读写都是线程安全的队列
* 是一个先进先出的无界阻塞队列 (无界也不是完全的没有大小, 最大为 int 的最大值, 21 亿左右)
* 只有在延迟期满的时候才能从中获取到元素, 放在队列头部的是延迟期满后保存时间最长的 Delayed 元素
* 如果延迟都还没有期满, 则队列没有头部, poll 将返回 null

# DelayQueue 使用场景

DelayQueue 可以用来实现延迟消息, 生产/消费队列, 如果不考虑分布式系统, 队列中的数据持久化问题和消息可靠性问题 (虽然可以自己实现重试机制解决一定的问题,但是如果正在处理某条消息时客户端崩溃还是导致这条消息丢失), DelayQueue 就可以满足要求

<!-- more -->

# DelayQueue 中的元素

放入 DelayQueue 的元素需要继承 Delayed 接口, Delayed扩展了 Comparable 接口, 用于比较延时的时间值, 示例:

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class DelayQueueItem implements Delayed {

    /**
     * 延迟时间, 单位毫秒
     */
    private long delay;

    /**
     * 过期时间, 单位毫秒
     */
    private long expire;

    /**
     * 保存在队列中的数据
     */
    private UserInfo userInfo;

    public DelayQueueItem(long delay, UserInfo userInfo) {
        this.delay = delay;
        this.expire = System.currentTimeMillis() * 1000 + delay;
        this.userInfo = userInfo;
    }

    /**
     * 当一个元素的 getDelay(TimeUnit.NANOSECONDS) 方法返回一个小于等于 0 的值时, 将发生到期, 此元素将放到队列头部
     */
    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(this.expire - System.currentTimeMillis() , TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed o) {
        return (int) (this.getDelay(TimeUnit.MILLISECONDS) -o.getDelay(TimeUnit.MILLISECONDS));
    }
}
```

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserInfo {
    private String username;
    private String password;
}
```

参考资料:

* [关于 getDelay 方法没有使用 TimeUnit 转换造成性能问题](http://www.blogjava.net/killme2008/archive/2010/10/22/335897.html)

# 使用示例

放入数据到队列使用 offer 方法, 获取数据通常使用 take, 而不是 poll, 对所有的 BlockingQueue:

* take 方法在没有过期数据时会阻塞并释放资源, poll 会直接返回 null
* offer 方法在队列空间不足的时候会直接返回 false, 而 put 则会阻塞等到队列有空间

以上两点再使用的时候要注意, 尤其是在 poll 获取数据的时候, 我曾经因为开多线程 poll 数据, 在队列没有数据的时候导致 CPU 飙升

```java
public static void main(String[] args) {
    DelayQueue<DelayQueueItem> delayQueue = new DelayQueue<>();
    delayQueue.offer(new DelayQueueItem(3000, UserInfo.builder().username("lpw1").password("123").build()));
    delayQueue.offer(new DelayQueueItem(6000, UserInfo.builder().username("lpw2").password("123").build()));
    delayQueue.offer(new DelayQueueItem(9000, UserInfo.builder().username("lpw3").password("123").build()));

    DelayThread delayThread = new DelayThread(delayQueue);
    delayThread.start();

    try {
        delayThread.join();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

DelayThread 是一个自定义的线程, 里面是一个循环, 用于获取过期的元素, 源码如下:

```java
@Slf4j
public class DelayThread extends Thread {
    private DelayQueue<DelayQueueItem> delayQueue;

    public DelayThread(DelayQueue<DelayQueueItem> delayQueue) {
        this.delayQueue = delayQueue;
    }

    @Override
    public void run() {
        try {
            while (true) {
                // take 方法会阻塞在这里, 知道有过期的元素
                DelayQueueItem delayQueueItem = delayQueue.take();
                log.info("userInfo = {}", delayQueueItem.getUserInfo());
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```