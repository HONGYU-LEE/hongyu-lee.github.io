---
title: 分布式ID
date: 2022-05-24T15:50:13+08:00
categories:
    - 分布式
tags:
    - 分布式
    - 分布式ID
---

# 分布式ID

<font color=red>**ID是数据的唯一标识**</font>，传统的做法是使用数据库的自增ID，但是随着业务规模的不断发展，数据量将越来越大，于是需要进行<font color=red>**分库分表**</font>，而分表后，每个表中的ID都会按自己的节奏进行自增，很有可能出现ID冲突的情况。这时就需要一个单独的机制来负责生成一个<font color=red>**全局唯一的ID**</font>。这个ID也可以叫做<font color=red>**分布式ID**</font>。

对于一个合格的分布式ID，它应该满足以下几项条件

- **全局唯一**
- **高可用**
- **高性能** 
- **趋势递增**
- **方便接入**

下面就介绍几种常见的分布式ID生成方案

## 数据库

### 自增ID

我们可以利用数据库的`auto_increment`自增ID来生成分布式ID，只需要一个数据库实例即可完成。

建表如下

```sql
CREATE DATABASE `SEQID`;

CREATE TABLE SEQID.SEQUENCE_ID (
	id bigint(20) unsigned NOT NULL auto_increment, 
	stub char(10) NOT NULL default '',
	PRIMARY KEY (id),
	UNIQUE KEY stub (stub)
) ENGINE=MyISAM;
```

我们通过往该表中插入数据，并获取到自增的主键id。为了考虑到并发的安全 ，我们可以通过事务来完成这个操作。

```sql
begin;
insert into SEQUENCE_ID(value) VALUES ('values');
select last_insert_id();
commit;
```

我们可以看出，这种依赖单点的方法虽然实现起来简单，但是这种依赖单点的方法并不可靠，因为Mysql并不能很好的支持高并发，当请求ID量特别大的时候就会因为单点的宕机而影响到整个业务系统。



### 多主模式

既然单点不可靠，那么我们可以考虑使用<font color=red>**集群**</font>的方式来解决这个问题。考虑到单个主节点可能会因为宕机而没有将数据同步到从节点，可能会导致ID重复的情况，因此我们可以考虑使用<font color=red>**多主集群**</font>。

为了防止多个主节点生成重复的ID，我们通常会通过控制<font color=red>**初始值**</font>和<font color=red>**步长**</font>来避免这个问题。

例如在有两个主节点的情况下，我们就可以通过这种方式来错开ID。

```sql
-- 节点1
set @@auto_increment_offset = 1;     -- 起始值
set @@auto_increment_increment = 2;  -- 步长

-- 节点2
set @@auto_increment_offset = 2;     -- 起始值
set @@auto_increment_increment = 2;  -- 步长
```

但是，这种方法的可拓展性不强，倘若我们要往集群中增加新的主节点时，我们就需要重新去设置步长，并且还需要根据前两个节点的自增ID的大小来考虑我们新节点的起始值，而这些操作只能**人工**来进行。如果在修改步长的时候出现了重复的ID，此时就还需要进行停机修改。



## 号段模式

前面两种方法的局限性在于其每次获取ID时都需要直接访问数据库，效率较低，<font color=red>**如果能够一次获取大量的ID，并将其缓存在本地，那样就可以大大的提升ID获取的效率，这也是号段模式的核心思想。**</font>

号段模式每次从数据库中批量的获取一段自增ID，即取出一个范围的ID交给号段服务维护。例如（1，2000]即代表着2000个ID。我们的业务系统只需要到号段服务中申请ID，不需要每次都去请求数据库，直到所有的ID都用完后才会去申请下一个号段。

数据库表修改如下

```sql
CREATE TABLE id_generator (
  id int(10) NOT NULL,
  max_id bigint(20) NOT NULL COMMENT '当前最大id',
  step int(20) NOT NULL COMMENT '号段的步长',
  biz_type int(20) NOT NULL COMMENT '业务类型',
  version int(20) NOT NULL COMMENT '版本号',
  PRIMARY KEY (`id`)
) 
```

号段服务不再强依赖数据库，即使数据库不可用，号段服务也可以继续工作直到申请的ID全部使用完。但是如果此时号段服务重启，就会导致剩余的ID丢失。

为了保证号段服务的高可用，我们同样需要建立一个集群，在请求方从号段服务获取ID时，就会随机的选取一个节点来获取，而这种并发场景下我们同样需要考虑到并发安全的问题，因此我们上面的表中也提供了一个版本号的字段version，我们可以使用**乐观锁**来进行并发的控制。

```sql
update id_generator set current_max_id=#{newMaxId}, version=version+1 where version = #{version}
```

我们可以使用上面的SQL来获取新号段，当update更新成功就说明号段获取成功了。



## 雪花算法

<font color=red>**雪花算法（Snowflake）** </font>是twitter开源的一个分布式ID的生成算法，它的核心思想是：<font color=red>**生成一个long类型的ID，一个long大小8字节，一个字节8个比特位，因此它使用64个比特位来确定一个分布式ID。其中41bit代表时间戳，10bit标识一台机器，剩下12bit则用来标识每个id**</font>
![在这里插入图片描述](http://img.orekilee.top//imgbed/distributed/distributed14.png)

<center>snowflake结构图</center>

- **第1个bit位为符号位**，因为生成的id通常为正数所以固定为0
- **2~42位为时间戳部分**，其精确至毫秒。同时为了更加合理的利用，其并不会存储当前的时间，而是使用时间戳的差值（当前时间 - 固定的开始时间），这样就可以保证ID从更小的起点开始生成。
- **43~52位为工作机器的id**，这里的计算会更加灵活，可以根据机房数量、机器数量来自行均衡，保证能够利用到更多的机器。
- **53~64位为序列编号**，在同一毫秒中的同一台机器上我们可以生成4096个ID

考虑到不同的业务场景以及各个公司的特性，大多数公司并不会去直接使用snowflake，而是会对其进行改造，让其更贴合自身的使用场景。如百度的uid-generator、美团的Leaf、滴滴的TinyId等。

下面是github上开源的一个java实现的snowflake

```java
/**
 * twitter的snowflake算法 -- java实现
 * github链接：https://github.com/beyondfengyu/SnowFlake
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

    public static void main(String[] args) {
        SnowFlake snowFlake = new SnowFlake(2, 3);

        for (int i = 0; i < (1 << 12); i++) {
            System.out.println(snowFlake.nextId());
        }
    }
}
```



## Redis

我们可以利用Redis中的`incr`命令来原子的获取自增ID

```sql
127.0.0.1:6379> set seq_id 1     // 初始化自增ID
OK
127.0.0.1:6379> incr seq_id      // 自增并返回结果
(integer) 2
```

使用Redis实现起来特别简单且高效，但是我们还需要考虑到<font color=red>**持久化**</font>时带来的一些问题

- **RDB持久化**：由于其是定期保存一次数据库的快照，为了保证效率他也存在着一定的时间间隔。倘若在我们刚保存一次快照后，连续获取了几次ID，而此时还没来得及做下一次持久化就宕机了。当我们通过持久化重启Redis后，这段时间生成的ID就会被重复使用。
- **AOF持久化**：AOF相当于是逻辑日志，其会通过保存我们执行过的命令来进行持久化。它并不像RDB出现一段时间的数据丢失而导致的ID重复的情况，但是在它恢复的过程中需要重新执行保存的命令，因此随着ID数量的增多，它重启恢复数据的时间也会越来越慢。



## 总结

|                | 优点                                                         | 缺点                                                         |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 数据库自增ID   | 实现简单，ID单调自增，数值类型查询效率高                     | 单点问题，在高并发时可能会有宕机的风险                       |
| 数据库多主集群 | 解决了单点问题，一定程度上提高了稳定性                       | 可拓展性不强，随着业务规模的不断扩大， 集群也会随之增加，但是新主节点的加入较为麻烦，需要人工操作 |
| 号段模式       | 不强依赖数据库，提高了效率                                   | 当为了保证高可用而使用多主集群时，仍然需要去修改起始值和步长 |
| 雪花算法       | 1.可以根据业务特性自由分配比特位，较为灵活。2.不依赖第三方系统，独立部署 | 强依赖机器时钟，如果出现时钟回拨则会导致系统不可用           |
| Redis          | 实现起来简单且高效                                           | 持久化恢复存在问题，如RDB重复ID，AOF速度慢                   |

