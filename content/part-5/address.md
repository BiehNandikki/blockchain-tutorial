地址
====

## 引言

在上一篇文章中，我们已经初步实现了交易。相信你应该了解了交易中的一些天然属性，这些属性没有丝毫“个人”色彩的存在：在比特币中，没有用户账户，不需要也不会在任何地方存储个人数据（比如姓名，护照号码或者 SSN）。但是，我们总要有某种途径识别出你是交易输出的所有者（也就是说，你拥有在这些输出上锁定的币）。这就是比特币地址（address）需要完成的使命。在上一篇中，我们把一个由用户定义的任意字符串当成是地址，现在我们将要实现一个跟比特币一样的真实地址。

>本文的代码实现变化很大，请点击 [这里](https://github.com/Jeiwan/blockchain_go/compare/part_4...part_5#files_bucket) 查看所有的代码更改。

## 比特币地址

这就是一个真实的比特币地址：[1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa](https://blockchain.info/address/1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa)。这是史上第一个比特币地址，据说属于中本聪。比特币地址是完全公开的，如果你想要给某个人发送币，只需要知道他的地址就可以了。但是，地址（尽管地址也是独一无二的）并不是用来证明你是一个“钱包”所有者的信物。实际上，所谓的地址，只不过是将公钥表示成人类可读的形式而已，因为原生的公钥人类很难阅读。在比特币中，你的身份（identity）就是一对（或者多对）保存在你的电脑（或者你能够获取到的地方）上的公钥（public key）和私钥（private key）。比特币基于一些加密算法的组合来创建这些密钥，并且保证了在这个世界上没有其他人能够取走你的币，除非拿到你的密钥。下面，让我们来讨论一下这些算法到底是什么。

## 公钥加密

公钥加密（public-key cryptography）算法使用的是成对的密钥：公钥和私钥。公钥并不是敏感信息，可以告诉其他人。但是，私钥绝对不能告诉其他人：只有所有者（owner）才能知道私钥，能够识别，鉴定和证明所有者身份的就是私钥。在加密货币的世界中，你的私钥代表的就是你，私钥就是一切。

本质上，比特币钱包也只不过是这样的密钥对而已。当你安装一个钱包应用，或是使用一个比特币客户端来生成一个新地址时，它就会为你生成一对密钥。在比特币中，谁拥有了私钥，谁就可以控制所有发送到这个公钥的币。

私钥和公钥只不过是随机的字节序列，因此它们无法在屏幕上打印，人类也无法通过肉眼去读取。这就是为什么比特币使用了一个转换算法，将公钥转化为一个人类可读的字符串（也就是我们看到的地址）。

>如果你用过比特币钱包应用，很可能它会为你生成一个助记符。这样的助记符可以用来替代私钥，并且可以被用于生成私钥。[BIP-039](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) 已经实现了这个机制。

好了，现在我们已经知道了在比特币中证明用户身份的是私钥。那么，比特币如何检查交易输出（和存储在里面的币）的所有权呢？

## 数字签名

在数学和密码学中，有一个数字签名（digital signature）的概念，算法可以保证：

1. 当数据从发送方传送到接收方时，数据不会被修改；
2. 数据由某一确定的发送方创建；
3. 发送方无法否认发送过数据这一事实。

通过在数据上应用签名算法（也就是对数据进行签名），你就可以得到一个签名，这个签名晚些时候会被验证。生成数字签名需要一个私钥，而验证签名需要一个公钥。签名有点类似于印章，比方说我做了一幅画，完了用印章一盖，就说明了这幅画是我的作品。给数据生成签名，就是给数据盖了章。

为了对数据进行签名，我们需要下面两样东西：

1. 要签名的数据
2. 私钥

应用签名算法可以生成一个签名，并且这个签名会被存储在交易输入中。为了对一个签名进行验证，我们需要以下三样东西：

1. 被签名的数据
2. 签名
3. 公钥

简单来说，验证过程可以被描述为：检查签名是由被签名数据加上私钥得来，并且公钥恰好是由该私钥生成。

>数据签名并不是加密，你无法从一个签名重新构造出数据。这有点像哈希：你在数据上运行一个哈希算法，然后得到一个该数据的唯一表示。签名与哈希的区别在于密钥对：有了密钥对，才有签名验证。但是密钥对也可以被用于加密数据：私钥用于加密，公钥用于解密数据。不过比特币并不使用加密算法。

在比特币中，每一笔交易输入都会由创建交易的人签名。在被放入到一个块之前，必须要对每一笔交易进行验证。除了一些其他步骤，验证意味着：

1. 检查交易输入有权使用来自之前交易的输出
2. 检查交易签名是正确的

如图，对数据进行签名和对签名进行验证的过程大致如下：

![the process of signing data and verifying signature](http://upload-images.jianshu.io/upload_images/127313-ec45a7fca855f2e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在来回顾一个交易完整的生命周期：

1. 起初，创世块里面包含了一个 coinbase 交易。在 coinbase 交易中，没有输入，所以也就不需要签名。coinbase 交易的输出包含了一个哈希过的公钥（使用的是
 **RIPEMD16(SHA256(PubKey))** 算法）

2. 当一个人发送币时，就会创建一笔交易。这笔交易的输入会引用之前交易的输出。每个输入会存储一个公钥（没有被哈希）和整个交易的一个签名。

3. 比特币网络中接收到交易的其他节点会对该交易进行验证。除了一些其他事情，他们还会检查：在一个输入中，公钥哈希与所引用的输出哈希相匹配（这保证了发送方只能花费属于自己的币）；签名是正确的（这保证了交易是由币的实际拥有者所创建）。

4. 当一个矿工准备挖一个新块时，他会将交易放到块中，然后开始挖矿。

5. 当新块被挖出来以后，网络中的所有其他节点会接收到一条消息，告诉其他人这个块已经被挖出并被加入到区块链。

6. 当一个块被加入到区块链以后，交易就算完成，它的输出就可以在新的交易中被引用。

## 椭圆曲线加密

正如之前提到的，公钥和私钥是随机的字节序列。私钥能够用于证明持币人的身份，需要有一个条件：随机算法必须生成真正随机的字节。因为没有人会想要生成一个私钥，而这个私钥意外地也被别人所有。

比特币使用椭圆曲线来产生私钥。椭圆曲线是一个复杂的数学概念，我们并不打算在这里作太多解释（如果你真的十分好奇，可以查看[这篇文章](http://andrea.corbellini.name/2015/05/17/elliptic-curve-cryptography-a-gentle-introduction/)，注意：有很多数学公式！）我们只要知道这些曲线可以生成非常大的随机数就够了。在比特币中使用的曲线可以随机选取在 0 与 2 ^ 2 ^ 56（大概是 10^77, 而整个可见的宇宙中，原子数在 10^78 到 10^82 之间） 的一个数。有如此高的一个上限，意味着几乎不可能发生有两次生成同一个私钥的事情。

比特币使用的是 ECDSA（Elliptic Curve Digital Signature Algorithm）算法来对交易进行签名，我们也会使用该算法。

## Base58

回到上面提到的比特币地址：1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa 。现在，我们已经知道了这是公钥用人类可读的形式表示而已。如果我们对它进行解码，就会看到公钥的本来面目（16 进制表示的字节）：

```
0062E907B15CBF27D5425399EBF6F0FB50EBB88F18C29B7D93
```

比特币使用 Base58 算法将公钥转换成人类可读的形式。这个算法跟著名的 Base64 很类似，区别在于它使用了更短的字母表：为了避免一些利用字母相似性的攻击，从字母表中移除了一些字母。也就是，没有这些符号：0(零)，O(大写的 o)，I(大写的i)，l(小写的 L)，因为这几个字母看着很像。另外，也没有 + 和 / 符号。

下图是从一个公钥获得一个地址的过程：

![get an address from a public key](http://upload-images.jianshu.io/upload_images/127313-6aa6cff5d863d496.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因此，上面提到的公钥解码后包含三个部分：

```
Version  Public key hash                           Checksum
00       62E907B15CBF27D5425399EBF6F0FB50EBB88F18  C29B7D93
```

由于哈希函数是单向的（也就说无法逆转回去），所以不可能从一个哈希中提取公钥。不过通过执行哈希函数并进行哈希比较，我们可以检查一个公钥是否被用于哈希的生成。

好了，所有细节都已就绪，来写代码吧。很多概念只有当写代码的时候，才能理解地更透彻。

## 私钥和公钥

ECDSA(Elliptic Curve Digital Signature Algorithm)，椭圆曲线数字签名算法将签名与验证进行了分离。

$$
y^2 = x^3 + ax + b
$$

异点相加和同点加倍

[比特币背后的数学](https://mp.weixin.qq.com/s/MigSmfzRfyw-ux1lm0iBhw)