---
layout: post
title: 区块链技术探索(二), 打造我们自己的比特币
---

> 让我们用匠人的精神设计实现一款让人爱不释手，欲罢不能的产品, 这款产品的分类恰好是货币.

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

下面我们要做的工作就是将上面的 'b', 'f', 'l', 'R', 'F', 't', 'c', 'B' 这 8 种类型的 key/value pairs 从 Block index (leveldb) 和 chainsate (leveldb) 数据库中正确的读取出来，并将对应的 value 解析出来. 作为热身我们先写一个简短的程序来熟悉下 leveldb.

#### 1.1.1 leveldb 热身

leveldb 中几乎所有的 c++ 接口都有对应的 c 接口, c 接口在 [leveldb/c.h](https://github.com/google/leveldb/blob/master/include/leveldb/c.h) 中定义.

在程序 [level_db_api_debug.c]() 中，我们对 leveldb 中的 `打开数据库`, `读`, `写`, `批量写`, `删除`, `迭代` 等 api 进行了实验, 程序中以 `leveldb`
开头的函数和数据结构都来自于 `leveldb/c.h`. 用于检查错误的宏: `check` 则来自于 dbg.h 文件，这个和 leveldb 没有关系。

```c
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
#include <stdint.h>
#include <leveldb/c.h>
#include "dbg.h"

int main()
{
    leveldb_t *db;                                                 /* db handler */
    leveldb_options_t *db_opts = leveldb_options_create();         /* 创建 db 相关的 options       */
    leveldb_options_set_create_if_missing(db_opts, 1);             /* 如果数据库不存在则创建一个数据库 */
    char *errptr = NULL;                                           /* 用于存储错误信息的指针, 如果为 NULL 则表示正常    */
    char *db_path = "/tmp/toy_data";                               /* 在 /tmp 路径下设置一个叫 toy_data 的 leveldb 数据库 */
    leveldb_writeoptions_t *write_opts = NULL;                     /* 写操作选项的指针 */
    leveldb_readoptions_t *read_opts = NULL;                       /* 读操作选项的指针 */
    leveldb_writebatch_t* wb = NULL;                               /* 批量写的指针    */
    leveldb_iterator_t *iter = NULL;                               /* 迭代器指针      */
    char *value = NULL;
    size_t vlen = 0;

    db = leveldb_open(db_opts, db_path, &errptr);                  /* 打开一个数据库   */
    check(errptr == NULL, "open db error: %s", errptr);            /* 检查打开或者创建数据库时是否出现错误 */
    write_opts = leveldb_writeoptions_create();                    /* 创建用于写操作的 options */
    char key1[] = {'a', 'b', 'c', '\0'};
    char val1[]  = {'1','2','3','\0'};
    leveldb_put(db, write_opts, key1, sizeof(key1), val1, sizeof(val1), &errptr);      /* 写入 key 为 "abc", value 为 "123" 的数据 */
    check(errptr == NULL, "write error: %s", errptr);

    char key2[] = {'f', 'o', 'o', '\0'};
    char val2[]  = {'b','a','r','\0'};
    leveldb_put(db, write_opts, key2, sizeof(key2), val2, sizeof(val2), &errptr);      /* 写入 key 为 "foo", value 为 "bar" 的数据 */
    check(errptr == NULL, "write error: %s", errptr);

    read_opts = leveldb_readoptions_create();                                 /* 创建用于读操作的 options */
    value = leveldb_get(db, read_opts, key1, sizeof(key1), &vlen, &errptr);   /* 读取 key 为 "abc" 的数据 */
    check(errptr == NULL, "get error: %s", errptr);
    printf("get value: %s from key: %s, value len: %zu\n", value, key1, vlen);

    value = leveldb_get(db, read_opts, key2, sizeof(key2), &vlen, &errptr);   /* 读取 key 为 "abc" 的数据 */
    check(errptr == NULL, "get error: %s", errptr);
    printf("get value: %s from key: %s, value len: %zu\n", value, key2, vlen);

    leveldb_delete(db, write_opts, "abc", 3, &errptr);  /* 删除 key 为 {'a','b','c'} 的 key/value pairs */
    leveldb_delete(db, write_opts, "abc", 4, &errptr);  /* 删除 key 为 {'a','b','c', '\0'} 的 key/value pairs */

    /*
      默认情况下 leveldb 的写操作是异步的，意思是写函数执行完以后，会立即返回而不是等待数据写入到了磁盘中才返回, 这样可以提高 leveldb 的写的性能
      现在我们设定写操作是同步的，这样写函数会等待数据写入到磁盘中以后才返回
     */
    leveldb_writeoptions_set_sync(write_opts, 1);
    
    /*
      创建批量写的容器, 批量写的容器还有 5 种对应的操作方法:
      1. destroy, leveldb_writebatch_destroy, 销毁容器，也就是回收容器所占的内存
      2. clear, leveldb_writebatch_clear, 清空容器里的元素
      3. put, leveldb_writebatch_put, 往容器里加元素，此时数据并没有写到磁盘中
      4. delete, leveldb_writebatch_delete, 删除容器里的某个元素
      5. iterate, leveldb_writebatch_iterate, 遍历容器里的元素
    */
    wb = leveldb_writebatch_create();             

    leveldb_writebatch_put(wb, "foo1", 5, "bar1", 5); /* 往容器 wb 里添加 key 为 "foo1", value 为 "bar1" 的数据 */
    leveldb_writebatch_put(wb, "foo2", 5, "bar2", 5); /* 往容器 wb 里添加 key 为 "foo2", value 为 "bar2" 的数据 */
    leveldb_writebatch_put(wb, "foo3", 5, "bar3", 5); /* 往容器 wb 里添加 key 为 "foo3", value 为 "bar3" 的数据 */
    leveldb_writebatch_put(wb, "foo4", 5, "bar4", 5); /* 往容器 wb 里添加 key 为 "foo4", value 为 "bar4" 的数据 */
    leveldb_writebatch_delete(wb, "foo3", 5);         /* 将 key 为 "foo3" 的元素从容器 wb 里删除掉 */
    leveldb_write(db, write_opts, wb, &errptr);       /* 将容器里的数据写入到磁盘中, 此时容器里没有 key 为 "foo3" 的这条数据 */


    leveldb_readoptions_set_verify_checksums(read_opts, 1); /* 对从文件系统读取的所有数据进行校验和验证, 默认不使用 */
    leveldb_readoptions_set_fill_cache(read_opts, 0);

    iter = leveldb_create_iterator(db, read_opts);         /* 创建迭代器 */
    leveldb_iter_seek_to_first(iter);                      /* 从第一条数据开始遍历 */

    while (leveldb_iter_valid(iter)) {
    	const char *key;
    	const char *val;
    	size_t klen;
    	size_t vlen;

    	key = leveldb_iter_key(iter, &klen);
    	val = leveldb_iter_value(iter, &vlen);

    	printf("key=%s value=%s\n", key, val);

    	leveldb_iter_next(iter);
    }

    leveldb_readoptions_destroy(read_opts);
    leveldb_iter_destroy(iter);
    leveldb_writebatch_destroy(wb);
    leveldb_writeoptions_destroy(write_opts);
    leveldb_options_destroy(db_opts);
    leveldb_close(db);
    return 0;
error:
    if(read_opts) leveldb_readoptions_destroy(read_opts);
    if(iter) leveldb_iter_destroy(iter);
    if(wb) leveldb_writebatch_destroy(wb);
    if(write_opts) leveldb_writeoptions_destroy(write_opts);
    if(db_opts) leveldb_options_destroy(db_opts);
    if(db) leveldb_close(db);
    return -1;
}

```

为了编译运行上面的程序，我们首先: `git clone git@github.com:baya/mybt_coin.git`, 然后 `cd mybt_coin`,

```bash
$ make

$ make dbg

$./debug/level_db_api_debug.out

get value: 123 from key: abc, value len: 4
get value: bar from key: foo, value len: 4
key=foo value=bar
key=foo1 value=bar1
key=foo2 value=bar2
key=foo4 value=bar4
```

#### 1.1.2 读取 'b', 'f', 'l', 'R', 'F', 't', 'c', 'B' key/value pairs

<b> 1. 读取 'b' key</b>

`'b' + 32-byte block hash`

程序: [read_b_key_debug.c](https://github.com/baya/mybt_coin/tree/master/debug/read_b_key_debug.c)

编译运行程序:

```bash
$ make dbg

$ ./debug/read_b_key_debug.out 0000000099c744455f58e6c6e98b671e1bf7f37346bfd4cf5d0274ad8ee660cb
```

其中 `read_b_key_debug.out` 程序接收一个 block hash 作为参数.

输出:

```text

bkey : 62cb60e68ead74025dcfd4bf4673f3f71b1e678be9c6e6585f4544c79900000000
raw value : 889271cd101d0100808cbf1199b74701000000a7c3299ed2475e1d6ea5ed18d5bfe243224add249cce99c5c67cc9fb00000000601c73862a0a7238e376f497783c8ecca2cf61a4f002ec8898024230787f399cb575d949ffff001d3a5de07f
wVersion: 150001
nHeight: 10000
nStatus: 29
nTx: 1
nFile: 0
nDataPos: 2318353
nUndoPos: 433223
Following is Block Header:
nVersion:1
PrevHash : 00000000fbc97cc6c599ce9c24dd4a2243e2bfd518eda56e1d5e47d29e29c3a7
hashMerkleRoot : 9c397f783042029888ec02f0a461cfa2cc8e3c7897f476e338720a2a86731c60
nTime:1238988213
nBits:1d00ffff
nNonce:2145410362

```

'b' key 的构造步骤: 首先将 block hash 的字节倒序得到: `cb60e68ead74025dcfd4bf4673f3f71b1e678be9c6e6585f4544c79900000000`, 然后将 'b' 的字节 0x62 作为首位与前者合并就得到了 'b' key.

'b' key 对应的 raw value 的格式如下图所示:

![bkey-format](/images/bkey-format.png)

下面的这些数据是通过 `varint` 编码的，但是这个 `varint`<sup>[[14]](#ref-14)</sup> 和以前的 `varint`<sup>[[13]](#ref-13)</sup> 不一样,

```c
    // wallet version
    int wVersion = 0;

    //! height of the entry in the chain. The genesis block has height 0
    int nHeight = 0;

    //! Verification status of this block.
    uint32_t nStatus = 0;

    //! Number of transactions in this block.
    //! Note: in a potential headers-first mode, this number cannot be relied upon
    unsigned int nTx = 0;

    //! Which # file this block is stored in (blk?????.dat)
    int nFile = 0;

     //! Byte offset within blk?????.dat where this block's data is stored
    unsigned int nDataPos = 0;

    //! Byte offset within rev?????.dat where this block's undo data is stored
    unsigned int nUndoPos = 0;
```

它们的编码和解码方式如下,

编码:

```c
size_t pack_varint(uint8_t *buf, int n)
{
    unsigned char tmp[(sizeof(n)*8+6)/7];
    int len=0;
    while(1) {
        tmp[len] = (n & 0x7F) | (len ? 0x80 : 0x00);
        if (n <= 0x7F)
            break;
        n = (n >> 7) - 1;
        len++;
    }

    kyk_reverse_pack_chars(buf, tmp, len+1);

    return len+1;
}
```

解码:

```c
size_t read_varint(const uint8_t *buf, size_t len, uint32_t *val)
{
    uint32_t n = 0;
    
    size_t i = 0;

    while(i < len) {
        unsigned char chData = *buf;
	buf++;
        n = (n << 7) | (chData & 0x7F);
        if (chData & 0x80) {
	    i++;
            n++;
        } else {
	    *val = n;
	    i++;
            return i;
        }
    }

    return 0;
}

```

<b> 2. 读取 'f' key</b>

`'f' + 4-byte file number`

程序: [read_f_key_debug.c](https://github.com/baya/mybt_coin/tree/master/debug/read_f_key_debug.c)

编译运行程序:

```bash
$ make dbg

$ ./debug/read_f_key_debug.out 1
```

输出:

```text

fkey : 6601000000
raw value : d703befd9860878acc1186a57e87804983ecc7e61b83eee7f21d
nBlocks: 11267 (number of blocks stored in file)
nSize: 134188256 (number of used bytes of block file)
nUndoSize: 16967313 (number of used bytes in the undo file)
nHeightFirst: 119678 (lowest height of block in file)
nHeightLast: 131273 (highest height of block in file)
nTimeFirst: 1303524251 (earliest time of block in file)
nTimeLast: 1308244381 (latest time of block in file)

```

'f' key 对应的 raw value 的格式如下所示, 它们都是以 varint<sup>[[14]](#ref-14)</sup> 的格式写入到 leveldb 中的.

```c

unsigned int nBlocks;      //!< number of blocks stored in file
unsigned int nSize;        //!< number of used bytes of block file
unsigned int nUndoSize;    //!< number of used bytes in the undo file
unsigned int nHeightFirst; //!< lowest height of block in file
unsigned int nHeightLast;  //!< highest height of block in file
uint64_t nTimeFirst;       //!< earliest time of block in file
uint64_t nTimeLast;        //!< latest time of block in file

```

<b> 3. 读取 </b> _'l'_ <b>key</b>

程序: [read_l_key_debug.c](https://github.com/baya/mybt_coin/tree/master/debug/read_l_key_debug.c)

编译运行程序:

```bash
$ make dbg

$ ./debug/read_l_key_debug.out
```

输出:

```text

lkey : 6c
raw value : 60000000
nFile: 96 (the last block file number used)

```

<b> 4. 读取 'R' key</b>

程序: [read_r_key_debug.c](https://github.com/baya/mybt_coin/tree/master/debug/read_r_key_debug.c)

编译运行程序:

```bash
$ make dbg

$ ./debug/read_r_key_debug.out
```

输出:

```text
Rkey : 52
raw value : 
nReindex: 0 (not reindexing)
```

<b> 5. 读取 'F' key</b>

`'F' + 1-byte flag name length + flag name string`

程序: [read_cap_f_key_debug.c](https://github.com/baya/mybt_coin/tree/master/debug/read_cap_f_key_debug.c)

编译运行程序:

```bash
$ make dbg

$ ./debug/read_cap_f_key_debug.out
```

输出:

```text
Fkey : 46077478696e646578
raw value : 30
txindex: false
```

<b> 6. 读取 't' key</b>

<b> 7. 读取 'c' key</b>

<b> 8. 读取 'B' key</b>

### 1.2 以比特币为参照, 存储我们自己创建的创世区块

## 2. 交易和区块的验证

## 3. RPC 服务

## 4. 钱包

## 5. P2P 网络

## 6. 参考资料

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

<b id="ref-12">[12]</b> [https://github.com/bitcoin/bitcoin/blob/f914f1a746d7f91951c1da262a4a749dd3ebfa71/src/chain.h#L295](https://github.com/bitcoin/bitcoin/blob/f914f1a746d7f91951c1da262a4a749dd3ebfa71/src/chain.h#L295) Bitcoin Block index SerializationOp

<b id="ref-13">[13]</b> [https://en.bitcoin.it/wiki/Protocol_documentation#Variable_length_integer](https://en.bitcoin.it/wiki/Protocol_documentation#Variable_length_integer) Variable length integer

<b id="ref-14">[14]</b> 本文的 varint 的解码和编码方式如下, 如果没有特殊说明, 本文的 varint 都按下面的方式进行解码和编码.

```c
size_t read_varint(const uint8_t *buf, size_t len, uint32_t *val)
{
    uint32_t n = 0;
    
    size_t i = 0;

    while(i < len) {
        unsigned char chData = *buf;
	buf++;
        n = (n << 7) | (chData & 0x7F);
        if (chData & 0x80) {
	    i++;
            n++;
        } else {
	    *val = n;
	    i++;
            return i;
        }
    }

    return 0;
}


size_t pack_varint(uint8_t *buf, int n)
{
    unsigned char tmp[(sizeof(n)*8+6)/7];
    int len=0;
    while(1) {
        tmp[len] = (n & 0x7F) | (len ? 0x80 : 0x00);
        if (n <= 0x7F)
            break;
        n = (n >> 7) - 1;
        len++;
    }

    kyk_reverse_pack_chars(buf, tmp, len+1);

    return len+1;
}

```
