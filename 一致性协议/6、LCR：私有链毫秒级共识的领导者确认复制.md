# 6、LCR：私有链毫秒级共识的领导者确认复制
- 原文：[Notes/论文原文/6、LCR：私有链毫秒级共识的领导者确认复制.pdf at master · LYuYang61/Notes · GitHub](https://github.com/LYuYang61/Notes/blob/master/%E8%AE%BA%E6%96%87%E5%8E%9F%E6%96%87/6%E3%80%81LCR%EF%BC%9A%E7%A7%81%E6%9C%89%E9%93%BE%E6%AF%AB%E7%A7%92%E7%BA%A7%E5%85%B1%E8%AF%86%E7%9A%84%E9%A2%86%E5%AF%BC%E8%80%85%E7%A1%AE%E8%AE%A4%E5%A4%8D%E5%88%B6.pdf)
### 1、简介
- 私有链通过强化身份验证机制来管理网络参与者
- 许多私有链系统普遍采用基于领导者的崩溃容错（CFT）共识协议【例如 Raft 和 Paxos】来实现状态机复制（SMR），这种设计在信任环境中能够有效提升数据一致性效率
- 传统的领导者型 SMR 的性能瓶颈：
	- 节点间不稳定的广域网（WAN）环境会引入高延迟，导致大量数据在 SMR 过程中因超时而需要重传，从而产生巨大的网络流量
	- Leader 节点需要同时处理大量的客户端请求和 SMR 复制任务，其有限的计算和网络资源容易达到饱和，引发工作线程饥饿甚至死锁问题，导致系统吞吐量（TPS）显著下降，并使 leader 面临高昂的开销
	- 所有客户端请求，无论最初发送给哪个节点，最终都必须重定向到 leader 进行处理，这为远端客户端带来了额外的通信延迟
- 现有解决方案的局限性：
	- **多领导者（Multi-leader）模型**：通过将数据存储分片，并为每个分片指定不同的领导者来分散负载。虽然这可以有效减轻单一领导者的压力，但当任何一个节点宕机时，其控制的分片将因需要重新选举而暂时不可用，这导致可用性问题。此外，频繁的领导者选举会带来额外的开销 
	- **无领导者（Leaderless）协议**：这类协议（例如 EPaxos）试图通过分析操作间的依赖关系来确定提交方式。如果操作之间没有依赖冲突，则可以快速达成一致。然而，一旦发生依赖冲突，则需要额外的网络往返来强制执行排序约束，这增加了协议的复杂性，并使性能对数据依赖性高度敏感 
	- **批处理（Batching Entries）**：该方法通过将多个日志条目合并成一个请求来减少 SMR 请求的次数，从而提高吞吐量。然而，这种方法会增加平均响应时间，因为客户端的第一个请求必须等待缓冲区被填满，这在低并发场景下尤为明显 

### 2、LCR 核心：去中心化复制与确认
- LCR 基本思想：允许 follower 承担非事务型数据的复制工作，而 leader 仅负责对这些复制进行最终的确认，可以显著减轻 leader 所面临的巨大负载，同时维持低请求延迟和高吞吐量
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250904111127.png)

#### 2.1 未来日志 Future-Log 的生成与复制
- 未来日志是一种独立于 leader“正常日志”的日志，存在于每个节点上
- 任何接收到非事务型数据的 follower 节点都可以充当“数据领导者”(data leader)，并主导该数据的复制过程
- 当一个 follower 节点 $S_i$ 接收到非事务型数据 $D_{nt}$ 时，它会：
	1. 成为该数据的“数据领导者”
	2. 为该数据分配一个全局唯一的“未来条目”索引 λ：$$ \lambda=S_{i}+G_{i}+lastIndex_{i}-lastIndex_{i}\%G_{i} $$
		- Si​ 是服务器的 ID，Gi​ 是当前集群的 Generation，lastIndexi​ 是该节点上未来条目的最后一个索引
	3. 将该未来条目 $FE_λ$ 写入其本地的未来日志
	4. 将该未来条目 $FE_λ$ 复制给集群中的所有其他节点 (follower + leader)
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250904111457.png)
#### 2.2  Leader 的确认 (Signal Entry) 与冲突处理
- Signal Entry：
	- 当数据领导者的 $FE_λ$ 被复制到一组节点后，leader 并不需要把整个 $FE_λ$ 重新复制到每个节点
	- Leader 会发布一个 signal entry，表示其已确认索引 λ 对应的未来条目
	- 只是一个轻量化的确认（内容只包含索引/标识信息）
- Leader 确认过程：Leader 在打包追加条目时，会查看每个 follower 的 nextIndex：
	- 若 follower 的 nextIndex+1 指向的是未来条目（leader 认为没有接收过 FE），leader 不会直接替换，而把该位置的条目作为正常条目打包，以确保 follower 能继续推进正常日志
	- 对于已经接收过 FE 的 follower，leader 用 signal entry 来确认（替换或标识为已确认），并通过追加条目把 signal 传播给尚未接收到 FE 的节点，使它们也能看到被 leader 确认的事实
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250904143124.png)
- $FE_λ$ 与正常条目冲突解决：
	- 若 leader 发来的正常条目 $E_λ$ 与某 follower 的 $FE_λ$ 冲突（两者索引相同但内容不同），follower 无条件相信 leader，但 follower 会检查自己是否为该 $FE_λ$ 的数据领导者，判断条件为 $\lambda \% G_i==S_i$
		- 若是数据领导者：该 follower 将把本地 $FE_λ$ 从未来日志中删掉，重新为该条目分配新的未来条目索引，并把这个重新分配后的 FE 重新发起复制
		- 若不是：直接丢弃 $FE_λ$（leader 的正常条目具有优先权）
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250904144003.png)
#### 2.3 未来条目索引分配
- LCR 设计了一个窗口来管理未来词条的写入过程
- 窗口是一个连续的索引范围，它有两种状态：关闭和打开
- 当窗口打开时，节点可以充当数据领导者，在其窗口内使用索引写入将来的条目
- 当窗口关闭时，数据领导者将不再能够生成索引在此范围内的未来条目
- 然而，对于节点 Si，窗口的状态不拒绝从其他数据领导者接收的条目
- 关闭窗口的条件是正常日志(在 SMR 中的已排序索引中复制的日志) 的 lastIndex 大于或等于窗口的起始索引
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250904161018.png)
#### 2.4 Generation：应对成员变更
- 由于未来条目的索引分配与集群中的节点数量有关，因此当集群成员发生动态变更时，索引冲突可能发生
- 为此，LCR 引入 Generation，其作用类似于 term
- Generation 是一个半酣集群节点数量信息的整数，具有更高 Generation 的节点将拒绝来自较低 Generation 节点的未来日志复制请求
- Generation 的变更由以下三种情况触发：
	1. Leader 接收到成员变更配置
	2. 节点在复制响应中得知了更新的 Generation
	3. 节点在复制请求中得知了新的 Generation
- 当 Generation 变更发生时，节点会立即停止发送旧 Generation 的未来条目复制请求
- 对于本地所有未应用的旧 Generation 未来条目，节点会重新分配一个新索引：$$\lambda^{\prime}=\lfloor\frac{\lambda}{G_{i}}\rfloor * G_{i}^{\prime}+S_{i} $$
- 该过程确保新索引总是大于等于旧索引 λ
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250904170033.png)
