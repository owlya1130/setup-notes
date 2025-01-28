# NFS

## NFS Server
```shell
# install packages
apt update
apt install nfs-kernel-server

# 创建共享目录
mkdir -p /share

# 设置权限（根据需求调整）：
chown nobody:nogroup /share
chmod 777 /share

# 编辑 NFS 配置文件： 在 /etc/exports 文件中添加共享目录配置。例如：
vim /etc/exports
# 添加以下内容（调整 192.168.1.0/24 为您的客户端子网范围）：
## 选项说明
## rw：读写权限。
## sync：确保更改同步到磁盘。
## no_subtree_check：提高性能，减少目录检查。
/share 192.168.1.0/24(rw,sync,no_subtree_check)

#应用配置：
sudo exportfs -ra
# 检查共享配置：
sudo exportfs -v

# 配置防火墙
sudo ufw allow from 192.168.1.0/24 to any port nfs
sudo ufw reload

# 启动 NFS 服务
sudo systemctl enable --now nfs-server
```

## NFS Client
```shell
# install packages
apt update
apt install nfs-common

# 创建挂载点
mkdir -p /share

# 挂载共享目录：
NFS_SERVER=192.168.1.101
mount $NFS_SERVER:/share /share

# 验证挂载：
df -h

# 设置开机自动挂载： 
vim /etc/fstab
# 在 /etc/fstab 文件中添加以下内容：
## "$NFS_SERVER" 需自行修改為實際IP
$NFS_SERVER:/share /share	 nfs defaults 0 0

mount -a
```