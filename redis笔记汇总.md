# 基本命令



绑定服务器要使用的配置文件，且==启动服务器==

`redis-server /myredis/redis.conf`



连接服务

`redis-cli -a 111111 -p 6379`(-a 后面是redis设置的密码，6379是默认端口号)



启动之后验证是否正确启动

`ping  返回pong代表启动成功`



退出连接

`quit`(并没有关闭服务器，只是退出连接的客户端)

`SHUTDOWN`（关闭服务器） quit(关闭服务器后退出连接)

关闭方法2

`redis-cli -a 111111 shutdown`



卸载

1.停止redis-server服务

![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/5.停止redis-server服务.png)

2.删除/usr/local/bin目录下与redis相关的文件

ls -l /usr/local/bin/redis-*

rm -rf /usr/local/bin/redis-*

![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/6.删除redis文件.png)







# redis持久化

RDB(快照) AOF(记录执行过的指令)

RDB

- save
  - 在主程序中执行会**阻塞**当前redis服务器，直到持久化工作完成执行save命令期间，Redis不能处理其他命令，**线上禁止使用**
- bgsave
  - ==redis会在后台异步进行快照操作==，**不阻塞**快照同时还可以相应客户端请求，该触发方式会fork一个子进程由子进程复制持久化过程

RDB优势

- 适合大规模的数据恢复
- 按照业务定时备份
- 对数据完整性和一致性要求不高
- RDB文件在内存中的加载速度要比AOF快很多

劣势

- 在一定间隔时间做一次备份，所以如果redis意外down掉的话，就会丢失从当前至最近一次快照期间的数据，**快照之间的数据会丢失**
- 内存数据的全量同步，如果数据量太大会导致IO严重影响服务器性能
- RDB依赖于主进程的fork，在更大的数据集中，这可能会导致服务请求的瞬间延迟。fork的时候内存中的数据被克隆了一份，大致2倍的膨胀性，需要考虑



### AOF持久化工作流程

![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/26.AOF%E6%8C%81%E4%B9%85%E5%8C%96%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.jpg)

1.Client作为命令的来源，会有多个源头以及源源不断的请求命令。

2.在这些命令到达Redis Server 以后并不是直接写入AOF文件，会将其这些命令先放入AOF缓存中进行保存。==这里的AOF缓冲区实际上是内存中的一片区域，存在的目的是当这些命令达到一定量以后再写入磁盘，避免频繁的磁盘IO操作==。



写回策略

![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/28.AOF%E4%B8%89%E7%A7%8D%E5%86%99%E5%9B%9E%E7%AD%96%E7%95%A5.jpg)



混合持久化

![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/56.%E6%B7%B7%E5%90%88%E6%8C%81%E4%B9%85%E5%8C%96.jpg)





# 事物



### Redis事务 VS 数据库事务

| 1.单独的隔离操作     | Redis的事务仅仅是保证事务里的操作会被连续独占的执行，redis命令执行是单线程架构，在执行完事务内所有指令前是不可能再去同时执行其他客户端的请求的 |
| -------------------- | ------------------------------------------------------------ |
| 2.没有隔离级别的概念 | 因为事务提交前任何指令都不会被实际执行，也就不存在”事务内的查询要看到事务里的更新，在事务外查询不能看到”这种问题了 |
| 3.不保证原子性       | Redis的事务不保证原子性，也就是不保证所有指令同时成功或同时失败，只有决定是否开始执行全部指令的能力，没有执行到一半进行回滚的能力 |
| 4.排它性             | Redis会保证一个事务内的命令依次执行，而不会被其它命令插入    |





### 常用命令

![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/2.Redis%E4%BA%8B%E5%8A%A1%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4.jpg)





# 管道

Redis是一种基于**客户端-服务端模型**以及请求/响应协议的TCP服务。一个请求会遵循以下步骤:
1客户端向服务端发送命令分四步(发送命令→命令排队→命令执行-返回结果)，并监听Socket返回，通常以阻塞模式等待服务端响应。
2服务端处理命令，并将结果返回给客户端。
==上述两步称为: Round Trip Time(简称RTT,数据包往返于两端的时间)。== 

![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/1.Redis%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%B8%8E%E6%9C%8D%E5%8A%A1%E7%AB%AF%E4%BA%A4%E4%BA%92%E6%A8%A1%E5%9E%8B.jpg)



加入管道，减少RTT



![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/2.Redis%20pipeline%E4%BA%A4%E4%BA%92%E6%A8%A1%E5%9E%8B.jpg)



pipeline与原生批量命令对比

1. 原生批量命令是原子性(例如：mset、mget)，$\textcolor{red}{pipeline是非原子性的}$
2. 原生批量命令一次只能执行一种命令，pipeline支持批量执行不同命令
3. 原生批量命令是服务端实现，而pipeline需要服务端与客户端共同完成







# Redis复制介绍

### 首次连接，全量复制

- master节点收到sync命令后会开始在后台保存快照(即RDB持久化，==主从复制时会触发RDB==)，同时收集所有接收到的用于修改数据集的命令并缓存起来，master节点执行RDB持久化完后，master将RDB快照文件和所有缓存的命令发送到所有slave，以完成一次完全同步
- 而slave服务在接收到数据库文件数据后，将其存盘并加载到内存中，从而完成复制初始化

### 心跳持续，保持通信

- repl-ping-replica-period 10

  ![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/25.%E5%BF%83%E8%B7%B3.png)

### 进入平稳，增量复制

- master继续将新的所有收集到的修改命令自动依次传送给slave，完成同步



### 从机下线，重连续传

- master会检查backlog里面的offset，master和slave都会保存一个复制的offset还有一个masterId，offset是保存在backlog中的。$\textcolor{red}{master只会把已经缓存的offset后面的数据复制给slave，类似断点续传}$



- master挂了怎么办？

  默认情况下，不会在slave节点中自动选一个master

  那每次都要人工干预？ -> $\textcolor{red}{无人值守变成刚需}$





# Redis 哨兵



![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/1.%E5%93%A8%E5%85%B5%E4%BD%9C%E7%94%A8.png)



作用

**主从监控**：监控主从redis库运行是否正常

**消息通知**：哨兵可以将故障转移的结果发送给客户端

**故障转移**：如果master异常，则会进行主从切换，将其中一个slave作为新master

**配置中心**：客户端通过连接哨兵来获得当前Redis服务的主节点地址



==监控==

SDown==主观下线==(Subjectively Down)

1. SDOWN（主观不可用）是**单个sentinel自己主观上**检测到的关于master的状态，从sentinel的角度来看，如果发送了PING心跳后，在一定时间内没有收到合法的回复，就达到了SDOWN的条件。

ODown==客观下线==(Objectively Down)

1. ODOWN需要一定数量的sentinel，$\textcolor{red}{多个哨兵达成一致意见}$才能认为一个master客观上已经宕机







选举出领导者哨兵(哨兵中选出兵王)

- 当主节点被判断客观下线后，各个哨兵节点会进行协商，先选举出一个$\textcolor{red}{\large 领导者哨兵节点（兵王）}$并由该领导者也即被选举出的兵王进行failover（故障转移）。
- 哨兵领导者，兵王如何选出来的？-> Raft算法
  - Raft 先到先得原则
  - 选兵王 让兵王选master

![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/23.Raft%E7%AE%97%E6%B3%95.jpg)

监视该主节点的所有哨兵都有可能被选为领导者，选举使用的算法是Raft算法;Raft算法的基本思路是==先到先得==:==即在一轮选举中，哨兵A向B发送成为领导者的申请、如果B没有同意过其他哨兵，则会同意A成为领导者==。



兵王选举master

- 选取规则
  - priority（权限) 【权限越高成为master】
  - replication offset（位置偏移量）【选取数据较多的slave称为master】
  -  最小Run ID 【最小的runid称为master】

![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/24.%E6%96%B0master%E9%80%89%E4%B8%BE.jpg)





群臣俯首

- $\textcolor{red}{\large 一朝天子一朝臣，换个码头重新拜}$

- 别的Sentinel拜新的master

旧主拜服

- $\textcolor{red}{\large 老master回来也认怂，会被降级为slave}$



### 哨兵使用建议

1. 哨兵节点的数量应为多个，哨兵本身应该集群，保证高可用
2. 哨兵节点的数量应该是奇数
3. 各个哨兵节点的配置应一致
4. 如果哨兵节点部署在Docker等容器里面，尤其要注意端口的正确映射
5. 哨兵集群+主从复制，并不能==保证数据零丢失==，$\textcolor{red}{\large所以需要使用集群}$
   1. master挂了之后此时不能立刻把数据写入，等到重新选举恢复里面肯定有时间差，如果在这个时间差里面写入数据是无法写入的，此时会造成数据丢失









# 集群



![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/2.redis%E9%9B%86%E7%BE%A4%E5%9B%BE.jpg)

一句话：Redis集群是一个提供在多个Redis节点间共享数据的程序集，Redis集群可以支持多个master



### 能干嘛

- Redis集群支持多个master，每个master又可以挂载多个slave
  1. 读写分离
  2. 支持数据的高可用
  3. 支持海量数据的读写存储操作
- 由于Cluster自带Sentinel的故障转移机制，内置了高可用的支持，$\textcolor{red}{\large 无需再去使用哨兵功能}$
- 客户端与Redis的节点连接，不再需要连接集群中所有的节点，只需要任意连接集群中的一个可用节点即可
- $\textcolor{red}{\large 槽位slot}$负责分配到各个物理服务节点，由对应的集群来负责维护节点、插槽和数据之间的关系





### redis集群的槽位slot

==Redis集群有16384个哈希槽==

- 每个哈希槽都可以配置一个redis服务器，但是官网推荐使用集群时<= 1000个redis服务器，此时一个redis服务器就对应了多个哈希槽
- 每个槽可以存放多个key





每个key通过CRC16校验后对==16384==取模来决定放置哪个槽，集群的每个节点负责一部分hash槽，举个例子，比如当前集群有3个节点，那么：

![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/5.%E6%A7%BD%E4%BD%8D%E7%A4%BA%E4%BE%8B.jpg)



### redis集群的分片

| 分片是什么            | 使用Redis集群时我们会将存储的数据分散到多台redis机器上，这==称为分片==。简言之，集群中的每个Redis实例都被认为是整个数据的一个分片。 |
| --------------------- | ------------------------------------------------------------ |
| 如何找到给定key的分片 | 为了找到给定key的分片，我们对key进行CRC16(key)算法处理并通过对总分片数量取模。然后，$\textcolor{red}{\large使用确定性哈希函数}$，这意味着给定的key$\textcolor{red}{\large将多次始终映射到同一个分片}$，我们可以推断将来读取特定key的位置。 |





### redis集群的分片

| 分片是什么            | 使用Redis集群时我们会将存储的数据分散到多台redis机器上，这==称为分片==。简言之，集群中的每个Redis实例都被认为是整个数据的一个分片。 |
| --------------------- | ------------------------------------------------------------ |
| 如何找到给定key的分片 | 为了找到给定key的分片，我们对key进行CRC16(key)算法处理并通过对总分片数量取模。然后，$\textcolor{red}{\large使用确定性哈希函数}$，这意味着给定的key$\textcolor{red}{\large将多次始终映射到同一个分片}$，我们可以推断将来读取特定key的位置。 |

 ### 分片和槽位的优势

$\textcolor{blue}{\large 最大优势，方便扩缩容和数据分派查找}$

这种结构很容易添加或者删除节点，比如如果我想==添加个节点D==，==我需要从节点A，B，C中得部分槽位到D上==。如果我想==移除节点A==，需要将A中的槽移动到B和C节点上，然后将没有任何槽的节点从集群中移除即可。由于一个结点将哈希槽移动到另一个节点不会停止服务，所以无论添加删除或者改变某个节点的哈希槽的数量都不会造成集群不可用的状态。





### slot槽位映射，一般业界有三种解决方案



1. 哈希取余分区(小厂)

![6.哈希取余分区.jpg](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/6.%E5%93%88%E5%B8%8C%E5%8F%96%E4%BD%99%E5%88%86%E5%8C%BA.jpg)



$\textcolor{blue}{\large 优点}$：简单粗暴，直接有效，只需要预估好数据规划好节点，例如3台、8台、10台，就能保证一段时间的数据 支撑。使用Hash算法让固定的一部分请求落到同一台服务器上，这样每台服务器固定处理一部分请求 (并维护这些请求的信息)， 起到负载均衡+分而治之的作用。

$\textcolor{blue}{\large 缺点}$：原来规划好的节点，进行扩容或者缩容就比较麻烦了额，不管扩缩，每次数据变动导致节点有变动，映射关系需要重新进行计算，在服务器个数固定不变时没有问题，如果需要弹性扩容或故障停机的情况下，原来的取模公式就会发生变化: Hash(key)/3会 变成Hash(key) /?。此时地址经过取余运算的结果将发生很大变化，根据公式获取的服务器也会变得不可控。





2. 一致性哈希算法分区(中厂)

$\textcolor{blue}{\large 算法构建一致性哈希环}$ 



![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/7.Hash%E7%8E%AF.jpg)

$\textcolor{blue}{\large 服务器IP节点映射}$ 

将集群中各个IP节点映射到环上的某一个位置。
将各个服务器使用Hash进行一个哈希，具体可以选择服务器的IP或主机名作为关键字进行哈希，这样每台机器就能确定其在哈希环上的位置。假如4个节点NodeA、B、C、D，经过IP地址的**哈希函数**计算(hash(ip))，使用IP地址哈希后在环空间的位置如下:



![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/8.%E5%AF%B9%E8%8A%82%E7%82%B9%E5%8F%96Hash%E5%80%BC.jpg)

$\textcolor{blue}{\large key落到服务器的落键规则}$ 

当我们需要存储一个kv键值对时，首先计算key的hash值，hash(key)，将这个key使用相同的函数Hash计算出哈希值并确定此数据在环上的位置，**从此位置沿环顺时针“行走”**，第一台遇到的服务器就是其应该定位到的服务器，并将该键值对存储在该节点上。





![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/10.%E4%B8%80%E8%87%B4%E6%80%A7%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E5%AE%B9%E9%94%99%E6%80%A7.jpg)

$\textcolor{green}{\large 一致性哈希算法的扩展性}$ 

- 数据量增加了，需要增加一台节点NodeX，X的位置在A和B之间，那收到影响的也就是A到X之间的数据，重新把A到X的数据录入到X上即可，不会导致hash取余全部数据重新洗牌。

  


![image-20240404102254890](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404102254890.png)



缺点

$\textcolor{green}{\large 一致性哈希算法的数据倾斜问题}$ 

一致性Hash算法在服务**节点太少时**，容易因为节点分布不均匀而造成**数据倾斜**（被缓存的对象大部分集中缓存在某一台服务器上)问题，例如系统中只有两台服务器:

![image-20240404102311631](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404102311631.png)



小总结

为了在节点数目发生改变时尽可能少的迁移数据

将所有的存储节点排列在收尾相接的Hash环上，每个key在计算Hash后会顺时针找到临近的存储节点存放。而当有节点加入或退出时仅影响该节点在Hash环上顺时针相邻的后续节点。

$\textcolor{green}{\large 优点}$ ：加入和删除节点只影响哈希环中顺时针方向的相邻的节点，对其他节点无影响。

$\textcolor{green}{\large 缺点}$ ：数据的分布和节点的位置有关，因为这些节点不是均匀的分布在哈希环上的，所以数据在进行存储时达不到均匀分布的效果。



$\textcolor{red}{\large 哈希槽分区}$(大厂)

- 是什么？ HASH_SLOT = CRC16(key) mod 16384



### $\textcolor{red}{\large 经典面试题：为什么Redis集群的最大槽数是16384个？}$

Redis集群并没有使用一致性hash而是引入了哈希槽的概念。Redis 集群有16384个哈希糟，每个key通过CRC16校验后对16384取模来决定放置哪个槽，集群的每个节点负责一部分hash槽。但为什么哈希槽的数量是16384 (2^14）个呢？

CRC16算法产生的hash值有16bit，该算法可以产生2^16=65536个值。
换句话说值是分布在0～65535之间，有更大的65536不用为什么只用16384就够?

作者在做mod运算的时候，为什么不mod65536，而选择mod16384? $\textcolor{blue}{\large HASH\_SLOT = CRC16(key) mod 65536为什么没启用？}$





（减少每个主机发送心跳包信息的大小）

$\textcolor{blue}{\large (1)如果槽位为65536，发送心跳信息的消息头达8k，发送的心跳包过于庞大。}$

**因为每秒钟，redis节点需要发送一定数量的ping消息作为心跳包，如果槽位为65536，这个ping消息的消息头太大了，浪费带宽。**

$\textcolor{blue}{\large (2)redis的集群主节点数量基本不可能超过1000个。}$
集群节点越多，心跳包的消息体内携带的数据越多。如果节点过1000个，也会导致网络拥堵。因此redis作者不建议redis cluster节点数量超过1000个。那么，对于节点数在1000以内的redis cluster集群，16384个槽位够用了。没有必要拓展到65536个。
$\textcolor{blue}{\large (3)槽位越小，节点少的情况下，压缩比高，容易传输}$
Redis主节点的配置信息中它所负责的哈希槽是通过一张bitmap的形式来保存的，在传输过程中会对bitmap进行压缩，但是如果bitmap的填充率slots /N很高的话(N表示节点数)， bitmap的压缩率就很低。如果节点数很少，而哈希槽数量很多的话，bitmap的压缩率就很低。



每台服务器写入的槽位固定了，如果在a服务器写入b槽位的数据就会报错

- 解决办法：让服务器之间实现路由的转发







# Redis多线程

Redis的性能瓶颈有时会出现在网络IO的处理上，也就是说，单个主线程处理网络请求的速度跟不上底层网络硬件的速度

$\textcolor{red}{采用多个IO线程来处理网络请求，提高网络请求处理的并行度，Redis6/7就是采用的这种方法。}$

Redis的多IO线程只是用来处理网络请求的，**对于读写操作命今Redis仍然使用单线程来处理**。



### 面试题：redis为什么这么快

IO多路复用+epoll函数使用，才是redis为什么这么快的直接原因，而不是仅仅单线程命令+redis安装在内存中。

### redis到底是单线程还是多线程？

Redis的多IO线程只是用来处理网络请求的，**对于读写操作命今Redis仍然使用单线程来处理**。

### IO多路复用听说过吗？

将用户socket对应的文件描述符(FileDescriptor)注册进epoll，epoll帮你监听哪些socket上有消息到达，这是事件驱动，所谓的reactor反应模式。



![image-20240404105555183](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404105555183.png)





# BigKey

## 面试题

阿里广告平台，海量数据里查询某一个固定前缀的key

- scan



小红书，你如何生产上限制 keys* /flushdb/flushall等危险命令以防止阻塞或误删数据？

- 通过配置设置==禁用这些命令==，redis.conf在SECURITY这一项中



美团，memory usage命令你用过吗？

- 使用memory usage查找bigkey





BigKey问题，多大算big？你如何发现？如何删除？如何处理？

- 多大算bigkey

  - list、hash、set和zset，value个数超过5000就是bigkey

  - string类型控制在10kb以内

- 如何发现

  - memory usage 来计算每个键值的字节数

- 如何删除
  - 渐进式删除 
  - string 使用 unlink key删除(del是阻塞的)，hash使用hscan每次遍历，然后少量删除（渐进式逐步删除，直到全部删除完成），set，list，都是使用渐进式删除的方法（一次获取每个key删除）





BigKey你做过调优吗？惰性释放lazyfree了解过吗？

- 使用非阻塞的删除命令
- 惰性释放lazyfree，后台回收内存，在恒定时间内执行，在空闲时间执行不影响服务器的运行



morekey问题，生产上redis数据库有1000W记录，你如何遍历数据？ keys *可以吗？

- 不能 keys *是阻塞的
- 应该使用 scan来遍历



<font color = blue>keys * 这个指令有致命的弊端，在实际环境中最好不要使用</font>

- 查询速度很慢
- 通过配置设置==禁用这些命令==，redis.conf在SECURITY这一项中

不用keys *避免卡顿，那该用什么

- Scan命令登场



多大算bigkey

- key大指的==Key对应的value很大==
- list、hash、set和zset，value个数超过5000就是bigkey
- string类型控制在10kb以内







# 缓存双写一致性



你只要用缓存，就可能涉及到redis缓存与数据库双存储双写，你只要是双写，就一定会有数据一致性的问题，那么你如何解决一致性问题？

- 给缓存设置过期时间，定期清理缓存并回写，是保证最终一致性的解决方案
- 先更新mysql后更新redis





双写一致性，你先动缓存redis还是数据库MySQL哪一个？why？

- 先更新数据库，在删除缓存，假如缓存删除失败或者来不及删除，导致请求再次访问redis时缓存命中，读取到的是缓存的旧值
- 1先删除缓存值再更新数据库，有可能导致请求因缓存缺失而访问数据库，给数据库带来压力导致打满mysql。
- 2如果业务应用中读取数据库和写缓存的时间不好估算，那么，延迟双删中的等待时间就不好设置。



<font color = red>延时删除</font>你做过吗？会有哪些问题？

什么情况下使用延时删除，先删除缓存在更新数据库

- 先删除缓存在更新数据库存在的问题，删了缓存后数据库还没更新完毕，就是redis可能读到的旧值
- 此时多个请求打倒mysql上，给redis写回数据时，有可能写到mysql的旧值
- 延时删除，线程B能够先从数据库读取数据，再把缺失的数据写入缓存，然后，线程A再进行删除。线程A sleep的时间，就需要大于线程B读取数据再写入缓存的时间。



有这么一种情况，微服务查询redis无 MySQL有，为保证数据双写一致性回写redis你需要注意什么？<font color = red>双检加锁</font>策略你了解过吗？如何尽量避免缓存击穿？

- 注意先更新mysql然后更新redis 最终做到

- 采用双检加锁策略避免缓存击穿

==双检加锁==

- 请求进来查缓存，发现缓存没数据，加锁
- 加锁后再访问一次缓存，发现缓存还是没有数据，这时候就可以访问数据库了
- 两次检查，中间加了锁



redis和MySQL双写100%会出纰漏，做不到强一致性，你如何保证<font color = red>最终一致性？</font>

- 给缓存设置过期时间，定期清理缓存并回写，是保证最终一致性的解决方案
- 先跟新mysql后redis

![image-20240404115145387](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404115145387.png)





我想mysql有记录改动，立刻同步到redis，怎么做？

- canal



canal工作原理

- canal 模拟MySQL slave的交互协议，伪装自己为MySQL slave，向MySQL master发送dump协议

- MySQL master收到dump请求，开始推送binary log给slave (即canal )canal 解析binary log对象(原始为byte流)





# bigmap





1. 抖音电商直播，主播介绍的商品有评论，1个商品对应了一系列的评论，排序+展现+取前10条记录

   - zset

   - 在面对需要展示最新列表、排行榜等场景时，如果数据更新频繁或者需要分页显示，建议使用ZSet

   

2. 用户在手机APP上的签到打卡信息：1天对应一系列用户的签到记录，新浪微博、钉钉打卡签到，来没来如何统计

   - bitmap

   

3. 应用网站上的网页访问信息：一个网页对应一系列的访问点击，淘宝网首页，每天有多少人浏览首页

   - PV

   

4. 公司系统上线后，说一下UV、PV、DAU分别是多少？

- 什么是UV，Unique Visitor，独立访客，一般理解为客户端IP

- 什么是PV，Page View，页面浏览量（不用去重）

- 什么是DAU，Daily Active User，日活跃量用户，登录或者使用了某个产品的用户数（去重复登录的用户）

- 什么是MAU，Monthly Active User，月活跃用户量



排序统计

- zset

二值统计

- bitmap

基数统计

- 指统计一个集合中<font color='red'>不重复的元素个数</font>，见HyperLogLog
- HyperLogLog提供不精确的去重计数方案，只牺牲准确率来换取空间，误差仅仅只是0.81%左右
- bitmap
- hashset



什么是UV，Unique Visitor，独立访客，一般理解为客户端IP

什么是PV，Page View，页面浏览量（不用去重）

什么是DAU，Daily Active User，日活跃量用户，登录或者使用了某个产品的用户数（去重复登录的用户）

什么是MAU，Monthly Active User，月活跃用户量





1w个亿级UV

- 只牺牲准确率来换取空间，误差仅仅只是0.81%左右使用HyperLogLog，只是进行不重复的基数统计，不是集合也不保存数据，只记录数量而不是具体内容





GEO数据结构

- GEOADD 添加经纬度坐标
- GEOPOS返回经纬度
- GEOHASH返回坐标的geohash表示
- GEODIST两个位置之间的距离

![image-20240404121858811](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404121858811.png)



# 布隆过滤器

现有50亿个电话号码，给你10万个电话号码，如何要快速准确的判断这些电话号码==是否已经存在==？



判断是否存在，布隆过滤器了解过吗？



安全连接网址，全球数10亿的网址判断



黑名单校验，识别垃圾邮件



白名单校验，识别出合法用户进行后续处理



对key进行多个hash函数取值

- 可能产生hash冲突
- 无肯定无，有不一定有（大概率有）



缓存穿透是什么

- 一般情况下，先查询缓存redis是否有该条数据，缓存中没有时，再查询数据库。当数据库也不存在该条数据时，每次查询都要访问数据库，这就是缓存穿透。

  缓存穿透带来的问题是，当有大量请求查询数据库不存在的数据时，就会给数据库带来压力，甚至会拖垮数据库。

- 可以使用布隆过滤器解决缓存穿透的问题

  把已存在数据的key存在布隆过滤器中，相当于redis前面挡着一个布隆过滤器。

  当有新的请求时，先到布隆过滤器中查询是否存在:

  如果布隆过滤器中不存在该条数据则直接返回;

  如果布隆过滤器中已存在，才去查询缓存redis，如果redis里没查询到则再查询Mysql数据库



访问redis时，先访问布隆过滤器，看redis里面是否有key



### 小总结：

- 是否存在：有，很很可能有；无是肯定无，100%无
- 使用时最好不要让实际元素数量远大于初始化数量，一次给够避免扩容
- 当实际元素数量超过初始化数量时，应该对布隆过滤器进行重建，重新分配一个size 更大的过滤器，再将所有的历史元素批量add进行



### 优点

高效地插入和查询，内存中占用bit空间小

### 缺点

不能删除元素。因为删除元素会导致误判率增加，因为hash冲突同一个位置可能存的东西是多个共有的，你删除一个的同时可能也把其他的删除了

存在误判，不能精准过滤；有，是很有可能有；无，是肯定无





# 缓存预热，缓存雪崩，缓存穿透，缓存击穿







## 缓存预热

- 将热点数据提前加载到redis缓存中，可以通过@PostConstruct提前运行某个程序，将其加载到redis中





## 缓存雪崩

Redis主机挂了，Redis全盘崩溃，偏硬件运维

Redis中有大量key同时过期大面积失效，偏软件开发



解决

- Redis中key设置为永不过期或者过期时间为指定时间+随机时间，错开同时过期的概率
- Redis缓存集群实现高可用





## 缓存穿透

- redis无，mysql无，请求去查询一条记录，**先查redis无，后查mysql无，都查询不到该条记录但是请求每次都会打到数据库上面去，导致后台数据库压力暴增，这种现象我们称为缓存穿透**，这个redis变成了一个摆设。



### 解决

缓存穿透 ， 恶意攻击 的解决办法

- 空对象缓存或者缺省值
- 布隆过滤器



## 缓存击穿

- 热点key失效，暴打mysql

大量请求同时查询一个key时，此时这个key正好失效了，就会导致大量的请求都打到数据库，简单点就是热点key突然失效了，暴打MySQL



**解决办法**

方案一：差异失效时间，对于访问频繁的热点key，**干脆就不设置过期时间**

方案二：互斥更新，采用双检加锁策略





![image-20240404154408995](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404154408995.png)









# 分布式锁



### Redis做分布式的时候需要注意什么问题？



### 你们公司自己实现的分布式锁是否用的setnx命令实现?这个是最合适的吗?你如何考虑分布式锁的可重入问题?



### 如果Redis是单点部署的，会带来什么问题？准备怎么解决单点问题呢？



### Redis集群模式下，比如主从模式下，CAP方面有没有什么问题？

Redis集群是AP，高可用；Redis单机是C，数据一致性

86592695

### 简单介绍一下RedLock，谈谈Redisson



### Redis分布式锁如何续期，看门狗是什么？





分布式锁要注意什么

- 过期时间,自动解锁，解锁前判断是不是自己的锁
- 自动续期



lua脚本加锁 (第一次锁和重入锁)

```lua
# 若没锁和有锁 都要加锁
if redis.call('exists', KEYS[1]) == 0 or redis.call('hexists', KEYS[1], ARGV[1]) == 1 then
	# 加 hincrby锁
    redis.call('hincrby', KEYS[1], ARGV[1], 1)
    # 然后设置过期时间
	redis.call('expire', KEYS[1], ARGV[2])
	return 1
else
	return 0
end
```



lua脚本解锁

```lua
# 若没有锁
if redis.call('hexists', KEYS[1], ARGV[1]) == 0 then
	return nil
# 若有锁 然后hincrby-1  然后判断是否为0 为0就 del
elseif redis.call('hincrby', KEYS[1], ARGV[1], -1) == 0 then
    return redis.call('del', KEYS[1])
else 
    return 0
end
```





自动续期的LUA脚本

```lua
// 自动续期的LUA脚本
if redis.call('hexists', KEYS[1], ARGV[1]) == 1 then
    return redis.call('expire', KEYS[1], ARGV[2])
else
    return 0
end
```





# 红锁

![image-20240404162143410](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404162143410.png)



redis锁的复制是异步的，当master的锁没复制过去之后就宕机了，会出现两个服务器拿到同一把锁，会弄脏数据，此时要用官方的红锁

- 客户端从超过半数（大于等于N/2+1）的Redis实例上成功获取到了锁
- 客户端获取锁的总耗时没有超过锁的有效时间









# Redis缓存过期淘汰策略

### 生产上你们redis内存设置多少

一般推荐Redis设置内存为最大物理内存的3/4



### 如何配置、修改redis的内存大小

- 设置maxmemory参数，maxmemory是bytes字节类型





**redis默认内存多少可用？**



如果不设置最大内存或者设置最大内存大小为0，<font color = 'red'>在64位操作系统下不限制内存大小</font>，在32位操作系统下最多使用3GB内存

注意：在64bit系统下，maxmemory设置为0表示不限制redis内存使用



### 如果内存满了怎么办

![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/5.%E8%B6%85%E5%87%BA%E5%86%85%E5%AD%98.png)

内存满了会报错 OOM，需要删除内存



### redis清理内存的方式？定期删除和惰性删除了解过吗？



立刻删除 

- （key过期后马上删除）对CPU不友好

惰性删除

- key过期后不做处理，等到下次访问后如果发现未过期，返回数据 ;发现已过期，删除，返回不存在。
- 对内存不友好，如果过期后一直访问不到 就一直在内存中，造成内存泄漏

定期删除

- 每隔一段时间执行一次删除过期键操作，并通过限制删除操作执行时长和频率来减少删除操作对CPU时间的影响。
- 周期性抽查存储空间 (随机抽查，重点抽查)
- 随机抽取肯定有没有抽到的

### redis缓存淘汰策略有哪些？分别是什么？用过那个？

- 立刻删除
- 惰性删除
- 定期删除

- LRU
- LFU

LRU：最近<font color = 'red'>最少使用</font>页面置换算法，淘汰最长时间未被使用的页面，看页面最后一次被使用到发生调度的时间长短，首先**淘汰最长时间未被使用的页面**。 **按照时间排序**

LFU：最近<font color = 'red'>最不常用</font>页面置换算法，淘汰一定时期内被访问次数最少的页面，看一定时间段内页面被使用的频率，淘汰一定时期内被访问次数最少的页。  **按照访问次数排序**

### redis的LRU了解过吗？请手写LRU



### LRU和LFU算法的区别是什么？

LRU means Least Recently Used 最近最少使用

LFU means Least Frequently Used 最不常用



### 内存打满了，超出了设置的最大值会怎么样？

![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/5.%E8%B6%85%E5%87%BA%E5%86%85%E5%AD%98.png)



### redis缓存淘汰策略配置性能建议

- 避免存储BigKey
- 开启惰性删除，lazyfree-lazy-eviction=yes







# 数据类型



![image-20240404165002306](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404165002306.png)



![image-20240404165031546](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404165031546.png)



![image-20240404165036851](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404165036851.png)



## string类型



string类型由三种格式组成

- int

  - 保存long 型（长整型）的64位（8个字节）有符号整数
  - 只有整数才会使用int，如果是浮点数，Redis内部其实先将浮点数转化为字符串值，然后再保存。

- embstr

  - 代表embstr格式的SDS(Simple Dynamic String简单动态字符串)，保存长度小于44字节的字符串


  ​    EMBSTR顾名思义即：embedded string，表示嵌入式的String

- raw

  - 保存长度大于44字节的字符串



![image-20240404165416182](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404165416182.png)

![image-20240404165429783](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404165429783.png)





## hash

1. 哈希对象保存的键值对数量小于 512个；
2. 所有的键值对的健和值的字符串长度都小于等于 64byte (一个英文字母一个字节时用listpack，反之用hashtable
3. listpack升级到Hashtable可以，反过来降级不可以





## List

![image-20240404170002484](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404170002484.png)



==redis7的quicklist中 使用listpack代替ziplist==

![image-20240404170447054](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404170447054.png)







## Set



Redis用intset或hashtable存储set。如果元素都是整数类型，就用intset存储。





## ZSet

当有序集合中包含的元素数量超过服务器属性 server.zset_max_ziplist_entries 的值(默认值为 128 )，或者有序集合中新添加元素的 member 的长度大于服务器属性 server.zset_max_ziplist_value 的值(默认值为 64 )时，redis会使用跳跃表作为有序集合的底层实现。
否则会使用ziplist作为有序集合的底层实现





## 跳表



在原链表上建立索引，牺牲空间换取时间

- skiplist是一种以空间换取时间的结构。由于链表，无法进行二分查找，因此借鉴数据库索引的思想，提取出链表中关键节点(索引)，先在关键节点上查找，再进入下层链表查找，提取多层关键节点，就形成了跳跃表。

![image-20240404170810719](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404170810719.png)



通过规律可得

- n个节点 2^h （最高级索引）级索引的节点个数为2，其中h为高度
- 此时高度就为时间复杂度

![image-20240404171157695](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404171157695.png)





# IO多路复用

### 能干嘛

Redis单线程如何处理那么多并发客户端连接，为什么单线程，为什么快

Redis利用epoll来实现IO多路复用，<font color = 'blue'>将连接信息和事件放到队列中</font>，依次放到事件<font color = 'blue'>分派器</font>，事件分派器将事件分发给事件处理器。



==其中事件队列是内核态来遍历==，这样就不用用户态和内核态之间切换，节省了很多时间

![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/9.RedisIO%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8.png)

Redis 服务采用 Reactor 的方式来实现文件事件处理器(每一个网络连接其实都对应一个文件描述符)谓 I/0 多路复用机制，就是说通过一种机制，可以监视多个描述符，一旦某个描述符就绪(一般是读就绪或写就绪)，能够通知程序进行相应读写操作。这种机制的使用需要 select 、 poll 、 epoll 来配合。多个连接共用一个阻塞对象，应用程序只需要在一个阻塞对象上等待，无需阻塞等待所有连接。当某条连接有新的数据可以处理时，操作系统通知应用程序，线程从阻塞状态返回，开始进行业务处理。



#### 结论

多路复用快的原因在于，操作系统提供了这样的系统调用，使得原来的while循环里多次系统调用，<font color = 'red'>变成了一次系统调用＋内核层遍历这些文件描述符。</font>

epoll是现在最先进的IO多路复用器，Redis、Nginx，linux中的Java NIO都使用的是epoll。

<font color = 'red'>这里“多路”指的是多个网络连接，“复用”指的是复用同一个线程。</font>

1、一个socket的生命周期中只有一次从用户态拷贝到内核态的过程，开销小

2、使用event事件通知机制，==每次socket中有数据会主动通知用户==，并加入到就绪链表中，不需要遍历所有的socket



![image-20240404172816013](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/image-20240404172816013.png)