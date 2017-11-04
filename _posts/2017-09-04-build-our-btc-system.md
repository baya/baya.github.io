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

<b>Block index (leveldb)</b><sup>[[9]](#ref-10)</sup>

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

<b>The UTXO set (chainstate leveldb)</b><sup>[[10]](#ref-10)</sup>

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

下面的这些数据是通过 `varint` 编码的，但是这个 `varint`<sup>[[13]](#ref-13)</sup> 和以前的 `varint`<sup>[[12]](#ref-12)</sup> 不一样,

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

'f' key 对应的 raw value 的格式如下所示, 它们都是以 varint<sup>[[13]](#ref-13)</sup> 的格式写入到 leveldb 中的.

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

上面的 30 其身是字符 '0' 的 Hex 表示值, 字符 '0' 用于表示 false, 字符 '1' 用于表示 true.

<b> 6. 读取 't' key</b>

程序: [read_t_key_debug.c](https://github.com/baya/mybt_coin/tree/master/debug/read_t_key_debug.c)

编译运行程序:

```bash
$ make dbg

$./debug/read_t_key_debug.out 4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b
```

输出:

```text
No record Found
```

貌似我下载的比特币钱包没有对交易进行索引.

<b> 7. 读取 'c' key<sup>[[14]](#ref-14)</sup></b>

`'c' + 32-byte transaction hash`

程序: [read_c_key_debug.c](https://github.com/baya/mybt_coin/tree/master/debug/read_c_key_debug.c)

编译运行程序:

```bash
$ make dbg

$./debug/read_c_key_debug.out 4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b
```

输出:

```text
ckey : 633ba3edfd7a7b12b27ac72c3e67768f617fc81bc3888a51323a9fb8aa4b1e5e4a
No record Found
```

<b> 8. 读取 'B' key</b>

程序: [read_cap_b_key_debug.c](https://github.com/baya/mybt_coin/tree/master/debug/read_cap_b_key_debug.c)

输出:

```text
Bkey : 42
raw value : 9acae02f84a810af603bc486996f273b9394ab1b308e56402d3f7171111520f9
```

得到的 raw value 不是一个 block hash, 不知道是什么原因.


### 1.2 以比特币为参照, 存储我们自己创建的创世区块

通过程序 [wallet2.c](https://github.com/baya/mybt_coin/tree/master/wallet2.c) 封装了初始化数据库, 读取写入 block 等工作.

编译运行程序:

```bash
$ make

$ make wallet2
```

初始化钱包,

```bash
./wallet2.out init
```

这条命令会在当前目录下构建一个钱包, 其中 blk00000.dat 里存储着我们自己的创世区块的原始数据.

```text
my_wallet2
└── blocks
    ├── blk00000.dat
    └── index
        ├── 000005.ldb
        ├── 000026.log
        ├── CURRENT
        ├── LOCK
        ├── LOG
        ├── LOG.old
        └── MANIFEST-000025
```

通过 block hash 查询创世区块,

```bash
./wallet2.out getblock 0000876c9ef8c1f8b2a3012ec1bdea7296f95ae21681799f8adf967f548bf8f3
```

查询结果:

```text
wVersion: 1
nHeight:  0
nStatus:  24
nTx:      1
nFile:    0
nDataPos: 8
nUndoPos: 0
Following is Block Header:
nVersion: 1
PrevHash : 0000000000000000000000000000000000000000000000000000000000000000
hashMerkleRoot : b76d27da4abf50387dd70f5d6cc7e4df1d54722631cbbfdd292463df77aa0dbd
nTime:    1504483200
nBits:    1f00ffff
nNonce:   65119
```

现在我们已经将我们自己的创世区块存储到了自己构建的数据库中了, 在后面我们将进行 `创建交易`, `生成新的区块` 等一系列工作.

## 2. 交易和区块的验证

首先我们看下 "tx" messages 的验证规则<sup>[[15]](#ref-15)</sup>,

### 2.1 交易的验证规则

1\. Check syntactic correctness;

检查的消息语法是否正确.

2\. Make sure neither in or out lists are empty;

输入和输出不能为空, 也就是说当你发起一笔交易, 你必须告诉比特币网络这笔交易的钱是从哪里来的，要到哪里去.

3\. Size in bytes <= MAX_BLOCK_SIZE;

消息的大小不能超过区块的最大容量, 因为消息中的交易最终是要被打包到区块中, 如果一笔交易的大小超过了区块的大小也就意味着这笔交易无法被打包到交易中去了.
目前区块的最大容量是 1M, 而基于纽约共识(New York Agreement)的比特币升级方案 SegWit2x<sup>[[18]](#ref-18)</sup> 的一项主要内容就是将区块的最大容量提升到 2M, 就是这么一个看似非常简单的扩容却要引起比特币网络的分裂, 对于我这种传统应用(中心化应用)的开发者来说这种分裂(hard fork)现象简直不可思议,
就好像我把数据库中的某个表增加了几个字段，或者把数据库从 mongodb 迁移到了 postgresql 就会造成我的应用分裂成两个应用一样，一个应用使用 mongodb, 另外一个应用使用 postgresql, 然后我的用户也跟着会分成两个派别，一种支持 mongodb, 一种支持 postgresql,
当然现实的情况是我的用户丝毫不关心我在后台使用了哪种数据库, 他们甚至可能都不知道世界上有数据库这种东西，他们只关心我的应用是否好用，是否安全稳定. 但是在比特币的世界里, 区块的大小确实是成为了一种规则，而我在 [构造比特币的创世区块](/2017/05/11/7daystalk.html) 这篇文章中
的第一段里也提到过:

> 在比特币的世界里, 规则塑造本尊，改变规则预示着分裂

分裂也不是什么坏事, 物竞天择，适者生存.

4\. Each output value, as well as the total, must be in legal money range;

交易的输出金额和总金额应该是一个合法的值, 比如不能为 0, 不能为负数, 比特币的最小单位是 0.00000001(一亿分之1), 称为 “1聪”, 输出的金额也不应该小于 “1聪”.

5\. Make sure none of the inputs have hash=0, n=-1 (coinbase transactions)

"tx" message 里不能包含 coinbase 交易(即造币交易), coinbase 交易只能通过旷工挖矿生成区块时产生, 旷工可以通过广播区块，将区块中包含的 coinbase 交易广播给其他比特币节点.

6\. Check that nLockTime <= INT_MAX<sup>[[19]](#ref-19)</sup>, size in bytes >= 100<sup>[[20]](#ref-20)</sup>, and sig opcount <= 2<sup>[[21]](#ref-21)</sup>


7\. Reject "nonstandard" transactions: scriptSig doing anything other than pushing numbers on the stack, or scriptPubkey not matching the two usual forms<sup>[[22]](#ref-22)</sup>

8\. Reject if we already have matching tx in the pool, or in a block in the main branch

如果交易池中已经有同一笔交易了，或者区块的主链(就是最长的那条链)上有同一笔交易了，我们就拒绝这条交易.

9\. For each input, if the referenced output exists in any other tx in the pool, reject this transaction.<sup>[[23]](#ref-23)</sup>

简单地讲就是交易的输入不能建立在一个已经消费的输出上. 为了更直观地理解这条规则，我画了一张图, 这条规则的主要目的就是为了防御双花攻击, 当然这种防御方式是建立在 POW 的基础上的.

![tx-chain](/images/simple-tx-chain.png)

上图是一个简化后的交易链,虽然实际情况上一笔交易可能会有多个 input 和多个 output, 但是简化后的交易链不影响我们分析规则 9.

按图中描述，如果我们要构造一笔新的交易，那么这笔新交易的 input 显然不能和 output1 对接, 因为 output1 已经被 Tx2 的 input2 所使用了, 这笔新的交易的 input 可以和 Tx2 的 output2 对接，因为 output2 没有被使用, 当然交易的构造者需要有能够解锁 output2 的私匙.

10\.For each input, look in the main branch and the transaction pool to find the referenced output transaction. If the output transaction is missing for any input, this will be an orphan transaction. Add to the orphan transactions, if a matching transaction is not in there already.

规则 9 说明了如果 input 的 output 已经被花费了，那么要拒绝这笔交易，而规则 10 表明了如果 input 对接的 output 所属的交易不存在(也可能是暂时证明不存在), 那么这笔交易要被纳入到孤交易集合(orphan transactions)中, 当然如果这笔交易已经被纳入到孤交易中去了，那么就不用做重复的纳入工作了. 如果后面能够重新找到对接的 output 所属的交易, 那么孤交易集合中的交易有可能被重新启用.

11\. For each input, if the referenced output transaction is coinbase (i.e. only 1 input, with hash=0, n=-1), it must have at least COINBASE_MATURITY (100) confirmations; else reject this transaction

如果交易的 input 对接的 output 来自 coinbase transaction, 那么这个 coinbase transaction 必须是经过了 100 次确认才能使用, 简单的说就是这个 coinbase transaction 所在的区块上面至少要有一个 100 个区块, 比如某个 coinbase transaction 所在的区块高度是 1000, 那么要必须等到主链的高度增加到 1100 个区块后才能使用这个 coinbase transaction 里的 output.

12\. For each input, if the referenced output does not exist (e.g. never existed or has already been spent), reject this transaction<sup>[[24]](#ref-24)</sup>

如果交易的 input 对接的 output 已经被花费或者根本不存在, 那么拒绝这笔交易, 这条规则可以说是由规则 9 和规则 10 结合而来.

13\. Using the referenced output transactions to get input values, check that each input value, as well as the sum, are in legal money range

14\. Reject if the sum of input values < sum of output values

在一笔交易中，它的 input value 是它解锁的对接 output 的 value, 如果你分析过 Txin 的数据结构，你就会发现 Txin 里并没有 value 或者 amout 之类的字段, Txin 的作用就是解锁对接交易的 output, 将对接交易的 output 的 value 释放给本交易的 output. 通过下面的图我们可以更加直观地理解规则 14,

![tx-input-output](/images/tx-input-output.png)

比特币中的 output 只能作为一个整体被解锁，一旦被解锁，它里面的 value 会被全部释放.

15\. Reject if transaction fee (defined as sum of input values minus sum of output values) would be too low to get into an empty block

16\. Verify the scriptPubKey accepts for each input; reject if any are bad

17\. Add to transaction pool<sup>[[17]](#ref-17)</sup>

18\. "Add to wallet if mine"

19\. Relay transaction to peers

20\. For each orphan transaction that uses this one as one of its inputs, run all these steps (including this one) recursively on that orphan

### 2.2 区块的验证规则

These messages hold a single block.<sup>[[15]](#ref-15)</sup>

1\. Check syntactic correctness

2\. Reject if duplicate of block we have in any of the three categories

3\. Transaction list must be non-empty

4\. Block hash must satisfy claimed nBits proof of work

5\. Block timestamp must not be more than two hours in the future

6\. First transaction must be coinbase (i.e. only 1 input, with hash=0, n=-1), the rest must not be

7\. For each transaction, apply "tx" checks 2-4

8\. For the coinbase (first) transaction, scriptSig length must be 2-100

9\. Reject if sum of transaction sig opcounts > MAX_BLOCK_SIGOPS

10\. Verify Merkle hash

11\. Check if prev block (matching prev hash) is in main branch or side branches. If not, add this to orphan blocks, then query peer we got this from for 1st missing orphan block in prev chain; done with block

12\. Check that nBits value matches the difficulty rules

13\. Reject if timestamp is the median time of the last 11 blocks or before

14\. For certain old blocks (i.e. on initial block download) check that hash matches known values

15\. Add block into the tree. There are three cases: 1. block further extends the main branch; 2. block extends a side branch but does not add enough difficulty to make it become the new main branch; 3. block extends a side branch and makes it the new main branch.

### 2.3 Wallet 和 Node 的交互

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

<b id="ref-10">[9]</b> [https://en.bitcoin.it/wiki/Bitcoin\_Core\_0.11\_(ch\_2):\_Data\_Storage#Block\_index\_.28leveldb.29](https://en.bitcoin.it/wiki/Bitcoin_Core_0.11_(ch_2):_Data_Storage#Block_index_.28leveldb.29) Block index (leveldb)

<b id="ref-11">[10]</b> [https://en.bitcoin.it/wiki/Bitcoin\_Core\_0.11\_(ch\_2):\_Data\_Storage#The\_UTXO\_set\_.28chainstate\_leveldb.29](https://en.bitcoin.it/wiki/Bitcoin_Core_0.11_(ch_2):_Data_Storage#The_UTXO_set_.28chainstate_leveldb.29) The UTXO set (chainstate leveldb)

<b id="ref-12">[11]</b> [https://github.com/bitcoin/bitcoin/blob/f914f1a746d7f91951c1da262a4a749dd3ebfa71/src/chain.h#L295](https://github.com/bitcoin/bitcoin/blob/f914f1a746d7f91951c1da262a4a749dd3ebfa71/src/chain.h#L295) Bitcoin Block index SerializationOp

<b id="ref-13">[12]</b> [https://en.bitcoin.it/wiki/Protocol_documentation#Variable_length_integer](https://en.bitcoin.it/wiki/Protocol_documentation#Variable_length_integer) Variable length integer

<b id="ref-14">[13]</b> 本文的 varint 的解码和编码方式如下, 如果没有特殊说明, 本文的 varint 都按下面的方式进行解码和编码.

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

<b id="ref-14">[14]</b> [https://github.com/bitcoin/bitcoin/blob/d4a42334d447cad48fb3996cad0fd5c945b75571/src/coins.h#L19](https://github.com/bitcoin/bitcoin/blob/d4a42334d447cad48fb3996cad0fd5c945b75571/src/coins.h#L19)  Serialized format of UTXO

<b id="ref-15">[15]</b> [https://en.bitcoin.it/wiki/Protocol_rules#.22tx.22_messages](https://en.bitcoin.it/wiki/Protocol_rules#.22tx.22_messages) Tx and Block message Validation

<b id="ref-16">[16]</b> [https://github.com/google/leveldb/blob/master/include/leveldb/c.h](https://github.com/google/leveldb/blob/master/include/leveldb/c.h) Leveldb C API

<b id="ref-17">[17]</b> Note that when the transaction is accepted into the memory pool, an additional check is made to ensure that the coinbase value does not exceed the transaction fees plus the expected BTC value (25BTC as of this writing)

<b id="ref-18">[18]</b> [https://github.com/btc1/bitcoin/tree/segwit2x](https://github.com/btc1/bitcoin/tree/segwit2x)

<b id="ref-19">[19]</b> nLockTime must not exceed 31 bits, as some clients will interpret it incorrectly

<b id="ref-20">[20]</b>  A valid transaction requires at least 100 bytes. If it's any less, the transaction is not valid

<b id="ref-21">[21]</b>  The number of signature operands in the signature (no, that is not redundant) for standard transactions will never exceed two

<b id="ref-22">[22]</b>  Note that this is not a hard requirement on clients

<b id="ref-23">[23]</b>  Note that this is not a hard requirement on clients. The network-enforced rule is that only one transaction spending a particular output can be in the blockchain, thus preventing double-spending. Technically miners can choose which one they want to put into the block they're working on as long as no other transaction has spent that output either previously in the blockchain, or in the same block. The in-memory transaction pool can technically be managed in whatever way the miner is willing to implement.

<b id="ref-24">[24]</b> This is the protection against double-spending

<b id="ref-25">[25]</b>  Note that when the transaction is accepted into the memory pool, an additional check is made to ensure that the coinbase value does not exceed the transaction fees plus the expected BTC value (25BTC as of this writing).

<b id="ref-26">[26]</b> [https://en.wikipedia.org/wiki/Double-spending](https://en.wikipedia.org/wiki/Double-spending) Double-spending
