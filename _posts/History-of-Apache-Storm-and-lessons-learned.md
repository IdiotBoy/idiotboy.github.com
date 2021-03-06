## Apache Storm 的历史与教训
作者：Nathanmarz &nbsp; 译者：徐软件 &nbsp; [[原文链接](http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1)]

Storm 最近成为 Apache 的顶级项目，这对于项目，对于我个人，都有巨大的里程碑意义。想到 Storm 四年前仅仅是我脑海中的一个想法，而今天已经蒸蒸日上，拥有庞大的社区、海量的用户，这一切太疯狂了。在这里，我想回顾一下 Storm 一路走来的历程，还有中间的一些教训。

![storm logo](http://nathanmarz.com/storage/storm_logo.png)

我将讲述在 Storm 发展到某些时间点，不得不面对的关键挑战。本文前 25% 介绍 Storm 的萌芽和初创阶段，一些重要技术问题得以解决，所以 Storm 诞生了；之后的 75% 是发布，然后成为一个被广泛使用的项目，社区中拥有活跃的用户和开发者，会主要介绍市场营销、沟通和社区发展。

任何一个成功的项目都依赖于两个条件：

1. 它确实解决了问题
2. 证明它是很多用户最好的选择

我认为很多开发者都没能真正理解，达到第二个条件与构建项目同样有趣和困难。读完 Storm 的历程，希望你能理解这一点。

### Storm 之前
Storm 起源于我在 BackType 的工作。BackType 的 产品用于帮助商业界分析和理解他们对社会媒体的影响，包括历史上的和实时的。没有 Storm 的时候，我们使用标准的 queue 和 worker 完成实时的部分。例如，我们可以将 Twiiter firehose 写进一个 queue，然后 Python（worker） 可以读取这些 tweet 和处理。通常，worker 可以通过新的queue 发送消息给其他 worker，然后更进一步处理。

我们不满足于这种实现。它不鲁棒 —— 需要确保所有的 queue 和 worker 正常运行，且很难使用。其中大部分困难的逻辑，是如何发送/接受消息，如何序列化/反序列化，等等。真正的业务逻辑占的代码量很少。部署也不优雅 —— 一个应用需要运行在许多 worker 上，但却是一台台分别部署的。似乎应用中的所有逻辑都要非常完备。

### 第一个想法
2010 年 12 月，我有了重大的领悟。就在那时，我想到了一种分布式的抽象模型 —— 流（stream）。流可以并行的产生和处理，但这也可以抽象成一个单独的程序。这让我想到了 spouts 和 bolts —— 一个 spout 是产生最初的流，一个 bolt 读取流，处理并输出新的流。其中最重要的是，spouts 和 bolts 天生是并行的，如果 Hadoop 中的 mappers 和 reducers。Bolts 可以指定输入的流，以及输入时的分组方式。最终，最高层的抽象模型是拓扑（topology）—— 一个包含 spouts 和 bolts 的网络。

在 BackType 的业务中，我尝试了这种抽象模型，结果非常合适。特别是，上文提到的所有繁琐无聊的工作 —— 发送/接受消息、序列化、部署等，都已经被这个抽象模型解决了。

在开始构建 Storm 之前，我想通过更广泛的用户，来证实我的想法。所以我写了一条 tweet：

	I'm working on a new kind of stream processing system. 
	If this sounds interesting to you, ping me. 
	I want to learn your use cases.

	— Nathan Marz (@nathanmarz) December 14, 2010
	
	翻译：
	我正在构建一种新的流式处理系统，如果你感兴趣，请告诉我。
	我希望了解更多的用例。

果然有很多反馈，我们通过邮件沟通。很明显，我的抽象模型非常有效。

接着，我立刻开始 Storm 的设计。但马上也遇到了难题 —— 如何在 spouts 和 bolts 之间传递消息。最初始的想法是，像之前一样模拟 queue 和 worker，使用一个消息代理如 RabbitMQ 传递系统内部的消息。我也确实花了很多时间，深入研究 RabbitMQ，希望能够达成目的。但是，使用消息代理的思想整体上并不合适，这件事先搁置，除非有更好的想法。

### 第二个想法
之所以需要消息代理，是为了给数据处理提供保障。如果一个 bolt 上的某个消息运行失败，该消息将被退回给源头的消息代理，重新尝试运行。然后，关于系统内部的消息代理，仍然有一些事情令人烦恼：

1. 它们太大，移动困难，不得不随着 Storm 一起扩展。
2. 会形成不舒适的情况，比如一个 topology 被重新部署时，会发生什么？消息代理中可能会存在就的消息，与新系统不兼容。这些消息应当被清理，至少是忽略。
3. 使得容错更加困难，因为问题不仅有 Storm 的 worker 死掉的情况，还有消息代理程序死掉的情形。
4. 太慢。并不是直接在 spout 和 bolts 之间传递消息，而是需要经过第三方，更可怕的是，消息还要写磁盘。

我的直觉是，必须得有一种新的消息保护机制，来取代系统内的消息代理。于是我又花了很多时间，沉浸在如何能够保证 spouts 与 bolts 间直接传递消息的可靠性。中间消息没有持久化，那重试功能就得来自数据源，这里指 spouts。可恶的是，在 spout 发出数据之后的下游任何位置，甚至是另一台机器上，都有可能出错；而且又得有精确的数据监测。

苦思冥想数个星期，终于在脑海中闪现了一道灵光。我发明了一种基于随机数和异或位运算的算法，仅需使用 20 字节来跟踪 spout 发出的每个元数据，不论该数据之后会有多少轮的处理。毫无疑问，这个算法是我发明的最好的算法之一；也是我职业生涯中为数不多的，在扎实的计算机科学教育的基础上孕育出来的算法。

当想出这个算法的时刻，我知道我做了一件了不起的事情。该算法避免了所有上述的问题，大大简化了系统设计，并且使得计算性能明显提升。（好笑的是，想出算法的当日，我正在于一个认识不久的女士约会。但刚刚的发现使我过于激动，以至于完全没法分心。不用说，我在那位女士那儿失败了）。

### 构建第一版
之后的 5 个月时间，我完成了第一版的 Storm。一开始，我就想要开源，所以也早早的做了一些重要的决定。第一，Storm 的接口都是 Java，实现却是 Clojure。Java 的接口保证了 Storm 拥有大量的潜在用户；Cloujure 语言，使得实现更加高效，进展神速。

一开始，我也计划 Storm 可以被非 JVM 语言使用。 Topology 使用 Thrift 数据结构定义，提交也使用 Thrift 的接口。另外，我实现了一种协议，使得 spout 和 bolt 能够使用其他语言实现。Storm 支持其他语言，就能有更多的用户。由于 Storm 能够很容易的跟很多项目结合在一起，人们不需要用 Java 重新实现实时计算。他们可以将代码运行在 Storm 的多语言支持接口上。

我使用了很长时间的 Hadoop，Hadoop 的设计理念帮助我把 Storm 设计的更好。比如一个非常重要的问题，在某些情况下 Hadoop 的 worker 执行完成后也不关闭，于是这些进程就傻傻的在那儿。最终这样的僵尸进程越积越多，抢占资源，导致集群丧失运算能力。该问题的原因是 Hadoop 把关闭 worker 的责任交给 worker 自身，然而一些原因会导致无法关闭自身。 于是在 Storm 的设计中，Storm 的守护进程启动 worker，同时也负责关闭 worker。这会更加鲁棒，Storm 绝不会产生僵尸进程。

Hadoop 中另外一个问题是，如果 JobTracker 挂了，所有运行中的作业都会终止。如果你有作业已经跑了很多天，这真会令人抓狂。因为 Storm 是实时计算且运行的是永久性的拓扑，就更不能容忍单点故障问题了。所以 Storm 被设计成了“进程级别的容忍”：如果 Storm 的进程被杀掉，然后重启，原本运行着的拓扑可以毫无影响的继续运行。这对工程是一个巨大的挑战，因为要考虑到进程是用命令“kill -9”杀死的，而且新的进程可以再集群中的任何一台机器中重启，当然，这样也使得一切更加鲁棒。

开发早期的一个关键决定是，给我们的实习生 Jason Jackson 分配了一个任务 —— 在亚马逊网络服务（AWS）中自动部署。这使得测试不同配置和不同大小的集群都更加简单，大大加速了 Storm 的开发。这个工具真的太重要太重要了，使得项目迭代加速非常多。

### 被 Twitter 收购
2011 年 5 月，BackType 被 Twitter 收购。此事对于我意义重大。因为发布 Storm 的时候借用 Twitter 的牌子比 BackType 更容易引发关注。

在收购的谈判期间，我在 BackType 的技术博客上发了一篇文章，向外宣布了 Storm。这篇博文是为了谈判中提升我们的价值，事实上非常有效。Twitter 对此技术非常有兴趣，当他们对我们公司做审查时，几乎变成了 Storm 的一个巨大展示。

那篇博文也有了其他的效应。博文中我随意的将 Storm 比作“实时计算版本的 Hadoop”（the Hadoop of realtime），这个短语变得极为流行。现在依然有人这么说，甚至很多人称之为 “realtime hadoop”。这个意外的标签能量很大，之后有助于被收购。

### Storm 开源
2011 年 7 月，我们正式加入 Twitter，我也计划发布 Storm。

