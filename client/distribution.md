# Cosmos经济模型

---

## 插件模块
* mint: 挖矿模块
* stake：股份管理等
* auth：管理手续费和验证等
* distribution: 分配利息和手续费等
* slashing：处罚模块

### 统计块收益
```seq
mint->auth(feekeeper): AddCollectedFees（记录通胀）
mint->stake(keeper):InflateSupply（通胀入pool里面的looseTokens池）
auth(ante)->auth(feekeeper):AddCollectedFees(记录手续费)
```


`mint`模块挖矿得到的通胀收益和`auth`模块得到每笔手续费都会以累计的放入到一个数据中，等待`distribution`模块来进行分配。

### 分配块收益
```seq
distribution->distribution:AllocateTokens
```
`distribution`模块在每个块开始时都会通过`BeginBlocker()`来调用`keeper:AllocateTokens()`方法来对每个块的收益进行分配，分配完毕后会调用`ClearCollectedFees()`方法来对累计块收益进行清零。

看不到图就[点击这里](https://www.zybuluo.com/elvindu05/note/1364063)
