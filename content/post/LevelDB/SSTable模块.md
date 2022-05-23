---
title: LevelDB SSTable模块
date: 2022-05-23T23:41:13+08:00
categories:
    - LevelDB
tags:
    - LevelDB
    - KV存储
---

  # SSTable模块

SSTable（Sorted Strings Table，有序字符串表），在各种存储引擎中得到了广泛的使用，包括 LevelDB、HBase、Cassandra 等。SSTable 会根据 Key 进行排序后保存一系列的 K-V 对，这种方式不仅方便进行范围查找，而且便于对 K-V 对进行更加有效的压缩。

  ## SSTable Format

SSTable 文件由一个个块组成，块中可以保存数据、数据索引、元数据或者元数据索引。整体的文件格式如下图：

  ![](http://img.orekilee.top//imgbed/leveldb/leveldb6.png)

  <center>LevelDB持久化数据格式</center>

如上图，SSTable 文件整体分为 4 个部分：

  - **Data Block（数据区域）**：保存具体的键-值对数据。
  - **Meta Block（元数据区域）**：保存元数据，例如布隆过滤器。
  - **Index Block（索引区域）**：分为数据索引和元数据索引。
    - **数据索引**：数据索引块中的键为前一个数据块的最后一个键（即一个数据块中最大的键，因为键是有序排列保存的）与后一个数据块的第一个键（即一个数据块中的最小键）的最短分隔符。
    - **元数据索引**：元数据索引块可指示如何查找该布隆过滤器的数据。
  - **File Footer（尾部）**：总大小为48个字节。

  

  ### BlockHandle

BlockHandle在SSTable中是经常使用的一个结构，其定义如下：

  ```c++
// https://github.com/google/leveldb/blob/master/table/format.h

class BlockHandle {
    public:
    // Maximum encoding length of a BlockHandle
    enum { kMaxEncodedLength = 10 + 10 };

    BlockHandle();

    // The offset of the block in the file.
    uint64_t offset() const { return offset_; }
    void set_offset(uint64_t offset) { offset_ = offset; }

    // The size of the stored block
    uint64_t size() const { return size_; }
    void set_size(uint64_t size) { size_ = size; }

    void EncodeTo(std::string* dst) const;
    Status DecodeFrom(Slice* input);

    private:
    uint64_t offset_;
    uint64_t size_;
};
  ```

BlockHandler 本质就是封装了 offset 和 size，用于定位某些区域。

  

  ### Block Format

SSTable 中一个块默认大小为 4 KB，由 4 部分组成：

  - **键-值对数据**：即我们保存到 LevelDB 中的多组键-值对。
  - **重启点数据**：最后 4 字节为重启点的个数，前边部分为多个重启点，每个重启点实际保存的是偏移量。并且每个重启点固定占据 4 字节的空间。
  - **压缩类型**：在 LevelDB 的 SSTable 中有两种压缩类型：
    - kNoCompression：没有压缩。
    - kSnappyCompression：Snappy压缩。
  - **校验数据**：4 字节的 CRC 校验字段。

  

  ## Block读写流程

  ### 生成Block

块生成主要在 BlockBuilder 中实现，下面先看看它的结构：

  ```c++
// https://github.com/google/leveldb/blob/master/table/block_builder.h

class BlockBuilder {
    public:
    explicit BlockBuilder(const Options* options);

    BlockBuilder(const BlockBuilder&) = delete;
    BlockBuilder& operator=(const BlockBuilder&) = delete;

    void Reset();

    void Add(const Slice& key, const Slice& value);

    Slice Finish();

    size_t CurrentSizeEstimate() const;

    bool empty() const { return buffer_.empty(); }

    private:
    const Options* options_;
    std::string buffer_;              // Destination buffer
    std::vector<uint32_t> restarts_;  // Restart points
    int counter_;                     // Number of entries emitted since restart
    bool finished_;                   // Has Finish() been called?
    std::string last_key_;
};
  ```

  - 成员变量
    - **options_**：在 BlockBuilder 类构造函数中传入，表示一些配置选项。
    - **buffer_**：块的内容，所有的键-值对都保存到 buffer_ 中。
    - **restarts_**：每次开启新的重启点后，会将当前 buffer_ 的数据长度保存到 restarts_ 中，当前 buffer_ 中的数据长度即为每个重启点的偏移量。
    - **counter_**：开启新的重启点之后加入的键-值对数量，默认保存 16 个键-值对，之后会开启一个新的重启点。
    - **finished_**：指明是否已经调用了 `Finish` 方法，BlockBuilder 中的 `Add` 方法会将数据保存到各个成员变量中，而 `Finish` 方法会依据成员变量的值生成一个块。
    - **last_key_**：上一个保存的键，当加入新键时，用来计算和上一个键的共同前缀部分。

  

介绍完了结构，下面来看具体的生成方法。当需要保存一个键-值对时，需要调用 BlockBuilder 类中的 `Add` 方法：

  ```c++
// https://github.com/google/leveldb/blob/master/table/block_builder.cc

void BlockBuilder::Add(const Slice& key, const Slice& value) {
    //保存上一个加入的key
    Slice last_key_piece(last_key_);

    assert(!finished_);
    assert(counter_ <= options_->block_restart_interval);
    assert(buffer_.empty()  // No values yet?
           || options_->comparator->Compare(key, last_key_piece) > 0);
    size_t shared = 0;

    //判断counter_是否大于block_restart_interval
    if (counter_ < options_->block_restart_interval) {
        const size_t min_length = std::min(last_key_piece.size(), key.size());
        //计算相同前缀的长度
        while ((shared < min_length) && (last_key_piece[shared] == key[shared])) {
            shared++;
        }
    } else {
        //如果键-值对数量超过block_restart_interval，则开启新的重启点，清空计数器
        restarts_.push_back(buffer_.size());
        counter_ = 0;
    }
    const size_t non_shared = key.size() - shared;

    //将共同前缀长度、非共享部分长度、值长度追加到buffer_中
    PutVarint32(&buffer_, shared);
    PutVarint32(&buffer_, non_shared);
    PutVarint32(&buffer_, value.size());

    //将key的非共享数据追加到buffer_中_
    buffer_.append(key.data() + shared, non_shared);
    //将Value数据追加到buffer_中
    buffer_.append(value.data(), value.size());

    //更新状态
    last_key_.resize(shared);
    last_key_.append(key.data() + shared, non_shared);
    assert(Slice(last_key_) == key);
    counter_++;
}
  ```

执行流程如下：

    1. 判断 counter_ 是否大于 block_restart_interval，如果大于，则开启新的重启点，清空计数器并保存 buffer_ 中数据长度的值（该值即每个重启点的偏移量）压到 restarts_ 数组中。
    2. 如果 counter_ 未超出配置的每个重启点可以保存的键-值对数值，则计算当前键和上一次保存键的共同前缀，然后将键-值对按格式保存到 buffer_ 中。
    3. 更新状态，将 last_key_ 置为当前保存的 key，并且将 counter_ 加 1。

  

从上面的代码中可以看出，`Add` 中将所有的键-值对按格式保存到成员变量 buffer_ 中。实际生成 Block 的其实是 `Finish` 。代码如下：

  ```c++
// https://github.com/google/leveldb/blob/master/table/block_builder.cc

Slice BlockBuilder::Finish() {
    //将重启点偏移量写入buffer_中
    for (size_t i = 0; i < restarts_.size(); i++) {
        PutFixed32(&buffer_, restarts_[i]);
    }
    PutFixed32(&buffer_, restarts_.size());
    finished_ = true;
    return Slice(buffer_);
}
  ```

`Finish ` 首先将所有重启点偏移量的值依次以 4 字节大小追加到 buffer_ 字符串，最后将重启点个数继续以 4 字节大小追加到 buffer_  后部，此时返回的结果就是一个完整的 Block。

  

  ### 读取Block

读取 Block 由 Block 类实现，其主要依靠 `NewIterator` 生成一个 Block 迭代器，再借助迭代器的 `Seek` 来查找对应的 Key。首先来看看 Block 的结构：

  ```c++
// https://github.com/google/leveldb/blob/master/table/block.h

class Block {
    public:
    // Initialize the block with the specified contents.
    explicit Block(const BlockContents& contents);

    Block(const Block&) = delete;
    Block& operator=(const Block&) = delete;

    ~Block();

    size_t size() const { return size_; }
    Iterator* NewIterator(const Comparator* comparator);

    private:
    class Iter;

    uint32_t NumRestarts() const;

    const char* data_;
    size_t size_;
    uint32_t restart_offset_;  // Offset in data_ of restart array
    bool owned_;               // Block owns data_[]
};
  ```

  

读取一个块通过在 `NewIterator` 中生成一个迭代器来实现，代码如下：

  ```c++
// https://github.com/google/leveldb/blob/master/table/block.cc

Iterator* Block::NewIterator(const Comparator* comparator) {
    if (size_ < sizeof(uint32_t)) {
        return NewErrorIterator(Status::Corruption("bad block contents"));
    }
    const uint32_t num_restarts = NumRestarts();
    if (num_restarts == 0) {
        return NewEmptyIterator();
    } else {
        return new Iter(comparator, data_, restart_offset_, num_restarts);
    }
}
  ```

这里的逻辑比较简单，就不多作介绍了。

核心的查找逻辑主要是迭代器中的 `Seek` 下面直接看代码：

  ```c++
// https://github.com/google/leveldb/blob/master/table/block.cc

void Seek(const Slice& target) override {
    uint32_t left = 0;
    uint32_t right = num_restarts_ - 1;
    int current_key_compare = 0;

    if (Valid()) {
        current_key_compare = Compare(key_, target);
        if (current_key_compare < 0) {
            left = restart_index_;
        } else if (current_key_compare > 0) {
            right = restart_index_;
        } else {
            return;
        }
    }

    //通过重启点进行二分查找
    while (left < right) {
        uint32_t mid = (left + right + 1) / 2;
        uint32_t region_offset = GetRestartPoint(mid);
        uint32_t shared, non_shared, value_length;
        const char* key_ptr =
            DecodeEntry(data_ + region_offset, data_ + restarts_, &shared,
                        &non_shared, &value_length);
        if (key_ptr == nullptr || (shared != 0)) {
            CorruptionError();
            return;
        }
        Slice mid_key(key_ptr, non_shared);
        //如果key小于target，则将left置为mid
        if (Compare(mid_key, target) < 0) {
            left = mid;

            //如果key大于等于target，则将right置为mid-1
        } else {
            right = mid - 1;
        }
    }

    assert(current_key_compare == 0 || Valid());
    //在块中线性查找，依次遍历每一个K-V对，将key与target对比，直到找到第一个大于等于target的后返
    bool skip_seek = left == restart_index_ && current_key_compare < 0;
    if (!skip_seek) {
        SeekToRestartPoint(left);
    }

    while (true) {
        if (!ParseNextKey()) {
            return;
        }
        if (Compare(key_, target) >= 0) {
            return;
        }
    }
}
  ```

查找逻辑如下：

    1. 对重启点数组进行二分查找，找到可能包含数据的重启点。
    2. 在块中线性查找，依次遍历每一个 K-V 对，将 key 与 target 对比。
    3. 找到第一个 key 大于等于 target 的后将该 K-V 对存储后返回。

  

  ## SSTable读写

  ### 生成SSTable

SSTable 的生成主要在 TableBuilder 中实现，下面先看看它的结构：

  ```c++
// https://github.com/google/leveldb/blob/master/include/leveldb/table_builder.h

class LEVELDB_EXPORT TableBuilder {
    public:
    TableBuilder(const Options& options, WritableFile* file);

    TableBuilder(const TableBuilder&) = delete;
    TableBuilder& operator=(const TableBuilder&) = delete;

    ~TableBuilder();

    Status ChangeOptions(const Options& options);

    void Add(const Slice& key, const Slice& value);

    void Flush();

    Status status() const;

    Status Finish();

    void Abandon();

    uint64_t NumEntries() const;

    uint64_t FileSize() const;

    private:
    bool ok() const { return status().ok(); }
    void WriteBlock(BlockBuilder* block, BlockHandle* handle);
    void WriteRawBlock(const Slice& data, CompressionType, BlockHandle* handle);

    struct Rep;
    Rep* rep_;
};
  ```

在介绍该 TableBuilder 的核心逻辑之前，首先我们要看看里面的一个结构体 Rep。其结构如下：

  ```c++
struct TableBuilder::Rep {
    Rep(const Options& opt, WritableFile* f)
        : options(opt),
    index_block_options(opt),
    file(f),
    offset(0),
    data_block(&options),
    index_block(&index_block_options),
    num_entries(0),
    closed(false),
    filter_block(opt.filter_policy == nullptr
                 ? nullptr
                 : new FilterBlockBuilder(opt.filter_policy)),
    pending_index_entry(false) {
        index_block_options.block_restart_interval = 1;
    }

    Options options;
    Options index_block_options;
    WritableFile* file;
    uint64_t offset;
    Status status;
    BlockBuilder data_block;
    BlockBuilder index_block;
    std::string last_key;
    int64_t num_entries;
    bool closed;  // Either Finish() or Abandon() has been called.
    FilterBlockBuilder* filter_block;
    bool pending_index_entry;
    BlockHandle pending_handle;  // Handle to add to index block

    std::string compressed_output;
};
  ```

我们需要注意的关键变量如下：

  - **file**：SSTable 生成的文件。
  - **data_block**：用于生成 SSTable 的数据区域。
  - **index_block**：用于生成 SSTable 的索引区域。
  - **pending_index_entry**：决定是否需要写数据索引。
  - **pending_handle：**写数据索引的方法。SSTable中每次完整写入一个块后需要生成该块的索引，索引中的键是当前块最大键与即将插入的键的最短分隔符，例如一个块中最大键为 abceg，即将插入的键为 abcqddh，则二者之间的最小分隔符为 abcf。

  

了解完结构后，接着就看看生成 SSTable 的核心函数 `Add` 和 `Finish`。

首先来看 `Add`，其代码如下：

  ```c++
// https://github.com/google/leveldb/blob/master/table/table_builder.cc

void TableBuilder::Add(const Slice& key, const Slice& value) {
    Rep* r = rep_;
    assert(!r->closed);
    if (!ok()) return;
    if (r->num_entries > 0) {
        assert(r->options.comparator->Compare(key, Slice(r->last_key)) > 0);
    }

    //判断是否需要写入数据索引块中
    if (r->pending_index_entry) {
        assert(r->data_block.empty());

        //找到最短分隔符，即大于等于上一个块最大的键，小于下一个块最小的键
        r->options.comparator->FindShortestSeparator(&r->last_key, key);
        std::string handle_encoding;
        r->pending_handle.EncodeTo(&handle_encoding);
        //在数据索引块中写入key和BlockHandle
        r->index_block.Add(r->last_key, Slice(handle_encoding));
        r->pending_index_entry = false;
    }

    //写入元数据块中
    if (r->filter_block != nullptr) {
        r->filter_block->AddKey(key);
    }

    //修改last_key为当前要插入的key
    r->last_key.assign(key.data(), key.size());
    r->num_entries++;

    //写入数据块中
    r->data_block.Add(key, value);

    //判断数据块大小是否大于配置的块大小，如果大于则调用Flush写入SSTable文件并刷新到硬盘
    const size_t estimated_block_size = r->data_block.CurrentSizeEstimate();
    if (estimated_block_size >= r->options.block_size) {
        Flush();
    }
}
  ```

`Add` 主要就是调用生成数据块与数据索引块的方法 `BlockBuilder::Add` 以及生成元数据块的方法 `FilterBlockBuilder::Add` 依次将键值对加入数据索引块、元数据块以及数据块。

可以看到，最后会判断数据块大小是否大于配置的块大小，如果大于则调用 Flush 写入 SSTable 文件并刷新到硬盘中。我们接着来看看 `Flush` 的执行逻辑：

  ```c++
// https://github.com/google/leveldb/blob/master/table/table_builder.cc

void TableBuilder::Flush() {
    Rep* r = rep_;
    assert(!r->closed);
    if (!ok()) return;
    if (r->data_block.empty()) return;
    assert(!r->pending_index_entry);

    //写入数据块
    WriteBlock(&r->data_block, &r->pending_handle);
    if (ok()) {
        //将pending_index_entry修改为true，表明下一次将写入数据索引块。
        r->pending_index_entry = true;
        //将文件刷新到磁盘	
        r->status = r->file->Flush();
    }
    if (r->filter_block != nullptr) {
        r->filter_block->StartBlock(r->offset);
    }
}

  ```

执行逻辑如下：

    1. 将数据写入数据块中。
    2. 将 pending_index_entry 修改为 true，表明下一次调用 `Add` 时将写入数据索引块。
    3. 将文件刷新到磁盘中。

介绍完了 `Add`，下面来看看 `Finish`。

  ```c++
// https://github.com/google/leveldb/blob/master/table/table_builder.cc

Status TableBuilder::Finish() {
    Rep* r = rep_;
    //写入SSTable文件并刷新到硬盘中
    Flush();
    assert(!r->closed);
    r->closed = true;

    BlockHandle filter_block_handle, metaindex_block_handle, index_block_handle;

    //写入元数据块
    if (ok() && r->filter_block != nullptr) {
        WriteRawBlock(r->filter_block->Finish(), kNoCompression,
                      &filter_block_handle);
    }

    //写入元数据块索引
    if (ok()) {
        BlockBuilder meta_index_block(&r->options);
        if (r->filter_block != nullptr) {
            std::string key = "filter.";
            key.append(r->options.filter_policy->Name());
            std::string handle_encoding;
            filter_block_handle.EncodeTo(&handle_encoding);
            meta_index_block.Add(key, handle_encoding);
        }

        WriteBlock(&meta_index_block, &metaindex_block_handle);
    }

    //写入数据块索引
    if (ok()) {
        if (r->pending_index_entry) {
            r->options.comparator->FindShortSuccessor(&r->last_key);
            std::string handle_encoding;
            r->pending_handle.EncodeTo(&handle_encoding);
            r->index_block.Add(r->last_key, Slice(handle_encoding));
            r->pending_index_entry = false;
        }
        WriteBlock(&r->index_block, &index_block_handle);
    }

    //写入尾部
    if (ok()) {
        Footer footer;
        footer.set_metaindex_handle(metaindex_block_handle);
        footer.set_index_handle(index_block_handle);
        std::string footer_encoding;
        footer.EncodeTo(&footer_encoding);
        r->status = r->file->Append(footer_encoding);
        if (r->status.ok()) {
            r->offset += footer_encoding.size();
        }
    }
    return r->status;
}
  ```

`Finish` 会按照我们一开始给出的 SSTable 的格式，将数据分别写入数据块、数据块索引、元数据块、元数据块索引、尾部。最后生成 SSTable。

  

  ### 读取SSTable

介绍完了写入，我们再来看看读取。SSTable 读取的逻辑与 Block 类似，都是借助于迭代器实现的。主要实现代码在 Table 类中。

  ```c++
// https://github.com/google/leveldb/blob/master/include/leveldb/table.h

class LEVELDB_EXPORT Table {
    public:
    static Status Open(const Options& options, RandomAccessFile* file,
                       uint64_t file_size, Table** table);

    Table(const Table&) = delete;
    Table& operator=(const Table&) = delete;

    ~Table();

    Iterator* NewIterator(const ReadOptions&) const;

    uint64_t ApproximateOffsetOf(const Slice& key) const;

    private:
    friend class TableCache;
    struct Rep;

    static Iterator* BlockReader(void*, const ReadOptions&, const Slice&);

    explicit Table(Rep* rep) : rep_(rep) {}

    Status InternalGet(const ReadOptions&, const Slice& key, void* arg,
                       void (*handle_result)(void* arg, const Slice& k,
                                             const Slice& v));

    void ReadMeta(const Footer& footer);
    void ReadFilter(const Slice& filter_handle_value);

    Rep* const rep_;
};
  ```

这里我们主要关注生成迭代器的 `NewIterator` 和 第二层生成迭代器的 `BlockReader`。

首先看 `NewIterator` 的代码：

  ```c++
// https://github.com/google/leveldb/blob/master/table/table.cc

Iterator* Table::NewIterator(const ReadOptions& options) const {
    return NewTwoLevelIterator(
        rep_->index_block->NewIterator(rep_->options.comparator),
        &Table::BlockReader, const_cast<Table*>(this), options);
}
  ```

这里返回了一个双层迭代器 `NewTwoLevelIterator`。第一层为数据索引块的迭代器，即 `rep_->index_block->NewIterator`。通过第一层的数据索引块迭代器查找一个键应该属于的块，然后通过第二层迭代器去读取这个块并查找该键。

接着看生成第二层块迭代器的 `BlockReader`：

  ```c++
// https://github.com/google/leveldb/blob/master/table/table.cc

Iterator* Table::BlockReader(void* arg, const ReadOptions& options,
                             const Slice& index_value) {
    Table* table = reinterpret_cast<Table*>(arg);
    Cache* block_cache = table->rep_->options.block_cache;
    Block* block = nullptr;
    Cache::Handle* cache_handle = nullptr;

    BlockHandle handle;
    Slice input = index_value;
    Status s = handle.DecodeFrom(&input);

    if (s.ok()) {
        //contents保存一个块的内容
        BlockContents contents;
        if (block_cache != nullptr) {
            char cache_key_buffer[16];
            EncodeFixed64(cache_key_buffer, table->rep_->cache_id);
            EncodeFixed64(cache_key_buffer + 8, handle.offset());
            Slice key(cache_key_buffer, sizeof(cache_key_buffer));
            cache_handle = block_cache->Lookup(key);

            if (cache_handle != nullptr) {
                block = reinterpret_cast<Block*>(block_cache->Value(cache_handle));
            } else {
                //通过BlockHandle存储的偏移量和大小读取一个块的数据到contents
                s = ReadBlock(table->rep_->file, options, handle, &contents);
                if (s.ok()) {
                    //根据contents生成一个Block结构
                    block = new Block(contents);
                    if (contents.cachable && options.fill_cache) {
                        cache_handle = block_cache->Insert(key, block, block->size(),
                                                           &DeleteCachedBlock);
                    }
                }
            }
        } else {
            s = ReadBlock(table->rep_->file, options, handle, &contents);
            if (s.ok()) {
                block = new Block(contents);
            }
        }
    }

    Iterator* iter;
    if (block != nullptr) {
        //生成块迭代器，通过该迭代器读取数据
        iter = block->NewIterator(table->rep_->options.comparator);
        if (cache_handle == nullptr) {
            iter->RegisterCleanup(&DeleteBlock, block, nullptr);
        } else {
            iter->RegisterCleanup(&ReleaseBlock, block_cache, cache_handle);
        }
    } else {
        iter = NewErrorIterator(s);
    }
    return iter;
}
  ```

SSTable 的读取先通过第一层迭代器（即数据索引）获取到一个键需要查找的块位置，读取该块的内容并且构造第二层迭代器遍历该块，通过两层迭代器即可在 SSTable 中进行键的查找。

  

  ## 布隆过滤器

如果在 LevelDB 中查找某个不存在的键，必须先检查内存表 MemTable，然后逐层查找，为了优化这种读取，LevelDB 中会使用布隆过滤器。当布隆过滤器判定键不存在时，可以直接返回，无须继续查找。

这里就不过多介绍原理，如果想了解可以看看我的往期博客 [海量数据处理（一） ：位图与布隆过滤器的概念以及实现](https://blog.csdn.net/qq_35423154/article/details/107363058)

  ### 实现

布隆过滤器继承了 LevelDB 中抽象出的过滤器纯虚类 FilterPolicy，其结构如下：

  ```c++
// https://github.com/google/leveldb/blob/master/include/leveldb/filter_policy.h

class LEVELDB_EXPORT FilterPolicy {
    public:
    virtual ~FilterPolicy();

    virtual const char* Name() const = 0;

    virtual void CreateFilter(const Slice* keys, int n,
                              std::string* dst) const = 0;

    virtual bool KeyMayMatch(const Slice& key, const Slice& filter) const = 0;
};
  ```

FilterPolicy 定义了几个接口：

  - **Name**：返回过滤器名称。
  - **CreateFilter**：将一个字符串加入过滤器中。
  - **KeyMayMatch**：通过过滤器内容判断一个元素是否存在。

接下来我们看看 BloomFilterPolicy 是如何实现这些接口的，代码如下：

  ```c++
// https://github.com/google/leveldb/blob/master/util/bloom.cc

class BloomFilterPolicy : public FilterPolicy {
    public:
    explicit BloomFilterPolicy(int bits_per_key) : bits_per_key_(bits_per_key) {
        k_ = static_cast<size_t>(bits_per_key * 0.69);  // 0.69 =~ ln(2)
        if (k_ < 1) k_ = 1;
        if (k_ > 30) k_ = 30;
    }

    const char* Name() const override { return "leveldb.BuiltinBloomFilter2"; }

    void CreateFilter(const Slice* keys, int n, std::string* dst) const override {
        size_t bits = n * bits_per_key_;

        if (bits < 64) bits = 64;

        size_t bytes = (bits + 7) / 8;
        bits = bytes * 8;

        const size_t init_size = dst->size();
        dst->resize(init_size + bytes, 0);
        dst->push_back(static_cast<char>(k_));  // Remember # of probes in filter
        char* array = &(*dst)[init_size];
        //依次处理每一个Key
        for (int i = 0; i < n; i++) {
            uint32_t h = BloomHash(keys[i]);
            //通过位运算获取delta
            const uint32_t delta = (h >> 17) | (h << 15);  // Rotate right 17 bits

            //计算哈希值，对每一个key计算k次哈希值（这里采用对每次计算出的的哈希值增加delta来模拟）
            for (size_t j = 0; j < k_; j++) {
                const uint32_t bitpos = h % bits;
                array[bitpos / 8] |= (1 << (bitpos % 8));
                h += delta;
            }
        }
    }

    bool KeyMayMatch(const Slice& key, const Slice& bloom_filter) const override {
        const size_t len = bloom_filter.size();
        if (len < 2) return false;

        const char* array = bloom_filter.data();
        const size_t bits = (len - 1) * 8;
        const size_t k = array[len - 1];
        if (k > 30) {
            return true;
        }

        //计算出key的哈希
        uint32_t h = BloomHash(key);
        const uint32_t delta = (h >> 17) | (h << 15);  // Rotate right 17 bits
        //判断对应位置是否全为1，如果有任何一个为0说明数据不可能存在布隆过滤器中，返回false，否则true
        for (size_t j = 0; j < k; j++) {
            const uint32_t bitpos = h % bits;
            if ((array[bitpos / 8] & (1 << (bitpos % 8))) == 0) return false;
            h += delta;
        }
        return true;
    }

    private:
    size_t bits_per_key_;
    size_t k_;
};
  ```

具体逻辑都标识在了注释中，就不多说了，这里提一下这里用到的一个哈希小技巧：

  - 为了避免哈希冲突，大部分布隆过滤器中都会采用多种哈希算法来计算。LevelDB 为了简化规则，使用位运算 计算出 delta，对每轮计算出的哈希值累加上 delta，模拟多轮哈希计算。

  

  ### 应用

LevelDB 中具体使用布隆过滤器时又封装了两个类，分别为 FilterBlockBuilder 和 FilterBlockReader。

FilterBlockBuilder 的主要功能是通过调用 BloomFilterPolicy 的 `CreateFilter` 方法生成布隆过滤器，并且将布隆过滤器的内容写入 SSTable 的元数据块。其结构如下：

  ```c++
// https://github.com/google/leveldb/blob/master/table/filter_block.h

class FilterBlockBuilder {
    public:
    explicit FilterBlockBuilder(const FilterPolicy*);

    FilterBlockBuilder(const FilterBlockBuilder&) = delete;
    FilterBlockBuilder& operator=(const FilterBlockBuilder&) = delete;

    void StartBlock(uint64_t block_offset);
    void AddKey(const Slice& key);
    Slice Finish();

    private:
    void GenerateFilter();

    const FilterPolicy* policy_;
    std::string keys_;             // Flattened key contents
    std::vector<size_t> start_;    // Starting index in keys_ of each key
    std::string result_;           // Filter data computed so far
    std::vector<Slice> tmp_keys_;  // policy_->CreateFilter() argument
    std::vector<uint32_t> filter_offsets_;
};
  ```

  - 成员变量
    - **policy_**：实现了 FilterPolicy 接口的类，在布隆过滤器中为 BloomFilterPolicy。
    - **keys_**：生成布隆过滤器的键。
    - **start_**：数组类型，保存 keys_ 参数中每一个键的开始索引。
    - **result_**：保存生成的布隆过滤器内容。
    - **tmp_keys_**：生成布隆过滤器时，会通过 keys_ 和 start_ 拆分出每一个键，将拆分出的每一个键保存到 tmp_keys_ 数组中。
    - **filter_offsets_**：过滤器偏移量，即每一个过滤器在元数据块中的偏移量。

写入的核心逻辑主要在 `AddKey` 和 `Finish` 中，代码如下：

  ```c++
// https://github.com/google/leveldb/blob/master/table/filter_block.cc

void FilterBlockBuilder::AddKey(const Slice& key) {
    Slice k = key;
    //记录key的索引位置
    start_.push_back(keys_.size());
    //将key追加到该字符串中
    keys_.append(k.data(), k.size());
}

Slice FilterBlockBuilder::Finish() {
    //写入内容
    if (!start_.empty()) {
        GenerateFilter();
    }

    //写入偏移量
    const uint32_t array_offset = result_.size();
    for (size_t i = 0; i < filter_offsets_.size(); i++) {
        PutFixed32(&result_, filter_offsets_[i]);
    }
    //写入总大小
    PutFixed32(&result_, array_offset);
    //写入基数
    result_.push_back(kFilterBaseLg);  // Save encoding parameter in result
    return Slice(result_);
}
  ```

首先调用 `AddKey` 记录 Key 的索引位置，并将 Key 追加到 Keys_中。接着，调用 `Finish` 生成一个元数据块，并分别写入布隆过滤器的内容、偏移量、内容总大小和基数。

  

FilterBlockReader 用于查找一个元素是否在一个布隆过滤器中，其结构如下：

  ```c++
// https://github.com/google/leveldb/blob/master/table/filter_block.h

class FilterBlockReader {
    public:
    // REQUIRES: "contents" and *policy must stay live while *this is live.
    FilterBlockReader(const FilterPolicy* policy, const Slice& contents);
    bool KeyMayMatch(uint64_t block_offset, const Slice& key);

    private:
    const FilterPolicy* policy_;
    const char* data_;    // Pointer to filter data (at block-start)
    const char* offset_;  // Pointer to beginning of offset array (at block-end)
    size_t num_;          // Number of entries in offset array
    size_t base_lg_;      // Encoding parameter (see kFilterBaseLg in .cc file)
};
  ```

  - **成员变量**
    - **policy_**：实现了 FilterPolicy 接口的类，在布隆过滤器中为 BloomFilterPolicy。
    - **data_**：指向元数据块的开始位置。
    - **offset_**：指向元数据块中过滤器偏移量的开始位置。
    - **num_**：过滤器偏移量的个数。
    - **base_lg_**：过滤器基数。

FilterBlockReader 中的 `KeyMayMatch` 根据数据块偏移量找到对应的过滤器内容，然后调用 BloomFilterPolicy 中的 `KeyMayMatch` 方法判断一个元素是否在该过滤器之中。代码如下：

  ```c++
// https://github.com/google/leveldb/blob/master/table/filter_block.cc

bool FilterBlockReader::KeyMayMatch(uint64_t block_offset, const Slice& key) {
    uint64_t index = block_offset >> base_lg_;
    if (index < num_) {
        uint32_t start = DecodeFixed32(offset_ + index * 4);
        uint32_t limit = DecodeFixed32(offset_ + index * 4 + 4);
        if (start <= limit && limit <= static_cast<size_t>(offset_ - data_)) {
            Slice filter = Slice(data_ + start, limit - start);
            return policy_->KeyMayMatch(key, filter);
        } else if (start == limit) {
            // Empty filters do not match any keys
            return false;
        }
    }
  ```

  

  ## LRU Cache

我们希望经常使用的 SSTable 内容尽量保存在内存中，但如果磁盘中的 SSTable 文件的总大小大于服务器内存大小，或者需要控制 LevelDB 的内存总占用量时，就需要使用 LRU（least recentlyused）Cache 来管理内存。LRU 是一种缓存置换策略，根据该策略不仅可以管理内存的占用量，还可以将热数据尽量保存到内存中，以加快读取速度。

内存是有限并且昂贵的资源，因此 LevelDB 通过 LRU 策略管理读取到内存的数据。LRU 基于这样一种假设：如果一个资源最近没有或者很少被使用到，那么将来也会很少甚至不被使用。因此如果内存不足，需要淘汰数据时，可以根据 LRU 策略来执行。

这里就不过多介绍原理，如果想了解可以看看我的往期博客 [高级数据结构与算法 | LRU缓存机制（Least Recently Used）](https://blog.csdn.net/qq_35423154/article/details/107876954)

  ### 结构

为了方便拓展其他的缓存置换算法（目前仅实现了 LRU），LevelDB 抽象出了一个 Cache 纯虚类，结构如下：

  ```c++
// https://github.com/google/leveldb/blob/master/include/leveldb/cache.h

class LEVELDB_EXPORT Cache {
    public:
    Cache() = default;

    Cache(const Cache&) = delete;
    Cache& operator=(const Cache&) = delete;

    virtual ~Cache();

    struct Handle {};

    virtual Handle* Insert(const Slice& key, void* value, size_t charge,
                           void (*deleter)(const Slice& key, void* value)) = 0;

    virtual Handle* Lookup(const Slice& key) = 0;

    virtual void Release(Handle* handle) = 0;

    virtual void* Value(Handle* handle) = 0;

    virtual void Erase(const Slice& key) = 0;

    virtual uint64_t NewId() = 0;

    virtual void Prune() {}

    virtual size_t TotalCharge() const = 0;

    private:
    void LRU_Remove(Handle* e);
    void LRU_Append(Handle* e);
    void Unref(Handle* e);

    struct Rep;
    Rep* rep_;
};
  ```

  

LevelDB 中 LRU 由 LRUCache 实现，并且 LRU 节点由 LRUHandle 实现。首先我们来看看 LRUHandle：

  ```c++
// https://github.com/google/leveldb/blob/master/util/cache.cc

struct LRUHandle {
    void* value;
    void (*deleter)(const Slice&, void* value);
    LRUHandle* next_hash;
    LRUHandle* next;	
    LRUHandle* prev;
    size_t charge;  // TODO(opt): Only allow uint32_t?
    size_t key_length;
    bool in_cache;     // Whether entry is in the cache.
    uint32_t refs;     // References, including cache reference, if present.
    uint32_t hash;     // Hash of key(); used for fast sharding and comparisons
    char key_data[1];  // Beginning of key

    Slice key() const {
        assert(next != this);

        return Slice(key_data, key_length);
    }
};
  ```

  - 成员变量
    - **value**：具体的值，指针类型。
    - **deleter**：自定义回收节点的回调函数。
    - **next_hash**：用于 hashtable 冲突时，下一个节点。
    - **next**：代表 LRU 中双向链表中下一个节点。
    - **prev**：代表 LRU 中双向链表中上一个节点。
    - **charge**：记录当前 value 所占用的内存大小，用于后面超出容量后需要进行 lru。
    - **key_length**：数据 key 的长度。
    - **in_cache**：表示是否在缓存中。
    - **refs**：引用计数，因为当前节点可能会被多个组件使用，不能简单的删除。
    - **hash**：记录当前 key 的 hash 值。

这个节点设计的非常巧妙，既可以用来当做 LRU 节点，也可以用来当作 hashtable 的节点。

接着我们来看看 LRUCache 的结构：

  ```c++
// https://github.com/google/leveldb/blob/master/util/cache.cc

class LRUCache {
    public:
    LRUCache();
    ~LRUCache();

    void SetCapacity(size_t capacity) { capacity_ = capacity; }

    Cache::Handle* Insert(const Slice& key, uint32_t hash, void* value,
                          size_t charge,
                          void (*deleter)(const Slice& key, void* value));
    Cache::Handle* Lookup(const Slice& key, uint32_t hash);
    void Release(Cache::Handle* handle);
    void Erase(const Slice& key, uint32_t hash);
    void Prune();
    size_t TotalCharge() const {
        MutexLock l(&mutex_);
        return usage_;
    }

    private:
    void LRU_Remove(LRUHandle* e);
    void LRU_Append(LRUHandle* list, LRUHandle* e);
    void Ref(LRUHandle* e);
    void Unref(LRUHandle* e);
    bool FinishErase(LRUHandle* e) EXCLUSIVE_LOCKS_REQUIRED(mutex_);

    size_t capacity_;

    mutable port::Mutex mutex_;
    size_t usage_ GUARDED_BY(mutex_);

    LRUHandle lru_ GUARDED_BY(mutex_);

    LRUHandle in_use_ GUARDED_BY(mutex_);

    HandleTable table_ GUARDED_BY(mutex_);
};
  ```

  

  ### 实现

LRU Cache 的实现有如下两处细节需要注意：

  - **减小锁的粒度**：LRUCache 的并发操作不安全，因此操作时需要加锁。为了减小锁的粒度，LevelDB  中通过哈希将键分为 16 个段，可以理解为有 16 个相同的 LRU Cache 结构，每次进行 Cache 操作时需要先去查找键属于的段。
  - **缓存淘汰**：每个LRU Cache结构中有两个成员变量：lru_ 和 in_use_ 。lru_ 双向链表中的节点是可以进行淘汰的，而 in_use_ 双向链表中的节点表示正在使用，因此不可以进行淘汰。

  #### 减小锁的粒度

LevelDB 为了减小锁的粒度，封装了一个 ShardedLRUCache，其中有一个大小为 16（默认值） 的 shard_ 成员，shard_ 成员的每个元素均为一个 LRUCache 结构。

  ```c++
// ShardedLRUCache
class ShardedLRUCache : public Cache {
    private:
    LRUCache shard_[kNumShards];
    port::Mutex id_mutex_;
    uint64_t last_id_;

    static inline uint32_t HashSlice(const Slice& s) {
        return Hash(s.data(), s.size(), 0);
    }

    static uint32_t Shard(uint32_t hash) { return hash >> (32 - kNumShardBits); }

    public:
    explicit ShardedLRUCache(size_t capacity) : last_id_(0) {
        const size_t per_shard = (capacity + (kNumShards - 1)) / kNumShards;
        for (int s = 0; s < kNumShards; s++) {
            shard_[s].SetCapacity(per_shard);
        }
    }
    ~ShardedLRUCache() override {}
    Handle* Insert(const Slice& key, void* value, size_t charge,
                   void (*deleter)(const Slice& key, void* value)) override {
        const uint32_t hash = HashSlice(key);
        return shard_[Shard(hash)].Insert(key, hash, value, charge, deleter);
    }
    Handle* Lookup(const Slice& key) override {
        const uint32_t hash = HashSlice(key);
        return shard_[Shard(hash)].Lookup(key, hash);
    }
    void Release(Handle* handle) override {
        LRUHandle* h = reinterpret_cast<LRUHandle*>(handle);
        shard_[Shard(h->hash)].Release(handle);
    }
    void Erase(const Slice& key) override {
        const uint32_t hash = HashSlice(key);
        shard_[Shard(hash)].Erase(key, hash);
    }
    void* Value(Handle* handle) override {
        return reinterpret_cast<LRUHandle*>(handle)->value;
    }
    uint64_t NewId() override {
        MutexLock l(&id_mutex_);
        return ++(last_id_);
    }
    void Prune() override {
        for (int s = 0; s < kNumShards; s++) {
            shard_[s].Prune();
        }
    }
    size_t TotalCharge() const override {
        size_t total = 0;
        for (int s = 0; s < kNumShards; s++) {
            total += shard_[s].TotalCharge();
        }
        return total;
    }
};
  ```

为了能使哈希值刚好能够映射 shard_ 数组，其通过位操作再次对哈希值进行处理，如下代码：

  ```c++
// https://github.com/google/leveldb/blob/master/util/cache.cc

static uint32_t Shard(uint32_t hash) { return hash >> (32 - kNumShardBits); }
  ```

其将哈希值右移 28 位，只取最高的 4 位，这样就能够保证处理过的哈希值刚好小于 16。

  

  #### 缓存淘汰

每个LRU Cache结构中有两个成员变量：lru_ 和 in_use_ 。lru_ 双向链表中的节点是可以进行淘汰的，而 in_use_ 双向链表中的节点表示正在使用，因此不可以进行淘汰。如果内存超出限制需要淘汰一个节点时，LevelDB 会将 lru_ 链表中的节点逐个淘汰。

那我们如何决定节点放置的规则呢？其采用如下规则：**如果一个缓存节点的 in_cache 为 true，并且 refs 等于 1，则放置到 lru_ 中；如果 in_cache 为 true，并且 refs 大于等于 2，则放置到 in_use_ 中。**

  - 对于 in_cache，其会在以下情况变为 false：
    - 删除该节点后。
    - 调用 LRUCache 的析构函数时，会将所有节点的 in_cache 置为 false。
    - 插入一个节点时，如果已经存在一个键值相同的节点，则旧节点的 in_cache 会置为 false。
  - 对于 refs，其变化规则如下:
    - 每次调用 `Ref` 函数，会将 refs 变量加 1，调用 `Unref` 函数，会将 refs 变量减 1。
    - 插入一个节点时，该节点会放到 in_use_ 链表中，并且初始的引用计数为 2，不再使用该节点时将引用计数减 1，如果此时节点也不再被其他地方引用，那么引用计数为 1，将其放到 lru_ 链表中。
    - 查找一个节点时，如果查找成功，则调用 `Ref`，将该节点的引用计数加 1，如果引用计数大于等于 2，会将节点放到 in_use_ 链表中，同理，不再使用该节点时将引用计数减 1，如果此时节点也不再被其他地方引用，那么引用计数为 1，将其放到 lru_ 链表中。

  

`Ref` 和 `Unref` 代码如下：

  ```c++
// https://github.com/google/leveldb/blob/master/util/cache.cc

void LRUCache::Ref(LRUHandle* e) {
  if (e->refs == 1 && e->in_cache) {  // If on lru_ list, move to in_use_ list.
    LRU_Remove(e);
    LRU_Append(&in_use_, e);
  }
  e->refs++;
}
  ```

如果缓存节点在 lru_ 链表中（refs  为 1，in_cache 为 true），则首先从 lru_ 链表中删除该节点，然后将节点放到 in_use_ 链表中。最后将节点的 refs 变量加 1。

  

  ```c++
// https://github.com/google/leveldb/blob/master/util/cache.cc

void LRUCache::Unref(LRUHandle* e) {
    assert(e->refs > 0);
    e->refs--;
    if (e->refs == 0) {  // Deallocate.
        assert(!e->in_cache);
        (*e->deleter)(e->key(), e->value);
        free(e);
    } else if (e->in_cache && e->refs == 1) {
        // No longer in use; move to lru_ list.
        LRU_Remove(e);
        LRU_Append(&lru_, e);
    }
}
  ```

`Unref` 首先将节点的 refs 变量减 1，然后判断如果 refs 已经等于 0，则删除并释放该节点，否则，如果 refs 变量等于 1 并且 in_cache 为 true，则将该节点从 in_use_ 链表中删除并且移动到 lru_ 链表中。如果内存超出限制需要淘汰一个节点时，LevelDB 会将 lru_ 链表中的节点逐个淘汰。

  

  ### 应用

LevelDB 中 Cache 缓存的主要是 SSTable，即缓存节点的键为 8 字节的文件序号，值为一个包含了 SSTable 实例的结构。其主要逻辑由 TableCache 实现，首先我们先来看看它的结构：

  ```c++
// https://github.com/google/leveldb/blob/master/db/table_cache.h

class TableCache {
    public:
    TableCache(const std::string& dbname, const Options& options, int entries);
    ~TableCache();

    Iterator* NewIterator(const ReadOptions& options, uint64_t file_number,
                          uint64_t file_size, Table** tableptr = nullptr);

    Status Get(const ReadOptions& options, uint64_t file_number,
               uint64_t file_size, const Slice& k, void* arg,
               void (*handle_result)(void*, const Slice&, const Slice&));

    void Evict(uint64_t file_number);

    private:
    Status FindTable(uint64_t file_number, uint64_t file_size, Cache::Handle**);

    Env* const env_;
    const std::string dbname_;
    const Options& options_;
    Cache* cache_;
};
  ```

  - 成员变量

    - **env_**：用于读取 SSTable 文件。
    - **dbname_**：SSTable 名字。
    - **options_**：Cache 参数配置。
    - **cache_**：缓存基类句柄。

    

这里的核心逻辑 `FindTable` 方法会使用文件序号（file_number）作为键，并且在 cache_ 中查找是否存在该键，如果不存在，则需要打开一个 SSTable，经过处理之后作为值插入 cache_ 中。

  ```c++
Status TableCache::FindTable(uint64_t file_number, uint64_t file_size,
                             Cache::Handle** handle) {
    Status s;
    char buf[sizeof(file_number)];
    EncodeFixed64(buf, file_number);
    Slice key(buf, sizeof(buf));
    //在缓存中查找key是否存在
    *handle = cache_->Lookup(key);

    //如果不存在，则打开一个SSTable文件
    if (*handle == nullptr) {
        //生成fname和RandomAccessFile实例
        std::string fname = TableFileName(dbname_, file_number);
        RandomAccessFile* file = nullptr;
        Table* table = nullptr;
        s = env_->NewRandomAccessFile(fname, &file);
        if (!s.ok()) {
            std::string old_fname = SSTTableFileName(dbname_, file_number);
            if (env_->NewRandomAccessFile(old_fname, &file).ok()) {
                s = Status::OK();
            }
        }

        //打开SSTable文件并且生成一个Table实例，并保存到table变量中。
        if (s.ok()) {
            s = Table::Open(options_, file, file_size, &table);
        }

        if (!s.ok()) {
            assert(table == nullptr);
            delete file;
        } else {
            TableAndFile* tf = new TableAndFile;
            tf->file = file;
            tf->table = table;
            //以文件序号为key，TableAndFile为Value，插入缓存中。
            *handle = cache_->Insert(key, tf, 1, &DeleteEntry);
        }
    }
    return s;
}
  ```

SSTable 在 Cache 中缓存时的键为文件序列号，值为一个 TableAndFile 实例，该实例中包括两个成员变量，分别为 file 和 table，查找时通过保存在 table 变量中的 Table 实例迭代器进行查找。

  
