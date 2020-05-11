---
layout: post
title:  "关于etcd multiple_member_cluster的部署.md"
date:   2020-02-05 14:06:05
categories: go
tags: go etcd kv 
---

* content
{:toc}

## 一. 关于goreman部分

1.  go get github.com/mattn/goreman将代码下载到本地的$GOPATH/src下
2. 在goreman根目录下执行 go build,生成goreman程序
3. 将goreman程序：cp goreman $GOPATH/bin下

## 二.启动local multiple_member_cluster

1. 切换到etcd根目录下

2. 执行goreman -f Profile start,输出如下内容

   ```
   16:00:04 etcd2 | Starting etcd2 on port 5100
   16:00:04 etcd1 | Starting etcd1 on port 5000
   16:00:04 etcd3 | Starting etcd3 on port 5200
   16:00:04 etcd1 | {"level":"info","ts":"2020-01-06T16:00:04.125+0800","caller":"etcdmain/etcd.go:110","msg":"failed to detect default host","error":"default host not supported on darwin_amd64"}
   16:00:04 etcd1 | {"level":"warn","ts":"2020-01-06T16:00:04.126+0800","caller":"etcdmain/etcd.go:119","msg":"'data-dir' was empty; using default","data-dir":"infra1.etcd"}
   16:00:04 etcd2 | {"level":"info","ts":"2020-01-06T16:00:04.125+0800","caller":"etcdmain/etcd.go:110","msg":"failed to detect default host","error":"default host not supported on darwin_amd64"}
   16:00:04 etcd2 | {"level":"warn","ts":"2020-01-06T16:00:04.126+0800","caller":"etcdmain/etcd.go:119","msg":"'data-dir' was empty; using default","data-dir":"infra2.etcd"}
   16:00:04 etcd2 | {"level":"info","ts":"2020-01-06T16:00:04.127+0800","caller":"embed/etcd.go:117","msg":"configuring peer listeners","listen-peer-urls":["http://127.0.0.1:22380"]}
   16:00:04 etcd1 | {"level":"info","ts":"2020-01-06T16:00:04.127+0800","caller":"embed/etcd.go:117","msg":"configuring peer listeners","listen-peer-urls":["http://127.0.0.1:12380"]}
   16:00:04 etcd3 | {"level":"info","ts":"2020-01-06T16:00:04.126+0800","caller":"etcdmain/etcd.go:110","msg":"failed to detect default host","error":"default host not supported on darwin_amd64"}
   16:00:04 etcd1 | {"level":"info","ts":"2020-01-06T16:00:04.127+0800","caller":"embed/etcd.go:127","msg":"configuring client listeners","listen-client-urls":["http://127.0.0.1:2379"]}
   16:00:04 etcd1 | {"level":"info","ts":"2020-01-06T16:00:04.127+0800","caller":"embed/etcd.go:602","msg":"pprof is enabled","path":"/debug/pprof"}
   16:00:04 etcd3 | {"level":"warn","ts":"2020-01-06T16:00:04.126+0800","caller":"etcdmain/etcd.go:119","msg":"'data-dir' was empty; using default","data-dir":"infra3.etcd"}
   16:00:04 etcd3 | {"level":"info","ts":"2020-01-06T16:00:04.127+0800","caller":"embed/etcd.go:117","msg":"configuring peer listeners","listen-peer-urls":["http://127.0.0.1:32380"]}
   16:00:04 etcd3 | {"level":"info","ts":"2020-01-06T16:00:04.127+0800","caller":"embed/etcd.go:127","msg":"configuring client listeners","listen-client-urls":["http://127.0.0.1:32379"]}
   16:00:04 etcd3 | {"level":"info","ts":"2020-01-06T16:00:04.127+0800","caller":"embed/etcd.go:602","msg":"pprof is enabled","path":"/debug/pprof"}
   
   ```

## 三.简单操作

1.查看cluster中member：etcdctl --write-out=table --endpoints=localhost:2379 member list

```
+------------------+---------+--------+------------------------+------------------------+------------+
|        ID        | STATUS  |  NAME  |       PEER ADDRS       |      CLIENT ADDRS      | IS LEARNER |
+------------------+---------+--------+------------------------+------------------------+------------+
| 8211f1d0f64f3269 | started | infra1 | http://127.0.0.1:12380 |  http://127.0.0.1:2379 |      false |
| 91bc3c398fb3c146 | started | infra2 | http://127.0.0.1:22380 | http://127.0.0.1:22379 |      false |
| fd422379fda50e48 | started | infra3 | http://127.0.0.1:32380 | http://127.0.0.1:32379 |      false |
+------------------+---------+--------+------------------------+------------------------+------------+
```

2. 添加kv/获取key对应内容

  etcdctl --endpoints=localhost:32379 put foo1 bar1

  etcdctl --endpoints=localhost:32379 get foo1

  etcdctl  get foo1  

3.watch 

```
./bin/etcdctl watch key<实际的key>
或
./bin/etcdctl watch -i watch key<实际的key>
```

可查看key执行的历史记录

```
./bin/etcdctl watch --rev=reversion key<实际的key>
```

>raft_term: 代表每次leader发生变化时，该值就会递增（全局）；
>
>revision：代表每次被修改(eg. Put/Delete/Txn等操作)该值会被递增；
>
>mod_revision: 代表当前kv被修改最近一次的版本；
>
>create_revision: 代表当前kv创建时的版本；
>
>version: 代表当前kv从创建到现在经历的版本数(mod_revision-create_revision);

4.transaction:

```
$ etcdctl put flag 0
$ etcdctl txn -i  # 执行txn
compares:
value("flag") = "1"   

success requests (get, put, del):
put hello world_123

failure requests (get, put, del):
put hello world_world_ooo

FAILURE

OK
```

5.lease：
etcd能为key设置超时时间，etcd需要先创建lease，然后使用put命令加上参数–lease=<lease ID>来设置

```
$ etcdctl lease grant 100  #创建lease
lease 38015a3c00490513 granted with TTL(100s)
$ etcdctl put hello world --lease=38015a3c00490513 # 授权lease
OK
$ etcdctl lease timetolive 38015a3c00490513  # 查看某个lease
lease 38015a3c00490513 granted with TTL(100s), remaining(67s)
$ etcdctl lease timetolive 38015a3c00490513 --keys # 查看某个lease关联的keys
lease 38015a3c00490513 granted with TTL(100s), remaining(59s), attached keys([hello])
```

> ### 补充：
>
> 1.Logical view
>
> 在etcd中logical view其实就是一个binary key space，并支持key按照词法index排查能够进行范围查询。logical view支持key的多版本内容，每当进行modify操作时都会触发，就会在key-space新增一个版本。同时以前的会保持不变的，通过指定revision来获取当前key对应的历史版本。同样revision会作为index，这样就可以结合watch来进行操作，完成对某个key的操作。随着key space不停的产生新版本的内容，会导致整个cluster维护数据量变大，本身消耗的资源递增，则通过compact来节省现有的空间。
>
> 存在key space中任一key的生命期：从创建到删除。每个key会有至少1次产生（每个可具有不止一个revision）。当创建一个不存在key，则会从1开始递增产生version；
>
> 而每当删除一个key时，则会产生tombstone并将key当前的version置为0；
>
> 针对每个key的修改，则会导致key的version+1；
>
> 在key产生时，其关联的version都是单调递增的。一旦发生compaction，在改compaction指定的revision前面的revision会被移除，同样该revision之前的values也会被移除。
>
> 2.Physical view
>
> 在etcd存储的数据，是以kv对的方式以B+ tree存储。存储状态的每个revision只包含前面revision的增量，以提高效率。单个revision可能对应于tree中的多个keys。
>
> 而kv中的key是一个三元组<major, sub, type>: Major对应key的revision；Sub用于区分属于同一个revision的不同keys；Type作为指定value的后缀(可选的)。
>
> ke中的value保留前面所有revision，当前revision的value都是前面revision的增量。b+ tree是按照词法字节排序，故而通过range查询速度相对比较快。在进行compation时会清除过时的kv对。
>
> etcd会在memory存放一个二级index加快数据的查询，特别是range查询。



## 四. 新增node

在前面已启动的local multiple_member_cluster新增member

1. 执行添加member：etcdctl member add infra4 --peer-urls="http://127.0.0.1:42380" --learner=true

2. 启动member：
etcd --name infra4 --listen-client-urls http://127.0.0.1:42379 --advertise-client-urls http://127.0.0.1:42379 --listen-peer-urls http://127.0.0.1:42380 --initial-advertise-peer-urls http://127.0.0.1:42380 --initial-cluster-token etcd-cluster-1 --initial-cluster 'infra4=http://127.0.0.1:42380,infra1=http://127.0.0.1:12380,infra2=http://127.0.0.1:22380,infra3=http://127.0.0.1:32380' --initial-cluster-state existing --enable-pprof --logger=zap --log-outputs=stderr

3. 验证member是否已添加到etcd cluster： etcdctl member promote 8de0eb3c0ff43347<新增member对应的id>

4. 查看当前cluster中member： etcdctl --write-out=table member list

   ```
   +------------------+---------+--------+------------------------+------------------------+------------+
   |        ID        | STATUS  |  NAME  |       PEER ADDRS       |      CLIENT ADDRS      | IS LEARNER |
   +------------------+---------+--------+------------------------+------------------------+------------+
   | 8211f1d0f64f3269 | started | infra1 | http://127.0.0.1:12380 |  http://127.0.0.1:2379 |      false |
   | 8de0eb3c0ff43347 | started | infra4 | http://127.0.0.1:42380 | http://127.0.0.1:42379 |       true |
   | 91bc3c398fb3c146 | started | infra2 | http://127.0.0.1:22380 | http://127.0.0.1:22379 |      false |
   | fd422379fda50e48 | started | infra3 | http://127.0.0.1:32380 | http://127.0.0.1:32379 |      false |
   +------------------+---------+--------+------------------------+------------------------+------------+
   ```

## 五. 关于learner设计

#### 第一部分 实例

###### 实例一：添加新member，leader负载

   当向现有etcd cluster添加新的member，此时该member node没有任何数据，需要从leader同步数据直至追上leader的最新数据。在此过程中可能会导致leader node的network负载，可能会导致发向followers的hearbeats被阻塞或丢失。这样在当前leader的election-timeout周期内，followers可能会触发新的一轮leader election。换而言之，当向一个etcd cluster添加新member时会影响leader election。故而leader election和后续的数据同步到一个新的member都会对cluster产生一定的影响，导致其是否可用。
![当一个新member加入cluster并没有数据，接着向leader请求同步数据直至追上leader.同步给新member的snapshots过大导致leader的network过载，进而导致leader向cluster其他follower发送hearbeat阻塞甚至丢失，在达到指定election-timeout有效期，follower会进行新一轮的leader election](https://upload-images.jianshu.io/upload_images/5525735-19a6d6ec08c78373.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  
###### 实例二：leader isolation

   当cluster中leader和其他部分follower隔离会对整个cluster产生影响导致其不可用，由于leader需要监控每个follower的进展，一旦不能和quorum的follower间互通，在超过election-timeout周期后followers也会触发新一轮的leader election。

   ![在一个包含3个node的cluster中，leader与其他两个node间隔离，leader就需要至少1个active follower(包括leader在内总共2个active node)，而此时leader没有其他任何一个active node，达不到quorum。接着就会到达election-timeout后，触发新一轮leader election](https://upload-images.jianshu.io/upload_images/5525735-c1f7dd20e72bdb9d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   在前面的两个实例中，展示向cluster添加一个新member会导致什么问题或潜在的情况，接下来结合实例三来讲解：向一个已包含3个nodes cluster添加一个新的member后，network partitions变化？是否取决于partition后新member隶属于哪个partition？

###### 实例三：向3个node cluster添加一个新的member

   1. 假如当前新增的member和leader属于相同的partition

      在此种情况下，leader仍维持3个active quorum，也就是说leader election是不会发生的，对现有的cluster不产生任何影响，如下图：
![image.png](https://upload-images.jianshu.io/upload_images/5525735-4a34024a505452bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


   2. 当新增的member和leader不在同一partition，形成2-2 partitioned，这样两个partition都没有达到quorum，会导致leader election发生
![image.png](https://upload-images.jianshu.io/upload_images/5525735-4e895e3d042b3ba7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   3. 达不到quorum

      当cluster先发生partition，接着有新member加入？比如现有3个node的cluster，此时有一个follower和leader间不可互通，这时有新member加入，原来集群的quorum也由2变更为3，然而此时4个node cluster其实只有2个active followers，就导致在进行新一轮的leader election时不能达到quorum的要求。
![image.png](https://upload-images.jianshu.io/upload_images/5525735-675d412b151ed713.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
      由于新member的添加会导致原有cluster的quorum size发生变更，此时要优先将集群现有不健康的节点剔除，在进行新member的添加替换原有不健康的node。

      当向1-node cluster添加新member时，会导致quorum size变更为2，当previous leader发现quorum是无效的会立刻会触发leader election：由于“member add”属于2-step操作，首先需要完成member添加，接着启动新member node的process。如下图
![image.png](https://upload-images.jianshu.io/upload_images/5525735-81d6b4d1830cb0b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   4. cluster配置错误

      在实际的应用可会出现比较糟糕的事情：当进行一个新member添加时配置错误，再加上membership reconfiguration属于2-steps操作，首先“etcdctl member add”，接着启动根据指定peer URL启动server process。也就是说 不管URL是什么甚至URL指定的值无效，也会应用成员添加命令。若是第一步使用了无效的url，那么第二步甚至不能启动新的etcd。一旦集群达不到指定的quorum，就无法恢复成员更改。

![image.png](https://upload-images.jianshu.io/upload_images/5525735-76b17cec3ce25d00.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同样在多节点集群中，比如集群有两个members宕机(一个失败，另一个配置错误)，两个members宕机，但现在需要至少3个quorum才能更改cluster membership。如下图

![image.png](https://upload-images.jianshu.io/upload_images/5525735-b98aefb1dbb6cb2e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如上所述，简单的错误配置可能会使整个群集无法工作。 在这种情况下，operator需要使用etcd --force-new-cluster来手动重新创建集群。 而由于etcd已成为Kubernetes的关键任务服务，即使是最轻微的中断也可能对用户产生重大影响。 我们怎样才能使etcd这样的操作更容易？ 除其他事项外，leader election对集群可用性至关重要：我们是否可以通过不更改quorum size来降低members reconfiguration的破坏性？ 一个新节点是否可以idle，仅向领导者请求最少的更新，直到它赶上leader？ membership 错误配置是否可以始终可撤销的并以更安全的方式处理（错误的member add命令运行应永远不会使集群fail）？ 添加新成员时，用户是否应该担心network topology？ 不管节点和正在进行的网络分区的位置如何，member add API都可以工作？

      
#### 第二部分 Raft Learner
为了解决前面实例中的情况，新增了一个新的node state：learner：当有新member加入到cluster时，首先该member作为一个**non-voting member**，直到其追上leader logs，进而转为member。

1. features in v3.4
要使一个新的learner node相对比较简单：member add --learner 来添加一个learner node，此时该member只是作为一个**non-voting member**，并能够接收leader的logs，直至追上leader。

![image.png](https://upload-images.jianshu.io/upload_images/5525735-9ad53358ea34ca7f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
一旦learner追赶上leader进度后，使用“member promote”api来将该learner变成具有quorum的member：

![image.png](https://upload-images.jianshu.io/upload_images/5525735-8449653fbaa69ac9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对于一个learner是否能够变为voting-member则需要etcd server来验证promoted request来确保安全，并保证learner已经赶上leader的进度了。

![image.png](https://upload-images.jianshu.io/upload_images/5525735-34dc04435b198bb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在etcd server没有promoted request检验之前，learner会一直作为standby node存在：Leadership不能变为leaner，并且learner不对外提供read和write（client balancer不会路由请求到learner）。也就是说learner不需要向leader发送read index请求。

![image.png](https://upload-images.jianshu.io/upload_images/5525735-fc14095b3e0f262f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

另外，etcd也会限制cluster中存在learners的数量，并避免leader进行log replication的负载。另外learner node不会主动提升自己变为voting-member，etcd也提供learner status信息和安全检查，而cluster operator会做出最终决定是够将learner提升为voting-member。

       
2. features in v3.5
默认情况下，新增一个member其状态为learner，在当前新member未变成“voting-member”前，是不会改变quorum size，同样Misconfiguration能够撤销保证quorum不会lose。

使voting-member promotion过程完全自动化：learner追上leader的logs后，cluster便能自动promote leaner。 etcd要求用户定义某些阈值，一旦满足要求，learner便会提升为voting-member。 从用户的角度来看，“member add”命令的工作方式相同，但learner功能可提供更高的安全性。

使learner成为standby failover node：learner加入并成为standby node，并在集群可用性受到影响时自动promoted。

使“learner”成为read-only节点:“learner”可以作为一个read-only节点，永远不会被promoted。在weak consistency模式下，learner只接收leader的数据，从不处理write操作。在没有consensus的情况下提供本地读操作将极大地减少leader的工作负载，但可能会提供stale data。在强consistency模式下，learner请求从leader处读取索引以提供最新数据，但仍然拒绝write操作。

3. Learner vs Mirror Maker
etcd使用watch API实现“mirror maker”，以持续地将key创建和更新到一个单独的集群中。 一旦完成初始同步，Mirror通常具有较低的延迟开销。learner和mirror的重叠之处在于，两者均可用于复制现有数据以只读方式。 但是，mirror不能保证线性化。 在网络断开连接期间，以前的key-values可能已被丢弃，并且希望clients验证监视响应的正确顺序。 因此，Mirror中没有订购保证。 使用Mirror来减少延迟（例如跨数据中心），以保持一致性为代价。 使用learner保留所有历史数据及其顺序。

 #### 第三部分 ： Learner实现
etcd client中添加一个flag在**Member Add**API来标示learner node。具体操作见前面**[新增node]**部分。

## 引用
 - Original github issue: [etcd#9161](https://github.com/etcd-io/etcd/issues/9161)
 - Use case: [etcd#3715](https://github.com/etcd-io/etcd/issues/3715)
 - Use case: [etcd#8888](https://github.com/etcd-io/etcd/issues/8888)
 - Use case: [etcd#10114](https://github.com/etcd-io/etcd/issues/10114)
 - Design-Leaner:[design-learner.md](https://github.com/etcd-io/etcd/blob/master/Documentation/learning/design-learner.md)
  
