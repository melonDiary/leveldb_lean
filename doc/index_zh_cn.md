# LevelDB 文档

_Jeff Dean, Sanjay Ghemawat_

LevelDB 库提供了一个持久化的键值存储。键和值都是任意字节数组。键值存储中的键按照用户指定的比较器函数进行排序。

## 打开数据库

LevelDB 数据库有一个名称，对应于文件系统目录。数据库的所有内容都存储在该目录中。以下示例展示了如何打开数据库，必要时创建它：

```c++
#include <cassert>
#include "leveldb/db.h"

leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
assert(status.ok());
...
```

如果你想在数据库已存在时抛出错误，请在 `leveldb::DB::Open` 调用之前添加以下行：

```c++
options.error_if_exists = true;
```

## 状态

你可能注意到上面的 `leveldb::Status` 类型。LevelDB 中大多数可能遇到错误的函数都会返回这种类型的值。你可以检查这样的结果是否正确，也可以打印相关的错误消息：

```c++
leveldb::Status s = ...;
if (!s.ok()) cerr << s.ToString() << endl;
```

## 关闭数据库

当你使用完数据库后，只需删除数据库对象。示例：

```c++
... 按上述方式打开数据库 ...
... 对数据库进行操作 ...
delete db;
```

## 读写操作

数据库提供 Put、Delete 和 Get 方法来修改/查询数据库。例如，以下代码将存储在 key1 下的值移动到 key2：

```c++
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);
if (s.ok()) s = db->Put(leveldb::WriteOptions(), key2, value);
if (s.ok()) s = db->Delete(leveldb::WriteOptions(), key1);
```

## 原子更新

注意，如果进程在 Put key2 之后但 Delete key1 之前崩溃，相同的值可能会存储在多个键下。可以通过使用 `WriteBatch` 类来原子性地应用一组更新来避免此类问题：

```c++
#include "leveldb/write_batch.h"
...
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);
if (s.ok()) {
  leveldb::WriteBatch batch;
  batch.Delete(key1);
  batch.Put(key2, value);
  s = db->Write(leveldb::WriteOptions(), &batch);
}
```

`WriteBatch` 保存了一系列对数据库的编辑操作，这些编辑在批处理中按顺序应用。注意我们在 Put 之前调用 Delete，这样如果 key1 与 key2 相同，我们就不会错误地完全删除该值。

除了原子性的好处外，`WriteBatch` 还可以通过将大量单独的变更放入同一个批处理中来加速批量更新。

## 同步写入

默认情况下，LevelDB 的每次写入都是异步的：它在将写入从进程推送到操作系统后返回。从操作系统内存到底层持久存储的传输是异步进行的。可以为特定写入打开 sync 标志，使写入操作在所写入的数据一直推送到持久存储后才返回。（在 Posix 系统上，这是通过在写入操作返回之前调用 `fsync(...)` 或 `fdatasync(...)` 或 `msync(..., MS_SYNC)` 来实现的。）

```c++
leveldb::WriteOptions write_options;
write_options.sync = true;
db->Put(write_options, ...);
```

异步写入通常比同步写入快一千倍以上。异步写入的缺点是机器崩溃可能导致最后几次更新丢失。注意，只是写入进程崩溃（即不是重启）不会导致任何丢失，因为即使 sync 为 false，更新在被认为完成之前也会从进程内存推送到操作系统。

异步写入通常可以安全使用。例如，在将大量数据加载到数据库时，可以通过在崩溃后重新启动批量加载来处理丢失的更新。也可以采用混合方案，即每第 N 次写入是同步的，在崩溃时，批量加载从上一次运行完成的最后一次同步写入之后重新开始。（同步写入可以更新一个标记，描述在崩溃时从哪里重新开始。）

`WriteBatch` 提供了异步写入的替代方案。可以将多个更新放在同一个 WriteBatch 中，并使用同步写入一起应用（即，`write_options.sync` 设置为 true）。同步写入的额外成本将分摊到批处理中的所有写入上。

## 并发

数据库一次只能被一个进程打开。LevelDB 实现从操作系统获取锁以防止误用。在单个进程内，同一个 `leveldb::DB` 对象可以被多个并发线程安全地共享。即，不同的线程可以在没有任何外部同步的情况下对同一个数据库进行写入或获取迭代器或调用 Get（LevelDB 实现将自动执行所需的同步）。但是，其他对象（如 Iterator 和 `WriteBatch`）可能需要外部同步。如果两个线程共享这样的对象，它们必须使用自己的锁协议来保护对它的访问。更多详细信息请参阅公共头文件。

## 迭代

以下示例演示如何打印数据库中的所有键值对。

```c++
leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  cout << it->key().ToString() << ": " << it->value().ToString() << endl;
}
assert(it->status().ok()); // 检查扫描过程中发现的任何错误
delete it;
```

以下变体展示了如何只处理范围 [start,limit) 内的键：

```c++
for (it->Seek(start);
     it->Valid() && it->key().ToString() < limit;
     it->Next()) {
  ...
}
```

你也可以按逆序处理条目。（注意：逆向迭代可能比正向迭代慢一些。）

```c++
for (it->SeekToLast(); it->Valid(); it->Prev()) {
  ...
}
```

## 快照

快照提供了对键值存储整个状态的一致只读视图。`ReadOptions::snapshot` 可以为非 NULL，以表示读取应在 DB 状态的特定版本上进行。如果 `ReadOptions::snapshot` 为 NULL，则读取将在当前状态的隐式快照上进行。

快照由 `DB::GetSnapshot()` 方法创建：

```c++
leveldb::ReadOptions options;
options.snapshot = db->GetSnapshot();
... 对数据库进行一些更新 ...
leveldb::Iterator* iter = db->NewIterator(options);
... 使用 iter 读取以查看创建快照时的状态 ...
delete iter;
db->ReleaseSnapshot(options.snapshot);
```

注意，当不再需要快照时，应使用 `DB::ReleaseSnapshot` 接口释放它。这允许实现删除仅为支持该快照读取而维护的状态。

## Slice

上面的 `it->key()` 和 `it->value()` 调用的返回值是 `leveldb::Slice` 类型的实例。Slice 是一个简单的结构，包含一个长度和一个指向外部字节数组的指针。返回 Slice 是返回 `std::string` 的更便宜的替代方案，因为我们不需要复制可能很大的键和值。此外，LevelDB 方法不返回以 null 结尾的 C 风格字符串，因为 LevelDB 的键和值允许包含 `'\0'` 字节。

C++ 字符串和以 null 结尾的 C 风格字符串可以轻松转换为 Slice：

```c++
leveldb::Slice s1 = "hello";

std::string str("world");
leveldb::Slice s2 = str;
```

Slice 可以轻松转换回 C++ 字符串：

```c++
std::string str = s1.ToString();
assert(str == std::string("hello"));
```

使用 Slice 时要小心，因为调用者需要确保 Slice 指向的外部字节数组在 Slice 使用期间保持有效。例如，以下代码有 bug：

```c++
leveldb::Slice slice;
if (...) {
  std::string str = ...;
  slice = str;
}
Use(slice);
```

当 if 语句超出作用域时，str 将被销毁，slice 的后备存储将消失。

## 比较器

前面的示例使用了键的默认排序函数，即按字典序排列字节。但是，你可以在打开数据库时提供自定义比较器。例如，假设每个数据库键由两个数字组成，我们应该按第一个数字排序，如果相同则按第二个数字排序。首先，定义一个 `leveldb::Comparator` 的适当子类来表达这些规则：

```c++
class TwoPartComparator : public leveldb::Comparator {
 public:
  // 三路比较函数：
  // 如果 a < b：返回负数
  // 如果 a > b：返回正数
  // 否则：返回零
  int Compare(const leveldb::Slice& a, const leveldb::Slice& b) const {
    int a1, a2, b1, b2;
    ParseKey(a, &a1, &a2);
    ParseKey(b, &b1, &b2);
    if (a1 < b1) return -1;
    if (a1 > b1) return +1;
    if (a2 < b2) return -1;
    if (a2 > b2) return +1;
    return 0;
  }

  // 暂时忽略以下方法：
  const char* Name() const { return "TwoPartComparator"; }
  void FindShortestSeparator(std::string*, const leveldb::Slice&) const {}
  void FindShortSuccessor(std::string*) const {}
};
```

现在使用此自定义比较器创建数据库：

```c++
TwoPartComparator cmp;
leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
options.comparator = &cmp;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
...
```

### 向后兼容性

比较器的 Name 方法的结果在创建数据库时附加到数据库，并在每次后续数据库打开时检查。如果名称更改，`leveldb::DB::Open` 调用将失败。因此，当且仅当新的键格式和比较函数与现有数据库不兼容，并且可以丢弃所有现有数据库的内容时，才更改名称。

但是，你仍然可以通过一些预先规划随时间逐渐演进键格式。例如，你可以在每个键的末尾存储一个版本号（一个字节对大多数用途应该足够）。当你希望切换到新的键格式（例如，添加可选的第三部分给 `TwoPartComparator` 处理的键）时，(a) 保持相同的比较器名称 (b) 为新键增加版本号 (c) 更改比较器函数，使其使用键中的版本号来决定如何解释它们。

## 性能

可以通过更改 `include/options.h` 中定义的类型的默认值来调整性能。

### 块大小

LevelDB 将相邻的键组合到同一个块中，这样的块是与持久存储之间传输的单位。默认块大小约为 4096 字节（未压缩）。主要进行批量扫描数据库内容的应用程序可能希望增加此大小。大量进行小值点读取的应用程序可能希望切换到较小的块大小，如果性能测量显示有改善。使用小于一千字节或大于几兆字节的块没有太大好处。还要注意，较大的块大小会使压缩更有效。

### 压缩

每个块在写入持久存储之前单独压缩。默认情况下压缩是开启的，因为默认压缩方法非常快，并且对于不可压缩的数据会自动禁用。在极少数情况下，应用程序可能想要完全禁用压缩，但只有在基准测试显示性能改善时才应这样做：

```c++
leveldb::Options options;
options.compression = leveldb::kNoCompression;
... leveldb::DB::Open(options, name, ...) ....
```

### 缓存

数据库的内容存储在文件系统中的一组文件中，每个文件存储一系列压缩块。如果 options.block_cache 不为 NULL，则用于缓存常用的未压缩块内容。

```c++
#include "leveldb/cache.h"

leveldb::Options options;
options.block_cache = leveldb::NewLRUCache(100 * 1048576);  // 100MB 缓存
leveldb::DB* db;
leveldb::DB::Open(options, name, &db);
... 使用数据库 ...
delete db
delete options.block_cache
```

注意，缓存保存的是未压缩的数据，因此应根据应用程序级数据大小来设置，不考虑压缩的影响。（压缩块的缓存留给操作系统缓冲区缓存，或客户端提供的任何自定义 Env 实现。）

执行批量读取时，应用程序可能希望禁用缓存，以便批量读取处理的数据不会最终取代大部分缓存内容。可以使用每次迭代的选项来实现这一点：

```c++
leveldb::ReadOptions options;
options.fill_cache = false;
leveldb::Iterator* it = db->NewIterator(options);
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  ...
}
delete it;
```

### 键布局

注意，磁盘传输和缓存的单位是块。相邻的键（按数据库排序顺序）通常会被放在同一个块中。因此，应用程序可以通过将一起访问的键放在彼此附近，并将不常用的键放在键空间的单独区域来提高性能。

例如，假设我们在 LevelDB 之上实现一个简单的文件系统。我们可能希望存储的条目类型有：

```
filename -> 权限位, 长度, file_block_id 列表
file_block_id -> 数据
```

我们可能希望用一字母（比如 '/'）作为文件名键的前缀，用不同的字母（比如 '0'）作为 `file_block_id` 键的前缀，这样仅对元数据的扫描就不会迫使我们获取和缓存庞大的文件内容。

### 过滤器

由于 LevelDB 数据在磁盘上的组织方式，单个 `Get()` 调用可能涉及多次磁盘读取。可选的 FilterPolicy 机制可以大大减少磁盘读取次数。

```c++
leveldb::Options options;
options.filter_policy = NewBloomFilterPolicy(10);
leveldb::DB* db;
leveldb::DB::Open(options, "/tmp/testdb", &db);
... 使用数据库 ...
delete db;
delete options.filter_policy;
```

上面的代码将基于布隆过滤器的过滤策略与数据库关联。基于布隆过滤器的过滤依赖于每个键在内存中保留一些数据位（在这种情况下每个键 10 位，因为这是我们传递给 `NewBloomFilterPolicy` 的参数）。此过滤器将把 Get() 调用所需的不必要磁盘读取次数减少约 100 倍。增加每个键的位数会导致更大的减少，但会增加内存使用。我们建议工作集不适合内存且进行大量随机读取的应用程序设置过滤策略。

如果你使用自定义比较器，应确保所使用的过滤策略与比较器兼容。例如，考虑一个在比较键时忽略尾随空格的比较器。`NewBloomFilterPolicy` 不能与此类比较器一起使用。相反，应用程序应提供也忽略尾随空格的自定义过滤策略。例如：

```c++
class CustomFilterPolicy : public leveldb::FilterPolicy {
 private:
  leveldb::FilterPolicy* builtin_policy_;

 public:
  CustomFilterPolicy() : builtin_policy_(leveldb::NewBloomFilterPolicy(10)) {}
  ~CustomFilterPolicy() { delete builtin_policy_; }

  const char* Name() const { return "IgnoreTrailingSpacesFilter"; }

  void CreateFilter(const leveldb::Slice* keys, int n, std::string* dst) const {
    // 在删除尾随空格后使用内置布隆过滤器代码
    std::vector<leveldb::Slice> trimmed(n);
    for (int i = 0; i < n; i++) {
      trimmed[i] = RemoveTrailingSpaces(keys[i]);
    }
    builtin_policy_->CreateFilter(trimmed.data(), n, dst);
  }
};
```

高级应用程序可以提供不使用布隆过滤器但使用其他机制来总结一组键的过滤策略。有关详细信息，请参阅 `leveldb/filter_policy.h`。

## 校验和

LevelDB 为其存储在文件系统中的所有数据关联校验和。关于这些校验和的验证程度，提供了两个单独的控制：

`ReadOptions::verify_checksums` 可以设置为 true，以强制对代表特定读取从文件系统读取的所有数据进行校验和验证。默认情况下，不进行此类验证。

`Options::paranoid_checks` 可以在打开数据库之前设置为 true，使数据库实现在检测到内部损坏时立即抛出错误。根据数据库的哪个部分被损坏，错误可能在打开数据库时抛出，或稍后由另一个数据库操作抛出。默认情况下，偏执检查是关闭的，以便即使持久存储的某些部分已损坏，数据库仍可使用。

如果数据库损坏（可能在打开偏执检查时无法打开），可以使用 `leveldb::RepairDB` 函数尽可能多地恢复数据。

## 近似大小

`GetApproximateSizes` 方法可用于获取一个或多个键范围使用的文件系统空间的近似字节数。

```c++
leveldb::Range ranges[2];
ranges[0] = leveldb::Range("a", "c");
ranges[1] = leveldb::Range("x", "z");
uint64_t sizes[2];
db->GetApproximateSizes(ranges, 2, sizes);
```

上面的调用将 `sizes[0]` 设置为键范围 `[a..c)` 使用的文件系统空间的近似字节数，将 `sizes[1]` 设置为键范围 `[x..z)` 使用的近似字节数。

## 环境

LevelDB 实现发出的所有文件操作（和其他操作系统调用）都通过 `leveldb::Env` 对象进行路由。复杂的客户端可能希望提供自己的 Env 实现以获得更好的控制。例如，应用程序可能在文件 IO 路径中引入人为延迟，以限制 LevelDB 对系统中其他活动的影响。

```c++
class SlowEnv : public leveldb::Env {
  ... Env 接口的实现 ...
};

SlowEnv env;
leveldb::Options options;
options.env = &env;
Status s = leveldb::DB::Open(options, ...);
```

## 移植

可以通过提供 `leveldb/port/port.h` 导出的类型/方法/函数的平台特定实现将 LevelDB 移植到新平台。有关更多详细信息，请参阅 `leveldb/port/port_example.h`。

此外，新平台可能需要新的默认 `leveldb::Env` 实现。请参阅 `leveldb/util/env_posix.h` 作为示例。

## 其他信息

有关 LevelDB 实现的详细信息，请参阅以下文档：

1. [实现说明](impl.md)
2. [不可变 Table 文件格式](table_format.md)
3. [日志文件格式](log_format.md)
