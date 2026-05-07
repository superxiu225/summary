# Git常见操作

##  1. Git：从零构建仓库并推送到远端

如果你本地有一堆 源码，想把它变成一个受版本控制的 Git 仓库并推送到 GitHub/GitLab：

1. **初始化本地库**： 进入你的项目目录（比如 `~/Documents/HOS_Core`）：

   ```bash
   git init
   ```

2. **暂存与提交**：

   ```bash
   git add .
   git commit -m "feat: init HOS system architecture"
   ```

3. **命名主分支**（建议统一使用 `main`）：

   ```bash
   git branch -M main
   ```

4. **关联远端并推送**：

   ```bash
   git remote add origin https://github.com/yourname/hos_repo.git
   git push -u origin main
   
   #修改远端地址
   git remote set-url origin https://github.com/yourname/hos_repo.git
   ```

## 2. HTTPS 模式下的“免密”配置

当 SSH 端口受限（比如在某些公司内网环境下）时，使用 HTTPS 配合**凭据管理器**是最稳妥的。

> **注意**：现在的 GitHub/GitLab 不再允许直接使用账号密码推送，你必须先去 Settings 生成一个 **Personal Access Token (PAT)**。

### 方案 A：永久存储（推荐）

Git 会将你的 Token 存储在本地磁盘上，下次直接调用。

- **Linux/Mac/Windows 通用指令**：

  ```bash
  git config --global credential.helper store
  git config --global http.sslVerify false
  
  #执行之后，直接带上账户密码进行拉去项目就会自动存储了
  git clone https://gitlab3.rd.supcon.com/xiuliangxing/helm-hos.git
  
  #需要填写账户和密码，账户就是你gitlab登录的账户，密码就是你账户内通过第一步获取的token
  正克隆到 'central-service'...
  Username for 'https://gitlab3.rd.supcon.com': xiuliangxing
  Password for 'https://xiuliangxing@gitlab3.rd.supcon.com': pt_v6-DmF7szaQyywtjY
  ```

  *执行后，下次 `push` 时手动输入一次用户名和 Token，之后就再也不用了。*

### 方案 B：使用系统原生管理器（更安全）

- **Mac 用户**：Mac 会自动调用“钥匙串 (Keychain)”。

- **Windows 用户**：安装 **Git Credential Manager**。

  ```bash
  git config --global credential.helper manager
  ```

## 3. GitLab Runner 深度配置指南

在你的 HOS 架构中，Runner 相当于执行 CI 任务的“工蜂”。

### 下载与安装 (以 Mac/Linux 为例)

1. **下载二进制文件**：

   ```bash
   sudo curl -L --output /usr/local/bin/gitlab-runner "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-darwin-amd64"
   sudo chmod +x /usr/local/bin/gitlab-runner
   ```

2. **安装并启动服务**：

   ```bash
   gitlab-runner install
   gitlab-runner start
   ```

### 三种 Token 的获取与区别

在 GitLab 中，Token 决定了 Runner 的“职权范围”：

| **类型**                    | **获取路径**                        | **作用范围 (Scope)**                             | **适用场景**                                     |
| --------------------------- | ----------------------------------- | ------------------------------------------------ | ------------------------------------------------ |
| **Instance (Shared) Token** | 管理中心 -> Runner                  | **全站可用**。只要是这个 GitLab 上的项目都能用。 | 适合通用的编译环境，多个项目共用。               |
| **Group Token**             | 群组 -> 设置 -> CI/CD -> Runner     | **整个组可用**。该文件夹下的所有子项目共享。     | 适合你的 HOS 团队，内部多个子模块共用。          |
| **Project Token**           | 具体项目 -> 设置 -> CI/CD -> Runner | **仅限该项目**。其他项目无法调用。               | 适合有特殊硬件需求（如特定显卡环境）的仿真项目。 |

### 如何配置（注册）？

拿到 Token 后，在终端输入：

```bash
sudo gitlab-runner register --non-interactive \
  --url "https://gitlab3.rd.supcon.com" \
  --registration-token "GR1348941YNYsRiWsyVsyBsuPwWoR" \
  --executor "shell" \
  --description "ros_shell_157" \
  --tag-list "ros_shell_157" \
  --run-untagged="true" \
  --locked="false" \
  --access-level="not_protected" \
  --tls-ca-file="/data/crt/_wildcard.rd.supcon.com.crt"
```

### 如何修改配置？

1. **编辑配置文件**：找到 GitLab Runner 的配置文件，通常在 `/etc/gitlab-runner/config.toml`。使用文本编辑器打开它，例如：

   ```shell
    sudo vim /etc/gitlab-runner/config.toml
   ```

