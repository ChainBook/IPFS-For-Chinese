# IPFS源码分析之get流程（二） #
接上篇继续走get API的流程，也就是dag.get()怎么获得DAGNode。上次说了dag是作为参数传进来的本节点的IPLD实例，也看了IPLD类的构造函数，所以这里就直接看它的get方法，代码比较长，分为两个图。
 ![](https://i.imgur.com/tEvo0tQ.png)

											Fig 1 ipld/src/index.js get(1)
 ![](https://i.imgur.com/4HtnBAa.png)

											Fig 2 ipld/src/index.js get(2)
对于get API来说，Fig 1就已经够了，因为根据上一层传入的第二个参数是一个函数，所以由get方法里的逻辑走到执行完this._get()就return出去了，不会执行下面的doUntil()函数，包括getReadableStream，getPullStream等API。但既然都到了这里，非常好奇很长一段的doUntil()函数的执行条件和作用。所以根据第二个参数是一个函数（即仅有CID+callback）还是一个string path（即CID+path+callback）分两种情况解释。
## （1）CID +callback ##
这个比较简单，最常用。在Fig 1的第一个if块中将path = undefined后就直接走到执行this._get()，这个函数也在这个文件中：
 ![](https://i.imgur.com/a78xCKB.png)

									Fig 3 ipld/src/index.js _get()
先根据CID对象的编码类型取出该数据类型（CBOR，Bitcoin，ETH系列等）在ipld中注册的resolver 函数。注册由上一篇的Fig 5中用很多个this.support.add()函数。然后bs.get(cid, cb)根据CID获取数据块，bs就是ipld依赖的blockservice类实例，可以是从其他节点获取也可以是从本地如果本地有缓存，先暂时不管具体的获取过程。

bs.get(cid, cb)获取到的是一个block类实例，这个类有两个基本属性block.data和block.cid。前者是二进制数据binaryBlob，经deserialize()后的内容就是ipld层的DAGNode。根据IPFS的白皮书，Blob是文件能被寻址的最小粒对象，block.cid正是它的地址。所以各种数据类型（CBOR，Bitcoin，ETH系列等）都有自己的一套方法将数据转换成Merkle DAG树上的一个个DAGNode，以Bitcoin为例，一个区块可以成为一个Merkle DAG树。然后将每个DAGNode的内容序列化，也根据内容计算出CID，然后将两者由blockservice统一封装成一个个block，最后Bitswap层的节点之间交换的数据就是block。
## （2）CID+path+callback ##
这种调用方式很少，和文件相关的API不会这样调用，不过根据官方的API文档，确实几个DAG API，作用是作为一组处理object的api供开发调用，下面是在github上dag.get API的调用例子。
 ![](https://i.imgur.com/8R8BeOs.png)

											Fig 4 dag.get实例
主要是Fig1 的doUntil()函数里的三个子函数来实现，第二个子函数是条件。同样先由bs.get(cid, cb)拿到block。对block.data的反序列化在r.resolver.resolve()函数中，其中r.resolver也是根据CID对象的编码类型取出该数据类型（CBOR，Bitcoin，ETH系列等）在ipld中注册的resolver 函数，以CBOR数据类型为例：

 ![](https://i.imgur.com/6t1TJ79.png)

								Fig 5 ipld-dag-cbor/src/resolver.js
如果传入的path刚好是在该DAGNode包含的文件树中，则直接返回DAGNode中这个路径的值。如果不在还要继续获取新的block再解析下去。get API在之前已经实现了类似的逻辑即对子文件精确地查找，所以这里就不用。

回到正轨的get API流程，看看bs.get(cid, cb)怎么拿到block：
 ![](https://i.imgur.com/XEJy2N1.png)

						Fig 6 ipfs-block-service/src/index.js
先判断本地有没有this.hasExchange()，一定是有的，在《IPFS源码分析之Bitswap层（一）》的Fig 1的最后一行代码setExchange (bitswap)，将bitswap实例作为交换控制系统。然后接着调用bitswap的get()方法获取文件块block，这个函数在ipfs-bitswap模块的index.js文件可以找到，get()直接调用同文件中的getMany()方法根据CID获取文件块block，即使只查询一个CID。可以看看这个getMany()方法：
 ![](https://i.imgur.com/42LmTTY.png)
						
											Fig 7 ipfs-bitswap/src/index.js
图中代码注释很多，这是get API的流程中关键一步。getFromOutside()是个闭包函数，所以先map，将cids中每一个cid执行一遍waterfall里的函数。对于每个cid，先判断本地仓库有没有该文件块，没有再调用getFromOutside()从其他节点获取。这里注意的是在判断完所有cid后才调用this.wm.wantBlocks(wantList)一次性将所有只能从外部获取的文件块cid传给wm子功能，这个子功能再负责从其他节点获取想要的文件块。图中这些子功能都是在bitswap初始化中添加的，详见《IPFS源码分析之Bitswap层（二）》。

先看如果本地仓库有所查找的文件块的情况，这里调用了this.blockstore.get(cid, cb)从本地仓库获取，this.blockstore的来源详见《IPFS源码分析之Bitswap层（一）》，不过这里还需要补充很多。下面这部分可以接着《IPFS源码分析之Bitswap层（一）》看。之前讲到this.blockstore是在FsDatastore类实例上再包装ShardingDatastore类，ShardingDatastore的构造函数及几个重要方法：
 ![](https://i.imgur.com/0VB2bAE.png)

										Fig 8 datastore-core/src/sharding.js
由createOrOpen()函数给FsDatastore类实例添加包装ShardingDatastore类，而ShardingDatastore类还再包装一个KeytransformStore类。
总的效果即get的流程是这样，回到blockstore.get(cid, cb)，这个函数的内容如下：
 ![](https://i.imgur.com/QdSWOws.png)

							Fig 9 ipfs-repo/src/blockstore.js
先将CID转化为key，仅仅是变为buffer。然后调用store.get()，这个函数就是KeytransformStore类下的get()方法：
 ![](https://i.imgur.com/XVPGoC2.png)

									Fig 10 datastore-core/src/keytransfrom.js
这里调用了this.transform.convert(key)对key做转化，该函数就是Fig 8中的_convertKey()函数，在key前面加上shard类型的前缀，shard类型是NextToLast(2)，这个前缀实际上是两个大写字母和一个斜杠’/’，不同CID的前缀字母可能不同。this.child是原始的FsDatastore类实例，所以最后还是调用了FsDatastore类的get方法：

 ![](https://i.imgur.com/m7iCg83.png)

									Fig 11 datastore-fs/src/index.js
顺便把has()函数也贴出来，之前判断文件块CID是否在本地仓库就是用has()方法。图中的get先对key做解析，实际上就是将key拼接到本地仓库self.repo的blocks文件夹路径的后面，以本机为例，得到文件的完整路径就是:
C:\Users\AFEE\.ipfs\blocks\两个大写字母或数字\CID.data

最后再读取这个文件，其内容就是get API所查找CID的在本地仓库存储的文件块block，前提是本地仓库有。进去本地仓库self.repo的blocks文件夹即C:\Users\AFEE\.ipfs\blocks\，可以看到很多以两个大写字母或数字命名的子文件夹，在进入子文件夹就能看到至少一个CID.data文件。
 ![](https://i.imgur.com/vozTjrc.png)

												Fig 12 本地仓库的blocks文件夹
不同CID的计算出来的前缀也可能是相同的，所以在一个子文件夹里有多个CID.data文件。

下一篇再看看根据CID从其他节点获取文件块block。

**个人微信：xxb3059 & 公众号：区块丛林 欢迎交流**