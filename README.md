# SnowFlake
Twitter的分布式自增ID算法snowflake (Java版)

# 概述

分布式系统中，有一些需要使用全局唯一ID的场景，这种时候为了防止ID冲突可以使用36位的UUID，但是UUID有一些缺点，首先他相对比较长，另外UUID一般是无序的。

有些时候我们希望能使用一种简单一些的ID，并且希望ID能够按照时间有序生成。

而twitter的snowflake解决了这种需求，最初Twitter把存储系统从MySQL迁移到Cassandra，因为Cassandra没有顺序ID生成机制，所以开发了这样一套全局唯一ID生成服务。
SnowFlake算法是Twitter设计的一个可以在分布式系统中生成唯一的ID的算法，它可以满足Twitter每秒上万条消息ID分配的请求，这些消息ID是唯一的且有大致的递增顺序。

# 结构

snowflake的结构如下(每部分用-分开):

```
0 - 0000000000 0000000000 0000000000 0000000000 0 - 00000 - 00000 - 000000000000
```

第一位为未使用，接下来的41位为毫秒级时间(41位的长度可以使用69年)，然后是5位datacenterId和5位workerId(10位的长度最多支持部署1024个节点） ，最后12位是毫秒内的计数（12位的计数顺序号支持每个节点每毫秒产生4096个ID序号）

一共加起来刚好64位，为一个Long型。(转换成字符串长度为18)

snowflake生成的ID整体上按照时间自增排序，并且整个分布式系统内不会产生ID碰撞（由datacenter和workerId作区分），并且效率较高。据说：snowflake每秒能够产生26万个ID。


### 参考
|#|URL|
|---|----|
|1|[理解分布式id生成算法SnowFlake](https://segmentfault.com/a/1190000011282426)|
|2|[Twitter的分布式自增ID算法snowflake (Java版)](https://www.cnblogs.com/relucent/p/4955340.html)|
|3|[数据库分库分表（二）Twitter-Snowflake（64位分布式ID算法）分析与JAVA实现](https://www.jianshu.com/p/80e68ae9e3a4)|

```java
public class SnowFlake {

    /**
     * 起始的时间戳(2016-11-26 21:21:05,631)
     */
    private static final long START_STMP = 1480166465631L;

    /** 数据中心标识占用的位数 */
    private static final long DATACENTER_BIT = 5;

    /** 机器标识占用的位数 */
    private static final long MACHINE_BIT = 5;

    /** 序列号占用的位数 */
    private static final long SEQUENCE_BIT = 12;

    /** 支持的最大数据中心标识id，结果是31 (这个移位算法可以很快的计算出几位二进制数所能表示的最大十进制数) */
    private static final long MAX_DATACENTER_NUM = -1L ^ (-1L << DATACENTER_BIT);

    /** 支持的最大机器id，结果是31 */
    private static final long MAX_MACHINE_NUM = -1L ^ (-1L << MACHINE_BIT);

    /** 支持的序列号id，结果是4095 (0b111111111111=0xfff=4095) */
    private static final long MAX_SEQUENCE = -1L ^ (-1L << SEQUENCE_BIT);

    /** 机器ID向左移12位 */
    private static final long MACHINE_LEFT = SEQUENCE_BIT;

    /** 数据中心标识id向左移17位(12+5) */
    private static final long DATACENTER_LEFT = SEQUENCE_BIT + MACHINE_BIT;

    /** 时间截向左移22位(5+5+12) */
    private static final long TIMESTMP_LEFT = DATACENTER_LEFT + DATACENTER_BIT;

    /** 数据中心ID(0~31) */
    private long datacenterId;

    /** 机器标识ID(0~31) */
    private long machineId;

    /** 毫秒内序列号(0~4095) */
    private long sequence = 0L;

    /** 上次生成ID的时间截 */
    private long lastStmp = -1L;

    /**
     * 构造函数
     * 
     * @param datacenterId 数据中心ID(0~31)
     * @param machineId    机器标识ID(0~31)
     */
    public SnowFlake(long datacenterId, long machineId) {
        if (datacenterId > MAX_DATACENTER_NUM || datacenterId < 0) {
            throw new IllegalArgumentException(
                    String.format("datacenterId can't be greater than %d or less than 0", MAX_DATACENTER_NUM));
        }
        if (machineId > MAX_MACHINE_NUM || machineId < 0) {
            throw new IllegalArgumentException(
                    String.format("machineId can't be greater than %d or less than 0", MAX_MACHINE_NUM));
        }
        this.datacenterId = datacenterId;
        this.machineId = machineId;
    }

    /**
     * 产生下一个ID (该方法是线程安全的)
     *
     * @return SnowflakeId
     */
    public synchronized long nextId() {
        long currStmp = getNewstmp();
        // 如果当前时间小于上一次ID生成的时间戳，说明系统时钟回退过这个时候应当抛出异常
        if (currStmp < lastStmp) {
            throw new RuntimeException(String.format(
                    "Clock moved backwards.  Refusing to generate id for %d milliseconds", lastStmp - currStmp));
        }

        // 如果是同一时间生成的，则进行毫秒内序列
        if (currStmp == lastStmp) {
            // 相同毫秒内，序列号自增
            sequence = (sequence + 1) & MAX_SEQUENCE;
            // 同一毫秒的序列数已经达到最大
            if (sequence == 0L) {
                // 阻塞到下一个毫秒,获得新的时间戳
                currStmp = getNextMill();
            }
        } else {
            // 不同毫秒内，序列号置为0
            sequence = 0L;
        }

        // 上次生成ID的时间截
        lastStmp = currStmp;

        // 移位并通过或运算拼到一起组成64位的ID
        return (currStmp - START_STMP) << TIMESTMP_LEFT // 时间戳部分
                | datacenterId << DATACENTER_LEFT // 数据中心部分
                | machineId << MACHINE_LEFT // 机器标识部分
                | sequence; // 序列号部分
    }

    /**
     * 阻塞到下一个毫秒，直到获得新的时间戳
     * 
     * @return 当前时间戳
     */
    private long getNextMill() {
        long mill = getNewstmp();
        while (mill <= lastStmp) {
            mill = getNewstmp();
        }
        return mill;
    }

    /**
     * 返回以毫秒为单位的当前时间
     * 
     * @return 当前时间(毫秒)
     */
    private long getNewstmp() {
        return System.currentTimeMillis();
    }

    public static void main(String[] args) throws Exception {
        SnowFlake snowFlake = new SnowFlake(2, 3);

        long start = System.currentTimeMillis();
        for (int i = 0; i < 1000000; i++) {
            System.out.println(snowFlake.nextId());
        }

        System.out.println(System.currentTimeMillis() - start);

    }

}


