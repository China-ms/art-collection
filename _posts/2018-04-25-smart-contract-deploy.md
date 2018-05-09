---
layout: post
title: 浅入浅出智能合约 - 部署（二）
permalink: /smart-contract-deploy
tags: blockchain ethereum transaction smart-contract solidity
toc: true
desc: 在这篇文章中我们将要介绍智能合约在编写之后是如何部署到 Ethereum 网络的。部署一个新的智能合约或者说 DApp 其实总共只需要两个步骤，首先要将已经编写好的合约代码编译成二进制代码，然后将二进制数据和构造参数打包成交易发送到网络中，等待当前交易被矿工追加到区块链就可以了。
---

+ [浅入浅出智能合约 - 概述（一）](https://draveness.me/smart-contract-intro)
+ [浅入浅出智能合约 - 部署（二）](https://draveness.me/smart-contract-deploy)
+ [浅入浅出智能合约 - 调用（三）](https://draveness.me/smart-contract-invoke)

上一篇文章 [浅入浅出智能合约 - 概述（一）](https://draveness.me/smart-contract-intro) 介绍了智能合约中的一些基本概念以及面向合约的编程语言 Solidity，在这篇文章中我们将要介绍智能合约在编写之后是如何部署到 Ethereum 网络的。

![ethereum-deploy-smart-contract](https://img.draveness.me/2018-04-25-ethereum-deploy-smart-contract.png)

部署一个新的智能合约或者说 DApp 其实总共只需要两个步骤，首先要将已经编写好的合约代码编译成二进制代码，然后将二进制数据和构造参数打包成交易发送到网络中，等待当前交易被矿工追加到区块链就可以了。

## 编译

合约代码的编译过程非常简单，我们使用如下的合约代码为例，简单介绍合约的编译过程：

```javascript
pragma solidity ^0.4.22;

contract Contract {
    constructor() public {
    }
}
```

编译 Solidity 代码需要 [solidity](https://github.com/ethereum/solidity) 编译器参与工作，编译器的使用也非常简单，我们可以直接使用如下的命令将上述合约编译成二进制：

```bash
$ solc --bin contract.sol

======= contract.sol:Contract =======
Binary:
6080604052348015600f57600080fd5b50603580601d6000396000f3006080604052600080fd00a165627a7a72305820d9b24bc33db482b29de2352889cc2dfeb66029c28b0daf251aad5a5c4788774a0029
```

如果我们使用了 Ethereum Wallet 等客户端，就可以将上述二进制数据添加到如下图所示的 DATA 中：

![send-transaction](https://img.draveness.me/2018-04-25-send-transaction.png)

用于创建合约的交易不需要填写目标地址需要在 DATA 中填写合约的二进制数据和编码后的构造器二进制参数，由于这个合约的构造器并不包含任何参数，所以我们只需要添加合约的二进制数据，点击发送后会生成如下的交易 [e74c796a041bad60469f2ee023c87e08](http://rinkeby.etherscan.io/tx/0xe74c796a041bad60469f2ee023c87e087847a6603b27972839d0c0de2e852315)：

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

在这个用于创建合约的特殊交易中，我们可以看到目标地址 `to` 的值为空，`input` 的值就是我们在 Ethereum Wallet 中发送交易时填写的 DATA，即合约的二进制代码。这笔交易被纳入区块链之后，我们就能在 [Etherscan](http://rinkeby.etherscan.io/tx/0xe74c796a041bad60469f2ee023c87e087847a6603b27972839d0c0de2e852315) 上看到这笔交易成功的创建了一个合约 [0xa6a158a131476d4e071f4a3a0d9af2d88769b25a](https://rinkeby.etherscan.io/address/0xa6a158a131476d4e071f4a3a0d9af2d88769b25a)。

## 发送

从上面的测试已经可以看到合约都是由交易（Transaction）创建的，每一个创建合约的交易的 `to` 字段都是 `null`，而 `input` 是合约代码编译之后的二进制；经历了编译这一过程，剩下的就是交易的打包、签名和发送了。

在这里，我们可以通过阅读一个经典的 Ethereum 实现 [parity](https://github.com/paritytech/parity) 的源代码来研究交易是如何打包、签名和发送的；从源代码中，我们可以找到签名交易的入口 `sign_transaction` 函数：

```rust
// parity/rpc/src/v1/traits/eth_signing.rs
#[rpc(meta, name = "eth_signTransaction")]
fn sign_transaction(&self, Self::Metadata, TransactionRequest) -> BoxFuture<RichRawTransaction>;

// parity/rpc/src/v1/impls/signing.rs
fn sign_transaction(&self, meta: Metadata, request: RpcTransactionRequest) -> BoxFuture<RpcRichRawTransaction> {
  let res = self.dispatch(
    RpcConfirmationPayload::SignTransaction(request),
    meta.dapp_id().into(),
    meta.origin,
  );

  Box::new(res.flatten().and_then(move |response| {
    match response {
      RpcConfirmationResponse::SignTransaction(tx) => Ok(tx),
      e => Err(errors::internal("Unexpected result.", e)),
    }
  }))
}
```

上述函数初始化了一个 `RpcConfirmationPayload::SignTransaction` 结构体并在最后调用 `fill_optional_fields` 将请求中的参数 `from`、`to`、`nonce`、`gas_price`、`gas`、`value`、`data` 以及 `condition` 等数据填入到最终被签名的交易中：

```rust
// rpc/src/v1/helpers/dispatch.rs
fn fill_optional_fields(&self, request: TransactionRequest, default_sender: Address, force_nonce: bool)
                        -> BoxFuture<FilledTransactionRequest>
{
  let request = request;
  let from = request.from.unwrap_or(default_sender);
  let nonce = if force_nonce {
    request.nonce.or_else(|| Some(self.state_nonce(&from)))
  } else {
    request.nonce
  };

  Box::new(future::ok(FilledTransactionRequest {
    from,
    used_default_from: request.from.is_none(),
    to: request.to,
    nonce,
    gas_price: request.gas_price.unwrap_or_else(|| {
      default_gas_price(&*self.client, &*self.miner, self.gas_price_percentile)
    }),
    gas: request.gas.unwrap_or_else(|| self.miner.sensible_gas_limit()),
    value: request.value.unwrap_or_else(|| 0.into()),
    data: request.data.unwrap_or_else(Vec::new),
    condition: request.condition,
  }))
}
```

经过内部的两次 RPC 请求 `SignTransaction` 和 `SignMessage` 最后会由 `signature.rs` 文件中的 `sign` 函数使用 secp256k1 完成对传入数据的签名：

```rust
// ethkey/src/signature.rs
pub fn sign(secret: &Secret, message: &Message) -> Result<Signature, Error> {
	let context = &SECP256K1;
	let sec = SecretKey::from_slice(context, &secret)?;
	let s = context.sign_recoverable(&SecpMessage::from_slice(&message[..])?, &sec)?;
	let (rec_id, data) = s.serialize_compact(context);
	let mut data_arr = [0; 65];

	// no need to check if s is low, it always is
	data_arr[0..64].copy_from_slice(&data[0..64]);
	data_arr[64] = rec_id.to_i32() as u8;
	Ok(Signature(data_arr))
}
```

`eth_signTransaction` 的作用其实只有两部分，一部分是将传入的参数组合成一个交易，另一部分是通过 secp256k1 对交易进行签名：

![compose-sign-transaction](https://img.draveness.me/2018-04-25-compose-sign-transaction.png)

签名好的二进制交易可以通过 `eth_sendRawTransaction` 广播到整个 Ethereum 网络，而 JSON 格式的交易可以通过 `eth_sendTransaction` 发送：

```rust
// parity/rpc/src/v1/traits/eth.rs
#[rpc(name = "eth_sendRawTransaction")]
fn send_raw_transaction(&self, Bytes) -> Result<H256>;

// parity/rpc/src/v1/impls/eth.rs
fn send_raw_transaction(&self, raw: Bytes) -> Result<RpcH256> {
	Rlp::new(&raw.into_vec()).as_val()
		.map_err(errors::rlp)
		.and_then(|tx| SignedTransaction::new(tx).map_err(errors::transaction))
		.and_then(|signed_transaction| {
			FullDispatcher::dispatch_transaction(
				&*self.client,
				&*self.miner,
				signed_transaction.into(),
			)
		})
		.map(Into::into)
}
```

`send_raw_transaction` 最终会将签名好的交易加入到当前节点的交易队列等待处理，交易队列中的交易随后会广播到整个网络中并被整个网络确认并纳入新的区块中。

```rust
fn import_own_transaction<C: miner::BlockChainClient>(
	&self,
	chain: &C,
	pending: PendingTransaction,
) -> Result<(), transaction::Error> {
	let client = self.pool_client(chain);
	let imported = self.transaction_queue.import(
		client,
		vec![pool::verifier::Transaction::Local(pending)]
	).pop().expect("one result returned per added transaction; one added => one result; qed");

	if imported.is_ok() && self.options.reseal_on_own_tx && self.sealing.lock().reseal_allowed() {
		if self.engine.seals_internally().unwrap_or(false) || !self.prepare_pending_block(chain) {
			self.update_sealing(chain);
		}
	}

	imported
}
```

当交易成为新区块的一部分之后，我们就能通过 Etherscan 或者其他方式查看被创建合约 [0xa6a158a131476d4e071f4a3a0d9af2d88769b25a](https://rinkeby.etherscan.io/address/0xa6a158a131476d4e071f4a3a0d9af2d88769b25a) 的信息了。

## 总结

在 Ethereum 上部署合约的过程其实与交易发送的过程基本完全相似，唯一的区别就是用于创建合约的交易目前地址为空，并且 `data` 字段中的内容就是合约的二进制代码，也就是合约的部署由两部分组成：编译合约和发送消息。对于合约的部署这里差不多介绍完了，在下一篇文章中，我们将分析如何调用智能合约中声明的函数。

## 相关文章

+ [浅入浅出智能合约 - 概述（一）](https://draveness.me/smart-contract-intro)
+ [浅入浅出智能合约 - 部署（二）](https://draveness.me/smart-contract-deploy)
+ [浅入浅出智能合约 - 调用（三）](https://draveness.me/smart-contract-invoke)

