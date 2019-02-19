# Bigtable

> Bigtable是Google内部的一个分布式结构化数据存储系统，被设计用来存储PB级的数据

### 数据模型

* Bigtable是一个稀疏的多纬度排序Map， Map的索引是行关键字，列关键字，以及时间戳
```json
 (row:string, column:string,time:int64)->string
```
* 行关键字可以是任意字符串，对同一个行关键字的读写操作都是原子的，即单行事务
* 列关键字(Column Key)必须属于某个列族(Column Families)， 数据以列为纬度进行压缩
* 时间戳： 通过时间戳可以保存数据的多个版本

### 架构

* Bigtable 将数据按照行关键字把数据划分成多个Tablet，每个Slave节点会加载多个Tablet，即Tablet Server
* Bigtable 使用一个三层的、类似Ｂ+树[10]的结构存储Tablet的位置信息, 如下图![bigtable_master](./bigtable_master.jpeg

* Root Tablet的位置记在chubby中, 且不可分裂
* 客户端会缓存Tablet的位置信息，如果缓存为空， 那么客户端需要通过三次网络通信来完成寻址（一次chubby读，一次root tablet访问，一次metadata tablet访问）。如果缓存内容过期的话，可能需要最多6次（多三次通信发现缓存过期）
* 每一个启动的Tablet Server会在chubby创建一个唯一的锁文件，master通过监控chubby来发现新的tablet服务器启动，并和这些服务器通信，取得当前正在服务的tablet信息
* tablet的写操作和[leveldb](/projects/leveldb.md)类似
* 缓存操作， Tablet Server会提供二级缓存，第一级是从tablet中读取的key-value对（更新数据时需要考虑缓存失效？）， 第二缓存是从GFS中读取的SSTable的Block， 由于LSM-Tree的特性， 生成新SSTable文件后不会被修改，因此不需要考虑第二级缓存的失效问题
* 操作日志，Bigtable 在写入操作日志的时候，同一个Tablet Server的tablet的日志混合在一起，写到同一个文件，为了恢复的时候读取日志加速，日志被分割成64MB大小的段，master会安排Tablet Server对日志进行多路归并排序 （类似于leveldb的leveled compaction）

###

* Bigtable如何保证tablet只被一个Tablet Server服务？ 考虑以下两种情况：
  * 某个tablet x从Tablet Server A 迁移到Tablet Server B， 假设在迁移过程中A宕机，由于迁移未完成，那么x的信息尚未被写入到Meta Server，因此tablet x仍然属于B；假设在迁移结束后，A宕机宕机，由于此时A、B都拥有完整的数据，因此A重新启动后，如果A发现自己的tablet已经被B加载，那么他应该放弃此次注册(理论上来说，A还可以和B对比双方的日志完整度，但不知道没有raft强一致性同步的情况，这种对比是否会存着误差，括号里都是我瞎说的)
  * 某个tablet x原本属于Tablet Server A，然而A因为网络中断或者其他故障，和master短暂失联，失联期间，master检查到A已经无法提供服务了，于是将x分配给了Tablet Server B，并且将该次结果写入到了MetaServer中。在此之后，A又重新恢复了服务，那么如果某个客户端持有的MetaServer的缓存没有及时更新，他会仍然寻址到A上。 由于A此时已经失去了Tablet Server锁，他会拒绝掉此次服务，让客户端知道缓存过期并且获取最新的Metadata信息。

### 单行事务

* 在bigtable的开源实现中，hbase通过行锁实现对单行数据的原子更新， 所有数据更新前都必须获得行锁
* MVCC， 为读写操作分别分配操作序号，同时记录全局的读取点memstoreRead，所有的未完成的写操作都在writeQueue，当某个更新线程完成后，从memstoreRead开始往后移动，直到遇到一个未完成的写操作，将所有完成写操作的踢出队列。如果操作完成时，memstoreRead小于当前操作的写序号WriteNumber，那么必须等待memstoreRead完成自己前面的操作。 当memstoreRead大于等于读操作的序号ReadNumber时，筛选出ReadNumber以后的所有key-value对，然后读取第一个目标key的值
