# amino编解码，加解密


---

## amino编解码

amino编解码可以称之为protobuf3的一次升级，再protobuf3的基础上添加了对`interface`的直接支持，也就说：可以直接把序列化好的数据unmarshal到一个接口对象中，但是这需要把定义的`interface`和实现`interface`的对象事前注册到amino的编解码对象中。具体例子如下：
```
cdc := amino.NewCodec()
cdc.RegisterInterface((*MyInterface1)(nil), nil)
cdc.RegisterInterface((*MyInterface2)(nil), nil)
cdc.RegisterConcrete(MyStruct1{}, "com.tendermint/MyStruct1", nil)
cdc.RegisterConcrete(MyStruct2{}, "com.tendermint/MyStruct2", nil)
cdc.RegisterConcrete(&MyStruct3{}, "anythingcangoinhereifitsunique", nil)
```
请注意，注册接口时要注册接口的指针，实现接口对象的注册要注意的是实现方法的接受对象是指针还是对象本身。

注册接口实现对象时需要提供一个名字。这个名字需要全局唯一，为了防止大量注册对象时的碰撞问题，会对这个名字进行一定的运算，从而计算出前缀，前缀一般是4个字节，如果为了取得更好的防碰撞性，也可以在4个字节的前面加上3个字节的消岐字节，前缀字节和消岐字节都不能是0，具体的算法如下：
对注册时的名字进行sha256计算：
```
> hash := sha256("com.tendermint.consensus/MyConcreteName")
> hex.EncodeBytes(hash) // 0x{00 00 A8 FC 54 00 00 00 BB 9C 83 DD ...} (example)
```
去掉0
```
> rest = dropLeadingZeroBytes(hash) // 0x{A8 FC 54 00 00 00 BB 9C 83 DD ...}
> disamb = rest[0:3]
> rest = dropLeadingZeroBytes(rest[3:])
> prefix = rest[0:4]
```

结果如下：
```
> <0xA8 0xFC 0x54> [0xBB 0x9C 9x83 9xDD] // <Disamb Bytes> and [Prefix Bytes]

```
详细标准请参考：[amino spec](https://github.com/tendermint/go-amino#amino-encoding-for-go)

## cosmos项目加解密
Ed25519，Secp256k1都称之为椭圆加密算法，只是他们采用的参数不一样，比特币和以太坊都是采用了Secp256k1算法，而Cosmos项目同时采用了这两种算法，验证者的consensus签名采用的是Ed25519非对称加密，用户交易的签名采用的是Secp256k1非对称加密。

在cosmos项目中定义了两个接口：
```
//tendermint/crypto/crypto.go
type PubKey interface {
	Address() Address
	Bytes() []byte
	VerifyBytes(msg []byte, sig []byte) bool
	Equals(PubKey) bool
}

type PrivKey interface {
	Bytes() []byte
	Sign(msg []byte) ([]byte, error)
	PubKey() PubKey
	Equals(PrivKey) bool
}
```
不管Ed25519还是Secp256k1的公私钥都实现以上两个接口，通过一下代码进行注册，可以对公私钥进行很方便的序列化。
```
//tendermint/crypto/encoding/amino/amino.go
// RegisterAmino registers all crypto related types in the given (amino) codec.
func RegisterAmino(cdc *amino.Codec) {
	// These are all written here instead of
	cdc.RegisterInterface((*crypto.PubKey)(nil), nil)
	cdc.RegisterConcrete(ed25519.PubKeyEd25519{},
		"tendermint/PubKeyEd25519", nil)
	cdc.RegisterConcrete(secp256k1.PubKeySecp256k1{},
		"tendermint/PubKeySecp256k1", nil)

	cdc.RegisterInterface((*crypto.PrivKey)(nil), nil)
	cdc.RegisterConcrete(ed25519.PrivKeyEd25519{},
		"tendermint/PrivKeyEd25519", nil)
	cdc.RegisterConcrete(secp256k1.PrivKeySecp256k1{},
		"tendermint/PrivKeySecp256k1", nil)
}
```
按照amino的规范，对注册名字进行sha256，并截取[3:4]可以得到各个公私钥的前缀，也可以用以下快捷表进行换算。

| Type | Name | Prefix | Length | Notes |
| :-   | :-   | :-     | :-     | :-    |
|PubKeyEd25519|	tendermint/PubKeyEd25519| 0x1624DE64| 0x20|	
|PubKeySecp256k1|tendermint/PubKeySecp256k1	| 0xEB5AE987| 0x21|	
|PrivKeyEd25519|tendermint/PrivKeyEd25519| 0xA3288910| 0x40|	
|PrivKeySecp256k1|tendermint/PrivKeySecp256k1|0xE1B0F79B| 0x20|	

```
例如：
经过amino编码的33个字节（21字节16进制）Secp256k1公钥：
020BD40F225A57ED383B440CF073BC5539D0341F5767D2BF2D78406D00475A2EE9
按照以上表格进行编码得到：
EB5AE98721020BD40F225A57ED383B440CF073BC5539D0341F5767D2BF2D78406D00475A2EE9
```
我们可以从前缀轻易的看出到底是公钥还是私钥，也可以很轻易的辨认出是那种非对称加密。

### 地址
* Ed25519地址算法
`address = SHA256(pubkey)[:20]`

* Secp256k1地址算法
`address = RIPEMD160(SHA256(pubkey))`
和比特币一样。`RIPEMD160`也是一种hash算法。

## bech32
为了增加地址的鲁棒性，可以更好的进行正确性检查，cosmos采用bech32格式来表示地址和公钥，当然，程序内部很多地方还是使用16进制编码或者base64编码来表示的。bech32的前缀我们称之为：human readable part(HRP)，以下表格详细解释了HRP所表示的意思。

|HRP |	Definition |
|:- | :- |
|cosmos |	Cosmos 账户地址，本地数据库|
|cosmospub|	Cosmos 账户公钥，本地数据库|
|cosmosvalcons|	Cosmos 验证者共识地址，也就是来自于priv_validator.json文件|
|cosmosvalconspub	|Cosmos 验证者共识公钥，也就是来自于priv_validator.json文件|
|cosmosvaloper|	Bond验证者共识地址的账户地址|
|cosmosvaloperpub|	Bond验证者共识地址的账户公钥|


