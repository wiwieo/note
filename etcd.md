# etcd 笔记
[etcd](https://coreos.com/etcd/)是一个高可用，持久化，分布一致，多版本，并发控制的key-value库，主要用于低频更新的配置及服务注册使用
## Demo实例
参考：[etcd 操作实例](https://coreos.com/etcd/docs/latest/demo.html)
## 数据模型
### 逻辑视图
* 1、key是一个字典排序的byte字串，因此，适合使用范围查询
* 2、key空间维护多版本（也维护了索引，便于查找），每次值的改变，在key空间下新增一条，老版本的值仍然可以访问。
### 物理视图
* 1、使用B+ Tree 存储，采用增量存储
* 2、key是一个三元组（major, sub, type），官网内容，没明白啥意思
```
Major 是指 存储key的版本， sub指 同一版本内key的区别， type指 对于特殊值给定的可选前缀
（Major is the store revision holding the key. Sub differentiates among keys within the same revision. Type is an optional suffix for special value）
```
* 3、value存储了与上个版本间的差别，即：只保存增量
* 4、etcd另外，在内存中维护了一份B Tree索引，以加快key的查找。此处的key是暴露给用户使用的，value是指向B+ Tree（非内存）的增量（修改值）

## 错误处理
L：Leader， F：Follower，C：Candidate
### 失败种类
* 1、少数`F`失败：少于`（n - 1）/ 2`个节点失败，则可以继续接收请求，并无故障地处理。
* 2、`L`失败：自动选举出新的`L`，但是选举不会立即进行，可能至多等待一个`选举期`，选举期间，不能进行写操作，
所有写操作会存放在一个队列中，直到新的`L`诞生。已经发送到旧的`L`的未提交的写可能会丢失，但已提交的不会。
另，新选出的`L`，会自动延申所有租约的超时时间。
* 3、大部分节点失败，则集群将处于不可用状态，需要进行`灾难恢复`
* 4、网络分区，类似于`少数失败`的情况
* 5、自荐期间失败

### 灾难恢复
* 1、快照 `ETCDCTL_API=3 etcdctl --endpoints $ENDPOINT snapshot save snapshot.db`
* 2、集群恢复，必须使用同一个快照，可以使用`--skip-hash-check`来跳过快照完整性验证（直接COPY的快照文件时，会验证不通过）
```
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
  --name m1 \
  --initial-cluster m1=http://host1:2380,m2=http://host2:2380,m3=http://host3:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls http://host1:2380
```

## 参数修改
正常情况下，默认参数就可以，但在网络延迟比较高或者需要穿过多台数据中心，则需要调整相应的参数
### 时间参数
* 1、`Heartbeat Interval`，`L`通知`F`活着仍然是`L`，默认是100ms，通常按照节点间通信的往返时间（RTT Round-Trip Time）
* 2、`Election Timeout`，心跳断开多久，`F`可以申请成为`L`，一般是心跳时间的10倍，默认为1000ms。上限时间为：50000ms（50s）
```
# Command line arguments:
$ etcd --heartbeat-interval=100 --election-timeout=500

# Environment variables:
$ ETCD_HEARTBEAT_INTERVAL=100 ETCD_ELECTION_TIMEOUT=500 etcd
```

### 快照
etcd是将所有改变追加到一个日志文件，线性增长，随着时间的推移，会生成一个特别庞大的文件。所以，需要使用快照来减小日志文件。
默认情况下，生成10000条日志后进行一次快照，如果内存和硬盘使用太大，可以降低条数：
```
# Command line arguments:
$ etcd --snapshot-count=5000

# Environment variables:
$ ETCD_SNAPSHOT_COUNT=5000 etcd
```

### 硬盘
etcd对磁盘延迟比较敏感，因为etcd必须将目标持久化到日志中，如果磁盘使用过高，会导致磁盘写入延迟，引起请求超时和临时`L`丢失。可以设置磁盘优先级来避免：
```
# Linux 设置磁盘优先级
# best effort, highest priority
$ sudo ionice -c2 -n0 -p `pgrep etcd
```

### 网络
```
If the etcd leader serves a large number of concurrent client requests, it may delay processing follower peer requests due to network congestion. This manifests as send buffer error messages on the follower nodes:

dropped MsgProp to 247ae21ff9436b2d since streamMsg's sending buffer is full
dropped MsgAppResp to 247ae21ff9436b2d since streamMsg's sending buffer is full

These errors may be resolved by prioritizing etcd's peer traffic over its client traffic. On Linux, peer traffic can be prioritized by using the traffic control mechanism:

tc qdisc add dev eth0 root handle 1: prio bands 3
tc filter add dev eth0 parent 1: protocol ip prio 1 u32 match ip sport 2380 0xffff flowid 1:1
tc filter add dev eth0 parent 1: protocol ip prio 1 u32 match ip dport 2380 0xffff flowid 1:1
tc filter add dev eth0 parent 1: protocol ip prio 2 u32 match ip sport 2739 0xffff flowid 1:1
tc filter add dev eth0 parent 1: protocol ip prio 2 u32 match ip dport 2739 0xffff flowid 1:1
```

## Raft算法
* [Raft论文-中文](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md)
* [Raft论文-英文](https://ramcloud.atlassian.net/wiki/download/attachments/6586375/raft.pdf)
* [秘密武器](http://thesecretlivesofdata.com/raft/)

### 总结
* 1、领导人选举
* 2、日志复制
* 3、成员变更

### Raft 算法实现
[etcd/raft](https://github.com/etcd-io/etcd)