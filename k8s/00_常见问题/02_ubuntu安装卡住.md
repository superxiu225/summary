## ubuntu22.04 安装卡住

**问题描述：在msi主板，RTX4060上，安装ubunut22.04 卡住**

在 MSI 主板上使用 RTX 4060 安装 Ubuntu 22.04 时出现卡顿或死机，通常是由**硬件兼容性滞后**和**开源驱动冲突**共同导致的。

以下是针对您硬件配置的深度分析及直接解决方案。

### 1. 核心原因总结

| **原因分类**         | **具体说明**                                                 |
| -------------------- | ------------------------------------------------------------ |
| **Nouveau 驱动冲突** | Ubuntu 默认使用的开源驱动 `nouveau` 无法正确识别 RTX 4060（Ada Lovelace 架构），导致系统在加载图形界面时死锁。 |
| **内核版本过低**     | Ubuntu 22.04 早期 ISO 使用的 5.15 内核对 RTX 4060 的支持较差。建议使用 **22.04.3** 及以后的 HWE 内核版本。 |
| **MSI BIOS 限制**    | MSI 主板的 **Secure Boot (安全启动)** 可能阻止第三方 Nvidia 驱动加载；**VMD 控制器** 可能导致安装程序找不到硬盘。 |
| **ACPI 兼容性**      | 新型主板的电源管理协议与旧版 Linux 内核不匹配，导致启动阶段挂起。 |

------

### 2. 解决方案 (BLUF)

#### 第一步：绕过安装时的卡顿

在启动进入 Grub 菜单（那个黑底白字的选项界面）时：

1. 选中 **"Try or Install Ubuntu"**，按键盘上的 **`E`** 进入编辑模式。

2. 找到以 linux 开头的那一行，在末尾的 quiet splash 后面空格添加：

   nomodeset 或 module_blacklist=nouveau

3. 按 **`F10`** 或 **`Ctrl+X`** 保存并启动。

   > **Rationale:** 这将强制使用基础显卡模式（VGA）进入系统，避开导致死机的显卡驱动。

#### 第二步：MSI BIOS 设置

进入 BIOS (开机连续按 Del)：

- **Security:** 将 **Secure Boot** 设为 `Disabled`（或在安装驱动时确保正确配置 MOK 密钥）。
- **Advanced:** 检查 **SATA Mode/VMD Mode**，如果安装程序找不到硬盘，请将其从 RAID/VMD 更改为 `AHCI`。

#### 第三步：grub 设置

##### 第一步：进入 Root 终端

1. 开机后立即反复按下 **`Shift`** 键（部分 UEFI 启动的主板可能需要按 **`Esc`**）进入 GRUB 菜单。

2. 选择 **"Advanced options for Ubuntu"**。

3. 选择带有 **"(recovery mode)"** 字样的内核版本（通常是第二个选项）。

4. 在弹出的菜单中，使用方向键选择 **"root  Drop to root shell prompt"**，然后按回车。

5. **关键一步：** 此时文件系统通常是“只读”的，执行以下命令以获得修改权限：

   ```bash
   mount -o remount,rw /
   ```

##### 第二步：修改配置文件

由于 Recovery 模式下可能没有图形界面，建议使用 `nano`（比 `gedit` 更通用）：

1. 输入命令：

   ```bash
   nano /etc/default/grub
   ```

2. 移动光标找到：`GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"`

3. 修改为：`GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nomodeset"`

4. 保存并退出：按 **`Ctrl + O`** (保存)，按 **回车** 确认，再按 **`Ctrl + X`** (退出)。

##### 第三步：更新 GRUB 并重启

这是确保修改生效的核心步骤：

1. **执行更新：**

   ```bash
   update-grub
   ```

   *注意：此时系统会扫描硬盘，你会看到类似 "Found linux image: /boot/vmlinuz..." 的输出。*

2. **退出并重启：**

   ```bash
   exit
   ```

   然后在菜单选择 **"resume"** 或直接输入 `reboot` 重启系统。