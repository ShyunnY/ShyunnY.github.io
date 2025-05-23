---
date: '2025-05-23T21:17:08+08:00'
draft: false
title: '分布式算法-Raft「3」'
categories: ['分布式']
tags: ['raft'] 
summary: "raft 日志复制流程"
---

Raft 中的日志是一个 **按顺序排列的指令列表**，每条日志记录都表示客户端的一次请求日志，本质上就是一系列状态变更命令。每一个日志条目一般包括三个属性：index，term，command。这些日志会先写入日志存储 `LogStore` 组件，然后才提交给状态机。格式通常如下：

* index: 这条日志在整个日志列表中的位置
* term: 这条日志在哪个 term 被写入
* command: 来自客户端的操作（如 key/value 设置、增删记录等）

```txt
// 日志结构体定义
Entry { index: u64, term: u64, command: Vec<u8> }

// 日志存储示例
[
  { "index": 1, "term": 1, "command": "SET x=1" },
  { "index": 2, "term": 1, "command": "SET y=2" },
  { "index": 3, "term": 2, "command": "DEL x" }
]
```

## 状态机

在学习日志复制流程之前，我们需要理解状态机的概念，因为日志是需要被状态机所使用。

复制状态机的基本思想是一个分布式的状态机，系统由多个复制单元组成，每个复制单元均是一个状态机，它的状态保存在操作日志中。服务器上的一致性模块负责接收外部命令，然后 **追加到自己的操作日志中** ，它与其他服务器上的一致性模块进行通信，以保证每一个服务器上的操作日志最终都以 <u>相同的顺序包含相同的指令</u> 。一旦指令被正确复制，那么每一个服务器的状态机都将按照操作日志的顺序来处理它们，然后将输出结果返回给客户端。

状态机 是 Raft 节点的 “业务逻辑核心”。只有日志被 _大多数节点确认_ (即是遵循过半原则)并标记为“已提交”（`committed`）之后，才会被提交给状态机执行。

* 状态机根据日志内容修改内部状态
* 所有节点的状态机必须在相同顺序下执行相同的命令
* 这样才能保证所有节点的状态一致

## 日志复制

### 流程

1.Leader 接受并处理客户端请求

* 客户端发送一条 SET name=foo 的请求
* Leader 创建一条新的日志条目，写入到自己的日志列表中（此时该日志状态是   **uncommitted** ，因为没有被大多数节点确认）
* 开始向所有 Follower 节点发送 `AppendEntries` RPC 请求进行复制

2.Leader 向 Followers 发送日志条目
AppendEntries 结构如下:

```txt
{
  term: 2,
  leader_id: "A",
  prev_log_index: 5,
  prev_log_term: 2,
  entries: [ { index: 6, term: 2, command: "SET name=foo" } ],
  leader_commit: 5
}
```

* prev_log_index 和 prev_log_term：代表前一条日志的 **索引** 和 **任期**，用来检测日志是否连续
* entries: 新的日志条目
* leader_commit: Leader 当前已提交到状态机的最后一条日志

3.Follower 接收后做验证，Follower 会根据 `prev_log_index` 和 `prev_log_term` 进行判断是否确认写入

* 如果本地日志中找不到匹配项，则说明发生冲突，拒绝这次日志追加
* 能找到匹配项，接受日志，并把新条目添加到本地日志，确认日志写入

4.Leader 收到多数响应后，认为日志是可以确认为 已提交 (`committed`)

* Leader 会更新自己的 commitIndex
* 然后发送新的 `AppendEntries`，通知其他节点 “**这条日志已经被提交**”
* 每个节点一旦知道日志已提交，就把它 **应用到状态机** (再强调一下，当日志提交之后才能应用到状态机上)

5.日志提交到状态机

* 日志一旦 “提交” ，就意味着它已被多数节点持久化
* 日志被按顺序交给状态机执行
* 然后客户端就能收到响应，例如“操作成功”

<u>总结为一句话就是：先写本地日志，保证持久性，再发送日志复制请求。确认 quorum 复制后，提交日志并最终应用到状态机上</u>

### 关键原则

在 raft 的日志复制中，需要遵循以下原则：

* **多数派原则**：只有日志被多数节点复制，才算 “已提交” ，可以容忍少数节点失败(在后续的同步中可以将缺失的日志补充上）。
* **日志冲突检测与回滚**：每次 AppendEntries 都带上前一条日志信息（`index + term`）。如果发现某个日志 A 不匹配（`term/index` 不同），Follower 必须删除 A 这条及后续日志并回滚。这保证了所有节点最终拥有一致的日志序列。
* **幂等性和顺序性**：同一个日志条目不能被执行多次，所有节点必须以相同顺序应用日志

## 日志复制的关键点

### 新 Leader 上任后的日志同步

Raft 协议中，只有 Leader 才负责日志复制。当一个新的 Leader 上任后，它必须让所有 Follower 的日志和自己保持一致。这个过程叫做 **日志同步** 。
Leader 会不断向所有 Follower 发送 `AppendEntries` RPC，包含：

* `prevLogIndex` 和 `prevLogTerm`（表示前一条日志的位置和任期, 让 Follower 判断日志是否同步）
* 新的日志条目（可以为空, 表示心跳）
* 当前 `commitIndex`（告诉 follower 哪些日志是已提交的）

### Follower 与 Leader 的日志不一致

在 raft 的新 Leader 上任后执行日志复制操作或是故障 Follower 重新加入集群时，Leader 会执行日志同步的操作，在此过程中可能会出现 Follower 日志与 Leader 日志不一致的情况。

当 Follower 收到 `AppendEntries` 请求时，它会执行以下检查：
1. 检查 `prevLogIndex` 是否存在
2. 检查 `prevLogIndex` 的 `term` 是否和本地日志中的 `term` 相同

如果以上两个检查不通过，说明日志冲突了（Leader 的日志和 Follower 的日志不同步）。此时 Follower 会拒绝这次 `AppendEntries` 。然后 Leader 将执行以下操作：
1. Leader 会将 **prevLogIndex 向前回退一点**（例如减一），重试发送更早的日志条目，直到找到一个匹配点为止。
2. 一旦找到匹配点，Leader 会把 从该点之后的所有日志覆盖发送给 Follower，Follower 会 **删除自己不一致的日志并接受 Leader 的新日志(实际上就是删除匹配点之后的所有日志条目)**

我们举一个例子来加深此部分的理解，存在一个集群其初始状态如下：

| Server | 日志条目 `index:term`          |
|--------|-------------------------------|
| A      | 1:1, 2:1, 3:2, 4:2            |
| B      | 1:1, 2:1, 3:2                 |
| C      | 1:1, 2:1, 3:2, 4:3, 5:3      |

现在服务器 A 成为新的 Leader，它的最新日志是 `index: 4, term: 2` 。我们可知 B 缺少日志 `index: 4, term: 2` , C 有了 `index: 4, term: 3` 和 `index: 5, term: 3`，但 term 是 3，与 Leader 的 term 不一致。

Leader 在上任后会开始执行日志同步操作，执行以下步骤
* Leader 向 B 和 C 发送 `AppendEntries(prevLogIndex=4, prevLogTerm=2)`
  * B 拒绝：因为它没有 `index=4` 的日志
  * C 拒绝：因为它的日志是 `index=4, term=3` 和 `index=5, term=3`，而 Leader 发送过来的日志请求中 `term=2`
* Leader 回退并尝试 `AppendEntries(prevLogIndex=3, prevLogTerm=2)`
  * B 接受：B 本地存在 `index=3,term=2`
  * C 接受：C 从本地的日志中寻找 `index=3 term=2`，这可以找到
  * 现在 Leader 发送从 `index=4` 开始的日志
* B 和 C 接受 `index=4, term=2` 新的日志
  * B 追加日志：B 添加日志 `index=4, term=2`
  * C 先删除旧日志再同步新日志：C 会删除旧的日志 `index=4, term=3` 和 `index=5, term=3`，因为这些日志与 Leader 不一致，然后添加新的日志 `index=4, term=2` (C 会删除匹配点之后的所有日志条目)

最终所有节点日志统一为： 1:1, 2:1, 3:2, 4:2，以让集群中的所有成员日志是同步的。

我们做一个总结，以下流程保证了 所有 Follower 的日志最终都会和 Leader 保持完全一致
1. Leader 选举成功后，开始发送 `AppendEntries`
2. 如果 Follower 的日志不匹配，则拒绝不匹配的日志
3. Leader 回退 `prevLogIndex` 重发请求，逐步查找匹配点
4. 一旦找到匹配点，Leader 会重发匹配点之后的所有日志
5. Follower 删除不一致的日志并追加 Leader 的日志


### 陈旧读

如果一个集群发生了重选举，此时 A 作为 Leader，对 B 发送读取请求。但是 B 的数据 _落后于 A，并且 B 还没有开始同步数据_ 。此时 B 未处于 Leader 的控制下，我们有可能读到 旧 Leader 时期的数据，这违反了线性一致性（`linearizability`）

Raft 的原始设计要求：读请求必须经过 Leader，并且在 Leader 确认自己是 **当前合法 Leader 的前提下才能返回读结果** 。我们不能直接对 Follower 读，即使允许从 Follower 读，也必须确认它是最新的并已和 Leader 同步过

> 在一些使用/实现 raft 的开源组件中，会使用 Lease 租约来确保 Follower 是属于 Leader 的控制
