# 设置主机名

各个节点完成主机名设置

```bash
hostnamectl set-hostname control1
```

# 准备hosts文件

```bash
cat > /etc/hosts <<EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.50.10 kolla.xiaohui.cn kolla
172.16.50.101 control1.xiaohui.cn control1
172.16.50.102 control2.xiaohui.cn control2
172.16.50.103 control3.xiaohui.cn control3
172.16.50.104 compute1.xiaohui.cn compute1
172.16.50.105 compute2.xiaohui.cn compute2
EOF


```

# 安装先决条件

```bash
dnf install python3-devel libffi-devel gcc openssl-devel python3-libselinux python3-pip git sshpass wget -y
pip3 install ansible -U pip
```

# 安装kolla

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
mkdir -p /etc/kolla
chown $USER:$USER /etc/kolla 

```

# 安装Ansible galaxy依赖

```bash
kolla-ansible install-deps
```

# 配置Ansible配置文件

```bash
mkdir /etc/ansible
cat > /etc/ansible/ansible.cfg <<EOF
[defaults]
host_key_checking=False
pipelining=True
forks=100
EOF
```

# 配置清单文件

一共有两个清单文件，all-in-one和multinode，后者可以配置多个站点，下面是官方样例清单文件

```ini
[control]
10.0.0.[10:12] ansible_user=ubuntu ansible_password=foobar ansible_become=true
# Ansible supports syntax like [10:12] - that means 10, 11 and 12.
# Become clause means "use sudo".

[network:children]
control
# when you specify group_name:children, it will use contents of group specified.

[compute]
10.0.0.[13:14] ansible_user=ubuntu ansible_password=foobar ansible_become=true

[monitoring]
10.0.0.10
# This group is for monitoring node.
# Fill it with one of the controllers' IP address or some others.

[storage:children]
compute

[deployment]
localhost       ansible_connection=local become=true
# use localhost and sudo
```

下面是我修改的清单

```ini
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
compute

[deployment]
localhost       ansible_connection=local become=true
# use localhost and sudo 

...下面的没改
```

# 生成并分发ssh密钥

```bash
ssh-keygen -f /root/.ssh/id_rsa -N ''
for server in  kolla control1 control2 control3 compute1 compute2; do ssh-copy-id root@$server ;done
```

测试一下机器是否在线

```bash
ansible -i multinode all -m ping
```

```bash
wget -O /etc/yum.repos.d/docer-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

```bash
for server in  koll control1 control2 control3 compute1 compute2; do \
sshpass -p 1 ssh -A -g root@$server.xiaohui.cn dnf install chrony docker-ce -y;ssh -A -g root@$server.xiaohui.cn systemctl enable chronyd docker --now;done
```

# 生成koll密码

```bash
kolla-genpwd -p /etc/kolla/passwords.yml
```

# 修改全局配置文件



```bash
sed -i 's/#kolla_base_distro: "centos"/kolla_base_distro: "centos"/'  /etc/kolla/globals.yml
sed -i 's/#kolla_install_type: "source"/kolla_install_type: "source"/'  /etc/kolla/globals.yml
sed -i 's/#network_interface: "eth0"/network_interface: "ens192"/'  /etc/kolla/globals.yml
sed -i 's/#neutron_external_interface: "eth1"/neutron_external_interface: "ens224"/'  /etc/kolla/globals.yml
sed -i 's/^#kolla_internal_vip_address.*/kolla_internal_vip_address: "172.16.50.240"/'  /etc/kolla/globals.yml
sed -i 's/#enable_cinder.*/enable_cinder: "yes"/'  /etc/kolla/globals.yml
sed -i '/#enable_cinder.*/a\enable_cinder_backend_lvm: "yes"'  /etc/kolla/globals.yml


```



```bash
kolla-ansible -i ./multinode bootstrap-servers
kolla-ansible -i ./multinode prechecks
kolla-ansible -i ./multinode deploy
```








































