```textile
作者：李晓辉

微信：Lxh_Chat

邮箱：939958092@qq.com
```

# 环境介绍
|CPU|内存|硬盘|系统|OpenStack版本|
|-|-|-|-|-|
|4C+|16G+|100G+|CentOS9 Stream|zed|

# 准备软件（在线版）
```bash
dnf config-manager --enable crb
dnf install -y centos-release-openstack-zed
```
# 准备软件（离线版）
这里的离线版仅指在安装过程中，不再需要去外网拉取rpm包，但是还需要保持外网连接，以便于安装过程中自动下载cirros等测试资源

下载离线包：https://pan.baidu.com/s/1uLSv5cnsC1pOmzbzNCcAnQ?pwd=8nus
```bash
mount /xxx.iso /mnt
mv /etc/yum.repos.d/* /opt
cat > /etc/yum.repos.d/ops.repo <<EOF
[ops]
name=ops
baseurl=file:///mnt
enabled=1
gpgcheck=0
EOF
```

# 关闭防火墙和SELINUX
```bash
systemctl disable firewalld --now
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
```

# 部署OpenStack
```bash
# 先安装部署工具
dnf install -y openstack-packstack

# 安装OpenStack
packstack --allinone
```

# 查询admin密码

安装完成后，会有输出告诉你密码在哪里，而且在root家目录，会有一个keystonerc_admin文件，其中也会明确写明admin的密码是多少
