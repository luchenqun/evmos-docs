---
sidebar_position: 3
---

# 编码

编码是指将数据从一种格式转换为另一种格式，以使其更安全和高效。
在区块链的上下文中，编码用于确保数据以安全且易于访问的方式存储和传输。

递归长度前缀（RLP）是以太坊执行客户端广泛使用的序列化格式。它的目的是对任意嵌套的二进制数据数组进行编码，它是以太坊中用于序列化对象的主要编码方法。RLP仅编码结构，并将编码特定的原子数据类型（如字符串、整数和浮点数）留给高阶协议。

在以太坊中，整数必须以大端二进制形式表示，没有前导零，使得整数值为零等同于空字节数组。RLP编码函数接受一个项作为输入，该项被定义为一个单字节，其值在[0x00, 0x7f]范围内，或者是一个长度为0-55字节的字符串。如果字符串长度超过55字节，则RLP编码由一个值为0xb7（十进制183）的单字节，加上字符串长度的二进制形式的字节长度，后跟字符串长度，再后跟字符串本身。RLP用于哈希验证，其中交易通过对交易数据的RLP哈希进行签名，块通过其头部的RLP哈希进行标识。RLP还用于在网络上传输数据以及某些需要对默克尔树数据结构进行高效编码的情况。以太坊执行层使用RLP作为主要编码方法来序列化对象，但较新的简单序列化（SSZ）取代了RLP作为以太坊2.0中新的共识层的编码方式。

Cosmos Stargate版本引入了protobuf作为客户端和状态序列化的主要编码格式。用于状态和客户端的所有EVM模块类型，如交易消息、创世块、查询服务等，将被实现为协议缓冲区消息。Cosmos SDK还支持传统的Amino编码。协议缓冲区（protobuf）是一种与语言无关的二进制序列化格式，比JSON更小且更快。它用于序列化结构化数据，如消息，并设计为高效且可扩展。编码格式在一种称为协议缓冲区语言（proto3）的与语言无关的语言中定义，并且编码的消息可用于生成各种编程语言的代码。protobuf的主要优势是其高效性，这导致较小的消息大小和更快的序列化和反序列化时间。RLP解码过程如下：根据输入数据的第一个字节（即前缀）和解码数据类型，确定实际数据的长度和偏移量；根据数据的类型和偏移量，相应地解码数据。

## 先决条件阅读

- [Cosmos SDK 编码](https://docs.cosmos.network/main/core/encoding.html)
- [以太坊 RLP](https://eth.wiki/en/fundamentals/rlp)

## 编码格式

### Protocol Buffers

Cosmos [Stargate](https://stargate.cosmos.network/) 发布引入了 [protobuf](https://developers.google.com/protocol-buffers) 作为主要的编码格式，用于客户端和状态序列化。所有用于状态和客户端的 EVM 模块类型（事务消息、创世块、查询服务等）将被实现为协议缓冲区消息。

### Amino

Cosmos SDK 还支持传统的 Amino 编码格式，以向后兼容以前的版本，特别是用于使用 Ledger 设备进行客户端编码和签名。Evmos 不支持 EVM 模块中的 Amino，但对于启用了它的所有其他 Cosmos SDK 模块都支持。

### RLP

递归长度前缀（[RLP](https://eth.wiki/en/fundamentals/rlp)）是一种编码/解码算法，用于序列化消息并允许快速重构编码数据。Evmos 使用 RLP 对以太坊消息进行编码/解码，以便符合正确的以太坊格式，以供 JSON-RPC 处理。这样可以以与以太坊完全相同的格式对消息进行编码和解码。

`x/evm` 事务（`MsgEthereumTx`）的编码是通过将消息转换为 go-ethereum 的 `Transaction`，然后使用 RLP 对事务数据进行编组：

```go
// TxEncoder overwrites sdk.TxEncoder to support MsgEthereumTx
func (g txConfig) TxEncoder() sdk.TxEncoder {
return func(tx sdk.Tx) ([]byte, error) {
  msg, ok := tx.(*evmtypes.MsgEthereumTx)
  if ok {
    return msg.AsTransaction().MarshalBinary()
  }
  return g.TxConfig.TxEncoder()(tx)
}
}

// TxDecoder overwrites sdk.TxDecoder to support MsgEthereumTx
func (g txConfig) TxDecoder() sdk.TxDecoder {
return func(txBytes []byte) (sdk.Tx, error) {
  tx := &ethtypes.Transaction{}

  err := tx.UnmarshalBinary(txBytes)
  if err == nil {
    msg := &evmtypes.MsgEthereumTx{}
    msg.FromEthereumTx(tx)
    return msg, nil
  }

  return g.TxConfig.TxDecoder()(txBytes)
}
}
```


---
sidebar_position: 3
---

# Encoding

Encoding refers to the process of converting data from one format to another to make it more secure and efficient.
In the context of blockchain, encoding is used to ensure that data is stored and transmitted in a way that is secure and
easily accessible.

The Recursive Length Prefix (RLP) is a serialization format used extensively in Ethereum's execution clients. Its purpose
is to encode arbitrarily nested arrays of binary data, and it is the main encoding method used to serialize objects in
Ethereum. RLP only encodes structure and leaves encoding specific atomic data types, such as strings, integers, and floats,
to higher-order protocols.

In Ethereum, integers must be represented in big-endian binary form with no leading zeroes,
making the integer value zero equivalent to the empty byte array. The RLP encoding function takes in an item, which is
defined as a single byte whose value is in the [0x00, 0x7f] range or a string of 0-55 bytes long. If the string is
more than 55 bytes long, the RLP encoding consists of a single byte with value 0xb7 (dec. 183) plus the length in
bytes of the length of the string in binary form, followed by the length of the string, followed by the string. RLP
is used for hash verification, where a transaction is signed by signing the RLP hash of the transaction
data, and blocks are identified by the RLP hash of their header. RLP is also used for encoding data over the wire
and for some cases where there should be support for efficient encoding of the merkle tree data structure. The
Ethereum execution layer uses RLP as the primary encoding method to serialize objects, but the newer Simple
Serialize (SSZ) replaces RLP as the encoding for the new consensus layer in Ethereum 2.0.

The Cosmos Stargate release introduces protobuf as the main encoding format for both client and state serialization.
All the EVM module types that are used for state and clients, such as transaction messages, genesis, query services,
etc., will be implemented as protocol buffer messages. The Cosmos SDK also supports the legacy Amino encoding.
Protocol Buffers (protobuf) is a language-agnostic binary serialization format that is smaller and faster than JSON.
It is used to serialize structured data, such as messages, and is designed to be highly efficient and extensible. The
encoding format is defined in a language-agnostic language called Protocol Buffers Language (proto3), and the encoded
messages can be used to generate code for a variety of programming languages. The main advantage of protobuf is its
efficiency, which results in smaller message sizes and faster serialization and deserialization times. The RLP decoding
process is as follows: according to the first byte (i.e., prefix) of input data and decoding the data type, the length
of the actual data and offset; according to the type and offset of data, decode the data correspondingly.

## Prerequisite Readings

- [Cosmos SDK Encoding](https://docs.cosmos.network/main/core/encoding.html)
- [Ethereum RLP](https://eth.wiki/en/fundamentals/rlp)

## Encoding Formats

### Protocol Buffers

The Cosmos [Stargate](https://stargate.cosmos.network/) release introduces
[protobuf](https://developers.google.com/protocol-buffers) as the main encoding format for both
client and state serialization. All the EVM module types that are used for state and clients
(transaction messages, genesis, query services, etc) will be implemented as protocol buffer messages.

### Amino

The Cosmos SDK also supports the legacy Amino encoding format for backwards compatibility with
previous versions, specially for client encoding and signing with Ledger devices. Evmos does not
support Amino in the EVM module, but it is supported for all other Cosmos SDK modules that enable it.

### RLP

Recursive Length Prefix ([RLP](https://eth.wiki/en/fundamentals/rlp)), is an encoding/decoding algorithm that serializes a message and
allows for quick reconstruction of encoded data. Evmos uses RLP to encode/decode Ethereum
messages for JSON-RPC handling to conform messages to the proper Ethereum format. This allows
messages to be encoded and decoded in the exact format as Ethereum's.

The `x/evm` transactions (`MsgEthereumTx`) encoding is performed by casting the message to a go-ethereum's `Transaction` and then marshaling the transaction data using RLP:

```go
// TxEncoder overwrites sdk.TxEncoder to support MsgEthereumTx
func (g txConfig) TxEncoder() sdk.TxEncoder {
return func(tx sdk.Tx) ([]byte, error) {
  msg, ok := tx.(*evmtypes.MsgEthereumTx)
  if ok {
    return msg.AsTransaction().MarshalBinary()
  }
  return g.TxConfig.TxEncoder()(tx)
}
}

// TxDecoder overwrites sdk.TxDecoder to support MsgEthereumTx
func (g txConfig) TxDecoder() sdk.TxDecoder {
return func(txBytes []byte) (sdk.Tx, error) {
  tx := &ethtypes.Transaction{}

  err := tx.UnmarshalBinary(txBytes)
  if err == nil {
    msg := &evmtypes.MsgEthereumTx{}
    msg.FromEthereumTx(tx)
    return msg, nil
  }

  return g.TxConfig.TxDecoder()(txBytes)
}
}
```
