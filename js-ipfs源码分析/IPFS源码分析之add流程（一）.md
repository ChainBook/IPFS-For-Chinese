# IPFS源码分析之add流程（一） #
> 在前面用了很多篇来分析get取文件的流程，基本上涉及了IPFS的所有模块之间的联系调用，出于研究的需要，把add添加文件过程也看了一下，以防遗忘，便简单记录。很多模块调用的细节可以参考之前的。

IPFS添加文件有三个API，分别是add，addReadableStream和addPullStream。比较常用的是前两个，最后那个需要额外用到Pull Stream这个模块，虽然在IPFS的源码中大量使用了这种流，输入输出也确实比较方便，但对于简单调用它的js接口来实现功能来说，还是用前两个API比较方便。如果再做比较，第一个比第二个使用起来更简单，从官方给出的使用示例（https://github.com/ipfs/interface-ipfs-core/blob/master/SPEC/FILES.md#filesadd）也可看出，看看它们的源码：

**add:**

    module.exports = (send) => {
      const createAddStream = SendFilesStream(send, 'add')
    
      const add = promisify((_files, options, _callback) => {
    if (typeof options === 'function') {
      _callback = options
      options = null
    }
    //外部回调函数
    const callback = once(_callback)
    
    if (!options) {
      options = {}
    }
    options.converter = FileResultStreamConverter
    //检查第一个参数的格式
    const ok = Buffer.isBuffer(_files) ||
       isStream.readable(_files) ||
       Array.isArray(_files) ||
       OtherBuffer.isBuffer(_files) ||
       typeof _files === 'object' ||
       isSource(_files)
    
    if (!ok) {
      return callback(new Error('first arg must be a buffer, readable stream, pull stream, an object or array of objects'))
    }
    
    const files = [].concat(_files)
    //根据传入的选项创建stream流
    const stream = createAddStream({ qs: options })
    const concat = ConcatStream((result) => callback(null, result))
    stream.once('error', callback)
    stream.pipe(concat)
    //将files对象中包含的文件内容逐个传入stream流
    files.forEach((file) => stream.write(file))
    stream.end()
      })

**addReadableStream:**

    const SendFilesStream = require('../utils/send-files-stream')
    const FileResultStreamConverter = require('../utils/file-result-stream-converter')
    module.exports = (send) => {
      return (options) => {
    options = options || {}
    options.converter = FileResultStreamConverter
    return SendFilesStream(send, 'add')(options)
      }
    }
从上面两段代码可以看到，两者都是创建了SendFilesStream来接收文件内容，注意，这里的文件内容指的是一个结构对象{
path: '/tmp/myfile.txt',content: (Buffer or Readable stream)}。不同的是，如果使用addReadableStream来添加文件，需要自行编写流的监听和写入代码，也就是类似add的源码的后8行，如果是使用add来添加文件，则直接传入上面提到的结构对象作为参数即可，当然还可传入一些options和回调函数。后面仅以使用add方式展开。先看看有哪些可用的options：


- cid-version (integer, default 0): 指定cid版本，默认为0。


- progress (function)：为返回内容添加字节数字段的函数，在服务器端有默认的处理函数。


- recursive (boolean):配合第一个参数使用，如果第一个参数包含一个路径，则这里决定是否递归上传这个路径所有的文件。


- hashAlg || hash (string):指定使用的哈希函数。


- wrapWithDirectory (boolean):是否为添加的内容指定目录。


- onlyHash (boolean):不实际地上传文件到IFS网络，仅仅计算一下这个文件的HASH。


- pin (boolean, default true):锁定文件，如果要从IPFS网络删除该文件必须先解锁。


- raw-leaves (boolean, default false): 如果为真，这个文件的DAG树的叶子节点将会保存原始文件内容。


- chunker (string, default size-262144): 指定分割文件的方式，默认是250K左右的大小，其他指定方式有：
size-{size}；
rabin；
rabin-{avg}；
rabin-{min}-{avg}-{max}

以上这些选项在后面的源码的中会涉及具体的判断。先继续看SendFilesStream：

    module.exports = (send, path) => {
      return (options) => {
    let request
    let ended = false
    let writing = false
    //如果有options，格式一般为{qs：options}整体
    options = options ? Object.assign({}, options, options.qs) : {}
    //实例化一个Multipart
    const multipart = new Multipart()
    
    const retStream = new Duplex({ objectMode: true })
    
    retStream._read = (n) => {}
    
    retStream._write = (file, enc, _next) => {
      const next = once(_next)
      try {
    //预处理file参数
    const files = prepareFile(file, options)
      //根据上一步的处理加入文件类型字段headers
      .map((file) => Object.assign({headers: headers(file)}, file))
    
    writing = true
    eachSeries(
      //将参数file中每个文件的{header，path，content}传入multipart
      files,
      (file, cb) => multipart.write(file, enc, cb),
      (err) => {
    writing = false
    if (err) {
      return next(err)
    }
    if (ended) {
      multipart.end()
    }
    next()
      })
      } catch (err) {
    next(err)
      }
    }
    //当add调用全部把文件上传完
    retStream.once('finish', () => {
      if (!ended) {
    ended = true
    if (!writing) {
      multipart.end()
    }
      }
    })
    
    const qs = options.qs || {}
    
    qs['cid-version'] = propOrProp(options, 'cid-version', 'cidVersion')
    qs['raw-leaves'] = propOrProp(options, 'raw-leaves', 'rawLeaves')
    qs['only-hash'] = propOrProp(options, 'only-hash', 'onlyHash')
    qs['wrap-with-directory'] = propOrProp(options, 'wrap-with-directory', 'wrapWithDirectory')
    qs.hash = propOrProp(options, 'hash', 'hashAlg')
    //构造请求中的参数，将原来请求options中一部分选项如recursive和progress分离出来，其他的(上面5行)保留在qs中
    const args = {
      path: path,
      qs: qs,
      args: options.args,
      multipart: true,
      multipartBoundary: multipart._boundary,
      stream: true,
	  //在这里recursive选项被默认为真
      recursive: true,
      progress: options.progress
    }
    
    multipart.on('error', (err) => {
      retStream.emit('error', err)
    })
    //构造请求
    request = send(args, (err, response) => {
      if (err) {
    return retStream.emit('error', err)
      }
    //处理返回数据
      if (!response) {
    // no response, which means everything is ok, so we end the retStream
    return retStream.push(null) // early
      }
    
      if (!isStream(response)) {
    retStream.push(response)
    retStream.push(null)
    return
      }
    
      response.on('error', (err) => retStream.emit('error', err))
    
      if (options.converter) {
    response.on('data', (d) => {
      if (d.Bytes && options.progress) {
    options.progress(d.Bytes)
      }
    })
    
    const Converter = options.converter
    const convertedResponse = new Converter()
    convertedResponse.once('end', () => retStream.push(null))
    convertedResponse.on('data', (d) => retStream.push(d))
    response.pipe(convertedResponse)
      } else {
    response.on('data', (d) => {
      if (d.Bytes && options.progress) {
    options.progress(d.Bytes)
      }
      retStream.push(d)
    })
    response.once('end', () => retStream.push(null))
      }
    })
    
    // signal the multipart that the underlying stream has drained and that
    // it can continue producing data..
    // 当发送完数据后触发request的drain事件，再触发multipart的drain事件，这样multipart可以继续接受数据
    request.on('drain', () => multipart.emit('drain'))
    //及时将multipart中的数据转移到request并发送
    multipart.pipe(request)
    
    return retStream
      }
    }

在上面的代码中实现了SendFilesStream即retStream的写方法retStream._write，retStream也只是个包装，里面实际用到的流是multipart，所以在写方法中将每个文件的{header，path，content}结构传到multipart，文件的内容在content中。这里的multipart是已经封装好的类实例，一般用nodejs写multipartFormdataType请求上传数据需要自己设定边界符，在这里直接使用封装的multipart._boundary。代码的倒数第二行multipart.pipe(request)，将multipart和request连接，只要multipart有数据，马上就流入到request，作为http请求的payload。request是由send()构造的请求，send()详看《IPFS源码分析之API结构》。send()的回调函数如何处理服务器返回结果有兴趣自己看。以上是请求端，每个文件一个请求，发送的数据主要是args参数的内容和文件的{header，path，content}结构。

下面看服务器端怎么处理。转接过程和之前get流程一样，直接进入add请求的处理函数。

    exports.add = {
      //请求数据格式检查
      validate: {
    query: Joi.object()
      .keys({
    'cid-version': Joi.number().integer().min(0).max(1).default(0),
    'raw-leaves': Joi.boolean(),
    'only-hash': Joi.boolean(),
    pin: Joi.boolean().default(true),
    'wrap-with-directory': Joi.boolean(),
    chunker: Joi.string()
      })
      // TODO: Necessary until validate "recursive", "stream-channels" etc.
      .options({ allowUnknown: true })
      },
    //此处的request即是前面所述的request请求
      handler: (request, reply) => {
    if (!request.payload) {
      return reply({
    Message: 'Array, Buffer, or String is required.',
    Code: 0,
    Type: 'error'
      }).code(400).takeover()
    }
    //ipfscore实例
    const ipfs = request.server.app.ipfs
    // TODO: make pull-multipart
    // 由parser解析上传的multipart data
    const parser = multipart.reqParser(request.payload)
    let filesParsed = false
    //建立一个pull流
    const fileAdder = pushable()
    //如果是个文件
    parser.on('file', (fileName, fileStream) => {
      //文件路径解码，因为文件路径存在特殊字符，不能直接传
      fileName = decodeURIComponent(fileName)
      const filePair = {
    path: fileName,
    content: toPull(fileStream)
      }
      filesParsed = true
      fileAdder.push(filePair)
    })
    //如果是个目录
    parser.on('directory', (directory) => {
      directory = decodeURIComponent(directory)
    
      fileAdder.push({
    path: directory,
    content: ''
      })
    })
    //所有上传的数据都解析完了
    parser.on('end', () => {
      if (!filesParsed) {
    return reply({
      Message: "File argument 'data' is required.",
      Code: 0,
      Type: 'error'
    }).code(400).takeover()
      }
      //结束pull流
      fileAdder.end()
    })
    //有一个pull流，装回应请求的内容
    const replyStream = pushable()
    const progressHandler = (bytes) => {
      replyStream.push({ Bytes: bytes })
    }
    //根据请求中带的options整理在一起
    const options = {
      cidVersion: request.query['cid-version'],
      rawLeaves: request.query['raw-leaves'],
      progress: request.query.progress ? progressHandler : null,
      onlyHash: request.query['only-hash'],
      hashAlg: request.query['hash'],
      wrapWithDirectory: request.query['wrap-with-directory'],
      pin: request.query.pin,
      chunker: request.query.chunker
    }
    
    const aborter = abortable()
    const stream = toStream.source(pull(
      replyStream,
      aborter,
      ndjson.serialize()
    ))
    
    // const stream = toStream.source(replyStream.source)
    // hapi is not very clever and throws if no
    // - _read method
    // - _readableState object
    // are there :(
    if (!stream._read) {
      stream._read = () => {}
      stream._readableState = {}
      stream.unpipe = () => {}
    }
    reply(stream)
      .header('x-chunked-output', '1')
      .header('content-type', 'application/json')
      .header('Trailer', 'X-Stream-Error')
    
    function _writeErr (msg, code) {
      const err = JSON.stringify({ Message: msg, Code: code })
      request.raw.res.addTrailers({
    'X-Stream-Error': err
      })
      return aborter.abort()
    }
    
    pull(
      fileAdder,
      //把上传的文件添加到IPFS网络
      ipfs.files.addPullStream(options),
      pull.map((file) => {
    //分别返回每个文件的路径，hash和size
    return {
      Name: file.path, // addPullStream already turned this into a hash if it wanted to
      Hash: file.hash,
      Size: file.size
    }
      }),
      pull.collect((err, files) => {
    if (err) {
      return _writeErr(err, 0)
    }
    
    if (files.length === 0 && filesParsed) {
      return _writeErr('Failed to add files.', 0)
    }
    //给上传者返回以上内容
    files.forEach((f) => replyStream.push(f))
    replyStream.end()
      })
    )
      }
    }

每当一个请求过来，先由parser对request.payload即所上传文件的{header，path，content}结构做解析，主要是根据header段的内容对上传的文件做分类，分别为目录，符号链接，文件，然后触发parser监听的不同文件类型事件（注意：没有符号链接），将不同的文件类型数据以{path: ,content: }添加到fileAdder流中。header段的内容是在请求端SendFilesStream中根据调用add API时指定的文件的路径判断那种文件类型添加进去的。fileAdder流是一个pushable类实例，而pushable = require('pull-pushable')，说到底还是用pull家族的流。

下面贴出parser的代码，已做详细注释，有兴趣可以看：

    function Parser (options) {
      // allow use without new
      if (!(this instanceof Parser)) {
    return new Parser(options)
      }
      //根据multipart的边界符实例化Dicer
      this.dicer = new Dicer({ boundary: options.boundary })
      //监听'part'事件，每当Parser接收完一个part触发，即两个边界符之间的数据，再传给handlePart函数处理
      this.dicer.on('part', (part) => this.handlePart(part))
    
      this.dicer.on('error', (err) => this.emit('err', err))
    
      this.dicer.on('finish', () => {
    this.emit('finish')
    this.emit('end')
      })
    
      Transform.call(this, options)
    }
    //对multipart data的解析类Parser同样继承自stream模块的Transform类
    util.inherits(Parser, Transform)
    
    Parser.prototype._transform = function (chunk, enc, cb) {
      this.dicer.write(chunk, enc)
      cb()
    }
    
    Parser.prototype._flush = function (cb) {
      this.dicer.end()
      cb()
    }
    //处理Parser接收到的每个part数据
    Parser.prototype.handlePart = function (part) {
      part.on('header', (header) => {
    //解析文件结构的header段
    const partHeader = parseHeader(header)
    //如果是目录
    if (isDirectory(partHeader.mime)) {
      part.on('data', () => false)
      this.emit('directory', partHeader.name)
      return
    }
    //如果是符号链接
    if (partHeader.mime === applicationSymlink) {
      part.on('data', (target) => this.emit('symlink', partHeader.name, target.toString()))
      return
    }
    //如果是文件，只有包含了具体内容的文件header中才有boundary
    if (partHeader.boundary) {
      // recursively parse nested multiparts
      // 很据boundary再把文件的真实内容解析出来
      const parser = new Parser({ boundary: partHeader.boundary })
      parser.on('file', (file) => this.emit('file', file))
      //递归解析
      part.pipe(parser)
      return
    }
    //单一文件，但是header中没有boundary
    this.emit('file', partHeader.name, part)
      })
    }

parser解析完后，所有要上传的文件包括目录都以{path: ,content: }放在fileAdder流中，如果是目录，content字段为空。在前面add的处理函数的后半段，接着在一个pull代码段中将fileAdder流传到ipfs.files.addPullStream(options)，后者将文件上传到IPFS网络，然后以一定格式返回每个文件或目录的相关消息如hash，详见第四段代码及注释。下面主要看ipfs.files.addPullStream(options)函数：

    function _addPullStream (options = {}) {
    let chunkerOptions
    try {
	//取chunker参数后面的数值
      chunkerOptions = parseChunkerString(options.chunker)
    } catch (err) {
      return pull.map(() => { throw err })
    }
	//设置文件切片的最低值，self._options.EXPERIMENTAL.sharding是系统设置，在构建IPFS对象时如果没有指定地传入这个设置参数，则为空
    const opts = Object.assign({}, {
      shardSplitThreshold: self._options.EXPERIMENTAL.sharding
    ? 1000
    : Infinity
    }, options, chunkerOptions)
    //如果用户指定了计算hash的算法，则要求cid版本一定要是1
    if (opts.hashAlg && opts.cidVersion !== 1) {
      opts.cidVersion = 1
    }
    
    let total = 0
    //如果add API的调用者不指定progress这个选项，则prog=noop，即什么都不做，仅仅统计总字节数
    const prog = opts.progress || noop
    const progress = (bytes) => {
      total += bytes
      prog(total)
    }
    //修改成新的总字节数处理函数
    opts.progress = progress
	//接收fileAdder流作为source
    return pull(
      //规范要添加的文件结构的content项内容，主要是区分有无path和content是一个buffer或readable stream
      pull.map(normalizeContent.bind(null, opts)),
      pull.flatten(),
      //添加到IPFS网络
      importer(self._ipld, opts),
      //预处理返回结果，以{path,hash,size}，如果原本上传的是一段文字，则path即是这个文字的hash
      pull.asyncMap(prepareFile.bind(null, self, opts)),
      pull.map(preloadFile.bind(null, self, opts)),
      //锁定文件，使得文件永久保留在本地仓库，除非解锁后删除
      pull.asyncMap(pinFile.bind(null, self, opts))
    )
      }

传进来的options参数就是调用add API时的options选项参数，从第三行开始就有了对这些选项的值的判断，如果API的调用者不指定这些选项，常常使用系统默认的，其中recursive选项被默认为真，progress选项指定的函数也被赋予了新的函数。随后又是在一个pull代码段中，先由pull.map()将fileAdder流中每个{path: ,content: }结构对象分别交给normalizeContent()函数处理，normalizeContent()函数对这些结构对象中的content项的内容规范化，下面是这个函数的源码：

    function normalizeContent (opts, content) {
      if (!Array.isArray(content)) {
    content = [content]
      }
    
      return content.map((data) => {
    // Buffer input
    if (Buffer.isBuffer(data)) {
      data = { path: '', content: pull.values([data]) }
    }
    
    // Readable stream input
    if (isStream.readable(data)) {
      data = { path: '', content: toPull.source(data) }
    }
    
    if (isSource(data)) {
      data = { path: '', content: data }
    }
    //由于传入的data一定是个{path: ,content: }结构对象，所以上面的三个判断属于多余，直接从这里开始
    if (data && data.content && typeof data.content !== 'function') {
      if (Buffer.isBuffer(data.content)) {
    data.content = pull.values([data.content])
      }
    
      if (isStream.readable(data.content)) {
    data.content = toPull.source(data.content)
      }
    }
    //如果有添加了wrapWithDirectory选项，必须要指定该文件的路径，否则报错
    if (opts.wrapWithDirectory && !data.path) {
      throw new Error('Must provide a path when wrapping with a directory')
    }
    //如果有添加了wrapWithDirectory选项，在data.path前加'wrapper/'
    if (opts.wrapWithDirectory) {
      data.path = WRAPPER + data.path
    }
    
    return data
      })
    }

处理流程如上代码段及注释，不管content项的内容（即文件的实质内容）是Buffer还是stream，都转化为pull stream。

回到ipfs.files.addPullStream(options)函数的代码，接着是一个pull.flatten()函数将前面normalizeContent()函数分别处理的{path: ,content: }结构对象汇总成一个流，再传给importer(self._ipld, opts)，之前看get流程时提到过这个函数，get API从IPFS的网络中获取文件的入口时exporter()函数，那importer()函数便是上传文件的入口。这里先不贴这个函数的源码，假设上传文件成功，对于每个文件它返回一个{path,hash,size}结构对象，如果原本上传仅仅是一段文字而不是某个文件，则path即是这段文字的hash。

接着在pull.asyncMap()的包裹下对每个返回的{path,hash,size}结构对象分别异步地调用prepareFile()函数：

    function prepareFile (self, opts, file, callback) {
      opts = opts || {}
    
      let cid = new CID(file.multihash)
    
      if (opts.cidVersion === 1) {
    cid = cid.toV1()
      }
    
      waterfall([
    //如果有onlyHash选项，则直接往下一步传返回的{path,hash,size}结构对象，如果没有则根据返回的file的hash从本地仓库取出整个file
    (cb) => opts.onlyHash
      ? cb(null, file)
      : self.object.get(file.multihash, opts, cb),
    (node, cb) => {
      const b58Hash = cid.toBaseEncodedString()
    
      let size = node.size
      //计算文件大小
      if (Buffer.isBuffer(node)) {
    size = node.length
      }
    
      cb(null, {
    //如果有添加了wrapWithDirectory选项，在data.path前加'wrapper/'
    path: opts.wrapWithDirectory ? file.path.substring(WRAPPER.length) : (file.path || b58Hash),
    hash: b58Hash,
    size
      })
    }
      ], callback)
    }

在prepareFile()函数中主要是涉及了cidVersion和onlyHash选项的判断，这两个的作用在前面已经给出。prepareFile()函数处理完后又交给preloadFile()函数处理，preload这个模块是在8月份IPFS更新之后新添加的，具体作用以后单独说，这里不影响整个add API的处理流程。最后又是在pull.asyncMap()的包裹下将前面返回的交给pinFile()函数处理：

    function pinFile (self, opts, file, cb) {
      // Pin a file if it is the root dir of a recursive add or the single file
      // of a direct add.
      // 一般pin选项都默认为true，除非指定为false
      const pin = 'pin' in opts ? opts.pin : true
      const isRootDir = !file.path.includes('/')
      const shouldPin = pin && isRootDir && !opts.onlyHash && !opts.hashAlg
      if (shouldPin) {
    //满足以上条件则加入pin管理
    return self.pin.add(file.hash, err => cb(err, file))
      } else {
    cb(null, file)
      }
    }

在pinFile()函数中主要涉及pin选项的判断，其作用在前面有说。关于pinning这个模块的使用可以参考http://ipfser.org/2018/09/17/pinningofipfs/这里的讲述。

以上是add API从请求端到服务器端的处理的一个大致流程，连接IPFS网络的关键接口importer()函数下次再详述。

