---
title: Raft算法
date: 2019-03-09 17:53:05
tags:
---

# Raft

Raft是一个<b>分布式共识一致性问题</b>算法。在介绍该算法之前，先来看看著名的拜占庭共识问题。

## 拜占庭将军问题

---
[以下内容引用知乎Tiney熊的回答](https://www.zhihu.com/search?type=content&q=%E6%8B%9C%E5%8D%A0%E5%BA%AD%E5%85%B1%E8%AF%86)

也被称为“拜占庭容错”、“拜占庭将军问题”。拜占庭将军问题是Leslie Lamport（2013年的图灵讲得住）用来为描述分布式系统一致性问题（Distributed Consensus）在论文中抽象出来一个著名的例子。

这个例子大意是这样的：拜占庭帝国想要进攻一个强大的敌人，为此派出了10支军队去包围这个敌人。这个敌人虽不比拜占庭帝国，但也足以抵御5支常规拜占庭军队的同时袭击。这10支军队在分开的包围状态下同时攻击。他们任一支军队单独进攻都毫无胜算，除非有至少6支军队（一半以上）同时袭击才能攻下敌国。他们分散在敌国的四周，依靠通信兵骑马相互通信来协商进攻意向及进攻时间。困扰这些将军的问题是，他们不确定他们中是否有叛徒，叛徒可能擅自变更进攻意向或者进攻时间。在这种状态下，拜占庭将军们才能保证有多于6支军队在同一时间一起发起进攻，从而赢取战斗？

拜占庭将军问题中并不去考虑通信兵是否会被截获或无法传达信息等问题，即消息传递的信道绝无问题。Lamport已经证明了在消息可能丢失的不可靠信道上试图通过消息传递的方式达到一致性是不可能的。所以，在研究拜占庭将军问题的时候，已经假定了信道是没有问题的.

作者：Tiny熊
链接：https://zhuanlan.zhihu.com/p/33666461
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

---

附：[论文](http://lamport.azurewebsites.net/pubs/byz.pdf)

## Raft

Raft要解决的问题就是在信道不可靠的情况下，分布式网络节点之间如何达成共识。

### 角色划分
Raft将分布式网络节点划分为三类角色：

* follower
* candidate
* leader

节点初始角色为follower，角色切换如下图所示：
![](https://upload-images.jianshu.io/upload_images/2736397-458eb385e8ccc1c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/605/format/webp)

### 超时定时器

节点身上绑定一个在（100， 300）ms之间随机的超时定时器，该定时器遇到如下情况时重置超时时间：

* 收到选举请求
* 收到leader心跳

### leader选举

1)
  ![](https://upload-images.jianshu.io/upload_images/2736397-63072559e6b9d35f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/671/format/webp)
  
  节点初始角色为Follower。

2)

  ![](https://upload-images.jianshu.io/upload_images/2736397-c639092cc6cd0804.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/670/format/webp)
  
  若某个节点的超时，则升级为cadidate，率先发起投票请求。

3) 
  ![](https://upload-images.jianshu.io/upload_images/2736397-1ad7ee7ae8fff9cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/668/format/webp)

  若集群中超过半数节点（包括cadidate自己）同意该次投票请求，则cadidate升级成为leader。

选举到此结束。

若之后的时间里，leader不可用(断电、断网等)，则按照如下流程再次选举：

1)

![](https://upload-images.jianshu.io/upload_images/2736397-25775188b6b66321.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/680/format/webp)

2)

剩余节点重复正常的选举流程，选出新leader：
![](https://upload-images.jianshu.io/upload_images/2736397-b0c6f7d0350db3d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/674/format/webp)

3)

新的leader开启心跳广播。之后的时间里，若上一任期leader恢复可用，在收到新任leader的心跳通知之后降级为follower：

![](https://upload-images.jianshu.io/upload_images/2736397-d3e2c1b65e0cd570.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/670/format/webp)
![](https://upload-images.jianshu.io/upload_images/2736397-249223e23550d8eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/676/format/webp)

集群中的节点之间又一次达成共识。

回头来看，节点的超时定时器在150-300ms之间随机，旨在减少多个follower同时（段时间内）升级为cadidate，发投票请求。由于节点之间信道的不可靠，即时每个节点的超时周期随机，也无法保证在第一个cadidate发起的投票请求到来之前第二个follower由于其定时器超时亦升级成为cadidate发出投票请求。

只要超过半数的选票，cadidate就能成功当选，但不可避免的会遇到cadidates之间票数相同的情况。如下图所示：

![](https://upload-images.jianshu.io/upload_images/2736397-235369e90df6c4dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/648/format/webp)
![](https://upload-images.jianshu.io/upload_images/2736397-8a96dd1604c08fc5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/660/format/webp)
![](https://upload-images.jianshu.io/upload_images/2736397-7844d9465c816ada.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/654/format/webp)

每轮选举，只能投票给某个cadidate，不能多投。

![](https://upload-images.jianshu.io/upload_images/2736397-8424138e1c39373d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/648/format/webp)

至此，本轮投票未选出leader，定时器重置，开启新一轮选举。由于超时定时器的随机性，总会有cadidate在接下某轮选举中脱颖而出，再一次的达到共识。

### 日志复制 

数据一致性，是分布式系统需要解决的问题之一。费老劲选举的leader，其核心作用就是保证分布式网络结点之间数据的一致性。

1）

日志初始状态如下：

![](https://upload-images.jianshu.io/upload_images/2736397-2615f4223329848d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/664/format/webp)

2)

客户端写日志的请求，只由leader代理。该日志在leader端缓存，状态为uncommited。

![](https://upload-images.jianshu.io/upload_images/2736397-33453ff94de067d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/668/format/webp)

3)

leader通知所有follower追加日志。

![](https://upload-images.jianshu.io/upload_images/2736397-1251c82292264ef0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/664/format/webp)


4）

follower陆续响应OK，追加的日志在follower端亦为uncommited
![](https://upload-images.jianshu.io/upload_images/2736397-8e4fe60b92e5f4dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/666/format/webp)

5)

等到超过半数的follower追加日志OK之后，leader端响应客户端，并将日志状态置为commited。

![](https://upload-images.jianshu.io/upload_images/2736397-59b9c16018de6a8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/674/format/webp)

至此，日志追加成功。

不可避免的，我们仍然会遇到如下情形：

![](https://upload-images.jianshu.io/upload_images/2736397-3ade6c4d64aea90f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/726/format/webp)

由于网络原因，红线两侧的节点无法通信，导致整个集群被切成两个区。上半区的日志追加，由于节点个数不足（少于3个），追加的日志始终处于uncommited状态。下半区在选取新的leader之后，能正常的追加日志。

如果之后的时间里，网络恢复，则集群状态如下图所示：

![](https://upload-images.jianshu.io/upload_images/2736397-da5f3690cb880c78.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/708/format/webp)

上下区的日志不同步。

待续。