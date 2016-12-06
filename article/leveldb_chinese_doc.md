# LevelDB中文文档

> 欢迎转载，转载请注明出处！

> Html版本: [leveldb\_chinese\_doc](http://kevins.pro/blog/leveldb_chinese_doc/)<br/>
> MarkDown版本: [leveldb\_source\_01\_data\_structure.md](http://github.com/KevinsBobo/KevinsBobo.github.io/blob/master/article/leveldb_chinese_doc.md)

> [Follow me on GitHub ^\_^](http://github.com/KevinsBobo/)

LevelDB提供一个持久的键值存储库。keys 和 values 都可以是任意的字节数组。key是通过用户提供的比较器函数在键值存储器内进行排序的。

## 创建并打开一个数据库

一个LevelDB数据库有一个名字并且对应一个文件系统的文件夹。所有的数据库内容都存储在这个文件夹下。在下面的例子中展示了怎样创建并打开一个数据库：
```cpp
#include <cassert>
#include "leveldb/db.h"

leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
assert(status.ok());
...
```

如果想要在存在已创建数据库的情况下引发错误，则在`leveldb::DB::Open`前加上<br/>
```cpp
options.error_if_exists = true;
```

## 状态 - Status

你可能已经注意到了上面的`leveldb::Status`这个类型。在`leveldb`中可能遇到错误的函数大多都返回这个类型的值。你可以检查返回的结果是不是正确执行，并且可以打印相关联的错误信息：
```cpp
leveldb::Status s = ...;
if (!s.ok()) cerr << s.ToString() << endl;
```

## 关闭一个数据库

当你完成数据库相关操作后，只用删除这个数据库对象就可以了。例：
```cpp
... open the db as described above ...
... do something with db ...
delete db;
```

## 读写操作

数据库提供`Put, Delete, Get`这些方法来修改和查询数据库。比如，以下操作时将存储在`key1`的值移动到`key2`中去：
```cpp
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);
if (s.ok()) s = db->Put(leveldb::WriteOptions(), key2, value);
if (s.ok()) s = db->Delete(leveldb::WriteOptions(), key1);
```

## 原子更新 - [Atomic Updates](https://en.wikipedia.org/wiki/Atomicity_(database_systems))

注意：上面的操作如果进程在`Put key2`和`Delete key1`两个操作之间结束，那么这两个键将存储相同的值。因此，尽可能使用`WriteBatch`类来避免这类问题：
```cpp
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

`WriteBatch`对象保存对数据库进行的一系列操作，然后在这一批次中按照顺序应用这些操作。注意：这里先进行`Delete`操作，然后再进行`Put`操作，是因为在`key1`和`key2`相同的情况下不会错误的将该值丢弃。

`WriteBatch`类除了原子性的优势外，也可以用于通过将大量个体变动放置在同一批次中而加速批量更新。

## 同步写入

在默认情况下，`levedb`中的每一次写操作都是异步的：它会在把写入操作从进程中推送到操作系统后返回，而从操作系统内存到底层持久化存储的传输是异步的。对于特定的写操作，是可以打开同步`sync`标志使写操作一直到数据被传输到底层存储器后再返回。（在基于Posix标准的操作系统系统中，这一步是通过在写操作返回之前调用`fsync(...)`或`fdatasync(...)`或`nsync(..., MS_SYNC)`实现的。）
```cpp
leveldb::WriteOptions write_options;
write_options.sync = true;
db->Put(write_options, ...);
```

异步写操作一般比同步写操作快很多很多。但异步写入的缺点是，在机器宕机时有可能导致最后几步的更新丢失。请注意，如果是在写入过程中的宕机（而非重新启动），即使`sync`设置为`false`，更新操作也会认为已经将更新从内存中推送到了操作系统。

通常可以安全地使用异步写入。比如，当加载大量数据到数据库中时，可以通过在宕机后重新启动批量加载来处理丢失的更新。有一个可用的混合方案，将多次写入的第N次写入设置为同步的，并在宕机重启后的情况下，批量加载由前一次运行的最后一次同步写入之后重新开始。（同步写入时可以更新描述宕机后批量加载重新开始的标记。）

`WriteBatch`提供一个代替异步写操作的选择：将多个更新操作放置在同一个`WriteBatch`对象中然后使用同步写入一起应用（`write_options.sync`设置为`ture`）；这时同步写入的额外成本开销将在这批次中所有的写入操作中摊销。

## 并发

一个数据库在同一时间内只能由一个进程打开，`leveldb`通过从操作系统中获取锁的方式防止误用。在单个进程中，同一个`leveldb::DB`对象可以安全的由多个并发线程共享使用。即，不同的线程可以同时写入或获取迭代器，或在没有任何外部同步的情况在同一数据库上调用`Get`（`leveldb`的实现过程中将自动自行所需的同步）。但是其他对象（如`Iterator`和`WriteBatch`）可能需要外部同步。如果两个线程共享这样的对象，它们必须使用自己的协议锁来保护自己的访问。更多详细的细节在公共的头文件中描述。

## 迭代

下面的例子演示了如何从数据库中成对的打印键值：
```cpp
leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  cout << it->key().ToString() << ": "  << it->value().ToString() << endl;
}
assert(it->status().ok());  // Check for any errors found during the scan
delete it;
```

下面的修改后的例子演示了如何仅获取`[start, limit)`范围内的键；
```cpp
for (it->Seek(start);
     it->Valid() && it->key().ToString() < limit;
     it->Next()) {
  ...
}
```

还可以通过相反的顺序进行处理。（警告：反向迭代比正向迭代慢一些）
```cpp
for (it->SeekToLast(); it->Valid(); it->Prev()) {
  ...
}
```

## 快照 - [Snapshots](https://msdn.microsoft.com/zh-cn/library/ms175158.aspx)

快照在键值存储的整体状态上提供了一致的只读视图。`ReadOptions::snapshot`可能是`non-NULL`，表示读取操作在特定的`DB`版本状态上进行的。如果`ReadOptions::snapshot`是`NULL`，则读取操作将在当前状态上进行隐式的快照操作。

快照是通过`DB::GetSnapshot`方法进行创建的：
```cpp
leveldb::ReadOptions options;
options.snapshot = db->GetSnapshot();
... apply some updates to db ...
leveldb::Iterator* iter = db->NewIterator(options);
... read using iter to view the state when the snapshot was created ...
delete iter;
db->ReleaseSnapshot(options.snapshot);
```

注意：当一个快照长期不用时，应该通过`DB::ReleaseSnapshot`接口释放它。这样既可以让底层实现丢弃那些为支持该快照的读取操作而进行维护的一些状态数据。

## Slice - LevelDB使用的数据结构
