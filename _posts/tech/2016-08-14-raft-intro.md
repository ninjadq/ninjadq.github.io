---
layout: post
title: Raft 一致性协议简介
category: tech
tags: [distributed system, raft]
---

# Raft算法的介绍

Raft算法现在的出镜率有点高啊，很多热门的技术都用raft算法来解决分布式系统的一致性问题，比如ETCD、TiDB还有各种Docker容器的编排系统。所以也对Raft算法的细节充满的好奇，所以这几天就研究了一下raft算法。

## 一致性（Consensus)
首先需要解释一下什么是一致性（consensus）,它是构建具有容错性（fault-tolerant）的分布式系统的基础。
在一个具有一致性的性质的集群里面，同一时刻所有的结点对存储在其中的某个值都有相同的结果，即对其共享的存储保持一致。

集群具有自动恢复的性质，当少数结点失效的时候不影响集群的正常工作，当大多数集群中的结点失效的时候，集群则会停止服务（不会返回一个错误的结果）。

在一个典型的一致性架构中，整个集群系统是由多个副本状态机（replicated state mathines)组成，对外部客户端的访问，整个系统就想一台非常可靠的状态机。

如下图所示：

![consensus states machine](http://ninjadq.qiniudn.com/raft-intro/raft/replicated-state-machine.png)

系统中每个结点，有三个组件*日志（Log）*、*一致性模块（Concensus Module）*　、*状态机（State Machine）*　三个部分组成。

状态机：当我们说一致性的时候，实际就是在说要保证这个状态机的一致性。状态机会从log里面取出所有的命令，然后执行一遍，得到的结果就是我们对外提供的保证了一致性的数据。

Log：Log中保存了所有的修改记录，比如如果是个hash表的话，就会依次记录各种set、delete信息。

一致性模块：　一致性模块算法就是用来保证写入的log的命令的一致性，这里可以是paxos算法也可以是raft算法。

一旦有多数的服务器是正常工作的，整个系统就会正常运行。

## Raft 算法的设计原则

由于paxos算法晦涩难懂很复杂，所以纯粹的paxos实现并不多，因此Raft提出时的设计原则之一就是其能够更加容易的理解。

我们需要raft算法能够在真实的生产环境能够实现所以在保持正确性、完整性和优秀的性能的同时还需要更容易的理解，所以与paxos的步骤有很大的不同，大大简化了其达到一致状态的机制。

## Raft　算法的总览

Raft算法主要有三个方面，即leader的选举、日志的复制、和安全机制。

1. Leader选举主要的行为是在一个集群中选择一个leader；以及在leader挂了以后重新选举新的leader
2. 日志复制，主要行为是 leader从客户端获得命令，然后将其写到自己的log中；以及将自己的log副本分发给其他的服务器
3. 安全机制，主要是为了保证只有log为最新的结点才能成为leader。

## Raft协议描述

[这里](https://raft.github.io/raftscope-replay/index.html)的动画展示了raft协议的过程，其中包含五个server，分别名为s1到s5，下面部分来解释动画的各个步骤：

选举部分

在最开始是没有leader的，如图所示，每个server外面都有一圈灰色，代表它的计时器，当计时器时间到的时候，就会推荐自己为leader

![初始状态](http://ninjadq.qiniudn.com/raft-intro/election/2-election-start-2.png)

S2服务器计时器最先结束：

![S2 Timeout](http://ninjadq.qiniudn.com/raft-intro/election/3-election-start-s2-tm.png)

S2 发起leader的选举，推荐自己为leader，只要大多数结点响应它，则他就会成为leader

![S2 Start Election](http://ninjadq.qiniudn.com/raft-intro/election/4-election-s2-leader-election.png)

所有的结点都同意S2为leader，所以S2理所当然的成为了leader。
![S2 2b Leader](http://ninjadq.qiniudn.com/raft-intro/election/5-election-s2-2b-leader.png)

Leader 宕机怎么办呢？可以看到现在S2已经宕了。

![S2 Down](http://ninjadq.qiniudn.com/raft-intro/election/6-election-s2-down.png)

每个server的计时器一直都在运行，当S2down了后，一直没有收到S2的信息，自己的计时器则会慢慢的到时，现在S3的计时器已经快要到时间了。

![S3 to](http://ninjadq.qiniudn.com/raft-intro/election/7-election-s3-to.png)

S3时间到，现在发起新一轮的选举，S3想成为leader，现在活着的机器超过半数，所以S3会成功成为leader

![S3 2b leader](http://ninjadq.qiniudn.com/raft-intro/election/8-election-s3-tb-leader.png)

![s3 be a leader](http://ninjadq.qiniudn.com/raft-intro/election/9-s3blearder.png)

如果两个结点同时发生选举，且同时获得相同的投票会发生什么呢？下面可以看到S1和 S5几乎同时发起选举

![s1 s5 to]()
![s1 s5 start election](http://ninjadq.qiniudn.com/raft-intro/election/11-s1_s5_start_election.png)

之后我们可以看到S1和S5全部都有两个投票，这个时候S2的投票至关重要，因为谁拿到S2的票，谁就是leader了，然而S2挂了

![s1 s5 waiting for s2](http://ninjadq.qiniudn.com/raft-intro/election/11-s1-s2-waiting-for-s2.png)

这时候谁都拿不到最后一票，但是其他的结点不会一直等待，可以看到S4的计时器到时，然后发起了新一轮的选举，然后成功成为leader

![s1 s5 wating s2 s3 to](http://ninjadq.qiniudn.com/raft-intro/election/12-s1s5-wating-s2-s3-to.png)
![s5 start election](http://ninjadq.qiniudn.com/raft-intro/election/13-s3-start-election.png)
![s5 be a leader](http://ninjadq.qiniudn.com/raft-intro/election/14-s5-be-a-leader.png)

下面我们来看看日志记录部分

刚开始，log都是空的，大家竞选leader
![start](http://ninjadq.qiniudn.com/raft-intro/log-replica/1-start-election.png)
![s1 to be leader ](http://ninjadq.qiniudn.com/raft-intro/log-replica/1-start-election.png)

s1成为了 leader，同时将自己的log同步到其他节点，两者每一次通信都会同步一次log，直到与leader的log相同。
[s2 sync](http://ninjadq.qiniudn.com/raft-intro/log-replica/2-s2-be-a-leader.png)

但是s2同步了所有的log后，记录还是虚线，因为，每一条log都需要超过半数的结点确认才算真正的在集群中存在。而现在集群中只有两台机器是可用的，少于半数。当第三台机器起来的时候，集群才进入可用状态。图中，S3起来收到了第一个log的信息，S1知道了，所以第一个log就在全局都保持一致了。

![s3 ack](http://ninjadq.qiniudn.com/raft-intro/log-replica/4-ack-s3.png)

接着S1、S2、S3全部达成一致

![sync all](http://ninjadq.qiniudn.com/raft-intro/log-replica/5-sync123.png)

这个时候S1又更新了两条log，但是没有同步到其他的节点，自己却挂了，同时S4活了

![s1 dead s4 alive](http://ninjadq.qiniudn.com/raft-intro/log-replica/6-s1-dead.png)

Leader挂了先进行选举，S2成为了新Leader

![S2 to be leader](http://ninjadq.qiniudn.com/raft-intro/log-replica/7-s2-to-be-leader.png)

经过几轮同步，S4也有了前面三个log，集群又进入可用且一致的状态

![constistent-s234](http://ninjadq.qiniudn.com/raft-intro/log-replica/8-consistent-s234.png)

现在全局达成共识的log有4个，其中一个是在S1挂了后，S2接收的，这个时候S1起来的，它有五个log，只有三个达成了共识

![s1 alive](http://ninjadq.qiniudn.com/raft-intro/log-replica/9-s1-alive.png)

当发生同步的时候，S1发现自己有两个log和全局不一致，所以这时候它会丢弃这两个log，然后与leader同步。

![s1 sync](http://ninjadq.qiniudn.com/raft-intro/log-replica/10-s1-sync.png)


接着是关于安全方面的考虑

下面这个场景，S5起来了，但是没有任何log，且发起选举，如果成为了leader，将会是非常恐怖，因为全局的数据就丢了。

![s5 election](http://ninjadq.qiniudn.com/raft-intro/log-replica/11-s5-election.png)

但是这个不会发生，因为没有其他的结点会给他投票，raft协议的安全机制保证，如果leader选举的节点如果log比投票结点的log信息要老，那么投票结点是不会给他投票的。所以会有新的结点计时器到时，然后发起选举，这里S3发起了选举，然后跟之前的步骤一样，使S5状态同步到全局一致。
![s3 election](http://ninjadq.qiniudn.com/raft-intro/log-replica/12-s3-election.png)
![sync all](http://ninjadq.qiniudn.com/raft-intro/log-replica/13-sync-all.png)
