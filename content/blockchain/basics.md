---
title: 区块链基础
date: 2026-04-11
description: 区块链技术入门——分布式账本、共识机制、加密货币与智能合约
tags:
  - blockchain
  - distributed-systems
  - cryptography
  - smart-contracts
draft: false
permalink:
---

## 1. 概述

**区块链**（Blockchain）是一种分布式账本技术（DLT），通过密码学方法将数据记录按时间顺序链接成不可篡改的链式结构。其核心特性包括：**去中心化**、**不可篡改**、**可追溯**和**共识机制**。

区块链技术最初作为比特币的底层技术而为人所知，如今已发展为支持智能合约、去中心化应用（DApp）的新型基础设施。

## 2. 核心数据结构

### 2.1 区块结构

区块是区块链的基本存储单元，包含两部分：

| 组成部分 | 说明 |
|----------|------|
| **区块头（Header）** | 包含版本号、前一区块哈希、时间戳、默克尔根、难度目标、随机数 |
| **区块体（Body）** | 包含交易数据列表 |

```
┌─────────────────────────────────┐
│           区块头                 │
│  Version │ PrevHash │ MerkleRoot│
│  Time   │ Bits     │ Nonce     │
├─────────────────────────────────┤
│           交易列表               │
│  TX1 │ TX2 │ TX3 │ ... │ TXn  │
└─────────────────────────────────┘
```

### 2.2 哈希指针

每个区块包含前一区块的哈希指针，形成链式结构：

$$
\text{Block}_n.\text{PrevHash} = \mathcal{H}(\text{Block}_{n-1})
$$

其中 $\mathcal{H}$ 为密码学哈希函数（如SHA-256）。

**不可篡改原理**：若修改某一区块的内容，其哈希值变化，后续所有区块的PrevHash均需修改，实际操作中不可能。

### 2.3 默克尔树

**默克尔树**（Merkle Tree）是一种二叉哈希树，用于高效验证交易完整性：

```
            Merkle Root
           /          \
      Hash(AB)     Hash(CD)
      /    \        /    \
   Hash(A) Hash(B) Hash(C) Hash(D)
     │       │       │       │
    TX1     TX2     TX3     TX4
```

优点：
- $O(\log n)$ 时间验证某交易是否存在
- 轻节点只需存储区块头即可验证交易

## 3. 分布式账本

### 3.1 账本模型

区块链是一种**只增不减**（append-only）的账本：

- **传统数据库**：支持CRUD（创建、读取、更新、删除）
- **区块链**：只支持CR（创建、读取），历史记录不可删除或修改

### 3.2 节点类型

| 节点类型 | 全量数据 | 参与共识 |
|----------|----------|----------|
| **全节点** | 完整区块链数据 | 是 |
| **轻节点** | 仅区块头 | 否 |
| **存档节点** | 全量数据+状态 | 是 |

### 3.3 网络传播

新交易/区块通过网络传播：

1. 节点验证交易/区块有效性
2. 广播给相邻节点
3. 邻居节点继续传播
4. 整个网络最终达成一致

## 4. 共识机制

**共识机制**是区块链网络中所有节点就账本状态达成一致的协议，是区块链系统的核心。

### 4.1 共识三要素

| 要素 | 说明 |
|------|------|
| **选择规则** | 如何从多个候选区块中选择"正确"链 |
| **激励措施** | 奖励诚实验证者，惩罚恶意行为 |
| **防Sybil攻击** | 防止攻击者通过创建多个身份控制网络 |

### 4.2 工作量证明（PoW）

**PoW**要求矿工消耗计算资源解决数学难题，最先解决者获得出块权：

**难题**：找到nonce使得区块头的SHA-256哈希值小于目标值

$$
\text{Find } nonce : \mathcal{H}(\text{header}) < \text{target}
$$

特性：
- **安全性**：攻击需掌握>50%算力
- **能耗高**：批评者认为浪费资源
- **去中心化**：矿池可能导致算力集中

### 4.3 权益证明（PoS）

**PoS**根据持有的加密货币数量和时间选择验证者：

$$
P(\text{选中}) = f(\text{stake}, \text{age})
$$

以太坊2.0采用PoS，验证者需质押32 ETH，诚实验证者获得奖励，恶意验证者受到惩罚（Slashing）。

**优势**：
- 能耗仅为PoW的约0.1%
- 安全性通过经济激励保证
- 无需专业矿机

### 4.4 拜占庭容错（BFT）

**BFT**类算法可在部分节点作恶时仍达成共识：

- **PBFT**（实用拜占庭容错）：适合联盟链，$f$个恶意节点下需 $3f+1$ 个总节点
- **Raft**：非拜占庭故障的共识算法，更简单易实现

### 4.5 其他共识机制

| 机制 | 代表项目 | 特点 |
|------|----------|------|
| DPoS | EOS | 持票人选举代表节点 |
| PoA | Polygon | 授权验证者 |
| DAG | IOTA | 无区块，有向无环图 |

## 5. 比特币与以太坊

### 5.1 比特币（Bitcoin）

**比特币**是第一个区块链应用，作为点对点电子现金系统：

**UTXO模型**：未花费交易输出

```
输入：引用上一交易的UTXO
输出：新交易的接收者 + 找零（发给自己的UTXO）
```

**货币政策**：
- 区块奖励每21万个区块（约4年）减半
- 总量上限：2100万BTC
- 当前区块奖励：6.25 BTC

### 5.2 以太坊（Ethereum）

**以太坊**是支持智能合约的区块链平台：

**账户模型**：

| 账户类型 | 说明 |
|----------|------|
| **外部拥有账户（EOA）** | 私钥控制，可发起交易 |
| **合约账户（CA）** | 合约代码控制，被EOA或其他合约调用 |

**以太坊虚拟机（EVM）**：执行智能合约的图灵完备虚拟机。

**Gas机制**：执行智能合约消耗Gas，防止无限循环：

$$
\text{Cost} = \text{GasUsed} \times \text{GasPrice}
$$

## 6. 智能合约

### 6.1 定义

**智能合约**（Smart Contract）是部署在区块链上、能够自动执行的程序：

```solidity
// Solidity示例：简单众筹合约
pragma solidity ^0.8.0;

contract Crowdfunding {
    address public owner;
    uint public goal;
    uint public deadline;
    uint public raised;
    mapping(address => uint) public contributions;
    
    constructor(uint _goal, uint _duration) {
        owner = msg.sender;
        goal = _goal;
        deadline = block.timestamp + _duration;
    }
    
    function contribute() public payable {
        require(block.timestamp < deadline, "Campaign ended");
        contributions[msg.sender] += msg.value;
        raised += msg.value;
    }
    
    function withdraw() public {
        require(msg.sender == owner, "Only owner");
        require(raised >= goal, "Goal not reached");
        payable(owner).transfer(address(this).balance);
    }
}
```

### 6.2 特性

| 特性 | 说明 |
|------|------|
| **确定性** | 相同输入产生相同输出 |
| **自动执行** | 满足条件自动触发 |
| **不可篡改** | 部署后代码不可更改 |
| **去中心化** | 无需中介，自执行 |

### 6.3 局限

- **Oracle问题**：无法直接获取链外数据
- **最大合约大小**：以太坊有Gas限制
- **不可升级**：除非预设升级机制

## 7. 区块链分类

### 7.1 按访问权限

| 类型 | 说明 | 代表 |
|------|------|------|
| **公有链** | 公开访问，无许可 | Bitcoin、Ethereum |
| **联盟链** | 多方授权访问 | Hyperledger Fabric、R3 Corda |
| **私有链** | 单个组织内部 | 企业内部使用 |

### 7.2 Layer 1 与 Layer 2

| 层级 | 说明 | 示例 |
|------|------|------|
| **Layer 1** | 底层区块链本身 | Ethereum、Bitcoin |
| **Layer 2** | 构建在L1之上的扩展方案 | Lightning Network、Polygon |

**Layer 2目的**：提升吞吐量，降低手续费。

## 8. 区块链应用场景

### 8.1 DeFi（去中心化金融）

| 应用 | 说明 |
|------|------|
| DEX | 去中心化交易所（Uniswap） |
| 借贷 | 抵押借贷（Aave） |
| 稳定币 | 锚定法币的加密货币 |

### 8.2 NFT

**NFT**（非同质化代币）用于表示独特数字资产的所有权：

```solidity
// ERC-721 NFT标准
interface IERC721 {
    function transferFrom(address from, address to, uint256 tokenId) external;
    function ownerOf(uint256 tokenId) external view returns (address);
}
```

### 8.3 供应链追踪

区块链可用于产品溯源：
- 食品安全追踪
- 药品防伪
- 奢侈品验证

### 8.4 数字身份

自主权身份（SSI）让用户掌控自己的身份数据。

## 9. 安全性考虑

### 9.1 51%攻击

攻击者控制超过50%算力/权益时可：
- 阻止新交易确认
- 逆转已确认交易（双花）

### 9.2 智能合约漏洞

| 漏洞 | 案例 |
|------|------|
| 重入攻击 | The DAO事件 |
| 整数溢出 | 多起DeFi攻击 |
| 闪电贷攻击 | 多起DeFi套利 |

### 9.3 隐私泄露

区块链的公开性意味着：
- 交易可追溯
- 地址与身份关联风险
- 需使用混币、零知识证明等隐私技术

## 10. 参考资料

[^1]: [Bitcoin Whitepaper](https://bitcoin.org/bitcoin.pdf)

[^2]: [Ethereum Documentation](https://ethereum.org/developers/docs/)

[^3]: [Blockchain: A Decentralized Consensus](https://dl.acm.org/doi/fullHtml/10.1145/3366370)

[^4]: [Stanford CS251: Bitcoin and Cryptocurrencies](https://cs251.stanford.edu/)

---

## 相关主题

- [[../distributed-systems/distributed-systems-overview|分布式系统概述]] — 共识算法基础
- [[../distributed-systems/consensus-algorithms|共识算法]] — Paxos、Raft等传统共识
- [[../security/cryptography|密码学基础]] — 哈希函数、数字签名
