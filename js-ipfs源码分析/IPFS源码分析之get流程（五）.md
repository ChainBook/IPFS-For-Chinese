# IPFS源码分析之get流程（五） #
上次看了怎么连接上节点，这次开始使用这条连接向节点发送消息，表明本节点需要哪些CID文件块。先回到《IPFS源码分析之get流程（二）》的Fig 7，为了方便看，这里再把这张图贴出来：
![](https://i.imgur.com/ViS7Lmu.png)
 
											Fig 1 ipfs-bitswap/src/index.js

之前讲到，在本地节点没有指定的CID文件块情况下，只能向其他节点找，所前面的两篇都是从第303行this.network.findAndConnect()函数开始引出如何找到有想get的CID文件块的节点并连接。此外，在上图中还做了两件事，分别是第266行的notifications.wantBlock()函数和第282行的wm.wantBlocks()，前者调用Bitswap的notifications子模块将要获取的文件块CID注册到监听器中，后者调用Bitswap的WantManager子模块将要获取的文件块CID转化成消息发送给其他节点。先看前者：

![](https://i.imgur.com/ClgyZtq.png)
 
								Fig 2 ipfs-bitswap/src/notifications.js

Notifications类继承自EventEmitter，所以可以直接设置监听unwant:cidStr和block:cidStr，onBlock()就是Fig 1的269行的函数，onUnwant()就是Fig 1的276行的函数。由于WantManager子模块中也有一个wantlist，不管收到的CID文件块是不是想要的，都要删除WantManager子模块的wantlist的内容，当然返回值不一样，因为只要wantlist有内容，就会一直向其他节点发包获取文件块，可以在《IPFS源码分析之Bitswap层（二）》的Fig 7，也就是WantManager子模块的启动代码看到。现在接着看wm.wantBlocks()，即WantManager子模块怎么发送想要某个CID文件块这样的消息，这个函数只有一句代码：this._addEntries(cids, false)，就继续看这个函数：
![](https://i.imgur.com/xlpuHsV.png)
 
									Fig 3 ipfs-bitswap/src/want-manager/index.js

先将每个CID包装成BitswapMessageEntry类实例，再添加个默认优先级。这个类本身没有什么多余的东西，但是后面的消息都是根据这个类的内容来转化，而不是直接的CID。对于获取文件块，33行的判断e.cancel都是False，所以直接到42行wantlist.add()函数，这就是WantManager子模块中的wantlist，其中wantlist = new Wantlist(stats)，可以看下这个类的构造和wantlist.add()函数：
![](https://i.imgur.com/orrgPRt.png)
 
						Fig 4 ipfs-bitswap/src/types/wantlist/index.js

这里同样将CID包装成BitswapMessageEntry类实例后，添加到wantlist的一个map结构的属性set，然后调用this._stats.push()函数更新wantListSize计数器的值，_stats是在上图构造函数中传递进来的Bitswap的stats子模块，还是在《IPFS源码分析之Bitswap层（二）》的Fig 3和Fig 4讲到了这个子模块的构造，它的功能是通过控制几个计数器的EMA衰减达到控制整体各种消息的收发频率，具体由stat类来管理各个计数器，其中Fig 4可以看到初始有哪些计数器，可以看看push()函数：

![](https://i.imgur.com/jFTkrfP.png)
 
								Fig 5 ipfs-bitswap/src/stats/index.js

95行的this._global是一个stat类实例，在初始构造stats子模块中，所以是个全局的代表本节点的计数器管理器，104行的peerStats则是关于某个节点的计数器管理器，从上图的代码可以看出，不管调用这个push()函数有没有传入peer参数，都会更新全局的计数器，而如果具体传入了那个peer参数，则还要更新具体的某个节点的计数器管理器，这直接关系到Bitswap的发送消息包括请求区块和回应区块的策略，因为不同的节点各种计数器的值不一样，看看95行stat类的push()方法更新计数器的流程：
![](https://i.imgur.com/ciPsTZo.png)
 
										Fig 6 ipfs-bitswap/src/stats/stat.js

将[计数器，输入数据，当前时间]存入this._queue，一个初始化为空的列表。然后调用_nextTimeout()函数根据当前列表长度计算一下触发执行this._update()函数的延迟时间，如果这段时间内push没有再次被调用，就开始执行this._update()函数：
![](https://i.imgur.com/TFHf7l6.png)
 
											Fig 7 ipfs-bitswap/src/stats/stat.js

从上到下是函数的逐步调用流程，107行的_updateFrequencyFor()函数主要是计算出EMA衰减的一些参数，每个计数器key都对应一个MovingAverage类来做EMA衰减，关于EMA衰减在《IPFS源码分析之Bitswap层（二）》有讲，由121行的movingAverage.push()函数实质地计算衰减后的值，每个计数器的当前值都记录在this._stats这个列表中。最后_update()函数触发‘update’事件，并将this._stats返回给上层。前面讲到this._global这个全局的stat类实例，代表了stats子模块，就是它监听着‘update’事件，所以不管是更新全局的计数器的值还是具体关于某个节点的计数器的值，都会触发‘update’事件，从而由stats这个子模块再通知Bitswap的其他子模块做处理。

回到Fig 3，在添加要获取的CID文件块到wantlist和更新了一下那些计数器后，接着到第48行，p.addEntries()函数向this.peers中所有的节点发送获取CID文件块的消息。可能会搞混，Libp2p层的模块也有this.peers属性，但和WantManager模块的this.peers是无关的，初始化WantManager模块时这里的this.peers是空的，那节点信息就需要通过某中方式添加进来。上一篇看了怎么建立普通连接，而复用连接没有展开，在上一篇的Fig 3的38行gotWarmedUpConn()函数对建立好的普通连接都尝试升级为复用连接，在同个图中也看到gotWarmedUpConn()函数马上调用了attemptMuxerUpgrade()函数：
![](https://i.imgur.com/0sCwMgH.png)
 
											Fig 8 libp2p-switch/src/dial.js

这就是建立复用连接的过程，具体不多说，在199行，建立了复用连接后Libp2p层的switch子模块就触发‘peer-mux-established’事件，这个事件的监听者正是上篇讲到的给连接添加监控的observer，在上篇的Fig 9的13行，‘peer-mux-established’事件的处理函数直接又触发‘peer:connected’事件，这个事件的监听者是Bitswap的network子模块，在《IPFS源码分析之Bitswap层（二）》的Fig 8中，即network开始运行。‘peer:connected’事件的处理函数_onPeerConnect()直接调用bitswap._onPeerConnected(peerInfo.id)函数，后者又直接调用wm.connected(peerId)函数，后者再直接调用this._startPeerHandler(peerId)，这个函数就在上面的Fig 3。其中第60行为新连接上的节点创建一个消息队列类MsgQueue实例，以后往这个节点的发的消息都装在队列。然后实例化一个消息Message类实例fullwantlist，再把当前WantManager模块的wantlist的内容即CID转化成消息实例中的一个个条目，最后调用mq.addMessage(fullwantlist)函数将消息添加到消息队列发送给这个新连接上的节点。

按流程来说，回到Fig 3的48行，p.addEntries()函数向this.peers中所有的节点发送获取CID文件块的消息，其中p也是某个节点的消息队列。那mq.addMessage(fullwantlist)函数和p.addEntries()函数有何区别：
![](https://i.imgur.com/zqoYZwg.png)
 
									Fig 9 ipfs-bitswap/src/want-manager/msg-queue.js

从图中24行可以看到addMessage()函数直接调用send()函数将消息发送，而addEntries()函数先是调用了_sendEntries()函数，同样实例化一个消息Message类实例msg，再把当前WantManager模块的wantlist的内容即CID转化成消息实例中的一个个条目，最后居然调用了addMessage()函数。看起来后者比前者饶了两步，但有个重要的差异，前者在实例化Message类时参数是true，而后者实例化Message类时参数是false，所以即使消息实例中CID的条目都一样，但仍然是不同的消息，这会导致消息接收方做出不同回应决策。而且前者被调用的背景是当与某个节点建立复用连接时，将WantManager模块的wantlist的已内容发送给该节点。后者被调用的背景是将要获取的CID文件块转换成消息发送给WantManager模块的this.peers中所有的节点，即使wantlist已有内容，也只发调用函数添加的CID。

下次再看具体的发送消息，即上图56行的this.network.sendMessage()函数，和对方节点的回应。


**个人微信：xxb3059 & 公众号：区块丛林 欢迎交流**