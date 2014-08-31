<pre>
  BIP: XX
  标题: 原子化跨链转账
  英文作者: Noel Tiernan <tier.nolan@gmail.com>
  英文地址：https://github.com/TierNolan/bips/blob/bip4x/bip-atom.mediawiki
  翻译： 招财币冷月 新浪微博 @招财币冷月
  翻译地址：https://github.com/zccoin/atomic/blob/master/bip-atomic_zh.md
  状态: 草稿
  类型: Standards Track
  创建日期: 2014-04-29
</pre>

==摘要==

这个BIP描述了一种在比特币和类比特币的虚拟币（altcoin）之间自动交易虚拟币的方法。 

==动机==

有很多的虚拟币交易平台。这些网站允许用户使用比特币来交易altcoin，并且也能在不同altcoin间兑换。

这些站点生来是中心化的。一个p2p虚拟币兑换系统要求一种能原子化交易虚拟币的方法。

==协议概述==

本BIP定义的协议包含两个阶段。 在第一阶段， 双方共同生成一系列交易但是并不广播。在第二阶段，交易按照一定顺序广播出去。仅仅在第一阶段需要双方间进行通信。
每个参与方都有各自的动机来按照定义的广播顺序进行。 如果协议在交易提交前任一阶段停止，所有参与方都能使用时间锁定的退款交易来恢复资金。

假定Bob想用B个比特币（BTC）从Alice买A个altcoin（ATC）。比特币网络的交易费用是fb，altcoin网络为fa。Bob将付所有比特币费用，并且Alice将付altcoin费用。 交易方达成的兑换价格将把这些考虑进去。

Alice的公钥被称为pub-AN，Bob的称为pub-BN。有一个可选的第三方签名也是可能的，称之为pub-T。

第三方仅仅用来防止交易延长性。一旦交易延展性被解决，第三方就不再需要。  

为了使协议工作，在其中一个网络中要求一个附加的标准交易类型。 一个网络支持P2SH，另外一个支持OP_HASH160交易锁定，交易就可以进行。

===交易创建===

1) Alice发给Bob三个公钥 (pub-A1, ..., pub-A3)

2) Bob发送给Alice三个公钥(pub-B1, ..., pub-B3)和Hash160(x)

    x = serialized{pub-B4 OP_CHECKSIG}

3) 所有交易方创建他们的 "bail-in" 交易。 

交易输出0仅能被双方签名花费。交易输出1仅能被Bob花费，但是它将使x被暴露。

    名称: Bob.Bail.In
    输入值:     B + 2*fb + change
    输入源:    (From Bob's coins, multiple inputs are allowed)
    0输出值:  B
    ScriptPubKey 0:  OP_HASH160 Hash160(P2SH Redeem) OP_EQUAL
    1输出值:  fb
    ScriptPubKey 1:  OP_HASH160 Hash160(x) OP_EQUALVERIFY pub-A1 OP_CHECKSIG
    2输出值:  change
    ScriptPubKey 2:  <= 100 bytes

    P2SH Redeem:  OP_2 pub-A1 pub-B1 OP_2 OP_CHECKMULTISIG
    P2SH Redeem:  OP_2 pub-A1 pub-B1 pub-T OP_3 OP_CHECKMULTISIG

交易输出0仅能被双方签名花费。  交易输出1仅能被Alice花费，但是它要求Bob首先给出x。

    名称: Alice.Bail.In
    输入值:  A + 2*fa + change
    输入源: (From Alice's altcoins, multiple inputs are allowed)
    0输出值: A
    ScriptPubKey 0: OP_HASH160 Hash160(P2SH Redeem) OP_EQUAL
    1输出值: fa
    ScriptPubKey 1: OP_HASH160 Hash160(x) OP_EQUAL
    2输出值: change
    ScriptPubKey 2: <= 100 bytes

    P2SH赎回:    OP_2 pub-A1 pub-B1 OP_2 OP_CHECKMULTISIG
    P2SH赎回:    OP_2 pub-A1 pub-B1 pub-T OP_3 OP_CHECKMULTISIG

注意: x = serialized{pub-B4 OP_CHECKSIG}

稍短版本的P2SH赎回用于当不使用第三方时候。

1输出使用P2SH，这意味着为了花掉它Bob必须提供x。

如果不是用第三方，那么pub-T不被包含在内并且密钥个数(OP_3)替换为OP_2.

4) Bob和Alice交换bail-in交易hashes

Bob发送给Alice Hash256(Bob.Bail.In)

Alice发送给Bob Hash256(Alice.Bail.In)

注意: bail-in交易的输出并不需要按照步骤1和2中给出的顺序进行。

5) 所有参与方创建付费交易

这个交易能被Alice花费。由于它包含Bob.Bail.In:1作为输入, 它不能被签名除非Bob给出x。

    名称: Alice.Payout
    输入值:  B
    输入源: Bob.Bail.In:0
    输入值:  fb
    输入源: Bob.Bail.In:1
    输出值: B
    ScriptPubKey: OP_HASH160 Hash160(P2SH Redeem) OP_EQUAL

    P2SH 赎回:  pub-A2 OP_CHECKSIG

这个交易能被Bob话费。  然而, 由于它包含Alice.Bail.In:1作为输入，他不能签名输入除非他提供x。

    名称: Bob.Payout
    输入值:  A
    输入源: Alice.Bail.In:0
    输入值:  fa
    输入源: Alice.Bail.In:1
    输出值: A
    ScriptPubKey: OP_HASH160 Hash160(P2SH Redeem) OP_EQUAL

    P2SH 赎回:  pub-B2 OP_CHECKSIG

6) 所有交易方创建退款交易

这个交易是时间锁定的，所有它不能被花费知道时间(T)已过。这个交易不要求Bob.Bail.In:B，所以Bob为了花费它并不需要提供x。

    名称: Bob.Refund
    输入值:  B
    输入源: Bob.Bail.In:0
    输出值: B - fb
    ScriptPubKey: OP_HASH160 Hash160(P2SH Redeem) OP_EQUAL
    Locktime:     (current block height) + (T / 10 minutes)

    P2SH 赎回:  pub-B3 OP_CHECKSIG

这个交易是时间锁定的，所以它不能被花费直到半数时间（T/2）已过。 这个交易并不要求Alice.Bail.In:B，所以Alice可以花费它而不需要知道x.

    名称: Alice.Refund
    输入值: A
    输入源: Alice.Bail.In:0
    输出值: A - fa
    ScriptPubKey: OP_HASH160 Hash160(P2SH Redeem) OP_EQUAL
    Locktime:     current block height + ((T/2)/(altcoin block rate))

    P2SH 赎回:  pub-A3 OP_CHECKSIG

7) Bob和Alice交换签名

Bob发送给Alice Alice.Payout (Input: Bob.Bail.In:0)和Alice.Refund的签名。

Alice签名了所有三个，并且现在有了三个签名的交易。  Alice.Payout并不能完全签名直到x被暴露出来。

Alice发给Bob Bob.Payout (Input: Alice.Bail.In:0), Bob.Refund and Bob.Trusted.Refund的签名.

Bob签名了所有三个，并且有了三个完全签名的交易。

8) 交换bail-in交易

现在所有参与方可以安全的交换bail-in交易。这对于协议并非必须，但是可以让参与方在锁定他们的资金前验证另外一方至少有一个合法的bail-in交易。 

参与方之间不再需要进一步的通信。 Bob和Alice能够通过监测两个区块链来判决交易状态。

===交易广播===

为了使复合交易原子化，交易必须按照特定的顺序广播出去。  

Bob拥有下面的交易:

* Bob.Bail.In: Bob的bail-in交易
* Bob.Refund:  Bob的退款交易 (时间锁定直到过期)
* Bob.Payout:  Bob的altcoin付款交易 (仅能通过给出x来被花费)

Alice拥有下面的交易

* Alice.Payout:  Alice的比特币付费交易 (仅当Bob给出x后被花费)
* Alice.Bail.In: Alice的bail-in交易
* Alice.Refund:  Alice的退款交易 (时间锁定直到一半锁定期)

====步骤 1: Bob执行Bail-in====

Bob广播Bob.Bail.In.

Bob除了等待没有其他选择
* 他不能广播Bob.Refund，因为它是时间锁定的
* 他不能广播Bob.Payout，因为它使用Alice.Bail.In作为输入

Alice唯一可选的广播选项是转到步骤2
* 她不能广播Alice.Payout，因为她不知道x
* 她不能广播Alice.Refund，因为它是时间锁定的
* 她能广播Alice.Bail.In，这也是协议的步骤2

如果协议在这时终止，Bob在时间过期后能使用他的退款交易来恢复他的比特币。

====步骤 2: Alice执行Bail in====

Alice广播Alice.Bail.In.

Alice除了等待没有其他选择
* 她不能广播Alice.Refund，因为它时间锁定的
* 她不能广播Alice.Payout，因为她不知道x

Bob的唯一广播选择是进行到步骤3
* 他不能广播Bob.Refund，因为它是时间锁定的
* 他能广播Bob.Payout，因为Alice.Bail.In已经被广播

推荐Bob在进行到步骤3前先等待Alice.Bail.In被多个区块确认。

====步骤 3: Bob完成交易====

Bob广播Bob.Payout来得到他的altcoins。为了花费Alice.Bail.In的第二个输出要求Bob发布x。

这完成了Bob在协议中的参与。

一旦Alice.Bail.In已经被确认到足够深度， Bob应该尽快广播Bob.Payout。 因为广播Bob.Payout给出了x，如果Bob一直等到Alice.Refund的锁定时间过期(或者临近过期), 那么就产生了一个竞争状态。  Alice可以广播Alice.Refund来拿回她的altcoins并且广播 Alice.Payout来得到Bob的比特币。

加入他立即广播，他就有锁定时间的一半来使他的交易被确认。

Alice仅有一个广播选项
* 她不能广播Alice.Refund，因为它是时间锁定的
* 她能广播Alice.Payout，因为她知道x，这将进行到协议的步骤4

====步骤 4: Alice完成交易====

Alice广播Alice.Payout来获得她的比特币。

假如Alice不在Bob.Refund的时间锁定结束前拿回她的比特币，那么Bob能使用Bob.Refund来恢复他在交易中使用的比特币。

由于Bob在完成步骤3前最多等待一半锁定时间，并且Bob.Refund有T的锁定时间， Alice至少有一半锁定时间来广播Alice.Payout。

==规格==

参与方之间通信应该使用JSON-RPC。

对于所有的字节数组应该使用16进制编码。

公钥必须使用严格的SEC格式进行序列化：

    byte(0x02) byte_array(32):                Compressed even key
    byte(0x03) byte_array(32):                Compressed odd key

压缩的key是强制性的。

当打包进交易时，hash_type必须设置为1。

签名必须使用严格的DER格式进行序列化。

    byte(0x30) byte(total_length) byte(0x02) byte(len(R)) byte_array(len(R)) byte(len(s)) byte_array(len(s))

total_length等于 3 + len(R) + len(S).

R和S表示为有符号的没有前导0的Big Endian格式, 例外情况是只能精确的有一个前导0，对于不如此就是负的数。 这发生在第一个非零字节的MSB被设置的情况下。

交易应该使用比特币的序列化协议进行序列化。没有被时间锁定的交易应该设置 lock_time和 sequence number为0. 时间锁定的输入应该有一个 UINT_MAX (0xFFFFFFFF)的sequence number。

一个参与方作为服务器，一个参与方作为客户端。

选择x和具有较长锁定时间的一方被定义为慢速交易方。另外一方是快速交易方。

===请求消息===

每个消息应该有下面的格式。

    {"id":1, "method": "method.name", "params": [param1, param2]}

id:     方法id，每个方法调应应该递增
method: 方法的名称
params: 方法参数

===结果消息===

服务器应该使用响应消息来回复请求消息。

    {"id": 1, "result": result, "error: Null}

===错误消息===

如果请求非法，服务器应该回复一个错误消息。

    {"id": 1, "result": Null, "error: [error_code, "error_string"]}

===方法===

如下所示的方法必须被支持。

===交易请求===

这个方法用来初始化一个交易。

    {"id":1, "method": "trade.request", [version, long_deposit, [third_parties], k_client,
                                         sell_magic_value, sell_coin_amount, sell_coin_fee, 
                                         sell_locktime,
                                         buy_magic_value, buy_coin_amount, buy_coin_fee, 
                                         buy_locktime]}

参数定义为

    version:                        握手的整数版本 (应该设置为1)
    slow_trader (boolean):          如果服务器是慢速交易者设置为True，否则为false
    third_party (list of string):   被接受的第三方公钥的16进制编码(没有第三方则为Null)
    k_client (string):              一个随机的16进制编码的字节数组(32 bytes)
    sell_coin_magic_value (string): 被出售币网络的magic value的16进制编码
    sell_coin_amount (number):      被出售币的整数数额(按最小单位计算)
    sell_locktime (number):         客户端退款交易的整数锁定时间
    buy_coin_magic_value (string):  被买币的网络magic value的16进制编码
    buy_coin_amount (number):       被买币的整数数额(按最小单位计算)
    buy_locktime (number):          服务器退款交易的整数锁定时间

服务器能够决定这个交易是否可接受的。

对于区块速率异常的altcoins，确保一个按照正确顺序发生的锁定时间可能困难。推荐采用比较稳定的块链作为慢速交易者。如果altcoin的区块速率崩溃了，这防止了慢速交易者不得不等待一个延长的时间。

注意: 锁定时间按照数值能够表示时间戳和区块高度。

这个方法的响应包含一个交易信息的子集。

    {"id":1, "result": [version, slow_trader, [third_parties], k_server,
                        sell_coin_amount, sell_coin_fee, sell_locktime,
                        buy_coin_amount, buy_coin_fee, buy_locktime]
             "error": Null}

假如返回值匹配了请求，那么交易被接受。否则，它是一个还价。

如果服务器不支持客户端请求的协议版本，响应中的版本应该等于较高支持的版本，并且不包含其他参数。否则，服务器应该使用请求的版本回应。

在公钥列表中，一个被接受的报价应该最多包含一个第三方的公钥。

假如服务器完全不想交易那种币，那么buy_coin_amount, _fee和_locktime应该在响应中设为Null。

假如交换比例不够，服务器应该优先修改sell_coin_amount而不是buy_coin_amount.

加入客户端回应给服务器，服务器应该接受交易。

每个交易生成一个trade-id(| 表示串联).

    tr_id = SHA-256(k_client | k_server)

SHA-256运算的结果被作为big endian编码的无符号整数。

假如tr_id超出elliptic curve finite field, 服务器应该选择一个不同哦自己数组并且重复直到成功。

第三方公钥修改后得到

    third_party_key_modified = tr-id * third_party_key

这个key应该用来生成第三方退款交易。

===交换公钥===

这个方法用来在参与方间交换公钥。每一方需要提供5个公钥，并且必须提供hash_x.  慢速交易者应该设置hash_x为Null。

    {"id": 1, "method":"keys.request", "params": [tr_id, key1, key2, ... key5, hash_x]}

服务器应该使用5个公钥和hash_x进行响应。

    {"id": 1, "result": [key1, key2, ... key5, hash_x], "error": Null}

===交换Bail-in交易Hashes===

这个方法用来交换bail-in交易hashes，A和B索引。

    {"id": 1, "method":"k, bail_in_hash.request", "params": [tr_id, client_bail_in_hash]}

服务器应该使用自己的bail-in交易hash来响应。

    {"id": 1, "result":"server_bail_in_hash", "error": Null}

所有hashes应该是编码为64 character (32 byte) hashes.  

备注1: 这是交易id hash, 不是 hash160。
备注2: 快速和慢速交易者的bail-in交易构造不同。

===交换签名===

这个方法用与参与方交换签名。

    {"id": 1, "method": "exchange.signatures", "params": [tr_id, server_payout_signature, server_refund_signature, server_third_party_signature]

参数定义如下
    server_payout_signature:          这是服务器付款交易签名 (input A)
    server_refund_signature:          这是服务器时间锁定退款交易签名
    server_third_party_signature:     这是服务器直接输出到第三方交易签名 

响应使用同样的形式

    {"id": 1, "result": [client_payout_signature, client_refund_signature, client_third_party_signature], "error": Null}

参数定义如下
    client_payout_signature:          这是客户端付款交易签名(input A)
    client_refund_signature:          这是客户端时间锁定退款交易签名
    client_third_party_signature:     这是客户端直接输出到第三方交易签名

一旦交换过三个签名，即不再需要进一步的通信。

===交换Bail-in交易===

这个方法是参与方用来交换bail-in交易。这使得双方广播所有bail-in交易。

在本步骤前广播bail-in交易可以降低另外一方的延展性风险。

    {"id": 1, "method": "exchange.bail.in", params: [tr_id, server_bail_in]}

响应是同样的形式

    {"id": 1, "result": [client_bail_in], "error": Null}

===取消交换===

这使得交易方在锁定过期前取消一次交易。这是一种礼貌而不是强制性的。

    {"id": 1, "method": "cancel.transaction", "params": "unlocked_server_refund_signature"}

参数定义如下
    unlocked_server_refund_signature:   这是锁定时间设为0的服务器退款交易的签名。

响应是
    {"id": 1, "result": [unlocked_client_refund_signature]}

参数定义如下
    unlocked_client_refund_signature:   这是锁定时间设为0的客户端退款交易的签名。

因为这个方法仅是一种礼貌，服务器不能提供给客户端退款交易也并不影响。

一旦使用了这个方法，交易方就不应该进入步骤3。

===第三方仲裁===

这个方法用来提交交易给第三方。

    {"id": 1, method:"arbitrate", "params": [tr_id third_party_key bail_in_p2sh_redeem refund_transaction new_transaction]}

参数定义如下
    tr_id:                  tr_id 参数编码为一个整数
    third_party_key:        第三方未修改的公钥
    bail_in_p2sh_redeem:    bail-in交易的P2SH赎回脚本
    refund_transaction:     退款交易，完全签名过
    new_transaction:        这是一个新的退款交易

第三方必须
 * 验证退款交易花费了 p2sh_redeem script
 * 验证退款和新的交易除了tx-in hashes外是相同的。

新的交易可能有额外的输出。这使得能付款给第三方。

响应是

    {"id": 1, "result": [new_transaction_signature], "error": Null}

第三方不需要监控所有的块链。只要它不允许修改锁定时间或者输出重新转向，系统就是安全的。

==兼容性==

本BIP中的协议要求使用一个非标准scriptPubKey.

Alice.Payout交易有下面形式的一个scriptPubKey。

    OP_HASH160 Hash160(x) OP_EQUAL_VERIFY pub-A2 OP_CHECKSIG

所有其它的scriptPubKeys是标准交易。

假如这个交易仅仅在两个网络中一个是标准的，那么在那个网络上卖币的一方应该是快速交易者。

==参考实现==

TBD

== 参考文献 ==

[1] https://bitcointalk.org/index.php?topic=193281.0

==Copyright==

This document is placed in the public domain.