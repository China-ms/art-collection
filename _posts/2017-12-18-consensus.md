---
layout: post
title: 分布式一致性与共识算法
permalink: /consensus
tags: consensus blockchain bitcoin ethereum EOS POW POS DPOS 
toc: true
desc: 这篇文章主要会介绍比特币（Bitcoin）、以太坊（Ethereum）和 EOS 作为一个分布式网络是如何达到分布式一致性的，文章中会从 CAP 理论、拜占庭将军问题以及 FLP 开始介绍分布式一致性相关概念，随后介绍传统分布式系统中的共识算法 Paxos 和 Raft 以及区块链网络中使用工作量证明（POW, Proof-of-Work）、权益证明（POS, Proof-of-Stake）以及委托权益证明（DPOS, Delegated Proof-of-Stake）几种共识算法的原理。
---

+ [分布式一致性与共识算法](https://draveness.me/consensus)
+ [UTXO 与账户余额模型](https://draveness.me/utxo-account-models)

区块链技术是近几年逐渐变得非常热门的技术，以比特币为首的密码货币其实已经被无数人所知晓，但是却很少有人会去研究它们的底层技术，也就是作为一个分布式网络比特币等加密货币是如何工作的。

![cryptocurrency](https://img.draveness.me/2017-12-18-cryptocurrency.png)

无论是 Bitcoin、Ethereum 还是 EOS，作为一个分布式网络，首先需要解决分布式一致性的问题，也就是所有的节点如何对同一个提案或者值达成共识，这一问题在一个所有节点都是可以被信任的分布式集群中都是一个比较难以解决的问题，更不用说在复杂的区块链网络中了。

## 分布式一致性

在一个分布式系统中，如何保证集群中所有节点中的数据完全相同并且能够对某个提案（Proposal）达成一致是分布式系统正常工作的核心问题，而共识算法就是用来保证分布式系统一致性的方法。

![distributed-system](https://img.draveness.me/2017-12-18-distributed-system.png)

然而分布式系统由于引入了多个节点，所以系统中会出现各种非常复杂的情况；随着节点数量的增加，节点失效、故障或者宕机就变成了一件非常常见的事情，解决分布式系统中的各种边界条件和意外情况也增加了解决分布式一致性问题的难度。

![distributed-system-with-failed-nodes](https://img.draveness.me/2017-12-18-distributed-system-with-failed-nodes.png)

在一个分布式系统中，除了节点的失效是会导致一致性不容易达成的主要原因之外，节点之间的网络通信收到干扰甚至阻断以及分布式系统的运行速度的差异都是解决分布式系统一致性所面临的难题。

#### CAP

在 1998 年的秋天，加州伯克利大学的教授 Eric Brewer 第一次发布了 CAP 理论，在 1999 年论文 [Brewer’s Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.67.6951&rep=rep1&type=pdf) 正式发布，其中总结了 Eric Brewer 提出的 CAP 理论。

![cap-theorem](https://img.draveness.me/2017-12-18-cap-theorem.png)

这篇论文证明了两个非常有意思的理论，首先是在异步的网络模型中，所有的节点由于没有时钟仅仅能根据接收到的消息作出判断，这时完全不能同时保证一致性、可用性和分区容错性，每一个系统只能在这三种特性中选择两种。

不过这里讨论的一致性其实都是强一致性，也就是所有节点接收到同样的操作时会按照完全相同的顺序执行，被一个节点提交的更新操作会立刻反映在其他通过**异步或部分同步网络**连接的节点上，如果想要同时满足一致性和分区容错性，在异步的网络中，我们只能**中心化存储所有数据**，通过其他节点将请求路由给中心节点达到这两个目的。

![consistency-and-partition-tolerant](https://img.draveness.me/2017-12-18-consistency-and-partition-tolerant.png)

但是在现实世界中其实并不存在**绝对**异步的网络环境，如果我们允许每一个节点拥有自己的时钟，这些时钟虽然有着完全不同的时间，但是它们的**更新频率是完全相同的**，所以我们可以通过时钟得知接收消息的间隔时间，在这种更宽松的前提下，我们能够得到更强大的服务。

然而在部分同步的网络环境中，我们仍然没有办法同时保证一致性、可用性和分区容错性，证明的过程其实非常简单，可以直接阅读 [论文](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.67.6951&rep=rep1&type=pdf) 的 4.2 节，然而时钟的出现能够让我们知道当前消息有多久没有得到回应，通过超时时间就能在一定程度上解决信息丢失的问题。

由于网络一定会存在延时，所以没有办法在分布式系统中做到强一致性的同时保证可用性，不过我们可以通过降低对一致性的要求，在一致性和可用性之间做出权衡，而这其实也是设计分布式系统首先需要考虑的问题，由于强一致性的系统会导致系统的可用性降低，仅仅将接受请求的工作交给其他节点对于高并发的服务并不能解决问题，所以在目前主流的分布式系统中都选择*最终一致性*。

![eventually-consistency](https://img.draveness.me/2017-12-18-eventually-consistency.png)

最终一致性允许多个节点的状态出现冲突，但是所有能够沟通的节点都能够在有限的时间内解决冲突，从不一致的状态恢复到一致，这里列出的两个条件比较重要，一是节点直接可以**正常通信**，二是冲突需要在**有限的时间**内解决，只有在这两个条件成立时才能达到最终一致性。

#### 拜占庭将军问题

拜占庭将军问题是 Leslie Lamport 在 [The Byzantine Generals Problem](https://web.archive.org/web/20170205142845/http://lamport.azurewebsites.net/pubs/byz.pdf) 论文中提出的分布式领域的容错问题，它是分布式领域中最复杂、最严格的容错模型。

在该模型下，系统不会对集群中的节点做任何的限制，它们可以向其他节点发送随机数据、错误数据，也可以选择不响应其他节点的请求，这些无法预测的行为使得容错这一问题变得更加复杂。

拜占庭将军问题描述了一个如下的场景，有一组将军分别指挥一部分军队，每一个将军都不知道其它将军是否是可靠的，也不知道其他将军传递的信息是否可靠，但是它们需要通过投票选择是否要进攻或者撤退：

![byzantine-generals-problem](https://img.draveness.me/2017-12-18-byzantine-generals-problem.png)

> 在这一节中，黄色代表状态未知，绿色代表进攻，蓝色代表撤退，最后红色代表当前将军的信息不可靠。

在这时，无论将军是否可靠，只要所有的将军达成了统一的方案，选择进攻或者撤退其实就是没有任何问题的：

![byzantine-generals-problem-with-plans](https://img.draveness.me/2017-12-18-byzantine-generals-problem-with-plans.png)

上述的情况不会对当前的战局有太多的影响，也不会造成损失，但是如果其中的一个将军告诉其中一部分将军选择进攻、另一部分选择撤退，就会出现非常严重的问题了。

![byzantine-generals-problem-split-votes](https://img.draveness.me/2017-12-18-byzantine-generals-problem-split-votes.png)

由于将军的队伍中出了一个叛徒或者信息在传递的过程中被拦截，会导致一部分将军会选择进攻，剩下的一部分会选择撤退，它们都认为自己的选择是大多数人的选择，这时就出现了严重的不一致问题。

拜占庭将军问题是对分布式系统容错的最高要求，然而这不是日常工作中使用的大多数分布式系统中会面对的问题，我们遇到更多的还是节点故障宕机或者不响应等情况，这就大大简化了系统对容错的要求；不过类似 Bitcoin、Ethereum 等分布式系统确实需要考虑拜占庭容错的问题，我们会在下面介绍它们是如何解决的。

#### FLP

FLP 不可能定理是分布式系统领域最重要的定理之一，它给出了一个非常重要的结论：**在网络可靠并且存在节点失效的异步模型系统中，不存在一个可以解决一致性问题的确定性算法**。

> In this paper, we show the surprising result that no completely asynchronous consensus protocol can tolerate even a single unannounced process death. We do not consider Byzantine failures, and we assume that the message system is reliable it delivers all messages correctly and exactly once.

这个定理其实也就是告诉我们不要浪费时间去为异步分布式系统设计在任意场景上都能够实现共识的算法，异步系统完全没有办法保证能在有限时间内达成一致，在这里作者并不会尝试去证明 FLP 不可能定理，读者可以阅读相关论文 [Impossibility of Distributed Consensuswith One Faulty Process](https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf) 了解更多的内容。

## 共识算法

在上一节中，我们已经简单了解了分布式系统中面对的问题与挑战，在这里我们会介绍不同共识算法的实现原理，包括传统分布式系统领域的 Paxos、Raft 以及密码货币中使用的工作量证明（POW）、权益证明（POS）和委托权益证明（DPOS），通过对这些共识算法原理的介绍和分析，我相信各位读者能对分布式一致性和共识算法有更深的理解。

### Paxos 和 Raft

Paxos 和 Raft 是目前分布式系统领域中两种非常著名的解决一致性问题的共识算法，两者都能解决分布式系统中的一致性问题，但是前者的实现与证明非常难以理解，后者的实现比较简洁并且遵循人的直觉，它的出现就是为了解决 Paxos 难以理解并和难以实现的问题。

![paxos-raft](https://img.draveness.me/2017-12-18-paxos-raft.png)

我们先来简单介绍一下 [Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science)) 究竟是什么，Paxos 其实是**一类**能够解决分布式一致性问题的协议，它能够让分布式网络中的节点在出现错误时仍然保持一致；Leslie Lamport 提出的 Paxos 可以在没有恶意节点的前提下保证系统中节点的一致性，也是第一个被证明完备的共识算法，目前的**完备的**共识算法包括 Raft 本质上都是 Paxos 的变种。

作为一类协议，Paxos 中包含 Basic Paxos、Multi-Paxos、Cheap Paxos 和其他的变种，在这一小节我们会简单介绍 Basic Paxos 和 Multi-Paxos 这两种协议。

#### Basic Paxos

Basic Paxos 是 Paxos 中最为基础的协议，每一个 Basic Paxos 的协议实例最终都会选择唯一一个结果；使用 Paxos 作为共识算法的分布式系统中，节点都会有三种身份，分别是 Proposer、Acceptor 和 Learner：

![paxos-roles](https://img.draveness.me/2017-12-18-paxos-roles.png)

我们在这里会忽略最后一种身份 Learner 简化协议的运行过程，便于读者理解；Paxos 的运行过程分为两个阶段，分别是准备阶段（Prepare）和接受阶段（Accept），当 Proposer 接收到来自客户端的请求时，就会进入如下流程：

![basic-paxos-phases](https://img.draveness.me/2017-12-18-basic-paxos-phases.png)

> 以上截图取自 [Paxos lecture (Raft user study)](https://ramcloud.stanford.edu/~ongaro/userstudy/paxos.pdf) 的第 12 页。

在整个共识算法运行的过程中，Proposer 负责提出提案并向 Acceptor 分别发出两次 RPC 请求，Prepare 和 Accept；Acceptor 会根据其持有的信息 `minProposal`、`acceptedProposal` 和 `acceptedValue` 选择接受或者拒绝当前的提案，当某一个提案被过半数的 Acceptor 接受之后，我们就认为当前提案被整个集群接受了。

![paxos-example](https://img.draveness.me/2017-12-18-paxos-example.png)

我们简单举一个例子介绍 Paxos 是如何在多个提案下保证最终能够达到一致性的，上述图片中 S1 和 S5 分别收到了来自客户端的请求 X 和 Y，S1 首先向 S2 和 S3 发出 Prepare RPC 和 Accept RPC，三个服务器都接受了 S1 的提案 X；在这之后，S5 向 S3 和 S4 服务器发出 `Prepare(2.5)` 的请求，S3 由于已经接受了 X，所以它会返回接受的提案和值 `(1.1, X)`，这时服务器使用接收到的提案代替自己的提案 Y，重新向其他服务器发送 `Accept(2.5, X)` 的 RPC，最终所有的服务器会达成一致并选择相同的值。

> 想要了解更多与 Paxos 协议在运行过程中的其他情况可以看一下 [Paxos lecture (Raft user study)](https://www.youtube.com/watch?v=JEpsBg0AO6o) 视频。

#### Multi-Paxos

由于大多数的分布式集群都需要接受一系列的值，如果使用 Basic Paxos 来处理数据流，那么就会导致非常明显的性能损失，而 Multi-Paxos 是前者的加强版，如果集群中的 Leader 是非常稳定的，那么我们往往不需要准备阶段的工作，这样就能够将 RPC 的数量减少一半。

![multi-paxos-example](https://img.draveness.me/2017-12-18-multi-paxos-example.png)

上述图片中描述的就是稳定阶段 Multi-Paxos 的处理过程，S1 是整个集群的 Leader，当其他的服务器接收到来自客户端的请求时，都会将请求转发给 Leader 进行处理。

当然，Leader 角色的出现自然会带来另一个问题，也就是 Leader 究竟应该如何选举，在 [Paxos Made Simple](http://140.123.102.14:8080/reportSys/file/paper/lei/lei_5_paper.pdf) 一文中并没有给出 Multi-Paxos 的具体实现方法和细节，所以不同 Multi-Paxos 的实现上总有各种各样细微的差别。

#### Raft

Raft 其实就是 Multi-Paxos 的一个变种，Raft 通过简化 Multi-Paxos 的模型，实现了一种更容易让人理解的共识算法，它们两者都能够对一系列连续的问题达成一致。

Raft 在 Multi-Paxos 的基础之上做了两个限制，首先是 Raft 中追加日志的操作必须是连续的，而 Multi-Paxos 中追加日志的操作是并发的，但是对于节点内部的状态机来说两者都是有序的，第二就是 Raft 对 Leader 选举的条件做了限制，只有拥有最新、最全日志的节点才能够当选 Leader，但是 Multi-Paxos 由于任意节点都可以写日志，所以在选择 Leader 上也没有什么限制，只是在选择 Leader 之后需要将 Leader 中的日志补全。

![multi-paxos-and-raft-log](https://img.draveness.me/2017-12-18-multi-paxos-and-raft-log.png)

在 Raft 中，所有 Follower 的日志都是 Leader 的子集，而 Multi-Paxos 中的日志并不会做这个保证，由于 Raft 对日志追加的方式和选举过程进行了限制，所以在实现上会更加容易和简单。

从理论上来讲，支持并发日志追加的 Paxos 会比 Raft 有更优秀的性能，不过其理解和实现上还是比较复杂的，很多人都会说 Paxos 是科学，而 Raft 是工程，当作者需要去实现一个共识算法，会选择使用 Raft 和更简洁的实现，避免因为一些边界条件而带来的复杂问题。

> 这篇文章并不会展开介绍 Raft 的实现过程和细节，如果对 Raft 有兴趣的读者可以在 [The Raft Consensus Algorithm](https://raft.github.io) 找到非常多的资料。

### POW(Proof-of-Work)

上一节介绍的共识算法，无论是 Paxos 还是 Raft 其实都只能解决非拜占庭将军容错的一致性问题，不能够应对分布式网络中出现的极端情况，但是这在传统的分布式系统都不是什么问题，无论是分布式数据库还是消息队列集群，它们内部的节点并不会故意的发送错误信息，在类似系统中，最常见的问题就是节点失去响应或者失效，所以它们在这种前提下是有效可行的，也是充分的。

这一节介绍的 [工作量证明](https://en.wikipedia.org/wiki/Proof-of-work_system)（POW，Proof-of-Work）是一个用于阻止拒绝服务攻击和类似垃圾邮件等服务错误问题的协议，它在 1993 年被 Cynthia Dwork 和 Moni Naor 提出，它能够帮助分布式系统达到拜占庭容错。

![proof-of-work-puzzle](https://img.draveness.me/2017-12-18-proof-of-work-puzzle.png)

工作量证明的关键特点就是，分布式系统中的请求服务的节点必须解决一个**一般难度但是可行（feasible）的问题**，但是验证问题答案的过程对于服务提供者来说却非常容易，也就是一个不容易解答但是容易验证的问题。

这种问题通常需要消耗一定的 CPU 时间来计算某个问题的答案，目前最大的区块链网络 - 比特币（Bitcoin）就使用了工作量证明的分布式一致性算法，网络中的所有节点计算通过以下的谜题来获得当前区块的记账权：

![bitcoin-puzzle](https://img.draveness.me/2017-12-18-bitcoin-puzzle.png)

SHA-256 作为一个哈希函数，想要通过 SHA-256 函数的输出推断出输入在目前来看可能性是**可以忽略不计的**，比特币网络就需要每一个节点不断改变 `NONCE` 来得到不同的结果 `HASH`，如果得到的 HASH 结果在小于某个范围，目前（2017.12.17）的难度是：

```c
0x0000000000000000000000000000000000000000000000000000017268d8a21a
```

也就是如果只计算一次 SHA-256 的值能够小于上述结果的可能性是 $$1.37*10^{-65}$$，当前的全网算力也达到了 13,919 PH/s，这是一个非常恐怖的数字，随着网络算力的不断改变比特币也会不断改变当前问题的难度，保证每个区块被发现的时间在 10min 左右；在整个比特币网络中，谁先得到当前问题的答案就能够获得这个区块的记账权并将当前区块通过 Gossip 协议发送给尽可能多的比特币节点。

工作量证明的原理其实非常简单，比特币网络选择的谜题非常好的适应了工作量证明定义中的问题，比较难以寻找同时又易于证明，我们可以简单理解为工作量证明防止错误或者无效请求的原理就是增加客户端请求服务的工作量，而适合难度的谜题又能够保证合法的请求不会受到影响。

由于工作量证明需要消耗大量的算力，同时比特币大约 10min 才会产生一个区块，区块的大小也只有 1MB，仅仅能够包含 3、4000 笔交易，平均下来每秒只能够处理 5~7（个位数）笔交易，所以比特币网络的拥堵状况非常严重。

### POS(Proof-of-Stake)

权益证明是区块链网络中的使用的另一种共识算法，在基于权益证明的密码货币中，下一个区块的选择是根据不同节点的股份和时间进行随机选择的。

由于创造新的区块并不会消耗大量的 CPU，如果它不诚实也不会失去什么，这也就给了很多节点作弊的理由，每一个节点为了最大化利益会在多条链上同时挖矿。

![pos-problem](https://img.draveness.me/2017-12-18-pos-problem.png)

在早期的所有权证明算法中，整个网络只会奖励创建区块的节点，不存在任何惩罚，在这时每个节点在创造的多条链上同时投票才能够最大化利益，在这种情况下网络中的节点很难对一条链达成共识。

有两种办法能够解决缺乏利害关系（nothing-at-stake）造成的问题，一种是使用 [Slasher](https://blog.ethereum.org/2014/01/15/slasher-a-punitive-proof-of-stake-algorithm/) 协议，惩罚同时在多条链上投票的节点，第二种方法时直接惩罚在错误的链上创建块的节点，总而言之就是通过算法之外的事情解决这个问题，引入激励和惩罚。

与工作量证明相比，权益证明不需要消耗大量的电力就能够保证区块链网络的安全性，同时也不需要在每个区块中创建新的货币来激励矿工参与当前网络的运行，这也就在一定程度上缩短了达成共识所需要的时间，基于权益证明的 Ethereum 每秒大概能处理 30 笔交易左右。

### DPOS(Delegated Proof-of-Stake)

前面介绍的权益证明算法可以将整个区块链网络理解为一家公司，出资最多、占比最大的人有更多的机会得到话语权（记账权）；对于小股东来说，千分之几甚至万分之几的股份很难有什么作为，只能得到股份带来的分红和收益。

但是在这里介绍的委托权益证明（DPOS，Delegated Proof-of-Stake）能够让每一个人选出可以代表自己利益的人参与到记账权的争夺中，这样多个小股东就能够通过投票选出自己的代理人，保障自己的利益。整个网络中选举出的多个节点能够在 1s 中之内对 99.9% 的交易进行确认，使用委托权益证明的 EOS 能够每秒处理几十万笔交易，同时也能够比较监管的干预。

![delegated-proof-of-stake-witnesses](https://img.draveness.me/2017-12-18-delegated-proof-of-stake-witnesses.png)

在委托权益证明中，每一个参与者都能够选举任意数量的节点生成下一个区块，得票最多的前 N 个节点会被选择成为区块的创建者，下一个区块的创建者就会从这样一组当选者中随机选取，除此之外，N 的数量也是由整个网络投票决定的，所以可以尽可能地保证网络的去中心化。

## 总结

在这篇文章中，我们首先介绍了分布式系统中面对的最重要问题，分布式一致性，随后又介绍了五种不同的共识算法，从解决非拜占庭问题下一致性的 Paxos 和 Raft 到解决拜占庭问题下的 POW、POS 和 DPOS，简单回忆一下，解决拜占庭问题的多个共识算法的实现反而更加简单，这是一件非常有意思的事情。

当整个网络需要实现拜占庭容错时，仅靠算法确实是比较难以实现的，往往都需要使用其他方面的激励或者惩罚，让诚实表现的节点利益最大化才是解决一致性的最佳方案；从工作量证明、权益证明再到委托权益证明，共识算法的不同导致性能也有着非常大的差异，我们可以看到随着网络中进行『投票』的节点越少，网络的处理能力就会越强和性能就会越快，委托权益证明选举了 N 个节点来保证性能和去中心化程度确实是一件非常聪明的事情。

中心化的网络确实能够带来性能的提升，但是在密码货币中，参与者往往更相信去中心化的机制，因为没有激励和惩罚我们并不能保证下一个负责记账的节点是否是诚实的，由此来看，如何在保证去中心化的同时提升网络的性能是每一个区块链网络都需要考虑的事情。

## Reference

+ [Consensus (computer science)](https://en.wikipedia.org/wiki/Consensus_(computer_science))
+ [区块链共识算法（POW,POS,DPOS,PBFT）介绍和心得](http://blog.csdn.net/lsttoy/article/details/61624287)
+ [Paxos 与 Raft](https://yeasy.gitbooks.io/blockchain_guide/content/distribute_system/paxos.html)
+ [Proof-of-work system](https://en.wikipedia.org/wiki/Proof-of-work_system)
+ [Proof-of-stake](https://en.wikipedia.org/wiki/Proof-of-stake)
+ [Proof of Stake FAQ · Ethereum Wiki](https://github.com/ethereum/wiki/wiki/Proof-of-Stake-FAQ)
+ [Delegated Proof of Stake](http://docs.bitshares.org/bitshares/dpos.html)
+ [Delegated Proof-of-Stake Consensus](https://bitshares.org/technology/delegated-proof-of-stake-consensus/)
+ [DPOS共识算法 -- 缺失的白皮书](https://steemit.com/dpos/@legendx/dpos)
+ [共识算法](https://yeasy.gitbooks.io/blockchain_guide/content/distribute_system/consensus.html)
+ [CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem)
+ [Paxos Made Simple](http://140.123.102.14:8080/reportSys/file/paper/lei/lei_5_paper.pdf)
+ [The Raft Consensus Algorithm](https://raft.github.io)
+ [In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)
+ [谈谈 paxos, multi-paxos, raft](https://baotiao.github.io/2016/05/05/paxos-raft/)
+ [Brewer’s Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.67.6951&rep=rep1&type=pdf)
+ [“Eventual Consistency” vs “Strong Eventual Consistency” vs “Strong Consistency”?](https://stackoverflow.com/questions/29381442/eventual-consistency-vs-strong-eventual-consistency-vs-strong-consistency)
+ [Impossibility of Distributed Consensuswith One Faulty Process](https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf)
+ [The Byzantine Generals Problem](https://web.archive.org/web/20170205142845/http://lamport.azurewebsites.net/pubs/byz.pdf)
+ [Byzantine fault tolerance](https://en.wikipedia.org/wiki/Byzantine_fault_tolerance)
+ [A Brief Tour of FLP Impossibility](http://the-paper-trail.org/blog/a-brief-tour-of-flp-impossibility/)
+ [Paxos Made Simple](http://140.123.102.14:8080/reportSys/file/paper/lei/lei_5_paper.pdf)
+ [Neat Algorithms - Paxos](http://harry.me/blog/2014/12/27/neat-algorithms-paxos/)
+ [FLP Impossibility 的证明](http://danielw.cn/FLP-proof)
+ [白话区块链](https://read.douban.com/ebook/42852957/)
+ [Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science))
+ [Paxos lecture (Raft user study)](https://www.youtube.com/watch?v=JEpsBg0AO6o)
+ [Bitcoin: A Peer-to-Peer Electronic Cash System](https://bitcoin.org/bitcoin.pdf)
+ [Proof of Stake · Bitcoin Wiki](https://en.bitcoin.it/wiki/Proof_of_Stake)
+ [Slasher](https://blog.ethereum.org/2014/01/15/slasher-a-punitive-proof-of-stake-algorithm/)
+ [Proof of Stake FAQ](https://github.com/ethereum/wiki/wiki/Proof-of-Stake-FAQ)


