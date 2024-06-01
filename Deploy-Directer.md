```textile
作者：李晓辉

微信：Lxh_Chat

邮箱：xiaohui_li@foxmail.com
```

# 系统平台

| CPU | 内存  | 系统版本            | OpenStack |
| --- | --- | --------------- | --------- |
| 8C  | 32G | CentOS 8 Stream | Yoga      |

# 网络规划

director有两个网络：

1. Provisioning 网络
   
   这是 director 用于置备节点和在执行 Ansible 配置时通过 SSH 访问这些节点的网络。此网络还支持从 undercloud 到 overcloud 节点的 SSH 访问。undercloud 包含在此网络上内省和置备其他节点的 DHCP 服务，这意味着此网络上不应存在其他 DHCP 服务。director 为此网络配置接口。

2. External 网络
   
   支持访问 OpenStack Platform 软件仓库、容器镜像源和 DNS 服务器或 NTP 服务器等的其他服务器。使用此网络从工作站对 undercloud 进行标准访问。必须在 undercloud 上手动配置一个接口以访问外部网络
   
   | Provisioning | External        |
   | ------------ | --------------- |
   | 10.0.0.10/24 | 172.16.50.10/24 |

# 准备 undercloud

创建stack用户，这一步是必须的，安装过程不能使用root用户

```bash
useradd stack
echo lixiaohui | passwd --stdin stack
```

允许stack用户sudo权限

```bash
echo "stack ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/stack
chmod 0440 /etc/sudoers.d/stack
```

设置主机名

```bash
hostnamectl set-hostname director.xiaohui.cn
```

切换到新的 `stack` 用户

```bash
su - stack
```

为系统镜像和 heat 模板创建目录

```bash
mkdir ~/images
mkdir ~/templates
```

配置hosts文件或DNS

```bash
sudo bash -c 'cat >> /etc/hosts <<EOF
127.0.0.1 director.xiaohui.cn director
EOF'
```

# 安装 director 软件包

```bash
sudo bash -c 'cat > /etc/yum.repos.d/tripleo.repo <<EOF
[tripleo]
name=tripleo
baseurl=https://trunk.rdoproject.org/centos8-yoga/component/tripleo/current/
enabled=1
gpgcheck=0
EOF'
```

```bash
sudo yum install python3-tripleo-repos -y
sudo tripleo-repos -b yoga current
sudo tripleo-repos -b yoga current ceph
sudo dnf install -y python3-tripleoclient ceph-ansible
```

```bash
sudo dnf install -y python3-tripleoclient ceph-ansible
```

# 准备容器镜像文件

```bash
sudo openstack tripleo container image prepare default \
  --local-push-destination \
  --output-env-file containers-prepare-parameter.yaml
```

添加用户名密码用于拉取镜像

```bash
  ContainerImageRegistryCredentials:
    quay.io:
      my_username: my_password
```

如果你不需要ceph，可以排除

```yaml
parameter_defaults:
  ContainerImagePrepare:
  - push_destination: true
    excludes:
      - ceph
      - prometheus
    set:
```

# 配置 director

director 安装过程需要 `undercloud.conf` 配置文件中的某些设置，这个文件包含用于配置 undercloud 的设置。如果忽略或注释掉某个参数，undercloud 安装将使用默认值

```bash
cp /usr/share/python-tripleoclient/undercloud.conf.sample ~/undercloud.conf
```

指定我们的容器准备文件

```ini
container_images_file = /home/stack/containers-prepare-parameter.yaml
```

指定Provisioning网络接口

```ini
[DEFAULT]
local_interface = ens224
local_ip = 10.0.0.10/24
local_mtu = 1500
local_subnet = ctlplane-subnet
overcloud_domain_name = xiaohui.cn
undercloud_hostname = director.xiaohui.cn
undercloud_admin_host = 10.0.0.1
undercloud_public_host = 10.0.0.2
```

```ini
custom_env_files = /home/stack/templates/custom-undercloud-params.yaml
```

# 定义ctlplane-subnet

```bash
[ctlplane-subnet]
cidr = 10.0.0.0/24
dhcp_start = 10.0.0.20
dhcp_end = 10.0.0.200
inspection_iprange = 10.0.0.210,10.0.0.250
gateway = 10.0.0.10
```

# 定义admin密码

```bash
cat > /home/stack/templates/custom-undercloud-params.yaml <<EOF
parameter_defaults:
  Debug: True
  AdminPassword: "Steven0608"
  AdminEmail: "admin@example.com"
EOF
```

# 确认文件归属

```bash
sudo chown stack:stack /home/stack/ -R
```

# 安装Undercloud

```bash
openstack undercloud install
```

# 配置Overcloud

从下面的网址下载镜像

```textile
https://buildlogs.centos.org/centos/7/cloud/x86_64/tripleo_images/
```

## 上传镜像

将下载好的镜像放到images目录里，然后把.tar解压

```bash
openstack overcloud image upload
```

## 注册节点

由于没有物理机，现在模拟一下BMC

### 安装BMC

```bash
yum -y install python3-pip python3-devel gcc libvirt-devel ipmitool
pip3 install --upgrade pip
pip3 install virtualbmc
```

```bash
python -m pip install vbmc4vsphere
```

```json
{
  "nodes": [
    {
      "pm_type": "pxe_ipmitool",
      "mac": [
        "00:50:56:94:54:ab"
      ],
      "pm_user": "admin",
      "pm_password": "lixiaohui!",
      "pm_addr": "192.168.31.105",
      "pm_port": "6230",
      "name": "Node01"
    }
  ]
}
```
