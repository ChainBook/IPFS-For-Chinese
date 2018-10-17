# IPFS源码分析之get流程（六） #
直接接上篇看this.network.sendMessage()函数怎么将想要某些文件块的消息发送给其他节点：
![](https://i.imgur.com/XOxyvcS.png)
 
										Fig 1 ifps-bitswap/src/network.js

参数msg是已经封装好的Message类实例，第144行调用this._dialPeer()函数和对方取得连接，这里可能会疑惑的是，之前的流程中已经用过dial()函数建立过连接了，但这里又取得连接，看看this._dialPeer()函数确实有不同之处：
![](https://i.imgur.com/whRQTLy.png)
 
										Fig 2 ifps-bitswap/src/network.js

这里主要是调用了Libp2p层的dialProtocol()函数，先是尝试使用BITSWAP110协议，如果不行再用BITSWAP100协议，从这里可以看到目前IPFS的BITSWAP协议只有两个版本，但是两个版本并不兼容。在《IPFS源码分析之get流程（四）》的Fig 1是最初开始连接节点调用的Libp2p层的dial()函数，但这里用的是dialProtocol()函数，下面一起贴出来比较:
![](https://i.imgur.com/AtQb24i.png)
 
									Fig 3 libp2p/src/index.js

两个函数几乎一模一样，唯一的区别在287行和307行调用this.switch.dial()函数时，后者多传了protocol参数，这个protocol就是BITSWAP110或BITSWAP100。那就得去到this.switch.dial()函数看有何区别，在《IPFS源码分析之get流程（四）》的Fig 3，分别在48行和58行出现判断是否有protocol参数。第一次判断是在查看本地switch有保存着与对方的复用连接时，如果有protocol参数，则直接拿出保存着的复用连接，如果没有则直接返回，无需取出复用连接。第二次判断是在建立了复用连接后，如果没有protocol参数直接返回，如果有则继续执行后面的gotMuxer()函数。

根据总体get流程，也就是连接一个新节点（没有保存着关于这个节点的任何连接），先用dial()函数建立了普通连接和复用连接，到了调用dialProtocol()函数时switch已经保存着与该节点的复用连接，经过48行的判断是否有protocol参数后，直接调用gotMuxer()函数，这个函数主要调用了两个函数openConnInMuxedConn()和protocolHandshake()。前者在已经建立好的复用连接中新建一个流传给后者，看看后者protocolHandshake()函数：
![](https://i.imgur.com/C8SPWwC.png)
 
									Fig 4 libp2p-switch/src/dial.js

看到ms.handle和ms.select就应该想到在《IPFS源码分析之get流程（四）》的Fig 4中，在用某个transport协议建立了连接后，将此连接升级为加密连接的cryptoDial()函数，这里改正一下之前的一个错误，ms.handle()函数和对方握手，比较的是两方的multistream版本而不是加密版本是否一致，当前IPFS默认是'/multistream/1.0.0'。在cryptoDial()函数中，ms.select()函数协调的是双方的加密算法，这里215行的ms.select()函数协调的是双方的共同使用的Bitswap协议版本。现在_dialPeer()函数的任务结束，返回复用连接和商量好使用的Bitswap协议版本，回到Fig 1的150行，根据选好的Bitswap协议版本，使用该版本的序列化函数如msg.serializeToBitswap110()将消息序列化：
![](https://i.imgur.com/zCAdRO2.png)
 
								Fig 5 ipfs-bitswap/src/types/message/index.js

这里贴出了两个Bitswap版本的序列化函数，可比较异同之处。唯一区别是第70行100版本序列化后的消息中包含blocks一项，而93行110版本序列化后的消息中包含payload一项，这两项都是根据原来msg实例中的this.blocks内容填充，如果这个消息是一个请求文件块的消息msg，则this.wantlist中包含一个个想要请求的CID，而this.blocks为空。如果这个消息是一个发送文件块的消息msg，则this.wantlist为空，this.blocks为要发送的文件块。最后都是再调用pbm.Message.encode(msg)处理格式。序列好之后就得发了，在Fig 1的162行，调用writeMessage()函数将序列化后的消息serialized放入前面拿到的复用连接上，也就是pull到复用连接的流。这个时候消息就发送出去了，最后还要更新'dataSent'和'blocksSent'两个计数器的值。

看看对方节点如何收到这个消息。

之前在《IPFS源码分析之Libp2p层》中说了整个Libp2p层各个模块的初始化和启动，但在启动那里只具体讲了discovery模块的启动，没有讲switch模块的启动，先看看它的启动函数：

![](https://i.imgur.com/nkreN7d.png)
 
									Fig 6 libp2p-switch/src/index.js

第85行根据本节点所能支持的transport协议分别调用transport.listen()函数启动监听服务器：
![](https://i.imgur.com/CJF6g9H.png)
 
												Fig 7 libp2p-tcp/src/index.js

在这个函数的上面就是之前用到的dial()函数。62行先判断有没有传入handler参数，从Fig 6可以看到是没有的，所以调用swtch.protocolMuxer(key)函数拿出默认的收到连接的处理函数，key参数就是Bitswap协议的版本。然后79调用具体某个transport协议的createListener(handler)函数来创建服务器，传入默认的连接处理函数作为参数handler，下面再次以TCP为例建立TCP服务器：
![](https://i.imgur.com/uQKObt0.png)
 
							Fig 8 libp2p-tcp/src/listener.js

23行调用net模块的net.createServer()函数创建服务器，返回一个监听套接字socket，这个套接字进程就一直运行着，如果有连接过来，27行的getMultiaddr(socket)函数就能获得这条连接的对方地址，然后trackSocket(server, socket)函数将这条连接和对应地址保存，最后是40行handler(conn)函数用上层逻辑处理这条连接。所以关键还是连接处理函数如何处理收到的消息。

刚刚说到由swtch.protocolMuxer(key)将默认的连接处理函数取出，在switch模块的构造函数中可以看到这个函数的来源，this.protocolMuxer = ProtocolMuxer(this.protocols, this.observer)，这里的this.protocols参数本身是个列表，装着不同版本Bitswap协议的连接的处理函数，在《IPFS源码分析之Bitswap层（二）》的Fig 8是Bitswap层的network模块的启动代码，其中有一行：this.libp2p.handle(BITSWAP100, this._onConnection)，当然还有相似的另外一行调用，只是参数变成BITSWAP110。libp2p.handle()函数将使用两个版本Bitswap协议的连接的处理函数装入到switch模块的protocols列表中，但是这两个版本的连接的处理函数都一样是this._onConnection()。先继续看具体看看ProtocolMuxer()函数的内部逻辑：

![](https://i.imgur.com/nHC3Tev.png)
 
								Fig 9 libp2p-switch/src/protocol-muxer.js

从16行到23行包装了一个handler()函数，27行在调用ms.addHandler()实际地将处理函数添加到服务器。当一个新的连接过来时，服务器解析原始数据能知道protocolName即Bitswap协议的版本，并将它和这条连接流传给handler()函数，handler()函数中在21行对该连接添加了监控，最后第22行调用的handlerFunc()函数就是从protocols列表中拿出来_onConnection()函数。看看这个最终的处理连接的函数：
![](https://i.imgur.com/zr4vFUO.png)
 
							Fig 10 libp2p-bitswap/src/network.js

使用pull家族的流，将连接conn流的内容作为输入源，80行调用Message.deserialize(data, cb)把流中的数据反序列化，就得到了对方传来的消息：

![](https://i.imgur.com/9qAGn3V.png)
 
								Fig 11 ipfs-bitswap/src/types/message/index.js

 
![](https://i.imgur.com/d8wW2Nv.png)

									Fig 12 ipfs-bitswap/src/types/message/index.js

这个函数很长，分成两个图。整个过程都是和序列化函数的流程相反，先调用pbm.Message.decode(raw)函数还原消息格式，然后146行查看wantlist中有没有东西，有的话这就是个请求文件块的消息，里面的内容也就是请求文件块的CID。下面在查看blocks和payload中有无内容，上面说到BITSWAP100版本序列化后的的消息有blocks这一项，而BITSWAP110版本序列化后的的消息将此替换成了payload，所以对不同Bitswap版本的连接可以用同一个连接处理函数就是因为它的反序列化函数能够区分。如果是对方发送文件块过来，那么该消息中的blocks或payload就是文件块数据了，这里先讲对方发送的是请求文件块的消息，所以只有wantlist中有内容，后面不用管。

下次再看Fig 10中的bitswap._receiveMessage(peerInfo.id, msg, cb)函数怎么处理反序列化后的消息，即怎么回应对方请求文件块。

**个人微信：xxb3059 & 公众号：区块丛林 欢迎交流**

