# ABCI APP

ABCI（Application Blockchain Interface）应用区块链接口是Cosmos团队定义的一套协议，实现这个协议的定制化应用程序就可以和Tendermint共识引擎进行通信，从而获取Tendermint引擎的能力（共识和P2P等），从而很方便的开发出符合自己业务逻辑的定制化区块链。Cosmos-SDK实现了ABCI接口的同时，也包含了一个公有区块链所必有的能力，比如：链上治理，权益委托，转账等。这些能力都是以插件的方法来实现的。
ABCI APP是基于Cosmos-SDK构建的应用程序。此应用程序必须实现如下接口：
```
//tendermint/tendermint/abci/types/application.go
type Application interface {
	// Info/Query Connection
	Info(RequestInfo) ResponseInfo                // Return application info
	SetOption(RequestSetOption) ResponseSetOption // Set application option
	Query(RequestQuery) ResponseQuery             // Query for state

	// Mempool Connection
	CheckTx(tx []byte) ResponseCheckTx // Validate a tx for the mempool

	// Consensus Connection
	InitChain(RequestInitChain) ResponseInitChain    // Initialize blockchain with validators and other info from TendermintCore
	BeginBlock(RequestBeginBlock) ResponseBeginBlock // Signals the beginning of a block
	DeliverTx(tx []byte) ResponseDeliverTx           // Deliver a tx for full processing
	EndBlock(RequestEndBlock) ResponseEndBlock       // Signals the end of a block, returns changes to the validator set
	Commit() ResponseCommit                          // Commit the state and return the application Merkle root hash
}
```
现实了ABCI协议的应用程序必须和Tendermint一起才能构成一个区块链的节点。ABCI APP和Tendermint合作的方式有两种：

1. ABCI APP和Tendermint单独启动，两个独立进程通过ABCI协议来进行通信，ABCI的传输协议可以选择socket或者grpc。

2. ABCI APP和Tendermint作为一个进程来运行。当Tendermint有数据需要和ABCI APP通信时，直接调用函数的方式进行通知。

当然，不管是上述哪一种通信方式，返回值的数据类型都是按照grpc定义的数据结构来处理的。

## tendermint/abci包目录解释
* client 
> ABCI接口实现。Tendermint有数据需要通知ABCI APP时调用此包，包已经封装好了ABCI的几个协议接口，会把需要的参数封装成符合grpc规范的参数传递给server端，local_client则是直接调用ABCI APP的包方法，不需要对入参进行封装。
* server
> ABCI APP作为一个独立进程运行时，需要调用此包，如果和tendermint一起作为一个进行运行的情况下，则不会使用此包。

* types 
> ABCI 接口定义，ABCI参数封装，GRPC接口等。

* examples
> ABCI App的一个简单实现，需要cmd包调用才能作为一个独立进程进行运行。

* cmd
> 调用examples/kvstore包作为一个ABCI APP独立进程运行，需要单独启动tendermint作为一个独立程序与之通信才能组成完整的区块链节点。
