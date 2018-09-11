# IPFS源码分析之API结构 #
之前想要从一个具体的“get” API来串联起IPFS的整个运作流程，所以就先看看IPFS的API结构。它的API包括js API和命令行，非常丰富，不仅可以用在IPFS上的文件和网络，还可以用在本地的文件系统，即使本地文件系统是多样的。
 ![](https://i.imgur.com/y8hrR48.png)

													Fig 1 API结构
在图中可以看到两种API使用方式的不同，命令行方式能够offline访问，这个offline并不是说不需要连接上IPFS的主网，而是无需再另外启动一个IPFS dameon提供访问网络的接口，因为在使用命令行如“ipfs get Qmxxxxxx”，直接调用了IPFS Core建立一个完整节点实例。使用js API就需要先以http方式访问提供了对应网络接口的节点。npm install ipfs-api后就可以在node下使用这些接口，而提供了对应网络接口的节点可以是go版本节点，也可以是在node下运行一个HTTP-API实例，因为HTTP-API实例会调用了IPFS Core建立一个完整节点实例，CLI,HTTP-API,IPFS Core都在ipfs modules中。下面以一个js的get API为例。
![](https://i.imgur.com/Gg9tmRo.png)

												Fig 2 get example
其中ipfs = require(ipfs-api)，ipfs-api模块的index.js中对各种api纳入，最主要的几行代码如下：
 ![](https://i.imgur.com/bxFWSKP.png)

											Fig 3 ipfs-api/src/index.js
图中的sendRequest()函数作为参数传给每个api的定义函数中，这个函数将具体的api封装成http请求发送到ipfs dameon。sendRequest = require('./utils/send-request')，所以在ipfs-api模块文件夹中可以看到这个函数的定义文件。
function requestAPI (config, options, callback) {……}，函数的主要代码如下：
 ![](https://i.imgur.com/c6TqZuN.png)

										Fig 4 ipfs-api/src/utils/send-requext.js
 ![](https://i.imgur.com/6u1AIdu.png)

										Fig 5 ipfs-api/src/utils/send-requext.js
当节点返回所要获取的文件内容时，仍然有这个文件的另外一个函数onRes()处理：
![](https://i.imgur.com/Ou9Rjdv.png)

									Fig 6 ipfs-api/src/utils/send-requext.js
更多的处理细节需要可以仔细看看这个文件。每一个API都有一个js文件定义，和文件操作有关的都在files目录下，可以找到get API的定义文件：
 ![](https://i.imgur.com/NvH28lu.png)

										Fig 7 ipfs-api/src/files/get.js
上图中的send正式前面的sendRequest()函数，以上是请求端，其他的API也一样。接下来是节点作为http server端的源码，即HTTP-API模块，这个模块不是单独的，它包含在IPFS模块的http目录下。但它可以单独使用。下面是这个模块的start()函数：
 ![](https://i.imgur.com/ssl3Y8i.png)

										Fig 8 ipfs/src/http/index.js
从图中可以看到建立了一个IPFS节点实例，其中IPFS=require(ipfs) ，调用了IPFS Core。这也是http-api要放在IPFS模块中的原因。 然后接着实现一个http服务器：

 ![](https://i.imgur.com/7DcgI1z.png)

										Fig 9 ipfs/src/http/index.js
其中Hapi = require('hapi')，IPFS选择hapi作为http框架主要原因我觉得是让API的编写更加清晰，虽然这里没有看到使用swagger插件来生成API文档，但总体API的编写只需要像写配置文件一样，方便维护，hapi是沃尔玛的团队开发的，已经在生产环境中得到充分的认可，虽然没有express那么丰富和完善，但性能并不输给express，尤其是在IPFS中需要同时处理大量的文件上传下载等请求时，hapi是个不错的选择，IPFS的团队也开发了需要的hapi插件。IPFS的上传下载等操作一般可以通过两种形式。http://localhost:5001/ipfs/CID和http://localhost:8001/CID。这里假设连接本地节点查看某个文件。所以图中建立了两个server监听，而且可以通过select来选择。下图是路由编写方式：
 ![](https://i.imgur.com/oNBNl4U.png)

								Fig 10 ipfs/src/http/api/routes/files.js
对应的所有和file相关的API的路由都写在这个文件，config中的pre是注明当请求过来时先用method的方法预处理，再交给handler的方法，预处理函数可以做许多合法性验证，虽然有valid插件，但不够IPFS的特殊需求。对于get API，预处理函数主要是验证请求带的url参数是否是CID，最后再交给get.handler()，这两个函数都在ipfs/src/http/api/resources/files.js中，下次再看看怎么从IPFS网络里拿到这个文件。

**个人微信：xxb3059 & 公众号：区块丛林 欢迎交流**
