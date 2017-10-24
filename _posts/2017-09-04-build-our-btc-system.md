---
layout: post
title: 区块链技术探索(二), 打造我们自己的比特币
---

## 0. 以匠人之心设计一款货币

这是一个推崇匠心的时代，人们相信用匠心做出的东西，是好东西，是值得信赖的东西。我既不懂经济学，也不懂货币理论, 我也不打算去学习经济学或者货币理论之类的东西，因为如果你要做一个产品的话，你最主要的精力应该是把产品做的非常棒, 能够自发地吸引(注意不是诱骗!)别人来使用你的产品, 
让别人用你做出来的东西用的很爽，而不是让你的产品去验证学院里的那些经济学理论, 当然你做出来的产品可能会符合经济学理论，也可能不符合，但是这些一点都不重要，重要的是你做的东西要有人用，更好的是有很多很多的人用, 所以你不必火急火燎地去买一堆经济和货币学之类的书来看, 这些书和书里面的理论
的唯一的作用就是拖延你的时间，浇灭你的激情, 你应该立马行动起来，开始做你的产品, 而中本聪发明的比特币就是你需要认真参考的模范, 你同时也可能要参考市面上很多其他类型的优秀的产品, 比如好用的电饭煲，自行车，手机，家具等等，因为让人爱不释手的产品都有一个共同点就是以人为本, 从好的产品中发掘人性, 而这些人性
是我们设计一款产品的灵感来源。

> 我们要用匠人的精神设计实现一款让人爱不释手，欲罢不能的产品, 这款产品的分类恰好是货币.

Talk is cheap, show me the code, 让我们从最简单的地方出发: 模仿中本聪的比特币.

## 1. 区块链的存储

在 [构造比特币的创世区块](/2017/05/11/7daystalk.html) 这篇文章中我们把生成的创世区块写到了一个叫 kyk-gens-block.dat 的文件中，这时候我们已经初步完成区块链的存储, 现在我们要为其加上索引, 在比特币的世界里, 使用了 [LevelDB](https://github.com/google/leveldb)<sup>[[1]](#ref-1)</sup> 来存储这些索引, 设计索引是一种比较精巧细致的活, 我们可以参考 [Bitcoin Core Data Storage](https://en.bitcoin.it/wiki/Bitcoin_Core_0.11_(ch_2):_Data_Storage)<sup>[[2]](#ref-2)</sup> 用自己的代码来一步步实现索引.

## 2. 交易的验证

## 3. P2P 网络

## 4. 参考资料

<b id="ref-1">[1]</b> [https://bitcoin.stackexchange.com/questions/28168/what-are-the-keys-used-in-the-blockchain-leveldb-ie-what-are-the-keyvalue-pair](https://bitcoin.stackexchange.com/questions/28168/what-are-the-keys-used-in-the-blockchain-leveldb-ie-what-are-the-keyvalue-pair) What are the keys used in the blockchain levelDB?

<b id="ref-2">[2]</b> [https://en.bitcoin.it/wiki/Bitcoin\_Core\_0.11\_(ch_2):\_Data\_Storage](https://en.bitcoin.it/wiki/Bitcoin_Core_0.11_(ch_2):_Data_Storage) Bitcoin Core Data Storage

<b id="ref-3">[3]</b> [https://bitcoin.stackexchange.com/questions/51387/how-does-bitcoin-read-from-write-to-leveldb/52167#52167](https://bitcoin.stackexchange.com/questions/51387/how-does-bitcoin-read-from-write-to-leveldb/52167#52167) How does Bitcoin read from/write to LevelDB

<b id="ref-4">[4]</b> [https://jeiwan.cc/posts/building-blockchain-in-go-part-3/](https://jeiwan.cc/posts/building-blockchain-in-go-part-3/) Building Blockchain in Go. Part 3: Persistence and CLI

<b id="ref-5">[5]</b> [https://github.com/google/leveldb/blob/master/doc/impl.md](https://github.com/google/leveldb/blob/master/doc/impl.md) The implementation of leveldb

<b id="ref-6">[6]</b> [https://en.bitcoin.it/wiki/Bitcoin\_Core\_0.11\_(ch\_4):\_P2P_Network](https://en.bitcoin.it/wiki/Bitcoin_Core_0.11_(ch_4):_P2P_Network) Bitcoin Core 0.11 (ch 4): P2P Network

<b id="ref-7">[7]</b> [https://bitcoin.org/en/developer-guide#term-utxo](https://bitcoin.org/en/developer-guide#term-utxo) Unspent Transaction Outputs (UTXOs)


