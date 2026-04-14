---
title: 智能合约开发入门
date: 2026-04-13
description: 入门Solidity智能合约开发，理解以太坊智能合约核心概念
tags:
  - blockchain
  - solidity
  - smart-contract
  - ethereum
draft: false
permalink:
---

## 概述

智能合约（Smart Contract）是部署在区块链上的程序代码，能够自动执行、管理和记录合约相关的法律事件和行动。Solidity是以太坊最常用的智能合约开发语言。[^1]

## 什么是Solidity

Solidity是一种**静态类型**、**面向合约**的编程语言，专门为以太坊虚拟机（EVM）设计。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleStorage {
    uint256 private storedNumber;
    
    event NumberChanged(uint256 newValue);
    
    function store(uint256 _number) public {
        storedNumber = _number;
        emit NumberChanged(_number);
    }
    
    function retrieve() public view returns (uint256) {
        return storedNumber;
    }
}
```

## 开发环境

### Remix IDE（推荐入门）

基于Web的在线开发环境：[remix.ethereum.org](https://remix.ethereum.org)

### Hardhat（专业开发）

```bash
# 初始化项目
mkdir my-project && cd my-project
npm init -y
npm install --save-dev hardhat
npx hardhat init

# 项目结构
my-project/
├── contracts/       # 合约代码
├── scripts/         # 部署脚本
├── test/           # 测试代码
└── hardhat.config.js
```

## Solidity核心概念

### 数据类型

```solidity
// 值类型
bool public flag = true;
uint256 public amount = 100;  // 无符号整数
int256 public balance = -50;   // 有符号整数
address public owner = 0x123...;  // 160位地址
bytes32 public data = "hello"; // 固定大小字节数组

// 引用类型
string public name = "Alice";
uint[] public numbers = [1, 2, 3];
mapping(address => uint) public balances;
```

### 函数

```solidity
contract FunctionExample {
    // 访问控制
    function setValue(uint _x) public {
        value = _x;
    }
    
    // view: 不修改状态
    function getValue() public view returns (uint) {
        return value;
    }
    
    // pure: 不读取也不修改状态
    function add(uint a, uint b) public pure returns (uint) {
        return a + b;
    }
    
    // payable: 可以接收ETH
    function deposit() public payable {
        balance[msg.sender] += msg.value;
    }
}
```

### 事件（Events）

```solidity
event Transfer(address indexed from, address indexed to, uint256 value);

// 触发事件
emit Transfer(msg.sender, recipient, amount);
```

### 访问控制

```solidity
contract AccessControl {
    address public owner;
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    function sensitiveOperation() public onlyOwner {
        // 只有owner可以执行
    }
}
```

## ERC代币标准

### ERC-20（可替代代币）

```solidity
// 简化版ERC-20
contract MyToken {
    string public name = "My Token";
    string public symbol = "MTK";
    uint256 public totalSupply = 1000000;
    mapping(address => uint256) public balanceOf;
    
    mapping(address => mapping(address => uint256)) public allowance;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    constructor() {
        balanceOf[msg.sender] = totalSupply;
    }
    
    function transfer(address to, uint256 value) public returns (bool) {
        require(balanceOf[msg.sender] >= value);
        balanceOf[msg.sender] -= value;
        balanceOf[to] += value;
        emit Transfer(msg.sender, to, value);
        return true;
    }
    
    function approve(address spender, uint256 value) public returns (bool) {
        allowance[msg.sender][spender] = value;
        emit Approval(msg.sender, spender, value);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 value) public returns (bool) {
        require(balanceOf[from] >= value);
        require(allowance[from][msg.sender] >= value);
        balanceOf[from] -= value;
        allowance[from][msg.sender] -= value;
        balanceOf[to] += value;
        emit Transfer(from, to, value);
        return true;
    }
}
```

### ERC-721（非同质化代币/NFT）

```solidity
interface IERC721 {
    function transferFrom(address from, address to, uint256 tokenId) external;
    function safeTransferFrom(address from, address to, uint256 tokenId) external;
    function tokenURI(uint256 tokenId) external view returns (string);
}
```

## 完整示例：投票合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Ballot {
    struct Voter {
        uint256 weight;
        bool voted;
        address delegate;
        uint256 vote;
    }
    
    struct Proposal {
        bytes32 name;
        uint256 voteCount;
    }
    
    address public chairperson;
    mapping(address => Voter) public voters;
    Proposal[] public proposals;
    
    constructor(bytes32[] memory proposalNames) {
        chairperson = msg.sender;
        voters[chairperson].weight = 1;
        
        for (uint256 i = 0; i < proposalNames.length; i++) {
            proposals.push(Proposal({
                name: proposalNames[i],
                voteCount: 0
            }));
        }
    }
    
    function giveRightToVote(address voter) public {
        require(
            msg.sender == chairperson,
            "Only chairperson can give right to vote."
        );
        require(!voters[voter].voted);
        require(voters[voter].weight == 0);
        
        voters[voter].weight = 1;
    }
    
    function delegate(address to) public {
        Voter storage sender = voters[msg.sender];
        require(!sender.voted, "You already voted.");
        require(to != msg.sender, "Self-delegation is disallowed.");
        
        while (voters[to].delegate != address(0)) {
            to = voters[to].delegate;
            require(to != msg.sender, "Found loop in delegation.");
        }
        
        sender.voted = true;
        sender.delegate = to;
        Voter storage delegate_ = voters[to];
        
        if (delegate_.voted) {
            proposals[delegate_.vote].voteCount += sender.weight;
        } else {
            delegate_.weight += sender.weight;
        }
    }
    
    function vote(uint256 proposal) public {
        Voter storage sender = voters[msg.sender];
        require(sender.weight != 0, "Has no right to vote");
        require(!sender.voted, "Already voted.");
        
        sender.voted = true;
        sender.vote = proposal;
        
        proposals[proposal].voteCount += sender.weight;
    }
    
    function winningProposal() public view returns (uint256 winningProposal_) {
        uint256 winningVoteCount = 0;
        for (uint256 p = 0; p < proposals.length; p++) {
            if (proposals[p].voteCount > winningVoteCount) {
                winningVoteCount = proposals[p].voteCount;
                winningProposal_ = p;
            }
        }
    }
    
    function winnerName() public view returns (bytes32 winnerName_) {
        winnerName_ = proposals[winningProposal()].name;
    }
}
```

## 安全考虑

### 常见漏洞

```solidity
// 1. 重入攻击
// ❌ 不安全
function withdraw(uint amount) public {
    require(balances[msg.sender] >= amount);
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success);
    balances[msg.sender] -= amount;  // 转移后才减余额
}

// ✅ 安全：检查-生效-交互模式
function withdraw(uint amount) public {
    require(balances[msg.sender] >= amount);
    balances[msg.sender] -= amount;  // 先减余额
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success);
}
```

```solidity
// 2. 整数溢出
// Solidity 0.8+ 自动检查
uint256 public balance;
function deposit() public payable {
    balance += msg.value;  // 溢出时自动回退
}
```

### 安全工具

```bash
# 使用Mythril进行静态分析
npx myth analyze contracts/MyContract.sol

# 使用Slither进行安全扫描
pip install slither-analyzer
slither . --detect reentrancy

# 使用OpenZeppelin Contracts
npm install @openzeppelin/contracts
```

## 测试

### Hardhat测试示例

```solidity
// test/Counter.t.sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.28;

import { Counter } from "../contracts/Counter.sol";
import "forge-std/Test.sol";

contract CounterTest is Test {
    Counter public counter;
    
    function setUp() public {
        counter = new Counter();
    }
    
    function testInitialValueIsZero() public view {
        require(counter.x() == 0);
    }
    
    function testIncrement() public {
        counter.inc();
        require(counter.x() == 1);
    }
    
    function testIncrementBy(uint256 by) public {
        counter.incBy(by);
        require(counter.x() == by);
    }
}
```

## 部署

### 部署到本地网络

```bash
npx hardhat compile
npx hardhat node  # 启动本地网络

npx hardhat run scripts/deploy.js --network localhost
```

### 部署到测试网

```javascript
// hardhat.config.js
module.exports = {
  solidity: "0.8.0",
  networks: {
    sepolia: {
      url: "https://rpc.sepolia.org",
      accounts: [process.env.PRIVATE_KEY]
    }
  }
};
```

```bash
# 部署
PRIVATE_KEY=0x... npx hardhat run scripts/deploy.js --network sepolia
```

## 参考资料

[^1]: [Hardhat Tutorial - Writing and Compiling Contracts](https://hardhat.org/tutorial/writing-and-compiling-contracts)
