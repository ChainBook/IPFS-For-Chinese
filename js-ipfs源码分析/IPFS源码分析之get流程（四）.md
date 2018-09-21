# IPFS源码分析之get流程（四） #
上一篇讲了怎么找到能够提供要查找的CID文件的20个节点，本来打算看具体的文件块收发，发现漏了一部分，就是找到这20个节点后怎么和这2些节点建立连接，在上一篇的Fig1中，findProviders ()函数找到20个节点后，接着用this.connectTo(p, cb)函数去连接这些节点，这个函数只有两行代码，第一行判断一下本节点有没有在线，然后调用libp2p.dial(peer, callback)函数去“拨通”这些节点。这里的libp2p就是之前看过的libp2p层。这个函数如下：

 ![](https://i.imgur.com/166SJZb.png)

									Fig 1 libp2p/src/index.js

从图中可以看到，这个函数没有多少逻辑，所有的连接操作都交给了this.switch.dial()函数，之前在《IPFS源码分析之Libp2p层》有简单提到这个switch子模块，这个模块是Libp2p层的基础，Libp2p层的所有子模块都用到它，可以看看它的部分构造代码：
![](https://i.imgur.com/q0IRKdr.png)
 
												Fig 2 libp2p-switch/src/index.js

这里重要的是this.transports，在《IPFS源码分析之Libp2p层》的Fig6中也有提到，它是两个节点之间发送信息所支持的网络协议，一般来说有{tcp，ws，wsstar}这三种，一个节点支持哪些transport协议，主要是通过该节点的peerinfo的multiaddrs知道，multiaddrs是IPFS比较有特色的东西，它包含了一个节点的所有地址格式：协议+IP。另外就是this.conns和this.muxedConns 分别保存着本节点与其他节点的普通连接和复用连接。由于在IPFS中两个节点之间一般建立的连接都是TCP连接，如果多次访问同个节点建立多个TCP长连接，会造成很大的资源浪费，因为每一个TCP连接都会用一个线程来管理，所以这里需要使用到多路复用连接。后面还会涉及。先回去看看switch.dial()函数怎么连接节点：
![](https://i.imgur.com/SxQCaTi.png)
 
									Fig 3 libp2p-switch/src/dialer.js

dial()函数里，实例化一个Connection类，所有底层建立的连接原始套接字都会被这个类包裹起来。然后查看上面讲到的conns和muxedConns对象有没有保存着普通连接和复用连接，如果都没有，则尝试发起连接；如果仅有普通连接，则尝试升级为复用连接，也就是图中的gotWarmedUpConn()函数；如果还保存着复用连接，则直接拿出来使用，也就是唤醒这个连接的看护进程。看看attemptDial()函数怎么发起连接：
![](https://i.imgur.com/zRQmZyw.png)
 
											Fig 4 libp2p-switch/src/dialer.js

函数内容很多，想要了解具体过程可以看代码注释。主要是选取双方都支持的transports协议来连接，其中Circuit协议并不是transports协议簇的一员，当没有共同的transports协议时才用这个。进入到具体的连接函数是115行的swtch.transport.dial(transport, pi, (err, _conn))，可以看到这个函数的第一个参数是具体一种transports协议，下面以TCP为例。连接成功则在callback函数中返回封装好的连接实例_conn，这个实例也是上面提到的Connection类的。看看swtch.transport.dial()函数怎么操作的：
![](https://i.imgur.com/t7yQnYd.png)
 
											Fig 5 libp2p-switch/src/transport.js

再次判断这个协议能不能用，具体怎么判断这里就省略。直接看到47行的dialer.dialMany()函数向对方的multiaddrs中所有格式的地址发起连接，只要有一个能成功连接就返回。其中dialer是一个LimitDialer类实例，LimitDialer = require('./limit-dialer')。接着看看dialMany()函数：
![](https://i.imgur.com/j1txgr1.png)
 
										Fig 6 libp2p-switch/src/limit-dailer/index.js

有dialMany()函数就一定能想到有dialSingle()函数，IPFS团队的编程风格可以感受到。对该节点每种格式的地址的连接就由dialSingle()函数来做，每一个要连接的地址都有一个寻呼任务队列DialQueue实例，长度限制为8，由于一个节点有多个格式的地址，每种格式的地址的连接任务都放在这个寻呼任务队列，目前IPFS的地址格式没有超过8种，所以这个长度是够的。每一次请求连接的寻呼时间限制是30秒，超过了30秒就放弃。87行的q.push()函数就是将一次任务加入队列中执行。DialQueue类其实就是包裹了'async/queue'模块的队列实例，可以看看其构造和寻呼任务的执行函数：

![](https://i.imgur.com/H5KK4vL.png)
 
									Fig 7 libp2p-switch/src/limit-dailer/queue.js

其中23行的queue = require('async/queue')。重点在由_doWork()函数调用_dialWithTimeout()函数，再调用transport.dial()函数来建立连接，这里的transport已经是具体的一个协议对象作为参数，比如TCP，看看TCP的dial()函数：
![](https://i.imgur.com/TManQpZ.png)
 
									Fig 8 libp2p-tcp/src/index.js

这里就是最后建立TCP原始套接字，用到了比较常用的net模块。以及将套接字包裹成Connection类实例。其他的transports协议类似，不再赘述。
现在回到Fig 4，swtch.transport.dial()函数已经成功地返回了某个transports协议建立的连接，接着调用observeConnection()函数，给返回连接添加信息监控通知，一个连接本质上也是一个流通道，只要这个通道上出现了数据，switch.observer就会触发’message’事件以通知上层：
![](https://i.imgur.com/28fc2Zu.png)
 
											Fig 9 libp2p-switch/src/observer.js

上图的代码有兴趣可以看，switch.observer都监看着哪些。添加了信息监控通知后，Fig 4的122行接着执行cryptoDial()函数，开始尝试建立加密连接。内部再调用ms.handle()函数和对方握手，比较两方的加密版本是否一致，当前IPFS默认是'/multistream/1.0.0'。看看ms.handle()函数：
![](https://i.imgur.com/EG1Vcj5.png)
 
										Fig 10 multistream-select/src/index.js

handle()函数里37行调用的select函数并不是上图中的select函数，而是下面的：
![](https://i.imgur.com/vW2kIqc.png)
 
									Fig 11 multistream-select/src/select.js

这个select函数才是和对面发起握手和协商共同的加密版本的操作者。需要注意的是，从handle()函数的参数可以看到这次协商共同的加密版本的操作使用的是前面添加了信息监控通知的连接conn，因为这里的握手信息需要通知上层处理。协商完共同的加密版本后，走到Fig 4的133行调用ms.select()函数，这个select则是Fig 10中的select函数，这个select函数的目的则是比较双方是否有共同的加密算法，目前一般都是用SECIO，这在《IPFS源码分析之Libp2p层》有讲到。从ms.select()函数的参数可知，这次select函数的操作使用的是没有添加信息监控通知的连接_conn，所以Fig 4的136行再次调用observeConnection()函数给这个确定了共同的加密算法的连接添加信息监控通知，再接着调用swtch.crypto.encrypt()函数真正加密这条连接。Fig 3的attemptDial()函数的callback接收到的就是这个加密连接，到此普通连接建立完毕，至于复用连接，其流程类似。
有了普通连接和复用连接，下次就可以看看想要的CID文件块怎么送过来。

**个人微信：xxb3059 & 公众号：区块丛林 欢迎交流**