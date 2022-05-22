---
title: ClickHouse
date: 2022-05-22
image: clickhouse.jpeg
categories:
    - ClickHouse
tags:
    - 列式存储
    - ClickHouse
    - OLAP 
---

## 什么是Clickhouse？

![](http://img.orekilee.top//imgbed/clickhouse/ch.jpeg)

**ClickHouse是一个用于联机分析(OLAP)的列式数据库管理系统(DBMS)。**

### OLAP

#### 什么是OLAP?

**OLAP名为联机分析，又可以称为多维分析**，是由关系型数据库之父埃德加·科德（EdgarFrank Codd）于1993年提出的概念。顾名思义，它指的是**通过多种不同的维度审视数据，进行深层次分析**。

维度可以看作观察数据的一种视角，例如人类能看到的世界是三维的，它包含长、宽、高三个维度。直接一点理解，**维度就好比是一张数据表的字段，而多维分析则是基于这些字段进行聚合查询。**

![](http://img.orekilee.top//imgbed/clickhouse/ch32.png)

<center>多维分析的常见操作</center>

如上图，多维分析包含以下几种操作：

- **下钻：** 从高层次向低层次明细数据穿透，例如从省下钻到市。
- **上卷：** 和下钻相反，从低层次向高层次汇聚，例如从市汇聚成省。
- **切片：** 观察立方体的一层，将一个或多个维度设为单个固定值，然后观察剩余的维度，例如将商品维度固定为足球。
- **切块：** 与切片类似，只是将单个固定值变成多个值。例如将商品维度固定成足球、篮球。
- **旋转：** 旋转立方体的一面，如果要将数据映射到一张二维表，那么就要进行旋转，这就等同于行列置换。



#### OLAP与OLTP

OLTP（on-line transaction processing）翻译为联机事务处理， OLAP（On-Line Analytical Processing）翻译为联机分析处理。

- 从字面上来看OLTP是做事务处理，OLAP是做分析处理。
- 从对数据库操作来看，OLTP主要是对数据的增删改，OLAP是对数据的查询。

- 因为OLTP所产生的业务数据分散在不同的业务系统中，而OLAP往往需要将不同的业务数据集中到一起进行统一综合的分析，这时候就需要根据业务分析需求做对应的数据清洗后存储在数据仓库中，然后由数据仓库来统一提供OLAP分析。
- OLTP是数据库的应用，OLAP是数据仓库的应用

下面用一张图来简要对比。

![](http://img.orekilee.top//imgbed/clickhouse/ch33.png)

<center>OLTP与OLAP</center>

### 列式存储

#### 列式存储与行式存储

在传统的行式数据库系统中，处于同一行中的数据总是被物理的存储在一起，存储方式如下图

![](http://img.orekilee.top//imgbed/clickhouse/ch34.png)

<center>行式存储</center>

在列式数据库系统中，来自不同列的值被单独存储，来自同一列的数据被存储在一起，数据按如下的顺序存储：

![](http://img.orekilee.top//imgbed/clickhouse/ch35.png)

<center>列式存储</center>

![](http://img.orekilee.top//imgbed/clickhouse/ch13.jpg)

<center>按行存储与按列存储的区别</center>

不同的数据存储方式适用不同的业务场景，而对于OLAP来说，列式存储是最适合的选择。



#### 列式存储与OLAP

为什么列式数据库更适合于OLAP场景呢？下面这两张图就可以给你答案

- 行式数据库 
  - ![](http://img.orekilee.top//imgbed/clickhouse/ch15.gif)
- 列式数据库
  - ![](http://img.orekilee.top//imgbed/clickhouse/ch12.gif)

下面分别从两个I/O和CPU两个角度来分析为什么他们有如此之大的差别

- **I/O**
  - 针对分析类查询，通常只需要读取表的一小部分列。在列式数据库中你可以只读取你需要的数据。
  - 由于数据总是打包成批量读取的，所以压缩是非常容易的。同时数据按列分别存储这也更容易压缩。这进一步降低了I/O的体积。
  - 由于I/O的降低，这将帮助更多的数据被系统缓存,进一步降低了数据传输的成本。
- **CPU**
  - 由于执行一个查询需要处理大量的行，因此在整个向量上执行所有操作将比在每一行上执行所有操作更加高效。同时这将有助于实现一个几乎没有调用成本的查询引擎。如果你不这样做，使用任何一个机械硬盘，查询引擎都不可避免的停止CPU进行等待。所以，在数据按列存储并且按列执行是很有意义的。



#### 列式存储与数据压缩

如果你想让查询变得更快，最简单且有效的方法是减少数据扫描范围和数据传输时的大小，而列式存储和数据压缩就可以帮助我们实现上述两点。

列式存储和数据压缩通常是伴生的。数据按列存储。而具体到每个列字段，数据也是独立存储的，每个列字段都拥有一个与之对应的.bin数据文件，相同类型的数据放在同一个文件中，对压缩更加友好。数据默认使用LZ4算法压缩，在Yandex.Metrica的生产环境中，数据总体的压缩比可以达到8:1（未压缩前17PB，压缩后2PB）。



### 核心特点

#### 完备的DBMS功能

ClickHouse拥有完备的管理功能，所以它称得上是一个DBMS（Database Management System，数据库管理系统），而不仅是一个数据库。作为一个DBMS，它具备了一些基本功能，

如下所示。

- **DDL（数据定义语言）**：可以动态地创建、修改或删除数据库、表和视图，而无须重启服务。
- **DML（数据操作语言）**：可以动态查询、插入、修改或删除数据。
- **权限控制**：可以按照用户粒度设置数据库或者表的操作权限，保障数据的安全性。
- **数据备份与恢复**：提供了数据备份导出与导入恢复机制，满足生产环境的要求。
- **分布式管理**：提供集群模式，能够自动管理多个数据库节点。



#### 关系模型与SQL查询

相比HBase和Redis这类NoSQL数据库，ClickHouse使用**关系模型**描述数据并提供了传统数据库的概念（数据库、表、视图和函数等）。与此同时，ClickHouse完全使用**SQL**作为查询语言（支持GROUP BY、ORDER BY、JOIN、IN等大部分标准SQL），这使得它平易近人，容易理解和学习。



#### 向量化表引擎

向量化执行，可以简单地看作从**硬件**的角度上消除程序中循环的优化。

为了实现向量化执行，需要利用CPU的**SIMD**指令。SIMD的全称是Single Instruction MultipleData，即用**单条指令操作多条数据**。现代计算机系统概念中，它是通过数据并行以提高性能的一种实现方式，它的原理是在CPU寄存器层面实现数据的并行操作。例如有8个32位整形数据都需要进行移位运行，则由一条对32位整形数据进行移位的指令重复执行8次完成。SIMD引入了一组大容量的寄存器，一个寄存器包含8 * 32位，可以将这8个数据按次序同时放到一个寄存器。同时，CPU新增了处理这种8 * 32位寄存器的指令，可以在一个指令周期内完成8个数据的位移运算。（本质就是将每次处理的数据从一条变为一批）



#### 多样化的表引擎

与MySQL类似，ClickHouse也将存储部分进行了抽象，把存储引擎作为一层独立的接口。ClickHouse共拥有合并树、内存、文件、接口和其他6大类20多种表引擎。其中每一种表引擎都有着各自的特点，用户可以根据实际业务场景的要求，选择合适的表引擎使用。



#### 多主架构

ClickHouse则采用**Multi-Master多主架构**，集群中的每个节点角色对等，客户端访问任意一个节点都能得到相同的效果。这种多主的架构有许多优势，例如对等的角色使系统架构变得更加简单，不用再区分主控节点、数据节点和计算节点，集群中的所有节点功能相同。所以它天然规避了单点故障的问题，非常适合用于多数据中心、异地多活的场景。



#### 多线程与分布式

在各服务器之间，通过网络传输数据的成本是高昂的，所以相比移动数据，更为聪明的做法是预先将数据分布到各台服务器，将数据的计算查询直接下推到数据所在的服务器。ClickHouse在数据存取方面，既支持分区（纵向扩展，利用多线程原理），也支持分片（横向扩展，利用分布式原理），可以说是将多线程和分布式的技术应用到了极致。



#### 分片与分布式查询

数据分片是将数据进行横向切分，这是一种在面对海量数据的场景下，解决存储和查询瓶颈的有效手段，是一种**分治思想**的体现。ClickHouse支持分片，而分片则依赖集群。每个集群由1到多个分片组成，而每个分片则对应了ClickHouse的1个服务节点。分片的数量上限取决于节点数量（1个分片只能对应1个服务节点）。

ClickHouse并不像其他分布式系统那样，拥有高度自动化的分片功能。ClickHouse提供了**本地表（Local Table）**与**分布式表（Distributed Table）**的概念。一张本地表等同于一份数据的分片，而分布式表本身不存储任何数据，它是本地表的访问代理，其作用类似分库中间件。借助分布式表，能够代理访问多个数据分片，从而实现分布式查询。



### 应用场景

#### 擅长的场景

- 绝大多数是读请求
- 数据以相当大的批次(> 1000行)更新，而不是单行更新;或者根本没有更新。
- 已添加到数据库的数据不能修改。
- 对于读取，从数据库中提取相当多的行，但只提取列的一小部分。
- 宽表，即每个表包含着大量的列
- 查询相对较少(通常每台服务器每秒查询数百次或更少)
- 对于简单查询，允许延迟大约50毫秒
- 列中的数据相对较小：数字和短字符串(例如，每个URL 60个字节)
- 处理单个查询时需要高吞吐量(每台服务器每秒可达数十亿行)
- 事务不是必须的
- 对数据一致性要求低
- 每个查询有一个大表。除了他以外，其他的都很小。
- 查询结果明显小于源数据。换句话说，数据经过过滤或聚合，因此结果适合于单个服务器的RAM中



#### 不擅长的场景

- OLTP事务性操作（不支持事务，不支持真正的更新/删除）
- 不擅长根据主键按行粒度进行查询（如select * from table where user_id in (xxx, xxx, xxx, ...)）
- 不擅长存储和查询 blob 或者大量文本类数据（按列存储）
- 不擅长执行有大量join的查询（Distributed引擎局限）
- 不支持高并发，官方建议QPS <= 100



### Clickhouse为什么会这么快？

首先亮出官方的测试报告：[Clickhouse性能对比报告](https://clickhouse.tech/benchmark/dbms/)

所有用于对比的数据库都使用了相同配置的服务器，在单个节点的情况下，对一张拥有133个字段的数据表分别在1000万、1亿和10亿三种数据体量下执行基准测试，基准测试的范围涵盖43项SQL查询。

![](http://img.orekilee.top//imgbed/clickhouse/ch36.png)

<center>各个存储中间件在一亿数据下的OLAP查询性能对比</center>

市面上有很多与Clickhouse采用了同样技术(如列式存储、向量化引擎等)的数据库，但为什么ClickHouse的性能能够将其他数据库远远甩在身后呢？这主要依赖于下面几个方面

![](http://img.orekilee.top//imgbed/clickhouse/ch1.jpg)

<center>ClickHouse为什么那么快？</center>

- **着眼硬件，先想后做**
  - ClickHouse会在内存中进行GROUP BY，并且使用HashTable装载数据。
  - ClickHouse非常在意CPU L3级别的缓存，因为一次L3的缓存失效会带来70～100ns的延迟。这意味着在单核CPU上，它会浪费4000万次/秒的运算；而在一个32线程的CPU上，则可能会浪费5亿次/秒的运算。
- **算法在前，抽象在后**
  - 对于常量，使用Volnitsky算法；
  - 对于非常量，使用CPU的向量化执行SIMD（用于文本转换、数据过滤、数据解压和JSON转换等），暴力优化；
  - 正则匹配使用re2和hyperscan算法。性能是算法选择的首要考量指标。
- **勇于尝鲜，不行就换**
  - 除了字符串之外，其余的场景也与它类似，ClickHouse会使用最合适、最快的算法。如果世面上出现了号称性能强大的新算法，ClickHouse团队会立即将其纳入并进行验证。如果效果不错，就保留使用；如果性能不尽人意，就将其抛弃。
- **特定场景，特殊优化**
  - 针对同一个场景的不同状况，选择使用不同的实现方式，尽可能将性能最大化。
  - 例如去重计数uniqCombined函数，会根据数据量的不同选择不同的算法：当数据量较小的时候，会选择Array保存；当数据量中等的时候，会选择HashSet；而当数据量很大的时候，则使用HyperLogLog算法。
  - 针对不同的场景，Clickhouse提供了MergeTree引擎家族，如MergeTree、ReplacingMergeTree、SummingMergeTree、AggregatingMergeTree、CollapsingMergeTree和VersionedCollapsingMergeTree等。
- **持续测试，持续改进**
  - 由于Yandex的天然优势，ClickHouse经常会使用真实的数据进行测试，这一点很好地保证了测试场景的真实性。
  - ClickHouse差不多每个月都能发布一个版本，正因为拥有这样的发版频率，ClickHouse才能够快速迭代、快速改进。



### ClickHouse的架构

目前ClickHouse公开的资料相对匮乏，比如在架构设计层面就很难找到完整的资料，甚至连一张整体的架构图都没有，根据官网提供的信息，我们能够得出一个大概的架构，如下图

![](http://img.orekilee.top//imgbed/clickhouse/ch3.jpg)

<center>ClickHouse 架构图 1</center>

![](http://img.orekilee.top//imgbed/clickhouse/ch2.png)

<center>ClickHouse 架构图 2</center>

- **Parser：**Parser分析器可以将一条SQL语句以递归下降的方法解析成AST语法树的形式。不同的SQL语句，会经由不同的Parser实现类解析。

- **Interpreter**：Interpreter解释器的作用就像Service服务层一样，起到串联整个查询过程的作用，它会根据解释器的类型，聚合它所需要的资源。首先它会解析AST对象；然后执行“业务逻辑”（例如分支判断、设置参数、调用接口等）；最终返回`IBlock`对象，以线程的形式建立起一个查询执行管道。

- **Tables**：Tables由 `IStorage` 接口表示。该接口的不同实现对应不同的表引擎。比如 `StorageMergeTree`、`StorageMemory` 等。这些类的实例就是表。

  - IStorage接口定义了DDL（如ALTER、RENAME、OPTIMIZE和DROP等）、read和write方法，它们分别负责数据的定义、查询与写入。在数据查询时，IStorage负责根据AST查询语句的指示要求，返回指定列的原始数据。
  - 后续对数据的进一步加工、计算和过滤，则会统一交由Interpreter解释器对象处理。对Table发起的一次操作通常都会经历这样的过程，接收AST查询语句，根据AST返回指定列的数据，之后再将数据交由Interpreter做进一步处理。

- **Block与Block Streams**：ClickHouse内部的数据操作是面向Block对象进行的，并且采用了流的形式。

  - **Block**：虽然Column和Filed组成了数据的基本映射单元，但对应到实际操作，它们还缺少了一些必要的信息，比如数据的类型及列的名称。于是ClickHouse设计了Block对象，Block对象可以看作数据表的子集。Block对象的本质是由数据对象、数据类型和列名称组成的三元组，即Column、DataType及列名称字符串。Column提供了数据的读取能力，而DataType知道如何正反序列化，所以Block在这些对象的基础之上实现了进一步的抽象和封装，从而简化了整个使用的过程，仅通过Block对象就能完成一系列的数据操作。在具体的实现过程中，Block并没有直接聚合Column和DataType对象，而是通过`ColumnWithTypeAndName`对象进行间接引用。
  - **Block Streams**：Block Streams用于处理数据。我们可以使用Block Streams从某个地方读取数据，执行数据转换，或将数据写到某个地方。`IBlockInputStream` 具有 `read` 方法，其能够在数据可用时获取下一个块。`IBlockOutputStream` 具有 `write` 方法，其能够将块写到某处。

- **Functions**：ClickHouse主要提供两类函数——普通函数和聚合函数。

  - **Function**：普通函数由`IFunction`接口定义，其是没有状态的，函数效果作用于每行数据之上。当然，在函数具体执行的过程中，并不会一行一行地运算，而是采用向量化的方式直接作用于一整列数据。
  - **AggregateFunction**：聚合函数由`IAggregateFunction`接口定义，相比无状态的普通函数，聚合函数是有状态的，并且聚合函数的状态支持序列化与反序列化，所以能够在分布式节点之间进行传输，以实现增量计算。

- **DataType**：数据的序列化和反序列化工作由DataType负责。根据不同的数据类型，`IDataType`接口会有不同的实现类。DataType虽然会对数据进行正反序列化，但是它不会直接和内存或者磁盘做交互，而是转交给Column和Filed处理。

- **Column与Field**：Column和Field是ClickHouse数据最基础的映射单元。

  - **Column**：内存中的一列数据由一个Column对象表示。Column对象分为接口和实现两个部分，在`IColumn`接口对象中，定义了对数据进行各种关系运算的方法，例如插入数据的`insertRangeFrom`和`insertFrom`方法、用于分页的`cut`，以及用于过滤的`filter`方法等。而这些方法的具体实现对象则根据数据类型的不同，由相应的对象实现。
  - **Field**：在大多数场合，ClickHouse都会以整列的方式操作数据，但凡事也有例外。如果需要操作单个具体的数值（也就是单列中的一行数据），则需要使用Field对象，Field对象代表一个单值。与Column对象的泛化设计思路不同，Field对象使用了聚合的设计模式。在Field对象内部聚合了Null、UInt64、String和Array等13种数据类型及相应的处理逻辑。

  

## MergeTree引擎

### 存储结构

![](http://img.orekilee.top//imgbed/clickhouse/ch47.png)

<center>MergeTree的存储结构</center>

- **partition**：**分区目录**，余下各类数据文件（primary.idx、[Column].mrk、[Column]. bin等）都是以分区目录的形式被组织存放的，属于相同分区的数据，最终会被合并到同一个分区目录，而不同分区的数据，永远不会被合并在一起。
- **checksums**：**校验文件**，使用二进制格式存储。它保存了余下各类文件(primary. idx、count.txt等)的size大小及size的哈希值，用于快速校验文件的完整性和正确性。
- **columns.txt**：**列信息文件**，使用明文格式存储，用于保存此数据分区下的列字段信息。
- **count.txt**：**计数文件**，使用明文格式存储，用于记录当前数据分区目录下数据的总行数。
- **primary.idx**：**一级索引文件**，使用二进制格式存储。用于存放稀疏索引，一张MergeTree表只能声明一次一级索引。借助稀疏索引，在数据查询的时能够排除主键条件范围之外的数据文件，从而有效减少数据扫描范围，加速查询速度。
- **[Column].bin**：**数据文件**，使用压缩格式存储，用于存储某一列的数据。由于MergeTree采用列式存储，所以每一个列字段都拥有独立的.bin数据文件，并以列字段名称命名。
- **[Column].mrk**：**使用二进制格式存储**。标记文件中保存了.bin文件中数据的偏移量信息。标记文件与稀疏索引对齐，又与.bin文件一一对应，所以MergeTree通过标记文件建立了primary.idx稀疏索引与.bin数据文件之间的映射关系。即首先通过稀疏索引（primary.idx）找到对应数据的偏移量信息（.mrk），再通过偏移量直接从.bin文件中读取数据。由于.mrk标记文件与.bin文件一一对应，所以MergeTree中的每个列字段都会拥有与其对应的.mrk标记文件
- **[Column].mrk2**：如果使用了自适应大小的索引间隔，则标记文件会以．mrk2命名。它的工作原理和作用与．mrk标记文件相同。
- **partition.dat与minmax_[Column].idx**：如果使用了分区键，例如PARTITION BY EventTime，则会额外生成partition.dat与minmax索引文件，它们均使用二进制格式存储。partition.dat用于保存当前分区下分区表达式最终生成的值；而minmax索引用于记录当前分区下分区字段对应原始数据的最小和最大值。
- **skp_idx_[Column].idx与skp_idx_[Column].mrk**：如果在建表语句中声明了二级索引，则会额外生成相应的二级索引与标记文件，它们同样也使用二进制存储。二级索引在ClickHouse中又称跳数索引。



### 一级索引

#### 稀疏索引

当我们定义主键之后，MergeTree会依据`index_granularity`间隔（默认8192行），为数据表生成一级索引并保存至`primary.idx`文件内，索引数据按照主键排序。相比使用主键定义，更为常见的简化形式是通过ORDER BY指代主键。在此种情形下，主键与ORDER BY定义相同，所以索引（primary.idx）和数据（.bin）会按照完全相同的规则排序。

一级索引底层采用了**稀疏索引**来实现，从下图我们可以看出它和稠密索引的区别。

![](http://img.orekilee.top//imgbed/clickhouse/ch46.png)

<center>稀疏索引与稠密索引的对比</center>

**对于稠密索引而言，每一行索引标记都会对应到具体的一行记录上。而在稀疏索引中，每一行索引标记对应的一大段数据，而不是具体的一行**（他们之间的区别就有点类似mysql中innodb的聚集索引与非聚集索引）。

稀疏索引的优势是显而易见的，它只需要使用少量的索引标记就能够记录大量数据的区间位置信息，并且数据量越大优势愈发明显。例如我们使用默认的索引粒度（8192）时，MergeTree只需要12208行索引标记就能为1亿行数据记录提供索引。由于稀疏索引占用空间小，所以primary.idx内的索引数据能够常驻内存，取用速度自然极快。



#### 索引粒度index_granularity

**索引粒度就如同标尺一般，会丈量整个数据的长度，并依照刻度对数据进行标注，最终将数据标记成多个间隔的小段**。数据以**index_granularity**的粒度(老版本默认8192，新版本实现了自适应粒度)被标记成多个小的区间，其中每个区间最多8192行数据，MergeTree使用**MarkRange**表示一个具体的区间，并通过**start**和**end**表示其具体的范围。

如下图所示。

![](http://img.orekilee.top//imgbed/clickhouse/ch4.png)

<center>MergeTree的存储结构</center>

index_granularity的命名虽然取了索引二字，但它不单只作用于一级索引(.idx)，同时也会影响数据标记(.mrk)和数据文件(.bin)。因为仅有一级索引自身是无法完成查询工作的，它需要借助数据标记才能定位数据，所以一级索引和数据标记的间隔粒度相同(同为index_granularity行)，彼此对齐。而数据文件也会依照index_granularity的间隔粒度生成压缩数据块。



#### 索引的查询过程

**索引查询其实就是两个数值区间的交集判断**。其中，一个区间是由基于主键的查询条件转换而来的条件区间；而另一个区间是刚才所讲述的与MarkRange对应的数值区间。

整个索引的查询过程可以分为三大步骤

1. **生成查询条件区间：** 将查询条件转换为条件区间。即便是单个值的查询条件，也会被转换成区间的形式。

   - ```sql
     --举例--
     WHERE ID = 'A000' 
     =
     ['A000', 'A000']
     
     WHERE ID > 'A000'
     =
     ('A000', '+inf')
     
     WHERE ID < 'A000' 
     =
     ('-inf', 'A000')
     
     WHERE ID LIKE 'A000%' 
     =
     ['A000', 'A001')
     ```

2. **递归交集判断：** 以递归的形式，依次对MarkRange的数值区间与条件区间做交集判断。从最大的区间[A000 , +inf)开始。

   - **如果不存在交集**，则直接通过剪枝算法优化此整段MarkRange
   - **如果存在交集，且MarkRange步长大于N**，则将这个区间进一步拆分为N个子区间，并重复此规则，继续做递归交集判断（N由merge_tree_coarse_index_granularity指定，默认值为8）
   - **如果存在交集，且MarkRange不可再分解**，则记录MarkRange并返回

3. **合并MarkRange区间：** 将最终匹配的MarkRange聚在一起，合并它们的范围。

![](http://img.orekilee.top//imgbed/clickhouse/ch5.png)

<center>索引查询全流程</center>

MergeTree通过递归的形式持续向下拆分区间，最终将MarkRange定位到最细的粒度，以帮助在后续读取数据的时候，能够最小化扫描数据的范围。



#### 联合主键

当我们以需要以多个字段为主键时，此时数据的查询和存储就涉及到另外一种规则。

例如以 `(CounterID, Date)` 以主键，片段中数据首先按 `CounterID` 排序，具有相同 `CounterID` 的部分按 `Date` 排序。排序好的索引的图示会是下面这样：

```sql
全部数据  :     [-------------------------------------------------------------------------]
CounterID:      [aaaaaaaaaaaaaaaaaabbbbcdeeeeeeeeeeeeefgggggggghhhhhhhhhiiiiiiiiikllllllll]
Date:           [1111111222222233331233211111222222333211111112122222223111112223311122333]
标记:            |      |      |      |      |      |      |      |      |      |      |
                a,1    a,2    a,3    b,3    e,2    e,3    g,1    h,2    i,1    i,3    l,3
标记号:          0      1      2      3      4      5      6      7      8      9      10

```

如果指定查询如下：

- `CounterID in ('a', 'h')`，服务器会读取标记号在 `[0, 3)` 和 `[6, 8)` 区间中的数据。
- `CounterID IN ('a', 'h') AND Date = 3`，服务器会读取标记号在 `[1, 3)` 和 `[7, 8)` 区间中的数据。
- `Date = 3`，服务器会读取标记号在 `[1, 10]` 区间中的数据。

上面例子可以看出使用索引通常会比全表描述要高效。

- 稀疏索引会引起额外的数据读取。当读取主键单个区间范围的数据时，每个数据块中最多会多读 `index_granularity * 2` 行额外的数据。

- 稀疏索引使得你可以处理极大量的行，因为大多数情况下，这些索引常驻与内存（RAM）中。

从上面可以看出，ClickHouse的联合主键在某种程度上与我们熟知的最左前缀规则有点类似，通常在以下几种场景下我们才会考虑使用联合索引

- 查询会使用 `b` 列作为条件
- 很长的数据范围（ `index_granularity` 的数倍）里 `a` 都是相同的值，并且这样的情况很普遍。换言之，就是加入另一列后，可以让你的查询略过很长的数据范围。
- 数据量大，需要改善数据压缩（以主键排序片段数据，数据的一致性越高，压缩越好）

长的主键会对插入性能和内存消耗有负面影响，但主键中额外的列并不影响 `SELECT` 查询的性能。



### 二级索引

除了一级索引之外，MergeTree同样支持二级索引。二级索引又称**跳数索引**，由数据的聚合信息构建而成。根据索引类型的不同，其聚合信息的内容也不同。跳数索引的目的与一级索引一样，也是帮助查询时减少数据扫描的范围。

**（二级索引目前还处于测试阶段，官方不建议大量使用）**

#### 跳数索引

目前，MergeTree共支持4种跳数索引，分别是**minmax（最值）、set（集合行数）、ngrambf_v1（N-Gram布隆过滤器）和tokenbf_v1（Token布隆过滤器**）。一张数据表支持同时声明多个跳数索引。

- **minmax（最值索引）**：minmax索引记录了一段数据内的**最小和最大极值**，其索引的作用类似分区目录的minmax索引，能够快速跳过无用的数据区间。

  - ```sql
    示例：INDEX [index_name] [column] TYPE minmax GRANULARITY [GRANULARITY SIZE]
    ```

- **set（集合行数索引）**：set索引直接记录了声明字段或表达式的**不重复值**，用于检测数据块是否满足`WHERE`条件。

  - ```sql
    示例：INDEX [index_name] [column] TYPE set(max_rows) GRANULARITY [index_granularity]
    
    -- max_rows是一个阈值，表示在一个index_granularity内，索引最多记录的数据行数。(如果max_rows=0，则表示无限制)
    ```

- **ngrambf_v1（N-Gram布隆过滤器）**：ngrambf_v1索引记录的是**指定长度的数据短语的布隆表过滤器**，只支持String和FixedString数据类型，同时只能够提升`in`、`notIn`、`like`、`equals`和`notEquals`查询的性能。

  - ```sql
    示例：INDEX [index_name] [column] TYPE ngrambf_v1(n, size_of_bloom_filter_in_bytes, number_of_hash_functions, random_seed) GRANULARITY [index_granularity]
    
    /*
    n:token长度，依据n的长度将数据切割为token短语。
    size_of_bloom_filter_in_bytes：布隆过滤器的大小。
    number_of_hash_functions：布隆过滤器中使用Hash函数的个数。
    random_seed: Hash函数的随机种子。
    */
    ```

  - ```sql
    布隆过滤器可能会包含不符合条件的匹配，所以 ngrambf_v1, tokenbf_v1 和 bloom_filter 索引不能用于负向的函数，例如：
    
    --可以用来优化的场景
    s LIKE '%test%'
    NOT s NOT LIKE '%test%'
    s = 1
    NOT s != 1
    startsWith(s, 'test')
    i
    --不能用来优化的场景
    NOT s LIKE '%test%'
    s NOT LIKE '%test%'
    NOT s = 1
    s != 1
    NOT startsWith(s, 'test')
    
    ```

- **tokenbf_v1（Token布隆过滤器）**：tokenbf_v1索引是ngrambf_v1的变种，同样也是一种布隆过滤器索引。tokenbf_v1除了短语token的处理方法外，其他与ngrambf_v1是完全一样的。tokenbf_v1会自动按照**非字符的、数字的字符串**分割token。

  - ```sql
    示例：INDEX d ID TYPE tokenbf_v1(size_of_bloom_filter_in_bytes, number_of_hash_functions, random_seed)
    ```

    

#### granularity

对于跳数索引而言，**index_granularity定义了数据的粒度**，而**granularity定义了聚合信息汇总的粒度**。换言之，**granularity定义了一行跳数索引能够跳过多少个index_granularity区间的数据。**

**作用规则如下**：首先，按照index_granularity粒度间隔将数据划分成n段，总共有[0 , n-1]个区间（n = total_rows /index_granularity，向上取整）。接着，根据索引定义时声明的表达式，从0区间开始，依次按index_granularity粒度从数据中获取聚合信息，每次向前移动1步(n+1)，聚合信息逐步累加。最后，当移动granularity次区间时，则汇总并生成一行跳数索引数据。

以minmax索引为例，假设index_granularity=8192且granularity=3，则数据会按照index_granularity划分为n等份，MergeTree从第0段分区开始，依次获取聚合信息。当获取到第3个分区时（granularity=3），则汇总并会生成第一行minmax索引（前3段minmax极值汇总后取值为[1 , 9]），如下图

![](http://img.orekilee.top//imgbed/clickhouse/ch8.png)

<center>granularity作用规则</center>


### 数据标记

如果把MergeTree比作一本书，primary.idx一级索引好比这本书的一级章节目录，.bin文件中的数据好比这本书中的文字，那么数据标记(.mrk)就好比书签一样，会为一级章节目录和具体的文字之间建立关联。

对于数据标记而言，它记录了两点重要信息：

- 一级章节对应的页码信息。
- 一段文字在某一页中的起始位置信息。

这样一来，通过数据标记就能够很快地从一本书中立即翻到关注内容所在的那一页，并知道从第几行开始阅读。

标记数据与一级索引数据不同，它并不能常驻内存，而是使用**LRU（最近最少使用）缓存策略**加快其取用速度。

#### 生成规则

![](http://img.orekilee.top//imgbed/clickhouse/ch29.png)

<center>通过索引下标编号找到对应的数据标记</center>

从上图可以看出，数据标记和索引区间是对齐的，均按照`index_granularity`的粒度间隔。如此一来，只需简单通过索引区间的下标编号就可以直接找到对应的数据标记。

为了能够与数据衔接，数据标记文件也与.bin文件一一对应。即每一个列字段[Column].bin文件都有一个与之对应的[Column].mrk数据标记文件，用于记录数据在.bin文件中的偏移量信息。同时，.mrk包含了.bin压缩和解压缩这两种不同状态的偏移量，如下图

![](http://img.orekilee.top//imgbed/clickhouse/ch30.png)

<center>标记数据示意图</center>



#### 工作方式

MergeTree在读取数据时，必须通过标记数据的位置信息才能够找到所需要的数据。整个查找过程大致可以分为**读取压缩数据块**和**读取数据**两个步骤。

对于下图来说，表的`index_granularity`粒度为8192，所以一个索引片段的数据大小恰好是8192B。按照压缩数据块的生成规则，如果单个批次数据小于64KB，则继续获取下一批数据，直至累积到size>=64KB时，生成下一个压缩数据块。因此在JavaEnable的标记文件中，每8行标记数据对应1个压缩数据块（1B * 8192 = 8192B, 64KB = 65536B, 65536 / 8192 =8）。

从图能够看到，其左侧的标记数据中，8行数据的压缩文件偏移量都是相同的，因为这8行标记都指向了同一个压缩数据块。而在这8行的标记数据中，它们的解压缩数据块中的偏移量，则依次按照8192B（每行数据1B，每一个批次8192行数据）累加，当累加达到65536(64KB)时则置0。因为根据规则，此时会生成下一个压缩数据块。

![](http://img.orekilee.top//imgbed/clickhouse/ch31.png)

<center>JavaEnable字段的标记文件和压缩数据文件的对应关系</center>

1. **读取压缩数据块：** 在查询某一列数据时，MergeTree无须一次性加载整个.bin文件，而是可以根据需要，只加载特定的压缩数据块。而这项特性需要借助标记文件中所保存的压缩文件中的偏移量。

2. **读取数据：** 在读取解压后的数据时，MergeTree并不需要一次性扫描整段解压数据，它可以根据需要，以`index_granularity`的粒度加载特定的一小段。为了实现这项特性，需要借助标记文件中保存的解压数据块中的偏移量。



#### 数据标记与压缩数据块的对应关系

由于压缩数据块的划分，与一个间隔`index_granularity`内的数据大小相关，每个压缩数据块的体积都被严格控制在64KB～1MB。而一个间隔`index_granularity`的数据，又只会产生一行数据标记。那么根据一个间隔内数据的实际字节大小，数据标记和压缩数据块之间会产生三种不同的对应关系。

- **一对一**
  - 一个数据标记对应一个压缩数据块，当一个间隔`index_granularity`内的数据未压缩大小size大于等于64KB且小于等于1MB时，会出现这种对应关系。
  - ![](http://img.orekilee.top//imgbed/clickhouse/ch28.png)
- **一对多**
  - 一个数据标记对应多个压缩数据块，当一个间隔`index_granularity`内的数据未压缩大小size直接大于1MB时，会出现这种对应关系。
  - ![](http://img.orekilee.top//imgbed/clickhouse/ch26.png)
- **多对一**
  - 多个数据标记对应一个压缩数据块，当一个间隔`index_granularity`内的数据未压缩大小size小于64KB时，会出现这种对应关系。
  - ![](http://img.orekilee.top//imgbed/clickhouse/ch27.png)



### 工作流程

#### 存储流程

数据的存储流程主要有以下几个步骤

- 首先生成分区目录，伴随着每一批数据的写入，都会生成一个新的分区目录。

- 在后续的某一时刻，属于相同分区的目录会依照规则合并到一起
- 接着，按照`index_granularity`索引粒度，会分别生成primary.idx一级索引（如果声明了二级索引，还会创建二级索引文件）、每一个列字段的．mrk数据标记和．bin压缩数据文件。

![](http://img.orekilee.top//imgbed/clickhouse/ch23.png)

<center>分区目录、索引、标记和压缩数据的生成过程示意</center>



#### 查询流程

数据查询的本质，可以看作一个不断减小数据范围的过程。在最理想的情况下，MergeTree首先可以依次借助分区索引、一级索引和二级索引，将数据扫描范围缩至最小。然后再借助数据标记，将需要解压与计算的数据范围缩至最小。

![](http://img.orekilee.top//imgbed/clickhouse/ch24.png)

<center>将扫描数据范围最小化的过程</center>

如果一条查询语句没有指定任何WHERE条件，或是指定了WHERE条件，但条件没有匹配到任何索引（分区索引、一级索引和二级索引），那么MergeTree就不能预先减小数据范围。在后续进行数据查询时，它会扫描所有分区目录，以及目录内索引段的最大区间。虽然不能减少数据范围，但是MergeTree仍然能够借助数据标记，以多线程的形式同时读取多个压缩数据块，以提升性能。



## ReplicatedMergeTree引擎

ReplicatedMergeTree是MergeTree的派生引擎，它在MergeTree的基础上加入了分布式协同的能力，只有使用了ReplicatedMergeTree复制表系列引擎，才能应用副本的能力。或者用一种更为直接的方式理解，即使用ReplicatedMergeTree的数据表就是副本。

![](http://img.orekilee.top//imgbed/clickhouse/ch37.png)

<center>ReplicatedMergeTree与MergeTree的逻辑关系</center>

在MergeTree中，一个数据分区由开始创建到全部完成，会历经两类存储区域。

- **内存**：数据首先会被写入内存缓冲区。
- **本地磁盘**：数据接着会被写入tmp临时目录分区，待全部完成后再将临时目录重命名为正式分区。

ReplicatedMergeTree在上述基础之上增加了ZooKeeper的部分，它会进一步在ZooKeeper内创建一系列的监听节点，并以此实现多个实例之间的通信。在整个通信过程中，ZooKeeper并不会涉及表数据的传输。



### 特点

作为数据副本的主要实现载体，ReplicatedMergeTree在设计上有一些显著特点。

- **依赖ZooKeeper**：在执行`INSERT`和`ALTER`查询的时候，ReplicatedMergeTree需要借助ZooKeeper的分布式协同能力，以实现多个副本之间的同步。但是在查询副本的时候，并不需要使用ZooKeeper。

- **表级别的副本**：副本是在表级别定义的，所以每张表的副本配置都可以按照它的实际需求进行个性化定义，包括副本的数量，以及副本在集群内的分布位置等。

- **多主架构（Multi Master）**：可以在任意一个副本上执行`INSERT`和`ALTER`查询，它们的效果是相同的。这些操作会借助ZooKeeper的协同能力被分发至每个副本以本地形式执行。

- **Block数据块**：在执行`INSERT`命令写入数据时，会依据`max_insert_block_size`的大小（默认1048576行）将数据切分成若干个Block数据块。所以Block数据块是数据写入的基本单元，并且具有写入的原子性和唯一性。

- **原子性**：在数据写入时，一个Block块内的数据要么全部写入成功，要么全部失败。

- **唯一性**：在写入一个Block数据块的时候，会按照当前Block数据块的数据顺序、数据行和数据大小等指标，计算Hash信息摘要并记录在案。在此之后，如果某个待写入的Block数据块与先前已被写入的Block数据块拥有相同的Hash摘要（Block数据块内数据顺序、数据大小和数据行均相同），则该Block数据块会被忽略。

  

### 数据结构

#### ZooKeeper内的节点结构

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



#### Entry日志对象的数据结构

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



### 副本协同的核心流程

副本协同的核心流程主要有`INSERT`、`MERGE`、`MUTATION`和`ALTER`四种，分别对应了数据写入、分区合并、数据修改和元数据修改。`INSERT`和`ALTER`是分布式执行的，借助ZooKeeper的事件通知机制，多个副本之间会自动进行有效协同，但是它们不会使用ZooKeeper存储任何分区数据。而其他操作并不支持分布式执行，包括`SELECT`、`CREATE`、`DROP`、`RENAME`和`ATTACH`。

在下列例子中，使用ReplicatedMergeTree实现一张拥有1分片、1副本的数据表来分别执行`INSERT`、`MERGE`、`MUTATION`和`ALTER`操作，演示执行流程。

#### INSERT

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



#### MERGE

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



#### MUTATION

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



####  ALTER

当对ReplicatedMergeTree执行`ALTER`操作进行元数据修改的时候，即会进入ALTER部分的逻辑，例如增加、删除表字段等，核心流程如下图

![](http://img.orekilee.top//imgbed/clickhouse/ch43.png)

<center>ALTER的核心执行流程</center>

ALTER的流程与前几个相比简单很多，其执行过程中并不会涉及/log日志的分发，整个流程大致分成3个步骤

1. **修改共享元数据**
2. **监听共享元数据变更并各自执行本地修改**
3. **确认所有副本完成修改**

在ALTER整个的执行过程中，ZooKeeper不会进行任何实质性的数据传输。所有的ALTER操作，最终都是由各个副本在本地完成的。本着谁执行谁负责的原则，在这个案例中由CH6负责对共享元数据的修改以及对各个副本修改进度的监控。



## Distributed引擎

Distributed表引擎是分布式表的代名词，**它自身不存储任何数据，而是作为数据分片的透明代理，能够自动路由数据至集群中的各个节点**，所以Distributed表引擎需要和其他数据表引擎一起协同工作。

ClickHouse并不像其他分布式系统那样，拥有高度自动化的分片功能。ClickHouse提供了**本地表（Local Table）**与**分布式表（Distributed Table）**的概念

![](http://img.orekilee.top//imgbed/clickhouse/ch14.png)

<center>由Distributed表将数据写入多个分片</center>

- **本地表**：通常以_local为后缀进行命名。本地表是承接数据的载体，可以使用非Distributed的任意表引擎，一张本地表对应了一个数据分片。
- **分布式表**：通常以_all为后缀进行命名。分布式表只能使用Distributed表引擎，它与本地表形成一对多的映射关系，日后将通过分布式表代理操作多张本地表。



### 分布式写入流程

在向集群内的分片写入数据时，通常有两种思路

- **借助外部计算系统，事先将数据均匀分片，再借由计算系统直接将数据写入ClickHouse集群的各个本地表。**

- **通过Distributed表引擎代理写入分片数据。**

第一种方案通常拥有更好的写入性能，因为分片数据是被并行点对点写入的。但是这种方案的实现主要依赖于外部系统，而不在于ClickHouse自身，所以这里主要会介绍第二种思路。为了便于理解整个过程，这里会将分片写入、副本复制拆分成两个部分进行讲解。

#### 数据写入分片

![](http://img.orekilee.top//imgbed/clickhouse/ch21.png)

<center>由Distributed表将数据写入多个分片</center>

1. **在第一个分片节点写入本地分片数据**：首先在CH5节点，对分布式表test_shard_2_all执行INSERT查询，尝试写入10、30、200和55四行数据。执行之后分布式表主要会做两件事情：

   - 根据分片规则划分数据

   - 将属于当前分片的数据直接写入本地表test_shard_2_local。

2. **第一个分片建立远端连接，准备发送远端分片数据**：将归至远端分片的数据以分区为单位，分别写入/test_shard_2_all存储目录下的临时bin文件，接着，会尝试与远端分片节点建立连接。

3. **第一个分片向远端分片发送数据**：此时，会有另一组监听任务负责监听/test_shard_2_all目录下的文件变化，这些任务负责将目录数据发送至远端分片，其中，每份目录将会由独立的线程负责发送，数据在传输之前会被压缩。

4. **第二个分片接收数据并写入本地**：CH6分片节点确认建立与CH5的连接，在接收到来自CH5发送的数据后，将它们写入本地表。

5. **由第一个分片确认完成写入**：最后，还是由CH5分片确认所有的数据发送完毕。

可以看到，在整个过程中，Distributed表负责所有分片的写入工作。本着谁执行谁负责的原则，在这个示例中，由CH5节点的分布式表负责切分数据，并向所有其他分片节点发送数据。

在由Distributed表负责向远端分片发送数据时，有异步写和同步写两种模式：如果是异步写，则在Distributed表写完本地分片之后，INSERT查询就会返回成功写入的信息；如果是同步写，则在执行INSERT查询之后，会等待所有分片完成写入。



#### 副本复制数据

如果在集群的配置中包含了副本，那么除了刚才的分片写入流程之外，还会触发副本数据的复制流程。数据在多个副本之间，有两种复制实现方式：

- **Distributed表引擎**：副本数据的写入流程与分片逻辑相同，所以Distributed会同时负责分片和副本的数据写入工作。但在这种实现方案下，它很有可能会成为写入的单点瓶颈，所以就有了接下来将要说明的第二种方案。

- **ReplicatedMergeTree表引擎**：如果使用ReplicatedMergeTree作为本地表的引擎，则在该分片内，多个副本之间的数据复制会交由ReplicatedMergeTree自己处理，不再由Distributed负责，从而为其减负。

![](http://img.orekilee.top//imgbed/clickhouse/ch22.png)

<center>使用Distributed与ReplicatedMergeTree分发副本数据的对比示意图</center>



### 分布式查询流程

与数据写入有所不同，在面向集群查询数据的时候，**只能**通过Distributed表引擎实现。当Distributed表接收到`SELECT`查询的时候，它会依次查询每个分片的数据，再合并汇总返回，流程如下：

#### 多副本的路由规则

在查询数据的时候，如果集群中的某一个分片有多个副本，此时Distributed引擎就会通过负载均衡算法从众多的副本中选取一个，负载均衡算法有以下四种。

在ClickHouse的服务节点中，拥有一个全局计数器`errors_count`，当服务发生任何异常时，该计数累积加1。

1. **random（默认）**：random算法会选择`errors_coun`t错误数量最少的副本，如果多个副本的`errors_count`计数相同，则在它们之中**随机**选择一个。
2. **nearest_hostname**：nearest_hostname可以看作random算法的变种，首先它会选择`errors_count`错误数量最少的副本，如果多个replica的`errors_count`计数相同，**则选择集群配置中host名称与当前host最相似的一个。而相似的规则是以当前host名称为基准按字节逐位比较，找出不同字节数最少的一个**。
3. **in_order**：in_order同样可以看作random算法的变种，首先它会选择`errors_count`错误数量最少的副本，如果多个副本的`errors_count`计数相同，则**按照集群配置中replica的定义顺序逐个选择**。
4. **first_or_random**：first_or_random可以看作in_order算法的变种，首先它会选择`errors_count`错误数量最少的副本，如果多个副本的`errors_count`计数相同，**它首先会选择集群配置中第一个定义的副本，如果该副本不可用，则进一步随机选择一个其他的副本**。



#### 多分片查询的流程

分布式查询与分布式写入类似，同样本着谁执行谁负责的原则，它会由接收`SELECT`查询的Distributed表，并负责串联起整个过程。首先它会将针对分布式表的SQL语句，按照分片数量将查询拆分成若干个针对本地表的子查询，然后向各个分片发起查询，最后再汇总各个分片的返回结果。

```sql
--查询分布式表
SELECT * FROM distributor_table

--转换为查询本地表，并将该命令推送到各个分片节点上执行
SELECT * FROM local_table
```

如下图

![](http://img.orekilee.top//imgbed/clickhouse/ch9.png)



<center>对分布式表执行查询的执行计划</center>

1. **查询各个分片数据**：One和Remote步骤是并行执行的，它们分别负责了本地和远端分片的查询动作。
2. **合并返回结果**：多个分片数据均查询返回后，在执行节点将所有数据`union`合并



#### 使用Global优化分布式子查询

如果现在有一项查询需求，例如要求找到同时拥有两个仓库的用户，应该如何实现？对于这类交集查询的需求，可以使用IN子查询，此时你会面临两难的选择：**IN查询的子句应该使用本地表还是分布式表？（使用JOIN面临的情形与IN类似）。**

<font size=4>**使用本地表的问题（可能查询不到结果）**</font>

如果在IN查询中使用本地表时，如下列语句

```sql
SELECT uniq(id) FROM distributed_table WHERE repo = 100 AND id IN (SELECT id FROM local_table WHERE repo = 200)
```

在实际执行时，分布式表在接收到查询后会将上述SQL替换成本地表的形式，再发送到每个分片进行执行，此时，每个分片上实际执行的是以下语句

```sql
SELECT uniq(id) FROM local_table WHERE repo = 100 AND id IN (SELECT id FROM local_table WHERE repo = 200)
```

**那么此时查询的最终结果就有可能是错误的，因为在单个分片上只保存了部分的数据，这就导致该SQL语句可能没有匹配到任何数据**，如下图

![](http://img.orekilee.top//imgbed/clickhouse/ch17.png)

<center>使用本地表作为IN查询子句的执行逻辑</center



**使用分布式表的问题（查询请求被放大N^2倍，N为节点数量**

如果在IN查询中使用本地表时，如下列语句

```sql
SELECT uniq(id) FROM distributed_table WHERE repo = 100 AND id IN (SELECT id FROM distributed_table WHERE repo = 200)
```

对于此次查询，每个分片节点不仅需要查询本地表，还需要再次向其他的分片节点再次发起远端查询，如下图

![](http://img.orekilee.top//imgbed/clickhouse/ch18.png)

<center>IN查询子句查询放大原因示意</center>

因此可以得出结论，**在IN查询子句使用分布式表的时候，虽然查询的结果得到了保证，但是查询请求会被放大N的平方倍**，其中N等于集群内分片节点的数量，假如集群内有10个分片节点，则在一次查询的过程中，会最终导致100次的查询请求，这显然是不可接受的。



<font size=4>**使用GLOBAL优化查询**</font>

为了解决查询放大的问题，我们可以使用GLOBAL IN或GLOBAL JOIN进行优化，下面就简单介绍一下GLOBAL的执行流程

```sql
SELECT uniq(id) FROM distributed_table WHERE repo = 100 AND id GLOBAL IN (SELECT id FROM distributed_table WHERE repo = 200)
```

![](http://img.orekilee.top//imgbed/clickhouse/ch19.png)

<center>使用GLOBAL IN查询的流程示意图</center>

如上图，主要有以下五个步骤

1. 将IN子句单独提出，发起了一次分布式查询。
2. 将分布式表转local本地表后，分别在本地和远端分片执行查询。
3. 将IN子句查询的结果进行汇总，并放入一张临时的内存表进行保存。
4. 将内存表发送到远端分片节点。
5. 将分布式表转为本地表后，开始执行完整的SQL语句，IN子句直接使用临时内存表的数据。

**在使用GLOBAL修饰符之后，ClickHouse使用内存表临时保存了IN子句查询到的数据，并将其发送到远端分片节点，以此到达了数据共享的目的，从而避免了查询放大的问题**。由于数据会在网络间分发，所以需要特别注意临时表的大小，IN或者JOIN子句返回的数据不宜过大。如果表内存在重复数据，也可以事先在子句SQL中增加DISTINCT以实现去重。