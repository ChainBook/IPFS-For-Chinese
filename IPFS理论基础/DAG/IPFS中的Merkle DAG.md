[![img](http://ipfser.org/wp-content/uploads/2018/01/25104405346.bmp)](http://ipfser.org/wp-content/uploads/2018/01/25104405346.bmp)

**今天带大家来深入探索一下IPFS的核心数据结构Merkle DAG**

**什么是** **Merkle DAG？**

Merkle DAG是IPFS系统的核心概念之一，当然Merkle DAG并不是IPFS团队发明的，它来自于Git数据结构，ipfs团队进行了改造（这一点ipfs团队一直是一个很努力的团队，并不是直接拿来使用，而是在此基础上修改更适合项目的使用）。

Merkle DAG的全称是 Merkle directed acyclic graph（默克有向无环图）。它是在Merkle tree基础上构建的，Merkle tree是由美国计算机学家merkle于1979年申请的专利。Merkle DAG跟Merkle tree很相似，但不完全一样，比如：Merkle DAG不需要进行树的平衡操作，非叶子节点允许包含数据等。

[![img](http://ipfser.org/wp-content/uploads/2018/01/25104505674.bmp)](http://ipfser.org/wp-content/uploads/2018/01/25104505674.bmp)

Merkle DAG

Merkle DAG拥有如下的功能：

- *内容寻址：使用多重哈希来唯一识别一个数据块的内容*
- *防篡改：可以方便的检查哈希值来确认数据是否被篡改*
- *去重：由于内容相同的数据块哈希是相同的，可以很容去掉重复的数据，节省存储空间*

IPFS的数据对象格式如下：

```
type IPFSLink struct {
  Name string             // link 的名字
  Hash Multihash        // 数据的加密哈希
  Size int                    // 数据大小
}
12345
```

```
type IPFSObject struct {
  links []IPFSLink             // link数组
  data []byte        // 数据内容
}
1234
```

IPFS让应用可以完全控制对象的数据字段，也就是说应用可以随意定义自己的data类型和结构，甚至可以是一些IPFS系统无法理解的数据结构，灵活度非常的大。

**下面以例子的方式带大家来看看IPFS的数据对象如何工作**

**第一步：准备数据**

一张图片，这张图片是小编在一个叫做 First 的地方拍摄的，好漂亮有没有。

[![img](http://ipfser.org/wp-content/uploads/2018/01/25104617451.bmp)](http://ipfser.org/wp-content/uploads/2018/01/25104617451.bmp)

文件名：first.JPG  文件大小：3646K

（如果你要测试的话，使用一个超过256k的数据文件比较好，因为ipfs当前的数据分片是 256k大小一个）

**第二步：执行****ipfs add** **命令，添加文件**

```
localhost:ipfs_pic tt$ ipfs add first.JPG
added
QmSnqm7y5SZ8tBV4c5Qjsu7qTZQQ3vHgPH4sCjJnh5cRSL first.JPG
123
```

**第三步：使用命令** **ipfs -ls -v** **来查看文件的分片**

如下所示，可以看到文件被分成了15个block。每个block大小时256k（除了最后一个）。

```
localhost:ipfs_pic tt$ ipfs ls -v
Hash                                           Size   Name
QmUNquYLeK8vMTX6U6dDhwWNPG5VVywyHoAgbSoCb6JCUe 262158
QmPtKCEs6L6LgFECxuAh9VGxaxRKzGzwC8hsWKUS3wiFi3 262158
QmeVDmX4M7YcDVXuHL691KWhAYxxzmGkvspJJn5Ftt86XR 262158
QmPJ7u77a6Ud2G4PbRoScKyVkf1aHyLrJZPYqC17H6z5ke 262158
QmQrEi8D6kYXjk9UpjbpRuaGhn5fNYH6JQkK5irfpiGanc 262158
QmRQ1fAFuAvREUFT5e3qp5i1FE9AX93XEjaEwrr79QWBCD 262158
Qmaixh1bG2GiDVZ4U4HBDJ27B6Sxzch1hsDEC3na88uzpE 262158
QmRkDshArwx4wai6vAgATNv4ETPSbh8Ahz7hRDkNsw9LCD 262158
QmXoNXfH3SnVYqk9xugyh5Zj4E4U826CfN4exWXwFrh1id 262158
Qma24pVnzvtKqnNE2aexEwgCYq4YqsAFRpxL5MX5ueFpgL 262158
QmTSVtNETAhugLDEpNTbRE2aKK2WrhunVNsbb6Z7htYZET 262158
QmdQy6wQNMyrbZfVarHMSktgJTZrbsaUkAEZY7ACZWmDVa 262158
QmatUyTZRYEj2Wi61RZ2EUqy78LJTAmz18vZYzpN2AZaES 262158
QmRK9tCe4Q8xHyS1vC9qX3BDYMSscFtydaNwVYDx1PXYFA 262158
QmUs5kkcutfbHzCKczAXcXri9cmfCXSincN5WTp8ard1fR 64097
1234567891011121314151617
```

当执行 ipfs add 的时候，文件first.JPG的数据被分成了一个个大小均等的block并且构造一个Merkle DAG把文件数据片组织起来。

[![img](http://ipfser.org/wp-content/uploads/2018/01/25105349606.bmp)](http://ipfser.org/wp-content/uploads/2018/01/25105349606.bmp)

可以使用IPFS提供的命令来查看数据块（block）的信息，数据块block下面还允许链接子数据块（sub-block，本文的例子中没有涉及）。

相关的查询命令如下：

- *ipfs block stat:* 查询block*的数据大小，不包含子块。*
- *ipfs refs*：列出来数据快的子块信息
- *ipfs ls or ipfs object links*：显示所有的子块和块的大小

例如：

```
localhost:ipfs_pic tt$ ipfs block stat
QmPtKCEs6L6LgFECxuAh9VGxaxRKzGzwC8hsWKUS3wiFi3
Key: QmPtKCEs6L6LgFECxuAh9VGxaxRKzGzwC8hsWKUS3wiFi3
Size: 262158
1234
```

既然每一个数据块我们都能查询获取到，那么我们可以手工来把数据拼起来。比如刚才小编的那张图片，可以使用命令：

```
ipfs cat hash1 hash2 hash2 …… , hashn > first.JPG
1
```

这样就可以手工得到上的那张照片，有兴趣的读者可以自己动手试试。

**直接操作****Merkle DAG**

IPFS可以让我们直接操作Merkle DAG的数据，举个例子：

```
localhost:ipfs_pic tt$ echo "hello world" | ipfs block put
QmZjTnYw2TFhn9Nn7tjmPSoTBoY7YRkwPzwSrSbabY24Kp
localhost:ipfs_pic tt$ ipfs block get
QmZjTnYw2TFhn9Nn7tjmPSoTBoY7YRkwPzwSrSbabY24Kp
hello world
12345
```

怎们样？很魔性吧，我们完全控制了block里面的数据内容和结构，IPFS把Merkle DAG操作权限几乎全部下放给了开发者，开发者可以很容易构造出来自己的数据结构。

IPFS在论文里面提出了可以自己一些潜在的数据结构：

- 键值对存储（key-value stores）
- 关系型数据库（traditional relatioinal databases）
- 三元组存储（Linked Data triple stores）
- 文档发布系统（Linked document publishing systems）
- 通信平台（Linked communications platforms）
- 加密货币区块链（cryptocurrency blockchains）

在此基础上开发者还可以完全自定义自己的数据结构。看到最后一条了吧，IPFS为所有的区块链准备好了数据存储结构，IPFS将作为区块链的基础设施存在。