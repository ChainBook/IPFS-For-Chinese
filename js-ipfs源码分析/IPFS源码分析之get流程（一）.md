# IPFS源码分析之get流程（一） #
接上篇分析的API请求和响应过程，以get API为例，最后跳到get API对应路由的处理函数，如下：
 ![](https://i.imgur.com/J8vsCBO.png)

										Fig 1 ipfs/src/http/api/resources/files.js
其中parseArgs是预处理函数，主要作用是检测所传入的CID是不是合格的，再以一定的object格式返回。handler既是真正的处理函数，从request接收预处理返回的内容。在这个函数里由ipfs.files.get(cid，……) 根据CID从IPFS网络中获得对应的文件或文件夹，全部以流的形式返回，后半部分代码则根据返回的流区分出文件或文件夹，用一个streams2流接收，全部接收完后reply给请求者。streams2流在nodejs里的网络请求和返回比较常用，尤其是文件的传输，因为浏览器接受到流数据后可以马上解析，不用等到全部接收完，这样会占用很多缓存。

再具体看看ipfs.files.get(cid，……)怎么获取对应CID的文件流，其中ipfs是上篇说到的IPFS Core实例，从它的index.js文件可以看到ipfs.files的引进。
 ![](https://i.imgur.com/O9KFRAz.png)

										Fig 2 ipfs/src/core/components/files.js
从代码图中可看ipfs.files.get(cid，……)实际上是被promisify()化的一个函数，简化回调深度。在这个函数里及后面的很多函数都大量使用pull家族的流操作，功能非常强大，具体的使用方式可以参考https://github.com/dominictarr/pull-stream-examples。

图中的pull()包含了三个操作步骤：

1.exporter()函数从IPFS网络获得的DAGNode信息流作为输入流；

2.pull.asyncMap()函数按照一定的的逻辑处理每个DAGNode信息流；

3.pull.collect(callback)函数收集所有第二步处理后的流返回给回调函数。

最后返回的也是一个pull流。由于所有的文件或文件夹在IPFS上以Merkle DAG树来组织，DAGNode则是这个树的一个节点，一个DAGNode可以是一个完整的文件，可以是一个大文件的一部分，也可以表示一个文件夹等等。每个DAGNode都有一个CID，所以根据CID来get，实际上是在IPFS上成千上万的Merkle DAG树找某一棵的一个DAGNode，而且如果这个DAGNode不是叶子节点，会迭代获取以该DAGNode为根的所有子孙DAGNode。这个由exporter()函数实现, exporter = require('ipfs-unixfs-engine').exporter。
 ![](https://i.imgur.com/eTU24Yz.png)

									Fig 3 ipfs-unixfs-engine/src/exporter/index.js
还有一个importer()函数，不难想到exporter()负责从IPFS网络下载数据，importer()函数负责上传数据到IPFS网络。Fig 2中调用exporter传入的第一个参数是ipfspath，之前一直简化当做是一个纯CID，这个ipfspath是get API请求直接带进来的参数，没有经过任何加工。它可以是个具体路径如：CID/子文件名，这种情况是我们知道某个文件夹的CID和某个子文件的名字，这样可以直接获得这个子文件而不需要知道这个子文件的CID。
exporter()的第二个传入参数是self._ipld，这个在IPFS Core实例的index.js文件中引入：

 ![](https://i.imgur.com/5e3aBpj.png)

							Fig 4 ipfs/src/core.index.js
这个文件是实例化一个IPFS节点类也既是IPFS Core的入口文件，所以之前说的libp2p和bitswap都是这个实例的属性。self._ipld是一个Ipld类实例，这个类的构造又依赖BlockService类的实例，BlockService类的构造依赖之前《IPFS源码分析之Bitswap层（一）》中的this._repo。
可以看看Ipld类的构造函数：

 ![](https://i.imgur.com/XXiRMtx.png)

											Fig 5 ipld/src/index.js
IPLD是IPFS的关键，是一个文件类型转换接口，所有数据格式比如Bitcoin的区块，必须由Bitcoin特有的方式封装成在IPLD层统一的数据块，由于图片大小有限，IPLD还实现很多种数据转换的接口，属于ETH的比较多：EthAccountSnapshot，EthBlock，EthBlockList，EthStateTrie，EthStorageTrie，EthTx，EthTxTrie。还有zcash的，不过目前版本还没看到EOS的数据接口。具体地封装成数据块就要用到它所依赖的BlockService类，最终存储到this._repo表示的本地仓库中也是一个个数据块blocks。

回到Fig 3，具体的获取DAGNode的函数是createResolver(dag, options)，也是在一个pull()中，接收pull.values()产生的流，再根据这个流的内容即CID等获取DAGNode。

 ![](https://i.imgur.com/9ddTvWe.png)

									Fig 6 ipfs-unixfs-engine/src/exporter/resolve.js
由图中createResolver()函数的实现，它只能这么用在pull()中。图中paramap((item, cb)的item即是pull.values()的内容。最后由dag.get()获得CID对应的那个DAGNode，dag就是前面所说的ipld对象。其他代码包括resolveItem()，resolve()实现迭代获取子孙DAGNode的逻辑。
resolve()中有个DAGNode类型判断type = typeOf(node)，根据类型的不同有不同的迭代获取子孙节点的函数。有四种类型：directory(CID对应的是一个目录)，'hamt-sharded-directory'(CID对应的是个大文件，被分隔存储)，file(CID对应一个叶子节点)，object(上面所说的get参数的一种情况，即CID/子文件名)。以object的为例：
 ![](https://i.imgur.com/07q3mHV.png)

								Fig 7 ipfs-unixfs-engine/src/exporter/object.js
由于get API传入的ipfs路径参数是：CID/子文件名。在Fig 6中获得CID对应的文件夹的信息DAGNode，还要获取该子文件对应DAGNode，可以看到Fig 6中createResolver()函数作为第五个参数传入到Fig 7的函数中，然后被再次执行。

下次再看dag.get()怎么获得DAGNode。

**个人微信：xxb3059 & 公众号：区块丛林 欢迎交流**