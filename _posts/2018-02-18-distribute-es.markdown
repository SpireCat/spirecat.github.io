---
layout: post
title: 码农孙的笔记(2)——浅析Elasticsearch的优良分布式特性
date: 2018-02-18 00:00:00 +0300
description:
img: es-icon.jpeg # Add image post (optional)
tags: [Record] # add tag
---

&emsp;&emsp;年前在公司参与的是项目内Elasticsearch（以下简称ES）升级改造的相关工作，基础性的知识、概念及用法已经掌握有十之八九了。但由于项目本身是云化的产品，所以在放假这几天读了一些专注于讲解 **ES分布式特性** 的文章，写一下自己的见解。

&emsp;&emsp;实际上，ES在架构上就是基于分布式进行设计的，所以“天生”就是分布式的。我们先来看看ES底层可以自动完成什么工作：

* 将文档分区到不同的的容器或者分片（shards）中，它可以存在于一个或多个节点中
* 将分片均匀分配到各个节点，对索引和搜索做负载均衡
* 冗余每一个分片，防止硬件故障
* 将集群中任意一个节点上的请求路由到相应数据所在的节点
* 增减节点时，无缝扩展和迁移分片

&emsp;&emsp;从本质上来讲，ES的分布式存储是依赖Shards机制来实现的。相对而言，目前项目本身对ES的应用比较基础，所以我们把关注的重点放在一些分布式带来的关键问题——共识问题、并发问题和一致问题——上，而暂时不去关注Shards本身的实现逻辑。

### 共识问题

&emsp;&emsp;对于一个分布式系统而言，使各个节点能够对于某个特定意义的状态和值达成共识是很重要的一件事。ES并没有使用我们所熟知的那些共识算法，如[Raft](https://en.wikipedia.org/wiki/Raft_(computer_science))、 [Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science))等，尽管这些已经广泛运用在了Zookeeper等成熟且优秀的组件中。究其原因，多数人认为ES的master选举场景并不需要Paxos这种功能过于强大的算法，这点我个人认为还需要考证，阅读了ES之父Shay Banon的[文章](https://www.elastic.co/blog/resiliency-elasticsearch)后我觉得更多的考虑的是Paoxs不具备ES所需要的“弹性”。ES实现了自己的共识系统——zen discovery，概括来讲，这个共识系统更类似于一个改良版的[Bully](https://en.wikipedia.org/wiki/Bully_algorithm)算法。在选举master节点时，ES对所有可以成为master的节点根据nodeId排序，每次选举每个节点都把自己所知道节点排一次序，然后选出第一个（第0位）节点，暂且认为它是master节点。

&emsp;&emsp;但是对于一个分布式的集群而言，这种算法在网络状态异常时有一定概率导致“脑裂”问题，即集群将由于选举出多个master节点而分裂。ES提供了minimum_master_nodes的属性来规避这个问题，这个属性限定了成为master节点的限定票数。一般而言，这个值被设定为（可以成为master节点数／2+1）。我们先扒段源码看看：

{% highlight java %}
public boolean hasEnoughMasterNodes(Iterable<DiscoveryNode> nodes) {
        if (minimumMasterNodes < 1) {
            return true;
        }
        int count = 0;
        for (DiscoveryNode node : nodes) {
            if (node.masterNode()) {
                count++;
            }
        }
        return count >= minimumMasterNodes;
    }
{% endhighlight %}

&emsp;&emsp;通过比较节点能检测到的候选master数量和配置的最小值来确定是否可以进行选举，如果数量不够会导致选举不能进行，这样就可以保证集群不会被分裂。使用网络拓扑图举例来看：

<center><img src="/assets/img/master-node.png" height="280" width="407"/></center>

&emsp;&emsp;A节点和B、C节点断开后，肯定是希望“上位”成为master节点的，但是由于A节点能检测到的节点数仅为为1，是小于法定投票数值的，因此上位失败不能成为“正宫”。

### 并发问题

&emsp;&emsp;作为一个分布式系统，处理并发理应是一种本能。当创建/更新/删除请求到达主分片时，它也会被平行地发送到分片副本上。但是，这些请求到达的顺序可能是乱序的。在这种情况下，Elasticsearch使用 **乐观并发控制**，来确保文档的较新版本不会被旧版本覆盖。

&emsp;&emsp;使用乐观锁（Optimistic Lock）是绝大部分分布式系统／应用的第一选择。乐观并发每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号等机制。（是的没错，Git也是这么玩的。）每个被索引的文档都拥有一个版本号，版本号在每次文档变更时递增并应用到文档中。这些版本号用来确保有序接受变更。为了确保在我们的应用中更新不会导致数据丢失，Elasticsearch的API允许我们指定文件的当前版本号，以使变更被接受。如果在请求中指定的版本号比分片上存在的版本号旧，请求失败，这意味着文档已经被另一个进程更新了。如何处理失败的请求，可以在应用层面来控制。

### 一致问题

&emsp;&emsp;一致性的问题在分布式系统中是十分头疼的问题，我们就不扯那个经典的CAP理论来讨论ES的一致性了，已经有很多文献资料进行了细致的讨论，大家想要深入理解的话可以去阿里云栖社区搜索。对于写来说，Elasticsearch对一致性的支持层级与其他大多数数据库不一样，它允许通过使用预检查来看有多少个分片可供写入。可选项有 仲裁集（quorum）、一（one） 和 所有（all）。缺省设置是 仲裁集（quorum），这也就意味着只有当大多数分片可用时，才允许被写入。即使在多数分片可用时，还是会发生向备份分片写入失败的情况，这种情况下，备份分片被认为是有错误的，分片会在另一个节点上重建。

&emsp;&emsp;对于读来说，新文档只有在刷新间隔之后才对搜索可见。为了保证搜索请求返回结果是最新版本的文档，备份可以被设置成为 sync（默认值），当写操作在主备分片同时完成之后才会返回请求的结果。这样，无论搜索请求至哪个分片都会返回最新的文档。甚至如果你的应用要求高索引吞吐率（higher indexing rate）时，replication = async，可以为搜索请求设置 **_preference** 参数为 primary 。这样搜索请求会查询主分片，从而保证结果中的文档是最新版本。

### 小结

&emsp;&emsp;本文所讲的三个问题仅仅从宏观层面概述了一些ES处理分布式系统典型问题的能力，很显然这只是ES诸多优良特性的冰山一角，毕竟这已经是一个可以养活一堆书贩子的产品了。后面有机会还是希望能把ES的更多源码看一看，写的确实牛，个人感觉代码风格很朴实但是很周全。另推荐《深入理解Elasticsearch》这本书。

&emsp;&emsp;溜了溜了，喝酒去了。
