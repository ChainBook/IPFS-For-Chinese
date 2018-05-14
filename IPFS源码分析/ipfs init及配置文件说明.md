## init_cmd.go

1. 位置

   ```
   cmd/ipfs/init.go/initCmd
   ```

2. 说明

   ```
   1. ./ipfs init 初始化节点； 初始化完成后，会在默认目录  ~/.ipfs/下生成一堆文件；
   2. ./ipfs config show 命令可以显示配置的全部信息；但是显示的不全；
   3. ./ipfs config --help可以查看更多和config相关的命令操作
   ```

3.  ./ipfs/config生成过程

   ```
   1. 如果init的时候，指定了配置文件，则使用配置文件；否则系统将自动生成配置文件；
   2. 配置文件生成主要在 init.go/doInit()/config.Init()中 
   3. 底层文件目录：$IPFS_PATH/go-ipfs/repo/config/init.go Init()函数
   ```

   ![image-20180514152321677](/var/folders/x6/p_w__rzj5l91r8ygv0qrd99w0000gn/T/abnerworks.Typora/image-20180514152321677.png)



## ipfs config说明

```json
{
  "Identity": {
    ## 节点ID，由公钥生成
    "PeerID": "QmWqoYotKsxnPnq1moUCEmQHXXkh2M3znTkdWMjeNsWgZz",  
	## 节点私钥值
      "PrivKey": "CAASqAkwggSkAgEAAoIBAQC9wEkQrVkfOXLkXZfG0s/re0ucVj8Q5WbxF9TtS2RLirCZ4O6Jk8itSO/SLuuOuPYo3l96VokQr29h9GzMyN+q1z+VP1LcgikUNMCmVfnCrgnS7FMN4Vut4mHYMSEt7pErcU5NeE9BfjmDGwszwFd0DymfrEMBNnXT26R57lI44/qyWnGZyWZJXbVCnGeMMkp24K4bmwfvla0C6dNSpwnnql+/seg8Z6s+GmnqDac0zu3rutMxP2d86HnZuSnVX160pSApQD8DB0TYPSFaCahOKdZo4pEAjkc3ltLPbxVxV8sjMsPdm8Qq6tmsyuCkqjEXuYhayTiK3olkmT12vWSJAgMBAAECggEAT3rKYATsLqsGl+coGuzUkIM9gYeStQYR32ynEJoisY2vOVVBNTlEtmi1o2lp24dX/Hhgr8KteOKzGemi5QhCv7GXfXFfyONwR3ltNH8Qtd3mWYYJp+e8WhJX/5Fcn3utLPAx5zs8n2c6udLLF2s6dm+fdLVX/5sLMalvtG8B27fQjyVrA/45Dmk5sTpKdC9VAi43V4Y7eFZQzdoq/5mod3z5MLm9S0qwlRz2gLRrg6raCg/K7XZBCKdZPiW4AyKt98W3Y8LD3f3oVNOpg50ngvSCMm4a27gs0T6Ojyr0SPlF3yMO9gotQR+XPnvaRxyUPdJUdlCmVNrNbMG2yA4d7QKBgQDCyc8vCmvVfmThL4BhJXtcAFxiF+S1sLdtad1+FSwPXNOp9o3M4v9BK37S4uMEN4+rI+xYmZl4WxTseglTKSNBSt64DPY6nh1wAMBA4NvROVYqADXiKnWIbwnUdqZMX4p2BPJZpXTUmg3BMJk6/BtYftT/nIz2xmIZRf2thVJvgwKBgQD5YT8wTkd2lOrlFJxvAzh5HOBAu5m4KTJ8eXi6ILkdgMgP+jkPv25nEldw9quLPGD1Fr2sjqDnJnLHxEicXq0AalQEG0MJIkiH7QCUpLlQdFTLuaWXvPkPkW6tcIXU+qjoYGip7B5JwuXD306AyPAs2ezEH4nsuMDK9MCqDD+yAwKBgQCYo8QzPJtb5Xvv6mVTuyd75NyAEfErX5udpcPntXedYkSLf6WG1KrpysfLQfhbqZ5voernUxYsdlNjLA56mFYEKEN3PtEFBjpTNoNxU8NtpNycdSXEYTlQ/JJbZ87RMl0yNpYjIcD3iPEWXpr02fIj2t/WnjrodnUREQPFIiCDOQKBgQDvHgfw0Z5EXdY9gd3dtEDaII4Gg9uJcjcuk2rnTakyWOF8MHm2V+AMhNHDR0KFZ4ewefW1F63A9mTol5ToGv/XfhzBM0K751uUufPsk2X9dw43qfLV5CUMgG6Xb2VkKlT7PDYfeIAySeb2QZCMfB+PYgZcp8Egcqap9LUoWEZa8QKBgG6Y94uZl88muQZoktHyN+x2MOvc6bd0UIxMvL295L+KCkSHqPVk0n7xXXz8Q7iVigs/s7jftvYgbVR/YgyR6RkrXeiWCOIw9l29r858zm1PtU+KlJqovL71US8+HGGU9CoN+Zoyq3EufBpkdLhU7PuEGOOCyd1WHpJEk/+j4vr/"
  },
   ## 数据库配置信息
  "Datastore": {
    "StorageMax": "10GB",
    "StorageGCWatermark": 90,
    "GCPeriod": "1h",
    "Spec": {
      "mounts": [
        {
          "child": {
            "path": "blocks",
            "shardFunc": "/repo/flatfs/shard/v1/next-to-last/2",
            "sync": true,
            "type": "flatfs"
          },
          "mountpoint": "/blocks",
          "prefix": "flatfs.datastore",
          "type": "measure"
        },
        {
          "child": {
            "compression": "none",
            "path": "datastore",
            "type": "levelds"
          },
          "mountpoint": "/",
          "prefix": "leveldb.datastore",
          "type": "measure"
        }
      ],
      "type": "mount"
    },
    "HashOnRead": false,
    "BloomFilterSize": 0
  },
  "Addresses": {
    "Swarm": [
      "/ip4/0.0.0.0/tcp/4001",
      "/ip6/::/tcp/4001"
    ],
    "Announce": [],
    "NoAnnounce": [],
    "API": "/ip4/127.0.0.1/tcp/5001",
    "Gateway": "/ip4/127.0.0.1/tcp/8080"
  },
  ## 系统挂载点
  "Mounts": {
    "IPFS": "/ipfs",
    "IPNS": "/ipns",
    "FuseAllowOther": false
  },
  "Discovery": {
    "MDNS": {
      "Enabled": true,
      "Interval": 10
    }
  },
  "Routing": {
    "Type": "dht"
  },
  "Ipns": {
    "RepublishPeriod": "",
    "RecordLifetime": "",
    "ResolveCacheSize": 128
  },
	## 种子连接节点
  "Bootstrap": [
    "/dnsaddr/bootstrap.libp2p.io/ipfs/QmNnooDu7bfjPFoTZYxMNLWUQJyrVwtbZg5gBMjTezGAJN",
    "/dnsaddr/bootstrap.libp2p.io/ipfs/QmQCU2EcMqAqQPR2i9bChDtGNJchTbq5TbXJJ16u19uLTa",
    "/dnsaddr/bootstrap.libp2p.io/ipfs/QmbLHAnMoJPWSCR5Zhtx6BHJX9KiKNN6tpvbUcqanj75Nb",
    "/dnsaddr/bootstrap.libp2p.io/ipfs/QmcZf59bWwK5XFi76CZX8cbJ4BhTzzA3gU1ZjYZcYW3dwt",
    "/ip4/104.131.131.82/tcp/4001/ipfs/QmaCpDMGvV2BGHeYERUEnRQAwe3N8SzbUtfsmvsqQLuvuJ",
    "/ip4/104.236.179.241/tcp/4001/ipfs/QmSoLPppuBtQSGwKDZT2M73ULpjvfd3aZ6ha4oFGL1KrGM",
    "/ip4/128.199.219.111/tcp/4001/ipfs/QmSoLSafTMBsPKadTEgaXctDQVcqN88CNLHXMkTNwMKPnu",
    "/ip4/104.236.76.40/tcp/4001/ipfs/QmSoLV4Bbm51jM9C4gDYZQ9Cy3U6aXMJDAbzgu2fzaDs64",
    "/ip4/178.62.158.247/tcp/4001/ipfs/QmSoLer265NRgSp2LA3dPaeykiS1J6DifTC88f5uVQKNAd",
    "/ip6/2604:a880:1:20::203:d001/tcp/4001/ipfs/QmSoLPppuBtQSGwKDZT2M73ULpjvfd3aZ6ha4oFGL1KrGM",
    "/ip6/2400:6180:0:d0::151:6001/tcp/4001/ipfs/QmSoLSafTMBsPKadTEgaXctDQVcqN88CNLHXMkTNwMKPnu",
    "/ip6/2604:a880:800:10::4a:5001/tcp/4001/ipfs/QmSoLV4Bbm51jM9C4gDYZQ9Cy3U6aXMJDAbzgu2fzaDs64",
    "/ip6/2a03:b0c0:0:1010::23:1001/tcp/4001/ipfs/QmSoLer265NRgSp2LA3dPaeykiS1J6DifTC88f5uVQKNAd"
  ],
	## 网关数据
  "Gateway": {
    "HTTPHeaders": {
      "Access-Control-Allow-Headers": [
        "X-Requested-With",
        "Range"
      ],
      "Access-Control-Allow-Methods": [
        "GET"
      ],
      "Access-Control-Allow-Origin": [
        "*"
      ]
    },
    "RootRedirect": "",
    "Writable": false,
    "PathPrefixes": []
  },
  "API": {
    "HTTPHeaders": {
      "Server": [
        "go-ipfs/0.4.16-dev"
      ]
    }
  },
  "Swarm": {
    "AddrFilters": null,
    "DisableBandwidthMetrics": false,
    "DisableNatPortMap": false,
    "DisableRelay": false,
    "EnableRelayHop": false,
    "ConnMgr": {
      "Type": "basic",
      "LowWater": 600,
      "HighWater": 900,
      "GracePeriod": "20s"
    }
  },
  "Reprovider": {
    "Interval": "12h",
    "Strategy": "all"
  },
  "Experimental": {
    "FilestoreEnabled": false,
    "ShardingEnabled": false,
    "Libp2pStreamMounting": false
  }
}
```



