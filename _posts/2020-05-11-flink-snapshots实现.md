---
layout: post
title:  "Lightweight Asynchronous Snapshots for Distributed Dataflows(译文)"
categories: flink
tags:  大数据 flink streaming Chandy-Lamport
author: 神奇的考拉
---

* content
{:toc}

## 摘要

分布式状态流式处理能够使得持久化计算能够大规模部署到云上进行执行，达到低延迟和高吞吐的目标。不过所面临的最大挑战提供针对潜在失败处理的保证，而目前已有的解决方案都是依赖周期性的全局state snapshot来进行故障恢复的。故而这些方案会存在两大缺陷：

- 由于需要进行全局快照的恢复，会影响到整体处理能力，导致整体数据的摄取能力
- 因为周期性的进行全局当前operator state以及数据的快照，故而会导致所需的资源变大

因此针对如上的情况，我们采用异步屏障快照Asynchronous Barrier Snapshotting(ABS)，相对周期性全局快照，ABS相对是一个比较适合现代数据流处理的轻量级引擎，并且所有的资源空间占用较少的轻量级算法。ABS只会存储非周期性执行topology上的operator state同时在循环数据流上保留最小化的record日志。接下来看看ABS在flink中的实现，通过实际的评估，ABS算法对flink执行性能不会产生很大的严重，并且水平扩张和频繁快照的情况下也能表现良好。

> Flink是一个分布式状态流式分析引擎。见[flink](github.com/apache/flink)

## 1.介绍

分布式数据流处理是一种新型的数据密集型计算的范式，它会对大数量级进行持续计算，同时能够保持end-to-end的低延迟和处理高吞吐性能。相对一些对时间要求比较高应用能够通过数据流处理系统获得不错的结果，比如apache flink和naiad，尤其在一些实时分析领域(eg.预测分析和复杂事件处理)。与此同时“fault tolerance”在这类系统中就变的尤为重要，在真实实际应用中failures是不能被彻底剔除的，需要对出现failure有对应的措施来保证consistent。在目前已知的方案中会通过依赖全局一致性的execution state针对那些stateful处理系统来保证exactly-once的语义。但是有两个“致命”的缺点会导致在real-time流式处理中性能低下。

- 同步快照技术是通过停止整体分布式计算的执行来获取一个全局一致性state的view，这会导致系统处理性能比较低下
- 现有已知的分布式快照算法会将在传输的记录或在execution graph中未处理的消息作为snapshot的一部分，通常情况下会导致state所有资源远远大于所需要的

故而，我们专注于提供轻量级snapshot，专门针对分布式状态流系统，并且在性能上损耗较小。通过提供异步状态snapshot，同时只包含在非循环执行topology上的operatot state的较小空间资源损耗。另外，通过down-stream在topology被选中的部分vertex的backup来描述cyclie execution graph的情况，同时保持snapshot state大小最小，这样就可以损耗较小runtime性能而非中断流operator。论文主要包括如下内容：

- 提出并实现一个异步snapshot算法，使得其在非循环执行图上实现资源损耗最小的snapshot

- 描述并实现用在循环执行图上的算法

- 与现有的apache flink streaming相比，当前算法的优势

  

## 2.相关工作

当前业界也为持续处理系统提供几种恢复机制，例如将处理处理模拟成无状态分布式批处理的系统(Discretized Stream或Comet)，再通过state重复计算来解决容错问题；另一方面，有状态流式计算系统比如Naiad/SDGS/Piccolo/SEEP，这也是我们算法实现的关注点，通过使用checkpoint来获取进行failure recovery的全局执行的一致性snapshot。由Chandy和Lamport提出的分布式环境中一致全局snapshot在过去一段时间得到广泛的研究。一个全局snapshot理论上反应了execution的整体状态，或是operator某个指定的实例上可能的状态。比如Naiad采用了一种简单而成本高的方法，分三步执行同步快照：

1. 中断/暂停当前execution graph的整体计算
2. 执行快照
3. 在全局快照完成后，使前面被中断/暂停的task继续执行

如此循环周期执行，故而该实现对系统的吞吐量和空间资源上都会有很大的影响，执行快照的时候需要block整个计算，同时还依赖up-stream，生产端产生logs记录的backup。

另外一种比较流行的方案是由Chandy和Lamport最初提出的，并且现已在很多系统中得到实现，是在做up-stream的backup时执行异步快照。通过exection graph来分配markers，而这些markers对应触发对应的operator和channel的状态持久化(operator代表对数据的操作结果，channel代表传输数据)。但是，这种方法仍然需要额外的空间，因为需要up-stream的backup，并且由于备份记录的重新处理而导致更高的恢复时间。故而我们下面讲述的方案扩展了Chandy和Lamport最初的异步快照的思想，但不会考虑非循环图记录的备份日志记录，同时会在循环执行图上保留必要的备份记录。



## 3.Apache Flink实现

接下来了解下Apache Flink Streaming的fault-tolerance，<!--Apache Flink Streaming是一个分布式流式分析系统，隶属于Apache Flink Stack的一部分。-->Apache Flink是围绕一个通用的Runtime引擎架构的，它统一处理由有状态的互连的task组成的批处理和流处理作业，换句话说也就是统一了流式和批处理的model。当执行flink job时，会将job编译成tasks的DAG。数据从外部源获取，并以pipeline的方式传输，在task graph进行路由。每个task就会基于接收到的输入来操作其内部state，并产生新的输出，继续往下游传输。

##### 3.1 流式编程模型

Apache Flink通过提供的api来将unbounded分区数据stream(部分有序的记录序列)作为其核心数据抽象，称之为Datastreams，进而实现复杂流分析的组合。DataStream既能通过外部数据源来创建(例如消息队列/socket流/自定义生成等)也可通过对其他DataStream调用operator产生。Datastream以高阶函数的形式提供多种operator如map/filter/reduce等多种函数，而这些函数能应用在每条记录上，也可生成新的Datastream。每个operator能通过使用并行实例执行各自的数据流分区，同样也可以进行分布式执行stream transformation。

通过下面的代码实例展示Apache flink实现一个增量的wordcount。
![增量wordcount执行图](https://upload-images.jianshu.io/upload_images/5525735-b5b6e9091b6df568.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
// 伪码如下：
val env : StreamExecutionEnvironment = ...
env.setParallelism(2)

val wordStream = env.readTextFile(path)
val countStream = wordStream.groupBy(_).count
countStream.print
```

由于需要从文本中读取单词，并对每个单词的count进行打印输出到标准输出流，source需要关注当前文件读取的offset，以及每个单词需要关注其内部state来完成count。故而这是一个stateful的流式处理程序。



##### 3.2 分布式数据流执行

当用户开始执行一个flink streaming application时，会将所有的DataStream operator编译成一个Execution Graph，也就是一个DAG图G=(T,E),这和Naiad相似，其中Vertex T代表task，Edge E代表task间数据channel。如上图例所示，每个operator实例都被封装到对应的task上。Tasks能够被分类为source(没有输入channel)和sink(没有对应的输出channel)。此外，M表示任务在并行执行期间所交换的所有记录的集合，每个task t∈T 都封装了一个独立执行的operator实例，包括如下内容：

1. inputs和outputs的数据channel： I,O⊆E；channel是用来连接上up-stream和down-stream，传输数据流的
2. 当前Task t其中的独立operator实例的状态
3. 结合用户自定义function：f

采用pull的方式获取数据，在执行期间，每个task都会消费inputs记录的，并更新其内部的operator state，然后通过自定义function产生新的记录。更具体的说：对于每个task  t∈T接收到每个记录，会产生一个新的状态s′t，并根据自定义function：ft:st,r →〈s′t,D〉产生一组输出记录。



## 4 异步屏障快照 Asynchronous Barrier Snapshotting， ABS

为了提供持续的结果，分布式处理系统需要对发生failure的task提供容忍性。可以周期性的获取当前execution graph的快照，可用于后续的故障恢复。一个snapshot就是一个execution graph当前的全局snapshot，获取所有必须的信息从指定的执行状态重新计算。

##### 4.1 问题定义

定义了一个Execution graph：G=(T,E)的全局快照G\*=(T\*,E\*)作为其所有task和edge的状态集合，分别用T\*和E\*表示。更详细地说，T\*由所有operator的状态s\*t∈T\*,∀t∈T组成，E\*是所有channel状态的集合e\*∈E\*，而e\*由在e中传输的records组成。

为每个快照G\*保留必要的属性，为了保证恢复的正确结果如所描述的终止（Termination）和可行性（Feasibility):  终止（Termination）保证了一个快照算法在所有进程alive的情况下最终能在有限的时间内完成。可行性（Feasibility）表示快照是有意义的的，即在快照过程中没有丢失有关计算的信息。从形式上讲，这意味着快照中维护了因果顺序，这样task中传递的records也是从快照的角度发送的。

##### 4.2 Acyclic数据流的ABS

在execution拆分成不同的stages时，允许对channel state不做任何持久化snapshot。在这些stages中会将注入的数据流及其关联的计算拆解成一系列的execution，而这些先前的inputs和outputs都以被安全处理。在最后一个stage所有的operator state反映了整个execution的历史，故而能用作snapshot。当前算法的核心就是保持连续数据流入的同时，使用stages snapshot来创建相同的快照。

在当前的算法实现中，通过在连续的数据流中的输入数据流中插入特殊的barrier标记来模拟阶段，这些barrier标记周期性地下发到整个execution graph直至sink。随着每个task接收到execution stage这些barriers，逐步构建全局快照。如下图所示：
![Asynchronous barrier snapshots for acyclic graphs](https://upload-images.jianshu.io/upload_images/5525735-5c77239ce38c8020.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图例简单说明：

如图所示其中的黑色粗线即为barrier，通过在source task摄取输入流时周期性添加到records中，在由channel向下传输，将barrier作为一个特殊的record：在barrier前的records称为presnapshot records，在barrier后的record称为postsnapshot records。当对应的presnapshot records被传递到operator(count)后，其对应的channel(src->count),就只剩下postsnapshot records时，此时对应的channel会处于block，而对应的records就会被存放到buffer中。当某个operator的全部inputs对应的barrier之前的presnapshots全部接收完成后（比如图b中count-1 operator的两个inputs channel: src-1 ---> count-1, src-2 ---->count-1）,对应的channel都处于blocked的，就可以对该task进行snapshot。其他的operator同理。

伪码如下：

```
// init
// 初始化，此时的channel没有被block
upon event 〈Init | inputchannels, outputchannels, fun, init_state〉 do
    state := init_state; 
	  blocked_inputs := ϕ;
    inputs := input_channels;
    outputs := output_channels; 
    udf := fun;

//  接收up-stream的input
// 接收到records中包括barrier的，需要判断当前的channel是否还有内容就是postsnapshot records，有的话当前的channel会被blocked。
// 该task的block的input通道的集合为当前已经block的通道和参数input通道的并集。
// 如果block的input通道等于所有input通道，代表所有input通道都已经被block了，此时触发该task的快照操作，并且把屏障往后广播（即对所有output通道加上这个屏障），然后对所有input通道解除block。
upon event 〈receive|input,〈barrier〉〉> do
    if input ≠ Nil then
      blocked_inputs := blocked_inputs ∪ {input};
      trigger 〈block|input〉;
    if blocked_inputs = inputs then
      blocked_inputs := ϕ;
      broadcast〈send|outputs,〈barrier〉〉;
      trigger〈snapshot|state〉;
      for each inputs as input
         trigger〈unblock|input〉;
         
// state update
// 接收正常的records
// 传入msg，通过UDF计算出结果record和结果状态，并且把结果状态赋值给当前状态，
// 并且把所有结果record往后发送（结果集的每个record对应的output通道不一定是同一个，只逐个往对应的output通道发送
upon event〈receive|input, msg〉do
    {state‘, out_records}:=udf(msg,state);
    state:=state‘;
    for each out_records as {out_put,out_record}
       trigger 〈send|output, outrecord〉;
```

- 网络信道是准可靠的，遵守FIFO传送次序，可以阻塞（*blocked*）和非阻塞（*unblock*）。
  当通道被阻塞（*blocked*）时，所有消息（msg）都被缓冲但在解除阻塞（*unblock*）之前不会继续传递。
- Task可以在它们的通道（*channel*）组件触发（**trigger**）操作如阻塞（*blocked*）、非阻塞（*unblock*）和发送（send）消息。广播（**broadcast**）消息也是在输出通道（*output_channel*）上支持的。
- 在source task上摄取数据流时注入的消息（msg），即消息屏障(barrier)，被解析为“Nil”输入通道（*input_channel*）。

关于ABS算法：由一个Center coordinator周期性的下发state barrier到所有的sources中。当一个source接收到一个barrier时会给当前的state做一个snapshot，然后broadcast这个barrier到所有的outputs。当一个非source task从其某个关联的inputs接收到barrier，该task就会将当前input进行block，直至接收到其关联的所有inputs同一个barrier。一旦该task相关的所有inputs的barrier被接收到，那么该task就会将其当前的state进行snapshot，接着继续broadcast这个barrier到当前task所关联的所有outputs。接着当前task就会取消所有前面被block的inputs channel，继续task的计算。最终的全局快照G\*=(T\*,E\*)是完全由所有E\*=ϕ的operator的状态T\*组成的.

简单论证：如前所述，一个snapshot算法应该保证终止termination和可行性feasibility。Termination由channel和acyclic execution graph来保证。channel的可靠性保证了只要task存活，最终将收到之前发送的每个barrier。 此外，由于始终存在来自source的路径，因此有向无环图（DAG）拓扑中的每个任务task都会从其所有inputs channel接收到barrier并生成snapshot。

至于可行性（Feasibility），它足以表明全局快照中的operator的状态只反映到最后一个stage处理的records的历史。这是由先入先出顺序（FIFO）和屏障上input channel的block来保证的，它确保在快照生成之前没有post-snapshot records会被处理。

##### 4.3 Cyclic数据流的ABS

在execution graph中存在有向循环的情况下，前面提到的ABS算法就不会终止并导致deadlock，因为循环中的tasks会无限期地等待以接收来自所有inputs的barriers。此外在循环内任意传输的records不会包含在快照中，这违反了可行性feasibility。因此，需要一致地将一个周期内生成的所有记录包括在快照中，以便于可行性，并在恢复时将这些记录放回传输中。在处理cyclic graph的方法扩展了基本算法，而不会引入任何额外的通道阻塞。首先通过静态分析，在execution graph的循环中定义back-edges L。根据控制流图理论，在一个有向图中，一个*back-edge*是一个指向已经在深度优先搜索（depth-first search）中被访问过的顶点（vertex）的边（edge）。定义execution graph：G(T, E \ L) 是一个包含拓扑topology中所有task的有向无环图（DAG）。从这个DAG的角度来看，该算法和以前一样工作，不过，我们在快照期间还使用从已定义的*back-edges*接收的记录的下游备份。这是由每个task *t* 实现的，back-edges的一个消费者Lt⊆It,Lt产生一个从Lt转发屏障到接收屏障们回LtLt的备份日志。屏障会push所有在循环中的records进入下游的日志，所以它们在连续不断的快照中只会存在一次。
![Asynchronous barrier snapshots for cyclic](https://upload-images.jianshu.io/upload_images/5525735-93aa66b7c30a66c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图例简单说明：

当前算法实现不同于前面非周期循环数据流的ABS的实现：把循环过的input边当作back-edge，其余边当作regular，除掉循环的DAG依然还是按之前的做法处理，然后有back-edge的边的task，在接收到屏障的时候需要把其state做一个备份，并且接受它的back-edge中在屏障之前的pre-shot record作为log。

在该算法实现中用back-edge作为输入通道的task，一旦它们的常规通道（e∉Le∉L）都接收到了屏障，该task就会产生了一个其状态的本地备份。接下来，从这一点开始，它们记录从back-edges收到的所有record，直到它们收到来自它们的stage屏障（算法第26行）。这就允许，像图3（c）中看到的，所有在循环中的pre-shot record，都会包含在当前快照中。注意，最后的全局快照Gx=(Tx,Lx)Gx=(Tx,Lx) 包含了所有task的状态Tx和在传输中Lx⊂Ex仅仅back-edge中的记录。

1. 对于有 back-edge 输入的节点（后边节点做输入的情况）来说，一旦它所有正常的输入 channel 都收到了这个 barrier，它会先对本地状态做本地 copy；
2. 从这个时间点开始，这个节点会将从 back-edge channel 接收到的所有数据记录下来直到接收到了相应的 barrier，第一步 copy 的状态及第二步记录的数据都会作为 snapshot 的一部分。

伪码如下：

```
upon event〈Init|inputchannels,backedgechannels,outputchannels,fun,initstate〉do
   state:=initstate;
   marked:=ϕ;
   inputs:=input_channels;
   logging:=False;
   outputs:=output_channels;
   udf:=fun;
   loopin_puts:=backedge_channels;
   state_copy:=Nil;
   backup_log:=[];
  
upon event〈receive|input,〈barrier〉〉do
	 marked:=marked ∪ {input};
	 regular:=inputs\loopin_puts;
	 if input ≠ Nil AND input ∉ loop_inputs then
	 		trigger〈block|input〉;
	 if ¬logging AND marked=regular  then
	    state_copy:=state;
	    logging:=True;
	    broadcast〈send|outputs,〈barrier〉〉;
	    for each inputs as input
	       trigger〈unblock|input〉;
	       
	 if marked=input_channels  then
	    trigger〈snapshot|  {statecopy,backuplog}〉;
	    marked:=ϕ;
	    logging:=False;
	    state_copy:=Nil;
	    backup_log:= [];
	    
upon event〈receive|input, msg〉do
	 if logging AND node ∈ loop_inputs then
	    backup_log:=backup_log::[in put];    
	 {state′,outrecords}:=udf(msg,state);
	 state:=state′;
	 for each out_records as {out put,outrecord} 
	    trigger〈send|output, outrecord〉;
```
按照改进后的算法，是可以避免死锁的，这样的话 Termination 的要求是可以满足的；Feasibility 的特性依然是依赖于 channel 的 FIFO 来保证，snapshot 中每个 task state 都会包含该 task 在收到前置节点 barrier 之后的状态，对于有后置节点输入的 task 来说，它会把从后置节点接收到的数据记录下来，只会 copy 非常少量的数据。

## 5. 故障恢复
有了前面的全局一致 snapshot 算法，failover 做起来就简单很多。在 Flink 中，还支持 partial graph recovery，对于失败的 task，只需要恢复它的上游即可，并不需要全局恢复。为了在内部实现 exactly-once，通过给数据进行编号来避免重复数据。
有几种故障恢复方案可以使用这种持续快照。在最简单的形式中，整个执行图可以从上一个全局快照重新启动，如下所示：每个任务t
（1）从持久化存储中检索其快照st的关联状态并将其设置为其初始状态，（2）恢复其备份日志并处理所有其中包含的records，
（3）开始从其输入通道中摄取records。类似于TimeStream [13]，部分图恢复方案也是可行的，通过仅重新安排上游依赖task（输出通道连接失败task的task）以及它们各自的上游任务直到源。 
示例恢复计划如图所示。为了提供exactly-once语义，应在所有下游节点中忽略重复记录以避免重新计算。 为了实现这一目标，我们可以遵循与SDG类似的方案[5]，使用来自源的序列号标记记录，因此，每个下游节点都可以丢弃序列号小于已处理的记录的记录
![故障恢复](https://upload-images.jianshu.io/upload_images/5525735-bca9afdb4324ad3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 6.实现
通过ABS算法为Apache Flink流运行时提供精确的一次处理语义。在我们当前的实现中，阻塞通道将所有传入记录存储在磁盘上，而不是将它们保留在内存中以提高可伸缩性。虽然这种技术确保了鲁棒性，但它增加了ABS算法的运行时间影响。

为了从数据中区分operator状态，我们引入了一个显式的OperatorState接口，该接口包含更新和检查状态的方法。 我们为Apache Flink支持的有状态的运行时operator提供了OperatorState实现，例如基于偏移量的源或聚合。

快照协调是作为JobManager上的参与者进程实现的，它为单个job的执行图保留全局状态。协调器定期向执行图的所有源注入阶段屏障。重新配置后，最后一个全局快照状态将从分布式in-memory的持久化存储中恢复到operator上。

## 性能测试
性能对比了本文提出的 ABS 算法以及 Naiad 中提出的全局同步 snapshot 算法，测试 case 选择了一个有 6 个 operator 的作业，它有三个地方会进行网络 shuffle，这样可以尽量增大 ABS 算法 channel block 带来的影响（如下图 5）。实验中，输入端会模拟 1 百万测试数据，operator 的状态信息主要包括按 key 聚合的中间结果以及 offset 信息，下图的纵坐标是作业运行时间，baseline 表示的是不开启 snapshot 时的性能，在这里做对比使用。

[![性能测试结果](https://upload-images.jianshu.io/upload_images/5525735-4d358212bc7effda.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://matt33.com/images/paper/abs-3.png "性能测试结果") 

如上图 ，可以看到，当 snapshot 时间间隔非常小，同步的 snapshot 性能非常差，因为它在做 snapshot 会阻塞计算，时间都花费在 snapshot 上了，而 ABS 算法的实验结果就好了很多。如上图 ，集群节点及作业并行度从 5 逐渐增加到 40，可以看到 ABS 算法的性能还很稳定的。

## 引用
1.[flink](github.com/apache/flink)
2.[Lightweight Asynchronous Snapshots for Distributed Dataflows](https://arxiv.org/pdf/1506.08603.pdf)
3.[checkpoints](https://ci.apache.org/projects/flink/flink-docs-release-1.9/ops/state/checkpoints.html)
