#### nfs 的作用

  nfs文件系统是一种协议：主要是允许多个客户端通过网络访问存储在远程服务器的文件的协议。通过这种协议可以在远程服务器上统一管理文件。客户端通过连接到远程服务器进而进行文件共享。



#### nfs-server 的 安装

在 CentOS 7 上搭建 NFS 文件系统涉及几个主要步骤：安装 NFS 服务、配置 NFS 共享目录、设置适当的文件权限和安全规则、启动 NFS 服务，并确保服务在系统启动时自动运行。下面是详细的步骤：

1. **安装 NFS 服务** 打开终端，使用以下命令安装 NFS 服务：

   ```shell
    # 查看nfs是否存在
    rpm -qa | grep nfs
    
    # 如果nfs不存在，则安装它
    sudo yum install -y nfs-utils
   ```

2. **启动并启用 NFS 服务** 安装完成后，启动 NFS 服务并设置为开机启动：

   ```shell
    sudo systemctl start nfs-server
    sudo systemctl enable nfs-server
   ```

3. **创建共享目录** 选择或创建一个目录来作为 NFS 共享目录。例如，如果您想共享 `/srv/nfs` 目录，您需要创建这个目录（如果它还不存在）：

   ```shell
   sudo mkdir -p /srv/nfs
   ```

4. **配置共享目录** 编辑 NFS 导出配置文件 `/etc/exports`，在文件中添加您的共享目录配置。例如，如果您想让局域网中的所有计算机都能访问 `/srv/nfs` 目录，可以添加如下行：

   ```shell
   /srv/nfs *(rw,sync,no_root_squash,no_subtree_check)
   ```

   解释：

   - `rw`: 允许读写访问
   - `sync`: 同步写入磁盘
   - `no_root_squash`: 允许远程用户以 root 身份访问
   - `no_subtree_check`: 防止子树检查，可以提高性能

5. **导出共享目录** 修改 `/etc/exports` 文件后，需要使配置生效：

   ```shell
   sudo exportfs -arv
   ```

6. **设置防火墙规则** 如果您的系统使用防火墙，需要添加规则以允许 NFS 通信：

   ```shell
   sudo firewall-cmd --permanent --add-service=nfs
   sudo firewall-cmd --permanent --add-service=mountd
   sudo firewall-cmd --permanent --add-service=rpc-bind
   sudo firewall-cmd --reload
   ```

7. **测试 NFS 挂载** 在客户端机器上，尝试挂载刚才配置的 NFS 共享目录来验证设置：

   ```shell
   #showmount命令可以查询 NFS 服务器导出的共享列表
   showmount -e <nfs-server-ip>
   
   # 客户端文件与nfs server 文件进行绑定，其中 <nfs-server-ip>应替换为实际的 NFS 服务器 IP 地址
   sudo mount -t nfs <nfs-server-ip>:/srv/nfs /mnt
   ```

   

#### nfs-client 的安装

1. **安装 NFS 客户端工具**： 使用以下命令安装 `nfs-utils`：

   ```shell
   rpm -qa | grep nfs 
   # 如果存在则不需要安装
   sudo yum install -y nfs-utils
   ```

2. **启动相关服务**： 安装完成后，您可能需要启动 RPC 绑定服务以确保 NFS 客户端正常工作：

   ```shell
   sudo systemctl start rpcbind
   sudo systemctl enable rpcbind
   ```

3. **挂载 NFS 共享目录**： 一旦 NFS 客户端安装完成，您可以尝试挂载 NFS 服务器上的共享目录。使用如下命令：

   ```shell
   sudo mount -t nfs <nfs-server-ip>:/path/to/nfs/share /local/mount/point
   ```

   其中 `<nfs-server-ip>` 是 NFS 服务器的 IP 地址或主机名，`/path/to/nfs/share` 是 NFS 服务器上的共享目录路径，`/local/mount/point` 是本地系统上用于挂载 NFS 共享的目录。

#### 小结

```powershell
nfs 是个共享文件系统，server 通过挂载出多个文件，client可以去mount 到本地，实现server 和 client 的文件共享。
```