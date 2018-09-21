# IPFS源码分析之get流程（七） #
接着看请求接收方的Bitswap层怎么处理发序列化后还原的消息，即bitswap._receiveMessage(peerInfo.id, msg, cb)函数：
![](https://i.imgur.com/dsfHOgl.png)
 
										Fig 1 libp2p-bitswap/src/index.js

111行马上先交给engine模块的engine.messageReceived()函数处理：
![](https://i.imgur.com/aIPDSKk.png)
 
										Fig 2 libp2p-bitswap/src/decision-engine/index.js

在《IPFS源码分析之Bitswap层（二）》有说到这个模块及其功能。191行先根据请求这个消息的节点的ID从本地的账本册中找到关于这个节点的账本，如果找不到就要创建一个新的，在engine模块的构造函数中可以看到账本册是个map结构，保存各个节点和对应账本。账本是一个Ledger类实例，看下其构造函数和一些方法：
![](https://i.imgur.com/JwlmdLo.png)
 
									Fig 3 libp2p-bitswap/src/decision-engine/ledger.js

在构造函数中主要属性是第8行wantlist，保存这个节点当前向自己索要的文件块CID，10行exchangeCount是两个人相互发送过的文件块数量，13行accounting中分别记录发送过给该节点多少byte数据和接收过多少byte数据，都是文件块数据，普通消息数据不算。

回到Fig 2，有了账本后，199行，判断对方发送的消息中full标志是不是真，如果是真，则清空原来账本中的wantlist，不是则留用原来的wantlist。这里的full标志在《IPFS源码分析之get流程（五）》中有提到，如果发送消息的节点和本节点是第一次有连接，则这个请求文件块的消息的full标志为真，而且消息中索要的文件块CID在本节点不一定有，因为对方把想要的文件块CID全部发过来，所以本节点就在账本里建一个空的wantlist。如果之前有过连接，则full标志为假，这种情况是对方基本上确定本节点有某个文件块才把请求发过来，所以本节点就直接将索要的文件块CID添加到账本原有的wantlist中。

接着203行调用_processBlocks()函数，如果收到是请求文件块的消息，这个函数就没有作用，如果收到包含文件块的消息，这个函数就处理收到的文件块。这个后面再具体展开。

接着211行对消息中的wantlist包含的每个CID分别处理，如果是撤销索要该CID，则本节点从账本的wantlist中将该CID删除，由223行的_cancelWants()函数执行。如果是新索要的CID，则添加到账本的wantlist中，由225行的_addWants()函数执行。看看这两个函数：

![](https://i.imgur.com/cAsl706.png)
 
						Fig 4 libp2p-bitswap/src/decision-engine/index.js

由于对方曾经向本节点发过消息索要某些文件块，本节点已经将给对方发送这些文件块的任务放进了任务队列，如果现在要撤销请求这些文件块，所以在cancelWants()函数中231行，pullAllWith()函数要从任务队列拿下这些任务。当然，对于正常的get流程，这里就不细说这种情况，认为对方发送的消息中都是正常索要某些文件块。所以主要看_addWants()函数，242行查看一下本地的文件块仓库有没有对方请求的文件块，如果没有则仅仅将请求的CID添加到账本的wantlist中，如果有则将发送文件块给该节点的任务加入队列，最后256行this._outbox()触发_processTasks()执行任务队列的任务。如何触发在《IPFS源码分析之Bitswap层（二）》的Fig 5和Fig 6有详细说明，这里不重复。发送文件块调用到Fig 6的this._sendBlocks()函数：
![](https://i.imgur.com/AIW2TOg.png)
 
							Fig 5 libp2p-bitswap/src/decision-engine/index.js

先计算这次要发送给对方的文件块的大小加上消息的必要信息是不是超过了512 * 1024 bytes，这是一个Message类实例的消息的最大限制，看不出这个设置值有何特殊，但已经很大了，毕竟IP包的MTU默认大小才576 bytes。如果超过了限制，就要分为多个消息来发送，也就是逐个添加文件块，一旦超过限制值，就先将这一部分装到消息中发送，可以看到68行调用this._sendSafeBlocks()函数，其内部87行再调用network.sendMessage()函数来发送消息，这个函数就非常熟悉了。

到此，engine.messageReceived()函数处理完成，作为请求文件块消息的接收者，它的任务也已完成，对方想要的文件块如果本地有就发出去了，从这里也可以看到目前的代码还没有实现IPFS白皮书所说的Bitswap层的信任机制和策略，但发生过文件块交换的节点之间都记录着双方发送过多少文件块，不过后续的策略还没跟上。

现在角色转换回来，本节点收到请求对方返回的文件块，由于这些文件块都是封装在消息中，所以接收消息的流程和前面一样，包括对消息的反序列化，因为双方用同一个Bitswap协议版本。直接到Fig 1，同样先进入engine.messageReceived()函数处理，前面也讲到，处理返回文件块的消息要用到Fig 2中203行的_processBlocks()函数，因为消息中的wantlist为空，后面的_cancelWants()函数和_addWants()函数就用不到了。在_processBlocks()函数中，先是更新账本中前面提到的exchangeCount和accounting两个属性的值，然后调用this.receivedBlocks()函数，传入参数是消息中包含的文件块的CID：

![](https://i.imgur.com/EcBHW1M.png)
 
						Fig 6 libp2p-bitswap/src/decision-engine/index.js

这个函数的处理倒是挺有趣，174行查看本节点的账本册中所有账本的wantlist中是否有这些CID，注意这里的wantlist包含是其他节点向本节点索要的CID。如果有，就把这些收到的文件块也发给该账本对应的节点。总的意思是，A节点向本节点请求某个CID文件块，但本节点没有，只能暂时添加到关于A的账本的wantlist，现在本节点的一个用户通过get请求这个相同的CID文件块，由于本节点没有，通过IPFS网络向B节点请求这个文件块，然后B节点返回了这个文件块。本节点看了一下账本册，发现A节点也需要这个文件块，那就顺手转发给A节点。

回到Fig 1的126行，前面讲处理请求文件块的消息的时候没讲这里以及后面，因为不需要，现在是处理返回文件块的消息，这里就有用了。先查看want-manager模块的wantlist有没有消息里的这些文件块的CID，如果有，那正是本节点想要的，然后将wantlist中收到的CID删除。最后136行调用this._handleReceivedBlock()函数继续处理收到的消息中每个文件块：
![](https://i.imgur.com/iPdrNmE.png)
 
								Fig 7 libp2p-bitswap/src/index.js

先判断本地的文件块仓库blockstore有没有这些收到的文件块，如果有则直接返回，如果没有就调用this._putBlock()函数添加到本地文件块仓库blockstore，然后更新stats模块的'blocksReceived'和'dataReceived'计数器。当然，this._putBlock()函数不仅仅是添加文件块，还有其他任务：
![](https://i.imgur.com/8nouKX5.png)
 
									Fig 8 libp2p-bitswap/src/index.js

190行的this.blockstore.put()函数才是添加文件块到本地文件块仓库blockstore，然后195行的this.notifications.hasBlock(block)触发notifications模块的事件以告知收到了想要的文件块，接着197行this.network.provide()函数更新Libp2p层的DHT，最后203行调用this.engine.receivedBlocks()函数也就是上面的Fig 6，又一次热心地查看一下其他节点是不是也需要这些文件块。比较重要的是this.network.provide()函数，这个函数内部直接调用this.libp2p.contentRouting.provide()函数：

![](https://i.imgur.com/DIQrP2Q.png)
 
							Fig 9 libp2p/src/content-routing.js

在这里碰到了熟悉的函数findProviders()，这个在《IPFS源码分析之get流程（三）》的Fig 2出现，那时候是准备从Kad DHT中寻找能够提供想要的CID文件块的节点，而现在收到了这些想要的文件块，那就更新它，其他节点能从这个Kad DHT找到提供这些CID文件块的节点，那就是本节点。

Get流程到这里就结束了，所涉及到的函数基本上覆盖了IPFS中Bitswap层和Libp2p层所有的模块。IPFS的nodejs版本代码在这两个月间也有更新，把Libp2p层的Multistream的相关接口放到了Bitswap层，这里为了和原来的代码保持流畅的联系，就没做更新，但总的流程没有变化，下次再看看IPFS中其他的API和有趣的地方。


**个人微信：xxb3059 & 公众号：区块丛林 欢迎交流**