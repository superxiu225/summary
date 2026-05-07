### 阶段 1：Linux 网络 + K8s Service（你已经在走）

> 打穿数据路径

------

### 阶段 2：eBPF（必须在这里）

> 把网络从“能跑”升级到“可控、可观测”

**原因一句话**：
 👉 *没有 eBPF，你进入 AI 网络后会“看不见问题”。*

------

### 阶段 3：RDMA / RoCE / Topology

> AI 训练数据平面

------

### 阶段 4：Multus + SR-IOV + AI 网络整体架构

> 解决“怎么在 K8s 里优雅地用 RDMA”



除了 K8s 网络，以下是你必须攻克的 **5大核心知识模块**。这也是目前市场上年薪百万的智算架构师的“看家本领”。

------

### 1. 高性能网络协议：RDMA 与 InfiniBand

**这是智算中心与普通数据中心最大的区别。** 在大模型训练中，GPU 之间的通信量极大，传统的 TCP/IP 协议延迟太高、CPU 开销太大，根本跑不动。

- **必须掌握的概念：**
  - **RDMA (Remote Direct Memory Access):** 也就是“零拷贝”技术。你需要理解它如何绕过 CPU，直接让显存与显存通信。
  - **RoCE v2 (RDMA over Converged Ethernet):** 如何在以太网上跑 RDMA？（这是目前性价比最高的主流方案）。
  - **InfiniBand (IB):** NVIDIA 原生的无损网络，你需要懂 IB 的子网管理 (Subnet Manager) 和它的 Credit 机制。
  - **PFC (Priority Flow Control) & ECN (Explicit Congestion Notification):** 怎么解决网络拥塞？这是智算网络调优中最痛苦也是最值钱的部分。
- **推荐学习路径：**
  - 阅读 NVIDIA Networking (Mellanox) 的官方白皮书。
  - 研究 **"Lossless Ethernet" (无损以太网)** 的配置最佳实践。

### 2. GPU 通信库与拓扑：NCCL

光懂网络协议不行，你得懂应用层怎么用网络。**NCCL (NVIDIA Collective Communications Library)** 是所有大模型训练的通信基石。

- **必须掌握的概念：**
  - **通信原语 (Primitives):** 什么是 All-Reduce, All-Gather, Broadcast, Reduce-Scatter？你需要脑补出数据在 GPU 之间是怎么流动的。
  - **拓扑感知 (Topology Awareness):** 为什么同一台机器内的卡用 NVLink，跨机器用网卡？如何利用 `ncc-test` 工具测试带宽？
  - **Ring vs. Tree 算法:** 数据是在围着圈转，还是按树状分发？这直接影响网络架构设计（Fat-Tree 还是 Rail-only）。
- **实战场景：**
  - 当训练速度只有预期的 50% 时，你能通过 NCCL 的日志分析出是哪根网线或者哪个 Switch 成了瓶颈。

### 3. 分布式训练并行策略

作为架构师，你不需要会写 Transformer 代码，但你必须懂**“模型是怎么被切分”**的。这决定了你需要多少显存、多大带宽。

- **必须掌握的概念：**
  - **3D 并行 (3D Parallelism):**
    1. **数据并行 (Data Parallelism, DP):** 大家都跑一样的模型，吃不一样的数据。
    2. **张量并行 (Tensor Parallelism, TP):** 一个算子切开，跨卡计算（通信量巨大，依赖 NVLink）。
    3. **流水线并行 (Pipeline Parallelism, PP):** 模型分层，跨节点串行（依赖网卡）。
  - **ZeRO (Zero Redundancy Optimizer):** DeepSpeed 是怎么节省显存的？
- **推荐资源：**
  - 阅读 **Megatron-LM** 和 **DeepSpeed** 的架构文档。

### 4. 智算存储：高性能并行文件系统

大模型训练有两个存储痛点：

1. **小文件加载：** 训练集可能有几十亿张小图，随机读（IOPS）要求极高。
2. **Checkpoint (断点续训)：** 几千张卡的显存数据要在一瞬间（几秒内）写入硬盘，吞吐量（Throughput）要求极高。

- **需要学习的技术栈：**
  - **并行文件系统：** Lustre, GPFS (IBM Spectrum Scale), JuiceFS (云原生方向)。
  - **缓存层：** Alluxio 或各类分布式缓存加速方案。
  - 你需要知道如何计算 **IO 墙 (I/O Wall)**：如果 Checkpoint 需要写 10TB 数据，你的存储带宽如果是 10GB/s，那么训练就要暂停 1000秒，这是不可接受的。

### 5. 智算调度器 (AI Scheduler)

K8s 原生的调度器（Default Scheduler）是为微服务设计的，不适合 AI。你需要学习专门针对 Batch Job 的调度。

- **关键组件：**
  - **Volcano / Kueue (K8s 生态):** 如何做 Gang Scheduling (帮派调度)？即“我们要么 100 张卡一起拿到资源开跑，要么一张卡都别给我（防止死锁）”。
  - **Slurm (HPC 生态):** 传统超算中心的调度之王，现在很多智算中心依然在用，或者是 K8s + Slurm 的混合架构。
- **核心逻辑：** 拓扑感知调度。调度器需要知道“这8个Pod必须跑在同一个交换机下”，否则跨交换机通信会拖慢整体训练。





### Level 1: 物理底座与拓扑 (The Anatomy)

**目标：** 看懂机房图纸，理解电流和数据流是怎么跑的。

- **核心知识点：**
  - **GPU 架构解剖：** 深入理解 NVIDIA Hopper/Blackwell 架构（H100/H800/B200）。搞懂 SM（流多处理器）、HBM（高带宽内存）、NVLink 互联带宽。
  - **服务器拓扑：** 什么是 PCIe Switch？什么是 NVLink Switch？为什么 8卡模组必须这样连？CPU与GPU之间的 NUMA 亲和性（Affinity）怎么影响性能？
  - **集群网络拓扑：** 胖树（Fat-Tree）架构 vs. Rail-Only 架构。为什么要分计算网（Compute Fabric）和存储网（Storage Fabric）？
- **必读资料：**
  - 📜 **NVIDIA H100 Tensor Core GPU Architecture Whitepaper**（官方白皮书，必读圣经）。
  - 📜 **NVIDIA DGX H100 System Architecture**（学习如何设计一台顶级服务器）。
- **🛠️ 动手实践 (Lab 1)：**
  - **任务：** 使用 `nvidia-smi topo -m` 和 `lspci -tv` 命令。
  - **操作：** 在你的 Orin 或者随便找一台云端 GPU 服务器上运行这些命令。
  - **目标：** 亲手画出该机器的 **PCIe 拓扑图**。解释为什么某些 GPU 之间显示 `NV` (NVLink)，有些显示 `PIX` (PCIe Switch内部)，有些显示 `PHB` (跨Host Bridge)。如果你看不懂这个输出，就无法做性能调优。

------

### Level 2: 高性能网络核心 (The Circulatory System)

**目标：** 解决智算的“心脏病”——网络拥塞与延迟。这是你提到的 RoCEv2 所在的层级。

- **核心知识点：**
  - **RDMA 机制：** 彻底搞懂 Zero-Copy 和 Kernel Bypass。
  - **RoCEv2 调优：** 深入理解 PFC（流量控制）和 ECN（拥塞控制）的配合机制。什么是 Head-of-Line Blocking（队头阻塞）？
  - **通信库 NCCL：** 必须看懂 NCCL 的 Ring 和 Tree 算法。理解 All-Reduce 在不同网络拓扑下的带宽需求。
- **必读资料：**
  - 📖 **《RDMA over Converged Ethernet (RoCE) Deployment Guide》** (NVIDIA 或 Cisco 出品)。
  - 📜 **NVIDIA NCCL Developer Guide** (重点看 "Communication Algorithms" 章节)。
  - 🔍 **GTC 演讲视频：** 搜索 "Scaling Deep Learning with NCCL"。
- **🛠️ 动手实践 (Lab 2)：网络仿真与性能测试**
  - **环境：** 你的 HOS 混合集群（x86 + Orin）。
  - **任务 A：** 部署 **Containerlab**。它可以让你在 Docker 里模拟复杂的网络交换机拓扑。尝试模拟一个 Spine-Leaf 架构。
  - **任务 B：** 运行 `ncc-test` (NVIDIA Collective Communication Test)。
  - **目标：** 在两台机器间跑 `all_reduce_perf`。记录带宽数据。尝试人为制造网络丢包（比如用 `tc` 命令），观察 NCCL 性能是如何断崖式下跌的，体验“RDMA 对丢包的敏感性”。

------

### Level 3: 调度与编排 (The Nervous System)

**目标：** 让一万张卡像一支军队一样整齐划一地行动。

- **核心知识点：**
  - **AI 调度器：** 学习 **Volcano** 或 **Kueue**。理解 Gang Scheduling（帮派调度）、Binpack（装箱策略）、Topology Aware Scheduling（拓扑感知调度）。
  - **断点续训 (Checkpointing)：** 理解大模型训练的 I/O 模式（周期性突发写）。
  - **异构纳管：** 如何在同一个 K8s 集群里管理 NVIDIA GPU、华为昇腾 NPU 和 CPU 节点，并打上 `Taint` 和 `Toleration`。
- **必读资料：**
  - 📖 **Volcano 官方文档** (重点看 Architecture 和 Queue/PodGroup 概念)。
  - 📖 **Slurm Workload Manager 文档** (了解传统 HPC 是怎么调度的，对比 K8s 的优劣)。
- **🛠️ 动手实践 (Lab 3)：手写调度策略**
  - **环境：** 你的 K8s 集群。
  - **任务：** 安装 Volcano Scheduler 替换默认 K8s 调度器。
  - **目标：** 定义一个 `PodGroup`，要求最小成员数（`minMember`）为 2。故意只给 1 个节点的资源。观察 Pod 的状态——它应该处于 `Pending` 而不是部分启动（这就是 Gang Scheduling 的作用）。
  - **进阶：** 尝试编写一个简单的 K8s Operator，自动为挂掉的训练任务重新挂载之前的 Checkpoint PVC。

------

### Level 4: 训练框架与并行策略 (The Muscle)

**目标：** 理解应用层（大模型）是如何“切分”自己的，以便你在底层配合它。

- **核心知识点：**
  - **3D 并行原理：** 数据并行 (DP)、张量并行 (TP)、流水线并行 (PP)。
  - **显存墙分析：** 会算算账。一个 175B 的模型，用 Adam 优化器，FP16 精度，到底需要多少显存？（公式：参数量 * 字节数 + 梯度 + 优化器状态 + 激活值）。
  - **Megatron-LM & DeepSpeed：** 只要看懂架构图，不需要会写代码。
- **必读资料：**
  - 📜 **The Megatron-LM Paper** (Efficient Large-Scale Language Model Training on GPU Clusters).
  - 📜 **DeepSpeed ZeRO Paper** (理解它是怎么把显存占用压缩到极致的)。
- **🛠️ 动手实践 (Lab 4)：微缩版分布式训练**
  - **环境：** 哪怕你只有 2 张卡（或者 1 台服务器 + 1 台 Orin）。
  - **任务：** 跑通一个微缩版的 **Llama 3 微调** 任务，使用 **DeepSpeed ZeRO-2**。
  - **目标：**
    1. 开启 NVLink/RDMA 时看 `nvlink -s` 或网卡流量。
    2. 关闭它们，强制走 TCP。
    3. **对比两者的 Token/s 速度差异**。这就是你作为架构师向客户证明“为什么要买昂贵网络设备”的最有力数据。