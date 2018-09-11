# IPFS源码分析之get流程（三） #
上一篇看了get API从本地仓库获得想要查找的CID文件块，在本地仓库存有的前提下。回到从本地获取还是从网络获取的分叉点，如果是从IPFS网络中获取则先查找能够提供该CID文件块的节点，首先在上篇的Fig 7中执行这样的函数：this.network.findAndConnect(cids[0], …)，一般cids[0]就是想要查找的CID，函数详情如下：
 ![](https://i.imgur.com/mjnnBlQ.png)

											Fig 1 ipfs-bitswap/src/network.js
图中findAndConnect(cids[0], …)马上就调用了findProviders ()函数，得到20个能够提供该CID文件块的节点后再分别连接，20是个定值，还有maxProvidersPerRequest = 3也是个定值，这些后面会说到。findProviders ()函数又直接调用了Libp2p模块的函数libp2p.contentRouting.findProviders(cid,…)，通过Libp2p层的路由去查，这个函数的详情如下：
 ![](https://i.imgur.com/X8RyELY.png)

								Fig 2 libp2p/src/content-routing.js
Libp2p层对content-routing和_dht的引入可以参考《IPFS源码分析之Libp2p层》中的Fig 5，上图中的node即是整个libp2p对象，_dht是一个Kad DHT分布式哈希表实例，然后直接调用该实例的findProviders()方法，后者内部没有多余的逻辑代码，直接就调用另外一个函数_findNProviders(key, timeout, c.K, callback)，前面提到的定值20就是这里添加进去的，c.K = 20。此后所有函数的任务就是找到20个能够提供该CID文件块的节点。这个函数比较长，分为两个图，顺便把另外一个它将调用到的函数也放进去：
 ![](https://i.imgur.com/wFKtcZL.png)

												Fig 3 libp2p-kad-dht/src/private.js 
 ![](https://i.imgur.com/vVS13R7.png)

											Fig 4 libp2p-kad-dht/src/private.js
先看Fig 4，调用dht.providers.getProviders()函数从本地内存中获取可以提供该CID文件块的节点peerId，dht还是那个Kad DHT分布式哈希表实例，providers是一个类，这个类的作用是提供两个内存空间，都存储了能够提供已知CID的节点的对应信息，一个是备份空间datastore，这个空间可以无限增加，当IPFS dameon关闭后再重启时能够快速通过这个内存空间找回这些已知信息。另一个是LRU Cache，这个是个固定大小的cache空间，以LRU算法管理，总是保存着最近查找的{CID+提供节点：日期}，作用就cache的作用，快速检索且命中率高。在providers类的构造函数中可以看到，LRU Cache使用的是第三方开发包：hashlru。
dht.providers.getProviders()函数本身也没用多余的逻辑代码，直接调用同文件下另外一个函数_getProvidersMap()：
 ![](https://i.imgur.com/CmuvBX4.png)

									Fig 5 libp2p-kad-dht/src/providers.js
图中167行的this指的是这个providers类实例，this后面的providers指的是LRU Cache实例。makeProviderKey (cid)的作用仅仅是在编码后的CID前加'/providers/'。先从LRU Cache中找，再到备份内存空间this.datastore中找，这种方式很像计算机cache和内存的换页。
可以看看在备份内存空间this.datastore中的查找函数loadProviders()：
 ![](https://i.imgur.com/ll7GHxj.png)

									Fig 6 libp2p-kad-dht/src/providers.js
不管是cache还是datastore，数据存储的格式都一样，LRU算法根据时间来更新和剔除最久的每访问的CID+节点，所以用时间作为values。
回到Fig 3，拿到可以提供该CID文件块的节点信息后（也许有也许没有），将这些节点比对dht.peerBook中保存的所有已知节点信息，有则取出完整信息，没有则添加进去。然后判断是否已经拿到20个节点，没有则需在IPFS网络上查询。

先实例化一个Query类，这个类的作用是以一个队列来管理查询任务和查询条件判断的细节。然后调用dht.routingTable.closestPeers(key.buffer,…)函数，key即CID，在本地的路由表即KBucket中获得距离该CID转换成DHT ID后的VOR距离最近的3个节点，这个获得方法很奇妙。closestPeers()函数内直接调用kb.closest()函数，后者的kb就是一个KBucket类实例，关于KBucket的用处需要了解Kad DHT，这里不赘述。这个类由k-bucket模块实现，它的实例其中包含一个很重要的值是根节点node，这个node的内容结构如下：
{ contacts: [], dontSplit: false, left: null, right: null }

以这个根节点node展开一个二叉树，这个二叉树的的非叶子节点node中contacts为空，只有叶子节点node的contacts才有最多不超过20个contact，每个contact表示一个peer。所以这颗二叉树的叶子节点不会出现单个，contact满20后会分裂两个叶节点。查找方式类似深度优先遍历，但具体路径和CID转换成DHT ID后的值有关，用一个较为复杂的算法，这是个神奇的地方。最后得到一些contact，再比较contact.id和DHT ID的距离，选择最近的3个contact返回。所以dht.routingTable.closestPeers(key.buffer,…)函数得到的3个节点，并不知道他们是否能够提供想要的CID文件块，只知道由上面的方法计算出来的距离最近，所以接下来需要发消息询问他们。
Fig 4的551行，在总时间为1分钟的限制内，调用query.run(peers, cb)查询：
 ![](https://i.imgur.com/aypxpsy.png)

												Fig 7 libp2p-kad-dht/src/query.js

主要是59行，执行workerQueue(this, run, cb)，建立查询任务队列，详情如下：
 ![](https://i.imgur.com/LhQo8fr.png)

										Fig 8 libp2p-kad-dht/src/query.js
 ![](https://i.imgur.com/ir3L6O0.png)
							
											Fig 9 libp2p-kad-dht/src/query.js
调用流程如上两图，Fig 8和Fig 9中的代码主要是管理等待被查询的节点，每次把一个节点传给Fig 9中156行的query.query(next，…)函数，它表示的函数是Fig 3的523到Fig 4的547行这段代码， next是要被查询的节点，next后的代码是回调函数，最后才执行。query.query(next，…)函数内部调用了dht._findProvidersSingle(peer, key, cb)真正地向节点发出询问消息，这个函数在Fig 4的末尾。
节点会返回在它知不知道我们想查找的CID文件块哪个节点可以提供，知道的放在msg.providerPeers中，如果这个节点恰好能提供该CID文件块，msg.providerPeers会包含它的信息，不知道会返回3个建议询问节点，放在msg.closerPeers中，就这样逐步扩散查询，直到找到20个能够提供指定CID文件块的节点。当然，它返回的3个建议询问节点，对它而言，就像前面调用dht.routingTable.closestPeers(key.buffer,…)函数一样，返回距离该CID转换成DHT ID后的VOR距离最近的3个节点
以上是通过IPFS网络找到能提供get API指定的CID文件块的节点过程，Fig 4中询问节点调用到dht.network.sendRequest(peer, msg, callback)，所以下次再看消息的收发和文件块的收发，真正地拿到那个CID文件块。

**个人微信：xxb3059 & 公众号：区块丛林 欢迎交流**
