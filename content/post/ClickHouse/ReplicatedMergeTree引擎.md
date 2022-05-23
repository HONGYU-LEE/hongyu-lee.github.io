---
title: ClickHouse 副本协同原理：ReplicatedMergeTree引擎
date: 2022-05-23T18:19:13+08:00
categories:
    - ClickHouse
tags:
    - 列式存储
    - ClickHouse
    - OLAP 
---

# ReplicatedMergeTree引擎

ReplicatedMergeTree是MergeTree的派生引擎，它在MergeTree的基础上加入了分布式协同的能力，只有使用了ReplicatedMergeTree复制表系列引擎，才能应用副本的能力。或者用一种更为直接的方式理解，即使用ReplicatedMergeTree的数据表就是副本。

![](http://img.orekilee.top//imgbed/clickhouse/ch37.png)

<center>ReplicatedMergeTree与MergeTree的逻辑关系</center>

在MergeTree中，一个数据分区由开始创建到全部完成，会历经两类存储区域。

- **内存**：数据首先会被写入内存缓冲区。
- **本地磁盘**：数据接着会被写入tmp临时目录分区，待全部完成后再将临时目录重命名为正式分区。

ReplicatedMergeTree在上述基础之上增加了ZooKeeper的部分，它会进一步在ZooKeeper内创建一系列的监听节点，并以此实现多个实例之间的通信。在整个通信过程中，ZooKeeper并不会涉及表数据的传输。



## 特点

作为数据副本的主要实现载体，ReplicatedMergeTree在设计上有一些显著特点。

- **依赖ZooKeeper**：在执行`INSERT`和`ALTER`查询的时候，ReplicatedMergeTree需要借助ZooKeeper的分布式协同能力，以实现多个副本之间的同步。但是在查询副本的时候，并不需要使用ZooKeeper。

- **表级别的副本**：副本是在表级别定义的，所以每张表的副本配置都可以按照它的实际需求进行个性化定义，包括副本的数量，以及副本在集群内的分布位置等。

- **多主架构（Multi Master）**：可以在任意一个副本上执行`INSERT`和`ALTER`查询，它们的效果是相同的。这些操作会借助ZooKeeper的协同能力被分发至每个副本以本地形式执行。

- **Block数据块**：在执行`INSERT`命令写入数据时，会依据`max_insert_block_size`的大小（默认1048576行）将数据切分成若干个Block数据块。所以Block数据块是数据写入的基本单元，并且具有写入的原子性和唯一性。

- **原子性**：在数据写入时，一个Block块内的数据要么全部写入成功，要么全部失败。

- **唯一性**：在写入一个Block数据块的时候，会按照当前Block数据块的数据顺序、数据行和数据大小等指标，计算Hash信息摘要并记录在案。在此之后，如果某个待写入的Block数据块与先前已被写入的Block数据块拥有相同的Hash摘要（Block数据块内数据顺序、数据大小和数据行均相同），则该Block数据块会被忽略。

  

## 数据结构

### ZooKeeper内的节点结构

ReplicatedMergeTree需要依靠ZooKeeper的事件监听机制以实现各个副本之间的协同。所以，在每张ReplicatedMergeTree表的创建过程中，它会以zk_path为根路径，在Zoo-Keeper中为这张表创建一组监听节点。按照作用的不同，监听节点可以大致分成如下几类：

- **元数据**
  - **/metadata**：保存元数据信息，包括主键、分区键、采样表达式等。
  - **/columns**：保存列字段信息，包括列名称和数据类型。
  - **/replicas**：保存副本名称，对应设置参数中的replica_name。
- **判断标识**
  - **/leader_election**：用于主副本的选举工作，主副本会主导MERGE和MUTATION操作（ALTER DELETE和ALTER UPDATE）。这些任务在主副本完成之后再借助ZooKeeper将消息事件分发至其他副本。
  - **/blocks**：记录Block数据块的Hash信息摘要，以及对应的partition_id。通过Hash摘要能够判断Block数据块是否重复；通过partition_id，则能够找到需要同步的数据分区。
  - **/block_numbers**：按照分区的写入顺序，以相同的顺序记录partition_id。各个副本在本地进行MERGE时，都会依照相同的block_numbers顺序进行。
  - **/quorum**：记录quorum的数量，当至少有quorum数量的副本写入成功后，整个写操作才算成功。quorum的数量由insert_quorum参数控制，默认值为0。
- **操作日志**
  - **/log**：常规操作日志节点（INSERT、MERGE和DROP PARTITION），它是整个工作机制中最为重要的一环，保存了副本需要执行的任务指令。log使用了ZooKeeper的持久顺序型节点，每条指令的名称以log-为前缀递增，例如log-0000000000、log-0000000001等。每一个副本实例都会监听/log节点，当有新的指令加入时，它们会把指令加入副本各自的任务队列，并执行任务。
  - **/mutations**：MUTATION操作日志节点，作用与log日志类似，当执行ALERT DELETE和ALERTUPDATE查询时，操作指令会被添加到这个节点。mutations同样使用了ZooKeeper的持久顺序型节点，但是它的命名没有前缀，每条指令直接以递增数字的形式保存，例如0000000000、0000000001等。
  - **/replicas/{replica_name}/***：每个副本各自的节点下的一组监听节点，用于指导副本在本地执行具体的任务指令，其中较为重要的节点有如下几个：
    - **/queue**：任务队列节点，用于执行具体的操作任务。当副本从/log或/mutations节点监听到操作指令时，会将执行任务添加至该节点下，并基于队列执行。
    - **/log_pointer**：log日志指针节点，记录了最后一次执行的log日志下标信息。
    - **/mutation_pointer**：mutations日志指针节点，记录了最后一次执行的mutations日志名称。



### Entry日志对象的数据结构

ReplicatedMergeTree在ZooKeeper中有两组非常重要的父节点，那就是/log和/mutations。它们的作用犹如一座通信塔，是分发操作指令的信息通道，而发送指令的方式，则是为这些父节点添加子节点。所有的副本实例，都会监听父节点的变化，当有子节点被添加时，它们能实时感知。这些被添加的子节点在ClickHouse中被统一抽象为Entry对象，而具体实现则由LogEntry和MutationEntry对象承载，分别对应/log和/mutations节点

- **LogEntry**
  - **source replica**：发送这条Log指令的副本来源，对应replica_name。
  - **type**：操作指令类型，主要有get、merge和mutate三种，分别对应从远程副本下载分区、合并分区和`MUTATION`操作。
  - **block_id**：当前分区的BlockID，对应/blocks路径下子节点的名称。
  - **partition_name**：当前分区目录的名称。
- **MutationEntry**
  - **source replica**：发送这条`MUTATION`指令的副本来源，对应replica_name。
  - **commands**：操作指令，主要有`ALTER DELETE`和`ALTER UPDATE`。
  - **mutation_id**：MUTATION操作的版本号。
  - **partition_id**：当前分区目录的ID。



## 副本协同的核心流程

副本协同的核心流程主要有`INSERT`、`MERGE`、`MUTATION`和`ALTER`四种，分别对应了数据写入、分区合并、数据修改和元数据修改。`INSERT`和`ALTER`是分布式执行的，借助ZooKeeper的事件通知机制，多个副本之间会自动进行有效协同，但是它们不会使用ZooKeeper存储任何分区数据。而其他操作并不支持分布式执行，包括`SELECT`、`CREATE`、`DROP`、`RENAME`和`ATTACH`。

在下列例子中，使用ReplicatedMergeTree实现一张拥有1分片、1副本的数据表来分别执行`INSERT`、`MERGE`、`MUTATION`和`ALTER`操作，演示执行流程。

### INSERT

当需要在ReplicatedMergeTree中执行INSERT查询以写入数据时，即会进入INSERT核心流程，它的核心流程如下图所示

![](http://img.orekilee.top//imgbed/clickhouse/ch45.png)

<center>INSERT的核心执行流程</center>

1. **向副本A写入数据**
2. **由副本A推送Log日志**
3. **各个副本拉取Log日志**
4. **各个副本向远端副本发起下载请求**
   - 选择一个远端的其他副本作为数据的下载来源。远端副本的选择算法大致是这样的：
     - 从/replicas节点拿到所有的副本节点。
     - 遍历这些副本，选取其中一个。选取的副本需要拥有最大的`log_pointer`下标，并且`/queue`子节点数量最少。`log_pointer`下标最大，意味着该副本执行的日志最多，数据应该更加完整；而`/queue`最小，则意味着该副本目前的任务执行负担较小。
5. **远端副本响应其它副本的数据下载**
6. **各个副本下载数据并完成本地写入**

在`INSERT`的写入过程中，ZooKeeper不会进行任何实质性的数据传输。本着谁执行谁负责的原则，在这个案例中由CH5首先在本地写入了分区数据。之后，也由这个副本负责发送Log日志，通知其他副本下载数据。如果设置了`insert_quorum`并且`insert_quorum>=2`，则还会由该副本监控完成写入的副本数量。其他副本在接收到Log日志之后，会选择一个最合适的远端副本，点对点地下载分区数据。



### MERGE

当ReplicatedMergeTree触发分区合并动作时，即会进入这个部分的流程，它的核心流程如下图所示

![](http://img.orekilee.top//imgbed/clickhouse/ch39.png)

<center>MERGE的核心执行流程</center>

无论MERGE操作从哪个副本发起，其合并计划都会交由主副本来制定。

1. **创建远程连接，尝试与主副本通信**
2. **主副本接收通信**
3. **由主副本制定MERGE计划并推送Log日志**
4. **各个副本分别拉取Log日志**
5. **各个副本分别在本地执行MERGE**

可以看到，在MERGE的合并过程中，ZooKeeper也不会进行任何实质性的数据传输，所有的合并操作，最终都是由各个副本在本地完成的。而无论合并动作在哪个副本被触发，都会首先被转交至主副本，再由主副本负责合并计划的制定、消息日志的推送以及对日志接收情况的监控。



### MUTATION

当对ReplicatedMergeTree执行`ALTER DELETE`或者`ALTER UPDATE`操作的时候（ClickHouse把delete和update操作也加入到了alter table的范畴中，它并不支持裸的delete或者update操作），即会进入MUTATION部分的逻辑

![](http://img.orekilee.top//imgbed/clickhouse/ch41.png)

<center>MUTATION的核心执行流程</center>

与MERGE类似，无论MUTATION操作从哪个副本发起，首先都会由主副本进行响应。

1. **推送MUTATION日志**
2. **所有副本实例各自监听MUTATION日志**
3. **由主副本实例响应MUTATION日志并推送Log日志**
4. **各个副本实例分别拉取Log日志**
5. **各个副本实例分别在本地执行MUTATION**

在MUTATION的整个执行过程中，ZooKeeper同样不会进行任何实质性的数据传输。所有的MUTATION操作，最终都是由各个副本在本地完成的。而MUTATION操作是经过/mutations节点实现分发的。CH6负责了消息的推送。但是无论MUTATION动作从哪个副本被触发，之后都会被转交至主副本，再由主副本负责推送Log日志，以通知各个副本执行最终的MUTATION逻辑。同时也由主副本对日志接收的情况实行监控。



###  ALTER

当对ReplicatedMergeTree执行`ALTER`操作进行元数据修改的时候，即会进入ALTER部分的逻辑，例如增加、删除表字段等，核心流程如下图

![](http://img.orekilee.top//imgbed/clickhouse/ch43.png)

<center>ALTER的核心执行流程</center>

ALTER的流程与前几个相比简单很多，其执行过程中并不会涉及/log日志的分发，整个流程大致分成3个步骤

1. **修改共享元数据**
2. **监听共享元数据变更并各自执行本地修改**
3. **确认所有副本完成修改**

在ALTER整个的执行过程中，ZooKeeper不会进行任何实质性的数据传输。所有的ALTER操作，最终都是由各个副本在本地完成的。本着谁执行谁负责的原则，在这个案例中由CH6负责对共享元数据的修改以及对各个副本修改进度的监控。