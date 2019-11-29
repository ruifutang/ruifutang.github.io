---
layout:     post  
title:      paxos 和 raft 该如何选  
subtitle:   那么，区别到底在哪呢
date:       2019-10-19  
author:     唐瑞甫  
header-img: img/post-bg-coffee.jpeg  
catalog: true  
tags:  
    - paxos  
    - raft

---  

本文并不打算介绍 paxos 跟 raft 相关的理论知识。如果大家对 paxos 或者 raft 的基础理论不熟悉或者有遗忘的话，建议去读下 [Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)，[In Search of an Understandable Consensus Algorithm](https://www.usenix.org/system/files/conference/atc14/atc14-paper-ongaro.pdf)。文末也会推荐一些 paxos 和 raft 相关的学习资源。  
  
### 一致性  
paxos 和 raft 都是一致性算法，而且都支持实现强一致性。那么实现一个通用的分布式一致性算法，需要满足什么要求呢？  
  
通常来讲，需要满足两方面的需求： 「safety」和 「liveness」。  
  
**safety** 这个要求是指系统决不能做出不对的事情。具体到 paxos 和 raft，必须保证每一轮的选举过程中最多只有一个提案(entry)被选定，并且一旦选定之后就不会再被更改。
  
而 **liveness** 这个要求是指只要 server 中一半以上（majority）的节点能正常工作，并且消息也能够在合理的延时范围内到达，系统就能正常工作，得出正确的结果。所谓正常工作指的是：总是能最终选定一个值，并且其他节点最终一定能同步到这个值。  
  
这里额外补充说下，paxos 这类一致性算法，解决的是如何在分布式的场景中对于一个/一系列的操作/值达成一致，对于消息在网络中的延时，丢失，重放都是可以接受并处理的，但是这类算法解决不了消息被篡改这种问题，也就是说 「paxos 这类算法无法处理拜占庭问题」。  
  
### 区别  
  
> There is only one consensus protocol, and that's Paxos -all other approaches are just broken versions of Paxos         -- Mike Burrows(author of Google Chubby)  
  
本质上，raft 也是 paxos 算法的一个变种，基于 paxos 增加了一些限制，使其效率更高(增加leader)，更容易被工程化(单写多读)。  
  
![演进图](/img/image/multipaxos.jpg)
  
工程上很少直接使用 basic-paxos ，basic-paxos 本质上只能就**一个**常量(提案或者值) 达成一致。所以一般使用 multi-paxos 的情况会多一些，然而 Leslie Lamport 老爷子并没有在他的大作里面给出具体如何实现的详细描述跟证明，只是简单提了一下：

> To guarantee that all servers execute the same sequence of state machine commands, we implement a sequence of separate instances of the Paxos consensus algorithm, the value chosen by the i-th instance being the i-th state machine command in the sequence.  
  
由于从根源理论上就没有过多的限制，所以在工程实现方面就会有更多的自由。好处是读者可以根据自己的场景进行对应的设计，会有更多创造性的空间。由此带来的坏处是更多得自由以为着更高的出错概率。
  
#### lease-based?  
目前对于 multi-paxos 最大的「误解」，就是认为 multi-paxos 是一个 lease-based 的算法，也就是说 multi-paxos 算法的角色中必须有 leader。  
  
需要明确的是，引入 leader 这个角色，并不是为了解决一致性的问题，而是为了提升性能，因为 basic-paxos 在高并发的情况下可能需要多轮提案才能确定最终的结果，而且有可能出现活锁，导致 basic-paxos的性能不佳。目前很多 multi-paxos 的实现方案引入 leader 只是实现起来方便，而且可以大幅提高性能，因为通过加入一定的限制条件可以在一轮 proposal 的情况下进行多轮 accept ，具体细节这里不做展开，本文的关注点是在 multi-paxos 跟 raft 的区别上。   
  
![](/img/image/paoxs.jpg) 
  
那么什么情况下不适用这种 lease-based 的算法呢？  
  
对系统要求高可用的情况下，不能使用这种基于租约的算法。因为一旦单机出现宕机的情况就需要切主，这时还需要进行选主，这一定会带来不可用的时间。目前 raft-based 系统在单节点异常情况下需要忍受几百毫秒的不可用时间。  
  
而且对于这种 lease-based 的算法，就会出现单节点的瓶颈，一旦 leader 节点出现过载的情况，要么整个系统会被拖垮，要么一部分请求会被拒绝服务。都会影响用户体验。  
  
由于 raft 是基于 lease-based 的算法，而 「multi-paxos 可以是 non-lease 的」。在实现上，鼓励细粒度的容灾单元，能够采取特定的策略提前避免冲突，在产生冲突之后可以进行差异化的避让，这样就能实现一个非租约的 multi-paxos 算法。当其中一个节点出现过载的情况时，该节点只需要将这个请求转发到其他节点一样可以提供服务，这样既实现了单节点的过载保护，整个系统也达到了高可用的目的。  
  
微信在 2016 年的时候就基于非租约的 multi-paxos 算法实现了一个高可用的存储系统 [paxosstore](https://github.com/Tencent/paxosstore)，目前已经在 GitHub 上开源，感兴趣的可以参考。  
  
#### hole  
multi-paxos 跟 raft 还有一个很大的区别就是 Raft要求日志一定是连续，而 multi-paxos则允许日志有空洞。  
  
![](/img/image/raft.jpg)
  
对于 raft 来说，日志的连续性也意味着两个不同节点上相同序号的日志，只要 term 相同，那么这两条日志必然相同，并且之前的日志也必然相同，这样当 leader 节点需要跟 follower 节点同步日志时，非常简洁跟高效。  
  
而且拥有最后一条 commit 的日志也就意味着拥有全部的commit日志。由于最后一条被 commit 的日志，至少被多数派记录。所以一个多数派里面至少有一个节点拥有全部的日志。这样做的好处就是一旦leader节点出现宕机需要切主的时候非常高效，主需要找到最后一条日志term最大且序号最大的节点作为 leader 节点。这里强调下，「高效并不意味着没有延时」，所以 raft 在每次切主的过程中也会出现几百毫秒的不可用的时间。  
  
而对于 multi-paxos 来说，日志是有空洞的，每个日志需要单独被确认是否可以 commit，也可以单独 commit。  
  
如果是基于租约实现的，则每次新 leader 产生后，都需要重新对每个未提交的日志进行确认，已确定它们是否可以被 commit，甚至于新 leader 可能缺失可以被提交的日志，需要通过 Paxos 的 prepare 阶段向其它节点学习到缺失的可以被提交的日志，一种比较好的做法是将这个流程合并到 leader 选举阶段，将所有日志的确认和同步在一次消息通信中进行处理。  
  
如果是基于非租约实现的，当单节点出现宕机后需要恢复时，由于容灾粒度足够下则意味着单条日志几乎都是**独立**的，这样恢复起来就非常高效。以上文中提到的 paxosstore 为例，paxosstore 鼓励细粒度的容灾单元，例如对于账号都建议使用user（对应 uin）作为容灾粒度。于是paxosstore 把 paxos log 做了细粒度的划分，每一个键值/队列/表（统称 entity）由独立的 paxos log 做数据同步，这样单机上可能维护着「互相独立」的数千万个 paxos log，使得宕机后的 catchup 代价降到非常低，尤其对于读多写少的服务，宕机恢复后大部分 entity 根本无需catchup。  
  
### 一些资源  

关于 Raft 的动态[流程图](http://thesecretlivesofdata.com/raft/)  
关于 paxos 我看过最好的[视频](https://www.youtube.com/watch?v=JEpsBg0AO6o)  
关于 paxos 的“[幽灵复现](http://oceanbase.org.cn/?p=111)”问题  
  
### 小结  
  
你会发现 raft 有很多可以优化的地方，然后改着改着，就成了 paxos。
  
---
  By 唐瑞甫  
  2019-10-19

