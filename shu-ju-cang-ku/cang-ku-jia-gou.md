# 仓库架构

ODS

保证数据一致性，完整性，及时性，同步源端数据

DWD

清理，整合，转换事实表

单一事实和多个事实，根据需求开发，一般都会有；

基于大数据需求，DWD表其实有三个表组成：

1. 纯粹事实表组合成的数据事实模型，和纯粹维度表组成的数据维度模型
2. 基于数据事实模型和数据维度模型关联形成的数据事实维度虚拟模型，简化关联和使用；
3. 基于数据事实维度视图模型物化形成的数据事实维度物化模型，加速查询；

DWS

跨主题的DWD关联，方便联合分析；

DWM

基于 DWD DWS 按照特定维度聚合的加速模型；

Data Market

面向需求的数据集市层，利用DWD DWS DWM 再做对接需求的工作

### 数据采集传输

* FLume、Kafka、Sqoop(异构离线数据采集)、DataX(异构离线数据采集)

### 数据存储

* Mysql(business data)
* HDFS(source)
* Hbase(ads)
* Redis(ads)
* Jindofs(ods、dwd)
* MongoDb(爬虫)

### 数据计算

* Hive(默认MR)、Tez、Spark、Flink

### 数据查询

* Presto、Druid、Kylin、Impala



