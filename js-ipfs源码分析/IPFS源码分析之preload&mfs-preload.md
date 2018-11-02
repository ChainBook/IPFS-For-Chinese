# IPFS源码分析之preload&mfs-preload #
js-ipfs在7月份更新后主要多了三个东西：preload，mfs-preload和ipns，在内核的components还增加了name和resolve组件。ipns在go版本中早就有了，至于它的作用了解过ipfs的都知道，有了它，以纯js在ipfs上架设网站就方便很多，下次单独拎出来分析，这里主要看看preload和mfs-preload。

preload在前面分析add API时就遇到过，在importer()函数上传完文件后，接着调用了preloadFile()函数，在这个函数里再调用了self._preload(file.hash)，仔细看了下这个东西，似乎可以解决在分析add API最后留下的疑问。也就是add一个文件到IPFS网络中后，本节点没有主动将文件块备份发到其他节点，而preload就实际地让其他节点主动地去拿文件块备份，至少ipfs官方网关的节点会拿到。下面先从ipfs内核引入这个模块说起。

![](https://i.imgur.com/sfHa04m.png)


带+号和小方块的都是js-ipfs内核新增加代码，23行引入ipns模块，29行引入preload模块，30引入mfs-preload模块，还有下面的41行到46行是新增的对preload的初始配置，这个初始配置很重要。接着在后面实例化：

    this._preload = preload(this)
    this._mfsPreload = mfsPreload(this)
    this._ipns = new IPNS(null, this)

this指的是整个ipfs对象。先看preload()函数，在js-ipfs/src/core/preload.js中：

    module.exports = self => {
      /* self._options.preload
      preload: {
    enabled: true,
    addresses: [
      '/dnsaddr/node0.preload.ipfs.io/https',
      '/dnsaddr/node1.preload.ipfs.io/https'
    ]
      }
       */
      const options = self._options.preload || {}
      options.enabled = Boolean(options.enabled)
      options.addresses = options.addresses || []
      //如果enabled为假或addresses为空，则构造一个空的api
      if (!options.enabled || !options.addresses.length) {
    const api = (_, callback) => {
      if (callback) {
    setImmediate(() => callback())
      }
    }
    api.start = () => {}
    api.stop = () => {}
    return api
      }
    
      let stopped = true
      let requests = []
      //对addresses中的每一个地址做格式替换
      const apiUris = options.addresses.map(apiAddrToUri)
    
      const api = (cid, callback) => {
    callback = callback || noop
    //处理CID
    if (typeof cid !== 'string') {
      try {
    cid = new CID(cid).toBaseEncodedString()
      } catch (err) {
    return setImmediate(() => callback(err))
      }
    }
    
    const fallbackApiUris = Array.from(apiUris)
    let request
    const now = Date.now()
    //如果执行出错，fallbackApiUris中有多少个地址就重试多少次
    retry({ times: fallbackApiUris.length }, (cb) => {
      if (stopped) return cb(new Error(`preload aborted for ${cid}`))
    
      // Remove failed request from a previous attempt
      requests = requests.filter(r => r !== request)
      //拿出一个新的的地址
      const apiUri = fallbackApiUris.shift()
      //开始从ipfs官方网关的节点preload
      request = preload(`${apiUri}/api/v0/refs?r=true&arg=${cid}`, cb)
      //记录用过的request
      requests = requests.concat(request)
    }, (err) => {
      requests = requests.filter(r => r !== request)
    
      if (err) {
    return callback(err)
      }
    
      log(`preloaded ${cid} in ${Date.now() - now}ms`)
      callback()
    })
      }
    
      api.start = () => {
    stopped = false
      }
    
      api.stop = () => {
    stopped = true
    log(`canceling ${requests.length} pending preload request(s)`)
    requests.forEach(r => r.cancel())
    requests = []
      }
    
      return api
    }
    
    function apiAddrToUri (addr) {
      //如果addr中不以http或者https为后缀，则默认使用http
      if (!(addr.endsWith('http') || addr.endsWith('https'))) {
    addr = addr + '/http'
      }
      //将addr转换成url的标准格式
      return toUri(addr)
    }

先判断preload的初始配置是否开启preload，默认是true。然后对配置的addresses字段的两个地址做转换，转换函数是apiAddrToUri(addr)，在上面代码的末尾，以其中一个地址为例，/dnsaddr/node0.preload.ipfs.io/https，经过转化后变成https://node0.preload.ipfs.io。也就是将ipfs的dns格式地址转换成url格式，在ipfs的bootstrap中也能看到几个类似/dnsaddr/的地址，默认使用ipfs的官方网关ipfs.io，这个网关也起到类似dns的作用，因为新加入的节点几乎都是从ipfs.io获取其他节点的信息。转换成url后可以看到在主域名ipfs.io下，不同的子域名的功能不一样，在三级域名preload下是多个提供preload的第四级域名node0和node1，对应到不同ipfs节点的主机，在默认配置只有两个，我尝试访问node2和node3都无效，估计总的也只有两个，但ipfs.io网关整体会是个大集群。

接着在一个retry代码块中构造完整的url尝试访问https://node0.preload.ipfs.io和https://node1.preload.ipfs.io，retry的作用是如果函数执行出错则重新执行，执行次数有第一个参数的time字段给出，默认没有等待时间。在这里的效果是当对第一个地址执行preload(完整url)返回错误时，换第二个地址。所以继续看看preload(完整url)，这个preload = require('./runtime/preload-nodejs')，继续看：

    module.exports = function preload (url, callback) {
      log(url)
    
      try {
    url = new URL(url)
      } catch (err) {
    return setImmediate(() => callback(err))
      }
    
      const transport = url.protocol === 'https:' ? https : http
      //以http或https请求ipfs官方节点
      const req = transport.get({
    hostname: url.hostname,
    port: url.port,
    path: url.pathname + url.search
      }, (res) => {
    if (res.statusCode < 200 || res.statusCode >= 300) {
      res.resume()
      log.error('failed to preload', url.href, res.statusCode, res.statusMessage)
      return callback(new Error(`failed to preload ${url}`))
    }
    //如果返回数据，仅仅打印返回内容
    res.on('data', chunk => log(`data ${chunk}`))
    
    res.on('abort', () => {
      callback(new Error('request aborted'))
    })
    
    res.on('error', err => {
      log.error('response error preloading', url.href, err)
      callback(err)
    })
    
    res.on('end', () => {
      // If aborted, callback is called in the abort handler
      if (!res.aborted) callback()
    })
      })
    
      req.on('error', err => {
    log.error('request error preloading', url.href, err)
    callback(err)
      })

这个函数传入的参数即是之前构造好的完整url，以第一个地址为例：https://node0.preload.ipfs.io/api/v0/refs?r=true&arg=cid。如果是在add一个文件后调用的preload，url参数的cid即是这个文件的CID或者说hash。在上面的函数中调用了标准的https模块发出请求，然后处理返回内容，如果出错就回到上一层换一个地址，如果没有出错，直接打印返回数据。并没有其他操作，这从另一个方面说明了preload的最大作用并不是在本节点，而是对整个ipfs网络，在本节点上传完文件后及时调用preload模块，ipfs.io网关下的节点群能够主动请求本节点拿到文件备份，而本节点即使执行preload出错也没什么影响。

在浏览器上模拟这个请求：https://node0.preload.ipfs.io/api/v0/refs?r=true&arg=Qmdku9pUysciPiGjDzHRDSD2iomymYpAmguPrNnW2SrLP7，执行成功返回{"Ref":"QmWrWZ1MpR3VvxPBEGwB4MUMHnjfvUf1TS6NjtuaU5DSt4","Err":""}，返回的数据中仅仅包括一个CID，当我get这个CID时发现内容和url中那个CID的文件是一样的，接着在客户端用ipfs object stat CID分别查这两个CID发现了不一样，最后用ipfs object links url中的CID，发现得到的结果正是返回数据的CID，也就是说preload返回一个指向url中要查询的CID的link DAGNode的CID。就没有其他的了，如果查询的CID不存在，则会返回{"Message":"invalid 'ipfs ref' path","Code":0,"Type":"error"}。

以上就是preload，下面继续看mfs-preload。

先说mfs。可变文件系统(MFS)是一个位于IPFS之上的虚拟文件系统，它在一个虚拟目录上公开类似于Unix的API。它允许用户从路径进行读写操作，而不必担心更新该文件及其所属文件夹的DAG树和CID。它使ipfs-blob-store之类的东西能够存在。MFS也有一个根目录，它和IPFS的宿主机文件系统的根目录不一样，它是虚拟的，但是不妨碍在MFS的根目录下进行所有与Unix的API类似的操作，包括cp，ls，flush，mkdir，mv，read，readPullStream，readReadableStream，rm，stat，write。所有这些操作的路径都是基于MFS的根目录，readPullStream和readReadableStream这两个操作在UNIX中是没有的，由于MFS本身是一个虚拟的文件系统，所以它是在内存上构建的对象，如果直接用read会产生很大的内存开销。MFS所有的操作都提供的js API接口，非常方便可视化管理这个虚拟文件系统。

mfs也是在内核的index.js文件中引，在新版本js-ipfs增加的下面几行代码中：

    const mfs = components.mfs(this, this._options)
    //将mfs的所有操作赋给files
    Object.keys(mfs).forEach(key => {
      this.files[key] = mfs[key]
    })

执行components.mfs()函数实例化这个插件：

    const mfs = require('ipfs-mfs/core')
    module.exports = self => {
      //此时mfsSelf等同于整个ipfs对象
      const mfsSelf = Object.assign({}, self)
    
      // A patched dag API to ensure preload doesn't happen for MFS operations
      // (MFS is preloaded periodically)
      // 修改原来self中的dag的API get和put
      mfsSelf.dag = Object.assign({}, self.dag, {
    get: promisify((cid, path, opts, cb) => {
      if (typeof path === 'function') {
    cb = path
    path = undefined
      }
    
      if (typeof opts === 'function') {
    cb = opts
    opts = {}
      }
    
      opts = Object.assign({}, opts, { preload: false })
    
      return self.dag.get(cid, path, opts, cb)
    }),
    put: promisify((node, opts, cb) => {
      if (typeof opts === 'function') {
    cb = opts
    opts = {}
      }
    
      opts = Object.assign({}, opts, { preload: false })
    
      return self.dag.put(node, opts, cb)
    })
      })
    
      return mfs(mfsSelf, mfsSelf._options)
    }

由于外面传入的参数是this，所以mfsSelf等同于整个ipfs对象。然后修改原来ipfs对象中的dag的API get和put。这么做是为了在调用mfs的各个API时不会触发preload，mfs中涉及到写的API都会更改了原来文件和各级目录的CID，但是因为mfs的写操作会频繁发生，如果每操作一次就preload，会极大地增加ipfs.io网关下的节点的负担，所以mfs采用mfs-preload的方式每间隔一段时间（30s）再preload，而不像普通的add API上传完文件后马上preload，preload的意义也就是让ipfs.io网关下的节点更新文件。继续看最后一行的mfs()函数，这里的mfs = require('ipfs-mfs/core')，这是一个单独的由npm管理的ipfs-mfs模块。

    const {
      createLock
    } = require('./utils')
    
    // These operations are read-locked at the function level and will execute simultaneously
    // 读操作
    const readOperations = {
      ls: require('./ls'),
      stat: require('./stat')
    }
    
    // These operations are locked at the function level and will execute in series
    // 写操作
    const writeOperations = {
      cp: require('./cp'),
      flush: require('./flush'),
      mkdir: require('./mkdir'),
      mv: require('./mv'),
      rm: require('./rm')
    }
    // 上面的读写操作由一个锁来管理，下面的操作也涉及到读写，但是锁各自管理
    // These operations are asynchronous and manage their own locking
    const unwrappedOperations = {
      write: require('./write'),
      read: require('./read')
    }
    
    // These operations are synchronous and manage their own locking
    const unwrappedSynchronousOperations = {
      readPullStream: require('./read-pull-stream'),
      readReadableStream: require('./read-readable-stream')
    }
    
    const wrap = ({
      ipfs, mfs, operations, lock
    }) => {
      Object.keys(operations).forEach(key => {
    mfs[key] = promisify(lock(operations[key](ipfs)))
      })
    }
    
    const defaultOptions = {
      repoOwner: true
    }
    
    module.exports = (ipfs, options) => {
      const {
    repoOwner   //在ipfs的config中默认也是true
      } = Object.assign({}, defaultOptions || {}, options)
    
      const lock = createLock(repoOwner)
      //读锁操作
      const readLock = (operation) => {
    return lock.readLock(operation)
      }
      //写锁操作
      const writeLock = (operation) => {
    return lock.writeLock(operation)
      }
    
      const mfs = {}
      //将所有操作装到mfs对象统一管理
      wrap({
    ipfs, mfs, operations: readOperations, lock: readLock
      })
      wrap({
    ipfs, mfs, operations: writeOperations, lock: writeLock
      })
    
      Object.keys(unwrappedOperations).forEach(key => {
    mfs[key] = promisify(unwrappedOperations[key](ipfs))
      })
    
      Object.keys(unwrappedSynchronousOperations).forEach(key => {
    mfs[key] = unwrappedSynchronousOperations[key](ipfs)
      })
    
      return mfs
    }

首先是将所有在各个子文件定义的操作引入，然后createLock(repoOwner)函数创建一个读写锁，这个读写锁针对上面开头的readOperations和writeOperations中的七个操作，也就是这七个操作（ls，stat，cp，flush，mkdir，mv，rm）用同一把锁控制。而unwrappedOperations中的write和read操作用的是宿主系统本身的锁。最后将这些操作添加到mfs对象。lock.readLock()函数和lock.writeLock()函数分别控制那七个操作的读写锁。这两个函数如下：
	
    //repoOwner默认为true
	module.exports = (repoOwner) => {
	//如果锁已经存在则直接返回
      if (lock) {
    return lock
      }
    //使用mortice模块创建锁
      const mutex = mortice({
    // ordinarily the main thread would store the read/write lock but
    // if we are the thread that owns the repo, we can store the lock
    // on this process even if we are a worker thread
    // 拥有仓库repo的线程可以存储读写锁，而不管是不是主线程
    singleProcess: repoOwner
      })
    
      const performOperation = (type, func, args, callback) => {
    log(`Queuing ${type} operation`)
    
    mutex[`${type}Lock`](() => {
      return new Promise((resolve, reject) => {
    //将下面的函数添加到args参数数组，接收func执行完后的结果
    args.push((error, result) => {
      log(`${type.substring(0, 1).toUpperCase()}${type.substring(1)} operation callback invoked${error ? ' with error: ' + error.message : ''}`)
    
      if (error) {
    return reject(error)
      }
    
      resolve(result)
    })
    log(`Starting ${type} operation`)
    //执行外部传入的操作func，而且将args参数数组作为参数传给func
    func.apply(null, args)
      })
    })
      .then((result) => {
    log(`Finished ${type} operation`)
    const cb = callback
    callback = null
    cb(null, result)
      })
      .catch((error) => {
    log(`Finished ${type} operation with error: ${error.message}`)
    if (callback) {
      return callback(error)
    }
    
    log(`Callback already invoked for ${type} operation`)
    
    throw error
      })
      }
    
      lock = {
    //func一般是多参数的第一个参数
    readLock: (func) => {
      return function () {
    //将最后一个参数弹出args数组并赋给callback
    const args = Array.from(arguments)
    let callback = args.pop()
    //执行读操作
    performOperation('read', func, args, callback)
      }
    },
    
    writeLock: (func) => {
      return function () {
    const args = Array.from(arguments)
    let callback = args.pop()
    //执行写操作
    performOperation('write', func, args, callback)
      }
    }
      }
    
      return lock
    }
在上面代码中首先用比较常用的mortice模块包来创建锁，由于传入的参数只有singleProcess字段且值为true，所以这个锁的特点是允许任意数量的读线程同时访问，只允许一个写线程访问，而且读写相斥。singleProcess字段指定是否只有主线程才能拥有锁，主线程就是ipfs的守护线程，而调用mfs的API可以另起一个线程来服务，所以这里singleProcess字段为true则让子线程也能拥有锁的控制权。读写的实际区域就是之前提到的文件块仓库。后面大部分代码都是用args数组+apply的方式处理lock.readLock()函数和lock.writeLock()函数传入的动态参数，将由该所控制的操作（前面那七个mfs的API）分别添加到锁的读写队列里去。

以上就是mfs的大致模样，下次再仔细看看mfs的一些API，深入了解。有了这些，mfs-preload就好理解了。

    module.exports = (self, options) => {
      options = options || {}
      options.interval = options.interval || 30 * 1000
    
      let rootCid
      let timeoutId
    
      const preloadMfs = () => {
    //获得MFS虚拟文件系统的根信息
    self.files.stat('/', (err, stats) => {
      if (err) {
    timeoutId = setTimeout(preloadMfs, options.interval)
    return log.error('failed to stat MFS root for preload', err)
      }
      //如果这里保存的rootCid和MFS的根hash不一致，则更新
      if (rootCid !== stats.hash) {
    log(`preloading updated MFS root ${rootCid} -> ${stats.hash}`)
    //调用preload通知ipfs.io网关的节点更新
    return self._preload(stats.hash, (err) => {
      //设置定时更新，间隔30s
      timeoutId = setTimeout(preloadMfs, options.interval)
      if (err) return log.error(`failed to preload MFS root ${stats.hash}`, err)
      //记录最新的MFS的根hash
      rootCid = stats.hash
    })
      }
      //如果一直也要设置定时更新
      timeoutId = setTimeout(preloadMfs, options.interval)
    })
      }
    
      return {
    start (cb) {
      self.files.stat('/', (err, stats) => {
    if (err) return cb(err)
    rootCid = stats.hash
    log(`monitoring MFS root ${rootCid}`)
    //设置定时更新
    timeoutId = setTimeout(preloadMfs, options.interval)
    cb()
      })
    },
    stop (cb) {
      clearTimeout(timeoutId)
      cb()
    }
      }
    }

前面提到在ipfs内核index.js文件的末尾会把mfs的操作复制到files对象中，所以在上面preloadMfs()函数中，self.files.stat()实际上就是mfs.stat。和UNIX中相似，这个api能够获得文件或目录的状态。在上面指定获取mfs文件系统的根目录状态，然后比较rootCid变量所记录的值和根目录的CID（或者说hash）是否一致，不一致则马上更新，随后设置定时更新，间隔30s。当本节点调用了mfs的写操作API后肯定会导致每一级目录包括根目录的hash改变，而设置定时更新的理由前面也说了。以上就是mfs-preload，为了配合mfs的preload。上面代码的末尾是start和stop方法，ipfs初始化时会调用每个component的start方法，mfs-preload就开始定时执行了。





