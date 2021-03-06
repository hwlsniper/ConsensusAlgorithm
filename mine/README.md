# 比特币CPU挖矿、GPU挖矿、矿池及矿机挖矿技术原理

### 比特币挖矿原理

比特币的区块头，共含6个字段，如下：
* int32_t nVersion，4字节，版本号，一般固定不变，仅在升级时改变。
* uint256 hashPrevBlock，32字节，前一个区块的区块头哈希，由前一个区块决定。
* uint256 hashMerkleRoot，32字节，包含进区块的所有交易构造的Merkle根，调整区块中的交易次序、增删交易、或修改Coinbase交易时改变。
* uint32_t nTime，4字节，时间戳，后一个区块时间略早于前一个区块是被允许的，但必须在合理的时间区间，一般会直接使用机器当前时间戳。
* uint32_t nBits，4字节，挖矿难度，由全网决定，每2016个区块按算法重新调整。
* uint32_t nNonce，4字节，随机数，提供2^32种取值。即4,294,967,296。

其中nVersion、hashPrevBlock、nBits是固定的，其他hashMerkleRoot、nTime、nNonce为可变的。

比特币挖矿原理即，不断变更区块头中的可变值，使得对区块头做双重SHA256哈希，结果小于挖矿难度目标值。即：
SHA256D(BlockHeader) <  F(nBits)

其中SHA256D(BlockHeader)即对区块头做双重SHA256哈希，F(nBits)即按nBits计算的难度目标值。

### 算力的表示

1 H/S = 每秒一次运算
1 KH/S = 1000 H/S，即每秒1千次运算
1 MH/S = 1000 KH/S，即每秒100万次运算
1 GH/S = 1000 MH/S，即每秒10亿次运算
1 TH/S = 1000 GH/S，即每秒1万亿次运算
1 PH/S = 1000 TH/S，即每秒1000万亿次运算
1 EH/S = 1000 PH/S，即每秒100万万亿次运算

### CPU挖矿原理

CPU挖矿，即利用RPC接口setgenerate控制挖矿。
控制台输入setgenerate true 2，即开始挖矿，后边的数字表示代表的挖矿线程数，当然前提先完成同步数据。
由于单CPU运算SHA256D算力约为2 MH/S，因此nNonce提供的4字节搜索空间完全够用，即支持4G种取值。

###  GPU挖矿原理

GPU运算SHA256D算力约为200M-1G，nNonce提供4G搜索空间，如果仅调整nNonce取值，可以支持4秒左右。
因此可以调整nTime，每调整一次nTime，可以继续挖矿4秒。

GPU挖矿使用GETWORK协议，即挖矿程序和节点分离，也即挖矿部件与区块链数据分离。
GPU挖矿时代，使用GETWORK协议，使得挖矿程序与节点交互。

核心思路为：节点构造区块，将区块头数据交给挖矿程序，挖矿程序遍历nNonce进行挖矿。
验证合格交付给节点，节点提取nNonce和nTime验证区块，如果符合要求即向全网广播。
遍历结束将调用GETWORK，节点构造新区块，然后重复上述过程。

GPU经典挖矿驱动为cgminer，源码为https://github.com/ckolivas/cgminer。

GPU挖矿缺陷：GETWORK协议给挖矿程序提供的搜索空间为4G，结束后需再次调用GETWORK RPC接口。
矿机出现后，矿机算力已达10 TH/S，继续使用GETWORK协议将频繁调用RPC接口，显然不太合适。
因此需转向更高效的getblocktemplate协议。

### 矿池挖矿原理

矿工通过getblocktemplate协议与节点交互，或矿池采用stratum协议与矿工交互，即为矿池的两种典型搭建模式。

与getwork相比，getblocktemplate协议让矿工自行构造区块，因此使得节点与挖矿完全分离。
矿工拿到一系列数据后，开始挖矿：
1、构建coinbase交易。
2、coinbase交易放在交易列表之前，构建hashMerkleRoot。因coinbase、以及交易次序均可调整，因此hashMerkleRoot空间可以认为无限大。
因此getblocktemplate协议也使矿工获得了巨大的搜索空间。
3、构建区块头。
4、挖矿，即矿工可以在nNonce、nTime、hashMerkleRoot提供的搜索空间中涉及任意的挖矿策略。
5、上交数据，如果挖矿成功即提交给节点，由节点验证并广播。

getblocktemplate协议的问题：
1、矿工通过HTTP方式调用RPC接口向节点申请挖矿数据，因此网络中最新区块变动无法告知矿工，造成算力浪费。
2、每次调用getblocktemplate，节点都会返回1.5M左右数据，因频繁交互将因此增加大量成本。
Stratum协议将解决上述问题。

### Stratum协议

Stratum协议，采用主动分配任务的方式，也即矿池任何时候都可以给矿工分派任务。
对于矿工，如收到新任务，将无条件转向新任务。另外矿工也可以向矿池申请新任务。

最核心问题为，如何使得矿工获得更大的搜索空间。
如果仅矿工仅可改变nNonce和nTime，交互数据少但搜索空间不足。
如果允许矿工构造coinbase，搜索空间大但代价是需要将所有交易交给矿工，因此对矿池带宽要求较高。

Stratum协议巧妙解决了这个问题。即：
基于Merkler树的原理，无需将全部交易发给矿工，只需将构造hashMerkleroot所需的少数几个节点交给矿工即可。
同时将构造coinbase所需信息交给矿工，矿工可基于少数信息构造hashMerkleroot。
照此方式，如果包含N笔交易，仅需将log2(N)个hash值交给矿工。因此可大大降低交互的数据量。

矿池的核心即给矿工分派任务，统计工作量并分发收益。
矿池可以将区块难度分成更小的任务发给矿工，矿工完成任务提交矿池。
如果全网区块难度要求前70位为0，那么矿池可以给矿工分派难度为前30位0的任务，矿池再判断是否碰巧前70位都为0。

几个开源矿池：
PHP-MPOS：https://github.com/MPOS/php-mpos
node-open-mining-portal：https://github.com/zone117x/node-open-mining-portal
Powerpool：https://github.com/sigwo/powerpool

### 混合挖矿

混合挖矿，即某种币的挖矿挂靠在另一种币的链条上。辅链需要做针对性设计（如域名币和狗狗币）。
混合挖矿，使用AuxPOW协议实现。AuxPOW的实现得益于比特币Coinbase的输入字段。

经典的PoW区块，规定符合要求才算合格的区块。AuxPOW协议附加两个要求：
1、辅链区块的hash值必须内置于父链区块的Coinbase里。
2、父链区块的难度比较符合辅链的难度要求。

一般来说，父链的算力比较辅链大，满足父链难度要求的区块一定满足辅链的难度要求。
因此过去很多达不到父链难度要求的区块，可以达到辅链难度，可以在辅链获得收益。

### ASIC矿机

FPGA，Field-Programmable Gate Array，译为现场可编程门阵列。是在PAL、GAL、CPLD等可编程器件的基础上进一步发展的产物。
是作为专用集成电路(ASIC)领域中的一种半定制电路而出现的，既解决了定制电路的不足，又克服了原有可编程器件门电路数有限的缺点。
能用FPGA实现各种AISC、DSP和单片机。
FPGA作为挖矿硬件，对于ASIC来说属于必然的过度技术。

ASIC，Application Specific Integrated Circuits，即专用集成电路。
是指应特定用户要求和特定电子系统的需要而设计、制造的集成电路。


### 参考文章

* [区块链核心技术演进之路——挖矿演进](https://zhuanlan.zhihu.com/p/23558268)
* [#比特币挖矿part1# 挖矿算法](http://blog.csdn.net/jason_cuijiahui/article/details/76672775)
* [#比特币挖矿part2# 矿池协议](http://blog.csdn.net/jason_cuijiahui/article/details/72512988)
* [比特币挖矿机在中国的前世今生，矿机发展历程完整简史](http://8btc.com/article-1767-1.html)
* [三天就回本？起底比特币矿机逆天发展史](http://news.mydrivers.com/1/540/540008.htm)
* [比特币矿机：什么是ASIC矿机](http://www.cybtc.com/article-1410-1.html)
* [什么是X11算法？什么是ASIC矿机？](http://8btc.com/article-1989-1.html)
* [比特史记--建国录](http://www.jizhuomi.com/internet/346.html)
* [原来他们在寻找 FPGA和ASIC搞不定的算法，盘活全球的GPU算力](https://463982.kuaizhan.com/81/16/p428304546c52ad)
