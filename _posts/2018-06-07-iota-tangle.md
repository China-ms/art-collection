---
layout: post
title: 物联网与『高效的』IOTA
permalink: /iota-tangle
tags: 聊聊区块链 blockchain iota tangle POW
toc: true
desc: 这一次我们介绍的区块链项目就是 IOTA，它的团队将 IOTA 定义为『次时代的无许可』分布式账本，我们先来看一下它解决了什么问题，再来讨论它的价值。IOTA 使用了基于有向无环图（DAG） 设计的 Tangle，有别于传统的区块链项目，在 IOTA 或者说 Tangle 中，没有区块和链的概念，同时也没有矿工和用户之间的转账并且交易也不收取手续费。Tangle 本质上就是一个有向无环图，所有由节点发出的交易最后都会成为图的一部分，也就是用于存储交易的分布式账本。所有的交易在发送时，都需要确认两笔之前的交易，交易的确认是通过『边』来表示的。
---

这一次我们介绍的区块链项目就是 IOTA，它的团队将 IOTA 定义为『次时代的无许可』分布式账本，无论是次时代（Next-Generation）还是无许可（Permissionless）在作者看来都没有什么意义，我们还是先来看一下它解决了什么问题，再来讨论它的价值。

![tangle](https://img.draveness.me/2018-06-07-tangle.png)

IOTA 使用了基于 **有向无环图（DAG）** 设计的 Tangle，有别于传统的区块链项目，在 IOTA 或者说 Tangle 中，没有区块和链的概念，同时也没有矿工和用户之间的转账并且交易也不收取手续费；IOTA 使用的 Tangle 是我们今天想要介绍的主要内容。

## Tangle

Tangle 本质上就是一个有向无环图，所有由节点发出的交易最后都会成为图的一部分，也就是用于存储交易的分布式账本。所有的交易在发送时，都需要确认两笔之前的交易，交易的确认是通过『边』来表示的。

![iota-tangle](https://img.draveness.me/2018-06-07-iota-tangle.png)

也就是说每一笔交易都为整个网络中交易的确认做出了贡献，新的交易能够降低历史交易被篡改的几率，这也是 IOTA 与其他区块链之间的主要区别之一。

在 DAG 中，分叉是无时无刻不在出现的；然而在其他区块链网络中，例如 Bitcoin，分叉只是整个网络中的节点**暂时**没有达成共识，网络中的节点最终会确定一条唯一的主链，IOTA 的 DAG 却没有单一主链的限制，这能够在理论上增加网络的吞吐量并降低响应时间。

### Tip Selection

为了将交易加入到 Tangle 中，新的交易必须选择同意两个之前的交易；在通常情况下，它都会选择两个最近没有被同意的交易，也就是 *tips*，选择 tips 的方法叫做 *tip selection algorithm*。

尖端的选择过程就是从创世交易向尖端的随机遍历，只是不同的交易会拥有不同的权重，拥有较多权重或者较多交易索引的 tips 会被优先选择，也就是说新创建的交易会比老的交易更容易被选择。

然而所有的节点不是一定需要遵循上述推荐的尖端选择算法，但是该算法的设计就是让绝大多数的节点都遵循的，在这个前提下，其他的节点也会被激励地使用这种方式选择需要同意的交易。

### 共识

到目前为止我们简单介绍了 Tangle 网络以及交易是如何发出的，但是对于交易的确认以及 [共识](https://draveness.me/consensus) 是我们还没有解决的；在 Bitcoin 中，由于所有的交易都是附着在 Block 上的，所以区块的确认就代表区块中交易的确认：

![bitcoin-consensus](https://img.draveness.me/2018-06-07-bitcoin-consensus.png)

存在的最长链就是网络中的主链，某个区块到主链末尾之间区块的个数就是确认数，这钟实现方式非常容易理解，也很简单高效；但是 IOTA 由于使用了 DAG 作为数据结构并且没有区块的概念，整个网络中的情况就变得非常复杂。

Tangle 为我们提供了两种不同的共识策略，一种是中心化的，另一种是去中心的方式；前者是使用一个 IOTA 基金会控制的节点，也就是调节者（Coordinator），它每两分钟会发出一个不包含任何价值的交易，我们称之为里程碑（milestone），这种中心化的策略非常简单，所有被里程碑链接的交易都是被确认的，反之就是未被确认的交易：

![iota-coordinator-consensus](https://img.draveness.me/2018-06-07-iota-coordinator-consensus.png)

另一种分布式的方式并没有一定确认或者未确认的状态，与 Bitcoin 一样，它会给出某个交易在某个时刻被确认的概率，我们经常说 Bitcoin 需要等待 6 个区块确认，这是因为在 6 个区块确认后，支出方想要进行双重支付（double-spend）必须控制非常强大的算力，这种攻击在理论上是有一定概率会发生的，只是概率非常小并且代价会比较大而已，所以攻击者出于收益小于支出的原因不会去做。

在 Tangle 中，如果我们想要确认一个交易被确认的概率，可以通过执行 100 次尖端选择算法，这 100 次执行的过程中，有多少次引用了该交易，当前交易就有百分之多少的概率被确认，这个概率会随着时间的增加和网络的运行不断提高，但是在作者看来这种方式听起来非常的不靠谱，还有点可笑，不过确实是一种在 DAG 中计算某交易被确认的『可行』方法。

## 交易

作为一个区块链项目，交易应该是不可或缺的一部分，但是 IOTA 中的『交易』有一些不一样，一个 `Transaction` 既可以表示一次交易的一次 `Input`，也可以表示 `Output`，其实 `Transaction` 比较像 Bitcoin 中的 UTXO，而真正的交易在 IOTA 中名字叫做 `Bundle`，我们可以得到如下的对应关系：

![terminology](https://img.draveness.me/2018-06-07-terminology.png)

> 在这篇文章中我们会分别使用 IOTA 中的术语，`Transaction` 和 `Bundle` 来介绍其底层的数据结构，当我们提到交易时，其实说的是 `Bundle`，IOTA 理解的一次 `Transaction` 其实就是对某一个账户的单向操作，一组 `Transaction` 的组合成的 `Bundle` 才是我们通常意义上的交易。

### Transaction

我们需要分别了解 `Transaction` 和 `Bundle`的结构和字段：

```java
public class Transaction implements Persistable {
    public static final int SIZE = 1604;

    public Hash address;
    public Hash bundle;
    public Hash trunk;
    public Hash branch;
    public Hash obsoleteTag;
    public long value;
    public long currentIndex;
    public long lastIndex;
    public long timestamp;

    public long attachmentTimestamp;
    public long attachmentTimestampLowerBound;
    public long attachmentTimestampUpperBound;
    public String sender = "";
}
```

在整个 `Transaction` 中，根据当前 `Transaction` 的类型 `Input` 或者 `Output`，上述的字段也会有一些不同，输入一般会使用 `sender` 保存发出者的签名，长度一般为 2187 个 *trytes*，我们会在这一小节的最后一部分介绍三进制和 trytes，在这里我们可以当做字节（bytes）来理解。

```javascript
{ 
  hash: 'WCWHEXQXTEWYSGGPTITZVJNENDGXVF9KTEJMLSEGQEMOJHJJKFXVRENQWARBFLXOBPMWAUQYRDCTZ9999',
  signatureMessageFragment: 'ASQNBNJCJXJIITXEWFDIXEEROSDITNETRMETJWHLNEVXBXSHJWVEAGMAYU9COAJTFHZYD9DBBYSJNQEOYRHDBVMZLKRONAAKIEFMRHONIXRX9TNCRLWMCBHBVAKZFOSCXRHFCIWSSNLSZEVBOHRXBICBBERNRCXEFCE9IBVLEZDCK9EYAJBVJPFGIAHTUNHDYXOQHOKRSQSXHXRMFZOBZLORHCMWRIWETOVDWN9FKEYSTDIPSTYYGCLHLISOXBPCFHIFTLYYLPXRHYKABDJCTWCUEUUFCPOLWEBTMWBBPYTZWTBHPODYCXVVFEKVZXBQEROXQQXBHYNUHKUOSWF9MZLAEIBKWM9ZHM9KHTWRCMEMYBRTCNIX9IP9IOMTWJRPNKXSJQHKXVTC9CMRPPDZZNGGNVLVQWEFNEAXYFAEOKTKOPEEPPGGETATTBCSEYDOJNVRFAFHZQZUMSVMQCYDTB9CUPPPZTQSMSFFYDFECJXAJSTLOLIGI9DTFDAUIMCIKSHLACSXQ9CKTCWUSQYKPQTHRWMFNSRYU9TKGIWX9XMVQW9WCZDJTLYTNMYMKFDTNXLSQWNBSOKWBDBRKNDLEKTHRILNJFFDYFMYYYTIGHRMMGZLM9QZVERTLWIXROCDXNEIGSW9GXZBQXJKNLPIWBFYYNKXPLNLFJRENTPTYPCTHCLUTCX9SB9UCGHYNNDISCAJNCNGCHRSXSQUUMBGHHOVBUMOJDQINW9HARJXXLMFBWL9D9CMCXKYL9ZEUITRYFRJFRERAPAPTDVQCTSRUPFFTYGXVTQSVSQOWIPPKCVOPUUQVFETQMTRUVFMENFLN9WUEUMVYJHX9RDONDJDNKYDIKOSDJHEP9FQSBVZTBMGITODYFCBMTUUZDDQNALVPHHPRESUJJWK9CBXFFYOG9EMLMORHLXEEVLTNDDVEDRGLLUCFXEQSGW9GXJFZZTBQULYEZLO9KADEUHVWSSESEWVXJXDWPGHTENPFYAKSPWOQYCO9CYPXDIXGZX9K9JSTBVQDQDGZUYWUPJGECGPJPEKOYYQ9NFLWJCNFVLIIBRXHIYDECZASUIHGSSFTCVLBONLQIWFQRMTOGBFRPIAD9TUJ9JSFI9QCOBXCB9IVXHHGYIOFNYZEIGQWTNZXTZZVEHCVADXFSSXLMPQGHOAWRICSRIVRCNEXPVQP9IETRDCPFSVSMYSEJAYM9SIDZ99P9LHSNFQSRKXYUPGJJHRTCVZZNC9XLRINWCRZHUDZCBKEIJOPYMZTAVM9RWLLLRYJXCPPVHAGMG9WMGCD9UMZNFMINSHKJGSROKAPQWUBVIWLWMRXZNQMKADZQOOURUEYJMSIVKPOFZZIQTOGUTBYYMMTWLRPHUOYMLOVIXUIPGSYCKKEEERAWTTRDYVNQWFJTKCJ9OWKAEOVZPNJQIHUVZDXCREXHIQOMTQGVTHRWEICZMIDDVVLZ9OSSMEPIRINRWKFXYUON9MVASPAMNHJNSSQVNVFWRPOHSAVLRCOIABGEYUYDYHEBJJGRBGBRQTGNXKOSACTKTH9NWUJIPUWAADRVRUTPRCYKSBTVQOZMCUBOFHTIHCCTTY9JZDA9VZJV9JKQTGGHBSGJEDAOMCSTJOOHS9HBETRMERVSZWMCVHBUKLGEWVPZAXYEKUVWYQNYAWTSTLXXVXDYAQHZTPQMCMMDVURZMAWJZTBJOFHPDODCJF9PMLIJTTDWUQKCMGHCVYGJADDDFGSZFLFEHSHVRIIEYHTMVKOKDQBJSQOASRRUVDVLRYGAPHZCDKK9UWKNEQWZYMHJSGOHIMDZUYKFYJQAMKICQUIUMA9FBOTCTQVHPHLVPHORLMXWFLOFVVWQBANEBLF9STFWLVGOKYTL9V9KYEYMGPRODNGKFCBEBLHTBGHXAPGEAYPQVSBKMOSZPECQEZQGHSOR99TPJTIDEWZFXASV9YJDMKMPDFUXLOOQCHPBTVTEGIJUXUVFAAYQI9RLWDNHTRMIC9IMJGKULGGXLKHLFJ9WWQDEVJA9DKSEXIIRQMMVBPFPNHBJYSEKKAJZAEWONTBCSYEFIEPOGG9USMBMOO9KQQNFAIJ9DOSZMUINJAWOKDEDCJTSMFOXGQNZMVPEFFFQPAZD9WDSYKDOGPOZZSMYH9B9OWHLOUBCHTPKECJKDCSVREBNXJGCZFWSRDLHD',
  address: 'TDIJAKIZYMBFUHRXXPTOFNMC9UPJQJBPELGMQWJOULAYSNKRPYKTMBZOEITGBMMGI9JMAVDNO9QNGOMFY',
  value: -6062321,
  obsoleteTag: 'MINEIOTADOTCOM9999999999999',
  timestamp: 1526454765,
  currentIndex: 4,
  lastIndex: 6,
  bundle: 'WIIPVFKRGKVEXGEEPQROFWRSDLW9WONYVTCGJDWJREGXPLGVSJAVJSKYHHZDWBX9JUISRTCUVSXWLYCKW',
  trunkTransaction: 'LIWIJSQSZNUWMBWQMNSXZQNPGCCVVUKTRMWFHIUKTSMQZ9GCOCTFXVKXWTFBGYEKVCLJWWIBYOSWZ9999',
  branchTransaction: 'ASIETWSCPXMKGMP9I9QPUNHEETTOFCKKLQAEBEYBONQXOMXUBPDSJGUFEXWTD9AM9HADUESAKIRD99999',
  tag: 'MINEIOTADOTCOM9999999999999',
  attachmentTimestamp: 1526456063898,
  attachmentTimestampLowerBound: 0,
  attachmentTimestampUpperBound: 12,
  nonce: 'RMDY9PXCX9YPTCDUBPDOYDHSPQF'
}
```

上述内容其实就是一个 `Transaction` 对象，其中的 `value` 就表示当前的输入或者输出，输入都为负数，输出都为正数，而 `trunkTransaction` 和 `branchTransaction` 分别表示当前 `Transaction` 同意的第一个和第二个 `Transaction`；另外的 `currentIndex` 和 `lastIndex` 表示当前 `Transaction` 在整个 `Bundle` 中的位置，以及最后一个 `Transaction` 的序列。

### Bundle

作为 IOTA 项目中真正的交易，`Bundle` 的数据结构其实非常简单，它其实就是一个从 `transactionHash` 到 `transaction` 的字典：

```java
public class Bundle extends Hashes {
    public Bundle(Hash hash) {
        set.add(hash);
    }

    public Bundle() {}
}
```

在一个常见的 `Bundle` 中往往会由四个以上的 `Transaction` 组成：

Index | Purpose | Balance
--- | :---: | ---
0 | 输出的地址，它是所有输入 IOTA 的接受方 | >0 (output)
1 | 整个 `Bundle` 会花费当前输入中的全部 `value`，当前 `Transaction` 中包含 IOTA 拥有者签名的一半 | < 0 (input)
2 | 仅包含当前 `Bundle` 中唯一 `Input` 中的另一半签名 | 0
3 | 输入与输出的差值会通过该 `Transaction` 交给目前地址 | >0 (input - output)

当然，在 IOTA 中并不是所有的 `Bundle` 都需要正好四个 `Transaction` 组成，也会有多于四个的情况出现，例如下面的 Bundle [#WIIPVFK](https://thetangle.org/bundle/WIIPVFKRGKVEXGEEPQROFWRSDLW9WONYVTCGJDWJREGXPLGVSJAVJSKYHHZDWBX9JUISRTCUVSXWLYCKW) 总共由 7 个 `Transaction` 组成：

Index | Transaction | Purpose | Balance
:-: | :---: | :--: | --:
0 | [#TANYXL](https://thetangle.org/transaction/TANYXLPLVBVTQB9NWUZQFOCUQHBSQCAYJFXQKPJVGOCEEYZVHGLK9EKJCGUNOWQNS9SRRURBGOWRZ9999) | Output | 413
1 | [#OLZZLH](https://thetangle.org/transaction/https://thetangle.org/transaction/OLZZLHYYTAVTABATNITGBEVXDWXPKTQMRAIJJQICESSXZXXNAHSBEVHDNEPJVDUASOFMBTZYSRMSZ9999) | Output | 5,214
2 | [#YUEYLQ](https://thetangle.org/transaction/YUEYLQRSZVSMGXUCFR9HYXCUT9OBJTUHZSFPPRNCBITHCEPXTQVEVCBDHJDQLX9CVQWR9JRMNOMZA9999) | Output | 618
3 | [#KWWCPW](https://thetangle.org/transaction/KWWCPWFA9XUJJOJKJHRDLGTSEJQOIMGRQQFHXYKDYLSLAK9ZDWOV9TESHIGKXLTWLZJPS9UQDQRCZ9999) | Output | 45,608
4 | [#WLWMSA](https://thetangle.org/transaction/https://thetangle.org/transaction/WLWMSA9WT9SJ9LRGCFCEKODAUQPXUKWMKNQXXGBELGIQFCRBBTVUGOALKXIVO9PJCLHXIGPMXOJTA9999) | Input | -6,062,321
5 | [#TCOMBN](https://thetangle.org/transaction/TCOMBNISFDHQEROSYZDTJNPY99HNFFAWAMOMRXMMFDFQVEKW9UWUXPNTMJBGDDIARFOTCZOMNBTPA9999) | Signature | 0
6 | [#QYYOLK](https://thetangle.org/transaction/QYYOLKQNMCAAGWM9PJRSFCBSVBYDQNATUPNVIOD9JVUVKNKITPK9LUXVEUSKIMBBZENKOSUKCXN9A9999) | Remainder Output | 6,010,468

刚刚了解 IOTA 的人会觉得非常奇怪，为什么它要把一个输入的签名放到两个 `Transaction` 中，作者也觉得这种实现方式不是特别优雅；这主要是因为 IOTA 的 `Transaction` 长度是固定的 2673 个 trytes，这种所有字段定长的方式非常利于解析器的解析，能够减少解析的工作量，但是在遇到这种时候确实非常别扭，我们需要在两个 `Transaction` 中分别包含一个签名的两部分。

作者猜测这种实现的方式是因为签名固定占用了 `2187 trytes`，但是签名的长度大于 `2187 trytes` 如果想要遵循所有字段的长度都是 3 的倍数，签名的固定长度就变成了 `6561 trytes`，远远大于签名的长度，并会增加其他不需要签名的 `Transaction` 的开销，所以选择了这种比较奇葩的实现方式。

> IOTA 的签名使用了后量子签名算法 [Winternitz one-time signature](https://eprint.iacr.org/2011/191.pdf)，这种签名算法为了提高安全性，签名内容长度非常长并且每个签名只能使用一次，想要了解更多内容的可以阅读相关论文。

### 三进制

trytes 其实可以理解为三进制世界的字节，IOTA 的底层编码使用三进制，与二进制的 bit 表示 0 和 1 不同，三进制的 trits 表示 0、1 和 2 三个状态，而 trytes 其实就是三个 trits 总共表示 27 个状态，用 26 个大写字母和 9 来表示，例如 `IPQYUNLDGKCLJ`；作者认为 IOTA 使用三进制编码数据，并且使用自己发明的 trytes 目前看来只是增加了开发和理解的复杂度，在上面的 `Transaction` 定义中，我们看到了一个非常诡异的数字 1604：

```java
public class Transaction implements Persistable {
    public static final int SIZE = 1604;
    //...
}
```

它其实就是 2673 个 trytes 对应的近似字节数，从 trytes 转换到 bytes 在作者看来十分别扭和难以理解，我们可以用一下的 [计算器](https://laurencetennant.com/iota-tools/) 帮助我们进行转换：

![bytes-trytes](https://img.draveness.me/2018-06-07-bytes-trytes.png)

从二进制到三进制，bytes 到 trytes 的转换非常丑陋，IOTA 的代码中为了兼容三进制的设计，做了非常多奇怪的设计：

```java
public void readMetadata(byte[] bytes) {
    int i = 0;
    if(bytes != null) {
        address = new Hash(bytes, i, Hash.SIZE_IN_BYTES);
        i += Hash.SIZE_IN_BYTES;
        // ...
        value = Serializer.getLong(bytes, i);
        i += Long.BYTES;
        currentIndex = Serializer.getLong(bytes, i);
        i += Long.BYTES;
        // ...
        byte[] senderBytes = new byte[bytes.length - i];
        if (senderBytes.length != 0) {
            System.arraycopy(bytes, i, senderBytes, 0, senderBytes.length);
        }
        sender = new String(senderBytes);
        parsed = true;
    }
}
```

IOTA 团推认为三进制架构的电路功耗低，并且其创始人相信未来世界会被三进制取代，所以选用的三进制作为 IOTA 的编码方式，但是从**目前**看来，三进制的实现方式不仅缺少硬件支持而且缺少软件和库的支持，没有得到预想的性能提升而且还增加了开发难度和理解障碍，是一件没有什么收益的事情。

### Curl

因为 IOTA 使用了三进制作为编码方式，所以使用了 SHA-3 发明者设计的三进制哈希算法 Curl；但是波士顿和 MIT 大学在 2017 年 9 月发布了一篇名为 [IOTA Vulnerability Report: Cryptanalysis of the Curl Hash Function Enabling Practical Signature Forgery Attacks on the IOTA Cryptocurrency](https://github.com/mit-dci/tangled-curl/blob/master/vuln-iota.md) 的报告，指出 IOTA 使用的 Curl 三进制哈希算法存在哈希碰撞的漏洞。

![cur](https://img.draveness.me/2018-06-07-curl.png)

在这里，作者并不想介绍 Curl 具体的实现原理，主要是想介绍密码学中的一条『黄金法则』- **永远不要自己发明加密算法**。

> The golden rule of cryptographic systems is “don’t roll your own crypto.”

作为区块链项目底层的重要设施的哈希算法，Curl 并没有经历过较长时间的研究和审查就这样使用在了公链中，这种做法并不是特别的**明智**，在出现安全隐患之后，IOTA 团队切换至更加成熟的 SHA-3 哈希算法。

## 总结

IOTA 的三进制其实也是一个有着理由比较『牵强』的设计 - 三进制的 CPU 效率比二进制的高，但是实际上市面上不存在成熟的三进制计算设备，而且软件工程在几十年的时间一直都是建立在二进制之上的，相关的哈希以及加密算法都不成熟，这也导致了 Curl 的使用并最终被披露具有安全隐患，整个项目迁移到其他的哈希算法上。

交易长度使用了定长的方式设计，可能一方面是因为如果加入定长的变量，二进制和三进制之间的相互转换变得更加复杂，另一方面定长的交易能够极大地降低解码器的复杂度，但是为了不大大增加 `Transaciton` 的平均长度，又需要将一个 `Transaction` 的签名分割到两个 `Transaction` 中，这种实现真的非常丑陋。

IOTA 虽然在设计上有非常多的问题，但是它使用有向无环图作为区块链底层的数据结构的思路确实非常新颖，我在网络上找到了如下所示的一张图，其中展示了 IOTA 相比 Bitcoin、Ethereum 等项目，TPS 有了指数级的提高。

![iota-tps](https://img.draveness.me/2018-06-07-iota-tps.png)

作者并不是特别清楚上述的 TPS 是如何计算的，网络上并没有找到关于 IOTA 性能分析的资料，在这里也希望读者简单了解一下就好。相比于其他的空气公链，IOTA 还是有实际产出的，但是它在设计上有很多问题，并且过了一年时间，官方的钱包和节点非常**难以使用**，很多资料也严重缺失，作者还是希望 IOTA 社区能够把相关的工具做好。

## 相关文章

{% include related/blockchain.md %}

## Reference

+ [IOTA 介绍](http://www.iotachina.com/what-is-iota)
+ [What is IOTA](https://docs.iota.org/introduction)
+ [Transaction Speed - Bitcoin, Visa, Iota, Paypal](https://steemit.com/cryptocurrency/@steemhoops99/transaction-speed-bitcoin-visa-iota-paypal)
+ [分布式一致性与共识算法](https://draveness.me/consensus)
+ [IOTA Deepdive · What is IOTA](https://domschiener.gitbooks.io/iota-guide/content/chapter1.html)
+ [What are trytes and trits?](https://iota.stackexchange.com/questions/328/what-are-trytes-and-trits)
+ [Why does IOTA use a ternary number system?](https://iota.stackexchange.com/questions/8/why-does-iota-use-a-ternary-number-system?noredirect=1&lq=1)
+ [Cryptographic vulnerabilities in IOTA](https://medium.com/@neha/cryptographic-vulnerabilities-in-iota-9a6a9ddc4367)
+ [Debunking the ‘IOTA Vulnerability Report’](https://medium.com/iota-demystified/debunking-the-iota-vulnerability-report-c40fb07a6ae8)
+ [Why does the signatureMessageFragment have a fixed size?](https://iota.stackexchange.com/questions/700/why-does-the-signaturemessagefragment-have-a-fixed-size)
+ [IOTA Converters](https://laurencetennant.com/iota-tools/)
+ [IOTA Vulnerability Report: Cryptanalysis of the Curl Hash Function Enabling Practical Signature Forgery Attacks on the IOTA Cryptocurrency](https://github.com/mit-dci/tangled-curl/blob/master/vuln-iota.md)
+ [On the Security of the Winternitz One-Time Signature Scheme](https://eprint.iacr.org/2011/191.pdf)

