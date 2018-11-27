# RPC设计解析

在node启动时会调用`node.startRPC()`方法来监听rpc请求，`node.startRPC()`方法同时会实现以下各种服务的注册。rpc服务可以同时监听两个端口，采用以下不同的两种协议：

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

2. websocket协议。具体注册的方法如下：
    ```
//tendermint/rpc/core/routes.go
    var Routes = map[string]*rpc.RPCFunc{
	// subscribe/unsubscribe are reserved for websocket events.
	"subscribe":       rpc.NewWSRPCFunc(Subscribe, "query"),
	"unsubscribe":     rpc.NewWSRPCFunc(Unsubscribe, "query"),
	"unsubscribe_all": rpc.NewWSRPCFunc(UnsubscribeAll, ""),
    }
    ```
    访问此服务的uri都是`/websocket`。`rpc/lib/server/handlers.go:WebsocketManager`类对进入的普通http请求进行升级为websocket协议，并处理后续的业务逻辑处理。

* GRPC协议。默认配置是不开启此服务的。注册的服务也只有以下两个：
```
service BroadcastAPI {
  rpc Ping(RequestPing) returns (ResponsePing) ;
  rpc BroadcastTx(RequestBroadcastTx) returns (ResponseBroadcastTx) ;
}
```
实现此服务的包地址是：`tendermint/rpc/grpc`