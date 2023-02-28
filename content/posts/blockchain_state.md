---
title: "Where is your blockchain state"
date: 2023-02-28T10:33:50+08:00
draft: false
tags:
- Blockchain
- Hyperledger fabric
- Ethereum
- Merkle Patricia Trie

---

# `Dapp`状态去哪了

## 世界状态

我们知道区块链每一笔执行成功的交易，背后都对应着世界状态的变化；可以是资产所有权的变更，亦或是账户余额的变化。作为`dapp`开发人员，你可能只关心自己合约的状态，而整个区块链是如何组织和存储世界状态的呢？

## 状态读写抽象

如果写过智能合约，我们会发现绝大多数的区块链底层都提供了最基础的`KV`存储模型。

### `Fabric`中的状态读写接口

如果你开发过`Fabric`的`chaincode`，你肯定使用过下面的接口。非常明确的将状态读写接口暴露给`chaincode`开发人员。

```go=
	// GetState returns the value of the specified `key` from the
	// ledger. Note that GetState doesn't read data from the writeset, which
	// has not been committed to the ledger. In other words, GetState doesn't
	// consider data modified by PutState that has not been committed.
	// If the key does not exist in the state database, (nil, nil) is returned.
	GetState(key string) ([]byte, error)

	// PutState puts the specified `key` and `value` into the transaction's
	// writeset as a data-write proposal. PutState doesn't effect the ledger
	// until the transaction is validated and successfully committed.
	// Simple keys must not be an empty string and must not start with a
	// null character (0x00) in order to avoid range query collisions with
	// composite keys, which internally get prefixed with 0x00 as composite
	// key namespace. In addition, if using CouchDB, keys can only contain
	// valid UTF-8 strings and cannot begin with an underscore ("_").
	PutState(key string, value []byte) error
```

查看底层处理状态读写的代码，我们可以看到底层的存储接口增加了`namespace`的前缀，这个`namespace`实际上对应的是`chaincodeID`。

```go=
func (q *queryExecutor) getState(ns, key string) ([]byte, []byte, error) {
	if err := q.checkDone(); err != nil {
		return nil, nil, err
	}
	versionedValue, err := q.txmgr.db.GetState(ns, key)
	if err != nil {
		return nil, nil, err
	}
	val, metadata, ver := decomposeVersionedValue(versionedValue)
	if q.collectReadset {
		q.rwsetBuilder.AddToReadSet(ns, key, ver)
	}
	return val, metadata, nil
}

// SetState implements method in interface `ledger.TxSimulator`
func (s *txSimulator) SetState(ns string, key string, value []byte) error {
	if err := s.checkWritePrecondition(key, value); err != nil {
		return err
	}
	s.rwsetBuilder.AddToWriteSet(ns, key, value)
	return nil
}
```

### `Solidity`中的状态变量

而你如果使用`Solidity`编写过智能合约，你可能会发现并没有明确的状态读写接口，但是你一定会用到`State Variables`。状态变量的读写在`EVM`的执行过程中会对应到指令码`SLOAD`和`SSTORE`。

```solidity=
contract ExampleContract { 
    uint storedData; // State variable
    event Sent ( msg ); 
    constructor () public {}
    
    function method ( uint x) public { 
        storeData = x;
        emit Sent ('Success');
   }
}
```

查看`hyperledger/burrow`项目实现的`EVM`，我们可以看到实际底层也是类似的`KV`存储。同样的我们发现在底层存储中增加了`params.Callee`这样一个类似的`namespace`，这里对应的其实是合约账户地址。

```go=
case SLOAD: // 0x54
    loc := stack.Pop()
    data := LeftPadWord256(maybe.Bytes(st.CallFrame.GetStorage(params.Callee, loc)))
    stack.Push(data)
    c.debugf("%v {0x%v = 0x%v}\n", params.Callee, loc, data)

case SSTORE: // 0x55
    loc, data := stack.Pop(), stack.Pop()
    maybe.PushError(engine.UseGasNegative(params.Gas, engine.GasStorageUpdate))
	maybe.PushError(st.CallFrame.SetStorage(params.Callee, loc, data.Bytes()))
    c.debugf("%v {%v := %v}\n", params.Callee, loc, data)
```

## 状态数据库

前面我们已经知道了合约中的状态读写接口是怎么和区块链底层关联的，但是状态又是如何持久化到状态数据库中的呢？绝大部分的区块链都是使用的`KV`数据库来作为状态数据库，但是也有一些区块链支持关系数据库和文档数据库等。

我们这里只介绍`KV`状态数据库，下面以`Fabric`和`Go-ethereum`为例子来看看它们是如何处理的。因为`KV`是非常简单的模型，我们只要搞清楚`KEY`和`VALUE`是如何编解码的，我们就能理解整个区块链中的状态管理。

### `Fabric`数据库中的`KEY`和`VALUE`

首先我们需要知道`Fabric`中有`channel`和`chaincode`的概念，每一个`channel`对应于一条独立的区块链，每一个`chaincode`对应于一个独立的智能合约。在内存中`Fabric`会使用多级`Map`的数据结构来组织整个状态数据，而其实底层只使用一个`KV`数据库作为状态数据库。

其中`KEY`的编码是将内存中的层级结构铺平，由`channelID`，`chaincodeID`和用户自己定义的`key`拼接而成。
> channelID + chaincodeID + user_defined_key

而`VALUE`的编码则是和`RWSet`紧密相关，可通过官方文档[Read-Write set semantics](https://hyperledger-fabric.readthedocs.io/en/latest/readwrite.html)章节了解`RWSet`的概念。

根据下面的`proto`数据结构定义，可以看出实际数据库中的`VALUE`除了包含用户自定义的`Value`，还包括了`Version`和`Metadata`两个附加的值，而`Version`的值实际上是`Height`编码后的内容。
```go=
type DBValue struct {
	Version              []byte   `protobuf:"bytes,1,opt,name=version,proto3" json:"version,omitempty"`
	Value                []byte   `protobuf:"bytes,2,opt,name=value,proto3" json:"value,omitempty"`
	Metadata             []byte   `protobuf:"bytes,3,opt,name=metadata,proto3" json:"metadata,omitempty"`
	XXX_NoUnkeyedLiteral struct{} `json:"-"`
	XXX_unrecognized     []byte   `json:"-"`
	XXX_sizecache        int32    `json:"-"`
}

type Height struct {
	BlockNum uint64
	TxNum    uint64
}

```

### `Go-ethereum`数据库中的`KEY`和`VALUE`
如果稍微了解过`Ethereum`，应该知道`Ethereum`中有一个特别的数据结构[PATRICIA MERKLE TREES](https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie/)。`MPT`实际被应用到多个模块，而和状态密切相关的，主要是`World State Trie`和`Storage Trie`。

虽然`PATRICIA MERKLE TREES`有`Branch node`，`Extension node`和`Leaf node`等多种类型(不讨论`NULL`节点)；同时在`Ethereum`中每一个`Address`都对应了一棵`World State Trie`，并且如果`Address`是一个合约地址，还同时对应了一棵`Storage Trie`。但是对于状态数据库中的`KEY`和`VALUE`编码，实际上可以只用一条统一的规则描述。
> 𝑘𝑒𝑐𝑐𝑎𝑘 (𝑅𝐿𝑃 (𝑛𝑜𝑑𝑒)) → 𝑅𝐿𝑃 (𝑛𝑜𝑑𝑒)

`𝑘𝑒𝑐𝑐𝑎𝑘`是`Ethereum`使用的`hash`算法，而[𝑅𝐿𝑃](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/)是`Ethereum`使用的编解码格式。

- Extension 𝑛𝑜𝑑𝑒 ≡ [𝐻𝑃 (𝑝𝑟𝑒𝑓𝑖𝑥 + 𝑝𝑎𝑡ℎ), 𝑘𝑒𝑦] 
- Branch 𝑛𝑜𝑑𝑒 ≡ [𝑏𝑟𝑎𝑛𝑐ℎ𝑒𝑠, 𝑣𝑎𝑙𝑢𝑒] 
- Leaf 𝑛𝑜𝑑𝑒 ≡ [𝐻𝑃 (𝑝𝑟𝑒𝑓𝑖𝑥 + 𝑝𝑎𝑡ℎ), 𝑣𝑎𝑙𝑢𝑒]

`𝐻𝑃`代表的是`Hex Prefix Encoding`，`𝑝𝑟𝑒𝑓𝑖𝑥`指的是用于区分不同的`Extension`和`Leaf`的`node`的标志位。

从上述的规则我们可以看到，`Ethereum`的状态数据库是把所有`MPT`的节点进行了存储，而对应`KEY`是节点编码后的`hash`值，而`VALUE`就是节点编码的实际值。

## 内存中的状态

前面我们说到了面向用户的状态读写抽象和状态是如何被持久化到数据库中的，下面我们来说一说位于两者中间的内存中的状态。

### 全局缓存
类似操作系统或者数据库系统，为了提升效率，区块链会将状态数据库中的数据在内存中进行缓存。全局的缓存通常是一个有序`Map`的数据结构，缓存的内容实际就是数据库中的`KEY`和`VALUE`。

### 区块执行时的状态

区块的执行实际就是执行该区块中的全部交易，交易在执行时都是从当前最新的世界状态中去读取数据，而将对状态的变更先更新到当前执行上下文的内存数据结构中；当全部交易执行完成后，将当前上下文中变更了的状态整个提交到状态数据库和全局缓存中。下面我们还是以`Fabric`和`Go-ethereum`为例子来看看它们分别是怎么处理的。

#### `Fabric`内存中的状态表示

`Fabric`的整体执行流程是`Execute`->`Orderer`->`Validate`方式，因此它的交易内容中的`RWSet`，实际上是该交易执行时读写过的状态，每笔交易执行时读取的状态都是以上一个区块提交后的世界状态为基准，写入的状态则写入到当前交易上下文的`WSet`中。因为所有交易的读取都是以上一个区块提交后的世界状态为基准，所以可能存在状态更新冲突的交易（还是参考[Read-Write set semantics](https://hyperledger-fabric.readthedocs.io/en/latest/readwrite.html)），在`Validate`阶段进行`MVCC`验证过程中，会保证冲突的交易只有一笔能够成功。而验证完后的结果实际上就得到了一个`Batch`对象，保存了所有成功交易变更过的状态的集合。

```go=
// UpdateBatch encapsulates the updates to Public, Private, and Hashed data.
// This is expected to contain a consistent set of updates
type UpdateBatch struct {
	PubUpdates  *PubUpdateBatch
	HashUpdates *HashedUpdateBatch
	PvtUpdates  *PvtUpdateBatch
}

// PubUpdateBatch contains update for the public data
type PubUpdateBatch struct {
	*statedb.UpdateBatch
}

// UpdateBatch encloses the details of multiple `updates`
type UpdateBatch struct {
	ContainsPostOrderWrites bool
	Updates                 map[string]*nsUpdates
}

type nsUpdates struct {
	M map[string]*VersionedValue
}
```
这里我们不讲`HashedUpdateBatch`和`PvtUpdates`(与其内部其他特性有关)，我们可以看到`PubUpdates`实际上就是一个两级的`Map`结构，外层的`KEY`就是`chaincodeID`，内层的`KEY`就是用户在调用`PutState`方法传入的`KEY`。这里我们没有看到`channelID`这个维度，是因为每个`channel`实际上都是独立的区块链，所以一个区块中是不可能包含其他`channel`的状态变更的，在最后`Batch`写入状态数据库的时候会统一的把这个区块所在的`channelID`拼接上。

#### `Go-ethereum`内存中的状态表示

###### `MERKLE PATRICIA TRIE`简介
在`MPT`中有三种`node`类型(不考虑`NULL`节点)
- Branch – a 17-item node [𝑖0,𝑖1, ...,𝑖15, 𝑣𝑎𝑙𝑢𝑒] 
- Extension – A 2-item node [𝑝𝑎𝑡ℎ, 𝑘𝑒𝑦]
- Leaf – A 2-item node [𝑝𝑎𝑡ℎ, 𝑣𝑎𝑙𝑢𝑒]

> The Branch node is used where branching takes place, i.e. keys at certain character position starts to differ. First 16
items are used for branching, which means that this node allows for 16 possible branches for 0 ′ 𝐹 hex character. This
concept is familiar from the Memory Trie structures, where branching was constructed as a column in a table. The
𝑖∀position contains a link to a child node whenever the child exists, i.e. this position corresponds with a next character
(hex 0 ′ 𝐹 ) in a key. The 17th item stores a value and it is used only if this node is terminating (for a certain key).

我们可以看到`Branch`节点的前16个`item`会存储`child node`的指针（实际就是`child node`的𝑘𝑒𝑐𝑐𝑎𝑘 (𝑅𝐿𝑃 (𝑛𝑜𝑑𝑒))），所以我们可以通过这个指针从状态数据库或者是全局缓存中加载并解码得到`child node`对象，以此类推我们可以在内存中逐步构建出`MPT`（实际上我们只会加载本次区块执行过程中所有交易读写过的节点）。而最后一个`item`是用来存储`value`的，这个`value`存储的不是一个节点指针而是一个实际的值。`value`可以看后面一个例子。

> The Extension node is where the compression takes place. Whenever there is a part common for multiple keys, this
part is stored in this node. In other words, it prevents descending several times should it follow only one path. This
node, in other words, resembles a Patricia feature of number of bits to skip.

`Extension`的作用非常明显，就是用于减少`path`上公共前缀的存储，压缩`path`。其中`path item`存储的是公共前缀，`value`是存储的是一个节点指针，同样可以根据这个指针加载出新的节点。

> The Leaf node terminates the path in the tree. It also uses a compression as it groups common suffixes for keys in
the 𝑝𝑎𝑡ℎ, which is a concept re-used again from Memory Trie.

`Leaf`用于存储实际的值。

从下面这个例子可以直观的看到`Branch node`中`value`存储的是`KEY`正好匹配到`EXTENSION`公共前缀对应的值。同时`Leaf node`中的前缀`10`代表的`path`是偶数长度的叶子节点类型（关于前缀的部分可以参考[Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)中
`Appendix C. Hex-Prefix Encoding`部分）。
![](https://i.imgur.com/Vmi1dRM.png)

###### `MPT`的构建
`Go-ethereum`内存中的状态表示其实之前已经提到了，就是`World State Trie`和`Storage Trie`。`Go-ethereum`中的状态变量和状态数据库中存储的`KEY`和`VALUE`看起来似乎没有明显的关联，用户不需要自己定义`KEY`，那状态变量的值是怎么跑到`Storage Trie`的节点里的呢？

我们从下面这张整体的图中，我们可以看到`Storage Trie`的`root`是存储在对应的`World State Trie`的`node`里的，而`World State Trie`的`root`是存储在对应的`Block header`里的。
![](https://i.imgur.com/ctszYuo.png)

当我们执行一个新的区块时，我们需要从上一个区块的`Block header`中取出最新的`World State Trie`的`root`，这个`root`实际上就是`𝑘𝑒𝑐𝑐𝑎𝑘 (𝑅𝐿𝑃 (𝑛𝑜𝑑𝑒))`，这样我们就可以从状态数据库或者是全局缓存中加载出`World State Trie`的`𝑅𝐿𝑃 (𝑛𝑜𝑑𝑒)`，解码后我们就得到了实际的`root node`对象。以此类推，我们就可以从`root node`开始，通过节点中存储的`child node`指针，加载出需要用到的`World State Trie`节点。因为`World State Trie`的`Leaf node`中存储的`value`实际上就是`Go-ethereum`中的`Account`对象，当这个`Account`对象对应的是一个合约的时候，它的`storageRoot`字段存储的就是`Storage Trie`的`root`。同理，我们就可以加载出用到的`Storage Trie`节点。

###### `World State Trie`中的`KEY`和`VALUE`
`World State Trie`中的`KEY`和`VALUE`可以描述为
> 𝑘𝑒𝑐𝑐𝑎𝑘 (𝑎𝑑𝑑𝑟𝑒𝑠𝑠) → 𝑅𝐿𝑃 (𝐴)

`KEY`的值是20字节的`Address`生成32字节的哈希，然后经过`HEX`编码后，映射到`World State Trie`的路径。而`Leaf`节点中存储的值是`𝑅𝐿𝑃 (𝐴)`，这个`A`就是我们前面提到的`Account`对象。

`Account`对象中包含如下属性
- nonce 
是一个自增的整数，对于外部账号，这个值代表从这个地址发出过的交易数；而对于合约账号，这个值代表合约的创建次数。
- balance
代表账户中的余额。
- storageRoot
外部账户这个值为空，合约账号这个值存储了`Storage Trie`的根节点哈希。
- codeHash
外部账户这个值为空，合约账号这个值对应合约源代码的哈希。


###### `Storage Trie`中的`KEY`和`VALUE`
`Storage Trie`的`KEY`和`VALUE`是最复杂的，因为这里和`Solidity`的数据类型布局紧密相关，用一个通用的方式描述如下
> 𝑘𝑒𝑐𝑐𝑎𝑘 (𝑖𝑛𝑑𝑒𝑥) → 𝑅𝐿𝑃 (𝑠𝑙𝑜𝑡)

`Storage Trie`中的`Leaf`节点存储的值就是`slot`，这个`slot`实际上就是一个长度32的字节数组，而`index`就是用于定位这个`slot`的值（可能就是`slot`的序号，也可能是(序号+key)）。`slot`的序号就是根据我们`Solidity`代码中定义的`State Variables`的顺序产生。

`KEY`的实际值就是`index`的哈希，然后经过`HEX`编码后，我们就能将`KEY`映射成`Storage Trie`的`path`，然后将`slot`存储到`Storage Trie`的`Leaf`节点中。

下面看看不同类型的`slot`的例子，更详细的描述请参考`Solidity`官方文档[layout_in_storage](https://docs.soliditylang.org/en/v0.8.17/internals/layout_in_storage.html)章节。

- Statically Sized Variables
![](https://i.imgur.com/4PgJlD4.png)

定长类型的变量会根据类型长度，多个变量共享或者单个变量独占一个`slot`。

- Maps
![](https://i.imgur.com/Zvl7wiT.png)

`Map`的`index`包含了用户`key`，从这个布局中我们就能看到实际上并没有存储用户`key`的原值，同时也没有存储`Map`的`size`，所有我们在`Solidity`中不能直接迭代`mapping`。

- Dynamic Arrays
![](https://i.imgur.com/igt0gMm.png)

动态数组的`index`包含了数组中的位置，同时还存储了数组的`size`。

- Byte Arrays and String
![](https://i.imgur.com/aBB0JJL.png)

对于字节数组和字符串，如果长度小于32，就用一个`slot`存储，同时最后一个字节存储实际长度。如果长度更长，则使用和动态数组一样的方式存储。

###### `MPT`的提交
在区块所有交易执行完成后，所有成功执行的交易导致变化过的`MPT`节点，最后会统一更新到状态数据库和全局缓存。根据`Ethereum`数据库中的`KEY`和`VALUE`的内容，我们知道了对于底层数据库来说实际只会新插入数据（状态膨胀问题）。同时我们还要注意，因为`Leaf`中的值改变了，导致`Leaf`节点内容变化了，`Leaf`节点的`hash`也变化了，意味着`Branch`节点中的节点指针也变化了，所以`Branch`节点的内容实际也变化了，以此类推，一直会得到一个变化了的`root`。这个正是`MPT`结构中`Merkle tree`的体现，利用这个性质，我们可以通过`Merkle proof`来证明`MPT`节点的存在性。所以`MPT`是一种`Authenticated Storage`，对于拥有`Light client`的链来说，是一种必要的属性。

## 结语
随着`Fabric`和`Go-ethereum`的不断更新迭代，上述内容可能会过时，但是不妨碍我们了解到区块链中对于状态管理的整体的设计思路。实际上针对状态膨胀问题`Go-ethereum`已经在从`hash-based state scheme`切换到[`path-based state scheme`](https://github.com/ethereum/go-ethereum/issues/25390)，而另一个数据结构[Verkle trie](https://blog.ethereum.org/2021/12/02/verkle-tree-structure)将很可能替换`MPT`用于解决`proof size`过大的问题。有兴趣的朋友可以自行了解。

纸上得来终觉浅，我创建了一个项目[verkle](https://github.com/GrapeBaBa/verkle)用来学习`verkle trie`，主要参考[go-verkle](https://github.com/gballet/go-verkle)。目前代码只实现了很少的一部分，有兴趣的朋友可以一起来动动手。

