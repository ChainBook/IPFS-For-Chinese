# IPFS源码分析之Bitswap层（一） #

## 一．Bitswap简要说明 ##
IPFS 中的BitSwap协议受到BitTorrent 的启发，通过对等节点间交换数据块来分发数据的。像BT一样， 每个对等节点在下载的同时不断向其他对等节点上传已下载的数据。和BT协议不同的是， BitSwap 不局限于一个torrent文件中的数据块。也就是下载某个大文件或目录（被分割为多个小blob和list存储），其中某个blob或list也是其他文件的组成部分。BitSwap 协议中存在一个永久的市场。 这个市场包括各个节点想要获取的所有块数据。而不管这些块是哪些如.torrent文件中的一部分。这些快数据可能来自文件系统中完全不相关的文件。 这个市场是由所有的节点组成的。

IPFS每一个节点都维护了两个列表：已有的数据块（have_list）和想要的数据块（want_list）
BITSWAP 信用机制，防止水蛭攻击（空负载节点从不共享块）。节点下载文件和共享帮忙传播文件应该平衡，通过BITSWAP账本记录着收发字节。A和B发生连接时，A先向B发送本地账本中和B有关的部分，即A和B之前的交换文件块记录。如果相同则建立链接，不同则B节点向A节点发送一个空白账本，如果某个节点经常账本不同，其他节点会记住，以后不予链接。而且建立链接后，B向A发送的文件块，A检查是否是需要的，若不是也会断开链接，且不会更新账本。
## 二．Bitswap源码探索 ##
整套IPFS源码和上篇文章的一样。Bitswap的引入在js-ipfs\src\core\components\start.js中：
 ![](https://i.imgur.com/KRmNAVS.png)

								Fig 1 Bitswap引入
创建一个Bitswap类实例，Bitswap = require('ipfs-bitswap')。需要传入的值有三个，其中self._libp2pNode在上篇文章有具体源码分析，作用简单来说就是在P2P层建立一个节点实例。self._repo.blocks包含对blocks文件夹根据文件CID的各种操作方法的集合。所以在探索Bitswap前，有必要先看看self._repo.blocks。

Self指的是整个IPFS实例。self._repo是IPFS非常重要的一个东西，是一个IpfsRepo类实例，描述了IPFS节点所在机器的仓库。默认是在家目录下创建的“.ipfs/”文件夹。

 ![](https://i.imgur.com/lNaw3q4.png)

				Fig 2 ipfs模块的index.js文件中引入
IpfsRepo类在ipfs-repo模块的index.js文件中定义：class IpfsRepo{ …… }
self._repo有三个重要属性, datastore，blocks，keys，分别对应“.ipfs/”下的datastore，blocks，keystore三个子文件夹，datastore存放了具体的文件块，包括本节点上传的和查看的，blocks存放了文件(以CID描述)的DAG结构，keystore存放了节点的公私钥。整体结构如下，可以看到底层使用的是levelDB。
 ![](https://i.imgur.com/To6rl6e.png)

													Fig 3 IpfsRepo的结构
在初始化时调用该模块的index.js中的open()函数创建。
 ![](https://i.imgur.com/wAHcsC6.png)

										Fig 4 ipfs-repo模块的index.js中的open()函数
主要看的是blocks这个属性，和另外两个不一样的是它还用了一个blockstore()函数将原始的FsDatastore类实例加工。blockstore = require('./blockstore')。在实际执行的是该ipfs-repo模块的blockstore.js中exports出去的方法：
 ![](https://i.imgur.com/XyCyLiW.png)

										Fig 5 ipfs-repo模块的blockstore.js
在途中可以看到增加了shard(与文件的分片存储相关)，和更深一层调用ShardingStore.createOrOpen(filestore, shard, callback)带来的对key操作的方法。其中ShardingStore=require('datastore-core'). ShardingDatastore，具体的实现这里就不贴详细注释的代码,因为嵌入层次较多。由于文件存储是DAG树结构，树的节点及其子节点都是以key:value和key:{key:value ……}来表示，所以需要在底层实现对key直接操作，文件的CID由上图中的cidToDsKey方法转换为对应key。最后由上图中的callback(null, createBaseStore(store))返回直接对CID操作的方法集合。以上就是self._repo.blocks的创建过程，下一节开始看Bitswap。
## 缩写词解释： ##
1.CID(Self-describing content-addressed identifiers for distributed systems):基于内容寻址的自我描述标识，内容ID。

2.DAG，或Merkle DAG，Merkle directed acyclic graph（默克有向无环图）。它是在Merkle tree基础上构建的，Merkle tree是由美国计算机学家merkle于1979年申请的专利。Merkle DAG跟Merkle tree很相似，但不完全一样，比如：Merkle DAG不需要进行树的平衡操作，非叶子节点允许包含数据等。

3.LevelDB， 是由 Google 开发的 key-value 非关系型数据库存储系统，是基于 LSM(Log-Structured-Merge Tree) 的典型实现，LSM 的原理是：当读写数据库时，首先纪录读写操作到 Op log 文件中，然后再操作内存数据库，当达到 checkpoint 时，则写入磁盘，同时删除相应的 Op log 文件，后续重新生成新的内存文件和 Op log 文件。

**个人微信：xxb3059 &公众号：区块丛林  欢迎交流**