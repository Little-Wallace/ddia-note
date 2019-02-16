# Bigtable

> Bigtable是Google内部的一个分布式结构化数据存储系统，被设计用来存储PB级的数据

### 数据模型

* Bigtable是一个稀疏的多纬度排序Map， Map的索引是行关键字，列关键字，以及时间戳
```json
 (row:string, column:string,time:int64)->string
```
* 

