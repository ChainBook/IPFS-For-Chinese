
## IPFS网络的Encryption层设计

### 1 具有PKI 特性

玩密码学的童鞋应该都不会陌生这个，**PKI技术：Public Key Infrastructure ，也叫公钥基础设施，**PKI是一种遵循标准的利用公钥加密技术为电子商务的开展提供一套安全基础平台的技术和规范。

 - HTTP协议中PKI的使用：可参考 [HTTPS协议详解(三)：PKI 体系](https://blog.csdn.net/hherima/article/details/52469488)
 - IPFS协议中PKI的使用：Node ID生成，IPNS挂载，私有集群网络搭建


#### 1.1 PKI特性：Node ID生成

Eg：

```
difficulty = <integer parameter>
n = Node{}
do {
n.PubKey, n.PrivKey = PKI.genKeyPair()
n.NodeId = hash(n.PubKey)
p = count_preceding_zero_bits(hash(n.NodeId))
} while (p < difficulty)
```
这是IPFS白皮书中在初始化生成NodeID（n）时，所执行的一段伪代码：

我们可以看到，在创建过程中将通过`PKI.genKeyPair()`(默认用的[RSA非对称加密算法](https://baike.baidu.com/item/RSA%E7%AE%97%E6%B3%95/263310?fr=aladdin&fromid=210678&fromtitle=RSA))生成一套公钥（`n.PubKey`）和私钥（`n.PrivKey`），`NodeId` 将通过`hash(n.PubKey)`公钥进行签名。

这样做的好处是当Node 节点在第一次连接对等Peer方时，可以通过互相交换公钥，来验证`hash(other.PublicKey) = other.NodeId`是否成立 ，如果成立，则确定可信身份，保持通信，否则连接终止，防范网络中的一些嗅探攻击。（这和我们在做后端微服务之前的接口通信时，常设置变量参数参与的对称加密签名来防止网络攻击一样）

在本地，我们可以通过`ipfs id `随时来查看我们的公钥（PublicKey）和其对应的NodeID：

![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/ipfs/re-encryption/ipfs-pub1keypair.png)

也可以通过vim ~/.ipfs/config 来进行私钥（PrivKey）的查看：


![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/ipfs/re-encryption/ipfs-prikey.png)


#### 1.2 PKI特性：IPNS挂载


这里要提一下之前`搬山工`童鞋小密圈发起的一个提问：  如何在不同节点中更新同一个IPNS Hash的内容？的问题

我们直接使用
```
ipfs name publish QmSomeHash

```
是默认挂载一个文件空间到的ipns/nodeID上，因为这边默认读取的公钥文件是生成NodeID的`Self公钥`，但是我们可以通过新生成一份`代理公钥`来实现不同节点中同一份IPNS地址的内容更新：

```

//A节点生成mykey：
ipfs key gen --type=rsa --size=2048 mykey

//公钥一般存储在./ipfs/keystore 中

//拷贝mykey 至B节点同目录下，更新挂载点，更新内容
ipfs name publish --key=mykey QmSomeHash

```

Type可选，默认是`RSA`，但也支持`ed25519`：

```go

//From：https://github.com/ipfs/go-ipfs/blob/master/core/commands/keystore.go
		var sk ci.PrivKey
		var pk ci.PubKey

		switch typ {
		case "rsa":
			if !sizefound {
				res.SetError(fmt.Errorf("please specify a key size with --size"), cmdkit.ErrNormal)
				return
			}

			priv, pub, err := ci.GenerateKeyPairWithReader(ci.RSA, size, rand.Reader)
			if err != nil {
				res.SetError(err, cmdkit.ErrNormal)
				return
			}

			sk = priv
			pk = pub
		case "ed25519":
			priv, pub, err := ci.GenerateEd25519Key(rand.Reader)
			if err != nil {
				res.SetError(err, cmdkit.ErrNormal)
				return
			}

			sk = priv
			pk = pub
		default:
			res.SetError(fmt.Errorf("unrecognized key type: %s", typ), cmdkit.ErrNormal)
			return
		}

```

#### 1.3 PKI特性：私有集群网络搭建

通过官方提供的共享密钥生成工具：https://github.com/Kubuxu/go-ipfs-swarm-key-gen

```go

//From:https://github.com/Kubuxu/go-ipfs-swarm-key-gen/blob/master/ipfs-swarm-key-gen/main.go
func main() {
	key := make([]byte, 32)
	_, err := rand.Read(key)
	if err != nil {
		log.Fatalln("While trying to read random source:", err)
	}

	fmt.Println("/key/swarm/psk/1.0.0/")
	fmt.Println("/base16/")
	fmt.Print(hex.EncodeToString(key))
}
```

生成一份公共的`swarm.key`，存储在不同节点的上`~/.ipfs/`根目录下，

在我们添加完各个节点的bootstrap地址后，执行`ipfs swarm connect`操作后，会对公共密钥`swarm.key`进行再校验：

```go

//From：https://github.com/ipfs/go-ipfs/blob/master/core/commands/swarm.go

	for _, c := range conns {
		swcon, ok := c.(*swarm.Conn)
		if ok {
		ci.Muxer = fmt.Sprintf("%T", swcon.StreamConn().Conn())
		}
	}
```

- [具体细节可以参考IPFS指南文章：私有网络的搭建与使用](https://mp.weixin.qq.com/s?__biz=MzUwOTE3NjY3Mw==&mid=2247483971&idx=1&sn=0c59f248a97fe96bd724fbc558471b18&chksm=f9177c4dce60f55b68f22fc1d7251860785b8872b505c4942f3ac08179bfcb715c7cae6b4d07&mpshare=1&scene=1&srcid=0519fbz7CrJdNqSKRcnAAJIU&key=0ba51a183a3efea036977696d92582582c6fdfeffad7e4214ffe08257de6576a5a7356b1fa877996d1fe88d0c968be18a1be01e105311c975369575393c19537bce8e5fa48426e594b7e9891c8689471&ascene=0&uin=NzkzNjUzMDYx&devicetype=iMac+MacBookPro14%2C2+OSX+OSX+10.13.3+build(17D102)&version=12010210&nettype=WIFI&lang=zh_CN&fontScale=100&pass_ticket=OF8lrCWLqFB2rIWkfgDKttNoOH2g%2FK8TY1jdbxehwped9yEd%2B8u%2FmAD46KqpZLh7)


**虽然以上三个过程没有像HTTPS一样引入数字证书，但是也足够诠释了 PKI 的设计机制。**


### 2 文件寻址加密

我们添加的`地址+文件名`都将通过`Multihash`模块加密映射成大家看到的Hash指纹，默认均采用`sha2-256`

```go

//From：https://github.com/ipfs/go-ipfs/blob/master/core/commands/add.go	

Options: []cmdkit.Option{
	     ...
		cmdkit.StringOption(hashOptionName, "Hash function to use. Implies CIDv1 if not sha2-256. (experimental)").WithDefault("sha2-256"),
	},

```

`Multihash`支持多达10余种哈希算法，目前只在IPFS本地测试环境下才可配置：

```go
// From: https://github.com/multiformats/go-multihash/blob/master/multihash.go
// Names maps the name of a hash to the code
var Names = map[string]uint64{
	"id":           ID,
	"sha1":         SHA1,
	"sha2-256":     SHA2_256,
	"sha2-512":     SHA2_512,
	"sha3":         SHA3_512,
	"sha3-224":     SHA3_224,
	"sha3-256":     SHA3_256,
	"sha3-384":     SHA3_384,
	"sha3-512":     SHA3_512,
	"dbl-sha2-256": DBL_SHA2_256,
	"murmur3":      MURMUR3,
	"keccak-224":   KECCAK_224,
	"keccak-256":   KECCAK_256,
	"keccak-384":   KECCAK_384,
	"keccak-512":   KECCAK_512,
	"shake-128":    SHAKE_128,
	"shake-256":    SHAKE_256,
}
```

### 3 DAG分片

这块应该是IPFS内容寻址的设计精髓，当然，IPFS采用Merkle DAG技术对整个系统带来的优点有很多，今天这里我们只讨论DAG分片对文件的安全保护属性：

这是我之前上传的  自己的头像数据：`Qmdv... `：

![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/ipfs/re-encryption/encry-1.png)


DAG分片之后：`QmVo...`,`QmYG...`，`QmdF...`，`QmXA...`，`QmSN...`
![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/ipfs/re-encryption/encry-2.png)

我们拿到的一些分片文件是[RAW格式](https://baike.baidu.com/item/RAW%E6%A0%BC%E5%BC%8F/1103387)（原始图像编码数据编码）的碎片文件，只获取其中的几个碎片数据是无法通过DAG解析器拼合成原图像的，一定意义上提升了网络中文件的安全性：

![](http://daijiale-cn.oss-cn-hangzhou.aliyuncs.com/djl-blog-pic/ipfs/re-encryption/encry-3.png)

观察IPFS的Core层源码可以了解到：DAG解析器这部分，专门适配了四种格式：

- [json](https://baike.baidu.com/item/JSON/2462549?fr=aladdin)
- [raw](https://baike.baidu.com/item/RAW/218261)
- [cbor](https://baike.baidu.com/item/CBOR)
- [protobuf](https://baike.baidu.com/item/Protocol%20Buffers/3997436)

```
//From :https://github.com/ipfs/go-ipfs/blob/master/core/coredag/dagtransl.go


// DagParser is function used for parsing stream into Node
type DagParser func(r io.Reader, mhType uint64, mhLen int) ([]ipld.Node, error)

// FormatParsers is used for mapping format descriptors to DagParsers
type FormatParsers map[string]DagParser

// InputEncParsers is used for mapping input encodings to FormatParsers
type InputEncParsers map[string]FormatParsers

// DefaultInputEncParsers is InputEncParser that is used everywhere
var DefaultInputEncParsers = InputEncParsers{
	"json":     defaultJSONParsers,
	"raw":      defaultRawParsers,
	"cbor":     defaultCborParsers,
	"protobuf": defaultProtobufParsers,
}

```


### 4 More

当然IPFS的Encryption层还有很多其他的设计，可能我还没研究到，如果大家有新发现，可以随时提交issue和PR，一起探讨。

