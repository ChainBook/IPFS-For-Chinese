# IPFS源码分析之add流程（二） #
上次说到impoter()函数作为文件上传的IPFS网络的入口，这篇涉及到的文件DAG树，文件块block和IPLD层具体参考get流程的相关部分，如果对它们之间的关系没有一定的了解，在这篇可能会有很多疑问。

先看看impoter()函数：

    const chunkers = {
      fixed: require('../chunker/fixed-size')
    }
    
    const defaultOptions = {
      chunker: 'fixed'
    }
    
    module.exports = function (ipld, _options) {
      const options = Object.assign({}, defaultOptions, _options)
      const Chunker = chunkers[options.chunker]
      //这里值得注意，如果原本_options中的chunker项有内容会报assert错误，chunker项为空才可以，也就是用户不能使用chunker项
      //默认使用'fixed'
      assert(Chunker, 'Unknkown chunker named ' + options.chunker)
    
      let pending = 0
      const waitingPending = []
    
      const entry = {
    //这里用到pull-write，第二个参数设为null，第三个参数为1表示缓冲区每次进来一个元素，都马上将该元素输出
    sink: writable(
      //nodes参数即是接收的所有{path: ,content: }结构对象
      (nodes, callback) => {
    //计算总共有多少个上述结构对象
    pending += nodes.length
    //将每一个该结构对象放入source给下一步处理函数dagStream
    nodes.forEach((node) => entry.source.push(node))
    setImmediate(callback)
      },
      null,
      1,
      (err) => entry.source.end(err)
    ),
    source: pushable()
      }
      const dagStream = DAGBuilder(Chunker, ipld, options)
      const treeBuilder = createTreeBuilder(ipld, options)
      const treeBuilderStream = treeBuilder.stream()
      const pausable = pause(() => {})
    
      // TODO: transform this entry -> pausable -> <custom async transform> -> exit
      // into a generic NPM package named something like pull-pause-and-drain
    
      pull(
	//由entry.sink接收外部输入流，加工后再从entry.source流出
    entry,
    pausable,
    //对每个{path: ,content: }结构对象即文件分别构造一个DAG树
    dagStream,
    pull.map((node) => {
      pending--
      //当pending减至0
      if (!pending) {
    process.nextTick(() => {
      while (waitingPending.length) {
    waitingPending.shift()()
      }
    })
      }
      return node
    }),
    //对属于一个目录的的文件再构造成一棵大的DAG树
    treeBuilderStream
      )
    
      return {
    sink: entry.sink,   //writable
    source: treeBuilderStream.source,   //pushable()
    flush: flush
      }
这个函数很长，截取了大半部分，基本上相关的重要流程都在上面，完整函数在/ipfs-unixfs-engine/src/importer/index.js中。
函数一开始就检测原本_options中的chunker项的内容，这一项的作用在上篇有说，但是根据代码逻辑，在当前nodejs版本的IPFS中并不支持add API调用者自己选择文件分片的算法，尤其是官方给出的可选项rabin算法，在这里只能使用默认的按照最大值来分片，自己可以提供这个值，也可以使用默认的值262144，后面会讲到。

> 上面代码主要用了pull家族的through流和pull-write模块，函数中的entry代码块就是一个through流，由sink接收外部输入，加工之后从source输出，所以sink经常使用pull-write模块，因为该模块的接收函数writable能够自由设置缓冲区。

函数的主要逻辑还是在一个pull块中,所有的{path: ,content: }结构对象先经过entry.sink，统计一下个数，记录在pending。然后从entry.source出来交给dagStream处理，也就是DAGBuilder()函数：

    const reducers = {
      flat: require('./flat'),
      balanced: require('./balanced'),
      trickle: require('./trickle')
    }
    //这三个项在这里首次出现
    const defaultOptions = {
      strategy: 'balanced',
      highWaterMark: 100,
      reduceSingleLeafToSelf: false
    }
    
    module.exports = function (Chunker, ipld, _options) {
      assert(Chunker, 'Missing chunker creator function')
      assert(ipld, 'Missing IPLD')
      //将defaultOptions中的三个项和_options合并
      const options = Object.assign({}, defaultOptions, _options)
    
      const strategyName = options.strategy
      //默认为balanced
      const reducer = reducers[strategyName]
      assert(reducer, 'Unknown importer build strategy name: ' + strategyName)
    
      const createStrategy = Builder(Chunker, ipld, reducer, options)
    
      return createBuildStream(createStrategy, ipld, options)
    }
在这个函数中还是先检测配置选项，没有就用默认的，这个配置选项是关于如何构建文件的DAG树，有三个可选方式：flat，balanced和trickle，但是这个选项不是提供给add API调用者的，而是服务器端的配置选项，默认是balanced。上面的代码重点在最后两行，分别是建立一个Builder()函数对象赋给createStrategy，然后将createStrategy作为参数传给createBuildStream()函数，最后将其返回，继续进去看createBuildStream函数：

    module.exports = function createBuildStream (createStrategy, _ipld, options) {
      const source = pullPushable()
    
      const sink = pullWrite(
    createStrategy(source),
    null,
    options.highWaterMark,//默认100
    (err) => source.end(err)
      )
    
      return {
    source: source,
    sink: sink
      }
    }
这函数也是一个through流，外部传进的数据即{path: ,content: }结构对象先由sink处理，在sink中直接就是调用createStrategy，把source作为参数传给它，也就是它处理后的结果直接就传给source再流出去。所以继续看createStrategy：

    const DAGNode = dagPB.DAGNode
    
    const defaultOptions = {
      chunkerOptions: {
    maxChunkSize: 262144//这个默认值正是官方API给出的默认分片大小
      }
    }
    
    module.exports = function (createChunker, ipld, createReducer, _options) {
      //又添加一个默认的设置项chunkerOptions，注意和_options.chunker项的区别
      const options = extend({}, defaultOptions, _options)
    
      return function (source) {
    //items接收的是所有{path: ,content: }结构对象
    return function (items, cb) {
      //对每个该结构对象分别处理
      parallel(items.map((item) => (cb) => {
    //根据结构对象中的content项有无内容来区分是一个目录还是文件构建不同的文件块block和DAG node
    if (!item.content) {
      // item is a directory
	  //如果是目录
      return createAndStoreDir(item, (err, node) => {
    if (err) {
      return cb(err)
    }
    if (node) {
      source.push(node)
    }
    cb()
      })
    }
    
    // item is a file
	//如果是文件
    createAndStoreFile(item, (err, node) => {
      if (err) {
    return cb(err)
      }
      if (node) {
    source.push(node)
      }
      cb()
    })
      }), cb)
    }
      }
在这个函数里开始对每个{path: ,content: }结构对象分别处理，根据content有无内容即是目录或是文件，交给不同函数处理。这里的目录上篇讲过，并不是文件所属目录，而是add API调用者上传的仅有目录路径，没有内容，所以content为空，这样的目录交由createAndStoreDir()函数处理，如果是文件即content不为空交由createAndStoreFile()函数处理。下面先贴出createAndStoreDir()函数：

    function createAndStoreDir (item, callback) {
    // 1. create the empty dir dag node
    // 2. write it to the dag store
    //构建目录文件块block，这里的参数是'directory'
    const d = new UnixFS('directory')
    
    waterfall([
      //根据文件块创建DAGNode，由于该目录没有指定包含文件，所以DAGlink为空(第二个参数)
      (cb) => DAGNode.create(d.marshal(), [], options.hashAlg, cb),
      (node, cb) => {
    //如果设置了onlyHash，直接返回上面的node
    if (options.onlyHash) return cb(null, node)
    
    let cid = new CID(node.multihash)
    
    if (options.cidVersion === 1) {
      cid = cid.toV1()
    }
    //上传到IPLD层统一管理并加入到dht，进行分布式存储
    ipld.put(node, { cid }, (err) => cb(err, node))
      }
    ], (err, node) => {
      if (err) {
    return callback(err)
      }
      //以下面结构返回
      callback(null, {
    path: item.path,
    multihash: node.multihash,
    size: node.size
      })
    })
      }
代码已有详细注释，处理过程比较简单，先对该目录创建文件块block（这里的文件块block是blockservice层管理的，详见之前的get流程），再依次block创建DAGNode，由ipld.put()函数将DAGNode上传到IPLD层统一管理并加入到dht，进行分布式存储，最后就直接返回相关信息，由于。需要重点关注的是处理文件的createAndStoreFile()函数：

    function createAndStoreFile (file, callback) {
    if (Buffer.isBuffer(file.content)) {
      file.content = pull.values([file.content])
    }
    
    if (typeof file.content !== 'function') {
      return callback(new Error('invalid content'))
    }
    
    const reducer = createReducer(reduce(file, ipld, options), options)
    
    let previous
    let count = 0
    
    pull(
      file.content,
      //将file.content切片成多个指定大小的块chunk，默认大小262144
      createChunker(options.chunkerOptions),
      pull.map(chunk => {
    //分别处理每个块的长度
    if (options.progress && typeof options.progress === 'function') {
      options.progress(chunk.byteLength)
    }
    return Buffer.from(chunk)
      }),
      //根据chunk构建文件块block，这里的参数是'file'
      pull.map(buffer => new UnixFS('file', buffer)),
      //fileNode = buffer(block)
      pull.asyncMap((fileNode, callback) => {
    //根据文件块block创建DAGNode，由于此时创建的节点都是叶子节点，所以没有DAGlink(第二个参数)
    DAGNode.create(fileNode.marshal(), [], options.hashAlg, (err, node) => {
      callback(err, { DAGNode: node, fileNode: fileNode })
    })
      }),
      pull.asyncMap((leaf, callback) => {
    if (options.onlyHash) return callback(null, leaf)
    
    let cid = new CID(leaf.DAGNode.multihash)
    
    if (options.cidVersion === 1) {
      cid = cid.toV1()
    }
    //上传到IPLD层统一管理并加入到dht，进行分布式存储
    ipld.put(leaf.DAGNode, { cid }, (err) => callback(err, leaf))
      }),
      pull.map((leaf) => {
    //以下面结构对象返回
    return {
      path: file.path,
      multihash: leaf.DAGNode.multihash,
      size: leaf.DAGNode.size,
      leafSize: leaf.fileNode.fileSize(),
      name: ''
    }
      }),
      through( // mark as single node if only one single node
    function onData (data) {
      count++
      if (previous) {
    this.queue(previous)
      }
      previous = data
    },
    function ended () {
      if (previous) {
    if (count === 1) {
      //如果仅有一个node，添加single项作为标志
      previous.single = true
    }
    this.queue(previous)
      }
      this.queue(null)
    }
      ),
      //将所有叶子node整合成一棵DAG树，这里的node是指前面返回的结构对象
      reducer,
      //返回这个文件的树根
      pull.collect((err, roots) => {
    if (err) {
      callback(err)
    } else {
      callback(null, roots[0])
    }
      })
    )
      }
    }
上面代码中先是由createReducer()创建reducer，之后的主要逻辑在pull代码块中，文件的file.content作为数据源先由createChunker()函数处理，这个函数溯源到前面DAGBuilder()函数中便可知它就是对文件切片的函数。所以这里就对文件的内容进行切片，参数options.chunkerOptions决定切片的大小，如果调用者不设置则默认大小为262144，如下代码：

	onst pullBlock = require('pull-block')
    module.exports = (options) => {
      let maxSize = (typeof options === 'number') ? options : options.maxChunkSize   //默认为262144
      //将大于maxSize的输入流切分成多个maxSize大小的输出流，第二个参数为false表示最后一个不满maxSize的输出流不用0来填充
      return pullBlock(maxSize, { zeroPadding: false, emitEmpty: true })
    }
有兴趣可以看pullBlock()函数具体怎么切，回到处理流程，假设文件很大，切片完后变成多个maxChunkSize大小的chunk，然后pull.map()将每个chunk新建UnixFS类对象，即是文件块block，再根据文件块block创建DAGNode，由于此时创建的节点都是叶子节点DAGNode，每个叶子节点就是文件的一片，对blockservice层来说每个文件块block就是最小的可寻址对象，他们之间不需要有任何联系（注意，文件块block只与存储它们的IPFS节点ID有关系，它们构成了DHT表，这个工作由上面的ipld.put()函数来完成），但是对IPLD层来说，根据属于同一个大文件的文件块blocks创建的各个DAGNode是有联系的，否则在获取该文件时，找不到被切片后分散的文件块block。

如何构建联系就是由前面创建的reducer来处理，将所有叶子节点整合成一棵DAG树，这棵树就代表这个文件。前面提到构建算法一般使用默认的balanced。看看这个算法：

    module.exports = function balancedReduceToRoot (reduce, options) {
      const pair = pullPair()
      const source = pair.source
      const result = pushable()
      //外部数据从pair.sink流入，在流出到pair.source，由source参数接收
      reduceToParents(source, (err, roots) => {
    if (err) {
      result.end(err)
      return // early
    }
    if (roots.length === 1) {
      result.push(roots[0])
      result.end()
    } else if (roots.length > 1) {
      result.end(new Error('expected a maximum of 1 roots and got ' + roots.length))
    } else {
      result.end()
    }
      })
    	//处理所有叶子节点构建DAG树
      	function reduceToParents (_chunks, callback) {
    	let chunks = _chunks
    if (Array.isArray(chunks)) {
      chunks = pull.values(chunks)
    }
    
    pull(
      chunks,
	  //将所有叶子节点分批，每批不超过174个
      batch(options.maxChildrenPerNode),//maxChildrenPerNode: 174
      //构造一个父节点，将这批叶子节点附上
      pull.asyncMap(reduce),
      //当叶子节点大于174时会被分成多批处理，所以会有多个父节点
      //收集上步返回的父节点，继续整合处理
      pull.collect(reduced)
    )
    
    function reduced (err, roots) {
      if (err) {
    callback(err)
      } else if (roots.length > 1) {
    //如果返回的根节点数量大于1，继续递归整合
    reduceToParents(roots, callback)
      } else {
    //返回最后剩下的一个根节点
    callback(null, roots)
      }
    }
      }
    
      return {
    sink: pair.sink,
    source: result
      }
    }
balancedReduceToRoot()函数整体也是一个through流，在在上面代码中外部传入的数据即某个文件的所有叶子节点，从pair.sink进去后，马上从pair.source出来（这里的pair只是提供一个双工流管道的作用），处理这些叶子节点的是中间的reduceToParents()函数。

在reduceToParents()函数里，主要逻辑也还是在pull代码块中，chunks就是这些叶子节点，然后batch将他们打包分批，因为options.maxChildrenPerNode的默认配置为174，也就是一个父节点最多有174个子节点，所以如果叶子节点总数大于174，需要分批处理，当然如果不到174个就只有一批，pull.asyncMap()将每一批分别交给reduce()函数，构造一个父节点，将这批叶子节点附上，如果有多批就有多个父节点，所以还要向上递归处理，直到只有一个树根，就是这个文件的DAG树的根。可以看看reduce()函数：

    module.exports = function (file, ipld, options) {
      //leaves参数接收的数据元素结构为{path:，multihash:，size:，leafSize:，name:}，如果只有一个叶子节点，还有single项
      return function (leaves, callback) {
    //如果只有一个叶子节点，直接以以下结构返回
    if (leaves.length === 1 && (leaves[0].single || options.reduceSingleLeafToSelf)) {
      const leave = leaves[0]
      callback(null, {
    path: file.path,
    multihash: leave.multihash,
    size: leave.size,
    leafSize: leave.leafSize,
    name: leave.name
      })
      return // early
    }
    
    // create a parent node and add all the leafs
    const f = new UnixFS('file')
    // 构造父文件块block
    const links = leaves.map((leaf) => {
	  //将所有子节点文件块总的大小记录到父文件块
      f.addBlockSize(leaf.leafSize)
      //构造指向这些叶子节点的DAGLink
      return new DAGLink(leaf.name, leaf.size, leaf.multihash)
    })
    
    waterfall([
      //根据父文件块block创建父DAGNode，并添加DAGlink
      (cb) => DAGNode.create(f.marshal(), links, options.hashAlg, cb),
      (node, cb) => {
    if (options.onlyHash) return cb(null, node)
    
    let cid = new CID(node.multihash)
    
    if (options.cidVersion === 1) {
      cid = cid.toV1()
    }
    //上传到IPLD层统一管理并加入到dht，进行分布式存储
    ipld.put(node, { cid }, (err) => cb(err, node))
      }
    ], (err, node) => {
      if (err) {
    callback(err)
    return // early
      }
    
      const root = {
    name: '',
    path: file.path,
    multihash: node.multihash,
    size: node.size,
    leafSize: f.fileSize()
      }
      //返回文件DAG树父node
      callback(null, root)
    })
      }
    }
主要过程是先构造一个父文件块block，再将所有子节点文件块总的大小记录到父文件块，重要的是对每一个叶子节点都创建一个DAGLink，通过DAGLink就能找到每个叶子节点，然后根据父文件块block创建父DAGNode，再把刚刚创建的那些DAGLink添加到父DAGNode中，同样要将该父DAGNode传到IPLD层统一管理并加入到dht，进行分布式存储，最后返回父DAGNode。

以上就是如何把一个文件切片后在IPLD层构建文件DAG树，尤其是对于一个大文件构建的DAG树会是多层的。但构建算法这里只看了默认的balanced算法，其他的两种有兴趣可以看看和比较不同。如果add API调用者一次性上传一个目录下多个文件，则在第一个代码段的末尾还要用到treeBuilderStream()函数对属于一个目录的的文件再构造成一棵更大的DAG树，这个就不详细展开了。还余留的一个重要的点是上面的几段代码都出现的函数ipld.put(),将各个DAGNode上传到IPLD层统一管理并把对应的文件块block加入到dht，进行分布式存储，也就是存到世界各地。下次再看。
