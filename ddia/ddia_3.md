# 数据存储与检索

### hash索引

* key值存在hash table中(全内存)， key-value对顺序写入日志（write ahead log）， 日志分段存储（以段为单位进行压缩，剔除重复数据）
* 故障恢复: 遍历日志，将key重新取出来构成hash table。为了减少故障恢复遍历日志的时间，通常会定期将hashtable生成快照存到磁盘，重启时先读取快照中的key, 然后从快照生成的时间点开始读取日志。
* 优势: 全内存加载的hashtable查找速度快, 合并段以及追加段的操作为顺训写，对ssd友好
* 缺点: hash表不适合磁盘存储，必须放入内存中，对于value较小，且不重复key数量较多的场景需要耗费大量内存; 不支持区间查询; hash表变满时，继续增长代价昂贵。（hash算法以及扩充hash表的重hash见[hash算法](/projects/hash.md)）

### LSM-Tree

即， Log Structure Merged Tree，将随即写操作转化为顺序写操作的数据结构。 leveldb的核心数据结构包括两个部分: memtable和sstable（sorted string table）， memtable可以是skiplist或者AVL之类的数据结构，sstable则是按照key有序的key-value对数组。 当memtable的内存占用达到一定阙值时，转化为sstable，然后顺序写入磁盘。 查找某一个key时，先检查memtable中是否存在，然后从新到旧查询所有的sstable，如果查到了就提前返回结果。为了减少查询多个sstable带来的开销，后台线程会按照一定的策略对sstable进行合并。

* 故障恢复： sstable中记录了数据在日志中的位置，因此只需要恢复memtable丢失的那部分数据。
* 实现细节见[leveldb](/projects/leveldb.md)。
* 优点:由于写入操作只发生在内存中以及顺序写如日志（为了防止内存数据丢失，即WAL）, 因此写入效率高, 对ssd友好，压缩效率高。
* 缺点:读放大，每次查找需要查找多个sstable，在密集写入的情况下，大量sstable来不及被合并，会严重影响到查询速度; 写放大， 同一个key会被反复写入sstable；合并操作会影响正常读写操作的磁盘IO; 在事务型数据库中，对某个范围的key加锁不方便

### B-Tree

> B Tree 与B+ Tree算法细节见[wiki](https://zh.wikipedia.org/zh-hans/B%2B%E6%A0%91)

将数据切割成4KB大小的页，同时B tree的每一个节点也是4KB大小的页，这样的数据结构相比于红黑树这样的AVL更适合磁盘存储，因为现代计算机磁盘的访问和存储通常都以4KB为单位, 即写入的时候即使只更改了1B的数据，也要将这4KB的页完全擦除重写。

### 二级索引

通常用在关系型数据库中，对table的某一列建立二级索引，key是该列的值，value则是主键(或者唯一标识的ID)

### 聚集索引

索引和值存储在一起。

### 多列索引

* 级联索引(concatenated index): 将几个字段的值拼接起来，然后建立索引。
* 多维索引: 如查询某个经纬度范围内的餐馆
  * 数据结构: KD-Tree, R-Tree.
  * 其他实现：利用geo hash 将范围拆分成多个不同层级小范围，然后建立索引。
  

