# 5、CP-Raft：高性能容错复制协议
- 原文：[Notes/论文原文/5、CP-Raft：高性能容错复制协议.pdf at master · LYuYang61/Notes · GitHub](https://github.com/LYuYang61/Notes/blob/master/%E8%AE%BA%E6%96%87%E5%8E%9F%E6%96%87/5%E3%80%81CP-Raft%EF%BC%9A%E9%AB%98%E6%80%A7%E8%83%BD%E5%AE%B9%E9%94%99%E5%A4%8D%E5%88%B6%E5%8D%8F%E8%AE%AE.pdf)
### 1、简介
- 现有 OORaft 面临的难题：
	- 依赖分析开销大
	- 幽灵日志与可用性问题
	- 形式化验证问题
- CP-Raft 的主要贡献：
	- 通过简明的选举策略修复错误：CP-Raft 引入了三阶段选举策略，将日志收集和同步与选举前和选举阶段相结合
	- 基于高效依赖分析的性能优化：CP-Raft 是第一个解决日志条目依赖分析开销的 OORaft 协议
	- 一种分阶段的 TLA+验证方法

### 2、CP-Raft 的核心设计
#### 2.1 日志条目设计
CP-Raft 通过引入新的日志条目类型来管理日志中的“间隙”：
- **RE 常规日志条目**：包含有效的负载，可根据协议乱序复制、提交、执行
- **GE 间隙日志条目**：仅保存任期信息，不含负载，可以被后续的 RE 和 CE 条目覆盖
- **CE 配置日志条目**：必须是新任期的第一个条目，记录了上一任期中所有 GE 的索引列表，当一个 CE 被提交时，上一任期的所有 GE 都会被确认
- **CGE 已确认的间隙日志条目**：由 CE 应用过程生成，不含负载，代表一个已被确认且不能被任何后续日志条目覆盖的间隙
#### 2.2 位图依赖分析方法
- 对于一个日志条目 $Entry_n$ ​，其依赖位图 $B_n$ ​ 存储了在 `n - look_back_size` 到 `n-1` 这一范围内，所有需要在 $Entry_n$ 之前被提交和应用的条目索引
- $B_n​=(b_{n,n-1},b_{n,n-2}​,...,b_{n,n-look_back_size}​)$，其中 $b_{i, j}=1$ 表示 $Entry_i$ 依赖于 $Entry_j$，否则为 0
- 当 leader 生成了 $B_n​$ 后，它将 $Entry_n​$ 及其位图复制到 follower $S_k$ ​
- Follower $S_k$ ​ 维护一个本地的索引位图 $I_k​$，该位图记录了其已持久化的所有日志条目
- Follower 通过一个简单的位操作来检查所有依赖是否都已满足： $B_n\wedge (I_k<<\epsilon)=B_n$，其中，$\epsilon$ 是 n-1 与 $I_k$ 中最大索引之间的差值
- 若该条件为真，则表示 $Entry_n$ 的所有依赖条目都已在 follower 上持久化，可以乱序处理条目；否则，$Entry_n$ 被添加到缓存中，以等待依赖条目的到达

![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250902110737.png)
- CP-Raft 的位图方法将依赖分析的复杂度降为 O (1)，与日志条目的数量或依赖分析范围 (DAR) 无关
- DP-Raft 采用的索引列表循环检查复杂度为 O (DAR)，ParallelRaft 采用的 LBA 列表检查复杂度为 O (N* DAR)
- 分布式系统中，依赖分析通常在锁内执行，以确保并发安全，CP-Raft 的位操作将锁的持有时间降至最低，从而显著减少了锁竞争，提升了并行处理效率

### 3、三阶段选举
1. **日志收集阶段**：对应 Raft 的预投票阶段，主要用于收集并恢复已提交但候选者缺失的日志条目
2. **日志同步阶段**：对应 Raft 的投票阶段，主要用于将已收集的日志同步给其他节点
3. **间隙确认阶段**：对应 Raft 的领导者提交 CE 的阶段，用于将上一任期的间隙永久性地确认为 CGE
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250902155210.png)
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250902165344.png)

#### 3.1 阶段一：日志收集（预投票）
- 当节点 Si 因超时未收到 leader 心跳时，会转换为 `pre-candidate` 状态，并向其他节点广播预投票请求
- 该请求不仅包含其 `lastLogTerm` 和 `lastLogIndex`，还额外附带一个 `List(GE)`，即它日志中的间隙索引列表
- 其他节点在响应时，如果同意其成为 leader，则会返回自己日志中缺失或已提交的条目，帮助 `pre-candidate` 在正式投票前就完成日志的收集
#### 3.2 阶段二：日志同步（投票）
- 当 Si 收集到多数节点的预投票许可后，它会成为 `candidate` 并增加任期
- 此时，Si 进入正式的投票阶段，并开始向其他节点同步其在第一阶段收集到的所有已提交日志条目
- 如果某个节点在第一阶段没有与该候选者通信，则会采用传统 Raft 的日志复制机制，通过任期和索引对比来同步日志
- 当 `candidate` 获得多数投票后，Si 将成为 `pre-leader`
#### 3.3 阶段三：间隙确认（全投票）
- 成为 `pre-leader` 的节点 Si 尚不能立即对外提供服务，它会生成一个新的 CE 日志条目，其中包含了上一任期所有的 GE 索引
- 这个 CE 条目会被复制到多数节点并被提交，之后，Si 才会正式成为 `leader`，并将上一任期的 GE 状态转换为不可覆盖的 CGE，然后才开始响应客户端的请求

![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250902170750.png)
1. 旧 leader 失败：在 t 0 时刻，S 1 是领导者，Entry 3 和 Entry 4 已提交，但 Entry 2 未提交，且 Entry 3 只在 S 2 上持久化，Entry 4 只在 S 3 上持久化
2. 日志收集：超时后 S 3 成为 `pre-candidate`，它通过预投票请求告诉其他节点它缺失 Entry 2 和日志 Entry3。在 t 1，S 3 从 S 2 获得 Entry 3，并了解到 S 2 也缺失 Entry 2 和 Entry 4
3. 日志同步：S 3 在获得多数预投票后成为 `candidate`。在 t 2，S 3 向 S 2 同步 Entry 4，并告诉 S 2 它缺失 Entry 2
4. 间隙确认：当 S 3 获得多数投票后，它成为 `pre-leader`。此时，它生成一个 CE（作为日志 5），其中包含 Entry 2 的索引，并将其复制给 S 2。当 Entry 5 在多数节点上提交后，Entry 2 就会在所有节点上被确认为 CGE，从而杜绝了其被意外恢复为“幽灵日志”的可能性 