---
layout: post
title: 区块链技术探索(二), 打造我们自己的比特币
---

> 我们要用匠人的精神设计实现一款让人爱不释手，欲罢不能的产品, 这款产品的分类恰好是货币.

Talk is cheap, show me the code, 让我们从最简单的地方出发: 模仿中本聪的比特币.

## 1. 区块链的存储

在 [构造比特币的创世区块](/2017/05/11/7daystalk.html) 这篇文章中我们把生成的创世区块写到了一个叫 kyk-gens-block.dat 的文件中，这时候我们已经初步完成区块链的存储, 现在我们要为其加上索引, 在比特币的世界里, 使用了 [LevelDB](https://github.com/google/leveldb)<sup>[[1]](#ref-1)</sup> 来存储这些索引, 设计索引是一件比较精巧细致的活, 我们可以参考 [Bitcoin Core Data Storage](https://en.bitcoin.it/wiki/Bitcoin_Core_0.11_(ch_2):_Data_Storage)<sup>[[2]](#ref-2)</sup> 用自己的代码来一步步实现索引.

### 1.1 比特币的后端存储结构

![btc_backend_struct.png](/images/btc_backend_struct.png)

<b>Block index (leveldb)</b><sup>[[10]](#ref-10)</sup>

Key-value pairs

Inside the actual LevelDB, the used key/value pairs are:

```
   'b' + 32-byte block hash -> block index record. Each record stores:
       * The block header                                                   
       * The height.                                                       
       * The number of transactions.                                        
       * To what extent this block is validated.
       * In which file, and where in that file, the block data is stored.
       * In which file, and where in that file, the undo data is stored.
```

```
   'f' + 4-byte file number -> file information record. Each record stores:
       * The number of blocks stored in the block file with that number.
       * The size of the block file with that number ($DATADIR/blocks/blkNNNNN.dat).
       * The size of the undo file with that number ($DATADIR/blocks/revNNNNN.dat).
       * The lowest and highest height of blocks stored in the block file with that number.
       * The lowest and highest timestamp of blocks stored in the block file with that number.
```

```
   'l' -> 4-byte file number: the last block file number used.
```

```
   'R' -> 1-byte boolean ('1' if true): whether we're in the process of reindexing.
```

```
   'F' + 1-byte flag name length + flag name string -> 1 byte boolean ('1' if true, '0' if false): various flags that can be on or off. Currently defined flags include:
        * 'txindex': Whether the transaction index is enabled.
```

```
   't' + 32-byte transaction hash -> transaction index record. These are optional and only exist if 'txindex' is enabled (see above). Each record stores:
       * Which block file number the transaction is stored in.
       * Which offset into that file the block the transaction is part of is stored at.
       * The offset from the start of that block to the position where that transaction itself is stored.
```

<b>The UTXO set (chainstate leveldb)</b><sup>[[11]](#ref-11)</sup>

Key-value pairs

The records in the chainstate levelDB are:

```
   'c' + 32-byte transaction hash -> unspent transaction output record for that transaction. These records are only present for transactions that have at least one unspent output left. Each record stores:
       * The version of the transaction.
       * Whether the transaction was a coinbase or not.
       * Which height block contains the transaction.
       * Which outputs of that transaction are unspent.
       * The scriptPubKey and amount for those unspent outputs.


   'B' -> 32-byte block hash: the block hash up to which the database represents the unspent transaction outputs.
```

下面我们要做的工作就是将上面的 'b', 'f', 'l', 'R', 'F', 't', 'c', 'B' 这 8 种类型的 key/value pairs 从 Block index (leveldb) 数据库中正确的读取出来，并将对应的 value 解析出来. 作为热身我们先写一个简短的程序来熟悉下 leveldb.

#### 1.1.1 leveldb 热身

#### 1.1.2 读取 'b', 'f', 'l', 'R', 'F', 't', 'c', 'B' key/value pairs

### 1.2 参照比特币存储我们自己创建的创世区块

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

<b id="ref-8">[8]</b> [http://www.cnblogs.com/pandang/p/7279306.html](http://www.cnblogs.com/pandang/p/7279306.html) LevelDB C API 整理分类

<b id="ref-9">[9]</b> [https://github.com/bit-c/bitc](https://github.com/bit-c/bitc) bitc is a thin SPV bitcoin client 100% C code

<b id="ref-10">[10]</b> [https://en.bitcoin.it/wiki/Bitcoin\_Core\_0.11\_(ch\_2):\_Data\_Storage#Block\_index\_.28leveldb.29](https://en.bitcoin.it/wiki/Bitcoin_Core_0.11_(ch_2):_Data_Storage#Block_index_.28leveldb.29) Block index (leveldb)

<b id="ref-11">[11]</b> [https://en.bitcoin.it/wiki/Bitcoin\_Core\_0.11\_(ch\_2):\_Data\_Storage#The\_UTXO\_set\_.28chainstate\_leveldb.29](https://en.bitcoin.it/wiki/Bitcoin_Core_0.11_(ch_2):_Data_Storage#The_UTXO_set_.28chainstate_leveldb.29) The UTXO set (chainstate leveldb)
