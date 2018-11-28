# RPC设计解析

## RPC服务端
在node启动时会调用`node.startRPC()`方法来监听rpc请求，`node.startRPC()`方法同时会实现以下各种服务的注册。`startRPC()`方法会调用如下代码：
```
func RegisterRPCFuncs(mux *http.ServeMux, funcMap map[string]*RPCFunc, cdc *amino.Codec, logger log.Logger) {
	// HTTP endpoints
	for funcName, rpcFunc := range funcMap {
		mux.HandleFunc("/"+funcName, makeHTTPHandler(rpcFunc, cdc, logger))
	}

	// JSONRPC endpoints
	mux.HandleFunc("/", handleInvalidJSONRPCPaths(makeJSONRPCHandler(funcMap, cdc, logger)))
}
```
这样rpc服务可以有两种方式进行访问，一种是jsonrpc的方法访问服务，这种方法uri都是`/`，通过接受post的json数据进行通信，tendermint代码封装的client就是用这种方法。另外一种是通过不同的uri进行访问，方便网页进行访问等。rpc服务可以同时监听两个端口，采用以下不同的两种协议：

* http协议。默认的是`:26657`端口。在这个地址上同时支持两种更具体的协议：
1. 普通http协议。具体注册的方法如下：
    ```
//tendermint/rpc/core/routes.go
var Routes = map[string]*rpc.RPCFunc{
	// info API
	"health":               rpc.NewRPCFunc(Health, ""),
	"status":               rpc.NewRPCFunc(Status, ""),
	"net_info":             rpc.NewRPCFunc(NetInfo, ""),
	"blockchain":           rpc.NewRPCFunc(BlockchainInfo, "minHeight,maxHeight"),
	"genesis":              rpc.NewRPCFunc(Genesis, ""),
	"block":                rpc.NewRPCFunc(Block, "height"),
	"block_results":        rpc.NewRPCFunc(BlockResults, "height"),
	"commit":               rpc.NewRPCFunc(Commit, "height"),
	"tx":                   rpc.NewRPCFunc(Tx, "hash,prove"),
	"tx_search":            rpc.NewRPCFunc(TxSearch, "query,prove,page,per_page"),
	"validators":           rpc.NewRPCFunc(Validators, "height"),
	"dump_consensus_state": rpc.NewRPCFunc(DumpConsensusState, ""),
	"consensus_state":      rpc.NewRPCFunc(ConsensusState, ""),
	"consensus_params":     rpc.NewRPCFunc(ConsensusParams, "height"),
	"unconfirmed_txs":      rpc.NewRPCFunc(UnconfirmedTxs, "limit"),
	"num_unconfirmed_txs":  rpc.NewRPCFunc(NumUnconfirmedTxs, ""),
	// broadcast API
	"broadcast_tx_commit": rpc.NewRPCFunc(BroadcastTxCommit, "tx"),
	"broadcast_tx_sync":   rpc.NewRPCFunc(BroadcastTxSync, "tx"),
	"broadcast_tx_async":  rpc.NewRPCFunc(BroadcastTxAsync, "tx"),
	// abci API
	"abci_query": rpc.NewRPCFunc(ABCIQuery, "path,data,height,prove"),
	"abci_info":  rpc.NewRPCFunc(ABCIInfo, ""),
}

    ```
    配置文件config.toml的`unsafe`字段默认配置为false,设置true时会注册如下api：
    ```
//tendermint/rpc/core/routes.go
func AddUnsafeRoutes() {
	// control API
	Routes["dial_seeds"] = rpc.NewRPCFunc(UnsafeDialSeeds, "seeds")
	Routes["dial_peers"] = rpc.NewRPCFunc(UnsafeDialPeers, "peers,persistent")
	Routes["unsafe_flush_mempool"] = rpc.NewRPCFunc(UnsafeFlushMempool, "")

	// profiler API
	Routes["unsafe_start_cpu_profiler"] = rpc.NewRPCFunc(UnsafeStartCPUProfiler, "filename")
	Routes["unsafe_stop_cpu_profiler"] = rpc.NewRPCFunc(UnsafeStopCPUProfiler, "")
	Routes["unsafe_write_heap_profile"] = rpc.NewRPCFunc(UnsafeWriteHeapProfile, "filename")
}
    ```

2. websocket协议。注册方法如下：
    ```
//tendermint/rpc/core/routes.go
    var Routes = map[string]*rpc.RPCFunc{
	// subscribe/unsubscribe are reserved for websocket events.
	"subscribe":       rpc.NewWSRPCFunc(Subscribe, "query"),
	"unsubscribe":     rpc.NewWSRPCFunc(Unsubscribe, "query"),
	"unsubscribe_all": rpc.NewWSRPCFunc(UnsubscribeAll, ""),
    }
    ```
    访问此服务的uri都是`/websocket`。`rpc/lib/server/handlers.go:WebsocketManager`类对进入的普通http请求进行升级为websocket协议，并处理后续的业务逻辑处理。任何`/websocket`的请求都会启动两个goroutine来处理websocket的读写问题。

* GRPC协议。默认配置是不开启此服务的。注册的服务也只有以下两个：
```
service BroadcastAPI {
  rpc Ping(RequestPing) returns (ResponsePing) ;
  rpc BroadcastTx(RequestBroadcastTx) returns (ResponseBroadcastTx) ;
}
```
实现此服务的包地址是：`tendermint/rpc/grpc`

## RPC客户端
不管客户端是读取信息还是发送交易都需要`client/context`包。`context`包封装了`tendermint/rpc/client`包的`HTTP`结构体：
```
type HTTP struct {
	remote string
	rpc    *rpcclient.JSONRPCClient
	*WSEvents
}
```

`WSEvents`主要结构如下：
```
type WSEvents struct {
	cmn.BaseService
	ws       *rpcclient.WSClient
}
```
`WSEvents`需要调用`start()`方法才能完成websocket的连接。

* `tendermint/rpc/lib/client`包
这个包主要包含三种client：
1. `JSONRPCClient` 通过`PostForm`方法把json结构的数据发送给server来进行通信。
1. `URIClient` 通过访问不同的uri来进行通信。cosmos项目中没有用到。
1. `WSClient` 用来建立websocket连接，实现sub/pub功能。