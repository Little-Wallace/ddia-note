# Rocksdb

## Rocksdb的事务

> Rocksdb中的事务分为悲观事务，和乐观事务，分别PessimisticTransactionDB与OptimisticTransactionDB中。悲观事务在每次修改数据时，会尝试加锁，加锁失败则等待至锁释放或事务超时；乐观事务则会记下事务开始时的log sequence number， 在事务提交时检查提交数据是否存在log sequence number更大的新值，若存在，则此次事务回滚

## 实现细节

* 事务的原子性
  rocksdb利用log sequence number来实现事务的原子性，同一个事务提交的所有数据会被写入到同一个WriteBatch中一起持久化，WriteBatch如何实现快照隔离参见[leveldb](/projects/leveldb.md)中的实现。
* 悲观事务
  * rocksdb会为每个column分配一个lock map，默认每一列lock map只有16个锁，key会hash后定位到某一个锁上，然后试图抢占这个锁。这也意味着或许会存在完全不想干的两个key争抢同一个锁。
  * 在获取lock map时，由于map本身是线程不安全的数据结构必须为此加上锁（虽然你可以设计出线程安全的hashmap，然而多线程操作带来的cpu cache miss就足以在高频操作极大地损耗性能）， rocksdb为了优化TryLock的性能，利用thread_local变量，为每个线程缓存了所有列的id到lock map的映射关系。
* 乐观事务（快照隔离）
  * rocksdb的乐观事务通过log sequence number来实现，每个事务开始时会将当前global lsn设置为自己的snapshot（用户也可以手动设置snapshot来实现历史快照的读取），之后的读取和写入都会以该snapshot来进行，提交时通过检测lsn的变化情况即可得知是否存在事务与自己冲突。
  * 检查lsn的实现方法也很简单，每个WriteBatch会持有一个callback函数指针，在写入WAL前的时候调用这个callback函数，callback函数内部会调用db的Get方法，读取提交数据的最新lsn。由于写入WAL操作只会有一个线程执行，因此可以认为callback与其他事务的WAL操作不存在线程冲突。

### 隔离级别
* 当未设置read option中的snapshot时，事务隔离级别为读已提交。
* 当设置了read option中的snapshot时，事务隔离级别为可重复读（快照隔离）。
* rocksdb由于没有类似mysql那样的gap锁（区间锁，或者使用谓词锁也能达到相同的目的），因此无法实现可串行化。（是否可由上层应用提供表级别锁以实现可串行化？）
