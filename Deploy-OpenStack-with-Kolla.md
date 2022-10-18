# 环境介绍

| 角色                          | 系统              | 版本   | 硬件配置           | 最小配置需求       | 备注                           |
| --------------------------- | --------------- | ---- | -------------- | ------------ | ---------------------------- |
| controller\|neutron\|cinder | CentOS Stream 8 | Yoga | 16C\|32G\|2NIC | 4C\|8G\|2NIC | 为Cinder额外增加了一个500G的盘/dev/sdb |
| controller\|neutron\|cinder | CentOS Stream 8 | Yoga | 16C\|32G\|2NIC | 4C\|8G\|2NIC | 为Cinder额外增加了一个500G的盘/dev/sdb |
| controller\|neutron\|cinder | CentOS Stream 8 | Yoga | 16C\|32G\|2NIC | 4C\|8G\|2NIC | 为Cinder额外增加了一个500G的盘/dev/sdb |
| Compute                     | CentOS Stream 8 | Yoga | 16C\|32G\|2NIC | 4C\|8G\|2NIC |                              |
| Compute                     | CentOS Stream 8 | Yoga | 16C\|32G\|2NIC | 4C\|8G\|2NIC |                              |

# 设置主机名

各个节点完成主机名设置，而且每个节点的root密码都为1

```bash
hostnamectl set-hostname control1
```

# 准备hosts文件

各个节点完成hosts设置

```bash
cat > /etc/hosts <<EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.50.10 kolla
172.16.50.101 control1
172.16.50.102 control2
172.16.50.103 control3
172.16.50.104 compute1
172.16.50.105 compute2
EOF
```

# Kolla 部署

## 安装先决条件

```bash
dnf install python3-devel libffi-devel gcc openssl-devel python3-libselinux python3-pip git sshpass wget vim bash-completion -y
pip3 install -U pip
pip3 install ansible -U pip
```

## 完成时间同步

```bash
for server in  kolla control1 control2 control3 compute1 compute2; \
do sshpass -p 1 ssh -A -g -o StrictHostKeyChecking=no root@$server \
dnf install chrony -y;ssh -A -g -o StrictHostKeyChecking=no root@$server systemctl enable chronyd --now ;done
```

## 为Cinder创建VG

这一步仅限于稍后将cinder后端设置为lvm时才需要做

```bash
for server in control1 control2 control3; \
do sshpass -p 1 ssh -A -g -o StrictHostKeyChecking=no root@$server \
vgcreate cinder-volumes /dev/sdb;done
```

## 安装kolla

```bash
pip3 install git+https://opendev.org/openstack/kolla-ansible@stable/yoga
```

```bash
mkdir -p /etc/kolla
chown $USER:$USER /etc/kolla 
cp -r /usr/local/share/kolla-ansible/etc_examples/kolla/* /etc/kolla 
cp /usr/local/share/kolla-ansible/ansible/inventory/* . 
git clone --branch stable/yoga https://opendev.org/openstack/kolla-ansible
pip3 install ./kolla-ansible 
```

## 安装Ansible galaxy依赖

```bash
kolla-ansible install-deps
```

## 配置Ansible配置文件

```bash
mkdir /etc/ansible
cat > /etc/ansible/ansible.cfg <<EOF
[defaults]
host_key_checking=False
pipelining=True
forks=100
EOF
```

## 配置清单文件

当前目录一共有两个清单文件，all-in-one和multinode，后者可以配置多个站点

下面是我修改的multinode清单

```textile
[control]
control1 ansible_become=true
control2 ansible_become=true
control3 ansible_become=true
# Ansible supports syntax like [10:12] - that means 10, 11 and 12.
# Become clause means "use sudo".

[network:children]
control
# when you specify group_name:children, it will use contents of group specified.

[compute]
compute1 ansible_become=true
compute2 ansible_become=true
[monitoring]
control1
# This group is for monitoring node.
# Fill it with one of the controllers' IP address or some others.

[storage:children]
control

[deployment]
localhost       ansible_connection=local become=true
# use localhost and sudo 

...下面的没改
```

## 生成并分发ssh密钥

```bash
ssh-keygen -f /root/.ssh/id_rsa -N ''
for server in  kolla control1 control2 control3 compute1 compute2; do sshpass -p 1 ssh-copy-id -o StrictHostKeyChecking=no root@$server ;done
```

测试一下机器是否在线

```bash
ansible -i multinode all -m ping
```

## 生成koll密码

```bash
kolla-genpwd -p /etc/kolla/passwords.yml
```

## 修改全局配置文件

```bash
# 定义安装平台
sed -i 's/#kolla_base_distro: "centos"/kolla_base_distro: "centos"/'  /etc/kolla/globals.yml
# 定义安装类型
sed -i 's/#kolla_install_type: "source"/kolla_install_type: "source"/'  /etc/kolla/globals.yml
# 定义管理网络
sed -i 's/#network_interface: "eth0"/network_interface: "ens192"/'  /etc/kolla/globals.yml
# 定义外部公开网络，这个网络要求没有IP地址
sed -i 's/#neutron_external_interface: "eth1"/neutron_external_interface: "ens224"/'  /etc/kolla/globals.yml
# 定义管理用的浮动IP，此IP将有keepalive实现高可用，此IP应该属于管理网络
sed -i 's/^#kolla_internal_vip_address.*/kolla_internal_vip_address: "172.16.50.240"/'  /etc/kolla/globals.yml
# 定义是否启用cinder
sed -i 's/#enable_cinder: .*/enable_cinder: "yes"/'  /etc/kolla/globals.yml 
sed -i '/enable_cinder:.*/a\enable_cinder_backend_lvm: "yes"'  /etc/kolla/globals.yml
```

# OpenStack 部署

```bash
# 为后期部署安装依赖
kolla-ansible -i ./multinode bootstrap-servers
# 为后续部署执行预部署条件检测
kolla-ansible -i ./multinode prechecks
# 真实的完成最终部署
kolla-ansible -i ./multinode deploy
# 生成凭据文件,将产生/etc/kolla/clouds.yaml文件，可以复制到/etc/openstack或~/.config/openstack，也 \
# 可以设置OS_CLIENT_CONFIG_FILE环境变量使用
kolla-ansible post-deploy
```

# 创建样例资源

执行后，将会创建样例网络、镜像等资源，请勿用于生产环境

```bash
/usr/local/share/kolla-ansible/init-runonce
```
