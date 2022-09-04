# dremio

知识储备:

数据湖    
运维    

## 数据湖: 
- 价值:    
数据湖的一部分价值是把不同种类的数据汇聚到一起;
另一部分价值是不需要预定义的模型就能进行数据分析。可提供更便捷高效的实时分析能力    
现在的大数据架构是可扩展的，并且可以为用户提供越来越多的实时分析.

- 问题

数据湖架构面向多数据源的信息存储，包括物联网在内.
1 数据湖的数据持久性.要求数据长期存储,这需要保证数据具有长期时效价值以抵消容量成本    
2 安全却是需要优先考虑的因素.将所有鸡蛋放到一个篮子里,如何保证每个数据源都安全可靠.

## 使用

### 下载

安装包下载地址: https://download.dremio.com/

安装依赖
- 4.0后不支持mac和windows.linux版本:     
RHEL and CentOS 6.7+, 7.3+, and 8.3 (RPM and tarball)     
SLES 12 SP2+ (tarball)     
Ubuntu 14.04+ (tarball)     
Debian 7+ (tarball)     

- jdk1.8    
-  Dremio UI要求浏览器版本:    
> Google Chrome 54+
Apple Safari 11+
Mozilla Firefox 50+
Microsoft Edge 14+

## 集群部署

dremio在设计上支持1000+个节点分布运行.也支持单节点本地运行.    
dremio也是一个可以部署在公有云和私有服务的分布式系统.dremio集群可以和数据源部署在一起也可以分离部署.

### 部署架构

集群部署架构:    
![architecutre](pic/architecutre.png)

查询(Queries): dremio REST/UI或Dremio OJBC/JDBC的发出        
协调者节点(Coordinator node): 主协调者(master-coordinator)角色可以有多个.但dremio集群推荐只有一个协调者节点    
执行者节点(Executor nodes): 可以配置多个执行者(executor)角色.实际数量看你的负载.如:当你有大数据量查询或者大量查询请求时可以增加执行器节点数    
元数据存储(Metadata storage): 一般存储在协调者节点本地.默认: 不要求配置.如果配置了高可用,元数据存储必须作为外部存储配置.    
Zookeeper: 可以嵌入在协调者节点也可以使用外部服务.默认:嵌入.如果配置了高可用,必须配置外部服务.    
分布式存储(Distributed Store): 本地安装并在所欲dremio节点配置.    

#### dremio services

Dremio "services"参数用于指定一个节点是否是主协调者,副协调者或执行者角色.    

dremio集群组成:    
> 一个或多个协调者节点
> 一个或多个执行器节点

- 主协调者角色(Master Coordinator Role)

主协调者角色用于专门管理元数据.其他职责:    
查询计划    
提供UI服务    
处理客户端连接,包括rest API

高可用实现中可以设置多个协调者节点.如果协调者节点故障,则其他节点将作为备用节点.    

- 副协调者角色(Secondary Coordinator Role)

副协调者节点用于增加并发和分配ODBC and JDBC客户端请求的查询计划到执行节点

- 执行者角色(Executor Role)

执行者节点用于执行查询

- 单节点和集群部署

单节点部署: 执行和协调都在一个节点中.单节点多角色    
集群部署: 每个节点一个角色,协调者或执行者.集群部署不支持单节点多角色.    


#### 元数据存储

dremio存储的元数据包括用户,空间,和数据集(users, spaces, and datasets).默认,这些元数据存在"${DREMIO_HOME}/data"目录.    
管理员可以在dremio.conf配置paths.local自定义元数据存储地址.    

- I/O性能

Dremio元数据服务于两种工作负载类型:    
| 工作负载类型 | 性能影响因素 |
| ---- | ---- |
| 用户查询请求和数据反射刷新 | 性能受查询并发数影响 |
| Dremio收集和记录源数据集(source datasets)信息的元数据刷新 | 性能受dremio连接的物理数据集数量和刷新频率影响.为满足查询工作负载和元数据刷新策略可能要去更大的吞吐量 |

- 用户查询请求和数据反射刷新

用户查询请求和数据反射刷新的性能要求与每秒并发查询数成线性相关

| 查询/秒 | 要求基线吞吐量 |
| ---- |---- |
| 50 | 60MB/s |
| 100 | 120MB/s |
| 200 | 240MB/s |

- 元数据刷新

元数据刷新的性能要求与数据集数量和每个数据集的列平均数和拆分数成线性比例.性能要求与刷新间隔成反比.

| 数据集数 | 数据集列平均值,拆分数 | 要求基线吞吐量 | 
| 1000 | 每数据集20列,1000片,刷新间隔30分钟 | 1MB/s |
| 2000 | 每数据集20列,1000片,刷新间隔30分钟 | 2MB/s |
| 2000 | 每数据集20列,1000片,刷新间隔10分钟 | 3MB/s |

#### 分布式存储

分布式存储缓存位置包含加速器(accelerator),表(tables),任务结果(job results),下载和上传数据;    
在dremio.conf文件中通过paths.dist参数指定缓存位置.集群节点间该值必须一致.这意味着如果使用本地存储或NAS(网络附加存储,一种使用NFS等协议进行网络提供文件的设备),所配置的路径必须存在或者可被所有节点访问.如果该值更新,则必须更新集群中所有的节点中的值.默认使用dremio本地磁盘存储.    

支持的存储类型: NAS,HDFS,MapR-FS,Amazon S3,Azure Data Lake Store (ADLS Gen1),Azure Storage (ADLS Gen2)

#### 协调器系统级缓存

每个协调器存储一个系统选项缓存,该值协调者间共享.你可以在任一协调者节点修改系统选项,该值会同步到其他协调者节点.每个协调者通过后台任务每60秒获取一次系统选项,它涵盖了系统选项缓存在协调器之间变得不一致的情况.情况包括:    
> 像删除通知等网络问题
> 跨多协调者间的系统选项快速更新.dremio不建议频繁改变系统选项.



### 部署环境

- 配置内容

服务高可用
缓存
存储
网络,带宽,交互端口
日志
磁盘,内存容量分配

当前生产配置:
内存 总量125G
磁盘 600G/+1.5T/data

dremio-env
```
DREMIO_MAX_MEMORY_SIZE_MB=102400M
DREMIO_MAX_HEAP_MEMORY_SIZE_MB=32768M
DREMIO_JAVA_SERVER_EXTRA_OPTS="-Dsaffron.default.charset=UTF-16LE -Dsaffron.default.nationalcharset=UTF-16LE -Dsaffron.default.nationalcharset=UTF-16LE"
```
DREMIO_JAVA_SERVER_EXTRA_OPTS编码配置有重复


dremio.conf master
```
paths: {
  # the local path for dremio to store data.
  local: "/data/dremio/data"

  # the distributed path Dremio data including job results, downloads, uploads, etc
  dist: "pdfs://"${paths.local}"/pdfs"
}

services: {
  coordinator.enabled: true,
  coordinator.master.enabled: true,
  executor.enabled: true
}
registration.publish-host: "10.10.4.72"
zookeeper: "10.16.8.99:2181,10.16.8.100:2181,10.16.8.105:2181"
```


zookeeper 3台
- 高可用要求

配置dremio服务,外部元数据存储,外部zookeeper集群    
启动协调者节点     
启动执行者节点    

### 使用

dataset: 分为物理数据集和虚拟数据集.物理数据集是将文件转化为数据表,支持外部查询操作;虚拟数据集是基于一个或多个物理数据集建立的查询表,类似于视图,它是对所有人可见的,通过space进行分类.
space是共享空间,建立space的数据表用于用户间数据共享.

### job profile

The following table describes each job state:

| Job State	Description | PENDING	Wait to be scheduled by the command pool, the pool of threads available to the coordinator. |
| ---- | ---- |
| METADATA_RETRIEVAL | Parse the SQL command, check permissions, and retrieve schema information from the KVStore. |
| PLANNING | Physical and logical planning, Match data reflections and substituions, prune partitions, and map the query to a queue using WLM rules. |
| ENGINE_START | Wait for engine start. Currently applies only to AWS Edition deployments. |
| QUEUED | Each queue has a limited number of concurrent jobs. Queries wait for an in-progress job to complete when the number of in-progress jobs is greater than the limit. |
| EXECUTION PLANNING | Select executor nodes to run the query, retrieve split metadata from the KVStore for the pruned partitions, parallelize the fragments, assign fragments among the executor nodes, and assign splits to fragments taking into account data locality. |
| STARTING | Send RPCs to each executor that contains information about the fragments assigned to it. |
| RUNNING | Wait for executor nodes to execute and complete the fragments assigned to them. Typically, queries spend most of their time in this job state. |
| COMPLETED | Query successfully completed. |
| CANCELLED | Query was cancelled either by a user or an internal issue, such as insufficient memory or heap. |
| FAILED | Query failed due to an error. |

### 分析profiles

- Considerations
A major consideration is whether a thread is in a running, blocked and waiting for data (or to send data), or sleeping state. A thread is usually sleeping if the thread is ready to run but another thread is currently using the CPU.

- Blocked State
A thread is usually in a blocked state for one of the following reasons:

It’s waiting on some data from another thread (a child phase in the tree).
It’s trying to send data to another thread (a parent phase) but the receiving thread isn’t responding. The receiving thread may be too slow or overwhelmed with work. What occurs is that the sending thread is forced to block until the receiver is able to receive the data.

- Network Activities
Network activities are tracked as part of the blocked metric.

> If a thread is blocked, nothing much can be done.
If a thread is sleeping, ensure that task.on_idle_load_shed is set to true.

- Troubleshooting
The following general areas should be reviewed:

> Errors
Performance

- Errors
To troubleshoot errors, consider the following:

> Dremio nodes – Determine whether the coordinator or executor nodes are impacted.
Verbose error
Out of memory
Incorrect results
No results

- Performance
To troubleshoot performance, consider the following:

> Planning time versus execution time
Number of threads which impact process time
Row count versus rows
Blocked versus sleep
Setup time version wait time
Operator metrics (contact support@dremio.com)

- Downloaded Profile Data
After downloading your jobs profile data, the following files provide valuable information.

> header.json – This file provides the full list of Dremio coordinators and executors, data sets, and sources. This information is useful when you are using REST calls.
profile_attempt_0.json – This file helps with troubleshooting out of memory and wrong results issues. Note that the start and end time of query is provided in EPOCH format. See the Epoch Converter utility for converting query time.

### 反射

Data Reflections are maintained in a high-performance columnar representation based on Apache Parquet and Apache Arrow, utilizing advanced compression techniques such as dictionary encoding, run-length encoding, and delta encoding.

系统反射表:select * from sys.reflections

- dremio查询核心技术: Apache Arrow Flight

Apache Arrow Flight is a general-purpose, client-server framework for simplifying high-performance transportation of large datasets over network interfaces.

Apache Arrow Flight SQL is a new API developed by the Apache Arrow community for interacting with SQL databases. It provides Arrow Flight a way to execute queries, create prepared statements, and fetch metadata about the supported SQL dialect, available types, defined tables.

Both are part of Apache Arrow, an open-source software development platform for building high-performance applications that process and transport large data sets. A critical component of Apache Arrow is its in-memory columnar format, a standardized, language-agnostic specification for representing structured, table-like datasets in-memory.

apache arrow filght: https://arrow.apache.org/docs/format/FlightSql.html#    
java客户端: https://github.com/apache/arrow/blob/dfca6a704ad7e8e87e1c8c3d0224ba13b25786ea/java/flight/flight-sql/src/main/java/org/apache/arrow/flight/sql/FlightSqlClient.java



### 扩展阅读

OpenId 实现SSO




### 问题

只有一个主协调者节点可对外服务,怎么实现负载均衡??
72master节点同时配置执行者角色,与官方推荐配置不符合?    



https://docs.dremio.com/software/rest-api/catalog/post-catalog-id/


