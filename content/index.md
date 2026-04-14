---
title: About
date: 2026-04-13
description: About Metaphor
tags:
draft: false
permalink:
---
本站是一个wiki站点，它
- 使用[Quartz 4](https://quartz.jzhao.xyz/)构建
- 基于[Obsidian](https://obsidian.md/)的知识库

目前，内容主要包括

## 算法与数据结构

- **排序算法**：快速排序、归并排序、堆排序、插入排序、选择排序
- **搜索与图论**：BFS、DFS、**回溯法**、Dijkstra最短路、**Bellman-Ford算法**、**A\*搜索**、Floyd-Warshall算法、拓扑排序、强连通分量、桥与割点、二分图匹配、欧拉路径、最大流、**2-SAT**、**树链剖分**、**矩阵树定理**、**哈密顿路径与回路**、**弦图与区间图**
- **字符串算法**：KMP、字符串哈希、AC自动机、Manacher、Z函数、后缀数组、后缀自动机、**回文树（Eertree）**、**Lyndon分解**
- **数据结构**：**栈**、单调栈、队列、堆、哈希表、链表、字典树、并查集、线段树、树状数组、稀疏表、**Treap（随机化平衡树）**、**AVL树**、**B树与B+树**、**Link-Cut Tree（动态树）**、**红黑树**、**KD-Tree**、**可持久化数据结构**、**跳跃表（Skip List）**、**LRU缓存**、**LFU缓存**、**布隆过滤器（Bloom Filter）**、**Counting Bloom Filter（计数布隆过滤器）**
- **算法技巧**：二分查找、前缀和、递归分治、贪心算法、**动态规划优化**（单调队列、斜率优化、Knuth优化）、背包问题、位运算、**线性基**、**莫队算法**、**差分约束**、**WQS二分**、**模拟退火**、**第k顺序统计量**、**分数规划**、**扫描线**、**双指针**（滑动窗口、快慢指针、碰撞指针）、**离散化**（坐标压缩）
- **高阶算法**：**近似算法**（顶点覆盖、集合覆盖、TSP）、**随机算法**（随机化快排、Quickselect、Karger最小割）、**在线算法**（缓存置换、竞争比分析）

## 计算几何

- **基础几何**：凸包、点在多边形内判定、线段相交、**旋转卡壳**

## 博弈论

- **博弈基础**：SG函数、Nim游戏、博弈树

## 数学基础

- **数论**：最大公约数、扩展欧几里得、快速幂、模运算、中国剩余定理、素数筛法、线性筛

- **组合数学**：排列组合、二项式定理、容斥原理、抽屉原理、**斯特林数**、**贝尔数**、**Burnside引理**、**生成函数**

- **概率与期望**：概率DP、期望计算、贝叶斯推断

- **FFT与多项式**：快速傅里叶变换（FFT）、数论变换（NTT）、多项式乘法

- **线性代数**：**线性基**、高斯消元

## 人工智能与机器学习

- **机器学习基础**：[[machine-learning/machine-learning-fundamentals|机器学习基础]] — 监督学习、无监督学习，强化学习
- **深度学习基础**：[[machine-learning/deep-learning-basics|深度学习基础]] — 神经网络、反向传播、CNN、RNN、Transformer
- **自然语言处理**：[[machine-learning/nlp-basics|NLP基础]] — 分词、词嵌入、Seq2Seq、Attention、Transformer
- **计算机视觉**：[[machine-learning/cv-basics|CV基础]] — CNN、目标检测（YOLO、Faster R-CNN）、图像分割（U-Net、Mask R-CNN）
- **AI工程**：[[machine-learning/llm/rag|RAG检索增强生成]] — 文档分块、嵌入模型、混合检索、重排序、RAGAS评估
- **MCP协议**：[[machine-learning/llm/mcp-protocol|MCP协议]] — Model Context Protocol、AI代理工具调用标准化、Tools/Resources/Prompts、JSON-RPC传输
- **Agentic AI**：[[machine-learning/llm/agentic-ai|代理式人工智能]] — 目标导向自主系统、工具调用、多智能体协作、ReAct/Plan-and-Execute模式
- **向量数据库**：[[machine-learning/llm/vector-database|向量数据库]] — ANN算法（HNSW/IVF）、距离度量、pgvector/Pinecone/Milvus选型
- **LLM评估**：[[machine-learning/llm/llm-evaluation|LLM评估]] — Faithfulness、Answer Relevancy、RAGAS框架、A/B测试

## 分布式系统

- **分布式系统基础**：[[distributed-systems/distributed-systems-overview|分布式系统基础]] — CAP定理、一致性模型、共识算法、数据复制
- **共识算法**：[[distributed-systems/consensus-algorithms|共识算法]] — Paxos、Raft、ZAB、PBFT拜占庭容错
- **分布式事务**：[[distributed-systems/distributed-transactions|分布式事务]] — 2PC、TCC、Saga、本地消息表、Seata框架
- **消息队列**：[[distributed-systems/message-queue|消息队列系统]] — RabbitMQ、Kafka、可靠性保证
- **分布式缓存**：[[distributed-systems/distributed-cache|分布式缓存]] — Redis集群、缓存策略（Cache-Aside/Write-Through）、缓存穿透/击穿/雪崩、内存优化
- **事件溯源**：[[distributed-systems/event-sourcing|事件溯源与CQRS]] — Event Store、Event Replay、CQRS模式、审计日志
- **分布式追踪**：[[distributed-systems/distributed-tracing|分布式追踪]] — OpenTelemetry、Jaeger、链路追踪、SpanContext、OTLP协议
- **OpenTelemetry生产实践**：[[distributed-systems/opentelemetry-at-scale|OpenTelemetry规模化]] — Collector架构、采样策略、上下文传播、基数管理、成本控制

## 计算机基础

- **计算机组成原理**：[[computer-architecture/computer-architecture-basics|CPU组成]]、指令系统、内存层次、Cache、虚拟内存、流水线
- **操作系统**：进程与线程、**[[operating-systems/process-scheduling|调度算法]]**、内存管理、死锁、文件系统
- **实时系统**：[[real-time-systems/basics|实时系统]] — 硬/软实时、RTOS、调度算法（Rate Monotonic、EDF）、优先级反转与优先级继承
- **并行与并发编程**：[[parallel-programming/concurrent-programming-basics|并发编程基础]] — 线程与进程、竞态条件、同步原语（锁、信号量、屏障）、死锁、内存可见性、C++并发编程
- **GPU并行计算**：[[parallel-programming/cuda|CUDA基础]] — GPU架构、线程层次（Grid/Block/Warp）、内存模型、Kernel编程
- **CPU多线程**：[[parallel-programming/openmp|OpenMP基础]] — fork-join模型、parallel/for指令、数据作用域、归约、同步

## 计算理论

- **自动机与复杂性**：[[theory-of-computation/automata-and-complexity|计算理论]] — 形式语言、DFA/NFA、正则表达式、乔姆斯基体系、图灵机、停机问题、P/NP/NP-complete复杂性、归约与莱斯定理

## 编程范式

- **函数式编程**：[[functional-programming/basics|函数式编程基础]] — 纯函数、不可变性、高阶函数、函数组合、Map/Filter/Reduce、柯里化、惰性求值

## 编程语言进阶

- **Python进阶**：[[programming-languages/python-advanced|Python进阶]] — 装饰器、元编程、异步编程（asyncio）、上下文管理器
- **Rust基础**：[[programming-languages/rust|Rust基础]] — 所有权系统、借用检查器、生命周期、内存安全
- **C++模板元编程**：[[programming-languages/cpp-templates|C++模板]] — 模板特化、SFINAE、类型萃取、可变参数模板
- **TypeScript基础**：[[programming-languages/typescript|TypeScript基础]] — 类型系统、泛型、接口、联合类型、条件类型

## 计算机网络

- **计算机网络**：[[networks/tcp-ip-protocol-stack|TCP/IP协议栈深入]] — 滑动窗口、拥塞控制
- **DNS深入解析**：[[networks/dns-deep-dive|DNS深入]] — DNS层级结构、解析流程、记录类型、DNSSEC、DoH/DoT、zone文件
- **CDN内容分发网络**：[[networks/cdn|CDN]] — 工作原理、边缘缓存、Anycast路由、边缘计算、安全功能

## Web技术

- **HTTP协议**：[[web/http|HTTP协议详解]] — 请求响应结构、状态码、缓存机制、HTTPS安全、HTTP/2与HTTP/3新特性
- **REST API设计**：[[web/rest-api-design|REST API设计]] — RESTful原则、HTTP方法语义、API版本管理、认证授权、限流策略
- **GraphQL**：[[web/graphql|GraphQL基础]] — Schema定义、Resolver、分页、错误处理、N+1问题与DataLoader、**Federation超图与子图设计**

## 前端开发

- **CSS布局**：[[frontend/css-layout|CSS布局]] — Flexbox与Grid核心概念、容器属性，项目属性、实战模式、性能优化
- **浏览器工作原理**：[[frontend/browser-internals|浏览器内部机制]] — 多进程架构、渲染管线、事件循环（宏任务/微任务）、requestAnimationFrame、渲染优化
- **React Hooks**：[[frontend/react-hooks|React Hooks基础]] — useState、useEffect、useCallback、useMemo、useRef、useContext、自定义Hooks
- **React进阶**：[[frontend/react-advanced|React进阶]] — Concurrent Mode、Suspense深入、useTransition、useDeferredValue、性能优化模式
- **前端构建工具**：[[frontend/build-tools|前端构建工具]] — Webpack模块联邦、Vite原理、Turbopack、构建优化实践
- **React服务端组件**：[[frontend/react/server-components|React Server Components]] — 服务端优先架构、流式渲染、Suspense边界、Next.js App Router
- **Vue基础**：[[frontend/vue-basics|Vue基础]] — 响应式系统、组件系统、Composition API、Vue Router、Pinia状态管理
- **Web无障碍**：[[accessibility/wcag|Web无障碍与WCAG]] — POUR原则、可感知性、可操作性、可理解性、WCAG 2.1/2.2、屏幕阅读器
- **微前端架构**：[[frontend/micro-frontends|微前端]] — Turborepo微前端支持、Module Federation、Webpack联邦模块、纵向拆分策略

## 数据库系统

- **数据库基础**：关系模型、SQL、索引原理 — B+树、InnoDB、查询优化、MVCC、**事务隔离级别**（读未提交、读已提交、可重复读、串行化）
- **NoSQL数据库**：[[databases/nosql|NoSQL数据库]] — MongoDB、Redis、Cassandra、CAP定理、BASE语义
- **LSM树**：[[databases/lsm-tree|LSM树]] — MemTable、SSTable、Leveled Compaction、RocksDB/LevelDB应用
- **数据库设计**：[[databases/database-design|数据库设计原则]] — 函数依赖、范式（1NF/2NF/3NF/BCNF）、反规范化技术

## 信息安全

- **密码学基础**：对称加密（AES/DES）、非对称加密（RSA/ECC）、混合加密、数字签名
- **哈希函数**：MD5、SHA系列、密码存储
- **Web安全**：XSS、CSRF、SQL注入、中间人攻击
- **TLS/SSL深入**：[[security/tls-deep-dive|TLS深入]] — TLS 1.2/1.3握手、证书链验证、OCSP/CRL撤销、mTLS

## 区块链

- **区块链基础**：[[blockchain/basics|区块链基础]] — 分布式账本、哈希指针、默克尔树、共识机制（PoW/PoS/BFT）、以太坊
- **智能合约开发**：[[blockchain/smart-contracts|智能合约入门]] — Solidity基础、合约结构、ERC-20/ERC-721代币、安全考虑

## 服务网格

- **服务网格**：[[distributed-systems/service-mesh|服务网格]] — Istio vs Linkerd对比、Ambient Mesh、流量管理、安全模型、可观测性

## WebAssembly

- **WebAssembly基础**：[[webassembly/basics|WebAssembly入门]] — 边缘计算（Cloudflare Workers）、插件系统（安全沙盒）、WASI、Component Model

## Serverless

- **Serverless基础**：[[serverless/basics|Serverless入门]] — FaaS/BaaS、冷启动优化、AWS Lambda、Cloudflare Workers、阿里云函数计算

## 新兴技术

- **量子计算基础**：[[quantum-computing/basics|量子计算入门]] — 量子比特、量子门（Hadamard、CNOT）、量子纠缠、Shor算法、Grover算法、后量子密码学
- **边缘计算**：[[edge-computing/basics|边缘计算架构]] — Cloudflare Workers、Lambda@Edge、Vercel Edge Functions、边缘缓存、边缘-云混合架构

## API安全

- **API安全**：[[security/api-security|API安全最佳实践]] — JWT认证（RS256 vs HS256）、OAuth 2.0 Flows（Authorization Code + PKCE）、Token刷新策略、限流算法

## 软件工程

- **设计模式**：[[software-engineering/design-patterns|设计模式]] — 单例、工厂、观察者、策略等GoF模式
- **系统设计**：[[software-engineering/system-design|系统设计]] — 负载均衡、缓存、分片、高可用架构
- **微服务架构**：[[software-engineering/microservices|微服务架构]] — 服务拆分、API网关、熔断器、Saga模式
- **持久化执行**：[[software-engineering/durable-execution|持久化执行]] — Temporal、Inngest、工作流持久化、检查点、自动重试

## 形式化验证

- **形式化验证基础**：[[formal-verification/basics|形式化验证入门]] — TLA+、模型检验、并发系统验证、PlusCal算法语言

## 平台工程

- **内部开发者平台**：[[platform-engineering/internal-developer-platform|IDP]] — Backstage、Golden Paths、自助服务、平台编排、Service Catalog

## 系统软件

- **编译器原理**：[[compilers/lexer|词法分析]] — Token、正则表达式、有限自动机
- **编译器原理**：[[compilers/parser|语法分析]] — 上下文无关文法、递归下降、AST构建
- **链接器与加载器**：[[system-software/linkers-loaders|链接器与加载器]] — ELF格式、符号解析、动态链接、重定位
- **内存管理**：[[system-software/memory-management|内存管理]] — Arena/Stack/Pool分配器、Slab分配器、线程安全、碎片管理
- **垃圾回收**：[[system-software/garbage-collection|垃圾回收算法]] — 标记-清除、标记-压缩、复制算法、分代回收、三色标记
- **缓存优化**：[[system-software/cache-optimization|缓存优化]] — CPU缓存层次、Cache-oblivious算法、矩阵转置/乘法/FFT应用

## 开发工具

- **Git**：[[tools/git|Git教程]] — 版本控制、分支管理、远程协作
- **Git进阶**：[[tools/git-advanced|Git进阶技巧]] — Rebase、Submodule、Hooks、Reflog、Worktree
- **Linux**：[[tools/linux|Linux基础]] — 文件系统、Shell命令、进程管理、网络配置
- **Vim**：[[tools/vim|Vim编辑器教程]] — 模态编辑、文本对象、宏、寄存器、配置
- **正则表达式**：[[tools/regex|正则表达式]] — 基础语法、量词、边界、分组、零宽断言、C++ regex
- **Docker**：[[tools/docker|Docker教程]] — 容器化、镜像构建、Docker Compose
- **Kubernetes**：[[tools/kubernetes|Kubernetes基础]] — Pod、Deployment、Service、存储与网络
- **Kubernetes进阶**：[[tools/kubernetes-advanced|Kubernetes进阶]] — 高级调度、存储、安全、Helm、Operator、自动扩缩容
- **CI/CD**：[[devops/cicd|CI/CD基础]] — 持续集成、持续部署、流水线设计、GitHub Actions
- **Shell脚本**：[[tools/shell-scripting|Shell脚本指南]] — Bash编程、函数、错误处理、最佳实践
- **Terraform**：[[tools/terraform|Terraform基础]] — HCL配置语言、Provider、State管理、Module设计、远程后端
- **Ansible**：[[tools/ansible|Ansible自动化]] — 主机清单、Playbook、Role、模块、无代理架构、幂等性
- **监控**：[[tools/monitoring|Prometheus+Grafana]] — 指标采集、PromQL查询、告警规则、Dashboard创建

## 考试与刷题

- **CCF-CSP题目解析**：模拟、栈、队列、哈希表、日期处理、表达式求值等（[[CCF-CSP/CSP201312B|ISBN号码矮、《CCF-CSP/CSP201609B|火车票》、《CCF-CSP/CSP201903B|二十四点》、「CCF-CSP/CSP202006B|稀疏向量》等）
- **LeetCode经典题解**：
  - 双指针/滑动窗口：[[Leetcode/leetcode1|LeetCode 1 - 两数之和]]、「Leetcode/leetcode15|LeetCode 15 - 三数之和」]、「Leetcode/leetcode3|LeetCode 3 - 无重复字符最长子串]]
  - 二分查找：[[Leetcode/leetcode153|LeetCode 153 - 旋转数组最小值]]
  - 动态规划：[[Leetcode/leetcode70|LeetCode 70 - 爬楼梯]]、「Leetcode/leetcode322|LeetCode 322 - 零钱兑换]]
  - 图论/拓扑排序：[[Leetcode/leetcode46|LeetCode 46 - 全排列]]、「Leetcode/leetcode207|LeetCode 207 - 课程表]]
  - 单调栈：[[Leetcode/leetcode739|LeetCode 739 - 每日温度]]、「Leetcode/leetcode84|LeetCode 84 - 柱状图最大矩形]]
  - DFS/BFS：[[Leetcode/leetcode200|LeetCode 200 - 岛屿数量]]
