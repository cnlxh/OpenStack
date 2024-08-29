```textile
作者：李晓辉

微信：Lxh_Chat

邮箱：939958092@qq.com
```

# 环境介绍

| 角色                  | 主机名      | 系统            | 版本             | 硬件配置           |
| ------------------- | -------- | ------------- | -------------- | -------------- |
| controller\|kolla   | control1 | Rocky 9.4     | stable/2024.1  | 16C\|32G\|2NIC |
| controller\|compute | control2 | Rocky 9.4     | stable/2024.1  | 16C\|32G\|2NIC |
| controller\|compute | control3 | Rocky 9.4     | stable/2024.1  | 16C\|32G\|2NIC |
| Compute             | control2 | stable/2024.1 | 16C\|32G\|2NIC | 4C\|8G\|2NIC   |
| Compute             | control3 | stable/2024.1 | 16C\|32G\|2NIC | 4C\|8G\|2NIC   |

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
172.25.254.129 kolla
172.25.254.129 control1
172.25.254.130 control2
172.25.254.131 control3
EOF
```

**除非另有说明，不然下面所有的步骤都在control1上执行，因为我将kolla工具安装在control1上**

# Kolla 部署

## 安装先决条件

```bash
dnf install python3-devel libffi-devel gcc openssl-devel python3-libselinux python3-pip git sshpass wget vim bash-completion -y
pip3 install -U pip -i https://pypi.tuna.tsinghua.edu.cn/simple/
pip3 install ansible -i https://pypi.tuna.tsinghua.edu.cn/simple/
```

## 完成时间同步

```bash
for server in  kolla control1 control2 control3; \
do sshpass -p 1 ssh -A -g -o StrictHostKeyChecking=no root@$server \
dnf install chrony -y;sshpass -p 1 ssh -A -g -o StrictHostKeyChecking=no root@$server systemctl enable chronyd --now ;done
```

## 安装kolla

```bash
pip3 install git+https://opendev.org/openstack/kolla-ansible@stable/2024.1
```

```bash
mkdir -p /etc/kolla
chown $USER:$USER /etc/kolla 
cp -r /usr/local/share/kolla-ansible/etc_examples/kolla/* /etc/kolla 
cp /usr/local/share/kolla-ansible/ansible/inventory/* . 
```

## 生成并分发ssh密钥

```bash
ssh-keygen -f /root/.ssh/id_rsa -N ''
for server in  kolla control1 control2 control3; do sshpass -p 1 ssh-copy-id -o StrictHostKeyChecking=no root@$server ;done
```

## 配置清单文件

当前目录一共有两个清单文件，all-in-one和multinode，后者可以配置多个站点

下面是我修改的multinode清单

```textile
[control]
control1
control2
control3

[network]
control1
control2
control3

[compute]
control2
control3

[monitoring]
control1
control2
control3

[storage]
control1
control2
control3

[deployment]
localhost       ansible_connection=local

...下面的没改
```

测试一下机器是否在线

```bash
ansible -i multinode all -m ping
```

## 修改全局配置文件

```bash
vim /etc/kolla/globals.yml
```

```textile
[root@control1 ~]# grep -v -e ^$ -e ^# /etc/kolla/globals.yml
---
workaround_ansible_issue_8743: yes
kolla_base_distro: "rocky" #这是定义了按照平台
kolla_internal_vip_address: "172.25.254.200" # 管理网络中目前没有被占用的IP
network_interface: "ens224" #不能上网的管理接口
neutron_external_interface: "ens160" #可以上网的外部接口
```

## 安装Ansible galaxy依赖

```bash
kolla-ansible install-deps
```

## 生成koll密码

```bash
kolla-genpwd
```

## 修正docker安装

国内无法从docker官方安装，改一下到南京大学

```bash
vim /root/.ansible/collections/ansible_collections/openstack/kolla/roles/docker/defaults/main.yml
```

```text
docker_yum_baseurl: https://mirrors.nju.edu.cn/docker-ce/linux/centos/9.4/x86_64/stable
docker_yum_gpgcheck: false
```

# OpenStack 部署

```bash
# 为后期部署安装依赖
kolla-ansible -i ./multinode bootstrap-servers
# 为后续部署执行预部署条件检测
kolla-ansible -i ./multinode prechecks
# 真实的完成最终部署
kolla-ansible -i ./multinode deploy
# 生成凭据文件,将产生/etc/kolla/admin-openrc.sh文件
kolla-ansible post-deploy
```

# 访问OpenStack

根据我们的配置文件，在浏览器打开你的vip地址即可

```text
https://172.25.254.200
```

管理员是admin，密码如下所示：

```text
[root@control1 ~]# grep admin /etc/kolla/passwords.yml
...
keystone_admin_password: PCG3fyD9QQw0E5XWjE01ibBFxdLBxVRoCB8omDQf
```

# 获取测试镜像

```textile
https://docs.openstack.org/image-guide/obtain-images.html
```

# 部署客户端

如果需要用命令行管理，可以安装客户端

```bash
pip3 install python-openstackclient
source /etc/kolla/admin-openrc.sh
openstack service list
```

```text
+----------------------------------+-------------+----------------+
| ID                               | Name        | Type           |
+----------------------------------+-------------+----------------+
| 0c8d90be2ce441cfa69f48fa35aac66b | neutron     | network        |
| 4fc07ae1d4ac45ba80793ba6bba7495b | cinderv3    | volumev3       |
| 50af95a65b3f424f9cf82876e6b6f34b | heat        | orchestration  |
| 57bd175e9b7146398b60f0f7aae3c23a | nova        | compute        |
| 85e1c9b035d44d86a50e54faa46b6d0d | placement   | placement      |
| d59544dc6069432993ef83a1a14b902b | heat-cfn    | cloudformation |
| ed8b173da8964adc9bcc04f0154cb0a9 | nova_legacy | compute_legacy |
| f8cd9e00923b498f8e0bdab1aaa818b7 | keystone    | identity       |
| fff1cf066b7e448babcb28eb6ec3aa66 | glance      | image          |
+----------------------------------+-------------+----------------+
```

# 查询容器列表

```bash
docker ps
```
