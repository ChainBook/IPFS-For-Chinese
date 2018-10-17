# IPFS源码分析之add流程（三） #
> 这一篇是add流程的结束，到DHT模块告知网络中距离文件块CID（转化为DHT ID后）最近的20个节点更新DHT表，有很多函数在get流程都是见过的，也大量比较了和get流程中这些函数功能的区别，看到这里也意识到了之前篇章的两个错误并改正，果然不能仅凭冰山一角来猜测整个冰山之大，常常会出错。
先继续看ipld.put()函数：

    put (node, options, callback) {
    if (typeof options === 'function') {
      callback = options
      return setImmediate(() => callback(
    new Error('IPLDResolver.put requires options')
      ))
    }
    callback = callback || noop
    //如果参数options中有CID，这个DAGnode属于普通文件，则直接put到网络
    if (options.cid && CID.isCID(options.cid)) {
      return this._put(options.cid, node, callback)
    }
    //下面是options中没有CID的情况，即其他独特数据类型
    // 如果不指定哈希函数，默认使用sha2-256
    options.hashAlg = options.hashAlg || 'sha2-256'
    //根据options.format获得对应的resolver
    const r = this.resolvers[options.format]
    if (!r) {
      return callback(new Error('No resolver found for codec "' + options.format + '"'))
    }
    // TODO add support for different hash funcs in the utils of
    // each format (just really needed for CBOR for now, really
    // r.util.cid(node1, hashAlg, (err, cid) => {
    // 计算该DAGNode的CID
    r.util.cid(node, (err, cid) => {
      if (err) {
    return callback(err)
      }
      
      this._put(cid, node, callback)
    })
      }
add上传的普通文件（包括图片等）默认使用sha2-256作为数据的哈希函数，当然也可以在之前所说的hashAlg选项中指定。总体上，IPFS支持multihash，即多种哈希函数来对数据加密，以达到更安全的效果，一般配合上面代码中的resolver函数来使用，不同数据类型的resolver不同，它内部的哈希函数也不同，关于不同数据类型的resolver在IPFS的注册详见get流程，比如Bitcoin数据，以太坊的交易，状态等数据都有不同的resolver，但是这些独特数据和普通文件不一样的是它们不是通过调用add API来上传，从前面的流程来看add API没有提供可选项来指定使用哪个resolver，但是这些独特的数据类型调用ipld.put()函数时传入了options.format选项，在上面的代码中可以看到，通过这个选项来找到对应的resolver。

不过奇怪的是，如果仔细看前面流程的代码会发现，对用add API上传的普通文件构造的所有DAGNode都是调用dagPB，可以认为属于dagPB这样的数据类型，在所有已经注册的resolver函数中第一个就是dagPB数据的resolver。所以为什么不在add API直接就提供resolver选项，仍然可以缺省默认dagPB。也许是因为IPFS直接暴露了IPLD层的API，也就是ipld.put()和ipld.get()可以作为API调用，这样的好处是其他独特类型的数据不需要经过转化为DAGNode而是保留原来的数据结构。**所以，这里修改之前《IPFS源码分析之get流程（二）》中的一个错误说法，Fig 3下面的一段话**：
> 各种数据类型（CBOR，Bitcoin，ETH系列等）都有自己的一套方法将数据转换成Merkle DAG树上的一个个DAGNode，以Bitcoin为例，一个区块可以成为一个Merkle DAG树。然后将每个DAGNode的内容序列化，也根据内容计算出CID，然后将两者由blockservice统一封装成一个个block，最后Bitswap层的节点之间交换的数据就是block。

**更正为：**这些数据类型不需要转化Merkle DAG树上的一个个DAGNode，而是保留原来的数据格式。然后不同数据类型由不同的resolver来计算出CID，因为CID的前四个字节（称为multicode）来表示使用的不同resolver，包括普通文件所属的dagPB数据类型。

在上面代码中，不管是普通文件还是其他数据类型，最后都调用了this._put()函数：

    _put (cid, node, callback) {
    callback = callback || noop
    //再取出对应的resolver
    const r = this.resolvers[cid.codec]
    if (!r) {
      return callback(new Error('No resolver found for codec "' + cid.codec + '"'))
    }
    
    waterfall([
      //对数据序列化
      (cb) => r.util.serialize(node, cb),
      //把序列化后的数据转化为文件块block，由blockservice层把文件块block分布存储
      (buf, cb) => this.bs.put(new Block(buf, cid), cb)
    ], (err) => {
      if (err) {
    return callback(err)
      }
      callback(null, cid)
    })
      }

函数里又先取出先对应的resolver，这次根据的正式CID的前四个字节。然后对数据序列化，不同resolver的序列化方式可能不同。最后把序列化后的数据转化为文件块block，由blockservice层的bs.put()函数把文件块block分布存储。**这里再修改上一篇的一个错误，在上篇的第五（即createAndStoreDir函数）和第六（即createAndStoreFile函数）中创建DAGNode前都使用了一个UnixFS()函数，这个函数的作用当时理解有误，它并没有构建文件块block，而仅仅是将原始数据根据Unix的文件系统数据类型包裹一下，真正构建文件块block是在上面代码的Block(buf, cid)函数。**

这样IPFS中各层的数据关系就比较清晰了，对于普通文件（dagPB类型数据）先在IPLD层之前转化为DAG文件树和DAGNode，其他数据类型则保持其原始结构。然后经过各自的resolver序列化后，被Block(buf, cid)函数包装为blockservice层的文件块block。

下面继续看bs.put()函数：

      put (block, callback) {
    if (this.hasExchange()) {
      //根据bitswap层的策略分布存储文件块block
      return this._bitswap.put(block, callback)
    }
    //将文件块block存储到本地仓库
    this._repo.blocks.put(block, callback)
      }
只有两个步骤，先调用this._bitswap.put()函数根据bitswap层的策略分布存储文件块block，再调用this._repo.blocks.put()函数将文件块block存储到本地仓库，存储到本地仓库在get流程中常见，所以只需要看看bitswap层分布存储的策略。this._bitswap.put()函数：

    put (block, callback) {
    this._log('putting block')
    
    waterfall([
      //检查本地仓库的blockstore有没有该文件块
      (cb) => this.blockstore.has(block.cid, cb),
      (has, cb) => {
    if (has) {
      //如果有则直接返回
      return cb()
    }
    //没有则存储
    this._putBlock(block, cb)
      }
    ], callback)
      }
先检查本地仓库的blockstore有没有该文件块，如果有则直接返回，没有则存储。存储调用的是this._putBlock()函数，这个函数在《IPFS源码分析之get流程（七）》的Fig 8讲过，在那里调用这个函数的背景是在收到想要的文件块后，将该文件块保存以及其他处理，为了方便这里再把这个函数贴出来：

     _putBlock (block, callback) {
    this.blockstore.put(block, (err) => {
      if (err) {
    return callback(err)
      }
      //让notifications子模块获悉收到想要的文件块并发出通知
      this.notifications.hasBlock(block)
      //更新Libp2p层的DHT，即从DHT可以查找到本节点以后可以提供该CID文件块
      this.network.provide(block.cid, (err) => {
    if (err) {
      this._log.error('Failed to provide: %s', err.message)
    }
      })
      //如果其他节点也向自己索要这些文件块则转发
      this.engine.receivedBlocks([block.cid])
      callback()
    })
      }
上面的代码的注释是在看get流程时写的，所以按照从接收到其他节点发过来自己想要的文件块后的处理描述。现在是add流程，有点殊途同归的意思。也就是不管是由get流程从其他节点收到的文件块还是由add流程上传的文件块对于本节点来说新的文件块，所以这些新的块都会来到这个函数处理，第一步是Bitswap层的notifications子模块处理，它对所有收到的文件块都触发一个`block:cidStr`事件，然后将文件块交给监听函数处理即从Bitswap层的wm子模块的wantlist中删除该文件块CID，但是由于add流程没有像get流程在wm子模块的wantlist注册想要的文件块CID，所以对add上来的文件块wm子模块会忽略掉。

回到上面代码，先说第三步engine.receivedBlocks()函数检查如果其他节点也向自己索要这些文件块则转发，add上传的一般是完全新的一个普通文件，其他节点不会知道，所以这里一般只在get流程中有用。
第二步调用network.provide()函数，这个函数在《IPFS源码分析之get流程（七）》的最后也讲过，但没有深入看下去，因为当时认为对get流程不重要。它的内部直接调用this.libp2p.contentRouting.provide()函数：

    module.exports = (node) => {
      return {
    findProviders: (key, timeout, callback) => {
      if (!node._dht) {
    return callback(new Error('DHT is not available'))
      }
    
      node._dht.findProviders(key, timeout, callback)
    },
    provide: (key, callback) => {
      if (!node._dht) {
    return callback(new Error('DHT is not available'))
      }
    
      node._dht.provide(key, callback)
    }
      }
    }
也就是上面provide项的函数。findProviders项是get流程的关键，负责从DHT表中找到能够提供想要get的CID文件块的节点并连接获取。provide项是add流程的关键，里面主要调用了node._dht.provide()函数：

      /**
       * Announce to the network that a node can provide the given key.
       * This is what Coral and MainlineDHT do to store large values
       * in a DHT.
       *
       * @param {CID} key
       * @param {function(Error)} callback
       * @returns {void}
       */
      provide (key, callback) {
    this._log('provide: %s', key.toBaseEncodedString())
    
    waterfall([
      //将该文件块的CID添加到provider的两个缓存中datastore和LRU Cache
      (cb) => this.providers.addProvider(key, this.peerInfo.id, cb),
      //查找20个距离最近的节点，这里距离是在DHT表计算的VOR距离
      (cb) => this.getClosestPeers(key.buffer, cb),
      (peers, cb) => {
    //向这20个节点发消息，让他们也把该文件块CID添加到他们的provider以及更新他们的DHT表
    const msg = new Message(Message.TYPES.ADD_PROVIDER, key.buffer, 0)
    msg.providerPeers = peers.map((p) => new PeerInfo(p))
    
    each(peers, (peer, cb) => {
      this._log('putProvider %s to %s', key.toBaseEncodedString(), peer.toB58String())
      this.network.sendMessage(peer, msg, cb)
    }, cb)
      }
    ], (err) => callback(err))
      }
先调用providers.addProvider()函数将该文件块的CID添加到provider的两个缓存中datastore和LRU Cache，然后调用getClosestPeers()函数查找20个距离最近的节点，这里距离是在DHT表计算的VOR距离，最后调用network.sendMessage()函数向这20个节点发消息，让他们也把该文件块CID添加到他们的provider以及更新他们的DHT表，这个消息的类型是ADD_PROVIDER，而且把这20个节点的信息都装入消息中，稍后再看那20个节点接收到这个消息的处理。先看查找节点：

    /**
       * Kademlia 'node lookup' operation.
       *
       * @param {Buffer} key
       * @param {function(Error, Array<PeerId>)} callback
       * @returns {void}
       */
      getClosestPeers (key, callback) {
    this._log('getClosestPeers to %s', key.toString())
    utils.convertBuffer(key, (err, id) => {
      if (err) {
    return callback(err)
      }
      //在本地的路由表即KBucket中获得距该CID转换成DHT ID后的VOR距离最近的3个节点
      const tablePeers = this.routingTable.closestPeers(id, c.ALPHA)
    
      const q = new Query(this, key, (peer, callback) => {
    waterfall([
      //向每个节点查询距离较近的3个节点，并且延伸查询
      (cb) => this._closerPeersSingle(key, peer, cb),
      (closer, cb) => {
    cb(null, {
      closerPeers: closer
    })
      }
    ], callback)
      })
      //对最近的3个节点执行以上查询
      q.run(tablePeers, (err, res) => {
    if (err) {
      return callback(err)
    }
    
    if (!res || !res.finalSet) {
      return callback(null, [])
    }
    
    waterfall([
      //对所有返回结果按照距离进行排列
      (cb) => utils.sortClosestPeers(Array.from(res.finalSet), id, cb),
      //选择前20个返回
      (sorted, cb) => cb(null, sorted.slice(0, c.K))
    ], callback)
      })
    })
      }
这个函数的逻辑和get流程寻找20个可以提供指定CID文件块的节点很像，都是先在本地的路由表即KBucket中获得距该要上传的文件块的CID转换成DHT ID后的VOR距离最近的3个节点，然后询问这三个节点，它们分别返回3个较近的节点ID，逐渐延伸。在延伸查询的过程中都是把文件块CID当做节点ID来比较VOR距离的远近。不过与get流程的区别是，后者只要找到20个就马上返回，而且对每个节点询问时间限制一分钟。add流程没有限制，直到找完整个网络，最后再对所有返回的节点按距离大小排序，选择最近的20个。

之前在get流程详细讲过消息的发送和建立连接。这里发送消息前建立连接虽然也是用Libp2p层switch.dial()，但是和Bitswap的请求和发送文件块消息使用的协议名字不一样，这里用的是DHT，路由前缀变成'/ipfs/kad/1.0.0'，消息的格式也不一样，但过程都一样，所以DHT可以看做是和Bitswap等层的，它们都使用Libp2p层来建立连接，各自有自己的一套消息的收发方法和格式。

最后看看那20个节点接收到这个消息的处理，switch子模块管理的连接收到消息后根据协议名字为DHT则将其送到DHT的消息处理函数protocolHandler()：

    /**
       * Handle incoming streams from the Switch, on the dht protocol.
       *
       * @param {string} protocol
       * @param {Connection} conn
       * @returns {undefined}
       */
      return function protocolHandler (protocol, conn) {
    conn.getPeerInfo((err, peer) => {
      if (err) {
    log.error('Failed to get peer info')
    log.error(err)
    return
      }
    
      log('from: %s', peer.id.toB58String())
    
      pull(
    conn,
    lp.decode(),
    //过滤掉过大的消息
    pull.filter((msg) => msg.length < c.maxMessageSize),
    pull.map((rawMsg) => {
      let msg
      try {
    //反序列化
    msg = Message.deserialize(rawMsg)
      } catch (err) {
    log.error('failed to read incoming message', err)
    return
      }
    
      return msg
    }),
    pull.filter(Boolean),
	//处理消息内容
    pull.asyncMap((msg, cb) => handleMessage(peer, msg, cb)),
    // Not all handlers will return a response
    pull.filter(Boolean),
    pull.map((response) => {
      let msg
      try {
    msg = response.serialize()
      } catch (err) {
    log.error('failed to send message', err)
    return
      }
      return msg
    }),
    pull.filter(Boolean),
    lp.encode(),
    conn
      )
    })
      }
    }
虽然比较长，但主要的逻辑就两步，对消息反序列化，还原真实内容，再调用handleMessage()函数根据消息的类型来用不同的处理函数，DHT协议总共有6个消息类型：add-provider，find-node，get-provider，get-value，ping和put-value。其中find-node和get-provider在get流程中也出现过，但没有仔细看。注意和Bitswap协议的消息类型如get-block区分。这里以add-provider消息类型为例，它的处理函数如下：

    function addProvider (peer, msg, callback) {
    log('start')
    
    if (!msg.key || msg.key.length === 0) {
      return callback(new Error('Missing key'))
    }
    
    let cid
    try {
      cid = new CID(msg.key)
    } catch (err) {
      return callback(new Error('Invalid CID: ' + err.message))
    }
    
    msg.providerPeers.forEach((pi) => {
      // Ignore providers not from the originator
      if (!pi.id.isEqual(peer.id)) {
    log('invalid provider peer %s from %s', pi.id.toB58String(), peer.id.toB58String())
    return
      }
    
      if (pi.multiaddrs.size < 1) {
    log('no valid addresses for provider %s. Ignore', peer.id.toB58String())
    return
      }
    
      log('received provider %s for %s (addrs %s)', peer.id.toB58String(), cid.toBaseEncodedString(), pi.multiaddrs.toArray().map((m) => m.toString()))
    
      if (!dht._isSelf(pi.id)) {
    dht.peerBook.put(pi)
      }
    })
    //将消息中的CID以及可以提供该CID文件块的20个节点信息存入LRU Cache和datastore缓存
    dht.providers.addProvider(cid, peer.id, callback)
      }
    }
对消息的内容经过一系列检测后，调用providers.addProvider()函数将消息中的CID以及可以提供该CID文件块的20个节点信息存入LRU Cache和datastore缓存。这个函数往上翻可以看到，到此add流程就结束了。

但是似乎还有些事没完成。从在IPLD层将普通文件转化为DAG文件树和DAGNode到在blockservice层包装为文件块，最后到DHT模块告知网络中距离文件块CID（转化为DHT ID后）最近的20个节点更新DHT表，却未发现有那个环节或函数将文件块发给那20个最近的节点。由于DHT和Bitswap是同一层的功能，虽然Bitswap可调用DHT，但后者没有调用前者的函数。而且DHT模块发送的消息Bitswap层是无法知悉的。有一种可能是那20个节点在有了get请求后再找本节点要这个文件块，本节点并没有主动发送文件块给他们，也有可能中间某个函数被我忽略了，这个疑问先留着吧，欢迎解疑。