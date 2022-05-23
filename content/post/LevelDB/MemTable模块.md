---
title: LevelDB MemTable模块
date: 2022-05-23T23:40:13+08:00
categories:
    - LevelDB
tags:
    - LevelDB
    - KV存储
---

# MemTable模块

## MemTable

在 LevelDB 中，MemTable 是底层数据结构 SkipList 的封装。

### 结构

首先我们来看看它的结构。

```c++
// https://github.com/google/leveldb/blob/master/db/memtable.h

class MemTable {
    public:
    explicit MemTable(const InternalKeyComparator& comparator);

    MemTable(const MemTable&) = delete;
    MemTable& operator=(const MemTable&) = delete;

    void Ref() { ++refs_; }

    void Unref() {
        --refs_;
        assert(refs_ >= 0);
        if (refs_ <= 0) {
            delete this;
        }
    }

    size_t ApproximateMemoryUsage();

    Iterator* NewIterator();

    void Add(SequenceNumber seq, ValueType type, const Slice& key,
             const Slice& value);

    bool Get(const LookupKey& key, std::string* value, Status* s);

    private:
    friend class MemTableIterator;
    friend class MemTableBackwardIterator;

    struct KeyComparator {
        const InternalKeyComparator comparator;
        explicit KeyComparator(const InternalKeyComparator& c) : comparator(c) {}
        int operator()(const char* a, const char* b) const;
    };

    typedef SkipList<const char*, KeyComparator> Table;

    ~MemTable();  // Private since only Unref() should be used to delete it

    KeyComparator comparator_;
    int refs_;
    Arena arena_;
    Table table_;
};
```

其组成如下：

  - **成员变量**
    - **comparator_**：比较器，用于决定 key 的顺序。
    - **refs__**：引用计数器，当计数为 0 时释放资源。
    - **arena_**：内存池，负责管理内存。
    - **table_**：底层存储的 SkipList。
  - **成员函数**
    - **Ref** ：引用计数增加。
    - **Unref**：引用计数减少。
    - **ApproximateMemoryUsage**：统计内存使用量。
    - **NewIterator**：返回首部迭代器。
    - **Add**：插入数据。
    - **Get**：查找数据。

  接着我们来看看最为核心的插入与查找。

  

### 插入

```c++
// https://github.com/google/leveldb/blob/master/db/memtable.cc

void MemTable::Add(SequenceNumber s, ValueType type, const Slice& key,
                   const Slice& value) {
    //计算需要的内存大小
    size_t key_size = key.size();
    size_t val_size = value.size();
    size_t internal_key_size = key_size + 8;
    const size_t encoded_len = VarintLength(internal_key_size) +
        internal_key_size + VarintLength(val_size) +
        val_size;
    //分配内存，并按照 len key sequencelValueType value_len value顺序将数据写入缓冲区
    char* buf = arena_.Allocate(encoded_len);
    char* p = EncodeVarint32(buf, internal_key_size);
    std::memcpy(p, key.data(), key_size);
    p += key_size;
    EncodeFixed64(p, (s << 8) | type);
    p += 8;
    p = EncodeVarint32(p, val_size);
    std::memcpy(p, value.data(), val_size);
    assert(p + val_size == buf + encoded_len);

    //调用底层SkipList的Insert将数据插入
    table_.Insert(buf)
}
```

插入的逻辑主要分为三步：

  1. 计算需要的内存大小。
  2. 分配内存，并按照 len key sequencelValueType value_len value顺序将数据写入缓冲区。
  3. 调用底层SkipList的Insert将数据插入。

  

### 查找

```c++
// https://github.com/google/leveldb/blob/master/db/memtable.cc

bool MemTable::Get(const LookupKey& key, std::string* value, Status* s) {
    Slice memkey = key.memtable_key();
    //获取迭代器
    Table::Iterator iter(&table_);	
    //通过迭代器的Seek查找数据
    iter.Seek(memkey.data());
    if (iter.Valid()) {
        // entry format is:
        //    klength  varint32
        //    userkey  char[klength]
        //    tag      uint64
        //    vlength  varint32
        //    value    char[vlength]
        const char* entry = iter.key();
        uint32_t key_length;
        const char* key_ptr = GetVarint32Ptr(entry, entry + 5, &key_length);
        //判断查找是否成功
        if (comparator_.comparator.user_comparator()->Compare(
            Slice(key_ptr, key_length - 8), key.user_key()) == 0) {
            const uint64_t tag = DecodeFixed64(key_ptr + key_length - 8);
            //判断查找到的类型
            switch (static_cast<ValueType>(tag & 0xff)) {
                    //如果数据存在，则将数据保存后返回true
                case kTypeValue: {
                    Slice v = GetLengthPrefixedSlice(key_ptr + key_length);
                    value->assign(v.data(), v.size());
                    return true;
                }
                    //如果数据删除，则将状态标记为未找到并返回true
                case kTypeDeletion:
                    *s = Status::NotFound(Slice());
                    return true;
            }
        }
    }
    //没有找到则返回false
    return false;
}
```

查找流程如下：

  1. 获取迭代器，通过 `Seek` 查找数据。
  2. 判断查找结果：
     - 查找成功，数据存在，保存数据后返回 true。
     - 查找成功，数据已删除（曾经存在），标记状态为 NotFound 后返回 true。
     - 查找失败，数据不存在，返回 false。

了解完了 MemTable，接下来再看看它底层使用的 SkipList 是如何实现的。

  

## SkipList

SkipList 是一个多层有序链表结构，通过在每个节点中保存多个指向其他节点的指针，将有序链表平均的复杂度`O（N） `降低到 `O（logN）`。SkipList 因具有实现简单、性能优良等特点得到了广泛应用，例如 Redis 中的 ZSet，以及 LevelDB 的 MemTable。

具体信息可以看看我之前写的博客 [高级数据结构与算法 | 跳跃表（Skip List）](https://blog.csdn.net/qq_35423154/article/details/108477428)。这里就不多作介绍，直接看代码。

### 结构

```c++
// https://github.com/google/leveldb/blob/master/db/skiplist.h

template <typename Key, class Comparator>
  class SkipList {
   private:
    struct Node;

   public:
    explicit SkipList(Comparator cmp, Arena* arena);

    SkipList(const SkipList&) = delete;
    SkipList& operator=(const SkipList&) = delete;

    void Insert(const Key& key);

    bool Contains(const Key& key) const;

    class Iterator {
     public:
      explicit Iterator(const SkipList* list);

      bool Valid() const;

      const Key& key() const;

      void Next();

      void Prev();

      void Seek(const Key& target);

      void SeekToFirst();

      void SeekToLast();

     private:
      const SkipList* list_;
      Node* node_;
    };

   private:
    enum { kMaxHeight = 12 };

    inline int GetMaxHeight() const {
      return max_height_.load(std::memory_order_relaxed);
    }

    Node* NewNode(const Key& key, int height);
    int RandomHeight();
    bool Equal(const Key& a, const Key& b) const { return (compare_(a, b) == 0); }

    bool KeyIsAfterNode(const Key& key, Node* n) const;

    Node* FindGreaterOrEqual(const Key& key, Node** prev) const;

    Node* FindLessThan(const Key& key) const;

    Node* FindLast() const;

    Comparator const compare_;
    Arena* const arena_; 

    Node* const head_;

    std::atomic<int> max_height_;

    Random rnd_;
  };
```

这里也是只介绍几个重要的函数——晋升、插入、查找。

  

### 晋升

在 LevelDB 中，每次插入节点的层高由 `RandomHeight` 决定，比起 Redis 的 1/2 来说，LevelDB 中的晋升逻辑更加复杂，代码如下：

```c++
// https://github.com/google/leveldb/blob/master/db/skiplist.h

template <typename Key, class Comparator>
  int SkipList<Key, Comparator>::RandomHeight() {
    static const unsigned int kBranching = 4;
    int height = 1;
    while (height < kMaxHeight && ((rnd_.Next() % kBranching) == 0)) {
      height++;
    }
    assert(height > 0);
    assert(height <= kMaxHeight);
    return height;
  }
```

上述代码中 `rnd_.Next()` 的作用是生成一个随机数，将该随机数对 4 取余，如果余数等于 0 并且层高小于规定的最大层高 12，则将层高加 1。因为对 4 取余数结果只有 0、1、2、3 这 4 种可能，因此可以推导得出每个节点层高为 1 的概率是 3/4，层高为 2 的概率是 1/4。依此类推，层高为 3 的概率是 3/16（ 1/4 × 3/4 ），层高为4的概率是3/64（ 1/4 × 1/4 × 3/4 ），即**层级越高，概率越小**。

  

### 查找

SkipList 的查找主要是借助迭代器的 `Seek` 来实现的，而在 `Seek` 中又调用了 `FindGreaterOrEqual`，下面看看它们的实现逻辑。

```c++
// https://github.com/google/leveldb/blob/master/db/skiplist.h

template <typename Key, class Comparator>
  inline void SkipList<Key, Comparator>::Iterator::Seek(const Key& target) {
    node_ = list_->FindGreaterOrEqual(target, nullptr);
  }

  SkipList<Key, Comparator>::FindGreaterOrEqual(const Key& key,
                                                Node** prev) const {
      //从顶层开始查询
      Node* x = head_;
      int level = GetMaxHeight() - 1;

      while (true) {
          Node* next = x->Next(level);

          //如果当前节点的值小于要查询的值，则在该层继续查找
          if (KeyIsAfterNode(key, next)) {
              x = next;
          } else {  
              //如果大于等于，则说明不可能在该层，前往下一层查找。

              if (prev != nullptr) prev[level] = x;
              //如果查询到底就直接返回，此时有两种情况1.查询成功，返回底层结果 2.查询失败，返回对应最底层位置 
              if (level == 0) {
                  return next;
              } else {
                  // Switch to next list
                  level--;
              }
          }
      }
  }
```

查找逻辑与常规 SkipList 实现一样：

  1. 从顶层开始查询。
  2. 对比当前阶段的值是否小于查询的值：
     - 小于：沿着当前层继续查找。
     - 大于等于：前往下一层查找。
  3. 判断当前层数是否到底，没到就继续往下走。
  4. 到底了返回数据，如果返回的数据与 key 相同则说明查询成功，否则失败。

  

### 插入

```c++
// https://github.com/google/leveldb/blob/master/db/skiplist.h

template <typename Key, class Comparator>
  void SkipList<Key, Comparator>::Insert(const Key& key) {
    //记录每一层级查找到的位置
    Node* prev[kMaxHeight];
    //查找适合插入的位置
    Node* x = FindGreaterOrEqual(key, prev);

    assert(x == nullptr || !Equal(key, x->key));

    //获取本次插入的最高层数
    int height = RandomHeight();
    //如果本次插入的最高层数大于目前最高层数，则将多出的几层指向新插入节点
    if (height > GetMaxHeight()) {
      for (int i = GetMaxHeight(); i < height; i++) {
        prev[i] = head_;
      }
      max_height_.store(height, std::memory_order_relaxed);
    }

    x = NewNode(key, height);
    //将需要更新的节点依次更新
    for (int i = 0; i < height; i++) {
      x->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i));
      prev[i]->SetNext(i, x);
    }
  }
```

具体逻辑如下：

  1. 首先用一个数组存储每一层所查找到的位置。
  2. 使用 `FindGreaterOrEqual` 获取适合插入的位置。
  3. 通过 `RandomHeight` 获取本次插入的最高层数：
     - 如果本次插入的最高层数大于目前最高层数，则将多出的几层指向新插入节点。
     - 如果小于等于，则无需更新。
  4. 遍历 prev，将需要更新的节点依次更新。

  

## MemTable写入SSTable

在 LSM 树的实现中，会先将数据写入 MemTable，当 MemTable 大于配置的阈值时，将其作为 SSTable 写入磁盘。

这里我们只看核心逻辑，代码如下：

```c++
// https://github.com/google/leveldb/blob/master/db/db_impl.cc

Status DBImpl::WriteLevel0Table(MemTable* mem, VersionEdit* edit,
                                Version* base) {
    //...

    //生成一个MemTable迭代器
    Iterator* iter = mem->NewIterator();
    Log(options_.info_log, "Level-0 table #%llu: started",
        (unsigned long long)meta.number);

    Status s;
    {
        mutex_.Unlock();
        //调用BuildTable，将MemTable迭代器作为参数传入，生成一个SSTable
        s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);
        mutex_.Lock();
    }

    //...
    return s;
}
```

在这里首先会生成一个 MemTable 迭代器，调用 `BuildTable` ，将 MemTable 迭代器作为参数传入，生成一个 SSTable。

  我们接着来看 `BuildTable`  的逻辑：

```c++
// https://github.com/google/leveldb/blob/master/db/builder.cc

Status BuildTable(const std::string& dbname, Env* env, const Options& options,
                  TableCache* table_cache, Iterator* iter, FileMetaData* meta) {
    Status s;
    meta->file_size = 0;
    //将迭代器移动到首部
    iter->SeekToFirst();

    //生成SSTable文件名
    std::string fname = TableFileName(dbname, meta->number);
    if (iter->Valid()) {
        WritableFile* file;
        s = env->NewWritableFile(fname, &file);
        if (!s.ok()) {
            return s;
        }
        //创建一个TableBuilder
        TableBuilder* builder = new TableBuilder(options, file);
        meta->smallest.DecodeFrom(iter->key());
        Slice key;

        //遍历迭代器，将MemTable中的每一对K-V写入TableBuilder中
        for (; iter->Valid(); iter->Next()) {
            key = iter->key();
            builder->Add(key, iter->value());
        }
        if (!key.empty()) {
            meta->largest.DecodeFrom(key);
        }

        //调用TableBuilder的Finish函数生成SSTable文件
        s = builder->Finish();
        if (s.ok()) {
            meta->file_size = builder->FileSize();
            assert(meta->file_size > 0);
        }
        delete builder;

        //调用Sync将文件刷新到磁盘中	
        if (s.ok()) {
            s = file->Sync();
        }

        //关闭文件
        if (s.ok()) {
            s = file->Close();
        }

        //...
        return s;
    }
```

核心逻辑如下：

  1. 将 MemTable 迭代器移动到首部。
  2. 生成 SSTable 文件名。
  3. 创建一个 TableBuilder。
  4. 遍历迭代器，将 MemTable 中的每一对 K-V 写入 TableBuilder 中。
  5. 调用 TableBuilder 的 `Finish` 函数生成 SSTable 文件。
  6. 调用 `Sync` 将文件刷新到磁盘中。
  7. 关闭文件。

  