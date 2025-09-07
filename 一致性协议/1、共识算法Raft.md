# 1、共识算法Raft
- 原文：[Notes/论文原文/1、共识算法Raft.pdf at master · LYuYang61/Notes · GitHub](https://github.com/LYuYang61/Notes/blob/master/%E8%AE%BA%E6%96%87%E5%8E%9F%E6%96%87/1%E3%80%81%E5%85%B1%E8%AF%86%E7%AE%97%E6%B3%95Raft.pdf)
### 1、复制状态机 Replicated state machine
- **核心思想：相同的初始状态 + 相同的输入 = 相同的结束状态**
- 多个节点上，从相同的初始状态开始，执行相同的一串命令，产生相同的最终状态
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250708113640.png)
1. Client 向 leader 发送一个命令
2. Leader 将客户端请求 (command) 封装成一个个日志实体 (log entry) 中，并将这些日志实体复制到所有 follower 节点
3. 所有节点按相同顺序将 log 中的 command 应用到自己的 state machine，并生成一致的状态
4. Client 无论查询哪个节点的状态机（正常应用日志），结果一致
- 分布式场景下的各节点间，就是通过共识算法来保证命令序列的一致，从容始终保持它们的状态一致，从而实现高可用

### 2、状态简化
- 任何时刻，每一个服务器节点都处于 **leader、follower 或 candidate 三个状态**之一
- Raft 只需考虑**状态的切换**，不用像 Paxos 需要考虑状态之间的共存和互相影响
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250708144732.png)
1. 任何一个节点启动时是 follower 状态
2. 若察觉到集群中没有 leader，则切换至 candidate 状态
3. 经历一次或多次选举，决定切换到 leader 状态或 follower 状态
4. 在 leader 状态任期结束或自身发生宕机等其他问题，切换回 follower 状态
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250708145425.png)
- Raft 把时间分割成任意长度的**任期 (term)**，用连续的整数标记，通常为 int 型
- 每一段任期从一次选举开始，选举成功后，一个 leader 将管理集群，直到任期结束
- 某些情况下，一次选举无法选出 leader（比如，两个节点收到了相同的票数），这一任期以没有 leader 结束，一个新的任期（包含一次新的选举）会很快重新开始
- Raft 保证在任意一个任期内，最多只有一个 leader
- 任期可以明确地标识集群的状态，可以确认一台服务器历史的状态（比如，查看服务器是否具有 t2 时期的日志，来判断是否出现宕机）
- Raft 中服务器节点之间使用 RPC 进行通信，主要有两种：
	- **RequestVote RPC（请求投票）**：由 candidate 在选举期间发起
	- **AppendEntries RPC（追加条目）**：由 leader 发起，用来复制日志和提供一种心跳机制
- 服务器之间通信的时候会交换当前任期号：
	- 如果一个服务器上的当前任期号比其他的小，该服务器会将自己的任期号更新为较大的那个值
	- 如果一个 candidate 或 leader 发现自己的任期号过期了，会立即回到 follower 状态
	- 如果一个节点接收到一个包含过期的任期号请求，会直接拒绝

### 3、领导者选举
- Raft 内部有一种心跳机制，如果存在 leader，它就会周期性地向所有 follower 发送心跳，来维持自己的地位
- 如果 follower 一段时间没有收到心跳，那么他就会认为系统中没有可用的 leader 了，然后开始进行选举
- 开始一个选举过程后，follower 先增加自己的当前任期号，并转换到 candidate 状态，然后投票给自己，并且并行地向集群中的其他服务器节点发送请求投票 RPC
- 最终会有三种结果：
	- 它获得**超过半数选票**赢得了选举 -> 成为 leader 并开始发送心跳
	- 从其他的服务器接收到声明它是 leader 的追加条目 RPC -> 若**该 leader 的任期号不小于自己当前的任期号**，就从 candidate 回到 follower 状态；否则，拒绝这次 RPC 并保持 candidate 状态
	- 一段时间之后没有任何获胜者（比如，多个 follower 同时成为 candidate，得票太过分散） -> 每个 candidate 都在一个**随机选举超时时间（150-300 ms）** 后，默认增加任期号开始新一轮投票
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250708161331.png)
- Follower 投票逻辑：
	- 收到请求投票 RPC 后，先校验 candidate 是否符合条件：
		- Term 是否比自己大
		- 日志（安全性）
	- 每个 follower 一张选票，按照先来先得的原则投出

### 4、日志复制
- Leader 被选举出来后，开始为客户端请求提供服务
- 客户端的每一个请求都包含一条被复制状态机执行的指令
- Leader 把这条指令作为一条新的日志条目附加到日志中，然后并行发送追加条目 RPC 给follower，让它们复制该条目
- 当该条目被**超过半数**的 follower 复制后，leader 应用这条日志条目到它的状态机中【**提交**】（对应的 follower 也要提交），然后把执行的结果返回给客户端
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250708210123.png)
- 一条日志中需要具有三个信息：
	- **状态机指令**：对某个值进行某个操作 (x <- 3)
	- **Leader 的任期号**：检测日志副本一致性，判断节点状态
	- **日志索引**：标识位置，区分日志前后关系
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250708215950.png)
- 在日志复制过程中，leader 或 follower 随时都有崩溃或缓慢的可能性，Raft 必须要在有岩机的情况下继续支持日志复制，并且保证每个副本日志顺序的一致（以保证复制状态机的实现），具体有三种可能：
	- **Follower 缓慢**：如果有 follower 因为某些原因没有给 leader 响应，那么 leader 会**不断地重发**追加条目请求 RPC，哪怕 leader 已经回复了客户端（超过半数 follower 复制），直至 follower 追上日志
	- **Follower 宕机**：如果有 follower 宕机后恢复，这时 Raft 追加条目的**一致性检查**生效，保证 follower 能按顺序恢复崩溃后的缺失的日志
		- **Raft 的一致性检查**：leader 在每一个发往 follower 的追加条目 RPC 中，会放入**前一个日志条目的索引位置和任期号**，如果 follower 在它的日志中找不到，那么它就会拒绝此日志，leader 收到 follower 的拒绝后，会再发送前一个日志条目，从而**逐渐向前定位到 follower 第一个缺失的日志**
	- **Leader 宕机**：
		- 如果 leader 宕机，那么宕机的 leader 可能已经复制了日志到部分 follower 但**还没有提交**，而被选出的新 leader 又可能不具备这些日志，导致**有部分 follower 中的日志和新 leader 的日志不相同**
		- 此时，leader 通过**强制 follower 复制它的日志**来解决不一致的问题，即 follower 中跟 leader 冲突的日志条目会被新 leader 的日志条目覆盖
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250708221928.png)

### 5、安全性
![](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250709103714.png)
#### 5.1 选举限制
- 保证在选举的时候新 leader 拥有所有之前任期中已经提交的日志条目
- 请求投票 RPC 实现了这样的限制：RPC 中包含了 candidate 的日志信息，然后投票人会**拒绝那些日志没有自己新的投票请求**
- Raft 通过比较两份日志中最后一条日志条目的索引值和任期号来定义谁的日志比较新：
	- 如果两份日志最后条目的任期号不同，那么任期号大的日志更新
	- 如果两份日志最后条目的任期号相同，那么日志较长的那个更新
- 如，c -> d：S 5 通过 S 2、S 3、S 4 的选票再次选举成功，因为 S 2、S 3、S 5 的日志号相同，但 S 5 的任期号更大，符合选举限制
#### 5.2 提交之前任期内的日志条目
- 一旦当前任期内的某个日志条目已经存储到过半的服务器节点上，leader 就知道该日志条目可以被提交了
- Follower 的提交触发：下一个追加条目 RPC（心跳 or 新日志），通过 leaderCommit 参数得知 leader 提交到哪个日志
- 如果某个 leader 在提交某个日志条目之前崩溃了，以后的 leader 会试图完成该日志条目的**复制**
- Raft 永远**不会通过计算副本数目的方式来提交之前任期内的日志条目**
- 如，c 中 S 1 将日志 2 复制给大多数节点，如果都提交，可以认为集群上日志 2 已提交，若此时 S 1 宕机，S 5 依靠任期号 3 当选 leader，进入 d，导致已提交的日志 2 被覆盖掉
- 只有 leader **当前任期内的日志条目**才通过计算副本数目的方式来提交
- 如，c -> e：S 1 在任期 4 内的日志提交时，之前的所有日志条目也都会被提交
#### 5.3 Follower 和 Candidate 宕机处理
- 如果 follower 或 candidate 崩溃了，那么后续发送给他们的请求投票和追加条目 RPCs 都会失败
- Raft 通过**无限的重试**来处理这种失败，如果崩溃的机器重启了，那么这些 RPC 就会成功地完成
- 如果一个服务器在完成了一个 RPC，但是还没有响应的时候崩溃了，那么它重启之后就会再次收到同样的请求【**Raft 的 RPC 都是幂等的**】
#### 5.4 时间和可用性
- 只要整个系统满足下面的时间要求，Raft 就可以选举出并维持一个稳定的 leader：
> 广播时间（broadcastTime）  <<  选举超时时间（electionTimeout） <<  平均故障间隔时间（MTBF）
- 广播时间：
	- 从一个服务器并行的发送 RPCs 给集群中的其他服务器并接收响应的平均时间
	- 必须比选举超时时间小一个量级，这样 leader 才能够发送稳定的心跳消息来阻止 follower 开始进入选举状态
	- Raft 的 RPCs 需要接收方将信息持久化的保存到稳定存储中去，所以广播时间大约是 0.5 ms 到 20 ms，取决于存储的技术
- 选举超时时间：
	- 要比平均故障间隔时间小上几个数量级，这样整个系统才能稳定的运行
	- 10 ms 到 500 ms
- 平均故障间隔时间：
	- 对于一台服务器而言，两次故障之间的平均时间
	- 大多数的服务器的平均故障间隔时间都在几个月甚至更长
- 广播时间和平均故障间隔时间是由系统决定的，但是选举超时时间是我们自己选择的

### 6、集群成员变更
- 在需要改变集群配置的时候 (如增减节点、替换宕机的机器或者改变复制的程度)，Raft 可以进行配置变更自动化
- 自动化配置变更机制最大的难点是**保证转换过程中不会出现同一任期的两个 leader**，因为转换期间整个集群可能划分为两个独立的大多数
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250709230011.png)
- 因此配置更改必须使用**两阶段**方法，集群先切换到一个过渡的配置，称之为**联合一致**(joint consensus)：
	- 第一阶段，leader 发起 Cold, new，使整个集群进入联合一致状态。这时，所有 RPC 都要在**新旧两个配置中都达到大多数支持才算成功**
	- 第二阶段，leader 发起 Cnew，使整个集群进入新配置状态。这时，所有 RPC 只要在**新配置下能达到大多数支持就算成功**
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250709232214.png)
- 虚线表示已经被创建但是还没有被提交的配置日志条目，实线表示最后被提交的配置日志条目
- Leader 首先创建了 C-old, new 的配置条目在自己的日志中，并提交到 C-old, new 中（C-old 的大多数和  C-new 的大多数）
- 然后 Leader 创建 C-new 条目并提交到 C-new 中的大多数
- 这样就不存在  C-new 和 C-old 可以同时做出决定的时间点

### 7、日志压缩
- Raft 采用的是一种**快照技术**，每个节点在达到一定条件之后，可以把当前日志中的命令都写入自己的快照，然后就可以把已经并入快照的日志都删除了
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250710155009.png)
- 一个服务器用新的快照替换了从 1 到 5 的条目，**快照值存储了当前的状态**
- 快照中包含了**最后的索引位置和任期号**：为了支持快照后紧接着的第一个条目的附加日志请求时的一致性检查，因为这个条目需要前一日志条目的索引值和任期号
- Leader 必须偶尔的发送快照给一些落后的 follower