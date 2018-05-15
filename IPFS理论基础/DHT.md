##  分布式散列表

（英语：distributed hash table，缩写**DHT**）是[分布式计算](https://zh.wikipedia.org/wiki/%E5%88%86%E6%95%A3%E5%BC%8F%E8%A8%88%E7%AE%97)系统中的一类，用来将一个关键值（key）的集合分散到所有在分布式系统中的节点，并且可以有效地将消息转送到唯一一个拥有查询者提供的关键值的节点（Peers）。这里的节点类似[散列表](https://zh.wikipedia.org/wiki/%E9%9B%9C%E6%B9%8A%E8%A1%A8)中的存储位置。分布式散列表通常是为了拥有极大节点数量的系统，而且在系统的节点常常会加入或离开（例如网络断线）而设计的。在一个结构性的[延展网络](https://zh.wikipedia.org/w/index.php?title=%E5%BB%B6%E5%B1%95%E7%B6%B2%E8%B7%AF&action=edit&redlink=1)（[overlay network](https://zh.wikipedia.org/wiki/Overlay_network)）中，参加的节点需要与系统中一小部分的节点沟通，这也需要使用分布式散列表。分布式散列表可以用以创建更复杂的服务，例如[分布式文件系统](https://zh.wikipedia.org/wiki/%E5%88%86%E6%95%A3%E5%BC%8F%E6%AA%94%E6%A1%88%E7%B3%BB%E7%B5%B1)、[点对点技术](https://zh.wikipedia.org/wiki/%E9%BB%9E%E5%B0%8D%E9%BB%9E%E6%8A%80%E8%A1%93)[文件分享](https://zh.wikipedia.org/wiki/%E6%AA%94%E6%A1%88%E5%88%86%E4%BA%AB)系统、合作的[网页缓存](https://zh.wikipedia.org/w/index.php?title=%E7%B6%B2%E9%A0%81%E5%BF%AB%E5%8F%96&action=edit&redlink=1)、[多播](https://zh.wikipedia.org/wiki/%E5%A4%9A%E6%92%AD)、[任播](https://zh.wikipedia.org/wiki/%E4%BB%BB%E6%92%AD)、[域名系统](https://zh.wikipedia.org/wiki/%E7%B6%B2%E5%9F%9F%E5%90%8D%E7%A8%B1%E7%B3%BB%E7%B5%B1)以及[即时通信](https://zh.wikipedia.org/wiki/%E5%8D%B3%E6%99%82%E9%80%9A%E8%A8%8A)等。

## 发展背景

研究分布式散列表的主要动机是为了开发点对点系统，像是[Napster](https://zh.wikipedia.org/wiki/Napster)、[Gnutella](https://zh.wikipedia.org/wiki/Gnutella)及[Freenet](https://zh.wikipedia.org/wiki/Freenet)。这些系统得益于使用分散在[互联网](https://zh.wikipedia.org/wiki/%E7%B6%B2%E9%9A%9B%E7%B6%B2%E8%B7%AF)上的各项资源以提供实用的应用，特别在[带宽](https://zh.wikipedia.org/wiki/%E9%A0%BB%E5%AF%AC)及[硬盘](https://zh.wikipedia.org/wiki/%E7%A1%AC%E7%A2%9F)存储空间上，他们所提供的文件分享功能因此得到最大的好处。

这些系统使用不同的方法来解决如何**找到拥有某数据的节点**的问题。Napster使用中央的索引服务器：每个节点加入网络的同时，会将他们所拥有的文件列表发送给服务器，这使得服务器可以进行搜索并将结果回传给进行查询的节点。但中央索引服务器让整个系统易受攻击，且可能造成法律问题。于是，Gnutella和相似的网络改用大量查询模式（flooding query model）：每次搜索都会把查询消息广播给网络上的所有节点。虽然这个方式能够防止[单点故障](https://zh.wikipedia.org/wiki/%E5%8D%95%E7%82%B9%E6%95%85%E9%9A%9C)（[single point of failure](https://zh.wikipedia.org/wiki/%E5%8D%95%E7%82%B9%E6%95%85%E9%9A%9C)），但比起Napster来说却极没效率。

最后，Freenet使用了完全分布式的系统，但它建置了一套使用经验法则的[基于关键值的转送方法](https://zh.wikipedia.org/w/index.php?title=%E5%9F%BA%E6%96%BC%E9%97%9C%E9%8D%B5%E5%80%BC%E7%9A%84%E8%BD%89%E9%80%81%E6%96%B9%E6%B3%95&action=edit&redlink=1)（[key based routing](https://zh.wikipedia.org/w/index.php?title=Key_based_routing&action=edit&redlink=1)）。在这个方法中，每个文件与一个关键值相结合，而拥有相似关键值的文件会倾向被相似的节点构成的集合所保管。于是查询消息就可以根据它所提供的关键值被转送到该集合，而不需要经过所有的节点。然而，Freenet并不保证存在网络上的数据在查询时一定会被找到。

分布式散列表为了达到Gnutella与Freenet的分散性（decentralization）以及Napster的效率与正确结果，使用了较为结构化的基于关键值的转送方法。不过分布式散列表也有个Freenet有的缺点，就是只能作精确搜索，而不能只提供部分的关键字；但这个功能可以在分布式散列表的上层实做。

最初的四项分布式散列表技术——[内容可定址网络](https://zh.wikipedia.org/w/index.php?title=%E5%85%A7%E5%AE%B9%E5%8F%AF%E5%AE%9A%E5%9D%80%E7%B6%B2%E8%B7%AF&action=edit&redlink=1)（[Content addressable network](https://zh.wikipedia.org/w/index.php?title=Content_addressable_network&action=edit&redlink=1)，CAN）、[Chord](https://zh.wikipedia.org/w/index.php?title=Chord&action=edit&redlink=1)（[Chord project](https://zh.wikipedia.org/w/index.php?title=Chord_project&action=edit&redlink=1)）[[1\]](https://zh.wikipedia.org/wiki/%E5%88%86%E6%95%A3%E5%BC%8F%E9%9B%9C%E6%B9%8A%E8%A1%A8#cite_note-1)、[Pastry](https://zh.wikipedia.org/w/index.php?title=Pastry&action=edit&redlink=1)（[Pastry (DHT)](https://zh.wikipedia.org/w/index.php?title=Pastry_(DHT)&action=edit&redlink=1)），以及[Tapestry (DHT)](https://zh.wikipedia.org/w/index.php?title=Tapestry_(DHT)&action=edit&redlink=1)（[Tapestry (DHT)](https://zh.wikipedia.org/w/index.php?title=Tapestry_(DHT)&action=edit&redlink=1)）皆同时于2001年发表。从那时开始，相关的研究便一直十分活跃。在学术领域以外，分布式散列表技术已经被应用在[BitTorrent](https://zh.wikipedia.org/wiki/BitTorrent)及[CoralCDN](https://zh.wikipedia.org/w/index.php?title=CoralCDN&action=edit&redlink=1)（[Coral Content Distribution Network](https://zh.wikipedia.org/w/index.php?title=Coral_Content_Distribution_Network&action=edit&redlink=1)）等。

## 性质

分布式散列表本质上强调以下特性：

- [离散性](https://zh.wikipedia.org/w/index.php?title=%E9%9B%A2%E6%95%A3%E6%80%A7&action=edit&redlink=1)：构成系统的节点并没有任何中央式的协调机制。
- [伸缩性](https://zh.wikipedia.org/wiki/%E4%BC%B8%E7%BC%A9%E6%80%A7)：即使有成千上万个节点，系统仍然应该十分有效率。
- [容错性](https://zh.wikipedia.org/w/index.php?title=%E5%AE%B9%E9%8C%AF%E6%80%A7&action=edit&redlink=1)：即使节点不断地加入、离开或是停止工作，系统仍然必须达到一定的可靠度。

要达到以上的目标，有一个关键的技术：任一个节点只需要与系统中的部分节点沟通。一般来说，若系统有{\displaystyle n}![n](https://wikimedia.org/api/rest_v1/media/math/render/svg/a601995d55609f2d9f5e233e36fbe9ea26011b3b)个节点，那么只有{\displaystyle \Theta (\log n)}![\Theta (\log n)](https://wikimedia.org/api/rest_v1/media/math/render/svg/65bac5223de9c91eb3e89a032b5c51fd3041dc66)个节点是必须的（见后述）。因此，当成员改变的时候，只有一部分的工作（例如数据或关键值的发送，散列表的改变等）必须要完成。

有些分布式散列表的设计寻求能对抗网络中恶意的节点的安全性，但仍然保留参加节点的[匿名性](https://zh.wikipedia.org/wiki/%E5%8C%BF%E5%90%8D%E6%80%A7)。在其他的点对点系统（特别是文件分享）中较为少见。参见[匿名点对点技术](https://zh.wikipedia.org/w/index.php?title=%E5%8C%BF%E5%90%8D%E9%BB%9E%E5%B0%8D%E9%BB%9E%E6%8A%80%E8%A1%93&action=edit&redlink=1)。

最后，分布式散列表必须处理传统分布式系统可能遇到的问题，例如[负载平衡](https://zh.wikipedia.org/wiki/%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1_(%E8%AE%A1%E7%AE%97%E6%9C%BA))、[数据完整性](https://zh.wikipedia.org/w/index.php?title=%E8%B3%87%E6%96%99%E5%AE%8C%E6%95%B4%E6%80%A7&action=edit&redlink=1)，以及性能问题（特别是确认转送消息、数据存储及读取等动作能快速完成）。

## 结构

分布式散列表的结构可以分成几个主要的组件[[2\]](https://zh.wikipedia.org/wiki/%E5%88%86%E6%95%A3%E5%BC%8F%E9%9B%9C%E6%B9%8A%E8%A1%A8#cite_note-2)[[3\]](https://zh.wikipedia.org/wiki/%E5%88%86%E6%95%A3%E5%BC%8F%E9%9B%9C%E6%B9%8A%E8%A1%A8#cite_note-3)。其基础是一个抽象的**关键值空间**（keyspace），例如说所有160位长的[字符串](https://zh.wikipedia.org/w/index.php?title=%E5%AD%97%E5%85%83%E4%B8%B2&action=edit&redlink=1)集合。**关键值空间分区**（keyspace partitioning）将关键值空间分区成数个，并指定到在此系统的节点中。而**延展网络**则连接这些节点，并让他们能够借由在关键值空间内的任一值找到拥有该值的节点。

当这些组件都准备好后，一般使用分布式散列表来存储与读取的方式如下所述。假设关键值空间是一个160位长的字符串集合。为了在分布式散列表中存储一个文件，名称为{\displaystyle filename}![filename](https://wikimedia.org/api/rest_v1/media/math/render/svg/5b9891400449fd99a9487e747576a94d1358d35f)且内容为{\displaystyle data}![data](https://wikimedia.org/api/rest_v1/media/math/render/svg/a2ce894feb2964e94a1302a992a2ef635dec5bfa)，我们计算出{\displaystyle filename}![filename](https://wikimedia.org/api/rest_v1/media/math/render/svg/5b9891400449fd99a9487e747576a94d1358d35f)的[SHA1](https://zh.wikipedia.org/wiki/SHA%E5%AE%B6%E6%97%8F)散列值——一个160位的关键值{\displaystyle k}![k](https://wikimedia.org/api/rest_v1/media/math/render/svg/c3c9a2c7b599b37105512c5d570edc034056dd40)——并将消息{\displaystyle put(k,data)}![put(k,data)](https://wikimedia.org/api/rest_v1/media/math/render/svg/7aa13f3ecb191d61ca8436f8ba0571c913f1f9cd)送给分布式散列表中的任意参与节点。此消息在延展网络中被转送，直到抵达在关键值空间分区中被指定负责存储关键值{\displaystyle k}![k](https://wikimedia.org/api/rest_v1/media/math/render/svg/c3c9a2c7b599b37105512c5d570edc034056dd40)的节点。而{\displaystyle (k,data)}![(k,data)](https://wikimedia.org/api/rest_v1/media/math/render/svg/d9121a5e8fefc1499df4b00d27dbeb18e70a16f3)即存储在该节点。其他的节点只需要重新计算{\displaystyle filename}![filename](https://wikimedia.org/api/rest_v1/media/math/render/svg/5b9891400449fd99a9487e747576a94d1358d35f)的散列值{\displaystyle k}![k](https://wikimedia.org/api/rest_v1/media/math/render/svg/c3c9a2c7b599b37105512c5d570edc034056dd40)，然后提交消息 {\displaystyle get(k)}![get(k)](https://wikimedia.org/api/rest_v1/media/math/render/svg/af7ef7ed971aca0c4f133bbee7702735302efd54)给分布式散列表中的任意参与节点，以此来找与{\displaystyle k}![k](https://wikimedia.org/api/rest_v1/media/math/render/svg/c3c9a2c7b599b37105512c5d570edc034056dd40)相关的数据。此消息也会在延展网络中被转送到负责存储{\displaystyle k}![k](https://wikimedia.org/api/rest_v1/media/math/render/svg/c3c9a2c7b599b37105512c5d570edc034056dd40)的节点。而此节点则会负责传回存储的数据{\displaystyle data}![data](https://wikimedia.org/api/rest_v1/media/math/render/svg/a2ce894feb2964e94a1302a992a2ef635dec5bfa)。

以下分别描述关键值空间分区及延展网络的基本概念。这些概念在大多数的分布式散列表实现中是相同的，但设计的细节部分则大多不同。

### 关键值空间分区

大多数的分布式散列表使用某些[稳定散列](https://zh.wikipedia.org/wiki/%E4%B8%80%E8%87%B4%E5%93%88%E5%B8%8C)（[consistent hashing](https://zh.wikipedia.org/wiki/%E4%B8%80%E8%87%B4%E5%93%88%E5%B8%8C)）方法来将关键值对应到节点。此方法使用了一个函数{\displaystyle \delta (k_{1},k_{2})}![\delta (k_{1},k_{2})](https://wikimedia.org/api/rest_v1/media/math/render/svg/393b0b9cbab3983c701a045715c4390cb84f31dd)来定义一个抽象的概念：从关键值{\displaystyle k_{1}}![k_1](https://wikimedia.org/api/rest_v1/media/math/render/svg/376315fd4983f01dada5ec2f7bebc48455b14a66)到{\displaystyle k_{2}}![k_2](https://wikimedia.org/api/rest_v1/media/math/render/svg/c51b4ba57ee596d8435fc4ed76703ca3a2fc444a)的距离。每个节点被指定了一个关键值，称为ID。ID为{\displaystyle i}![i](https://wikimedia.org/api/rest_v1/media/math/render/svg/add78d8608ad86e54951b8c8bd6c8d8416533d20)的节点拥有根据函数{\displaystyle \delta }![\delta ](https://wikimedia.org/api/rest_v1/media/math/render/svg/c5321cfa797202b3e1f8620663ff43c4660ea03a)计算，最接近{\displaystyle i}![i](https://wikimedia.org/api/rest_v1/media/math/render/svg/add78d8608ad86e54951b8c8bd6c8d8416533d20)的所有关键值。

> **例：**Chord分布式散列表实现将关键值视为一个圆上的点，而{\displaystyle \delta (k_{1},k_{2})}![\delta (k_{1},k_{2})](https://wikimedia.org/api/rest_v1/media/math/render/svg/393b0b9cbab3983c701a045715c4390cb84f31dd)则是沿着圆顺时钟地从{\displaystyle k_{1}}![k_1](https://wikimedia.org/api/rest_v1/media/math/render/svg/376315fd4983f01dada5ec2f7bebc48455b14a66)走到{\displaystyle k_{2}}![k_2](https://wikimedia.org/api/rest_v1/media/math/render/svg/c51b4ba57ee596d8435fc4ed76703ca3a2fc444a)的距离。结果，圆形的关键值空间就被切成连续的圆弧段，而每段的端点都是节点的ID。如果{\displaystyle i_{1}}![i_1](https://wikimedia.org/api/rest_v1/media/math/render/svg/5484b6123d92ccfcef3204a32720eeae60998e29)与{\displaystyle i_{2}}![i_2](https://wikimedia.org/api/rest_v1/media/math/render/svg/14feff7997a635a64f7dfacfbd0374a24ab279bd)是邻近的ID，则ID为{\displaystyle i_{2}}![i_2](https://wikimedia.org/api/rest_v1/media/math/render/svg/14feff7997a635a64f7dfacfbd0374a24ab279bd)的节点拥有落在{\displaystyle i_{1}}![i_1](https://wikimedia.org/api/rest_v1/media/math/render/svg/5484b6123d92ccfcef3204a32720eeae60998e29)及{\displaystyle i_{2}}![i_2](https://wikimedia.org/api/rest_v1/media/math/render/svg/14feff7997a635a64f7dfacfbd0374a24ab279bd)之间的所有关键值。

稳定散列拥有一个基本的性质：增加或移除节点只改变邻近ID的节点所拥有的关键值集合，而其他节点的则不会被改变。对比于传统的散列表，若增加或移除一个位置，则整个关键值空间就必须重新对应。由于拥有数据的改变通常会导致数据从分布式散列表中的一个节点被搬到另一个节点，而这是非常浪费带宽的，因此若要有效率地支持大量密集的节点增加或离开的动作，这种重新配置的行为必须尽量减少。

将相近的关键值分配给了距离相近的节点[Locality-preserving_hashing](https://zh.wikipedia.org/w/index.php?title=Locality-preserving_hashing&action=edit&redlink=1)，可以实现更短的查询延迟，从而提高DHT的查询效率。相关工作包括Self-Chord[[4\]](https://zh.wikipedia.org/wiki/%E5%88%86%E6%95%A3%E5%BC%8F%E9%9B%9C%E6%B9%8A%E8%A1%A8#cite_note-4)和LDHT[[5\]](https://zh.wikipedia.org/wiki/%E5%88%86%E6%95%A3%E5%BC%8F%E9%9B%9C%E6%B9%8A%E8%A1%A8#cite_note-5)

### 延展网络

每个节点保有一些到其他节点（它的邻居）的链接。将这些链接总合起来就形成延展网络。而这些链接是使用一个结构性的方式来挑选的，称为[网络拓朴](https://zh.wikipedia.org/wiki/%E7%B6%B2%E8%B7%AF%E6%8B%93%E6%A8%B8)。

所有的分布式散列表实现拓朴有某些基本的性质：对于任一关键值{\displaystyle k}![k](https://wikimedia.org/api/rest_v1/media/math/render/svg/c3c9a2c7b599b37105512c5d570edc034056dd40)，某个节点要不就拥有{\displaystyle k}![k](https://wikimedia.org/api/rest_v1/media/math/render/svg/c3c9a2c7b599b37105512c5d570edc034056dd40)，要不就拥有一个链接能链接到距离较接近{\displaystyle k}![k](https://wikimedia.org/api/rest_v1/media/math/render/svg/c3c9a2c7b599b37105512c5d570edc034056dd40)的节点。因此使用以下的[贪心算法](https://zh.wikipedia.org/wiki/%E8%B2%AA%E5%BF%83%E6%B3%95)即可容易地将消息转送到拥有关键值{\displaystyle k}![k](https://wikimedia.org/api/rest_v1/media/math/render/svg/c3c9a2c7b599b37105512c5d570edc034056dd40)的节点：在每次运行时，将消息转送到ID较接近{\displaystyle k}![k](https://wikimedia.org/api/rest_v1/media/math/render/svg/c3c9a2c7b599b37105512c5d570edc034056dd40)的邻近节点。若没有这样的节点，那我们一定抵达了最接近{\displaystyle k}![k](https://wikimedia.org/api/rest_v1/media/math/render/svg/c3c9a2c7b599b37105512c5d570edc034056dd40)的节点，也就是拥有{\displaystyle k}![k](https://wikimedia.org/api/rest_v1/media/math/render/svg/c3c9a2c7b599b37105512c5d570edc034056dd40)的节点。这样的转送方法有时被称为“基于关键值的转送方法”。

除了基本的转送正确性之外，拓朴中另有两个关键的限制：其一为保证任何的转送路径长度必须尽量短，因而请求能快速地被完成；其二为任一节点的邻近节点数目（又称[最大节点度](https://zh.wikipedia.org/wiki/%E5%9B%BE)（[Degree (graph theory)](https://zh.wikipedia.org/w/index.php?title=Degree_(graph_theory)&action=edit&redlink=1)））必须尽量少，因此维护的花费不会过多。当然，转送长度越短，则最大节点度越大。以下列出常见的最大节点度及转送长度（{\displaystyle n}![n](https://wikimedia.org/api/rest_v1/media/math/render/svg/a601995d55609f2d9f5e233e36fbe9ea26011b3b)为分布式散列表中的节点数）

- 最大节点度{\displaystyle O(1)}![O(1)](https://wikimedia.org/api/rest_v1/media/math/render/svg/e66384bc40452c5452f33563fe0e27e803b0cc21)，转送长度{\displaystyle O(\log n)}![O(\log n)](https://wikimedia.org/api/rest_v1/media/math/render/svg/aae0f22048ba6b7c05dbae17b056bfa16e21807d)
- 最大节点度{\displaystyle O(\log n)}![O(\log n)](https://wikimedia.org/api/rest_v1/media/math/render/svg/aae0f22048ba6b7c05dbae17b056bfa16e21807d)，转送长度{\displaystyle O(\log n/\log \log n)}![O(\log n/\log \log n)](https://wikimedia.org/api/rest_v1/media/math/render/svg/dc17efd2d66135a2972e35ab7680ec720efc8d5b)
- 最大节点度{\displaystyle O(\log n)}![O(\log n)](https://wikimedia.org/api/rest_v1/media/math/render/svg/aae0f22048ba6b7c05dbae17b056bfa16e21807d)，转送长度{\displaystyle O(\log n)}![O(\log n)](https://wikimedia.org/api/rest_v1/media/math/render/svg/aae0f22048ba6b7c05dbae17b056bfa16e21807d)
- 最大节点度{\displaystyle O(n^{1/2})}![O(n^{{1/2}})](https://wikimedia.org/api/rest_v1/media/math/render/svg/1c36793b19a2cdc012e9b62f10acc548a2e64ce8)，转送长度{\displaystyle O(1)}![O(1)](https://wikimedia.org/api/rest_v1/media/math/render/svg/e66384bc40452c5452f33563fe0e27e803b0cc21)

第三个选择最为常见。虽然他在最大节点度与转送长度的取舍中并不是最佳的选择，但这样的拓朴允许较为有弹性地选择邻近节点。许多分布式散列表实现利用这种弹性来选择延迟较低的邻近节点。

最大的转送长度与[直径](https://zh.wikipedia.org/w/index.php?title=%E7%9B%B4%E5%BE%91_(%E5%9C%96%E8%AB%96)&action=edit&redlink=1)有关：最远的两节点之间的最短跳数（Hop Distance）。无疑地，网络的最大转送长度至少要与它的直径一样长，因而拓朴也被最大节点度与直径的取舍限制住[[6\]](https://zh.wikipedia.org/wiki/%E5%88%86%E6%95%A3%E5%BC%8F%E9%9B%9C%E6%B9%8A%E8%A1%A8#cite_note-6)，而这在[图论](https://zh.wikipedia.org/wiki/%E5%9C%96%E8%AB%96)中是基本的性质。因为贪婪算法（Greedy Method）可能找不到最短路径，因此转送长度可能比直径长[[7\]](https://zh.wikipedia.org/wiki/%E5%88%86%E6%95%A3%E5%BC%8F%E9%9B%9C%E6%B9%8A%E8%A1%A8#cite_note-7)。

## 示例

### 分布式散列表实现与协议

- [Bamboo](https://zh.wikipedia.org/w/index.php?title=Bamboo_(DHT)&action=edit&redlink=1)[[8\]](https://zh.wikipedia.org/wiki/%E5%88%86%E6%95%A3%E5%BC%8F%E9%9B%9C%E6%B9%8A%E8%A1%A8#cite_note-8)
- [Bunshin](https://zh.wikipedia.org/w/index.php?title=Bunshin&action=edit&redlink=1)[[9\]](https://zh.wikipedia.org/wiki/%E5%88%86%E6%95%A3%E5%BC%8F%E9%9B%9C%E6%B9%8A%E8%A1%A8#cite_note-9)
- [内容可定址网络](https://zh.wikipedia.org/w/index.php?title=%E5%85%A7%E5%AE%B9%E5%8F%AF%E5%AE%9A%E5%9D%80%E7%B6%B2%E8%B7%AF&action=edit&redlink=1)（Content Addressable Network）
- [Chord](https://zh.wikipedia.org/w/index.php?title=Chord&action=edit&redlink=1)
- [DKS系统](https://zh.wikipedia.org/w/index.php?title=DKS%E7%B3%BB%E7%B5%B1&action=edit&redlink=1)[[10\]](https://zh.wikipedia.org/wiki/%E5%88%86%E6%95%A3%E5%BC%8F%E9%9B%9C%E6%B9%8A%E8%A1%A8#cite_note-10)
- [Kademlia](https://zh.wikipedia.org/wiki/Kademlia)
- [Leopard](https://zh.wikipedia.org/w/index.php?title=Leopard_(DHT)&action=edit&redlink=1)
- [MACE](https://zh.wikipedia.org/w/index.php?title=MACE&action=edit&redlink=1)[[11\]](https://zh.wikipedia.org/wiki/%E5%88%86%E6%95%A3%E5%BC%8F%E9%9B%9C%E6%B9%8A%E8%A1%A8#cite_note-11)
- [Pastry](https://zh.wikipedia.org/w/index.php?title=Pastry_(DHT)&action=edit&redlink=1)
- [P-Grid](https://zh.wikipedia.org/w/index.php?title=P-Grid&action=edit&redlink=1)
- [Tapestry](https://zh.wikipedia.org/w/index.php?title=Tapestry_(DHT)&action=edit&redlink=1)

### 分布式散列表的应用

- [BitTorrent](https://zh.wikipedia.org/wiki/BitTorrent)：文件分享应用。BitTorrent可以选用DHT作为分布式Tracker。
- [Warez P2P](https://zh.wikipedia.org/w/index.php?title=Warez_P2P&action=edit&redlink=1)：文件分享应用。
- [The Circle](https://zh.wikipedia.org/wiki/The_Circle)：文件分享应用与聊天。
- [CSpace](https://zh.wikipedia.org/w/index.php?title=CSpace&action=edit&redlink=1)：安全的沟通系统。
- [Codeen](https://zh.wikipedia.org/w/index.php?title=Codeen&action=edit&redlink=1)：网页缓存。
- [CoralCDN](https://zh.wikipedia.org/w/index.php?title=CoralCDN&action=edit&redlink=1)
- [Dijjer](https://zh.wikipedia.org/w/index.php?title=Dijjer&action=edit&redlink=1)
- [eMule](https://zh.wikipedia.org/wiki/EMule)：文件分享应用。
- [I2P](https://zh.wikipedia.org/wiki/I2P)：匿名网络。
- [JXTA](https://zh.wikipedia.org/w/index.php?title=JXTA&action=edit&redlink=1)：开放源代码的点对点平台。
- [NEOnet](https://zh.wikipedia.org/w/index.php?title=NEOnet&action=edit&redlink=1)：文件分享应用。
- [Overnet](https://zh.wikipedia.org/wiki/Overnet)：文件分享应用。