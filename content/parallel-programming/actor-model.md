---
title: Actor模型
date: 2026-04-10
description: Actor并发模型——Erlang/Elixir的并发哲学与消息传递机制
tags:
  - actor-model
  - concurrency
  - distributed-systems
draft: false
permalink:
---

## 1. 定义

**Actor模型**（Actor Model）是一种并发编程范式，由Carl Hewitt于1973年提出。其核心思想是：**通过消息传递来避免共享内存**——每个Actor拥有独立的私有状态，通过异步发送消息进行通信和协作。[^1]

在Actor模型中，所有计算都是通过**Actor之间的消息传递**完成的。与传统的共享内存并发模型不同，Actor之间不共享任何状态，每个Actor只能访问自己的局部状态。这从根本上避免了竞态条件和死锁问题。

Erlang/OTP是Actor模型最成功的工业级实现，支撑着WhatsApp、Discord、Ericsson交换机等高可靠系统。Elixir构建于Erlang VM之上，继承了这一并发哲学。

## 2. 核心概念

### 2.1 Actor是什么

Actor是Actor模型中的基本计算单元，具有以下特性：

| 特性 | 说明 |
|------|------|
| **封装状态** | 每个Actor拥有私有状态，外部无法直接访问 |
| **单线程执行** | Actor内部消息处理是串行的，无需担心竞态条件 |
| **消息驱动** | Actor通过接收和处理消息来响应事件 |
| **动态创建** | Actor可以创建新的Actor，形成层级结构 |

形式化地，一个Actor可以表示为以下状态机：

$$
\text{Actor} = (\text{state}, \text{mailbox}, \text{behavior})
$$

其中：
- $\text{state}$：Actor的私有状态
- $\text{mailbox}$：接收消息的队列
- $\text{behavior}$：处理消息的行为函数

### 2.2 消息传递机制

Actor之间的通信遵循以下规则：

1. **发送消息是异步的**——发送方将消息放入接收方的邮箱后立即返回，不阻塞等待
2. **每个Actor有独立地址**——通过地址（PID）标识和寻址
3. **消息按FIFO顺序处理**——邮箱中的消息按入队顺序串行处理
4. **消息传递是at-most-once语义**——消息可能丢失，不保证必达

```
Actor A (PID: 1234)
    │
    │  send(PID: 5678, {task, data})
    ▼
    ┌─────────────────────────────────┐
    │         Mailbox (队列)           │
    │  [{task, data}, {ping, self()}] │
    └─────────────────────────────────┘
              ▲
              │  dequeue()
              │
         Actor B (PID: 5678)
```

### 2.3 地址与 mailbox

每个Actor都有一个唯一的**地址（Address）**，在Erlang中称为PID（Process Identifier）。地址使得Actor之间的解耦成为可能——发送方只需要知道接收方的地址，而不需要了解接收方的物理位置。

**邮箱（Mailbox）**是Actor的消息队列，用于存储待处理的消息。邮箱是私有的，只能由所属Actor访问。

## 3. 为什么需要Actor模型

### 3.1 共享内存模型的困境

传统的线程+锁模型存在以下问题：

| 问题 | 描述 |
|------|------|
| **竞态条件** | 多个线程同时访问共享数据，导致数据不一致 |
| **死锁** | 多个线程相互等待对方持有的锁，形成循环等待 |
| **复杂度** | 锁的粒度、顺序、层级设计困难，容易出错 |
| **难以调试** | 竞态条件通常是概率性的，难以复现 |

考虑一个简单的计数器场景：

```cpp
// 共享内存模型：需要锁来保护
class Counter {
    std::mutex mtx;
    int count = 0;
public:
    void increment() {
        std::lock_guard<std::mutex> lock(mtx);  // 必须加锁
        count++;
    }
};
```

### 3.2 Actor模型的优势

Actor模型通过**避免共享状态**来规避上述问题：

1. **天然避免竞态条件**：每个Actor内部串行处理，无需锁
2. **降低复杂度**：只需关注消息协议，无需管理锁
3. **天然适合分布式**：消息传递与位置无关
4. **错误隔离**：Actor崩溃不会直接损坏其他Actor的状态

```cpp
// Actor模型：每个Actor拥有独立状态，无需共享锁
// C++风格的简单Actor实现

#include <thread>
#include <mutex>
#include <queue>
#include <condition_variable>
#include <memory>
#include <iostream>

class Actor {
public:
    using Message = std::function<void()>;
    
    Actor() : running_(true) {
        thread_ = std::thread([this] { run(); });
    }
    
    ~Actor() {
        stop();
    }
    
    // 发送消息（异步）
    void send(Message msg) {
        {
            std::lock_guard<std::mutex> lock(mtx_);
            mailbox_.push(std::move(msg));
        }
        cv_.notify_one();
    }
    
    // 获取地址
    void* getAddress() const {
        return const_cast<Actor*>(this);
    }
    
    void stop() {
        send([this]() { running_ = false; });
        if (thread_.joinable()) {
            thread_.join();
        }
    }

protected:
    virtual void handle(Message& msg) = 0;

private:
    void run() {
        while (running_) {
            Message msg;
            {
                std::unique_lock<std::mutex> lock(mtx_);
                cv_.wait(lock, [this] { return !mailbox_.empty() || !running_; });
                if (!running_ && mailbox_.empty()) break;
                msg = std::move(mailbox_.front());
                mailbox_.pop();
            }
            handle(msg);
        }
    }
    
    std::thread thread_;
    std::mutex mtx_;
    std::condition_variable cv_;
    std::queue<Message> mailbox_;
    bool running_;
};

// Counter Actor实现
class CounterActor : public Actor {
public:
    void increment(int delta) {
        send([this, delta]() {
            count_ += delta;
            std::cout << "Counter updated: " << count_ << std::endl;
        });
    }
    
    void get(std::function<void(int)> callback) {
        send([this, callback]() {
            callback(count_);
        });
    }
    
    void stop() {
        send([this]() { running_ = false; });
        Actor::stop();
    }

protected:
    void handle(Message& msg) override {
        msg();  // 执行消息携带的操作
    }

private:
    int count_ = 0;
    bool running_ = true;
};

int main() {
    auto counter = std::make_unique<CounterActor>();
    
    // 发送消息
    counter->increment(5);
    counter->increment(3);
    
    // 获取当前值（通过回调）
    counter->get([](int value) {
        std::cout << "Current value: " << value << std::endl;
    });
    
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    counter->stop();
    
    return 0;
}
```

上述示例展示了Actor模型的核心思想：**每个Actor拥有独立的状态，通过消息传递进行通信，消息处理是串行的**。

## 4. Erlang/Elixir实现

Erlang/OTP是Actor模型最成熟的工业级实现。在Erlang中，每个Actor称为一个**轻量级进程**（区别于操作系统进程），由Erlang VM管理，开销极低。

### 4.1 进程创建与消息传递

```elixir
# Elixir代码示例：Ping-Pong通信

defmodule PingPong do
  # 启动pong进程
  def start do
    pong_pid = spawn(PingPong, :pong, [])
    spawn(PingPong, :ping, [3, pong_pid])
  end
  
  # ping进程：发送ping消息后等待pong回复
  def ping(0, pong_pid) do
    send(pong_pid, :finished)
    IO.puts("Ping finished")
  end
  
  def ping(n, pong_pid) when n > 0 do
    send(pong_pid, {:ping, self()})
    receive do
      :pong ->
        IO.puts("Ping received pong")
    end
    ping(n - 1, pong_pid)
  end
  
  # pong进程：接收ping消息并回复
  def pong do
    receive do
      {:ping, ping_pid} ->
        IO.puts("Pong received ping")
        send(ping_pid, :pong)
        pong()
      :finished ->
        IO.puts("Pong finished")
    end
  end
end

# 运行
PingPong.start()
```

关键原语说明：

| 原语 | 说明 |
|------|------|
| `spawn(Module, Function, Args)` | 创建新进程，返回PID |
| `self()` | 获取当前进程的PID |
| `send(Pid, Message)` | 异步发送消息 |
| `receive do ... end` | 模式匹配接收消息 |

### 4.2 消息模式匹配

Erlang/Elixir的消息处理使用**模式匹配**，使消息处理既简洁又安全：

```elixir
# 计数器Actor，支持增量和查询
defmodule Counter do
  # 启动counter，返回其PID
  def start(initial \\ 0) do
    spawn(fn -> loop(initial) end)
  end
  
  # 主循环
  defp loop(count) do
    receive do
      {:increment, delta} ->
        loop(count + delta)
      
      {:decrement, delta} ->
        loop(count - delta)
      
      {:get, caller} ->
        send(caller, {:value, count})
        loop(count)
      
      :stop ->
        :ok
    end
  end
end

# 使用示例
counter = Counter.start(0)
send(counter, {:increment, 10})
send(counter, {:get, self()})

receive do
  {:value, v} -> IO.puts("Counter value: #{v}")  # 输出: Counter value: 10
end

send(counter, :stop)
```

模式匹配使得：
- 不同类型的消息可以有不同的处理逻辑
- 消息格式错误会被明确拒绝
- 无需复杂的条件判断

## 5. OTP行为

OTP（Open Telecom Platform）是Erlang的标准库，提供了**行为（Behavior）**抽象，封装了通用模式，避免重复实现。

### 5.1 GenServer

**GenServer**（Generic Server）是最核心的OTP行为，封装了client-server模式：

```elixir
# GenServer实现的Counter
defmodule CounterGenServer do
  use GenServer
  
  # 客户端接口
  def start_link(initial \\ 0) do
    GenServer.start_link(__MODULE__, initial, name: __MODULE__)
  end
  
  def increment(delta) do
    GenServer.cast(__MODULE__, {:increment, delta})
  end
  
  def value do
    GenServer.call(__MODULE__, :get)
  end
  
  # 回调实现
  def init(initial) do
    {:ok, initial}
  end
  
  # 同步调用处理
  def handle_call(:get, _from, state) do
    {:reply, state, state}
  end
  
  # 异步调用处理
  def handle_cast({:increment, delta}, state) do
    {:noreply, state + delta}
  end
  
  def terminate(_reason, _state) do
    :ok
  end
end

# 使用
{:ok, _pid} = CounterGenServer.start_link(0)
CounterGenServer.increment(5)
CounterGenServer.increment(3)
IO.puts(CounterGenServer.value())  # 输出: 8
```

**GenServer的回调函数**：

| 回调 | 说明 |
|------|------|
| `init/1` | 初始化状态 |
| `handle_call/3` | 处理同步调用（等待回复） |
| `handle_cast/2` | 处理异步调用（不等待回复） |
| `handle_info/2` | 处理非OTP消息（如exit信号） |
| `terminate/2` | 进程终止前清理 |

### 5.2 Agent

**Agent**是GenServer的简化封装，适合简单的状态管理：

```elixir
# Agent示例：简单的kv存储
{:ok, agent} = Agent.start_link(fn -> %{} end)

Agent.update(agent, fn map -> Map.put(map, :name, "Alice") end)
Agent.update(agent, fn map -> Map.put(map, :age, 30) end)

name = Agent.get(agent, fn map -> map[:name] end)
IO.puts(name)  # 输出: Alice

Agent.stop(agent)
```

### 5.3 Task

**Task**用于一次性异步执行：

```elixir
# Task示例
task = Task.async(fn ->
  # 模拟耗时操作
  Process.sleep(1000)
  42
end)

# 做其他事情...
result = Task.await(task)
IO.puts(result)  # 输出: 42
```

## 6. 容错机制

### 6.1 "Let it Crash"哲学

Erlang/OTP的核心设计哲学是**"Let it Crash"**——不要试图处理所有错误，而是让错误的进程崩溃，并让监督者负责恢复。[^2]

这与传统的企业级Java开发形成鲜明对比。在Java中，开发者通常需要捕获并处理每一种可能的异常；而在Erlang中，期望某些进程会失败，错误处理集中在监督层。

### 6.2 Linking与Monitoring

**Link**是两个进程之间的双向关联——当一个进程崩溃时，链接的另一个进程会收到exit信号：

```elixir
# Link示例
spawn_link(fn ->
  # 这个进程崩溃会导致当前进程也终止
  raise "Oops!"
end)
```

**Monitor**是单向的监控——监控者收到被监控进程的状态通知，不影响被监控进程：

```elixir
# Monitor示例
ref = Process.monitor(child_pid)

receive do
  {:DOWN, ^ref, :process, _pid, reason} ->
    IO.puts("Child exited with reason: #{inspect(reason)}")
end
```

### 6.3 监督树

**监督树（Supervision Tree）**是Erlang/OTP的层级错误处理机制：

```
        TopSupervisor
            │
    ┌───────┼───────┐
    │       │       │
Worker1  Worker2  SubSupervisor
                     │
                ┌────┴────┐
           Worker3    Worker4
```

```elixir
# Supervisor配置示例
children = [
  # 简单的一对一监督
  {Counter, 0},
  
  # 共享工作池
  {WorkerPool, []},
  
  # 子监督者
  %{
    id: SubSupervisor,
    start: {SubSupervisor, :start_link, []},
    type: :supervisor,
    children: [
      {Worker3, []},
      {Worker4, []}
    ]
  }
]

# 启动监督者
{:ok, pid} = Supervisor.start_link(children, strategy: :one_for_one)
```

**监督策略**：

| 策略 | 说明 |
|------|------|
| `:one_for_one` | 重启崩溃的子进程 |
| `:one_for_all` | 任一子进程崩溃，重启所有子进程 |
| `:rest_for_one` | 崩溃进程后面的进程重启 |
| `:simple_one_for_one` | 简化版，适合动态创建的子进程 |

### 6.4 实际应用：带监督的Counter

```elixir
# 完整的监督树示例
defmodule CounterSupervisor do
  use Supervisor
  
  def start_link(arg) do
    Supervisor.start_link(__MODULE__, arg, name: __MODULE__)
  end
  
  def init(_arg) do
    children = [
      {Counter, 0},  # 监督的worker
    ]
    
    # max_restarts: 3, max_seconds: 5
    # 5秒内最多重启3次，超过则终止整个监督树
    Supervisor.init(children, strategy: :one_for_one, max_restarts: 3, max_seconds: 5)
  end
  
  def value do
    # 获取当前计数值
    case Supervisor.which_children(__MODULE__) do
      [{_, pid, _, _}] -> GenServer.call(pid, :get)
      _ -> :error
    end
  end
end

# 启动
{:ok, _pid} = CounterSupervisor.start_link([])

# 即使Counter崩溃，也会被自动重启
```

## 7. 分布式Actor

### 7.1 位置透明性

Erlang/OTP的**位置透明性（Location Transparency）**意味着：无论Actor在本地还是远程，通信语法完全相同。[^3]

```elixir
# 本地进程
local_pid = spawn(Module, :func, [])

# 远程节点进程
remote_pid = spawn(:"node@host", Module, :func, [])

# 发送消息语法完全相同
send(local_pid, {:msg, data})
send(remote_pid, {:msg, data})
```

### 7.2 节点与集群

```elixir
# 节点A (在终端1启动)
iex --name node_a@127.0.0.1

# 节点B (在终端2启动)
iex --name node_b@127.0.0.1

# 在节点A中连接节点B
Node.connect(:"node_b@127.0.0.1")

# 在节点A中创建到节点B的Actor
remote_pid = spawn(:"node_b@127.0.0.1", Module, :func, [])
send(remote_pid, {:hello, "world"})
```

### 7.3 分布式消息传递

```elixir
# 分布式Ping-Pong
defmodule DistributedPingPong do
  def ping(pong_node, count) do
    pong_pid = {:pong, pong_node}
    
    Enum.each(1..count, fn _ ->
      send(pong_pid, {:ping, self()})
      receive do
        :pong -> IO.puts("Received pong")
      end
    end)
  end
end

# 在node_a上
 DistributedPingPong.ping(:"node_b@127.0.0.1", 10)
```

## 8. 其他实现

### 8.1 Akka (JVM)

Akka是JVM平台的Actor模型实现，使用Scala或Java开发：

```scala
// Akka示例 (Scala)
import akka.actor.{ Actor, ActorSystem, Props }

class PingActor extends Actor {
  def receive = {
    case msg: String =>
      println(s"Ping received: $msg")
      sender() ! "pong"
  }
}

val system = ActorSystem("MySystem")
val ping = system.actorOf(Props[PingActor], name = "ping")

ping ! "hello"  // 发送消息
```

**Akka vs Erlang/Elixir**：

| 特性 | Akka | Erlang/OTP |
|------|------|------------|
| 监督策略 | 相似 | 相似 |
| 消息传递 | Akka Serializer | 内部二进制 |
| 集群 | Akka Cluster | Phoenix/Redis |
| 生态系统 | JVM生态 | BEAM生态 |

### 8.2 Microsoft Orleans

Orleans是.NET平台的Actor框架，强调**虚拟Actor**概念：

```csharp
// Orleans示例 (C#)
public interface IHello : IGrainWithIntegerKey
{
    Task<string> SayHello(string greeting);
}

public class HelloGrain : Grain, IHello
{
    public Task<string> SayHello(string greeting)
    {
        return Task.FromResult($"Hello, {greeting}");
    }
}
```

**Orleans特点**：
- **虚拟Actor**：Actor无需显式创建，调用时自动激活
- **单线程执行**：同Akka，保证状态安全
- **位置透明**：与Erlang类似

### 8.3 各实现对比

| 实现 | 平台 | 特点 |
|------|------|------|
| **Erlang/OTP** | BEAM VM | 工业级成熟，电信级可靠性 |
| **Elixir** | BEAM VM | 语法简洁，Phoenix框架 |
| **Akka** | JVM | Scala/Java生态 |
| **Orleans** | .NET | 虚拟Actor，简化使用 |
| **Celluloid** | Ruby | 面向对象风格 |

## 9. 应用场景

### 9.1 电信系统

Erlang最初为Ericsson的交换机开发，支撑着：
- **Ericsson AXD301**：99.9999%可用性（每年< 31秒宕机）
- **WhatsApp**：数亿并发连接
- **Elixir/Phoenix**：实时通信系统

### 9.2 即时通讯与社交

| 公司 | 技术栈 | 规模 |
|------|--------|------|
| Discord | Elixir | 百万级并发用户 |
| WhatsApp | Erlang | 20亿+用户 |
| Ericsson | Erlang | 电信级 |

### 9.3 IoT与边缘计算

Actor模型适合IoT场景：
- **轻量级进程**：适合资源受限设备
- **位置透明**：适合分布式设备管理
- **容错机制**：适合不可靠网络环境

### 9.4 分布式数据库

| 数据库 | Actor应用 |
|--------|-----------|
| **Cassandra** | 节点间协调使用消息传递 |
| **Riak** | 分布式复制与冲突解决 |
| **CouchDB** | 文档同步 |

## 10. 参考资料

[^1]: [Learn You Some Erlang - The Hitchhiker's Guide to Concurrency](https://learnyousomeerlang.com/the-hitchhikers-guide-to-concurrency)

[^2]: [The Little Elixir & OTP Guidebook - Supervision Trees](https://livebook.manning.com/book/the-little-elixir-and-otp-guidebook/chapter-4)

[^3]: [Erlang Documentation - Distributed Programming](https://www.erlang.org/doc/system/distributed_prog)

---

## 相关主题

- [[../parallel-programming/concurrent-programming-basics|并发编程基础]]
- [[../distributed-systems/distributed-systems-overview|分布式系统概述]]
