# Libp2p层 #
写在前面：水平有限，欢迎指正。

本文的ipfs模块是npm install ipfs后node_modules里ipfs文件夹的内容。Libp2p的引入流程如下，具体的函数执行在代码图片中有注释：

先在IPFS模块的index.js中：this.libp2p = components.libp2p(this)

传到component/start.js中self=this：self.libp2p.start(cb)

再到component/libp2p.js的start函数中，根据libp2pOptions设置建立一个p2p节点类：self._libp2pNode = new Node(libp2pOptions)，其中Node = require('../runtime/libp2p-nodejs')
类的继承关系如下：
在runtime/libp2p-nodejs'中定义的类：class Node extends libp2p{ constructor ( ) {……}; super( ) }，其中libp2p = require('libp2p')
在libp2p模块的index.js中定义的类：class Node extends EventEmitter{ constructor ( ) {……}; super() }, 其中EventEmitter = require('events').EventEmitter

 ![](https://i.imgur.com/bu1nVCY.png)

                       Fig 1 js-ipfs\src\core\components\libp2p.js
设置监听self._libp2pNode.on('peer:discovery', (peerInfo))；self._libp2pNode.on('peer:connect', (peerInfo)); self._libp2pNode.once('start', dial)

启动节点：self._libp2pNode.start（），执行的是最后继承的libp2p模块的index.js中定义的类start（）方法
所以先插入说明这个类的构造函数：constructor (_modules, _peerInfo, _peerBook, _options) {……}

首先引入this.switch = new Switch(this.peerInfo, this.peerBook, _options.switch)，libp2p-switch是一种拨号机，它利用多个libp2p传输、流muxers、加密通道和其他连接升级来拨号到libp2p网络中的对等点。它还支持通过多编解码器和多流选择握手的协议多路复用。在libp2p中作用非常大。
然后根据传入的_modules里的四个模块分别进行实例化各个模块里的插件，_modules默认的配置在js-ipfs\src\core\runtime\libp2p-node.js中

 ![](https://i.imgur.com/7hzcTBK.png)

									Fig 2 muxer
 ![](https://i.imgur.com/6ZNqx4V.png)

									Fig 3 crypto

![](https://i.imgur.com/zKKAz9P.png)

								Fig 4 discovery
 ![](https://i.imgur.com/VTiyAD5.png)

										Fig 5 DHT
最后Ping.mount(this.switch)，节点开启ping协议，能够回应其他节点发来的ping消息

现在回到这个类的start（）方法
先是根据已有的multiaddrs(包含一些IPFS可识别的地址格式，如：/ip4/8.8.8.8/tcp/1080/ipfs/)中获知_modules里的另一个模块transports，用来直接传文件数据的协议

 ![](https://i.imgur.com/ZqN2VEn.png)

										Fig 6 start()
然后顺序地将switch，_modules里各个模块和推送订阅协议实例开启，即执行各自的start()方法。并且将self._libp2pNode设置为开启的状态。
 ![](https://i.imgur.com/lv6ExyO.png)

											Fig 7 start()
最后发送的‘start’事件，将被Fig 1里的监听self._libp2pNode.once('start', dial)获悉，执行dial()。在start()执行之前，前面if (self.isOnline())得到的_libp2pNode的状态是False。

最后再看一下这个dial()做了什么。这个dial()是包含在Fig 1里的监听self._libp2pNode.on('peer: discovery', (peerInfo))中，而触发这个监听的正是在Fig 4里discovery.on('peer', (peerInfo) => this.emit('peer:discovery', peerInfo))，_modules的discovery模块的。
这个模块的作用是在IPFS网络中寻找对等节点，有三种方式MulticastDNS(以广播的形式)，BootStrap(本地的config提供BootStrap节点列表)，wsstar.discovery。每种方式的实例都监听着一个peer事件。
后面的Fig 7start()中启动discovery模块的的3个实例
return each(this.modules.discovery, (d, cb) => d.start(cb), cb)，即分别调用这3个实例的 start()方法。以MulticastDNS的start()为例

 ![](https://i.imgur.com/y59Y0ES.png)

								Fig 8 MulticastDNS的start()
MulticastDNS以广播的形式给周围普通的地址发送报文，若该地址是IPFS中的一个节点则能识别该报文并返回节点信息。得到回应后触发了对‘response’的监听，再触发外层对‘peer’的监听，最后触发Fig 1里对'peer: discovery'的监听。
以上就是Libp2p的大体框架，具体的模块功能以后会继续分析。


Libp2p.switch模块中定义的类：class Switch extends EE { constructor ( ) {……}; EE= EventEmitter

Libp2p.ping 模块中定义的类：class Ping extends EventEmitter { constructor ( ) {……};

**个人微信：xxb3059 & 公众号：区块丛林  欢迎交流**
