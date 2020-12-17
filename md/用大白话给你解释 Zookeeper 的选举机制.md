> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=MzU0OTk3ODQ3Ng==&mid=2247490552&idx=1&sn=c8666791054ba7adfac6af2bd338965a&chksm=fba6f3fbccd17aed0c519bf00eb29ba846cdbc983723cada94244abc48a6864b721e211e284b&scene=0&xtrack=1&key=dcf73b7f1e6b1d920b623a7f67cefb31a6f4e11eece456551262ce8cae7a6b65378714c7782afb860a81392f4e76cd896c14567d75a4eeb6b5423eba27f6f7cf80d0ca41d7d172ea4749e3b30d1500968c96c63a2b801327d97507aaab332017fcac849b8448e53fa0d7e38ea3327d515572fc064afdba047717b63eacdc80b1&ascene=1&uin=NzI1NTIwNTYy&devicetype=Windows+XP&version=62060841&lang=zh_CN&exportkey=AY7XXeJIqm%2BKmptJISfg0yc%3D&pass_ticket=Jrj%2BPnSMTW0EH%2B2SwRZrZONOOfYyJsitqg6n33ReFpXobmBB%2FweL%2FP3Fi4vGV21o&wx_header=0)

![](https://mmbiz.qpic.cn/mmbiz_jpg/1J6IbIcPCLb7MHbaKYsNHHcVdjevwsrBeAPfQjibq8v9od4L7YYzVkjb8bxznjicicFMt1yniaHXohjfHlReqpCCZw/640?wx_fmt=jpeg)  

我的新课《C2C 电商系统微服务架构 120 天实战训练营》在公众号**儒猿技术窝**上线了，感兴趣的同学，可以长按扫描下方二维码了解课程详情：  

**课程大纲请参见文末**

![](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLalNxFCJnahictuASd7C1OhpE2bnqmlKyUoywicicHV7f2kq7BOib4pRucD8I0ribKlbwhpibmRPiboZLbyA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLalNxFCJnahictuASd7C1OhpTBgrr9oWQBriaUNH6mA9BUfnsn5XlQ7iaQMYHNvSBQ2tUEryF5vczTHg/640?wx_fmt=png)

转自公众号：爱笑的架构师

`Zookeeper` 是一个分布式服务框架，主要是用来解决分布式应用中遇到的一些数据管理问题如：`统一命名服务`、`状态同步服务`、`集群管理`、`分布式应用配置项的管理`等。

我们可以简单把 `Zookeeper` 理解为分布式家庭的大管家，那么管家团队是如何选出`Leader`的呢？好奇吗，接下来带领大家一探究竟。

人类选举的基本原理
=========

讲解 `Zookeeper` 选举过程前先来介绍一下人类的选举。

我们每个人或多或少都经历过几次选举，在投票的过程中可能会遇到这样几种情况：

**情况 1**：自己与几个候选人都比较熟，你会将票投给你认为`能力比较强的人`；

![](https://mmbiz.qpic.cn/mmbiz_png/RXvHpViaz3EpXZFmBor7Po6SCAW9ibMgIgWicQJ5DI8tmjS8XZ956hG4ZodeSvTwfibEQ9FkIBMQ3Sr8Q6ERVAeZiaQ/640?wx_fmt=png)熟人选举

**情况 2**：自己也是候选人，并且与其他几个候选人都不熟，这个时候你肯定想着要去拉票，因为觉得自己才是最厉害的人呀，所有人都应该把票投给我。但是遗憾的是在拉票的过程中，你`发现别人比你强`，你开始自卑了，最终还是把票投给了自己认为最强的人。

![](https://mmbiz.qpic.cn/mmbiz_png/RXvHpViaz3EpXZFmBor7Po6SCAW9ibMgIgBFHQMTlEcVgzQhbfic12NrC21xN2QYyC3CqicEAhTAqicciaO1laZrNiaQw/640?wx_fmt=png)自己参与选举

所有人都投完票之后，最后从投票箱中进行统计，获得票数最多的人当选。

![](https://mmbiz.qpic.cn/mmbiz_png/RXvHpViaz3EpXZFmBor7Po6SCAW9ibMgIgY3X8TpnPrWZjiaF8fEyb5Jk6vibL74gPqrfBpUibgGUOjD6yUFNWT0mPA/640?wx_fmt=png)思维导图

在整个投票过程中我们可以提炼出四个最核心的概念：

*   `候选人能力`：投票的基本原则是选最强的人。
    
*   `遇强改投`：如果后面发现更强的人可以改投票。
    
*   `投票箱`：所有人的票都会放在投票箱。
    
*   `领导者`：得票最多的人即为领导者。
    

从人类选举的原理我们来简单推导一下 Zookeeper 的选举原理。

Zookeeper 选举的基本原理
=================

> 注意如果 Zookeeper 是单机部署是不需要选举的，集群模式下才需要选举。

Zookeeper 的选举原理和人类选举的逻辑类似，套用一下人类选举的四个基本概念详细解释一下 Zookeeper。

*   **个人能力**
    

如何衡量 Zookeeper 节点个人能力？答案是靠`数据是否够新`，如果节点的数据越新就代表这个节点的个人能力越强，是不是感觉很奇怪，就是这么定的！

在 Zookeeper 中通常是以事务 id（后面简称`zxid`）来标识数据的新旧程度（版本），节点最新的 zxid 越大代表这个节点的数据越新，也就代表这个节点能力越强。

> zxid 的全称是 `ZooKeeper Transaction Id`，即 Zookeeper 事务 id。

*   **遇强改投**
    

在集群选举开始时，节点首先认为自己是最强的（即数据是最新的），然后在选票上写上自己的名字（包括`zxid`和`sid`），zxid 是事务 id，sid 唯一标识自己。

紧接着会将选票传递给其他节点，同时自己也会接收其他节点传过来的选票。每个节点接收到选票后会做比较，这个人是不是比我强（zxid 比我大），如果比较强，那我就需要`改票`，明明别人比我强，我也不能厚着脸皮对吧。

*   **投票箱**
    

与人类选举投票箱稍微有点不一样，Zookeeper 集群会在每个节点的内存中维护一个投票箱。节点会将自己的选票以及其他节点的选票都放在这个投票箱中。由于选票是互相传阅的，所以最终每个节点投票箱中的选票会是一样的。

*   **领导者**
    

在投票的过程中会去统计是否有超过一半的选票和自己选择的是同一个节点，即都认为某个节点是最强的。一旦集群中有`超过半数`的节点都认为某个节点最强，那该节点就是领导者了，投票也宣告结束。

什么场景下 Zookeeper 需要选举？
=====================

当 Zookeeper 集群中的一台服务器出现以下两种情况之一时，需要进入 `Leader 选举`。

（1）服务器初始化启动。

（2）服务器运行期间 Leader 故障。

启动时期的 Leader 选举
---------------

假设一个 Zookeeper 集群中有 5 台服务器，id 从 1 到 5 编号，并且它们都是最新启动的，没有历史数据。

![](https://mmbiz.qpic.cn/mmbiz_png/RXvHpViaz3EpXZFmBor7Po6SCAW9ibMgIglnOXwk3Gsceic0aWhH6FLactKKqcj65htlaIIlFe8tCiaDM4vGPiaic9ibg/640?wx_fmt=png)集群刚启动选举过程  

假设服务器依次启动，我们来分析一下选举过程：

**（1）服务器 1 启动**

发起一次选举，服务器 1 投自己一票，此时服务器 1 票数一票，不够半数以上（3 票），选举无法完成。

投票结果：服务器 1 为 1 票。

服务器 1 状态保持为`LOOKING`。

**（2）服务器 2 启动**

发起一次选举，服务器 1 和 2 分别投自己一票，此时服务器 1 发现服务器 2 的 id 比自己大，更改选票投给服务器 2。

投票结果：服务器 1 为 0 票，服务器 2 为 2 票。

服务器 1，2 状态保持`LOOKING`

**（3）服务器 3 启动**

发起一次选举，服务器 1、2、3 先投自己一票，然后因为服务器 3 的 id 最大，两者更改选票投给为服务器 3；

投票结果：服务器 1 为 0 票，服务器 2 为 0 票，服务器 3 为 3 票。此时服务器 3 的票数已经超过半数（3 票），**服务器 3 当选`Leader`**。

服务器 1，2 更改状态为`FOLLOWING`，服务器 3 更改状态为`LEADING`。

**（4）服务器 4 启动**

发起一次选举，此时服务器 1，2，3 已经不是 LOOKING 状态，不会更改选票信息。交换选票信息结果：服务器 3 为 3 票，服务器 4 为 1 票。此时服务器 4 服从多数，更改选票信息为服务器 3。

服务器 4 并更改状态为`FOLLOWING`。

**（5）服务器 5 启动**

与服务器 4 一样投票给 3，此时服务器 3 一共 5 票，服务器 5 为 0 票。

服务器 5 并更改状态为`FOLLOWING`。

**最终的结果**：

服务器 3 是 `Leader`，状态为 `LEADING`；其余服务器是 `Follower`，状态为 `FOLLOWING`。

运行时期的 Leader 选举
---------------

在 Zookeeper 运行期间 `Leader` 和 `非 Leader` 各司其职，当有非 Leader 服务器宕机或加入不会影响 Leader，但是一旦 Leader 服务器挂了，那么整个 Zookeeper 集群将暂停对外服务，会触发新一轮的选举。

初始状态下服务器 3 当选为`Leader`，假设现在服务器 3 故障宕机了，此时每个服务器上 zxid 可能都不一样，server1 为 99，server2 为 102，server4 为 100，server5 为 101

![](https://mmbiz.qpic.cn/mmbiz_png/RXvHpViaz3EpXZFmBor7Po6SCAW9ibMgIgx0lf2BzgAzJWGyxiaAeQeBow4ibqTwxnR02fV5Gmb8Kx9kjfHoeKg0UA/640?wx_fmt=png)集群 Leader 节点故障

运行期选举与初始状态投票过程基本类似，大致可以分为以下几个步骤：

（1）状态变更。Leader 故障后，余下的`非 Observer` 服务器都会将自己的服务器状态变更为`LOOKING`，然后开始进入`Leader选举过程`。

（2）每个 Server 会发出投票。

（3）接收来自各个服务器的投票，如果其他服务器的数据比自己的新会改投票。

（4）处理和统计投票，每一轮投票结束后都会统计投票，超过半数即可当选。

（5）改变服务器的状态，宣布当选。

话不多说先来一张图：

![](https://mmbiz.qpic.cn/mmbiz_png/RXvHpViaz3EpXZFmBor7Po6SCAW9ibMgIgLxUap97DMSydShdtlSK6SJeO0icibLRxYl5gqdia1HxaYmTssYMnqGsVg/640?wx_fmt=png)运行器 Leader 故障后选举流程

（1）第一次投票，每台机器都会将票投给自己。

（2）接着每台机器都会将自己的投票发给其他机器，如果发现其他机器的 zxid 比自己大，那么就需要改投票重新投一次。比如 server1 收到了三张票，发现 server2 的 xzid 为 102，pk 一下发现自己输了，后面果断改投票选 server2 为老大。

选举机制中涉及到的核心概念
=============

敲黑板了，这些概念是面试必考的。

**（1）Server id（或 sid）：服务器 ID**

比如有三台服务器，编号分别是 1,2,3。编号越大在选择算法中的权重越大，比如初始化启动时就是根据服务器 ID 进行比较。

**（2）Zxid：事务 ID**

服务器中存放的数据的事务 ID，值越大说明数据越新，在选举算法中数据越新权重越大。

**（3）Epoch：逻辑时钟**

也叫投票的次数，同一轮投票过程中的逻辑时钟值是相同的，每投完一次票这个数据就会增加。

**（4）Server 状态：选举状态**

`LOOKING`，竞选状态。

`FOLLOWING`，随从状态，同步 leader 状态，参与投票。

`OBSERVING`，观察状态, 同步 leader 状态，不参与投票。

`LEADING`，领导者状态。

总结
==

（1）Zookeeper 选举会发生在服务器初始状态和运行状态下。

（2）初始状态下会根据服务器 sid 的编号对比，编号越大权值越大，投票过半数即可选出 Leader。

（3）Leader 故障会触发新一轮选举，`zxid` 代表数据越新，权值也就越大。

（4）在运行期选举还可能会遇到脑裂的情况，大家可以自行学习。

**END**

_征稿：愿意技术分享的朋友，欢迎投稿，每篇文章提供 800 ~ 1000 元的稿酬_

_投稿请扫描下方二维码，添加微信：jeversoncui_

![](https://mmbiz.qpic.cn/mmbiz_png/1J6IbIcPCLYaic2t7ffxHnTn6Hc80Jwic9w5icKWNV39kwKHKNshVogm0Korja8ae6Gw5eKxs21ntcxcJIcGBkUSw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_jpg/1J6IbIcPCLalNxFCJnahictuASd7C1Ohpd4jq2icWvJ4NGzoQcMqUmRvucyLsSywGL6EjSrbgj3K0WKER6GO1Wpw/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/1J6IbIcPCLalNxFCJnahictuASd7C1OhpVuVdHMoGUheKboJzzR5hA521UENuWGP1SUWBibLeX4l1rbcia6KA8ZJQ/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/1J6IbIcPCLalNxFCJnahictuASd7C1Ohp2jQ29ggD0BOywydLqNiaLzmIKOkQB9vM5qSFjyMlWN7V5nz4PhnibGCA/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/1J6IbIcPCLalNxFCJnahictuASd7C1Ohp60CmiausZrL5nNBYeeX6zRh1wux0D5TjicdCr2Urib7YWCUPRcx6mMqKg/640?wx_fmt=jpeg)