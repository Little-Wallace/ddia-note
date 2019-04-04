# The Google File System

场景： 存储的绝大多数文件是大文件（通常在100MB以上）；绝大部分文件的变更是采用在追加新数据，而不是重写原有数据的方式；一旦写完之后，文件只能读，而且通常只能顺序读

### 架构

* 一个GFS 集群包含一个单独的master和多台chunkserver
* 文件都被分割成固定大小的块。master会对每个块分配一个不可变、唯一的64位的块句柄来进行标识。Chunkserver将块作为linux 文件保存在本地硬盘上，并且根据指定的块句柄和字节范围来读写块数据。为了可靠，每个块都会复制到多个chunkserver上。
* Master维护所有的文件系统元数据。这些元数据包括:
  * 命名空间
  * 访问控制信息
  * 文件到块的映射信息
  * 块的位置。
* Master节点还控制着系统范围内的活动，比如，块租用管理 、孤立块的垃圾回收、以及块在Chunkserver间的迁移。Master用心跳信息周期地和每个Chunkserver通讯，向Chunkserver发送指令并收集其状态。
* 客户端询问master它应该联系哪一个Chunkserver, 然后和chunserver交互并进行读写操作
* Chunk的单个大小为64MB，master为每个chunk维护了少于64B的元数据信息， 所有chunk的元数据信息都存储在master的内存中，也就是说，GFS的单个集群容量上限为x PB，x为master的内存大小xGB。
* master通过操作日志进行故障恢复，定期进行快照以缩短日志文件大小

### 一致性模型

* 命名空间的变更是由master单机完成的（包括文件的删除和创建以及移动），命名空间锁保证了原子性和正确性。
* 单个客户端写文件时，所有chunk server都写入成功后才会在master 元数据中写入成功， 并返回给客户端
* 多个客户端并发写相同的文件时，primary chunk server会对写操作进行排序，然后通知其他副本的chunk server按照排序的结果执行，最终只有一个客户端会写入成功，但不能确定是哪个客户端。 所有chunk server的数据都是一致的
  * 实际上目前的dfs实现中，多个客户端并发写同一个文件，通常会先分配一个临时的文件名，然后在写入完成以后由master进行原子MOVE操作，先写入完成的那个任务会成功
* 单个客户端append record时，如果append操作失败，有部分副本chunk server没有写入完成，那么客户端重试时，之前失败的chunk server会填充padding，然后在所有的chunk server上重新append这份record，并返回最后一次append成功时的写入的offset。 所以append操作可能会造成chunk server中的文件副本不一致，但是append操作返回的offet读到的数据一定是一致，因此需要应用层记录每次record的索引（offset），或者对该文件的每次record进行校验。
* 多个客户端并发append record，参加上两条，由于primary chunk server会对操作进行排序后再广播，因此仍然是defined interspersed with inconsistent


### 租约

* master为每个块的某个副本分配一个租约，持有这个租约的副本是primary chunk server，当他故障或失联时，master会分配租约给secondary chunk server。只有持有租约的chunkserver才能发起写操作。
* master为每个chunkserver分配了一个版本号。chunkserver失效后，当master授权新租约时，获得租约的块的版本号会增加，然后通知其他的chunkserver应用该版本号，如果一个chunkserver失联，他上面的块副本的版本号会小于主副本的版本号， 这个块被称作过期副本，会在之后的垃圾回收中移除掉。
