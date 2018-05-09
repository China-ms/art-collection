---
layout: post
title: 浅入浅出智能合约 - 调用（三）
permalink: /smart-contract-invoke
tags: blockchain ethereum transaction smart-contract solidity
toc: true
desc: 在这篇文章中，我们将简单介绍 Ethereum 智能合约中的应用程序二进制接口（ABI）以及如何使用 ABI 调用其他智能合约中的函数，同时包含函数选择器以及参数编码等话题。
---

+ [浅入浅出智能合约 - 概述（一）](https://draveness.me/smart-contract-intro)
+ [浅入浅出智能合约 - 部署（二）](https://draveness.me/smart-contract-deploy)
+ [浅入浅出智能合约 - 调用（三）](https://draveness.me/smart-contract-invoke)

当我们谈到 Ethereum 的智能合约时，很难不涉及 Solidity 的 [ABI](https://solidity.readthedocs.io/en/develop/abi-spec.html)，这里的 ABI 就是一种与 Ethereum 生态系统中合约交互的标准方法。我们可以使用 ABI 从区块链外部调用合约（DApp）的提供的服务，也可以在合约中调用其他合约的函数。

![ethereum-and-abi](https://img.draveness.me/2018-05-09-ethereum-and-abi.png)

在这篇文章中，我们将简单介绍 Ethereum 智能合约中的*应用程序二进制接口*（ABI）以及如何使用 ABI 调用其他智能合约中的函数。

## ABI

当我们使用 Ethereum ABI 调用智能合约中的某些方法时，我们其实只需要关注 ABI 中的两方面内容，一是函数的选择器，二是参数的编码。

![solidity-smart-contract-abi](https://img.draveness.me/2018-05-09-solidity-smart-contract-abi.png)

前者能够帮助我们选择智能合约中的函数，后者会在解码后作为参数传到函数中；函数的选择加上参数的传递就能在启动智能合约中的分布式应用。

### 函数选择器

Ethereum ABI 中的函数选择器其实就是四个字节的数据，它可以通过如下的方式进行计算：

```ruby
require 'digest/sha3'

Digest::SHA3.hexdigest("baz(uint32,bool)", 256)[0...8] # => "cdcd77c0"
```

其实就是函数签名的 SHA-3 哈希的前四个字节，也可以理解为最左侧的四个字节，函数的签名包含函数名以及所有参数类型用 `,` 连接后的字符串，从这里我们可以看到，ABI 中函数选择器的设计是非常简单的。

### 参数编码

函数选择器由于其本身的特点，只要让其本身遵循一定的约定并不会复杂太多，但是语言中的类型系统相比于函数选择器就是复杂的多。Solidity 作为一门图灵完备的编程语言，它包含两种数据类型，一些是占用固定大小的*静态*类型，另一些是占用动态大小的*动态*类型。

在这里我们简单介绍两几种动态类型的编码方式，数组、字节和字符串；由于变长数组的中元素个数的不确定，所以我们在编码时需要保留其长度信息：

```javascript
encode(arr) = encode(len(arr) encode(arr[0],arr[1],...arr[len(arr)-1])
```

数组中的其他内容将按照元组的方式进行编码，字节和字符串与数组一样，由于长度的不固定，都需要保留长度信息：

```javascript
encode(bytes) = encode(len(bytes)) pad_right(bytes)
encode(string) = encode(encode_utf8(string))
```

字节在编码时需要将内容补充到长度为 32 的倍数，而字符串其实就会按照 UTF8 编码后的字节进行编码；静态类型的编码其实没有太多好介绍的，因为它们的长度固定，所以编码的方式也非常简单，需要注意的是所有的参数在编码后的长度都是 32 的倍数。

> 想要了解更多与函数选择器与类型编码有关的内容，可以直接阅读官方的 [ABI](https://solidity.readthedocs.io/en/develop/abi-spec.html) 文档。

## 调用合约

我们在上一篇文章 [浅入浅出智能合约 - 部署（二）](https://draveness.me/smart-contract-deploy) 中介绍了如何在 Ethereum 上部署合约以及它的实现原理，其实**无论是部署合约还是调用合约都是 Ethereum 网络上的一笔交易**。

```javascript
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "blockHash": "0xfb508342b89066fe2efa45d7dbb9a3ae241486eee66103c03049e2228a159ee8",
        "blockNumber": "0x208c0a",
        "from": "0xe118559d65f87aaa8caa4383b112ff679a21223a",
        "gas": "0x2935a",
        "gasPrice": "0x9502f9000",
        "hash": "0xe74c796a041bad60469f2ee023c87e087847a6603b27972839d0c0de2e852315",
        "input": "0x6080604052348015600f57600080fd5b50603580601d6000396000f3006080604052600080fd00a165627a7a72305820d9b24bc33db482b29de2352889cc2dfeb66029c28b0daf251aad5a5c4788774a0029",
        "nonce": "0x2",
        "to": null,
        "transactionIndex": "0x5",
        "value": "0x0",
        "v": "0x2c",
        "r": "0xa5516d78a7d486d111f818b6b16eef19989ccf46f44981ed119f12d5578022db",
        "s": "0x7125e271468e256c1577b1d7a40d26e2841ff6f0ebcc4da073610ab8d76c19d5"
    }
}
```

如果我们想从自己的地址发送一笔交易，能够主动改变的值并不多，`gasPrice`、`to` 和 `input` 是三个我们能够控制的字段。

其中 `gasPrice` 一般都是一个合理的数值，它仅仅用来表示当前交易愿意承担的『手续费』，无法携带太多的信息，`to` 也没有太多可以操作的空间，当它置空时就表示这是一笔用于创建合约的交易，当它为非空时，往往表示交易的接收方或者智能合约的地址。

最后的 `input` 是能够搞事情的地方了，从理论上来讲我们可以在这里存储任何数据，也就意味着拥有了无限可能；当我们调用合约时，我们通过 `input` 传递函数选择器和编码后的参数，这样节点在解码选择器和参数之后就可以执行合约中的某些函数了。

```javascript
pragma solidity ^0.4.18;

contract Contract {
    uint number;
    
    function Contract() public { }
    
    function add(uint x) public returns (uint) {
        number += x;
        return number;
    }
}
```

当我们将上述合约部署到 Ethereum 网络上之后，就能找到如下的合约 [ae9691592751a80f](https://rinkeby.etherscan.io/address/0xae9691592751a80feccaa87a8898ea39dfb2cd4f)，在这时我们创建一个新的交易调用当前合约并改变其状态。

我们可以看到当前交易 [1f7fda75602f7b71](https://rinkeby.etherscan.io/tx/0x1f7fda75602f7b71fe4fd79cb119d009aad2b89616fb98a93bd241f43f9165cd) 的 `input` 中包含了当前合约调用的函数选择器和编码后的参数。

```javascript
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "blockHash": "0x6cd8ec67ecd8d9cf2cb488521439d92703036f20402a812634c5f6946f0d1ca1",
        "blockNumber": "0x226335",
        "from": "0xe118559d65f87aaa8caa4383b112ff679a21223a",
        "gas": "0x22a2a",
        "gasPrice": "0xee6b2800",
        "hash": "0x1f7fda75602f7b71fe4fd79cb119d009aad2b89616fb98a93bd241f43f9165cd",
        "input": "0x1003e2d2000000000000000000000000000000000000000000000000000000000000000a",
        "nonce": "0x7",
        "to": "0xae9691592751a80feccaa87a8898ea39dfb2cd4f",
        "transactionIndex": "0x3",
        "value": "0x0",
        "v": "0x2b",
        "r": "0x57096309a868e74e06ce62e99667c228dd677ba0ced74f4eb2b3c488a309e420",
        "s": "0x2ee943a546d9dac3835a0c7bc99e5e25aa71e8a9cb5b7dba64941318d99f950"
    }
}
```

`input` 中的数据可以被分割成以下的几个部分，函数选择器和十六进制编码的 `10`：

```javascript
// 0x1003e2d2000000000000000000000000000000000000000000000000000000000000000a

Function: add(uint256 x) ***

MethodID: 0x1003e2d2
- [0]:  000000000000000000000000000000000000000000000000000000000000000a
```

你可以在 Ethereum 的测试网络 Rinkbey 中找到这笔用于调用合约函数的交易 [1f7fda75602f7b71](https://rinkeby.etherscan.io/tx/0x1f7fda75602f7b71fe4fd79cb119d009aad2b89616fb98a93bd241f43f9165cd)。

## 总结

无论是部署合约还是调用合约中函数，我们都需要构建一笔交易并发送到整个网络中，合约的调用使用了 `input` 字段传递函数选择器以及编码后的参数，但是除此之外，我们也可以使用 `input` 做任何意想不到的事情，因为对于 Ethereum 来说 `input` 可能是无意义的，但是对于链外的应用来讲却可以存储有意义的数据。

## 相关文章

+ [浅入浅出智能合约 - 概述（一）](https://draveness.me/smart-contract-intro)
+ [浅入浅出智能合约 - 部署（二）](https://draveness.me/smart-contract-deploy)
+ [浅入浅出智能合约 - 调用（三）](https://draveness.me/smart-contract-invoke)

## Reference

+ [Application Binary Interface Specification](https://solidity.readthedocs.io/en/develop/abi-spec.html)
