## ubunut22.04 安装 nvidia驱动

#### 第一步：查看推荐及可用版本

使用 Ubuntu 官方工具自动检测显卡型号并匹配源中的驱动：

```bash
sudo apt update
sudo ubuntu-drivers devices
```

#### 第二步：执行安装

假设系统推荐的是 **535** 版本（请根据上一步的实际输出来替换数字）：

```bash
# -y 表示自动确认，确保脚本不中断
sudo apt install -y nvidia-driver-535-server
```

#### 第三步：免重启热加载 (Critical Step)

安装完成后，因为不能重启，你需要手动将驱动模块“注入”内核。**请依次执行以下命令，缺一不可：**

```bash
# 1. 重新构建模块依赖关系图
sudo depmod -a

# 2. 按顺序加载内核模块 (顺序极其重要)
sudo modprobe nvidia
sudo modprobe nvidia_uvm
sudo modprobe nvidia_modeset
sudo modprobe nvidia_drm

# 3. 初始化设备节点并启动持久化守护进程
sudo nvidia-persistenced --persistence-mode

# 4. 验证
nvidia-smi
```

#### 第四步：禁止驱动自动升级

**必须设置。** 作为 K8s Master 节点，**确定性 (Determinism)** 高于一切。自动升级导致的驱动与内核版本不匹配（Kernel Mismatch）是导致 GPU 节点掉线的头号杀手。

```bash
# 1. 锁定所有包含 'nvidia' 关键词的已安装包
sudo apt-mark hold $(dpkg -l | grep nvidia | awk '{print $2}')

# 2. 锁定内核头文件与核心（可选但推荐，防止内核自动升级导致驱动需要重编译）
# 警告：锁定内核可能导致错过安全补丁，需权衡。对于纯内网AI节点，通常建议锁定。
sudo apt-mark hold $(dpkg -l | grep -E "linux-image|linux-headers" | awk '{print $2}')

# 3. 验证锁定状态
apt-mark showhold
```

#### 第五步：手动升级nvidia驱动（仅为手动升级时操作）

**当且仅当**你需要主动升级驱动（例如为了支持新的 CUDA 版本）时，使用以下命令解锁：

```bash
# 解锁所有 nvidia 包
sudo apt-mark unhold $(dpkg -l | grep nvidia | awk '{print $2}')

# 然后正常升级
sudo apt update && sudo apt upgrade
```



## 国内安装nvidia-toolkit

#### 第一步：彻底清理旧的“毒素”

之前尝试失败的配置会干扰 `apt`。

```bash
# 1. 删掉所有之前尝试失败的 list 文件
sudo rm -f /etc/apt/sources.list.d/nvidia-container-toolkit.list

# 2. 清除 apt 缓存
sudo apt-get clean
sudo apt-get update
```

------

#### 第二步：使用 USTC (中科大) 2026 最新路径安装

重点在于路径：**不要在 URL 里加 `ubuntu22.04`**，现在的标准路径是 `stable/deb`。

```bash
# 1. 下载国内镜像站的 GPG 秘钥 (防止被墙)
curl -fsSL https://mirrors.ustc.edu.cn/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

# 2. 写入正确的 2026 镜像源路径 (注意末尾的 deb/amd64)
echo "deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://mirrors.ustc.edu.cn/libnvidia-container/stable/deb/amd64 /" | \
sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# 3. 安装工具包 (2026年3月对应版本通常为 1.18.x)
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

------

#### 第三步：一键“焊死” 570.172.18 驱动

作为 AI 解决方案架构师，你最不希望看到的就是明天一个 `apt upgrade` 把你的驱动升到了 580，导致 Isaac 容器直接罢工。

执行以下命令，将当前 570 驱动的所有组件全部锁定（Hold）：

```bash
# 获取当前所有 570 系列已安装包并锁定
dpkg -l | grep nvidia | grep 570 | awk '{print $2}' | xargs sudo apt-mark hold

# 补充锁定核心组件 (确保万无一失)
sudo apt-mark hold nvidia-driver-570 nvidia-dkms-570 nvidia-utils-570 libnvidia-compute-570 libnvidia-common-570
```

*提示：执行 `apt-mark showhold` 确认列表里全是 570 的包。*

#### 第四步：激活 Docker GPU 运行时

安装了包之后，如果不配置 `daemon.json`，Docker 还是瞎子。

```bash
# 自动修改 /etc/docker/daemon.json
sudo nvidia-ctk runtime configure --runtime=docker

# 重启 Docker
sudo systemctl restart docker

# 检查 Docker 是否能透视到宿主机的 5880 显卡
docker run --rm --gpus all ubuntu:22.04 nvidia-smi
```
