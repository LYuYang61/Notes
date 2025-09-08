# 2、分布式文件系统PolarFS
- 原文：[Notes/论文原文/2、分布式文件系统PolarFS.pdf at master · LYuYang61/Notes · GitHub](https://github.com/LYuYang61/Notes/blob/master/%E8%AE%BA%E6%96%87%E5%8E%9F%E6%96%87/2%E3%80%81%E5%88%86%E5%B8%83%E5%BC%8F%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9FPolarFS.pdf)
### 1、摘要
- PolarFS 是一个具有超低延迟和高可用性的分布式文件系统，专为 POLARDB 数据库服务而设计，目前已在阿里云上提供
- PolarFS 在用户空间中利用了轻量级网络堆栈和 I/O 堆栈，充分利用了 RDMA、NVMe 和 SPDK 等新兴技术
- 为了在保证 PolarFS 的副本一致性的同时最大限度地提高 I/O 吞吐量，本文开发了 **ParallelRaft 协议，它利用数据库的无序 I/O 完成容错能力，打破了 Raft 的严格串行化**

### 2、简介
#### 2.1 云计算产业的发展趋势
- 存储与计算的分离，有助于开发共享存储能力：
	- 计算和存储节点可以采用不同类型的硬件并独立定制
    - 存储节点上的磁盘可以组成一个统一的存储池，减少碎片化、磁盘使用不平衡和空间浪费的风险
    - 存储集群的容量和吞吐量可以透明地扩展
    - 由于数据都存储在存储集群上，计算节点没有本地持久状态，这使得数据库迁移更方便、更快速
    - 数据可靠性也因底层分布式存储系统的数据复制和高可用性特性而得到提高
- 这种架构让云数据库受益，例如基于虚拟化技术构建更安全和可扩展的环境，以及通过后端存储集群的快速 I/O、数据共享和快照功能增强多只读实例和检查点等数据库特性
#### 2.2 现有挑战
- 当前的云平台很难充分利用 RDMA 和 NVMe SSD 等新兴硬件标准，许多开源分布式文件系统（如 HDFS 和 Ceph）的延迟远高于本地磁盘，甚至可能相差几个数量级
- 实例存储使用本地 SSD 和高 I/O 虚拟机实例提供高性能，仍存在缺点：
	- 实例存储容量有限，不适合大型数据库服务
	- 无法在磁盘驱动器故障中生存，数据库必须自己管理数据复制
	- 使用通用文件系统 (ext 4 或 XFS)，导致用户空间与内核空间消息传递开销大
	- 不支持共享一切数据库集群架构
#### 2.3 PolarFS 的解决方案
- **利用新兴硬件和用户空间 I/O 栈**：充分利用 RDMA 和 NVMe SSD 等新兴硬件，并在用户空间实现轻量级网络和 I/O 栈，避免陷入内核和处理内核锁
- **POSIX 兼容的 API**：提供类似 POSIX 的文件系统 API，可编译到数据库进程中，替换操作系统提供的文件系统接口，从而使整个 I/O 路径保持在用户空间
- **无锁和无上下文切换的 I/O 模型**：PolarFS 数据平面上的 I/O 模型旨在消除锁，避免关键数据路径上的上下文切换，并消除不必要的内存拷贝，同时大量利用 DMA 在主内存和 RDMA NIC/NVMe 磁盘之间传输数据
- **ParallelRaft 协议**：开发了 ParallelRaft，一个基于 Raft 的增强型共识协议，允许乱序的日志确认、提交和应用，同时让 PolarFS 遵循传统的 I/O 语义，显著提高了并行 I/O 并发性
- **对 POLARDB 的支持**：PolarFS 在其之上实现了 POLARDB，一个基于 AliSQL（MySQL/InnoDB 的一个分支）的数据库系统，支持共享存储架构和多个只读实例。 PolarFS 可以同步文件元数据修改到只读节点，确保并发修改的序列化，并在网络分区时保证只有真正的主节点服务成功，防止数据损坏

### 3、架构
- PolarFS 由两个主要层组成：
	- 下层是**存储管理层**：负责所有存储节点的磁盘资源，并为每个数据库实例提供一个数据库卷
	- 上层是**文件系统层**：支持卷中的文件管理，负责对文件系统元数据的并发访问的互斥和同步
- 对于数据库实例，PolarFS 将文件系统元数据存储在其卷中
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250714101909.png)
#### 3.1 文件系统层
- **Libpfs**：
	- 一个完全在用户空间中运行的轻量级文件系统实现，具有一组类似 POSIX 的文件系统 API，链接到 POLARDB 进程中
	- 负责挂载卷、加载文件系统元数据、构建目录树和文件/块映射表等
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250714105018.png)
#### 3.2 存储层
- 存储层为文件系统层提供管理和访问卷的接口
- 为每个数据库实例分配一个卷，并由一组数据块 chunk 组成
- **数据块 Chunk**：
	- 一个 volume 被分成 chunks，这些 chunk 分布在块服务器之间
	- Chunk 是数据分布的最小单位
	- Chunk 的大小被设置为 10 GB（远大于其他系统）：
		- 在数量级上减少了元数据数据库中维护的元数据量，并且简化了元数据管理
		- 所有的元数据都可以缓存在 PolarSwitch 的主内存中，避免了关键 I/O 路径上的额外元数据访问成本
		- 缺点：一个 chunk 上的热点不能进一步分离
- **块 Block**：
	- Chunk 在 ChunkServer 中进一步划分为 block，每个 block 被设置为 64 KB
	- Block 按需分配并映射到数据块，以实现精简配置
- **PolarSwitch**：
	- **部署在计算节点上的守护进程**，负责将应用程序的 I/O 请求重定向到 ChunkServers
	- 根据本地元数据缓存识别相关的数据块，并将 I/O 请求分解为子请求，最终发送给 ChunkServer
	- PolarSwitch 会缓存数据块副本的位置和 leader信息，并在超时时重试并切换到新的 leader
- **ChunkServers**：
	- **部署在存储节点上**，负责处理 I/O 请求并存储数据块
	- 每个 ChunkServer 拥有一个独立的 NVMe SSD 磁盘并绑定到专用的 CPU 核心，以避免资源争用
	- ChunkServer 使用预写日志 (WAL) 技术确保原子性和持久性，日志优先写入 3D XPoint SSD 缓冲区作为写入缓存
	- ChunkServers 之间使用 ParallelRaft 协议复制 I/O 请求并形成共识组
- **PolarCtrl**：
	- **PolarFS 集群的控制平面**，部署在专用机器组上，提供高可用服务，包括节点管理、卷管理、资源分配、元数据同步、监控等
	- PolarCtrl 负责跟踪 ChunkServer 的成员和活跃状态，并在 ChunkServer 过载或不可用时启动数据块副本迁移
	- 维护所有卷和数据块位置的状态，创建卷并分配数据块给 ChunkServer，并通过推拉方式同步元数据到 PolarSwitch
	- 会监控每个卷和数据块的延迟、IOPS 指标，并定期进行 CRC 检查
	- PolarCtrl 不在关键的 I/O 路径上，其服务连续性可通过传统高可用技术保证

### 4、I/O 执行模型
- 当 POLARDB 访问数据时，它通过 PFS 接口将文件 I/O 请求委托给 libpfs，通常通过 `pfs.pread` 或 `pfs.pwrite`
- 对于写请求，由于设备块通过 `pfs_fallocate` 预分配给文件，几乎不需要修改文件系统元数据，从而避免了读写节点之间昂贵的元数据同步
- **请求处理流程**：
	1. Libpfs 将文件偏移量映射到块偏移量，并将文件 I/O 请求分解为一个或多个固定大小的块 I/O 请求
	2. 这些块 I/O 请求通过 libpfs 和 PolarSwitch 之间的共享内存（由多个环形缓冲区构成）发送到 PolarSwitch
	3. PolarSwitch 不断轮询所有环形缓冲区，发现新请求后，将其从环形缓冲区中取出并转发给相应的 ChunkServer
	4. ChunkServer 使用预写日志 (WAL) 技术确保原子性和持久性。I/O 请求在提交和应用之前写入日志
	5. 日志被复制到一组副本中，并使用 ParallelRaft 共识协议保证副本之间的数据一致性。 只有当 I/O 请求在多数副本的日志中持久记录后，才被认为是已提交的，然后才能响应给客户端并应用于数据块
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250714111632.png)
- **写 I/O 执行流程**：
	1. POLARDB 通过 libpfs 和 PolarSwitch 之间的环形缓冲区发送写入 I/O 请求给 PolarSwitch
	2. PolarSwitch 根据本地缓存的集群元数据将请求传输到相应数据块的 leader 节点
	3. Leader 节点上的 RDMA NIC 将写入请求放入预注册缓冲区，并将请求条目添加到请求队列。I/O 循环线程不断轮询请求队列并处理新请求
	4. 请求通过 SPDK 写入磁盘上的日志块，并通过 RDMA 传播到 follower 节点。这两个操作都是异步调用，实际数据传输并行触发
	5. 当复制请求到达 follower 节点时，其 RDMA NIC 也会将复制请求放入预注册缓冲区并添加到复制队列
	6. Follower 上的 I/O 循环线程被触发，通过 SPDK 异步写入请求到磁盘
	7. 当写入回调成功返回后，确认响应通过 RDMA 发回 leader
	8. 当成功从多数 follower 收到响应后，leader 通过 SPDK 将写入请求应用于数据块
	9. 之后，leader 通过 RDMA 回复 PolarSwitch
	10. PolarSwitch 标记请求完成并通知客户端
- **读 I/O 请求**：
	- 由 leader 单独处理
	- 在 ChunkServer 中有一个名为 IoScheduler 的子模块，它负责仲裁并发 I/O 请求发出的磁盘 I/O 操作的顺序，以便在 ChunkServer 上执行
	- IoScheduler 保证读取操作始终可以检索最新提交的数据
- ChunkServer 使用**轮询模式**和**事件驱动的有限状态机**作为**并发模型**
- 每个 I/O 线程使用一个专用的内核，并使用分离的 RDMA 和 NVMe 队列对
- 即使在一个 ChunkServer 上有多个 I/O 线程，由于 **I/O 线程之间没有共享的数据结构**，所以实现 I/O 线程时不会产生锁开销

### 5、一致性模型
#### 5.1 Raft 局限
- Raft 为求简单易懂而高度序列化，它的日志在 leader 和 follower 上都不允许有空洞，这意味着日志条目必须按顺序被 follower 确认、leader 提交并应用于所有副本
- 这导致并发写入请求按顺序提交，队尾的请求必须等待所有前序请求持久化并响应后才能提交和响应，从而增加了平均延迟并降低了吞吐量
- Raft 在使用多连接传输日志时表现不佳，因为日志条目可能会乱序到达，而 Raft 的 follower 必须按顺序接受日志条目，这会导致阻塞
- 像数据库这样的事务处理系统，其并发控制算法允许事务以交错和乱序的方式执行，同时生成可序列化的结果，这些系统能够容忍传统存储语义导致的乱序 I/O 完成，并通过自身机制保证数据一致性
#### 5 .2 ParallelRaft 协议
 - ParallelRaft 的结构与 Raft 相当相似，通过复制日志实现复制状态机
 - ParallelRaft 划分为更小的部分：**日志复制、leader 选举、Catch up**
##### 5.2.1 乱序日志复制
- ParallelRaft 和 Raft 有一个基本的区别：在 ParallelRaft 中，**当一个条目被识别为已提交时，并不意味着所有先前的条目都已成功提交**
- ParallelRaft 的乱序日志执行遵循以下规则：
	- 如果日志项的**写入范围不重叠**，则认为这些日志项没有冲突，可以**按任何顺序执行**
	- 否则，冲突条目将在到达时按严格的顺序执行，这样，较新的数据就不会被旧版本覆盖
- **乱序确认**：在 ParallelRaft 中，一旦日志条目成功写入，follower 可以立即确认，从而避免了额外的等待时间，优化了平均延迟
- **乱序提交**：在 ParallelRaft 中，一个日志条目在多数副本确认后即可立即提交，这对于通常不承诺强一致性语义（如事务处理系统）的存储系统是可接受的
- **允许日志上有漏洞**：
	- 乱序日志复制和提交允许日志中存在空洞
	- ParallelRaft 引入回溯缓冲区 (look behind buffer)，每个日志条目中的回溯缓冲区**包含前 N 个日志条目修改的 LBA（逻辑块地址）**，充当日志中可能存在的空洞上的桥梁
	- N 是这个桥梁的跨度，也是允许的最大日志空洞大小，通常为 2
##### 5.2.2 leader 选举
- 在 Raft 中，新当选的 Leader 包含了所有以前任期的提交条目。然而，由于日志可能有漏洞，ParallelRaft 中选出的 Leader 最初可能无法满足这一要求
- 因此，在开始处理请求之前，需要一个**额外的 Merge 阶段，使 Leader 拥有所有已提交的条目**
- Merge 时会遇到三种情况：
	- 对于一个已提交（提交指已被共识组中大多数节点确认）但不在 leader 中的项，leader 候选人总是可以从至少一个 follower 中找到，因为这个提交的项已被大多数接受
	- 对于没有在任何一个候选上提交的项，如果这个项也没有被任何一个候选保存，则 leader 可以安全地跳过，因为根据 ParallelRaft 或 Raft 机制，这个项不可能已经被提交
	- 如果某些候选人保存了一个未提交的项（ index 相同但 term 不同），则 leader 候选人选择其中其中 term 版本最高的，并认为该项有效
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250716104541.png)
1. Follower 候选者将其本地日志条目发送给 Leader 候选
2. Leader 候选者接收这些条目并与自己的日志条目合并
3. Leader 候选者和 Follower 候选者同步状态
4. Leader 候选人可以提交所有的条目，并通知 Follower 候选人提交
5. Leader 候选人升级为 Leader ，Follower 候选人也升级为 Follower
- 在 ParallelRaft 中，不定期地建立一个**检查点（checkpoint）**：检查点之前的所有日志项都应用于数据块，并且**允许该检查点包含一些在检查点之后提交的日志项**
- ParallelRaft **选择检查点最新的节点作为候选节点**，而不是日志最长的节点，以达到追赶的目的
##### 5.2.3 catch up
- ParallelRaft 把落后的 Follower 追上 Leader 的过程称为 catch up，有两种类型：
	- **Fast-catch-up**：Follower 和 Leader 差距较小时（差距小于上一个 checkpoint），仅同步日志
	- **Streaming-catch-up**：Follower 和 Leader 差距较大时（差距大于上一个 checkpoint），同步 checkpoint 和日志
![image.png](https://qingwu-oss.oss-cn-heyuan.aliyuncs.com/lian/img/20250716105557.png)
- Case 1 需要 streaming-catch-up
- Case 2 只用 fast-catch-up
- Case 3 中多出来的日志会被 Leader 覆盖掉，和 Raft 一样
