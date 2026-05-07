## iptable

### 1.是什么：Netfilter 的“代言人”

**iptables** 是一个运行在用户态的命令行工具，其本质是 **Linux 内核 Netfilter 框架** 的配置接口。

- **Netfilter**：内核中的防火墙框架，在网络协议栈的特定位置埋下了“钩子”（Hooks）。
- **iptables**：负责将用户定义的规则写入这些钩子中，实现对数据包的精准操控。

## 2. 原理：四表五链（4 Tables & 5 Chains）

iptables 的执行逻辑可以概括为：**在正确的时机（链），做正确的事（表）。**

<span style='color:red'> **iptables 就是在内核三层的 5 个关口（链）上，按优先级翻阅 4 个功能口袋（表）里的规则，一旦命中“放行”或“丢弃”指令，数据包就立刻带着判决离开，不再看后续规则**。</span>

### **五链 (Chains)：处理的时机**

数据包在协议栈中流转时触发的五个关键节点：

- **PREROUTING**：数据包刚进入网卡，路由决策前。
- **INPUT**：数据包目标是本机进程。
- **FORWARD**：数据包目标不是本机，需要转发。
- **OUTPUT**：本机进程产生的包。
- **POSTROUTING**：路由决策后，准备发出网卡。

### **四表 (Tables)：处理的逻辑分类**

- **filter**：默认表。负责过滤（放行/拦截），是传统防火墙的核心。

- **nat**：负责地址转换（SNAT/DNAT）。这是 **K8s Service** 负载均衡的命脉。

- **mangle**：负责拆解报文并修改包头（如 TTL、TOS）。

- **raw**：优先级最高。负责配置包豁免连接跟踪（Connection Tracking）。

  <span style='color:red'> **表优先级：raw --> mangle --> nat --> filter**</span>

------

## 3. 有什么用：K8s 网络的底座

- **Service 访问控制**：利用 `filter` 表实现 Pod 间的安全策略。
- **负载均衡 (DNAT)**：当访问 Service IP 时，iptables 会根据规则将其重定向到真实的 Pod IP。
- **出口伪装 (Masquerade)**：让 Pod 能够共用节点的 IP 访问外网。
- **连接跟踪 (Conntrack)**：记录连接状态，确保回包能正确找到请求方。

------

## 4. 怎么用：万能语法公式

要快速掌握 iptables 的使用，你只需要记住这个**“五元组”公式**：

> **`iptables [-t table] COMMAND chain [matching-criteria] [-j target]`**

### **拆解使用逻辑：**

1. **`-t` (Table)**：指定表（不写默认为 `filter`）。
2. **`COMMAND`**：你要做什么？
   - `-A`：在末尾追加规则。
   - `-D`：删除规则。
   - `-L -n`：查看规则（`-n` 数字显示，速度更快）。
3. **`chain`**：在哪个节点拦截？（如 `INPUT` 或 `PREROUTING`）。
4. **`matching-criteria`**：过滤哪些包？
   - `-p`：协议（tcp, udp, icmp）。
   - `-s` / `-d`：源 IP / 目的 IP。
   - `--sport` / `--dport`：源端口 / 目的端口。
5. **`-j target`**：怎么处理？
   - `ACCEPT` / `DROP` / `REJECT`：接受/丢弃/拒绝。
   - `DNAT` / `SNAT` / `REDIRECT`：地址与端口转换。
   - `MASQUERADE`：动态 IP 伪装。

### 5. netfilter原理图

<span style='color:red'>**Netfilter 是“站岗在三层的哨兵”，但手里拿着“修改四层的手术刀”。**</span>

#### **精炼总结：Netfilter 的“越权手术”**

- **物理位置（在三层）**：Netfilter 的五条链（Hooks）物理上嵌入在 Linux 内核的 **IPv4/IPv6 协议栈代码**中。
- **检查能力（跨四层）**：虽然哨兵站在三层，但他可以解开包头，读取并匹配 **TCP/UDP 的端口信息**。
- **执行操作（改三层与四层）**：在执行 `NAT`（地址转换）或 `REDIRECT`（端口转发）时，内核会在三层的钩子点上**直接重写**报文内容：
  - **修改 IP 信息**（属于第三层网络层）。
  - **修改端口信息**（属于第四层传输层）。
- **执行时机**：这一切“手术”都发生在数据包进入真正的 **L4 协议栈**处理之前。

![image-20260209135228087](D:\xlx\note\summary_note\k8s\01_网络\01_linux网络内核\assets\image-20260209135228087.png)

