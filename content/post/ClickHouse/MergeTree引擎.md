---
title: ClickHouse 数据存储原理：MergeTree引擎
date: 2022-05-23T18:18:13+08:00
categories:
    - ClickHouse
tags:
    - 列式存储
    - ClickHouse
    - OLAP 
---

# MergeTree 引擎

## 存储结构

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



## 一级索引

### 稀疏索引

当我们定义主键之后，MergeTree会依据`index_granularity`间隔（默认8192行），为数据表生成一级索引并保存至`primary.idx`文件内，索引数据按照主键排序。相比使用主键定义，更为常见的简化形式是通过ORDER BY指代主键。在此种情形下，主键与ORDER BY定义相同，所以索引（primary.idx）和数据（.bin）会按照完全相同的规则排序。

一级索引底层采用了**稀疏索引**来实现，从下图我们可以看出它和稠密索引的区别。

![](http://img.orekilee.top//imgbed/clickhouse/ch46.png)

<center>稀疏索引与稠密索引的对比</center>

**对于稠密索引而言，每一行索引标记都会对应到具体的一行记录上。而在稀疏索引中，每一行索引标记对应的一大段数据，而不是具体的一行**（他们之间的区别就有点类似mysql中innodb的聚集索引与非聚集索引）。

稀疏索引的优势是显而易见的，它只需要使用少量的索引标记就能够记录大量数据的区间位置信息，并且数据量越大优势愈发明显。例如我们使用默认的索引粒度（8192）时，MergeTree只需要12208行索引标记就能为1亿行数据记录提供索引。由于稀疏索引占用空间小，所以primary.idx内的索引数据能够常驻内存，取用速度自然极快。



### 索引粒度index_granularity

**索引粒度就如同标尺一般，会丈量整个数据的长度，并依照刻度对数据进行标注，最终将数据标记成多个间隔的小段**。数据以**index_granularity**的粒度(老版本默认8192，新版本实现了自适应粒度)被标记成多个小的区间，其中每个区间最多8192行数据，MergeTree使用**MarkRange**表示一个具体的区间，并通过**start**和**end**表示其具体的范围。

如下图所示。

![](http://img.orekilee.top//imgbed/clickhouse/ch4.png)

<center>MergeTree的存储结构</center>

index_granularity的命名虽然取了索引二字，但它不单只作用于一级索引(.idx)，同时也会影响数据标记(.mrk)和数据文件(.bin)。因为仅有一级索引自身是无法完成查询工作的，它需要借助数据标记才能定位数据，所以一级索引和数据标记的间隔粒度相同(同为index_granularity行)，彼此对齐。而数据文件也会依照index_granularity的间隔粒度生成压缩数据块。



### 索引的查询过程

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



### 联合主键

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



## 二级索引

除了一级索引之外，MergeTree同样支持二级索引。二级索引又称**跳数索引**，由数据的聚合信息构建而成。根据索引类型的不同，其聚合信息的内容也不同。跳数索引的目的与一级索引一样，也是帮助查询时减少数据扫描的范围。

==（二级索引目前还处于测试阶段，官方不建议大量使用）==

### 跳数索引

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

    

### granularity

对于跳数索引而言，**index_granularity定义了数据的粒度**，而**granularity定义了聚合信息汇总的粒度**。换言之，**granularity定义了一行跳数索引能够跳过多少个index_granularity区间的数据。**

**作用规则如下**：首先，按照index_granularity粒度间隔将数据划分成n段，总共有[0 , n-1]个区间（n = total_rows /index_granularity，向上取整）。接着，根据索引定义时声明的表达式，从0区间开始，依次按index_granularity粒度从数据中获取聚合信息，每次向前移动1步(n+1)，聚合信息逐步累加。最后，当移动granularity次区间时，则汇总并生成一行跳数索引数据。

以minmax索引为例，假设index_granularity=8192且granularity=3，则数据会按照index_granularity划分为n等份，MergeTree从第0段分区开始，依次获取聚合信息。当获取到第3个分区时（granularity=3），则汇总并会生成第一行minmax索引（前3段minmax极值汇总后取值为[1 , 9]），如下图

![](http://img.orekilee.top//imgbed/clickhouse/ch8.png)

<center>granularity作用规则</center>


## 数据标记

如果把MergeTree比作一本书，primary.idx一级索引好比这本书的一级章节目录，.bin文件中的数据好比这本书中的文字，那么数据标记(.mrk)就好比书签一样，会为一级章节目录和具体的文字之间建立关联。

对于数据标记而言，它记录了两点重要信息：

- 一级章节对应的页码信息。
- 一段文字在某一页中的起始位置信息。

这样一来，通过数据标记就能够很快地从一本书中立即翻到关注内容所在的那一页，并知道从第几行开始阅读。

标记数据与一级索引数据不同，它并不能常驻内存，而是使用**LRU（最近最少使用）缓存策略**加快其取用速度。

### 生成规则

![](http://img.orekilee.top//imgbed/clickhouse/ch29.png)

<center>通过索引下标编号找到对应的数据标记</center>

从上图可以看出，数据标记和索引区间是对齐的，均按照`index_granularity`的粒度间隔。如此一来，只需简单通过索引区间的下标编号就可以直接找到对应的数据标记。

为了能够与数据衔接，数据标记文件也与.bin文件一一对应。即每一个列字段[Column].bin文件都有一个与之对应的[Column].mrk数据标记文件，用于记录数据在.bin文件中的偏移量信息。同时，.mrk包含了.bin压缩和解压缩这两种不同状态的偏移量，如下图

![](http://img.orekilee.top//imgbed/clickhouse/ch30.png)

<center>标记数据示意图</center>



### 工作方式

MergeTree在读取数据时，必须通过标记数据的位置信息才能够找到所需要的数据。整个查找过程大致可以分为**读取压缩数据块**和**读取数据**两个步骤。

对于下图来说，表的`index_granularity`粒度为8192，所以一个索引片段的数据大小恰好是8192B。按照压缩数据块的生成规则，如果单个批次数据小于64KB，则继续获取下一批数据，直至累积到size>=64KB时，生成下一个压缩数据块。因此在JavaEnable的标记文件中，每8行标记数据对应1个压缩数据块（1B * 8192 = 8192B, 64KB = 65536B, 65536 / 8192 =8）。

从图能够看到，其左侧的标记数据中，8行数据的压缩文件偏移量都是相同的，因为这8行标记都指向了同一个压缩数据块。而在这8行的标记数据中，它们的解压缩数据块中的偏移量，则依次按照8192B（每行数据1B，每一个批次8192行数据）累加，当累加达到65536(64KB)时则置0。因为根据规则，此时会生成下一个压缩数据块。

![](http://img.orekilee.top//imgbed/clickhouse/ch31.png)

<center>JavaEnable字段的标记文件和压缩数据文件的对应关系</center>

1. **读取压缩数据块：** 在查询某一列数据时，MergeTree无须一次性加载整个.bin文件，而是可以根据需要，只加载特定的压缩数据块。而这项特性需要借助标记文件中所保存的压缩文件中的偏移量。

2. **读取数据：** 在读取解压后的数据时，MergeTree并不需要一次性扫描整段解压数据，它可以根据需要，以`index_granularity`的粒度加载特定的一小段。为了实现这项特性，需要借助标记文件中保存的解压数据块中的偏移量。



### 数据标记与压缩数据块的对应关系

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



## 工作流程

### 存储流程

数据的存储流程主要有以下几个步骤

- 首先生成分区目录，伴随着每一批数据的写入，都会生成一个新的分区目录。

- 在后续的某一时刻，属于相同分区的目录会依照规则合并到一起
- 接着，按照`index_granularity`索引粒度，会分别生成primary.idx一级索引（如果声明了二级索引，还会创建二级索引文件）、每一个列字段的．mrk数据标记和．bin压缩数据文件。

![](http://img.orekilee.top//imgbed/clickhouse/ch23.png)

<center>分区目录、索引、标记和压缩数据的生成过程示意</center>



### 查询流程

数据查询的本质，可以看作一个不断减小数据范围的过程。在最理想的情况下，MergeTree首先可以依次借助分区索引、一级索引和二级索引，将数据扫描范围缩至最小。然后再借助数据标记，将需要解压与计算的数据范围缩至最小。

![](http://img.orekilee.top//imgbed/clickhouse/ch24.png)

<center>将扫描数据范围最小化的过程</center>

如果一条查询语句没有指定任何WHERE条件，或是指定了WHERE条件，但条件没有匹配到任何索引（分区索引、一级索引和二级索引），那么MergeTree就不能预先减小数据范围。在后续进行数据查询时，它会扫描所有分区目录，以及目录内索引段的最大区间。虽然不能减少数据范围，但是MergeTree仍然能够借助数据标记，以多线程的形式同时读取多个压缩数据块，以提升性能。