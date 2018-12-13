#Elasticsearch部署实践

## 1.引言
Elasticsearch(简称ES)是一个基于Lucene的搜索引擎，使用java语言进行开发，RESTful web接口进行交互，不同语言很容易开箱即用。同时ES完整生态中包含了elasticsearch、logstash、kibana(3个组合简称ELK)、Beats、APM等多个组件。ES的简单快捷部署、成熟的生态、稳定的服务为搜索引擎与日志分析带来了极大的便利。在贝壳搜索架构的技术迭代中，也从Solr Cloud迁移到了ES上。

## 2.Elasitcsearch基本概念

在介绍安装与部署之前先了解一下ES的基本概念

* **Cluster(集群)** 
	* 集群是由一个或者多个node组成，进行跨节点的数据存储于查询功能
* **Node(节点)** 
	* 是集群组成部分，每个节点都可提供对集群的查询和搜索功能
	* 每个节点由唯一的名字进行标识，默认情况下使用UUID，node通过集群名称与集群地址加入集群。
	* node可扮演master、data、ingest、tribe等不同的角色，不同的角色可以同时存在。例如一个node节点作为master角色的同时也可以为data角色
* **Index(索引)** 
	* 是文本集合，类似与mysql中的表
	* 存在mapping对索引字段结构、查询分词类型、写入分词类型等属性进行定义
	* 通过settings对索引进行备份、分片、最大查询数目、分词器、refresh等进行设置。
* **Document(文本)** 
	* 是集群中的最基本的存储信息，每个document存在唯一的ID。
	* 每个文本以json格式进行存储。
* **Shard(分片)** 
	* 每个index由多个shard组成，每个shard是一个单独的Lucene索引，用来存储真正的文本数据，每个shard最多存储2,147,483,519( Integer.MAX_VALUE - 128)个document。
	* 查询与写入的时候可通过routing字段进行路由到不同的分片上
* **replica(备份)**
	* 每个shard会存在主分片与备份分片，为保证高可用，同一个shard无论主备分片，不会存储在同一个node节点上

## 3.Elasticsearch部署实践
安装部署一个完整的高可用集群主要有以下几个步骤

* 官方下载、解压
* 修改配置
* 加密传输
* 数据备份
* 集群监控
* 性能测试

### 1.官方下载
[elasticsearch_download]: https://www.elastic.co/downloads/elasticsearch
当前最新版本6.5 [官方下载地址] [elasticsearch_download]，下载后有30天试用版。试用版中支持账号密码、报警、Machine Learning、以及最新**扩集群备份**等特性。试用期结束后，转为基础版，影响较大的是账号密码验证、报警等功能不再支持。

### 2.修改配置
修改配置主要有三部分配置：linux系统配置、elasticsearch.yml配置、jvm.options配置

#### 2.1修改linux配置(需要root权限)
* 修改linux服务器文件句柄打开数目限制。

	通过使用命令```ulimit -n 65536```或者修改``/etc/security/limits.conf``文件，在文件中添加```user - nofile 65536```
来修改操作系统对文件句柄的限制
* 调高线程数目限制。

	ES中使用的线程池为不同的操作，高并发下可能需要创建大量的线程，因此官方建议至少调高到4096。通过修改``/etc/security/limits.conf``文件，在文件中添加``user - nproc 4096``
* 修改虚拟内存限制。

	ES默认使用mmapfs目录来存储其索引。mmap计数的默认操作系统限制可能太低，这可能导致内存不足异常，在 /etc/sysctl.conf文件，添加```vm.max_map_count=262144```，并执行``sysctl -p``使配置生效。
* 禁用文件系统缓存swap。

	因为这可能使得JVM的堆甚至正在执行的文件被交换到磁盘上，通过```sudo swapoff -a```命令进行关闭交互区。或者在 ``/etc/sysctl.conf``文件，添加```vm.swappiness=1```,减少内核交换的倾向，同时仍然允许在紧急条件下使用系统交换。


包含但不限于以上linux配置修改，修改之后，ES即可以成功启动
#### 2.2修改elasticsearch.yml文件
修改位于下载目录中config下的elasticsearch.yml文件，以下配置文件与生产环境配置文件有出入，附带注释，供参考。

```
# ---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
# 集群名称
cluster.name: cluster-test
# ------------------------------------ Node ------------------------------------
# Use a descriptive name for the node:
# 节点名称
node.name: node-name
# 为该节点设置角色，如上介绍，一个节点可同时为多个角色，如果都为false，则为协调节点
node.master: true
node.data: true
node.ingest: false
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
# 数据存储路径
path.data: /home/work/data/elasticsearch/data
#
# Path to log files:
# 日志存储路径，包含执行日志与慢查询等日志，慢查询可通过修改log4j2.properties文件来设置
path.logs: /home/work/data/elasticsearch/logs
#
# ----------------------------------- Memory -----------------------------------
#
# Lock the memory on startup:
#启动的时候是否 锁定内存与弃用系统过滤器
bootstrap.memory_lock: true
bootstrap.system_call_filter: false
#
#
# ---------------------------------- Network -----------------------------------
#
# Set the bind address to a specific IP (IPv4 or IPv6):
#发布地址
network.host: 127.0.0.1
#
# Set a custom port for HTTP:
#
#jmxport:7300
提供的http端口
http.port: 8300
用来不同Node节点之间进行tcp交互的端口
transport:
  tcp:
    port: 8309
    compress: true
    connect_timeout: 30s

http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-headers: Authorization,Content-Type
# 用到docker的时候，因为dockerIP与宿主机IP不同，端口也可能不同，因此需要指定发布地址与端口
http.publish_host: 10.26.9.44
http.publish_port: 8309
#
# For more information, consult the network module documentation.
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when new node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
# 服务发现所配置的node节点
discovery.zen.ping.unicast.hosts: ["192.168.0.1:9201","192.168.0.2:9201","192.168.0.3:9201"]
#℡
# Prevent the "split brain" by configuring the majority of nodes (total number of master-eligible nodes / 2 + 1):
# 至少2个节点存活
discovery.zen.minimum_master_nodes: 2

#ping 选主节点超时
discovery.zen.ping_timeout: 5s
#ping 其它节点的超时时间
discovery.zen.fd.ping_timeout: 30s
discovery.zen.fd.ping_retries: 6
discovery.zen.fd.ping_interval: 10s
#如果没有主节点 阻塞写操作
discovery.zen.no_master_block: write
#
# For more information, consult the zen discovery module documentation.

#增加查询与写入的队列长度
thread_pool:
    search:
        queue_size: 2048
    index:
        queue_size: 1024
# ---------------------------------- Various -----------------------------------
#
# Require explicit names when deleting indices:
#是否可以自动创建索引以及模糊匹配进行删除修改等操作，如果需要打开则设置为true
action.destructive_requires_name: false
+.*,-*意思是可以自动创建以 .开头的索引
action.auto_create_index: +.*,-*

# 最大限制bool查询的条数
indices.query.bool.max_clause_count: 4096

相同shard，不能分布在同一个IP机器上，如果一台机器上部署多个节点，使用该配置增加安全性
cluster:
  routing:
    allocation:
      same_shard.host: true
# ---------------------------------- x-pack ----------------------------------
# x-pack相关配置以及机密连接证书配置
# 打开账号密码认证，6.X之后需要使用加密传输才能进行账号密码认证
# 创建CA证书
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.ssl.verification_mode: certificate
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12
xpack.ml.enabled: false
node.ml: false
xpack.security.audit.enabled: true
xpack.monitoring.exporters.my_local:
  type: local
  use_ingest: false

```
#### 2.3修改文件jvm.options
ES使用java语音开发，那不可避免的就是使用了JVM。该文件在下载目录的config下，包含了ES的JVM配置，介绍以下几个关键参数与调优建议
[oracle_g1]:https://www.oracle.com/technetwork/cn/articles/java/g1gc-1984535-zhs.html
* **-Xms与-Xmx**设置JVM堆大小，建议设置小于32G，并且不要超过服务器内存大小50%。

	如果JVM堆小于32 GB，采用一个内存对象指针压缩技术，这可以大大降低内存的使用：每个指针 4 字节而不是 8 字节。Lucene 能很好利用文件系统的缓存，它是通过系统内核管理的。如果没有足够的文件系统缓存空间，性能会受到影响。 此外，专用于堆的内存越多意味着其他所有使用 doc values 的字段内存越少。
* **-XX:+UseG1GC** 

 	建议使用使用G1收集器。-XX:MaxGCPauseMillis=200、-XX:G1HeapRegionSize=32M、-XX:InitiatingHeapOccupancyPercent=70 等配置；具体参数含义可见[oracle官网][oracle_g1]介绍。对于搜索引擎来说，秒级别的stop the world是不可容忍的，通过配置最长暂停时间设置目标值200ms，来减少回收停顿时间。
* **-XX:+HeapDumpOnOutOfMemoryError** 
	
	当ES发生OOM异常的时候， 可以保存堆内存数据到设置的固定目录，可用来分析ES节点异常原因。

其他一些常规JVM配置，可根据配置文件中介绍修改，不再在这里介绍。
至此，修改完以上介绍的配置之后，可直接启动了。启动后，白金版功能特性有一个月的试用期。

### 3.加密传输
ES从6.0开始，如果需要使用X-pack安全功能，要求SSL/TLS加密传输。通过以下步骤进行加密传入：

* xpack.security.enabled 设置为true。

	刚下载的使用的是试用版的license，则默认的该值为false，需要在elasticsearch.yml配置文件中设置为true。
* 为集群生成安全证书。

	使用``elasticsearch-certutil ca``命令，生成一个默认名称为```elastic-stack-ca.p12```的文件，在生成改文件的时候，会提示输入一个保护该证书的密码。构建集群则需要将该证书拷贝到其他机器上
* 为集群中的每个节点生成私钥、安全证书。

	使用``bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12``命令生成一个默认名称为``elastic-certificates.p12``文件，包含了节点证书、节点秘钥，CA证书。如果想根据每个节点hostname查实，``bin/elasticsearch-certutil cert``可通过使用 --name，--dns，--ip等参数生成不同的证书。如果不使用，则可共用相同证书，直接将该文件拷贝到其他机器上。在生成该文件的时候，同时会输入一个保护该证书的密码。拷贝到其他集群上使用该证书的时候，需要输入该密码
	
* 增加证书相关配置
  ```
  xpack.security.enabled: true   //开启安全认证
  xpack.security.transport.ssl.enabled: true   //使用加密传输
   xpack.ssl.verification_mode: certificate   //只通过证书认证，如果有需要，可更加严格的进行机器认证
   xpack.security.transport.ssl.verification_mode: certificate
   xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12 //指定证书所在路径
   xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12
  ```
  
* 添加证书到秘钥库

使用``bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password``与``bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password``命令，将秘钥添加到ES的秘钥库中

### 4.数据备份

为了增加集群的稳定性与数据安全性，每天对索引数据进行快照备份。现在ES提供了S3、Hadoop、谷歌云等多种备份到共享存储上的插件，由于公司内部大数据集群比较完善，因此采用将快照数据备份到HDFS上。

#### 4.1Hadoop HDFS插件安装
使用``bin/elasticsearch-plugin install repository-hdfs``命令安装该插件

#### 4.2连接Hadoop集群
为每个ES节点安装玩插件之后，需要为节点所在的机器确认能与Hadoop客户端进行连接。如果集群中一个节点在启动的时候，未与Hadoop连接通，则需要将该节点移除，或者重启该节点。公司内部Hadoop 使用了kerberos作为安全认证，需要使用keytab，获取到keytab之后，在config目录下创建repository-hdfs目录，将keytab名称改为krb5.keytab，然后使用keytab验证是否能连接通Hadoop集群

#### 4.3配置仓库
与Hadoop联通后，创建仓库。在创建仓库的时候，应该执行在HDFS下的路径，快照数据存储在该路径下。同时也应该对写入与恢复速度进行限制，防止在进行快照备份与恢复的情况下，将网络IO打满，影响正常业务的使用。

```
PUT _snapshot/hdfs_repository
{
 	"type": "hdfs",
​	"settings": {
​	"uri": "hdfs://nn-cluster",
​	"path": "/user/search/elasticsearch/repositories"
​	"compress":"true",
​	"max_restore_bytes_per_sec":"300mb",
​	"max_snapshot_bytes_per_sec":"100mb",  
​	"security.principal": "search/_HOST@HADOOP.COM",
  }
}
```
#### 4.4将集群上的数据，选择性的备份
在备份集群上数据的时候，可能会进行选择性的备份数据，可通过模糊匹配的方式进行索引的批量选取

```
put _snapshot/hdfs_repository/snapshot_1 
{
​	"indices":"+白名单索引,-黑名单索引"
}
```

#### 4.5从集群上恢复数据
可以选择性的从仓库中恢复索引，如果未指定，则将该快照下的所有索引恢复到当前集群中。

```
POST _snapshot/仓库名称/快照名称/_restore
{
​	"indices":"索引名称"
} 
```
#### 4.6意义
通过每天在业务低峰期将ES集群中的重要索引数据定时备份到HDFS上，可以增加数据的安全性，如果多个集群共用相同的HDFS目录，则可进行跨集群恢复。例如将A集群中的数据快照备份到仓库中，B集群同样可以使用该快照，恢复到B集群中。当一个集群出现故障的时候，可快速在另一个集群中，将数据恢复到备份的时间点。再结合我们引入docker部署ES集群，可分钟级部署一个新集群，使用快照恢复数据。在一个集群完全不可用的情况下，分钟级就可重建一个相同的集群提供服务。

同时如果有数据回溯需求或者开发、测试、线上等不同环境跨集群拷贝索引等需求，也可使用快照进行拷贝。

### 5.集群监控
#### 5.1性能监控
性能监控包含主要的三大资源：磁盘、CPU、内存

**5.1.1通过kibana监控**

安装kibana，安装后，每个Node节点的CPU、内存、QPS、查询队列、segment数目等信息都可在界面中看到。
如下图所示

![Rendering preferences pane](https://ws2.sinaimg.cn/large/006tNbRwly1fxydhdzo6dj31cb0khabu.jpg)

**5.1.2通过grafana监控报警**

kibana虽然对每个节点都有详细的监控，但是无法在一个界面中展现出整个集群的状态，而且kibana的报警机制并不完善。通过grafana读取ES中内置索引``.monitoring-es-6-*``数据，索引中包含了对整个集群的监控信息。可对整个集群甚至多个集群的状态进行监控与报警。在grafana中配置阈值，超过阈值时候讲报警信息发送到指定API进行微信、短信、邮件等报警处理。
![](https://ws3.sinaimg.cn/large/006tNbRwly1fxydpjxdj0j31dv0nzwih.jpg)
#### 5.2慢查询监控
对搜索引擎来说，查询所花费的时间是至关重要的，因此对于每个索引可设置慢查询阈值，目前线上设置的为100ms，超过100ms时候会打印出所有的慢查询请求，随着集群cluster数目与节点node数目增多，对于慢查询的监控也是非常有必要的。通过使用rsyslog+kafka+logstash+Elasticsearch+Grafana对慢查询日志进行统计收集后进行监控报警
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxydz4jycmj31e90hf40v.jpg)

至此，一个完整的高可用集群搭建完毕！！！


### 6.性能测试
集群搭建好，选择多少Node节点合适呢，不同的Node扮演什么角色呢。通过对搭建好的ES集群进行压力测试，来根据业务规模，预估ES集群。因为ES检索会存在缓存，所以不适合使用apacheAB进行压力测试，需要模拟线上真实请求进行测试。以下压测为线上一个集群部署服务之前进行的测试，与线上使用的多个集群中可能存在出入供参考：
#### 6.1集群规模
**ES集群：** 3台master+6台data 

**index索引：** 500万左右Document，1个shard，5个replica

**客户端：** 使用6台直接连接ES的搜索客户端
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxygfl7kvcj30su0k8mxq.jpg)

#### 6.2压力测试
复制贝壳找房线上搜索请求，结论如下：

| 请求总数 | 408错误| 错误率 |QPS |响应时间 |
|:------- |:----:| ---------:|----:|----:|
| 1200000次| 60条 |   0.005% |5402.36 |18.21ms |

成功率统计
![](https://ws4.sinaimg.cn/large/006tNbRwly1fxygkga1fwj32jk0rg0vg.jpg)
速率监控
![](https://ws3.sinaimg.cn/large/006tNbRwly1fxygllhhcsj32o00nc7cm.jpg)

####6.3小结
对集群进行更高压力测试，超时错误增多，响应时间也同比上升。为保证搜索时效性，并且20ms以内将数据返回与集群的稳定性，因此选择了资源与效果的权衡，使用当前3+6节点提供5000QPS线上搜索。当然不同的集群也要根据应用场景的不同，进行集群规模与参数的一些调整，例如日志集群多用于写可适当调整写入队列等参数。

[sniffer_link]:https://www.elastic.co/guide/en/elasticsearch/client/java-rest/master/sniffer.html
在线上使用的时候，客户端可以使用[sniffer][sniffer_link]与集群中每个节点获得连接，采用轮询的方式与集群进行数据交互，既可以防止配置中只连接的某几个节点而导致压力过大，又可以动态的添加节点后直接与新增节点获得连接。

## 4.高可用集群架构
![](https://ws2.sinaimg.cn/large/006tNbRwly1fy50acfm3lj30hc0a4mxe.jpg)
### 4.1跨机房部署
为了保证数据的安全性与集群容错性，采用冷备跨集群部署。通过index模块，双写到不同机房的ES集群，保持主备集群都可用，对外提供搜索服务，正常情况下，同一机房读取模块使用当前机房的集群提供服务，一旦当前机房网络出现问题，则降级切换到远程其他机房。有跨机房备份之后，大大提高了集群的安全性与可用性。
### 4.2业务隔离
虽然一个ES集群能够很容的动态扩展，但是大量业务使用同一个集群带来的风险是比较高的，因此建议采用多集群部署，进行业务隔离，避免业务之间的相互干扰，提高稳定性。

### 4.3容器化部署
使用Docker容器化部署ES节点，既可充分利用机器资源，也进行CPU、内存、磁盘、网络等资源隔离，避免不同集群之间相互干扰。

## 5.总结
elasticsearch是一个快速迭代的开源组件，迭代更新非常迅速，很方便用于搜索引擎和日志分析。本文主要介绍了一个高可用集群完整的安装部署实践，在使用的过程中如何对集群进行配置、监控、测试、数据备份，以及最后介绍我们的高可用架构，使用docker容器支持多个机房进行主备，K8s编排快速创建集群。希望本文对于计划使用Elasticsearch的维护着可以带来一些指导。