---
date: '2025-05-17T17:16:36+08:00'
draft: false
title: '分布式算法-Raft「2」'
categories: ['分布式']
tags: ['raft'] 
summary: "raft 领导选举流程"
---

在 Raft 协议中，系统只能有一个 Leader，Leader 负责接收客户端请求并将操作复制到其他节点。一旦 Leader 宕机或网络分区导致心跳丢失，Follower 节点将认为 Leader 不再可用，从而触发一次新的领导选举。
这套机制的目的是保证系统在任何时刻都能恢复服务，并且保持日志一致性和状态同步。

## 选举超时

每个节点有一个随机的 `Election Timeout` 选举超时（150ms～300ms），如果在这段时间内 _没有收到 Leader 的心跳_ ，它就会开始执行选举流程：
* 将自己的状态变为 Candidate
* term + 1，启动新一轮任期
* 投票给自己
* 向集群中其他节点发送投票请求
* 若收到超过一半节点的投票，则成为 Leader

## 当选领导

当一个 Candidate 得到了集群内过半票数后，将会成为新任期的 Leader。并立刻发送一个 `AppendEntries` 请求给集群内的所有成员，以宣誓自己的 Leader 地位。进一步，开始处理客户端的写入请求，并将日志同步给其他节点。在后续的运行中会 **周期性的向集群内所有成员发送心跳检测请求** ，避免其他节点再次选举

> Leader 的心跳检测周期必须 **小于** 选举定时器的超时时间，否则会导致其他节点频繁成为 Candidate 开展选举流程

## 选举过程

### Follower 投票原则

Follower 节点在收到投票请求时，会进行一系列判断：
* 首先会进行 term 检查，如果请求中的 term < 自己当前的 term，说明该 Candidate 比自己还落后，直接拒绝。如果请求中的 term > 自己当前的 term，则更新自己的任期
* 紧接着会判断是否已投过票（Raft 明确规定 **一个节点在一个 term 中只能投出一张票**），如果已经投票了，并且不是投给这个 Candidate 则拒绝投票
* 最后会判断候选人的日志是否足够新，如果 Candidate 日志较旧，会被拒绝投票。如果候选人日志 _“更新”_ 或者 _“同样新”_ ，才有资格成为 Leader。通过比较 Candidate 最后一条日志的 term 和 index 来确定
日志新旧的比较逻辑

举个例子：
* 本地最后一条日志是 (index=5, term=3)
* 请求者的最后日志是 (index=4, term=3)

虽然 term 一样，但 index 小，说明对方落后则当前节点不投票。但如果请求者日志是 (index=6, term=3) 或 (index=5, term=4)，说明日志领先于当前节点，可以投票

### Candidate 选举成功判断

Candidate 会统计投票请求的响应
* 如果收到超过半数的投票（vote_granted == true），则成功当选 Leader
* 如果收到多数拒绝或超时（无回复），则选举失败
* 如果发现某个回复中 term > 自己的 term，说明已有更高任期存在，则放弃本轮选举并退回 Follower 状态
* 如果失败，则等待下一轮随机选举超时重试

### Candidate 平票

当出现多个 Candidate 时，两个 Candidate 有可能会同时发起投票请求。如果获取的票数不同，则最新获取过半票数的 Candidate 会成为 Leader，这很好理解

但可能会出现一种情况：这两个 Candidate 获取的票数一致。这将导致没人能成为 Leader，所有节点继续等待新的选举超时。下一轮由于超时时间是随机的，某个节点会更快发起新的一轮选举流程

### Term 变更

每个节点的选举定时器一旦超时, 就会 **将自己的任期号 term 增加 1**，并发起一轮新的选举。
每个节点都有一个独立、随机的选举超时时间（比如 150–300ms），只要在这段时间内没有收到来自 Leader 的心跳（AppendEntries请求）或合法 Candidate 的投票请求，就认为 Leader __“失踪”__ 了。

一旦超时节点就会自增当前 term（term += 1）-> 转为 Candidate -> 给自己投票 -> 向其他节点发送 RequestVote 请求

注意，这个 term 增加操作只会在 自己主动发起新一轮选举时执行
* **如果是 Follower 或刚刚投过票, 没超时就不会加 term**
* **节点只在收到 更高 term 的请求时，才会更新自己的 term**

每个节点都维护一个本地的 currentTerm，但它们之间会通过消息通信来同步，使整个集群趋于共识。
* term 是局部维护的，但在选举或 RPC 通信过程中自动同步
* term 不是全局共享变量，而是通过比较传播、协商出来的全局事实

### 选举流程中其余节点定时器变化

在集群中，即使已有 Candidate 开始了选举过程，其余节点的选举定时器依旧会继续执行。当收到有效的 Leader 心跳，节点定时器被重置，收到某个 Candidate 的投票请求，投票成功后定时器也会被重置，若超时了，自己也会变成 Candidate，再次发起投票。
换句话说：
* 即便有人先发起选举，其他节点的定时器仍然在跑
* 如果第一位 Candidate 没有拿到大多数票，其他节点可能会陆续超时并成为 Candidate

但这就可能出现 选票分裂现象，导致多轮选举（多轮选举可能因为网络延迟或选票分裂而发生，是正常现象）直到某个节点赢得大多数选票

## 读/写请求的机制

**写请求**只能由 Leader 接受和处理。这是 Raft 的核心原则之一：“强一致写”。此原则确保了 **线性一致性（Linearizability）** ：所有客户端看到的数据修改顺序是唯一确定的，并且符合实际时间顺序。

* 客户端只能将写请求发送给 Leader
* Leader 会将操作作为日志项（LogEntry）追加到自己的日志中
* 然后通过 _日志复制机制_ 发送给大多数 Follower
* 一旦被多数节点确认（committed），Leader 才认为写成功并响应客户端

**读请求** 可以有多种策略处理，但要注意：Raft 默认不是所有情况下都允许 Follower 直接处理读请求。因为 Follower 可能是旧 Leader 崩溃后的 “旧状态”，如果立刻接受读请求，有可能读到陈旧数据（stale read）
我们可以使用以下两种策略来进行安全读取：
* 由 Leader 处理读请求：Leader 一定是最新、合法的节点，所有读请求都发送到 Leader，读取 Leader 最新的状态。这保障了读取的强一致性
* Follower 读 + Lease 机制：从 Follower 读取的一个原则是需要确保该 Follower 还在 Leader 的控制下

> Lease 机制是确保 Follower 节点在指定时间内仍处于 Leader 的管控下, Leader 在与 Follower 通信时会更新 Follower 的 Lease(也就是续约)

### ReadIndex 机制

在 Raft 协议中，为了实现线性一致读（即：读取的数据保证是最新已经被 quorum 同意的数据），Raft 引入了 ReadIndex 流程，它相比于提交一个空的 NoOp 日志开销更小，且无需实际写日志。让 Leader 确保它仍然是 Leader，并且其日志已被复制到 quorum 上。

Leader 向自身确认是否是 Leader 且日志未变
1. 发送 ReadIndex 请求到多数派节点（Follower），即发送带有当前 commit index 的 AppendEntries 心跳
2. 等待 quorum 确认该 index 是已知的
3. 如果确认成功，则表示 从此 index 开始的数据是 linearizable 的，可以安全读
4. 否则返回错误

可以看出此流程强制确保 **当前 Leader 的读取在集群内是线性一致的** 。

如果 Leader 不是合法的 Leader，那么这个 ReadIndex 请求是不会获得多数响应。如果当前 commit index 还未被 Follower 复制，也不会返回 OK
