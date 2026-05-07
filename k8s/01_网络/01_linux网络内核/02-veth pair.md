## 1. veth pair

### 基本概念

**veth pair (Virtual Ethernet Pair)** 是 Linux 内核提供的虚拟网络设备，成对出现，逻辑上等同于一根 **“两端自带网卡的虚拟双绞线”**。

- **本质**：它是连接不同网络命名空间（Network Namespace）的 **二层隧道**。
- **作用**：打破 Namespace 的网络壁垒，实现容器与宿主机、容器与容器之间的联通。它是所有容器网络插件（CNI）实现通信的物理基石。

### 基本使用

```bash
# 创建veth pair, 在linux宿主机上创建一对veth pair.分别为 veth0 veth1
ip link add veth0 type veth peer name veth1

#启动虚拟网卡
ip link set dev veth0 up
ip link set dev veth1 up

#设置虚拟网卡ip
ifconfig veth0 192.168.7.10/24
ifconfig veth1 192.168.7.11/24

# 查看网卡信息
ip link show
# 9:veth1@veth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
#    link/ether ca:48:54:5c:53:b5 brd ff:ff:ff:ff:ff:ff
# 10: veth0@veth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
#    link/ether de:cb:54:1b:e7:d0 brd ff:ff:ff:ff:ff:ff

# 查看veth pair 对应的另一对
cat /sys/class/net/veth0/iflink #返回的是另一头虚拟网卡设备的id 这里返回的是9(veth1)
```

## 2. linux bridge