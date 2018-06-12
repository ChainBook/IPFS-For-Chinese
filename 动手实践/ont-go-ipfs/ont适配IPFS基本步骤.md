[TOC]

## IPFS对数据插件的处理流程

1. daemon启动的时候

   ```flow
   st=>start: Start
   op0=>operation: main.go/main()/mainRet()
   op1=>operation: mainRet()/cli.Run()/makeExecutor()
   op2=>operation: loader.LoadPlugins()/run()
   op3=>operation: runIPLDPlugin()：此步骤会完成对区块解析接口的注册
   e=>end: End
   st->op0->op1->op2->op3->e
   ```

2. put数据的时候

   ```flow
   st=>start: Start
   op0=>operation: core/commands/dag/dag.go/DagPutCmd/Run:参数实现
   op1=>operation: Run:/addAllAndPin()/ParseInputs()根据参数，解析输入文件
   op2=>operation: Batch.Add() & cids.Add() & Batch.Commit()
   e=>end: End
   st->op0->op1->op2->e
   ```

   

## 实现过程

1. 实现如下插件接口，位于文件：`go-ipfs/plugin/ipld.go`

   ```go
   type Plugin interface {
   	Name() string
   	Version() string
   	Init() error
   }
   type PluginIPLD interface {
   	Plugin
   	RegisterBlockDecoders(dec ipld.BlockDecoder) error
   	RegisterInputEncParsers(iec coredag.InputEncParsers) error
   }
   ```

   

2. 实现如下交易及区块接口，接口文件位于：`go-block-format/blocks.go`

   ```go
   // Block provides abstraction for blocks implementations.
   type Block interface {
   	RawData() []byte
   	Cid() *cid.Cid
   	String() string
   	Loggable() map[string]interface{}
   }
   ```



## 编译运行过程

```
1. 在$IPFS_PATH/go-ipfs/plugin/plugins下，增加ont适配插件
2. 修改$IPFS_PATH/go-ipfs/plugins/Rules.mk
3. 在$IPFS_PATH 下执行 make build_plugins，此时会在plugins目录下生成 .so 文件
4. 将 .so 文件存到 ./ipfs/plugins/目录下
```



## 调用过程

1. 命令行

   ```shell
   cat ontology-block.json | ipfs put --input-enc json --format ont-block
   ```

   

2. api接口调用

   ```
   - 下载 ipfs/go-ipfs-api 仓库
   - 使用 shell.DagPut()及shell.DagGet()接口
   ```

   

