---
title: "v0.2"
linkTitle: "v0.2"
weight: 4
---

### 1 测试环境

#### 1.1 软硬件信息

起压和被压机器配置相同，基本参数如下：

| CPU                                          | Memory | 网卡       |
|----------------------------------------------|--------|----------|
| 24 Intel(R) Xeon(R) CPU E5-2620 v2 @ 2.10GHz | 61G    | 1000Mbps |

测试工具：apache-Jmeter-2.5.1

#### 1.2 服务配置

- HugeGraph版本：0.2
- 后端存储：使用服务内嵌的cassandra-3.10，单点部署；
- 后端配置修改：修改了cassandra.yaml文件中的以下两个属性，其余选项均保持默认

```
  batch_size_warn_threshold_in_kb: 1000
  batch_size_fail_threshold_in_kb: 1000
```

- HugeGraphServer 与 HugeGremlinServer 与cassandra都在同一机器上启动，server 相关的配置文件除主机和端口有修改外，其余均保持默认。

#### 1.3 名词解释

- Samples -- 本次场景中一共完成了多少个线程
- Average -- 平均响应时间
- Median -- 统计意义上面的响应时间的中值
- 90% Line -- 所有线程中90%的线程的响应时间都小于xx
- Min -- 最小响应时间
- Max -- 最大响应时间
- Error -- 出错率
- Throughput -- 吞吐量Â
- KB/sec -- 以流量做衡量的吞吐量

_注：时间的单位均为ms_

### 2 测试结果

#### 2.1 schema

| Label         | Samples | Average | Median | 90%Line | Min | Max | Error% | Throughput | KB/sec |
|---------------|---------|---------|--------|---------|-----|-----|--------|------------|--------|
| property_keys | 331000  | 1       | 1      | 2       | 0   | 172 | 0.00%  | 920.7/sec  | 178.1  |
| vertex_labels | 331000  | 1       | 2      | 2       | 1   | 126 | 0.00%  | 920.7/sec  | 193.4  |
| edge_labels   | 331000  | 2       | 2      | 3       | 1   | 158 | 0.00%  | 920.7/sec  | 242.8  |

结论：schema的接口，在1000并发持续5分钟的压力下，平均响应时间1-2ms，无压力

#### 2.2 single 插入

##### 2.2.1 插入速率测试

###### 压力参数

测试方法：固定并发量，测试server和后端的处理速率

- 并发量：1000
- 持续时间：5min

###### 性能指标

| Label                  | Samples | Average | Median | 90%Line | Min | Max | Error% | Throughput | KB/sec |
|------------------------|---------|---------|--------|---------|-----|-----|--------|------------|--------|
| single_insert_vertices | 331000  | 0       | 1      | 1       | 0   | 21  | 0.00%  | 920.7/sec  | 234.4  |
| single_insert_edges    | 331000  | 2       | 2      | 3       | 1   | 53  | 0.00%  | 920.7/sec  | 309.1  |

###### 结论

- 顶点：平均响应时间1ms，每个请求插入一条数据，平均每秒处理920个请求，则每秒平均总共处理的数据为1*920约等于920条数据；
- 边：平均响应时间1ms，每个请求插入一条数据，平均每秒处理920个请求，则每秒平均总共处理的数据为1*920约等于920条数据；

##### 2.2.2 压力上限测试

测试方法：不断提升并发量，测试server仍能正常提供服务的压力上限

###### 压力参数

- 持续时间：5min
- 服务异常标志：错误率大于0.00%

###### 性能指标

| Concurrency  | Samples | Average | Median | 90%Line | Min | Max  | Error% | Throughput | KB/sec |
|--------------|---------|---------|--------|---------|-----|------|--------|------------|--------|
| 2000(vertex) | 661916  | 1       | 1      | 1       | 0   | 3012 | 0.00%  | 1842.9/sec | 469.1  |
| 4000(vertex) | 1316124 | 13      | 1      | 14      | 0   | 9023 | 0.00%  | 3673.1/sec | 935.0  |
| 5000(vertex) | 1468121 | 1010    | 1135   | 1227    | 0   | 9223 | 0.06%  | 4095.6/sec | 1046.0 |
| 7000(vertex) | 1378454 | 1617    | 1708   | 1886    | 0   | 9361 | 0.08%  | 3860.3/sec | 987.1  |
| 2000(edge)   | 629399  | 953     | 1043   | 1113    | 1   | 9001 | 0.00%  | 1750.3/sec | 587.6  |
| 3000(edge)   | 648364  | 2258    | 2404   | 2500    | 2   | 9001 | 0.00%  | 1810.7/sec | 607.9  |
| 4000(edge)   | 649904  | 1992    | 2112   | 2211    | 1   | 9001 | 0.06%  | 1812.5/sec | 608.5  |

###### 结论

- 顶点：
  - 4000并发：正常，无错误率，平均耗时13ms；
  - 5000并发：每秒处理5000个数据的插入，就会存在0.06%的错误，应该已经处理不了了，顶峰应该在4000
- 边：
  - 1000并发：响应时间2ms，跟2000并发的响应时间相差较多，主要是 IO network rec和send以及CPU几乎增加了一倍）；
  - 2000并发：每秒处理2000个数据的插入，平均耗时953ms，平均每秒处理1750个请求；
  - 3000并发：每秒处理3000个数据的插入，平均耗时2258ms，平均每秒处理1810个请求；
  - 4000并发：每秒处理4000个数据的插入，平均每秒处理1812个请求；

#### 2.3 batch 插入

##### 2.3.1 插入速率测试

###### 压力参数

测试方法：固定并发量，测试server和后端的处理速率

- 并发量：1000
- 持续时间：5min

###### 性能指标

| Label                 | Samples | Average | Median | 90%Line | Min | Max   | Error% | Throughput | KB/sec |
|-----------------------|---------|---------|--------|---------|-----|-------|--------|------------|--------|
| batch_insert_vertices | 37162   | 8959    | 9595   | 9704    | 17  | 9852  | 0.00%  | 103.4/sec  | 393.3  |
| batch_insert_edges    | 10800   | 31849   | 34544  | 35132   | 435 | 35747 | 0.00%  | 28.8/sec   | 814.9  |

###### 结论

- 顶点：平均响应时间为8959ms，处理时间过长。每个请求插入199条数据，平均每秒处理103个请求，则每秒平均总共处理的数据为199*131约等于2w条数据；
- 边：平均响应时间31849ms，处理时间过长。每个请求插入499个数据，平均每秒处理28个请求，则每秒平均总共处理的数据为28*499约等于13900条数据；
