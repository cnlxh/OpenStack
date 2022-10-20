```textile
作者：李晓辉

微信：Lxh_Chat

邮箱：939958092@qq.com
```

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
172.16.50.253 registry.xiaohui.cn
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
dnf install chrony -y;sshpass -p 1 ssh -A -g -o StrictHostKeyChecking=no root@$server systemctl enable chronyd --now ;done
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

如果你已经在私有容器仓库下载好了镜像，可以加上以下几个参数，不然你跳过私有仓库部分，用在线完成安装，然后按照稍后的上传步骤上传以备下次使用

你必须将私有仓库所使用的https CA证书分发给所有openstack节点，并使这些节点信任证书，不然就需要docker_registry_insecure：yes参数

```bash
sed -i '/^#docker_registry:/a\docker_registry: registry.xiaohui.cn/library'  /etc/kolla/globals.yml
sed -i '/^#docker_registry_username:/a\docker_registry_username: admin'  /etc/kolla/globals.yml
sed -i 's/^docker_registry_password:.*/docker_registry_password: admin/'  /etc/kolla/passwords.yml
```

## 常见参数介绍(可选)

可以用如下方法设置内外API的地址

```bash
kolla_internal_vip_address: "10.10.10.254"
network_interface: "eth0"
kolla_external_vip_address: "10.10.20.254"
kolla_external_vip_interface: "eth1"
```

使用FQDN规范互联网寻址

```bash
kolla_internal_fqdn: inside.mykolla.example.net
kolla_external_fqdn: mykolla.example.net
```

在/etc/kolla/config/nova/myhost/nova.conf中修改CPU和内存分配比率

```ini
[DEFAULT]
cpu_allocation_ratio = 16.0
ram_allocation_ratio = 5.0
```

要在为外部、内部和后端 API 启用 TLS 的情况下部署 OpenStack，请在`globals.yml` 中配置以下内容：

```bash
kolla_enable_tls_internal: "yes"
kolla_enable_tls_external: "yes"
kolla_enable_tls_backend: "yes"
kolla_copy_ca_into_containers: "yes" 
# ubuntu CA证书位置
openstack_cacert: "/etc/ssl/certs/ca-certificates.crt"
# centos CA证书位置
openstack_cacert: "/etc/pki/tls/certs/ca-bundle.crt"
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

# 添加local网络支持

默认的kolla部署不支持local类型的网络，创建local类型时会报错如下：

```textile
network_type value 'local' not supported.
```

解决如下：

需要在所有网络节点都修改，一般网络节点并置在control节点，所以需要在所有control节点修改

```bash
sed -i 's/^type_drivers.*/&,local/' /etc/kolla/neutron-server/ml2_conf.ini
docker restart neutron_server
```

# 创建样例资源

执行后，将会创建样例网络、镜像等资源，请勿用于生产环境

```bash
/usr/local/share/kolla-ansible/init-runonce
```

# 上传镜像到私有仓库

需在每个节点执行，以便于将所有镜像都上传

```bash
tag=yoga
images=`docker images | sed 1d | cut -d ' ' -f1`
remoteurl=registry.xiaohui.cn/library
for localimage in $images;do docker tag $localimage:$tag "$remoteurl/`echo $localimage | cut -d ' ' -f1 | sed 's/quay.io\///'`:$tag";done 

docker login registry.xiaohui.cn -u admin -p admin 
privateimage=`docker images | grep registry.xiaohui.cn | cut -d ' ' -f1`
for image in $privateimage;do docker push $image:$tag;done
```

# 部署客户端

```bash
pip3 install python-openstackclient
source /etc/kolla/admin-openrc.sh
openstack service list

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

# 处理OpenStack重启服务异常

OpenStack 节点全部重启或机房断电之后，会导致mariadb集群异常，这将导致mariadb的容器无限重启，永远无法就绪，后果就是其他需要数据库的容器也用于无法healthy，此处我们可以用kolla的mariadb恢复功能来实现数据库集群状态同步工作

```bash
kolla-ansible mariadb_recovery -i multinode
```

执行之后，再去看容器列表，就会全部处于healthy状态了
