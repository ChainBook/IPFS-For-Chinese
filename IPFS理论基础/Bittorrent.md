## 技术特点

2003年，[软件工程师](https://baike.baidu.com/item/%E8%BD%AF%E4%BB%B6%E5%B7%A5%E7%A8%8B%E5%B8%88)Bram Cohen发明了BitTorrent协议。

BitTorrent(简称BT)是一个文件分发协议，每个下载者在下载的同时不断向其他下载者上传已下载的数据。而在[FTP](https://baike.baidu.com/item/FTP/13839),[HTTP](https://baike.baidu.com/item/HTTP)协议中，每个下载者在下载自己所需文件的同时，各个下载者之间没有交互。当非常多的用户同时访问和下载服务器上的文件时，由于FTP服务器处理能力和带宽的限制，下载速度会急剧下降，有的用户可能访问不了服务器。BT协议与FTP协议不同，特点是下载的人越多，下载速度越快，原因在于每个下载者将已下载的数据提供给其他下载者下载，充分利用了用户的上载带宽。通过一定的策略保证上传速度越快，下载速度也越快。在很短时间内，BitTorrent协议成为一种新的变革技术。

## 实现原理

普通的[HTTP](https://baike.baidu.com/item/HTTP)/[FTP](https://baike.baidu.com/item/FTP)下载使用[TCP/IP](https://baike.baidu.com/item/TCP%2FIP)协议，BitTorrent协议是架构于TCP/IP协议之上的一个P2P[文件传输协议](https://baike.baidu.com/item/%E6%96%87%E4%BB%B6%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE)，处于TCP/IP结构的[应用层](https://baike.baidu.com/item/%E5%BA%94%E7%94%A8%E5%B1%82)。 BitTorrent协议本身也包含了很多具体的内容协议和扩展协议，并在不断扩充中。

根据BitTorrent协议，文件发布者会根据要发布的文件生成提供一个.torrent文件，即[种子文件](https://baike.baidu.com/item/%E7%A7%8D%E5%AD%90%E6%96%87%E4%BB%B6)，也简称为“种子”。

.torrent文件本质上是文本文件，包含Tracker信息和文件信息两部分。[Tracker](https://baike.baidu.com/item/Tracker)信息主要是[BT下载](https://baike.baidu.com/item/BT%E4%B8%8B%E8%BD%BD)中需要用到的Tracker[服务器](https://baike.baidu.com/item/%E6%9C%8D%E5%8A%A1%E5%99%A8)的地址和针对Tracker服务器的设置，文件信息是根据对[目标文件](https://baike.baidu.com/item/%E7%9B%AE%E6%A0%87%E6%96%87%E4%BB%B6)的计算生成的，计算结果根据BitTorrent协议内的B编码规则进行编码。它的主要原理是需要把提供下载的文件虚拟分成大小相等的块，块大小必须为2k的整数次方（由于是虚拟分块，硬盘上并不产生各个块文件），并把每个块的索引信息和[Hash](https://baike.baidu.com/item/Hash)验证码写入种子文件（.torrent）中。所以，种子文件（.torrent）就是被下载文件的“索引”。

## 技术依赖

[![img](http://b.hiphotos.baidu.com/baike/s%3D220/sign=6b635611a1d3fd1f3209a538004f25ce/aa18972bd40735fa0f258b8697510fb30f24083c.jpg)](https://baike.baidu.com/pic/BitTorrent/142795/0/aa18972bd40735fa0f258b8697510fb30f24083c?fr=lemma&ct=single)

P2P是近年来互联网最热门的技术 ,在[VoIP](https://baike.baidu.com/item/VoIP)、下载、[流媒体](https://baike.baidu.com/item/%E6%B5%81%E5%AA%92%E4%BD%93)、协调技术等领域得到飞速发展 , 被[财富杂志](https://baike.baidu.com/item/%E8%B4%A2%E5%AF%8C%E6%9D%82%E5%BF%97)评为影响互联网的四大科技之一。P2P技术体现了互联网最根本的内涵——自由和免费,它的主要优点如下：和检索相关的节点上去 , 存储有和该检索 ；对等性高 : 非中心化 , 互联网回归本色——联系和传输 ;扩展性强 : 用户扩展与资源、服务、系统同步扩展 ;健壮性高 : 服务分散和自适应 , 耐攻击、高容错性 ;性价比高 :P2P成本低、存储和技术能力强 ;负载均衡 :分布存储和技术 , 整个网络负载得以均衡。

服务器

客户端

[![img](http://f.hiphotos.baidu.com/baike/s%3D220/sign=303983844e10b912bbc1f1fcf3fdfcb5/aa64034f78f0f73652eb883e0355b319ebc413fc.jpg)](https://baike.baidu.com/pic/BitTorrent/142795/0/aa64034f78f0f73652eb883e0355b319ebc413fc?fr=lemma&ct=single)

图中的[检索](https://baike.baidu.com/item/%E6%A3%80%E7%B4%A2)过程分为以下几个阶段 :每个节点在加入网络的时候 , 会对存储在本节点上的内容进行索引 , 以满足本地内容检索的目的。然后按某种预定的规则选择一些节点作为自己的邻居 , 加入到P2P网络当中。发起者P提出检索请求q,并将 q发送给自己的邻居 P的邻居收到 q后 , 再按照某种策略转发给它在网络中的其它邻居节点。这样 ,q就在整个网络中传播开来。收到请求 q 的节点如果存储有相应内容信息 , 则将对应的内容返回。

如何在一个大规模分布的环境下定位资源是个十分具有挑战性的问题。集中在如何组建P2P网络,如何选择有效的资源请求路由策略以便以较少的消息通信开销 ,获得较多的相关查询结果返回 , 同时能够保证较好的服务均衡性。 [1] 

## 下载特点

和常规下载文件不一样的是，当你进行BT下载时，你开始链接的地址都是.torrent结尾的文件。其实只要下载此文件，在本机运行此文件一样可以进行BT下载工作。而网上的BT下载链接都是由广大用户自己发布提供的，这样使得下载资料非常广，不受常规管理人员的限制。 [2] 

### 种子

[![BT原理示意图](http://b.hiphotos.baidu.com/baike/s%3D220/sign=872975ac0a7b020808c938e352d9f25f/d8f9d72a6059252d3d05400d349b033b5bb5b9ad.jpg)](https://baike.baidu.com/pic/BitTorrent/142795/0/b3508d135951a2b96538dbce?fr=lemma&ct=single)BT原理示意图

当下载结束后，如果未关闭BT客户端程序（例如一边运行BT提供上传服务，一边浏览网页、编辑文档等），这时你将成为一个传递圣火的使者，即“种子”（seed）。换句话说，如果一个文件被分成10个部分，但拥有第9部分的人只有一个，即只有一个种子，如果这位用户由于某种原因断线或关机，那么其他用户就只能下载到90%了，在进行BT下载时是令人最为苦恼的。

想想自己下载时遇到的“种子数为0”的痛苦吧，将心比心，尽可能在下载结束后不要立即关闭BT程序窗口，做一个传递圣火的使者吧。

### 下载注意

[下载者](https://baike.baidu.com/item/%E4%B8%8B%E8%BD%BD%E8%80%85)要下载文件内容，需要先得到相应的.torrent文件，然后使用BT客户端软件进行下载。

下载时，BT客户端首先解析.torrent文件得到Tracker地址，然后连接Tracker[服务器](https://baike.baidu.com/item/%E6%9C%8D%E5%8A%A1%E5%99%A8)。Tracker服务器回应下载者的请求，提供下载者其他下载者（包括发布者）的IP。下载者再连接其他下载者，根据.torrent文件，两者分别对方告知自己已经有的[块](https://baike.baidu.com/item/%E5%9D%97/13684986)，然后交换对方没有的数据。此时不需要其他服务器参与，分散了单个线路上的数据流量，因此减轻了服务器负担。

下载者每得到一个块，需要算出下载块的Hash验证码与.torrent文件中的对比，如果一样则说明块正确，不一样则需要重新下载这个块。这种规定是为了解决下载内容准确性的问题。

### 存在问题

一般的[HTTP](https://baike.baidu.com/item/HTTP)/[FTP](https://baike.baidu.com/item/FTP)下载，发布文件仅在某个或某几个[服务器](https://baike.baidu.com/item/%E6%9C%8D%E5%8A%A1%E5%99%A8)，下载的人太多，服务器的带宽很易不胜负荷，变得很慢。而BitTorrent协议下载的特点是，下载的人越多，提供的[带宽](https://baike.baidu.com/item/%E5%B8%A6%E5%AE%BD)也越多，种子也会越来越多，下载速度就越快。

而有些人下载完成后关掉下载任务，提供较少量数据给其他用户，为尽量避免这种行为，在非官方BitTorrent协议中存在[超级种子](https://baike.baidu.com/item/%E8%B6%85%E7%BA%A7%E7%A7%8D%E5%AD%90)的算法。这种算法允许文件发布者分几步发布文件，发布者不需要一次提供文件所有内容，而是慢慢开放的下载内容的比例，延长下载时间。此时，速度快的人由于未下载完必须提供给他人数据，速度慢的人有更多机会得到数据。

## 网络技术

又发展出DHT网络技术，使得无Tracker下载成为可能。

DHT全称为分布式[哈希表](https://baike.baidu.com/item/%E5%93%88%E5%B8%8C%E8%A1%A8)(Distributed Hash Table)，是一种分布式存储方法。在不需要[服务器](https://baike.baidu.com/item/%E6%9C%8D%E5%8A%A1%E5%99%A8)的情况下，每个客户端负责一个小范围的[路由](https://baike.baidu.com/item/%E8%B7%AF%E7%94%B1)，并负责存储一小部分数据，从而实现整个DHT网络的寻址和存储。使用支持该技术的BT下载软件，用户无需连上Tracker就可以下载，因为软件会在DHT网络中寻找下载同一文件的其他用户并与之通讯，开始下载任务。

有些软件([比特精灵](https://baike.baidu.com/item/%E6%AF%94%E7%89%B9%E7%B2%BE%E7%81%B5))还会自动通过DHT搜索种子资源，构成[种子市场](https://baike.baidu.com/item/%E7%A7%8D%E5%AD%90%E5%B8%82%E5%9C%BA)。

另外，这里使用的DHT算法叫Kademlia（在[eMule](https://baike.baidu.com/item/eMule)中也有使用，常把它叫做[KAD](https://baike.baidu.com/item/KAD)，具体实现协议有所不同）。

这种技术好处十分明显，就是大大减轻了Tracker的负担（甚至不需要）。用户之间可以更快速建立通讯（特别是与Tracker连接不上的时候）。

## 相关概念

Tracker：收集[下载者](https://baike.baidu.com/item/%E4%B8%8B%E8%BD%BD%E8%80%85)信息的[服务器](https://baike.baidu.com/item/%E6%9C%8D%E5%8A%A1%E5%99%A8)，并将此信息提供给其他下载者，使下载者们相互连接起来，[传输数据](https://baike.baidu.com/item/%E4%BC%A0%E8%BE%93%E6%95%B0%E6%8D%AE)。

种子：指一个下载任务中所有文件都被某下载者完整的下载，此时下载者成为一个种子。发布者本身发布的文件就是原始种子。也指.torrent文件。

[做种](https://baike.baidu.com/item/%E5%81%9A%E7%A7%8D)：发布者提供下载任务的全部内容的行为；下载者下载完成后继续提供给他人下载的行为。