# IPFS源码分析之Bitswap层（二） #
前面的两篇对于Bitswap来说是功能实现的基础，libp2p和repo.blocks的东西会经常被调用。先看看Bitswap模块所包含的文件的结构及其实现的功能。

 ![](https://i.imgur.com/F36qd4c.png)

										Fig 1 Bitswap模块文件结构
Bitswap的初始化就是将这些子功能添加进来，如Bitswap类的构造函数所示：
 ![](https://i.imgur.com/PavAOIi.png)

										Fig 2 Bitswap模块的index.js
在上图中可以看到this._stats被后面很多个子功能用到，它控制着所有操作，在initialCounters数组中比如文件块的收发的频率，指向的Stats类构造函数如下：
 ![](https://i.imgur.com/ySkzAZ5.png)

										Fig 3 ipfs-bitswap/src/stats/index.js
这里又引到另一个类：this._global = new Stat(initialCounters, options)，这个类在同目录的stat.js文件中有定义，再看看它的构造：
 ![](https://i.imgur.com/NFOiP5z.png)

									Fig 4 ipfs-bitswap/src/stats/stat.js
最后引到MovingAverage类，这个类实现了EMA衰减的各项指标如移动平均值，Bitswap实现了三种衰减速率，后面再根据最近的收发频率去动态调节。
回到Bitswap的构造函数，接着实例化一个节点的网络操作(如收发消息，发现连接节点) Network类，这个的构造函数比较简单。再实例化一个DecisionEngine类，这个功能对Bitswap来说相当于决策中心，它的构造函数如下：
 ![](https://i.imgur.com/h1kuo9o.png)

							Fig 5 ipfs-bitswap/src/decision-engine/index.js
重点最后一句，使用Lodash的debounce方式控制this._processTasks这个函数的执行条件或者说频率，并将这种控制效果由this._outbox触发。关于debounce和Lodash的另一个控制方式throttle，这里有个很好的讲解Lodash官方源码的博客https://www.tuicool.com/articles/ZNRj2eA。
在这里debounce的leading和trailing参数都是默认的，只传入了时延100ms，总的效果是调用this._outbox，经过100ms的时延后this._processTasks函数才执行，而且在这100ms内this._outbox没有被再调用。如果被再调用，则根据最后一次调用的时间推延100ms才执行this._processTasks。该函数内容如下：

 ![](https://i.imgur.com/evqZbkm.png)

							Fig 6 ipfs-bitswap/src/decision-engine/index.js
函数功能在图中有注释。当有文件块需要发送，即this._tasks添加了一些任务，然后this._outbox被调用，如果在时延的100ms内，又有一些文件块需要发送，即this._tasks又添加了一些任务，则重新计算100ms时延，直到在最后的100ms内没有被重新调用，则实际地执行this._processTasks将累积的文件块一次性处理。由于前后需要发送的文件块的目的peer很大可能一样，这样做就减少了重新和相同peer建立connection的时间。

Bitswap的构造函数中接着实例化一个WantManager类，追踪管理本节点需要的文件块，由这个类的一个属性this.wantlist = new Wantlist(stats)负责需要的文件块列表，Wantlist = require('../types/wantlist')。
最后实例化一个Notifications类，由这个类区分开哪些接收的文件块是想要的，哪些是不想要的，并通知上层逻辑。Bitswap初始化完后就是启动，self._bitswap.start()。在Bitswap模块的index.js文件可看到start()，逐个启动WM，Network，DecisionEngine这三个子功能实例。前两个的start()稍有些内容。

 ![](https://i.imgur.com/IVohIna.png)

				Fig 7 ipfs-bitswap/src/want-manager/index.js
每隔60秒向本节点所连接的peer发送想要的文件块CID及优先级。
 ![](https://i.imgur.com/7vrpqVz.png)

											Fig 8 ipfs-bitswap/src/network.js
开始从Bitswap层监听节点连接和重连之前保持连接的节点。
以上就是Bitswap的整体结构，具体的运转和更多的函数执行再用实际的一次搜索文件和上传文件来结合分析。
## 缩写词解释： ##
1.	EMA，Exponential Moving Average，指数平均数。也叫EXPMA指标，它也是一种趋向类指标，指数平均数指标是以指数式递减加权的移动平均。各数值的加权是随时间而指数式递减，越近期的数据加权越重，但较旧的数据也给予一定的加权。

**个人微信：xxb3059 & 公众号：区块丛林 欢迎交流**
