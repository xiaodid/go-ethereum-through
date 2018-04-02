# BLOCK

## 区块
区块（Block）是以太坊的核心结构体之一，一个个区块以单项链表的形式相互关联起来，就形成了区块链（BlockChain）。
``` go
# /core/types/block.go

// Block represents an entire block in the Ethereum blockchain.
type Block struct {
	header       *Header
	uncles       []*Header
	transactions Transactions  // type Transactions []*Transaction

	// caches
	hash atomic.Value
	size atomic.Value

	// Td is used by package core to store the total difficulty
	// of the chain up to and including the block.
	td *big.Int

	// These fields are used by package eth to track
	// inter-peer block relay.
	ReceivedAt   time.Time
	ReceivedFrom interface{}
}
```

所有账户的相关活动，以交易(Transaction)的格式存储，每个Block有一个交易对象的列表；每个交易的执行结果，由一个Receipt对象与其包含的一组Log对象记录；所有交易执行完后生成的Receipt列表，存储在Block中(经过压缩加密)。不同Block之间，通过前向指针ParentHash一个一个串联起来成为一个单向链表，BlockChain结构体管理着这个链表。

## Header
``` go
# /core/types/block.go

// Header represents a block header in the Ethereum blockchain.
type Header struct {
	ParentHash  common.Hash    `json:"parentHash"       gencodec:"required"`
	UncleHash   common.Hash    `json:"sha3Uncles"       gencodec:"required"`
	Coinbase    common.Address `json:"miner"            gencodec:"required"`
	Root        common.Hash    `json:"stateRoot"        gencodec:"required"`
	TxHash      common.Hash    `json:"transactionsRoot" gencodec:"required"`
	ReceiptHash common.Hash    `json:"receiptsRoot"     gencodec:"required"`
	Bloom       Bloom          `json:"logsBloom"        gencodec:"required"`
	Difficulty  *big.Int       `json:"difficulty"       gencodec:"required"`
	Number      *big.Int       `json:"number"           gencodec:"required"`
	GasLimit    uint64         `json:"gasLimit"         gencodec:"required"`
	GasUsed     uint64         `json:"gasUsed"          gencodec:"required"`
	Time        *big.Int       `json:"timestamp"        gencodec:"required"`
	Extra       []byte         `json:"extraData"        gencodec:"required"`
	MixDigest   common.Hash    `json:"mixHash"          gencodec:"required"`
	Nonce       BlockNonce     `json:"nonce"            gencodec:"required"`
}
```

* ParentHash
指向父区块(parentBlock)的指针。除了创世块(Genesis Block)外，每个区块有且只有一个父区块。

* Coinbase
挖掘出这个区块的作者地址。在每次执行交易时系统会给与一定补偿的Ether，这笔金额就是发给这个地址的。

* UncleHash
Block结构体的成员uncles的RLP哈希值。uncles是一个Header数组，它的存在，颇具匠心。

* Root
StateDB中的“state Trie”的根节点的RLP哈希值。Block中，每个账户以stateObject对象表示，账户以Address为唯一标示，其信息在相关交易(Transaction)的执行中被修改。所有账户对象可以逐个插入一个Merkle-PatricaTrie(MPT)结构里，形成“stateTrie”。

* TxHash
Block中 “tx Trie”的根节点的RLP哈希值。Block的成员变量transactions中所有的tx对象，被逐个插入一个MPT结构，形成“txTrie”。

* ReceiptHash
Block中的 "Receipt Trie”的根节点的RLP哈希值。Block的所有Transaction执行完后会生成一个Receipt数组，这个数组中的所有Receipt被逐个插入一个MPT结构中，形成"ReceiptTrie"。

* Bloom
Bloom过滤器(Filter)，用来快速判断一个参数Log对象是否存在于一组已知的Log集合中。

* Difficulty
区块的难度。Block的Difficulty由共识算法基于parentBlock的Time和Difficulty计算得出，它会应用在区块的‘挖掘’阶段。

* Number
区块的序号。Block的Number等于其父区块Number +1。

* GasLimit
区块内所有Gas消耗的理论上限。该数值在区块创建时设置，与父区块有关。具体来说，根据父区块的GasUsed同GasLimit * 2/3的大小关系来计算得出。

* GasUsed
区块内所有Transaction执行时所实际消耗的Gas总和。

* Time
区块“应该”被创建的时间。由共识算法确定，一般来说，要么等于parentBlock.Time + 10s，要么等于当前系统时间。

* Nonce
一个64bit的哈希数，它被应用在区块的"挖掘"阶段，并且在使用中会被修改。

```Root```, ```TxHash```和```ReceiptHash```，分别取自3个MPT（stateTrie，txTrie和receiptTrie）的跟节点哈希值。
