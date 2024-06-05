# 纯手工安装OpenStack

| 计算机名       | 管理网络IP                              | 角色                                                        | 操作系统            |
| ---------- | ----------------------------------- | --------------------------------------------------------- | --------------- |
| controller | ens160: 192.168.8.10<br>ens192: 无IP | controller<br>网络服务<br>镜像服务<br>placement服务<br>存储服务<br>网页服务 | centos 9 stream |
| node       | ens160: 192.168.8.20<br>ens192: 无IP | 计算节点                                                      | centos 9 stream |


# 控制节点上的服务

## 配置网络

**管理网络配置:**

1. IP: 192.168.8.10/23

2. 网关: 192.168.8.2

3. DNS: 223.5.5.5

配置方法：

```bash
cat > /etc/NetworkManager/system-connections/ens160.nmconnection <<-'EOF'
[connection]
id=ens160
uuid=b1de734a-07e8-3538-8200-724c6dffd8a7
type=ethernet
autoconnect-priority=-999
interface-name=ens160
timestamp=1715837362

[ethernet]

[ipv4]
address1=192.168.8.10/24,192.168.8.2
dns=223.5.5.5;
method=manual

[ipv6]
addr-gen-mode=eui64
method=auto

[proxy]
EOF
```

**Provider网络要求：**

1. 这个网络要求没有任何有效的IP地址

2. 重启服务器要求会自动激活接口

配置方法：

```bash
cat > /etc/NetworkManager/system-connections/ens192.nmconnection <<-'EOF'
[connection]
id=ens192
uuid=ab11eecc-deb3-33a7-ad7e-aa081cfe8a1c
type=ethernet
autoconnect-priority=-999
interface-name=ens192
timestamp=1715865778

[ethernet]

[ipv4]
method=auto

[ipv6]
addr-gen-mode=eui64
method=auto

[proxy]
EOF
```

两个网卡配置好之后，重启NetworkManager服务

```bash
systemctl restart NetworkManager
```

## 配置名称解析

```textile
cat > /etc/hosts <<EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.8.10 controller
192.168.8.20 node
EOF
```

这里要自己测试是否能ping通各个其他节点，确保通信正常

## 关闭防火墙和SELINUX

```bash
systemctl disable firewalld --now
setenforce 0
sed -i 's/SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
```

## 配置NTP

对于centos来说，默认就在chrony的配置文件中包含了正确的NTP服务器，所以只需要启动服务即可，allow是允许其他服务器来同步时间

```bash
dnf install chrony -y
sed -i '/#allow.*/a\allow 192.168.0.0/16' /etc/chrony.conf
systemctl enable chronyd --now
```

## 验证NTP

```bash
[root@controller ~]# chronyc sources
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^- ntp.wdc2.us.leaseweb.net      2   6    17    44    -13ms[  -13ms] +/-  261ms
^- makaki.miuku.net              3   6    17    44    -33ms[  -33ms] +/-   86ms
^* dns1.synet.edu.cn             1   6    17    44   +352us[  +25ms] +/-   28ms
^- ntp7.flashdance.cx            2   6    17    44    -30ms[  -30ms] +/-  165ms
```

## 启用软件仓库

```bash
dnf install dnf-plugins-core epel-release -y
dnf config-manager --set-enabled crb
dnf install centos-release-openstack-caracal vim bash-completion -y
```

## 安装openstack客户端

```bash
dnf install python3-openstackclient openstack-selinux -y
```

## 安装SQL数据库

```bash
yum install mariadb mariadb-server python3-PyMySQL -y
```

准备配置文件

```ini
cat > /etc/my.cnf.d/openstack.cnf <<'EOF'
[mysqld]
bind-address = 192.168.8.10
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
EOF
```

启动数据库服务

```bash
systemctl enable mariadb.service --now
```

初始化root密码

```bash
mysql_secure_installation << EOF

n
y
LiXiaohui
LiXiaohui
y
n
y
y
EOF
```
输出

```text

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user. If you've just installed MariaDB, and
haven't set the root password yet, you should just press enter here.

stty: 'standard input': Inappropriate ioctl for device
Enter current password for root (enter for none):
stty: 'standard input': Inappropriate ioctl for device
OK, successfully used password, moving on...

Setting the root password or using the unix_socket ensures that nobody
can log into the MariaDB root user without the proper authorisation.

You already have your root account protected, so you can safely answer 'n'.

Switch to unix_socket authentication [Y/n]  ... skipping.

You already have your root account protected, so you can safely answer 'n'.

Change the root password? [Y/n] stty: 'standard input': Inappropriate ioctl for device
New password:
Re-enter new password:
stty: 'standard input': Inappropriate ioctl for device
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n]  ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n]  ... skipping.

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n]  - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n]  ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!

```

## 配置消息队列

```bash
yum install rabbitmq-server -y
```

启动服务

```bash
systemctl enable rabbitmq-server.service --now
```

添加openstack用户

```bash
rabbitmqctl add_user openstack LiXiaohui
```

分配此用户为管理员

```bash
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

## 配置memcached

```bash
yum install memcached python3-memcached -y
```

将服务的IP更改为使用管理接口的IP

```bash
sed -i 's/OPTIONS=.*/OPTIONS="-l 127.0.0.1,::1,controller"/' /etc/sysconfig/memcached
```

启动服务

```bash
systemctl enable memcached.service --now
```

## 配置ETCD

```bash
yum install etcd -y
```

把这/etc/etcd/etcd.conf的几个IP都换成管理接口的IP： `ETCD_INITIAL_CLUSTER`, `ETCD_INITIAL_ADVERTISE_PEER_URLS`, `ETCD_ADVERTISE_CLIENT_URLS`, `ETCD_LISTEN_CLIENT_URLS`  

```bash
cat > /etc/etcd/etcd.conf <<'EOF' 
#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.8.10:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.8.10:2379"
ETCD_NAME="controller"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.8.10:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.8.10:2379"
ETCD_INITIAL_CLUSTER="controller=http://192.168.8.10:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

启动服务

```bash
systemctl enable etcd --now
```

## 安装配置keystone服务

### 准备keystone数据库

```bash
mysql -uroot -pLiXiaohui
```

创建数据库

```bash
CREATE DATABASE keystone;
```

分配权限

```bash
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'LiXiaohui';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'LiXiaohui';
exit
```

### 安装配置 keystone组件

```bash
yum install openstack-keystone httpd python3-mod_wsgi -y
```

### 安装CRUDINI编辑器（可选）

这一步是为了防止手工vim修改配置文件改错，如果你熟练掌握vim搜索和输入，这一步并不是必须的

```bash
yum install crudini -y
```

### 准备keystone配置文件

在`/etc/keystone/keystone.conf`中`[database]和[token]`修改正确的值，注意密码要正确，controller名称需要能解析

```bash
crudini --set /etc/keystone/keystone.conf database connection mysql+pymysql://keystone:LiXiaohui@controller/keystone
crudini --set /etc/keystone/keystone.conf token provider fernet
```

或者手工编辑以下选项

```ini
[database]
...
connection = mysql+pymysql://keystone:LiXiaohui@controller/keystone
[token]
...
provider = fernet
```

### 准备keystone数据库

```bash
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

### 初始化Fernet密钥库

```bash
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

### 构建Keystone服务

```bash
keystone-manage bootstrap --bootstrap-password LiXiaohui \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

### 配置HTTPD服务

```bash
sed -i '/^ServerRoot/a\ServerName controller:80' /etc/httpd/conf/httpd.conf
```

链接wsgi-keystone.conf

```bash
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```

### 启动httpd服务

```bash
systemctl enable httpd.service --now
```

### 导出环境变量

```bash
export OS_USERNAME=admin
export OS_PASSWORD=LiXiaohui
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```

### 创建domain、project、user、roles

这些步骤是用于测试keystone是否工作正常的，不报错就行了

```bash
[root@controller ~]# openstack domain create --description "An Example Domain" example
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | bd66615ead9545bd818e38de04330ddd |
| name        | example                          |
| options     | {}                               |
| tags        | []                               |
+-------------+----------------------------------+
```

```bash
[root@controller ~]# openstack project create --domain default --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 45b31ccce6854ea4925b731e257c53de |
| is_domain   | False                            |
| name        | service                          |
| options     | {}                               |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```

```bash
[root@controller ~]# openstack project create --domain default --description "Demo Project" myproject
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 18c97bc23c844cf8b6aa950927779d59 |
| is_domain   | False                            |
| name        | myproject                        |
| options     | {}                               |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```

```bash
[root@controller ~]# openstack user create --domain default --password LiXiaohui myuser
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | bf296091131d4465891fe70de4c6c528 |
| name                | myuser                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

```bash
[root@controller ~]# openstack role create myrole
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | None                             |
| domain_id   | None                             |
| id          | cfc708779c4f4119a77488e0f6001159 |
| name        | myrole                           |
| options     | {}                               |
+-------------+----------------------------------+
[root@controller ~]# openstack role add --project myproject --user myuser myrole
```

现在发现都不报错，可以创建自己的环境变量文件了

```bash
cat > adminrc.sh <<'EOF'
export OS_USERNAME=admin
export OS_PASSWORD=LiXiaohui
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
EOF
```

## 安装配置glance服务

### 准备glance数据库

```bash
mysql -u root -pLiXiaohui
```

```bash
CREATE DATABASE glance; 
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'LiXiaohui';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'LiXiaohui';
exit
```

加载刚创建的rc文件

```bash
source adminrc.sh
```

### 创建镜像服务的用户和角色

```bash
[root@controller ~]# openstack user create --domain default --password LiXiaohui glance
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 34b86e5097f24acfaa9fd185f601a482 |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
[root@controller ~]# openstack role add --project service --user glance admin
```

### 创建镜像服务的endpoint

```bash
[root@controller ~]# openstack service create --name glance --description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | b17d10f67e354756aa14adc895c45039 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne image public http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 84f29dc346be4d0cb1dd014017ecee4e |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b17d10f67e354756aa14adc895c45039 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+ 

[root@controller ~]# openstack endpoint create --region RegionOne image internal http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | f2f8e0708ade4b6d86fcb566b40e9878 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b17d10f67e354756aa14adc895c45039 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+ 

[root@controller ~]# openstack endpoint create --region RegionOne image admin http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 3cfbe27dbecb4f3fbe686f17d8d5ebf2 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | b17d10f67e354756aa14adc895c45039 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```

### 安装和配置glance组件

```bash
yum install openstack-glance -y
```

### 准备配置文件

`/etc/glance/glance-api.conf` 是glance的配置文件

```bash
crudini --set /etc/glance/glance-api.conf DEFAULT enabled_backends fs:file
crudini --set /etc/glance/glance-api.conf database connection mysql+pymysql://glance:LiXiaohui@controller/glance
crudini --set /etc/glance/glance-api.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/glance/glance-api.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_type password
crudini --set /etc/glance/glance-api.conf keystone_authtoken project_domain_name Default
crudini --set /etc/glance/glance-api.conf keystone_authtoken user_domain_name Default
crudini --set /etc/glance/glance-api.conf keystone_authtoken project_name service
crudini --set /etc/glance/glance-api.conf keystone_authtoken username glance
crudini --set /etc/glance/glance-api.conf keystone_authtoken password LiXiaohui
crudini --set /etc/glance/glance-api.conf glance_store default_backend fs
crudini --set /etc/glance/glance-api.conf fs filesystem_store_datadir /var/lib/glance/images/
crudini --set /etc/glance/glance-api.conf oslo_limit auth_url http://controller:5000
crudini --set /etc/glance/glance-api.conf oslo_limit auth_type password
crudini --set /etc/glance/glance-api.conf oslo_limit user_domain_id default
crudini --set /etc/glance/glance-api.conf oslo_limit username glance
crudini --set /etc/glance/glance-api.conf oslo_limit system_scope all
crudini --set /etc/glance/glance-api.conf oslo_limit password LiXiaohui
crudini --set /etc/glance/glance-api.conf oslo_limit endpoint_id 84f29dc346be4d0cb1dd014017ecee4e #这是public的id
crudini --set /etc/glance/glance-api.conf oslo_limit region_name RegionOne
crudini --set /etc/glance/glance-api.conf paste_deploy flavor keystone
```
或者

```ini
[DEFAULT]
...
enabled_backends = fs:file
# use_keystone_limits = True # 如果想启用配额，启用这行
```

```ini
[database]
...
connection = mysql+pymysql://glance:LiXiaohui@controller/glance
```

```ini
[keystone_authtoken]
www_authenticate_uri  = http://controller:5000
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = LiXiaohui
```

```ini
[glance_store]
...
default_backend = fs

[fs]
filesystem_store_datadir = /var/lib/glance/images/
```

```ini
[oslo_limit]
auth_url = http://controller:5000
auth_type = password
user_domain_id = default
username = glance
system_scope = all
password = LiXiaohui
endpoint_id = 84f29dc346be4d0cb1dd014017ecee4e #这是public的id
region_name = RegionOne
```

```ini
[paste_deploy]
flavor = keystone
```

确保glance用户可以对系统范围内有读权限

```bash
[root@controller ~]# openstack role add --user glance --user-domain Default --system all reader
```

### 填充glance服务数据库

```bash
su -s /bin/sh -c "glance-manage db_sync" glance
```

### 启动glance服务

```bash
systemctl enable openstack-glance-api.service --now
```

## 安装配置plancement服务

### 准备placement数据

```bash
mysql -u root -pLiXiaohui
```

```bash
CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'LiXiaohui';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'LiXiaohui';
exit
```

### 创建用户和endpoint

```bash
[root@controller ~]# openstack user create --domain default --password LiXiaohui placement
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | c6f92a93afcb44bf9627a46131e010a4 |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

```bash
[root@controller ~]# openstack role add --project service --user placement admin
```

```bash
[root@controller ~]# openstack service create --name placement --description "Placement API" placement
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Placement API                    |
| enabled     | True                             |
| id          | 5e175cbd51a04a8aa2a9821947769347 |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne placement public http://controller:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | e939b955c89840daa86b7971da69571b |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 5e175cbd51a04a8aa2a9821947769347 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne placement internal http://controller:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 14e61050ac91462aa067d29b02b60532 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 5e175cbd51a04a8aa2a9821947769347 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne placement admin http://controller:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 58f6dcc9f88749658dcc339ea800c73f |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 5e175cbd51a04a8aa2a9821947769347 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
```

### 安装配置placement组件

```bash
yum install openstack-placement-api -y
```

### 准备配置文件

`/etc/placement/placement.conf` 是配置文件

```bash
crudini --set /etc/placement/placement.conf placement_database connection mysql+pymysql://placement:LiXiaohui@controller/placement
crudini --set /etc/placement/placement.conf api auth_strategy keystone
crudini --set /etc/placement/placement.conf keystone_authtoken auth_url http://controller:5000/v3
crudini --set /etc/placement/placement.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/placement/placement.conf keystone_authtoken auth_type password
crudini --set /etc/placement/placement.conf keystone_authtoken project_domain_name Default
crudini --set /etc/placement/placement.conf keystone_authtoken user_domain_name Default
crudini --set /etc/placement/placement.conf keystone_authtoken project_name service
crudini --set /etc/placement/placement.conf keystone_authtoken username placement
crudini --set /etc/placement/placement.conf keystone_authtoken password LiXiaohui
```

或者

```ini
[placement_database]
...
connection = mysql+pymysql://placement:LiXiaohui@controller/placement

[api]
...
auth_strategy = keystone

[keystone_authtoken]
...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = LiXiaohui
```

### 填充placement数据库

```bash
su -s /bin/sh -c "placement-manage db sync" placement
```

默认情况下，placement配置文件有bug，需要对placement api命令位置授权

```bash
cat >> /etc/httpd/conf.d/00-placement-api.conf <<'EOF'
<Directory /usr/bin>
  Require all granted
</Directory>
EOF
```

### 重新启动httpd服务

```bash
systemctl restart httpd
```

## 安装配置nova服务

**需要注意，此章节是在控制节点完成的**

**这个部分将描述如何在控制节点上安装和配置 Compute 的API服务**

### 准备nova数据库

```bash
mysql -u root -pLiXiaohui
```

```bash
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
```

```bash
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'LiXiaohui';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'LiXiaohui';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'LiXiaohui';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'LiXiaohui';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'LiXiaohui'; 
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'LiXiaohui';
exit
```

### 创建计算服务凭据

```bash
[root@controller ~]# openstack user create --domain default --password LiXiaohui nova
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 0f41bea70add4b9d97c3c91eaf7d6e7b |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

```bash
[root@controller ~]# openstack role add --project service --user nova admin
```

```bash
[root@controller ~]# openstack service create --name nova --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 7e6cb73bc6a1465282cce7d9af3d37a1 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```

### 创建nova endpoint

```bash
[root@controller ~]# openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 44b7bb17a1a24455971de09c3a1ee2e9 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 7e6cb73bc6a1465282cce7d9af3d37a1 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 2fd4074deb8d49738d0060ede3164c9d |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 7e6cb73bc6a1465282cce7d9af3d37a1 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 30f44d023bcf44e09874f4faafcb1f1b |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 7e6cb73bc6a1465282cce7d9af3d37a1 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```

### 安装配置nova组件

```bash
yum install openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler -y
```

### 准备配置文件

 `/etc/nova/nova.conf` 是nova配置文件

```bash
crudini --set /etc/nova/nova.conf DEFAULT enabled_apis osapi_compute,metadata
crudini --set /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:LiXiaohui@controller:5672/
crudini --set /etc/nova/nova.conf DEFAULT my_ip 192.168.8.10
crudini --set /etc/nova/nova.conf api_database connection mysql+pymysql://nova:LiXiaohui@controller/nova_api
crudini --set /etc/nova/nova.conf database connection mysql+pymysql://nova:LiXiaohui@controller/nova
crudini --set /etc/nova/nova.conf api auth_strategy keystone
crudini --set /etc/nova/nova.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/nova/nova.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/nova/nova.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/nova/nova.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/nova/nova.conf keystone_authtoken auth_type password
crudini --set /etc/nova/nova.conf keystone_authtoken project_domain_name Default
crudini --set /etc/nova/nova.conf keystone_authtoken user_domain_name Default
crudini --set /etc/nova/nova.conf keystone_authtoken project_name service
crudini --set /etc/nova/nova.conf keystone_authtoken username nova
crudini --set /etc/nova/nova.conf keystone_authtoken password LiXiaohui
crudini --set /etc/nova/nova.conf service_user send_service_user_token true
crudini --set /etc/nova/nova.conf service_user auth_url http://controller:5000/identity
crudini --set /etc/nova/nova.conf service_user auth_strategy keystone
crudini --set /etc/nova/nova.conf service_user auth_type password
crudini --set /etc/nova/nova.conf service_user project_domain_name Default
crudini --set /etc/nova/nova.conf service_user user_domain_name Default
crudini --set /etc/nova/nova.conf service_user project_name service
crudini --set /etc/nova/nova.conf service_user username nova
crudini --set /etc/nova/nova.conf service_user password LiXiaohui
crudini --set /etc/nova/nova.conf vnc enabled true
crudini --set /etc/nova/nova.conf vnc server_listen ' $my_ip'
crudini --set /etc/nova/nova.conf vnc server_proxyclient_address ' $my_ip'
crudini --set /etc/nova/nova.conf glance api_servers http://controller:9292
crudini --set /etc/nova/nova.conf oslo_concurrency lock_path /var/lib/nova/tmp
crudini --set /etc/nova/nova.conf placement region_name RegionOne
crudini --set /etc/nova/nova.conf placement project_domain_name Default
crudini --set /etc/nova/nova.conf placement project_name service
crudini --set /etc/nova/nova.conf placement auth_type password
crudini --set /etc/nova/nova.conf placement user_domain_name Default
crudini --set /etc/nova/nova.conf placement auth_url http://controller:5000/v3
crudini --set /etc/nova/nova.conf placement username placement
crudini --set /etc/nova/nova.conf placement password LiXiaohui
```

或者

```ini
[DEFAULT]
...
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:LiXiaohui@controller:5672/
my_ip = 192.168.8.10

[api_database]
...
connection = mysql+pymysql://nova:LiXiaohui@controller/nova_api

[database]
...
connection = mysql+pymysql://nova:LiXiaohui@controller/nova

[api]
...
auth_strategy = keystone

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = LiXiaohui


[service_user]
send_service_user_token = true
auth_url = http://controller:5000/identity
auth_strategy = keystone
auth_type = password
project_domain_name = Default
project_name = service
user_domain_name = Default
username = nova
password =  LiXiaohui


[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip 

[glance]
...
api_servers = http://controller:9292
[oslo_concurrency]
...
lock_path = /var/lib/nova/tmp

[placement]
...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = LiXiaohui
```

### 填充nova-api数据库

```bash
su -s /bin/sh -c "nova-manage api_db sync" nova
```

### 注册cell0数据库

```bash
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

### 创建cell1

```bash
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
```

### 填充nova数据库

```bash
su -s /bin/sh -c "nova-manage db sync" nova
```

### 验证cell0和cell1注册正常

```bash
[root@controller ~]# su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
Modules with known eventlet monkey patching issues were imported prior to eventlet monkey patching: urllib3. This warning can usually be ignored if the caller is only importing and not executing nova code.
+-------+--------------------------------------+------------------------------------------+-------------------------------------------------+----------+
|  Name |                 UUID                 |              Transport URL               |               Database Connection               | Disabled |
+-------+--------------------------------------+------------------------------------------+-------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                  none:/                  | mysql+pymysql://nova:****@controller/nova_cell0 |  False   |
| cell1 | a906846b-650e-42c1-8d9e-dcf5cbec6a6e | rabbit://openstack:****@controller:5672/ |    mysql+pymysql://nova:****@controller/nova    |  False   |
+-------+--------------------------------------+------------------------------------------+-------------------------------------------------+----------+
```

### 启动nova服务

```bash
systemctl enable --now \
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service
```

## 安装配置neutron网络服务

### 控制节点需要做的事

#### 准备neutron数据库

```bash
mysql -uroot -pLiXiaohui
```

```bash
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'LiXiaohui';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'LiXiaohui';
exit
```

#### 创建服务凭据

```bash
[root@controller ~]# openstack user create --domain default --password LiXiaohui neutron
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | f63963ed733f48e9acdeaaf0d4b00b07 |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

```bash
[root@controller ~]# openstack role add --project service --user neutron admin
```

#### 创建网络服务的endpoint

```bash
[root@controller ~]# openstack service create --name neutron --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | 0501b46a9152497d96dee63653ac6b46 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne network public http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | dbe8552ab73a4d9fa247f473b3dc1996 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 0501b46a9152497d96dee63653ac6b46 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne network internal http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | a982e24225034b50bd325c158813f901 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 0501b46a9152497d96dee63653ac6b46 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne network admin http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 9c6373c019904792b72e97466d704c6c |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 0501b46a9152497d96dee63653ac6b46 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```

启用模块支持

```bash
cat <<EOF | sudo tee /etc/modules-load.d/openstack.conf
br_netfilter
EOF
modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

接下来提供了LinuxBridge、OpenvSwitch和Geneve三种方式，随便哪个都可以，不要在本文档中采用Geneve，没有写完

#### 安装配置Geneve网络机制

```bash
yum install ovn-central openvswitch ovn-host openstack-neutron openstack-neutron-ml2 -y
```

##### 准备配置文件

`/etc/neutron/neutron.conf` 是网络服务的主配置文件

```bash
crudini --set /etc/neutron/neutron.conf DEFAULT core_plugin ml2
crudini --set /etc/neutron/neutron.conf DEFAULT service_plugins ovn-router
crudini --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:LiXiaohui@controller
crudini --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
crudini --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_status_changes true
crudini --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_data_changes true
crudini --set /etc/neutron/neutron.conf DEFAULT allow_overlapping_ips true
crudini --set /etc/neutron/neutron.conf DEFAULT rpc_backend rabbit
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_host controller
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_userid openstack
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_password LiXiaohui
crudini --set /etc/neutron/neutron.conf database connection mysql+pymysql://neutron:LiXiaohui@controller/neutron
crudini --set /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_type password
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_domain_name Default
crudini --set /etc/neutron/neutron.conf keystone_authtoken user_domain_name Default
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_name service
crudini --set /etc/neutron/neutron.conf keystone_authtoken username neutron
crudini --set /etc/neutron/neutron.conf keystone_authtoken password LiXiaohui
crudini --set /etc/neutron/neutron.conf nova region_name RegionOne
crudini --set /etc/neutron/neutron.conf nova project_domain_name Default
crudini --set /etc/neutron/neutron.conf nova project_name service
crudini --set /etc/neutron/neutron.conf nova auth_type password
crudini --set /etc/neutron/neutron.conf nova user_domain_name Default
crudini --set /etc/neutron/neutron.conf nova auth_url http://controller:5000
crudini --set /etc/neutron/neutron.conf nova username nova
crudini --set /etc/neutron/neutron.conf nova password LiXiaohui
crudini --set /etc/neutron/neutron.conf experimental linuxbridge true
crudini --set /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp

```

或者

```ini
[DEFAULT]
...
core_plugin = ml2
service_plugins = ovn-router
transport_url = rabbit://openstack:LiXiaohui@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
allow_overlapping_ips = True
rpc_backend = rabbit

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = LiXiaohui

[database]
...
connection = mysql+pymysql://neutron:LiXiaohui@controller/neutron

[keystone_authtoken]
...
www_authenticate_uri = http://controller:5000
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = LiXiaohui

[nova]
...
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = nova
password = LiXiaohui

[experimental]
linuxbridge = true

[oslo_concurrency]
...
lock_path = /var/lib/neutron/tmp
```
##### 配置ML2插件

ML2插件使用Geneve机制来为实例创建layer－2虚拟网络基础设施

`/etc/neutron/plugins/ml2/ml2_conf.ini` 是ml2插件的配置文件

```bash
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers local,flat,vlan,vxlan,geneve
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types geneve
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers ovn
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 extension_drivers port_security
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 overlay_ip_version 4
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_flat flat_networks provider
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_security_group true
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_vxlan vni_ranges 1001:1100
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_geneve vni_ranges 1:65536
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_geneve max_header_size 50
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ovn ovn_nb_connection tcp:192.168.8.10:6641
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ovn ovn_sb_connection tcp:192.168.8.10:6642
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ovn ovn_l3_scheduler leastloaded
```

或者

```ini
[ml2]
mechanism_drivers = ovn
type_drivers = local,flat,vlan,vxlan,geneve
tenant_network_types = geneve
extension_drivers = port_security
overlay_ip_version = 4

[ml2_type_vxlan]
vni_ranges = 1001:1100

[ml2_type_geneve]
vni_ranges = 1:65536
max_header_size = 50

[securitygroup]
...
enable_security_group = true

[ovn]
...
ovn_nb_connection = tcp:192.168.8.10:6641
ovn_sb_connection = tcp:192.168.8.10:6642
ovn_l3_scheduler = leastloaded

```

启动openvswitch服务

```bash
systemctl enable openvswitch --now
```
默认情况下， ovsdb-server服务仅允许通过 Unix 套接字本地访问数据库。然而计算节点上的 OVN 服务需要访问这些数据库，所以设置一下监听的IP地址

```bash
ovn-nbctl set-connection ptcp:6641:192.168.8.10 -- set connection . inactivity_probe=60000
ovn-sbctl set-connection ptcp:6642:192.168.8.10 -- set connection . inactivity_probe=60000
ovs-appctl -t ovsdb-server ovsdb-server/add-remote ptcp:6640:192.168.8.10
```

启动ovn-northd服务

```bash
systemctl enable ovn-northd --now
```

设置 Open vSwitch 的外部标识，启用机架作为网关的选项

```bash
ovs-vsctl set open . external-ids:ovn-cms-options=enable-chassis-as-gw
```

##### 初始化数据库

网络服务初始化脚本需要一个超链接

```bash
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

```bash
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

##### 重启相关服务

```bash
systemctl restart openstack-nova-api.service
systemctl restart neutron-server
systemctl enable neutron-server --now
```

#### 安装配置LinuxBridge网络机制

此处选择了LinuxBridge网络以及option 2，也就是支持租户自建网络、路由器的模式，而不是直连外网的模式

```bash
yum install -y openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables
```

##### 准备配置文件

`/etc/neutron/neutron.conf` 是网络服务的主配置文件

```bash
crudini --set /etc/neutron/neutron.conf DEFAULT core_plugin ml2
crudini --set /etc/neutron/neutron.conf DEFAULT service_plugins router
crudini --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:LiXiaohui@controller
crudini --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
crudini --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_status_changes true
crudini --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_data_changes true
crudini --set /etc/neutron/neutron.conf DEFAULT allow_overlapping_ips true
crudini --set /etc/neutron/neutron.conf DEFAULT rpc_backend rabbit
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_host controller
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_userid openstack
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_password LiXiaohui
crudini --set /etc/neutron/neutron.conf database connection mysql+pymysql://neutron:LiXiaohui@controller/neutron
crudini --set /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_type password
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_domain_name Default
crudini --set /etc/neutron/neutron.conf keystone_authtoken user_domain_name Default
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_name service
crudini --set /etc/neutron/neutron.conf keystone_authtoken username neutron
crudini --set /etc/neutron/neutron.conf keystone_authtoken password LiXiaohui
crudini --set /etc/neutron/neutron.conf nova region_name RegionOne
crudini --set /etc/neutron/neutron.conf nova project_domain_name Default
crudini --set /etc/neutron/neutron.conf nova project_name service
crudini --set /etc/neutron/neutron.conf nova auth_type password
crudini --set /etc/neutron/neutron.conf nova user_domain_name Default
crudini --set /etc/neutron/neutron.conf nova auth_url http://controller:5000
crudini --set /etc/neutron/neutron.conf nova username nova
crudini --set /etc/neutron/neutron.conf nova password LiXiaohui
crudini --set /etc/neutron/neutron.conf experimental linuxbridge true
crudini --set /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp

```

或者

```ini
[DEFAULT]
...
core_plugin = ml2
service_plugins = router
transport_url = rabbit://openstack:LiXiaohui@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
allow_overlapping_ips = True
rpc_backend = rabbit

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = LiXiaohui

[database]
...
connection = mysql+pymysql://neutron:LiXiaohui@controller/neutron

[keystone_authtoken]
...
www_authenticate_uri = http://controller:5000
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = LiXiaohui

[nova]
...
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = nova
password = LiXiaohui

[experimental]
linuxbridge = true

[oslo_concurrency]
...
lock_path = /var/lib/neutron/tmp
```

##### 配置ML2插件

ML2插件使用Linuxbridge机制来为实例创建layer－2虚拟网络基础设施

`/etc/neutron/plugins/ml2/ml2_conf.ini` 是ml2插件的配置文件

```bash
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers flat,vlan,vxlan
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types vxlan
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers linuxbridge,l2population
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 extension_drivers port_security
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_flat flat_networks provider 
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_vxlan vni_ranges 1:1000
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_ipset True
```

或者

```ini
[ml2]
...
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security

[ml2_type_flat]
...
flat_networks = provider

[ml2_type_vxlan]
...
vni_ranges = 1:1000

[securitygroup]
enable_ipset = True
```

##### 配置Linuxbridge代理

Linuxbridge代理为实例建立layer－2虚拟网络并且处理安全组规则

`/etc/neutron/plugins/ml2/linuxbridge_agent.ini` 是ml2插件的配置文件

```bash
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini linux_bridge physical_interface_mappings provider:ens160
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan enable_vxlan True
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan local_ip 192.168.8.10
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan l2_population True
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup enable_security_group True
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup firewall_driver neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

或者

```ini
[linux_bridge]
physical_interface_mappings = provider:ens160
...

[vxlan]
enable_vxlan = True
local_ip = 192.168.8.10
l2_population = True

[securitygroup]
...
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

##### 配置Layer-3代理

Layer-3代理为私有虚拟网络提供路由和NAT服务

`/etc/neutron/l3_agent.ini` 是L3的配置文件

```bash
crudini --set /etc/neutron/l3_agent.ini DEFAULT interface_driver neutron.agent.linux.interface.BridgeInterfaceDriver
crudini --set /etc/neutron/l3_agent.ini DEFAULT external_network_bridge
```

或者

```ini
[DEFAULT]
...
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
external_network_bridge =
```

##### 配置DHCP代理

DHCP代理给实例提供了DHCP服务

`/etc/neutron/dhcp_agent.ini` 的DHCP配置文件

```bash
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT interface_driver neutron.agent.linux.interface.BridgeInterfaceDriver
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT dhcp_driver neutron.agent.linux.dhcp.Dnsmasq
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT enable_isolated_metadata True
```

或者

```ini
[DEFAULT]
...
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
```

##### 配置元数据代理

负责提供配置信息，例如：访问实例的凭证

`/etc/neutron/metadata_agent.ini`是主配置文件

```bash
crudini --set /etc/neutron/metadata_agent.ini DEFAULT nova_metadata_host controller
crudini --set /etc/neutron/metadata_agent.ini DEFAULT metadata_proxy_shared_secret LiXiaohui
```

或者

```ini
[DEFAULT]
...
nova_metadata_host = controller
metadata_proxy_shared_secret = LiXiaohui
```

##### 在nova服务中使用neutron

`/etc/nova/nova.conf`是nova配置文件

```bash
crudini --set /etc/nova/nova.conf neutron url http://controller:9696
crudini --set /etc/nova/nova.conf neutron auth_url http://controller:5000
crudini --set /etc/nova/nova.conf neutron auth_type password
crudini --set /etc/nova/nova.conf neutron project_domain_name default
crudini --set /etc/nova/nova.conf neutron user_domain_name default
crudini --set /etc/nova/nova.conf neutron region_name RegionOne
crudini --set /etc/nova/nova.conf neutron project_name service
crudini --set /etc/nova/nova.conf neutron username neutron
crudini --set /etc/nova/nova.conf neutron password LiXiaohui
crudini --set /etc/nova/nova.conf neutron service_metadata_proxy True
crudini --set /etc/nova/nova.conf neutron metadata_proxy_shared_secret LiXiaohui
```

或者

```ini
[neutron]
...
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = LiXiaohui
service_metadata_proxy = True
metadata_proxy_shared_secret = LiXiaohui
```

##### 初始化数据库

网络服务初始化脚本需要一个超链接

```bash
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

```bash
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

##### 重启相关服务

重启计算API 服务

```bash
systemctl restart openstack-nova-api.service
```

网络服务

```bash
systemctl enable neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service --now
```

对于网络选项2，同样启用layer－3服务并设置其随系统自启动

```bash
systemctl enable neutron-l3-agent.service --now
```


#### 安装配置OpenvSwitch网络机制

此处选择了OpenvSwitch网络以及option 2，也就是支持租户自建网络、路由器的模式，而不是直连外网的模式

```bash
yum install -y openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-openvswitch ebtables
```

##### 准备配置文件

`/etc/neutron/neutron.conf` 是网络服务的主配置文件

```bash
crudini --set /etc/neutron/neutron.conf DEFAULT core_plugin ml2
crudini --set /etc/neutron/neutron.conf DEFAULT service_plugins router
crudini --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:LiXiaohui@controller
crudini --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
crudini --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_status_changes true
crudini --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_data_changes true
crudini --set /etc/neutron/neutron.conf DEFAULT allow_overlapping_ips true
crudini --set /etc/neutron/neutron.conf DEFAULT rpc_backend rabbit
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_host controller
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_userid openstack
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_password LiXiaohui
crudini --set /etc/neutron/neutron.conf database connection mysql+pymysql://neutron:LiXiaohui@controller/neutron
crudini --set /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_type password
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_domain_name Default
crudini --set /etc/neutron/neutron.conf keystone_authtoken user_domain_name Default
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_name service
crudini --set /etc/neutron/neutron.conf keystone_authtoken username neutron
crudini --set /etc/neutron/neutron.conf keystone_authtoken password LiXiaohui
crudini --set /etc/neutron/neutron.conf nova region_name RegionOne
crudini --set /etc/neutron/neutron.conf nova project_domain_name Default
crudini --set /etc/neutron/neutron.conf nova project_name service
crudini --set /etc/neutron/neutron.conf nova auth_type password
crudini --set /etc/neutron/neutron.conf nova user_domain_name Default
crudini --set /etc/neutron/neutron.conf nova auth_url http://controller:5000
crudini --set /etc/neutron/neutron.conf nova username nova
crudini --set /etc/neutron/neutron.conf nova password LiXiaohui
crudini --set /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp
```

或者

```ini
[DEFAULT]
...
core_plugin = ml2
service_plugins = router
transport_url = rabbit://openstack:LiXiaohui@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
allow_overlapping_ips = True
rpc_backend = rabbit

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = LiXiaohui

[database]
...
connection = mysql+pymysql://neutron:LiXiaohui@controller/neutron

[keystone_authtoken]
...
www_authenticate_uri = http://controller:5000
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = LiXiaohui

[nova]
...
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = nova
password = LiXiaohui

[oslo_concurrency]
...
lock_path = /var/lib/neutron/tmp
```

##### 配置ML2插件

ML2插件使用Linuxbridge机制来为实例创建layer－2虚拟网络基础设施

`/etc/neutron/plugins/ml2/ml2_conf.ini` 是ml2插件的配置文件

```bash
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers flat,vlan,vxlan
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types vxlan
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers openvswitch,l2population
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 extension_drivers port_security
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_flat flat_networks provider 
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_vxlan vni_ranges 1:1000
crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_ipset True
```

或者

```ini
[ml2]
...
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security

[ml2_type_flat]
...
flat_networks = provider

[ml2_type_vxlan]
...
vni_ranges = 1:1000

[securitygroup]
enable_ipset = True
```
##### 准备OVS网桥

后续的配置中，会用到br-int和br-ex网桥，提前创建出来

```bash
systemctl enable neutron-openvswitch-agent.service --now
```

检查br-int网桥是否创建出来，如果没有，那就自己创建br-int网桥用于虚拟机内部通信，一般来说，启动服务会自动创建

可以看到我的br-int已经自动创建

```bash
[root@controller ~]# ovs-vsctl show
1b37f8e1-ef53-46f1-8d0e-ce60268a0346
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port br-int
            Interface br-int
                type: internal
    ovs_version: "3.3.1"
```

如果没有创建，就执行以下命令即可

```bash
[root@controller ~]# ovs-vsctl add-br br-int
```

确认了br-int之后，需要创建br-ex用于外部网络通信，并添加具有外部通信的网卡进来

```bash
[root@controller ~]# ovs-vsctl add-br br-ex
[root@controller ~]# ovs-vsctl add-port br-ex ens192
```

##### 配置Open vSwitch代理

Open vSwitch代理为实例建立layer－2虚拟网络并且处理安全组规则

`/etc/neutron/plugins/ml2/openvswitch_agent.ini` 是ml2插件的配置文件

```bash
crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs bridge_mappings provider:br-ex
crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs local_ip 192.168.8.10
crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini agent tunnel_types vxlan
crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini agent l2_population true
crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini securitygroup enable_security_group True
crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini securitygroup firewall_driver openvswitch
```

或者

```ini
[ovs]
bridge_mappings = provider:ens160
local_ip = 192.168.8.10
...

[agent]
tunnel_types = vxlan
l2_population = true

[securitygroup]
...
enable_security_group = True
firewall_driver = openvswitch
```

##### 配置Layer-3代理

Layer-3代理为私有虚拟网络提供路由和NAT服务

`/etc/neutron/l3_agent.ini` 是L3的配置文件

```bash
crudini --set /etc/neutron/l3_agent.ini DEFAULT interface_driver openvswitch
```

或者

```ini
[DEFAULT]
...
interface_driver = openvswitch
```

##### 配置DHCP代理

DHCP代理给实例提供了DHCP服务

`/etc/neutron/dhcp_agent.ini` 的DHCP配置文件

```bash
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT interface_driver openvswitch
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT dhcp_driver neutron.agent.linux.dhcp.Dnsmasq
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT enable_isolated_metadata True
```

或者

```ini
[DEFAULT]
...
interface_driver = openvswitch
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
```

##### 配置元数据代理

负责提供配置信息，例如：访问实例的凭证

`/etc/neutron/metadata_agent.ini`是主配置文件

```bash
crudini --set /etc/neutron/metadata_agent.ini DEFAULT nova_metadata_host controller
crudini --set /etc/neutron/metadata_agent.ini DEFAULT metadata_proxy_shared_secret LiXiaohui
```

或者

```ini
[DEFAULT]
...
nova_metadata_host = controller
metadata_proxy_shared_secret = LiXiaohui
```

##### 在控制节点上为nova配置网络服务

`/etc/nova/nova.conf`是nova配置文件

```bash
crudini --set /etc/nova/nova.conf neutron url http://controller:9696
crudini --set /etc/nova/nova.conf neutron auth_url http://controller:5000
crudini --set /etc/nova/nova.conf neutron auth_type password
crudini --set /etc/nova/nova.conf neutron project_domain_name default
crudini --set /etc/nova/nova.conf neutron user_domain_name default
crudini --set /etc/nova/nova.conf neutron region_name RegionOne
crudini --set /etc/nova/nova.conf neutron project_name service
crudini --set /etc/nova/nova.conf neutron username neutron
crudini --set /etc/nova/nova.conf neutron password LiXiaohui
crudini --set /etc/nova/nova.conf neutron service_metadata_proxy True
crudini --set /etc/nova/nova.conf neutron metadata_proxy_shared_secret LiXiaohui
```

或者

```ini
[neutron]
...
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = LiXiaohui
service_metadata_proxy = True
metadata_proxy_shared_secret = LiXiaohui
```

##### 初始化数据库

网络服务初始化脚本需要一个超链接

```bash
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

```bash
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

##### 重启相关服务

重启计算API 服务

```bash
systemctl restart openstack-nova-api.service
```

网络服务

```bash
systemctl enable neutron-server.service \
  neutron-openvswitch-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service --now
```

对于网络选项2，同样启用layer－3服务并设置其随系统自启动

```bash
systemctl enable neutron-l3-agent.service --now
```

## 安装配置cinder服务

### 控制节点需要完成的事

#### 准备cinder数据库

```bash
mysql -u root -pLiXiaohui
```

```bash
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'LiXiaohui';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'LiXiaohui';
exit
```

#### 创建cinder服务凭据

```bash
[root@controller ~]# openstack user create --domain default --password LiXiaohui cinder
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 8d60192ed38646519d542b3f1ea32d5a |
| name                | cinder                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
[root@controller ~]# openstack role add --project service --user cinder admin
```

#### 创建cinder的service和endpoint

```bash
[root@controller ~]# openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | 0fccaf7817214304a4cd80b2890655ac |
| name        | cinderv3                         |
| type        | volumev3                         |
+-------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 1234bbb80dc244299bd2e8a4eaf3d168         |
| interface    | public                                   |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 0fccaf7817214304a4cd80b2890655ac         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 2e54e026ae3c42e282c4d2bb7ac5113d         |
| interface    | internal                                 |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 0fccaf7817214304a4cd80b2890655ac         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 6b3c08a639854f22ae9b687bca0e9d27         |
| interface    | admin                                    |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | 0fccaf7817214304a4cd80b2890655ac         |
| service_name | cinderv3                                 |
| service_type | volumev3                                 |
| url          | http://controller:8776/v3/%(project_id)s |
+--------------+------------------------------------------+
```

#### 安装配置cinder组件

```bash
yum install openstack-cinder -y
```

`/etc/cinder/cinder.conf` 是cinder的配置文件

```bash
crudini --set /etc/cinder/cinder.conf DEFAULT transport_url rabbit://openstack:LiXiaohui@controller
crudini --set /etc/cinder/cinder.conf DEFAULT auth_strategy keystone
crudini --set /etc/cinder/cinder.conf DEFAULT my_ip 192.168.8.10
crudini --set /etc/cinder/cinder.conf database connection mysql+pymysql://cinder:LiXiaohui@controller/cinder
crudini --set /etc/cinder/cinder.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/cinder/cinder.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_type password
crudini --set /etc/cinder/cinder.conf keystone_authtoken project_domain_name Default
crudini --set /etc/cinder/cinder.conf keystone_authtoken user_domain_name Default
crudini --set /etc/cinder/cinder.conf keystone_authtoken project_name service
crudini --set /etc/cinder/cinder.conf keystone_authtoken username cinder
crudini --set /etc/cinder/cinder.conf keystone_authtoken password LiXiaohui
crudini --set /etc/cinder/cinder.conf oslo_concurrency lock_path /var/lib/cinder/tmp
```

或者

```ini
[DEFAULT]
transport_url = rabbit://openstack:LiXiaohui@controller
auth_strategy = keystone
my_ip = 192.168.8.10

[database]
connection = mysql+pymysql://cinder:LiXiaohui@controller/cinder

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = LiXiaohui

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

#### 填充cinder数据库

```bash
su -s /bin/sh -c "cinder-manage db sync" cinder
```

#### 配置计算服务使用cinder

在`/etc/nova/nova.conf`中编辑

```bash
crudini --set /etc/nova/nova.conf cinder os_region_name RegionOne

```

或

```ini
[cinder]
os_region_name = RegionOne
```

#### 启动相关服务

```bash
systemctl restart openstack-nova-api.service
systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service --now
```

### 存储节点需要完成的事

**没有那么多机器，我并置在了控制节点，存储节点最起码需要有一个额外的硬盘**

#### 准备相应软件包

```bash
yum install lvm2 device-mapper-persistent-data -y
```

#### 准备lvm卷组

我这里额外的盘是/dev/nvme0n2，你的也许是别的名称，用`lsblk`确认

```bash
pvcreate /dev/nvme0n2
vgcreate cinder-volumes /dev/nvme0n2
```

#### 定义lvm扫描范围

默认情况下，LVM卷扫描工具会扫描``/dev`` 目录，查找包含卷的块存储设备。如果项目在他们的卷上使用LVM，扫描工具检测到这些卷时会尝试缓存它们，可能会在底层操作系统和项目卷上产生各种问题。您必须重新配置LVM，让它只扫描包含``cinder-volume``卷组的设备。

需要修改的配置文件是：`/etc/lvm/lvm.conf`

在devices中，添加了一个filter参数

每个过滤器组中的元素都以``a``开头，即为 accept，或以 r 开头，即为**reject**，并且包括一个设备名称的正则表达式规则。过滤器组必须以``r/.*/``结束，过滤所有保留设备。

```bash
devices {
filter = [ "a/nvme0n2/", "r/.*/"]
```

#### 安装配置cinder存储组件

```bash
yum install openstack-cinder targetcli -y
```

#### 准备cinder配置文件

`/etc/cinder/cinder.conf` 是cinder配置文件

```bash
crudini --set /etc/cinder/cinder.conf DEFAULT transport_url rabbit://openstack:LiXiaohui@controller
crudini --set /etc/cinder/cinder.conf DEFAULT auth_strategy keystone
crudini --set /etc/cinder/cinder.conf DEFAULT my_ip 192.168.8.10
crudini --set /etc/cinder/cinder.conf DEFAULT enabled_backends lvm
crudini --set /etc/cinder/cinder.conf DEFAULT glance_api_servers http://controller:9292
crudini --set /etc/cinder/cinder.conf database connection mysql+pymysql://cinder:LiXiaohui@controller/cinder
crudini --set /etc/cinder/cinder.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/cinder/cinder.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/cinder/cinder.conf keystone_authtoken auth_type password
crudini --set /etc/cinder/cinder.conf keystone_authtoken project_domain_name Default
crudini --set /etc/cinder/cinder.conf keystone_authtoken user_domain_name Default
crudini --set /etc/cinder/cinder.conf keystone_authtoken project_name service
crudini --set /etc/cinder/cinder.conf keystone_authtoken username cinder
crudini --set /etc/cinder/cinder.conf keystone_authtoken password LiXiaohui
crudini --set /etc/cinder/cinder.conf oslo_concurrency lock_path /var/lib/cinder/tmp
crudini --set /etc/cinder/cinder.conf lvm volume_driver cinder.volume.drivers.lvm.LVMVolumeDriver
crudini --set /etc/cinder/cinder.conf lvm volume_group cinder-volumes
crudini --set /etc/cinder/cinder.conf lvm target_helper lioadm
crudini --set /etc/cinder/cinder.conf lvm target_protocol iscsi

```

或者

```ini
[DEFAULT]
auth_strategy = keystone
transport_url = rabbit://openstack:LiXiaohui@controller
my_ip = 192.168.8.10
enabled_backends = lvm
glance_api_servers = http://controller:9292

[database]
connection = mysql+pymysql://cinder:LiXiaohui@controller/cinder

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = LiXiaohui

[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = lioadm

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

#### 启动cinder服务

```bash
systemctl enable openstack-cinder-volume.service target.service --now
```

**在所有节点都要完成下列服务启动**

```bash
systemctl enable iscsid --now
```

#### 安装配置cinder备份服务

安装cinder组件

```bash
yum install openstack-cinder -y
```

#### 准备配置文件

##### 备份到本地

`/etc/cinder/cinder.conf` 是cinder的配置文件

```bash
crudini --set /etc/cinder/cinder.conf DEFAULT backup_driver cinder.backup.drivers.posix.PosixBackupDriver
crudini --set /etc/cinder/cinder.conf DEFAULT backup_posix_path /var/lib/cinder/backup
crudini --set /etc/cinder/cinder.conf DEFAULT backup_compression_algorithm zlib
crudini --set /etc/cinder/cinder.conf DEFAULT backup_enable_progress_timer True
```

或者

```ini
[DEFAULT]
backup_driver = cinder.backup.drivers.posix.PosixBackupDriver
backup_posix_path = /var/lib/cinder/backup
backup_compression_algorithm = zlib
backup_enable_progress_timer = True
```

创建出需要的目录

```bash
mkdir -p /var/lib/cinder/backup
chown cinder:cinder /var/lib/cinder/backup
chmod 750 /var/lib/cinder/backup
```

同步数据库

```bash
cinder-manage db sync
```

重启服务

```bash
systemctl enable openstack-cinder-api.service \
openstack-cinder-backup.service \
openstack-cinder-scheduler.service \
openstack-cinder-volume.service

systemctl restart openstack-cinder-api.service \
openstack-cinder-backup.service \
openstack-cinder-scheduler.service \
openstack-cinder-volume.service
```

测试效果

```bash
[root@controller ~]# cd /var/lib/cinder/backup/
[root@controller backup]# ll
total 0
[root@controller backup]# openstack volume create --size 1 test-volume-lxh
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2024-05-30T13:36:09.774469           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 987a3ed8-031e-46aa-a2dd-79399af522fe |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | test-volume-lxh                      |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 1                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | __DEFAULT__                          |
| updated_at          | None                                 |
| user_id             | db56879a38d943ca8aacb93ffddb8f66     |
+---------------------+--------------------------------------+
[root@controller backup]# openstack volume backup create --name backupfile test-volume-lxh
+-----------+--------------------------------------+
| Field     | Value                                |
+-----------+--------------------------------------+
| id        | 57fbf893-7fb9-4609-94c2-f9fab222ede8 |
| name      | backupfile                           |
| volume_id | 987a3ed8-031e-46aa-a2dd-79399af522fe |
+-----------+--------------------------------------+
[root@controller backup]# ll
total 0
drwxr-xr-x. 3 cinder cinder 16 May 30 21:36 57

[root@controller backup]# ll -R /var/lib/cinder/backup/
/var/lib/cinder/backup/:
total 0
drwxr-xr-x. 3 cinder cinder 16 May 30 21:36 57

/var/lib/cinder/backup/57:
total 0
drwxr-xr-x. 3 cinder cinder 50 May 30 21:36 fb

/var/lib/cinder/backup/57/fb:
total 4
drwxrwx---. 2 cinder cinder 4096 May 30 21:36 57fbf893-7fb9-4609-94c2-f9fab222ede8

/var/lib/cinder/backup/57/fb/57fbf893-7fb9-4609-94c2-f9fab222ede8:
total 3332
-rw-rw----. 1 cinder cinder 1043644 May 30 21:36 volume_987a3ed8-031e-46aa-a2dd-79399af522fe_20240530133624_backup_57fbf893-7fb9-4609-94c2-f9fab222ede8-00001
-rw-rw----. 1 cinder cinder    2125 May 30 21:36 volume_987a3ed8-031e-46aa-a2dd-79399af522fe_20240530133624_backup_57fbf893-7fb9-4609-94c2-f9fab222ede8_metadata
-rw-rw----. 1 cinder cinder 2359578 May 30 21:36 volume_987a3ed8-031e-46aa-a2dd-79399af522fe_20240530133624_backup_57fbf893-7fb9-4609-94c2-f9fab222ede8_sha256file

```
##### 备份到Ceph

`/etc/cinder/cinder.conf` 是cinder的配置文件

**需要先搞定ceph的账号和keyring**

```bash
crudini --set /etc/cinder/cinder.conf DEFAULT backup_driver cinder.backup.drivers.ceph.CephBackupDriver
crudini --set /etc/cinder/cinder.conf DEFAULT backup_ceph_conf /etc/ceph/ceph.conf
crudini --set /etc/cinder/cinder.conf DEFAULT backup_ceph_user cinder-backup
crudini --set /etc/cinder/cinder.conf DEFAULT backup_ceph_chunk_size 134217728
crudini --set /etc/cinder/cinder.conf DEFAULT backup_ceph_pool backups
crudini --set /etc/cinder/cinder.conf DEFAULT backup_compression_algorithm zlib
crudini --set /etc/cinder/cinder.conf DEFAULT backup_enable_progress_timer True
```

或者

```ini
[DEFAULT]
backup_driver = cinder.backup.drivers.ceph
backup_ceph_conf = /etc/ceph/ceph.conf  # Ceph集群配置文件路径
backup_ceph_user = cinder-backup  # Ceph用户
backup_ceph_chunk_size = 134217728  # 分块大小（字节）
backup_ceph_pool = backups  # Ceph备份池的名称
backup_compression_algorithm = zlib  # 备份压缩算法
backup_enable_progress_timer = True  # 启用备份进度计时器
```


同步数据库

```bash
cinder-manage db sync
```

重启服务

```bash
systemctl enable openstack-cinder-api.service \
openstack-cinder-backup.service \
openstack-cinder-scheduler.service \
openstack-cinder-volume.service

systemctl restart openstack-cinder-api.service \
openstack-cinder-backup.service \
openstack-cinder-scheduler.service \
openstack-cinder-volume.service
```

测试效果

原来ceph池是空的

```bash
[root@ceph ~]# rados -p backups ls
rbd_directory
rbd_info
rbd_trash

```

创建个卷并备份

```bash
[root@controller cinder]# openstack volume create --size 1 ceph-test-lxh
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2024-05-30T14:03:12.482823           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 4a6f25d5-93e2-49f7-9bf9-ac9913a7dba5 |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | ceph-test-lxh                        |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 1                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | __DEFAULT__                          |
| updated_at          | None                                 |
| user_id             | 1a138f1f200b4d75a50b22ecd1a71903     |
+---------------------+--------------------------------------+
[root@controller cinder]# openstack volume backup create --name cephtest-lxh ceph-test-lxh
+-----------+--------------------------------------+
| Field     | Value                                |
+-----------+--------------------------------------+
| id        | 363b7645-eb5d-46b1-a209-e1348dc2d76a |
| name      | cephtest-lxh                         |
| volume_id | 4a6f25d5-93e2-49f7-9bf9-ac9913a7dba5 |
+-----------+--------------------------------------+

```

已经出现数据

```bash
[root@ceph ~]# rados -p backups ls
rbd_object_map.b6807bde1542.0000000000000006
rbd_object_map.b6807bde1542
rbd_directory
rbd_header.b6807bde1542
rbd_info
rbd_id.volume-4a6f25d5-93e2-49f7-9bf9-ac9913a7dba5.backup.363b7645-eb5d-46b1-a209-e1348dc2d76a
backup.363b7645-eb5d-46b1-a209-e1348dc2d76a.meta
rbd_trash

```

##### 备份到NFS服务器

搭建NFS服务器

```bash
yum install nfs-utils -y
mkdir /backup
chmod 777 /backup -R
echo /backup *(rw,no_all_squash,no_root_squash) > /etc/exports
systemctl enable nfs-server --now
exportfs -rav
```

`/etc/cinder/cinder.conf` 是cinder的配置文件

```bash
crudini --set /etc/cinder/cinder.conf DEFAULT backup_driver cinder.backup.drivers.nfs.NFSBackupDriver
crudini --set /etc/cinder/cinder.conf DEFAULT backup_mount_point_base /mnt/backup
crudini --set /etc/cinder/cinder.conf DEFAULT backup_share 192.168.8.10:/backup
crudini --set /etc/cinder/cinder.conf DEFAULT backup_file_size 209715200
crudini --set /etc/cinder/cinder.conf DEFAULT backup_sha_block_size_bytes 32768
crudini --set /etc/cinder/cinder.conf DEFAULT backup_compression_algorithm zlib
crudini --set /etc/cinder/cinder.conf DEFAULT backup_enable_progress_timer True
```

或者

```ini
[DEFAULT]
backup_driver = cinder.backup.drivers.nfs.NFSBackupDriver
backup_mount_point_base = /mnt/backup  # 备份挂载点基础路径
backup_share = 192.168.8.10:/backup  # NFS共享路径
backup_file_size = 209715200  # 备份文件大小限制（字节）
backup_sha_block_size_bytes = 32768  # SHA校验块大小（字节）
backup_enable_progress_timer = True  # 启用备份进度计时器
```

创建出挂载点

```bash
mkdir /mnt/backup
chown cinder:cinder /mnt/backup -R
chmod g+s /mnt/backup
```

同步数据库

```bash
cinder-manage db sync
```

重启服务

```bash
systemctl enable openstack-cinder-api.service \
openstack-cinder-backup.service \
openstack-cinder-scheduler.service \
openstack-cinder-volume.service

systemctl restart openstack-cinder-api.service \
openstack-cinder-backup.service \
openstack-cinder-scheduler.service \
openstack-cinder-volume.service
```

测试效果

```bash
[root@controller cinder]# openstack volume create --size 1 nfs-test
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2024-05-30T15:01:15.277805           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 3a601d59-020c-4851-bfd6-5d983ac9627f |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | nfs-test                             |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 1                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | __DEFAULT__                          |
| updated_at          | None                                 |
| user_id             | db56879a38d943ca8aacb93ffddb8f66     |
+---------------------+--------------------------------------+
[root@controller cinder]# openstack volume backup create --name nfstest nfs-test
+-----------+--------------------------------------+
| Field     | Value                                |
+-----------+--------------------------------------+
| id        | c30bff9d-e4cb-4fb8-a838-0acf30b3e619 |
| name      | nfstest                              |
| volume_id | 3a601d59-020c-4851-bfd6-5d983ac9627f |
+-----------+--------------------------------------+
[root@controller cinder]# ls /mnt/backup
49  c3
[root@controller cinder]# ls /mnt/backup
49/ c3/
[root@controller cinder]# ls /mnt/backup/c3/0b/c30bff9d-e4cb-4fb8-a838-0acf30b3e619/
volume_3a601d59-020c-4851-bfd6-5d983ac9627f_20240530150137_backup_c30bff9d-e4cb-4fb8-a838-0acf30b3e619-00001
volume_3a601d59-020c-4851-bfd6-5d983ac9627f_20240530150137_backup_c30bff9d-e4cb-4fb8-a838-0acf30b3e619-00002
volume_3a601d59-020c-4851-bfd6-5d983ac9627f_20240530150137_backup_c30bff9d-e4cb-4fb8-a838-0acf30b3e619-00003
volume_3a601d59-020c-4851-bfd6-5d983ac9627f_20240530150137_backup_c30bff9d-e4cb-4fb8-a838-0acf30b3e619-00004
volume_3a601d59-020c-4851-bfd6-5d983ac9627f_20240530150137_backup_c30bff9d-e4cb-4fb8-a838-0acf30b3e619-00005
volume_3a601d59-020c-4851-bfd6-5d983ac9627f_20240530150137_backup_c30bff9d-e4cb-4fb8-a838-0acf30b3e619-00006
volume_3a601d59-020c-4851-bfd6-5d983ac9627f_20240530150137_backup_c30bff9d-e4cb-4fb8-a838-0acf30b3e619_metadata
volume_3a601d59-020c-4851-bfd6-5d983ac9627f_20240530150137_backup_c30bff9d-e4cb-4fb8-a838-0acf30b3e619_sha256file
```

## 安装配置manila服务

### 为manila准备控制节点

并置在controller这台机器上

#### 准备manila数据库

```bash
mysql -u root -pLiXiaohui
```
```bash
CREATE DATABASE manila;
GRANT ALL PRIVILEGES ON manila.* TO 'manila'@'localhost' IDENTIFIED BY 'LiXiaohui';
GRANT ALL PRIVILEGES ON manila.* TO 'manila'@'%' IDENTIFIED BY 'LiXiaohui';
exit
```

#### 创建manila服务凭据

```bash
[root@controller ~]# openstack user create --domain default --password LiXiaohui manila
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 4427fcb6f82d412cac8da44118c2d1b1 |
| name                | manila                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
[root@controller ~]# openstack role add --project service --user manila admin

[root@controller ~]# openstack service create --name manila --description "OpenStack Shared File Systems" share
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Shared File Systems    |
| enabled     | True                             |
| id          | 3fc0e0b3f5404a8b8fe2e22e31c1fb97 |
| name        | manila                           |
| type        | share                            |
+-------------+----------------------------------+
[root@controller ~]# openstack service create --name manilav2 --description "OpenStack Shared File Systems V2" sharev2
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Shared File Systems V2 |
| enabled     | True                             |
| id          | e41e8c69b86e4e74bba6b2c931f7d673 |
| name        | manilav2                         |
| type        | sharev2                          |
+-------------+----------------------------------+

```

#### 创建manila endpoint

```bash
[root@controller ~]# openstack endpoint create --region RegionOne share public http://controller:8786/v1/%\(tenant_id\)s
+--------------+-----------------------------------------+
| Field        | Value                                   |
+--------------+-----------------------------------------+
| enabled      | True                                    |
| id           | 22a3ac8a89874f94a8c9a53a120ac165        |
| interface    | public                                  |
| region       | RegionOne                               |
| region_id    | RegionOne                               |
| service_id   | 3fc0e0b3f5404a8b8fe2e22e31c1fb97        |
| service_name | manila                                  |
| service_type | share                                   |
| url          | http://controller:8786/v1/%(tenant_id)s |
+--------------+-----------------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne share internal http://controller:8786/v1/%\(tenant_id\)s
+--------------+-----------------------------------------+
| Field        | Value                                   |
+--------------+-----------------------------------------+
| enabled      | True                                    |
| id           | 385626d50d324d39873a503e6a81423c        |
| interface    | internal                                |
| region       | RegionOne                               |
| region_id    | RegionOne                               |
| service_id   | 3fc0e0b3f5404a8b8fe2e22e31c1fb97        |
| service_name | manila                                  |
| service_type | share                                   |
| url          | http://controller:8786/v1/%(tenant_id)s |
+--------------+-----------------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne share admin http://controller:8786/v1/%\(tenant_id\)s
+--------------+-----------------------------------------+
| Field        | Value                                   |
+--------------+-----------------------------------------+
| enabled      | True                                    |
| id           | 28568224c35e4c5b8d533888decfbe5f        |
| interface    | admin                                   |
| region       | RegionOne                               |
| region_id    | RegionOne                               |
| service_id   | 3fc0e0b3f5404a8b8fe2e22e31c1fb97        |
| service_name | manila                                  |
| service_type | share                                   |
| url          | http://controller:8786/v1/%(tenant_id)s |
+--------------+-----------------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne sharev2 public http://controller:8786/v2
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 7fdf805093a24aa09a4eee27681b38c2 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | e41e8c69b86e4e74bba6b2c931f7d673 |
| service_name | manilav2                         |
| service_type | sharev2                          |
| url          | http://controller:8786/v2        |
+--------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne sharev2 internal http://controller:8786/v2
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 47b4ce4125294061a945e595c22770c4 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | e41e8c69b86e4e74bba6b2c931f7d673 |
| service_name | manilav2                         |
| service_type | sharev2                          |
| url          | http://controller:8786/v2        |
+--------------+----------------------------------+
[root@controller ~]# openstack endpoint create --region RegionOne sharev2 admin http://controller:8786/v2
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | a88c310beef743fda7d8f93cd7f44b27 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | e41e8c69b86e4e74bba6b2c931f7d673 |
| service_name | manilav2                         |
| service_type | sharev2                          |
| url          | http://controller:8786/v2        |
+--------------+----------------------------------+

```

#### 安装配置minila组件

```bash
yum install openstack-manila python3-manilaclient -y
```

#### 准备配置文件

`/etc/manila/manila.conf` 是manila配置文件

```bash
crudini --set /etc/manila/manila.conf DEFAULT my_ip 192.168.8.10
crudini --set /etc/manila/manila.conf DEFAULT auth_strategy keystone
crudini --set /etc/manila/manila.conf DEFAULT default_share_type default_share_type
crudini --set /etc/manila/manila.conf DEFAULT share_name_template share-%s
crudini --set /etc/manila/manila.conf DEFAULT rootwrap_config /etc/manila/rootwrap.conf
crudini --set /etc/manila/manila.conf DEFAULT api_paste_config /etc/manila/api-paste.ini
crudini --set /etc/manila/manila.conf DEFAULT transport_url rabbit://openstack:LiXiaohui@controller
crudini --set /etc/manila/manila.conf database connection mysql+pymysql://manila:LiXiaohui@controller/manila
crudini --set /etc/manila/manila.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/manila/manila.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/manila/manila.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/manila/manila.conf keystone_authtoken auth_type password
crudini --set /etc/manila/manila.conf keystone_authtoken project_domain_name Default
crudini --set /etc/manila/manila.conf keystone_authtoken user_domain_name Default
crudini --set /etc/manila/manila.conf keystone_authtoken project_name service
crudini --set /etc/manila/manila.conf keystone_authtoken username manila
crudini --set /etc/manila/manila.conf keystone_authtoken password LiXiaohui
crudini --set /etc/manila/manila.conf oslo_concurrency lock_path /var/lock/manila
```


或者

```ini
[DEFAULT]
my_ip = 192.168.8.10
auth_strategy = keystone
default_share_type = default_share_type
share_name_template = share-%s
rootwrap_config = /etc/manila/rootwrap.conf
api_paste_config = /etc/manila/api-paste.ini
transport_url = rabbit://openstack:LiXiaohui@controller

[database]
connection = mysql+pymysql://manila:LiXiaohui@controller/manila

[keystone_authtoken]
...
memcached_servers = controller:11211
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = manila
password = LiXiaohui

[oslo_concurrency]
lock_path = /var/lock/manila
```

#### 填充manila数据库

```bash
su -s /bin/sh -c "manila-manage db sync" manila
```

#### 启动manila服务

```bash
systemctl enable openstack-manila-api.service openstack-manila-scheduler.service --now
```

### 安装配置manila共享节点

并置在controller上

#### 安装配置相应软件

```bash
yum install openstack-manila-share python3-PyMySQL -y
```

#### 准备配置文件

`/etc/manila/manila.conf`是manila的配置文件

```bash
crudini --set /etc/manila/manila.conf DEFAULT my_ip 192.168.8.10
crudini --set /etc/manila/manila.conf DEFAULT auth_strategy keystone
crudini --set /etc/manila/manila.conf DEFAULT default_share_type default_share_type
crudini --set /etc/manila/manila.conf DEFAULT rootwrap_config /etc/manila/rootwrap.conf
crudini --set /etc/manila/manila.conf DEFAULT transport_url rabbit://openstack:LiXiaohui@controller
crudini --set /etc/manila/manila.conf database connection mysql+pymysql://manila:LiXiaohui@controller/manila
crudini --set /etc/manila/manila.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/manila/manila.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/manila/manila.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/manila/manila.conf keystone_authtoken auth_type password
crudini --set /etc/manila/manila.conf keystone_authtoken project_domain_name Default
crudini --set /etc/manila/manila.conf keystone_authtoken user_domain_name Default
crudini --set /etc/manila/manila.conf keystone_authtoken project_name service
crudini --set /etc/manila/manila.conf keystone_authtoken username manila
crudini --set /etc/manila/manila.conf keystone_authtoken password LiXiaohui
crudini --set /etc/manila/manila.conf oslo_concurrency lock_path /var/lock/manila
```


或

```ini
[DEFAULT]
auth_strategy = keystone
my_ip = 192.168.8.10
default_share_type = default_share_type
rootwrap_config = /etc/manila/rootwrap.conf
transport_url = rabbit://openstack:LiXiaohui@controller

[database]
connection = mysql+pymysql://manila:LiXiaohui@controller/manila

[keystone_authtoken]
memcached_servers = controller:11211
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = manila
password = LiXiaohui

[oslo_concurrency]
lock_path = /var/lib/manila/tmp
```

#### manila 驱动模式说明

共享节点（Share Node）可以支持两种模式，即带有共享服务器处理和不带共享服务器处理。这两种模式的选择取决于所使用的驱动程序是否支持。

共享节点可以根据所使用的驱动程序的支持情况，在两种不同的模式下运行。

1. **带有共享服务器处理**：在这种模式下，共享节点负责创建和管理共享服务器实例，每个共享都有一个独立的服务器实例来处理共享存储的访问和管理。这种模式通常用于需要对外提供服务的场景，需要为每个共享创建独立的服务器实例。

2. **不带有共享服务器处理**：在这种模式下，共享节点不会为每个共享创建单独的服务器实例，而是直接在存储后端上创建共享文件系统。这种模式通常用于简单的部署场景或私有云环境，不需要为每个共享创建独立的服务器实例。

#### 部署本地驱动模式的manila

需要注意的是，我们这里默认采用了LVM，而且单独给controller又添加了一个硬盘，注意不要复用cinder服务的硬盘

##### 安装相应软件并启动服务

```bash
yum install lvm2 nfs-utils nfs4-acl-tools portmap targetcli -y
systemctl enable target.service --now
```

##### 准备后端存储

我新添加的硬盘是**/dev/nvme0n3**

```bash
pvcreate /dev/nvme0n3
vgcreate manila-volumes /dev/nvme0n3
```

屏蔽lvm扫描其他设备

/etc/lvm/lvm.conf中修改

```ini
devices {
filter = [ "a/nvme/","r/.*/"]
```

##### 准备配置文件

`/etc/manila/manila.conf`是manila的配置文件

```bash
crudini --set /etc/manila/manila.conf DEFAULT enabled_share_backends lvm
crudini --set /etc/manila/manila.conf DEFAULT enabled_share_protocols NFS
crudini --set /etc/manila/manila.conf lvm share_backend_name LVM
crudini --set /etc/manila/manila.conf lvm share_driver manila.share.drivers.lvm.LVMShareDriver
crudini --set /etc/manila/manila.conf lvm driver_handles_share_servers False
crudini --set /etc/manila/manila.conf lvm lvm_share_volume_group manila-volumes
crudini --set /etc/manila/manila.conf lvm lvm_share_export_ips 192.168.8.10
```


或者

```bash
[DEFAULT]
enabled_share_backends = lvm
enabled_share_protocols = NFS

[lvm]
share_backend_name = LVM
share_driver = manila.share.drivers.lvm.LVMShareDriver
driver_handles_share_servers = False
lvm_share_volume_group = manila-volumes
lvm_share_export_ips = 192.168.8.10
```

##### 启动manila共享服务

```bash
systemctl enable openstack-manila-share.service openstack-manila-data.service --now
```

## 安装配置horizon服务

### 安装相应软件

```bash
yum install openstack-dashboard -y
```

### 准备配置文件

`/etc/openstack-dashboard/local_settings`是horizon的配置文件

搜索以下的参数并修改，如果不存在，就添加

```ini
OPENSTACK_HOST = "controller"
ALLOWED_HOSTS = ['*'] #这里是允许谁来访问
```

配置memcached

这个可能需要添加

**bobcat 版本(2023.2)用下面这个**

```ini
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}
```

**caracal 版本(2024.1)用下面这个**

```ini
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
         'LOCATION': 'controller:11211',
    }
}
```


配置keystone api

```ini
OPENSTACK_KEYSTONE_URL = "http://%s:5000/identity/v3" % OPENSTACK_HOST
```

配置WEBROOT

这个可能需要添加

```ini
WEBROOT = '/dashboard/'
```

启用domain的支持

这个可能需要添加

```ini
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
```

配置API版本

这个可能需要添加

```ini
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}
```

配置创建的用户默认角色为user

这个可能需要添加

```ini
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
```

配置时区（可选）

```ini
TIME_ZONE = "TIME_ZONE"
```

**确保在/etc/httpd/conf.d/openstack-dashboard.conf中包括以下行**

```bash
vi /etc/httpd/conf.d/openstack-dashboard.conf
```

```ini
WSGIApplicationGroup %{GLOBAL}
```

### 处理配置文件故障

horizon的默认配置文件存在bug，需要解决

将以下相似行，修改为下列的路径

```bash
WSGIScriptAlias /dashboard /usr/share/openstack-dashboard/openstack_dashboard/wsgi.py
<Directory /usr/share/openstack-dashboard/>
```

### 启动相关服务

```bash
systemctl restart httpd.service memcached.service
```

# 计算节点上的服务

## 配置网络

**管理网络配置:**

1. IP: 192.168.8.20/23

2. 网关: 192.168.8.2

3. DNS: 223.5.5.5

配置方法：

```bash
cat > /etc/NetworkManager/system-connections/ens160.nmconnection <<-'EOF'
[connection]
id=ens160
uuid=0a87b242-54f7-49f2-addd-9cf55772e2f3
type=ethernet
autoconnect-priority=-999
interface-name=ens160
timestamp=1715837362

[ethernet]

[ipv4]
address1=192.168.8.20/24,192.168.8.2
dns=223.5.5.5;
method=manual

[ipv6]
addr-gen-mode=eui64
method=auto

[proxy]
EOF
```

**Provider网络要求：**

1. 这个网络要求没有任何有效的IP地址

2. 重启服务器要求会自动激活接口


配置方法：

```bash
cat > /etc/NetworkManager/system-connections/ens192.nmconnection <<-'EOF'
[connection]
id=ens192
uuid=782ef496-4742-4bda-9e4a-01ef691ab0a8
type=ethernet
autoconnect-priority=-999
interface-name=ens192
timestamp=1715865778

[ethernet]

[ipv4]
method=auto

[ipv6]
addr-gen-mode=eui64
method=auto

[proxy]
EOF
```

两个网卡配置好之后，重启NetworkManager服务

```bash
systemctl restart NetworkManager
```

## 配置名称解析

```textile
cat > /etc/hosts <<EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.8.10 controller
192.168.8.20 node
EOF
```

这里要自己测试是否能ping通各个其他节点，确保通信正常

## 关闭防火墙和SELINUX

```bash
systemctl disable firewalld --now
setenforce 0
sed -i 's/SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
```

## 配置NTP

对于centos来说，默认就在chrony的配置文件中包含了正确的NTP服务器，所以只需要启动服务即可，你也可以在/etc/chrony.conf中注释掉自带的服务器，然后设置为控制节点也行

```bash
dnf install chrony -y
systemctl enable chronyd --now
```

## 验证NTP

```bash
[root@node ~]# chronyc sources
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^+ 111.230.189.174               2   8   377    59  -3680us[-3718us] +/-   51ms
^* dns1.synet.edu.cn             1   7   377    58  +5535us[+5497us] +/-   28ms
^- ntp1.flashdance.cx            2   8   377   124    -32ms[  -32ms] +/-  128ms
^- ntp7.flashdance.cx            2   8   377   123    -26ms[  -26ms] +/-  125ms
```

## 启用软件仓库

```bash
dnf install dnf-plugins-core -y
dnf config-manager --set-enabled crb
dnf install centos-release-openstack-caracal vim -y
```

## 安装openstack客户端

```bash
dnf install python3-openstackclient openstack-selinux -y
```

## 安装CRUDINI编辑器（可选）

这一步是为了防止手工vim修改配置文件改错，如果你熟练掌握vim搜索和输入，这一步并不是必须的

```bash
yum install crudini -y
```

## 安装配置nova服务

控制节点已经准备好了数据库以及nova用户密码等信息，无需重复在controller做，只在计算节点

```bash
yum install libvirt openstack-nova-compute -y
```

## 准备nova配置文件

`/etc/nova/nova.conf `是nova配置文件

```bash
crudini --set /etc/nova/nova.conf DEFAULT enabled_apis osapi_compute,metadata
crudini --set /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:LiXiaohui@controller
crudini --set /etc/nova/nova.conf DEFAULT my_ip 192.168.8.20
crudini --set /etc/nova/nova.conf DEFAULT compute_driver libvirt.LibvirtDriver
crudini --set /etc/nova/nova.conf DEFAULT log_dir /var/log/nova
crudini --set /etc/nova/nova.conf DEFAULT lock_path /var/lock/nova
crudini --set /etc/nova/nova.conf DEFAULT state_path /var/lib/nova
crudini --set /etc/nova/nova.conf api auth_strategy keystone
crudini --set /etc/nova/nova.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/nova/nova.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/nova/nova.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/nova/nova.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/nova/nova.conf keystone_authtoken auth_type password
crudini --set /etc/nova/nova.conf keystone_authtoken project_domain_name Default
crudini --set /etc/nova/nova.conf keystone_authtoken user_domain_name Default
crudini --set /etc/nova/nova.conf keystone_authtoken project_name service
crudini --set /etc/nova/nova.conf keystone_authtoken username nova
crudini --set /etc/nova/nova.conf keystone_authtoken password LiXiaohui
crudini --set /etc/nova/nova.conf service_user send_service_user_token true
crudini --set /etc/nova/nova.conf service_user auth_url http://controller:5000/identity
crudini --set /etc/nova/nova.conf service_user auth_strategy keystone
crudini --set /etc/nova/nova.conf service_user auth_type password
crudini --set /etc/nova/nova.conf service_user project_domain_name Default
crudini --set /etc/nova/nova.conf service_user user_domain_name Default
crudini --set /etc/nova/nova.conf service_user project_name service
crudini --set /etc/nova/nova.conf service_user username nova
crudini --set /etc/nova/nova.conf service_user password LiXiaohui
crudini --set /etc/nova/nova.conf neutron url http://controller:9696
crudini --set /etc/nova/nova.conf neutron auth_url http://controller:5000
crudini --set /etc/nova/nova.conf neutron auth_type password
crudini --set /etc/nova/nova.conf neutron project_domain_name default
crudini --set /etc/nova/nova.conf neutron user_domain_name default
crudini --set /etc/nova/nova.conf neutron region_name RegionOne
crudini --set /etc/nova/nova.conf neutron project_name service
crudini --set /etc/nova/nova.conf neutron username neutron
crudini --set /etc/nova/nova.conf neutron password LiXiaohui
crudini --set /etc/nova/nova.conf neutron service_metadata_proxy True
crudini --set /etc/nova/nova.conf neutron metadata_proxy_shared_secret LiXiaohui
crudini --set /etc/nova/nova.conf vnc enabled true
crudini --set /etc/nova/nova.conf vnc server_listen ' $my_ip'
crudini --set /etc/nova/nova.conf vnc server_proxyclient_address ' $my_ip'
crudini --set /etc/nova/nova.conf vnc novncproxy_base_url http://192.168.8.10:6080/vnc_auto.html
crudini --set /etc/nova/nova.conf glance api_servers http://controller:9292
crudini --set /etc/nova/nova.conf oslo_concurrency lock_path /var/lib/nova/tmp
crudini --set /etc/nova/nova.conf placement region_name RegionOne
crudini --set /etc/nova/nova.conf placement project_domain_name Default
crudini --set /etc/nova/nova.conf placement project_name service
crudini --set /etc/nova/nova.conf placement auth_type password
crudini --set /etc/nova/nova.conf placement user_domain_name Default
crudini --set /etc/nova/nova.conf placement auth_url http://controller:5000/v3
crudini --set /etc/nova/nova.conf placement username placement
crudini --set /etc/nova/nova.conf placement password LiXiaohui
crudini --set /etc/nova/nova.conf libvirt virt_type qemu  #如果是物理机或者支持虚拟化硬件加速，请设置为kvm

```

或者

```ini
[DEFAULT]
...
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:LiXiaohui@controller
my_ip = 192.168.8.20
compute_driver = libvirt.LibvirtDriver
log_dir = /var/log/nova
lock_path = /var/lock/nova
state_path = /var/lib/nova

[api]
...
auth_strategy = keystone

[keystone_authtoken]
...
www_authenticate_uri = http://controller:5000/
auth_uri = http://controller:5000
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = LiXiaohui

[service_user]
send_service_user_token = true
auth_url = http://controller:5000/identity
auth_strategy = keystone
auth_type = password
project_domain_name = Default
project_name = service
user_domain_name = Default
username = nova
password = LiXiaohui

[vnc]
...
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://192.168.8.10:6080/vnc_auto.html

[glance]
...
api_servers = http://controller:9292

[oslo_concurrency]
...
lock_path = /var/lib/nova/tmp

[placement]
...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = LiXiaohui
```

## 启动nova服务

```bash
systemctl enable libvirtd.service openstack-nova-compute.service --now
```

## 启用网络模块支持

```bash
cat <<EOF | sudo tee /etc/modules-load.d/openstack.conf
br_netfilter
EOF
modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

接下来提供了LinuxBridge、OpenvSwitch和Geneve三种方式，随便哪个都可以，不要在本文档中采用Geneve，没有写完

## 配置Geneve方式的网络代理

```bash
yum install ovn-host openvswitch openstack-neutron-ovn-metadata-agent openstack-neutron openstack-neutron-ml2 -y
```

启动openvswitch服务

```bash
systemctl enable openvswitch --now
```

### 配置通用组件

通用组件的配置包括认证机制、消息队列和插件

`/etc/neutron/neutron.conf`是主服务配置文件

```bash
crudini --set /etc/neutron/neutron.conf DEFAULT rpc_backend rabbit
crudini --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:LiXiaohui@controller
crudini --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_host controller
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_userid openstack
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_password LiXiaohui
crudini --set /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_type password
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_domain_name Default
crudini --set /etc/neutron/neutron.conf keystone_authtoken user_domain_name Default
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_name service
crudini --set /etc/neutron/neutron.conf keystone_authtoken username neutron
crudini --set /etc/neutron/neutron.conf keystone_authtoken password LiXiaohui
crudini --set /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp
```

或者


```ini
[DEFAULT]
...
rpc_backend = rabbit
transport_url = rabbit://openstack:LiXiaohui@controller
auth_strategy = keystone

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = LiXiaohui

[keystone_authtoken]
...
www_authenticate_uri = http://controller:5000
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = LiXiaohui

[oslo_concurrency]
...
lock_path = /var/lib/neutron/tmp
```

### 配置OVS服务

配置对端IP来使用控制节点数据库

```bash
ovs-vsctl set open . external-ids:ovn-remote=tcp:192.168.8.10:6642
```

启用多个网络协议

```bash
ovs-vsctl set open . external-ids:ovn-encap-type=geneve,vxlan
```

配置本端IP地址

```bash
ovs-vsctl set open . external-ids:ovn-encap-ip=192.168.8.20
```

### 启动ovn服务

```bash
systemctl enable ovn-controller neutron-ovn-metadata-agent --now
```

## 配置Linux Bridge方式的网络代理

```bash
yum install libvirt openstack-nova-compute -y
```

```bash
yum install openstack-neutron-linuxbridge ebtables ipset -y
```


### 配置通用组件

通用组件的配置包括认证机制、消息队列和插件

`/etc/neutron/neutron.conf`是主服务配置文件

```bash
crudini --set /etc/neutron/neutron.conf DEFAULT rpc_backend rabbit
crudini --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:LiXiaohui@controller
crudini --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_host controller
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_userid openstack
crudini --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_password LiXiaohui
crudini --set /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/neutron/neutron.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/neutron/neutron.conf keystone_authtoken auth_type password
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_domain_name Default
crudini --set /etc/neutron/neutron.conf keystone_authtoken user_domain_name Default
crudini --set /etc/neutron/neutron.conf keystone_authtoken project_name service
crudini --set /etc/neutron/neutron.conf keystone_authtoken username neutron
crudini --set /etc/neutron/neutron.conf keystone_authtoken password LiXiaohui
crudini --set /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp
```

或者


```ini
[DEFAULT]
...
rpc_backend = rabbit
transport_url = rabbit://openstack:LiXiaohui@controller
auth_strategy = keystone

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = LiXiaohui

[keystone_authtoken]
...
www_authenticate_uri = http://controller:5000
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = LiXiaohui

[oslo_concurrency]
...
lock_path = /var/lib/neutron/tmp
```

### 配置Linuxbridge代理

Linuxbridge代理为实例建立layer－2虚拟网络并且处理安全组规则

`/etc/neutron/plugins/ml2/linuxbridge_agent.ini` 是Linuxbridge代理配置文件

```bash
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini linux_bridge physical_interface_mappings provider:ens160
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan enable_vxlan True
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan local_ip 192.168.8.20 #这里是管理物理接口的IP
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan l2_population true
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup enable_security_group True
crudini --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup firewall_driver neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

或者


```ini
[linux_bridge]
physical_interface_mappings = provider:ens160

[vxlan]
enable_vxlan = True
local_ip = 192.168.8.20 #这里是管理物理接口的IP
l2_population = true

[securitygroup]
...
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

### 准备配置文件

`/etc/nova/nova.conf `是nova配置文件

```bash
crudini --set /etc/nova/nova.conf DEFAULT enabled_apis osapi_compute,metadata
crudini --set /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:LiXiaohui@controller
crudini --set /etc/nova/nova.conf DEFAULT my_ip 192.168.8.20
crudini --set /etc/nova/nova.conf DEFAULT auth_strategy keystone
crudini --set /etc/nova/nova.conf DEFAULT compute_driver libvirt.LibvirtDriver
crudini --set /etc/nova/nova.conf DEFAULT log_dir /var/log/nova
crudini --set /etc/nova/nova.conf DEFAULT lock_path /var/lock/nova
crudini --set /etc/nova/nova.conf DEFAULT state_path /var/lib/nova
crudini --set /etc/nova/nova.conf api auth_strategy keystone
crudini --set /etc/nova/nova.conf keystone_authtoken www_authenticate_uri http://controller:5000
crudini --set /etc/nova/nova.conf keystone_authtoken auth_uri http://controller:5000
crudini --set /etc/nova/nova.conf keystone_authtoken auth_url http://controller:5000
crudini --set /etc/nova/nova.conf keystone_authtoken memcached_servers controller:11211
crudini --set /etc/nova/nova.conf keystone_authtoken auth_type password
crudini --set /etc/nova/nova.conf keystone_authtoken project_domain_name Default
crudini --set /etc/nova/nova.conf keystone_authtoken user_domain_name Default
crudini --set /etc/nova/nova.conf keystone_authtoken project_name service
crudini --set /etc/nova/nova.conf keystone_authtoken username nova
crudini --set /etc/nova/nova.conf keystone_authtoken password LiXiaohui
crudini --set /etc/nova/nova.conf service_user send_service_user_token true
crudini --set /etc/nova/nova.conf service_user auth_url http://controller:5000/identity
crudini --set /etc/nova/nova.conf service_user auth_strategy keystone
crudini --set /etc/nova/nova.conf service_user auth_type password
crudini --set /etc/nova/nova.conf service_user project_domain_name Default
crudini --set /etc/nova/nova.conf service_user user_domain_name Default
crudini --set /etc/nova/nova.conf service_user project_name service
crudini --set /etc/nova/nova.conf service_user username nova
crudini --set /etc/nova/nova.conf service_user password LiXiaohui
crudini --set /etc/nova/nova.conf neutron url http://controller:9696
crudini --set /etc/nova/nova.conf neutron auth_url http://controller:5000
crudini --set /etc/nova/nova.conf neutron auth_type password
crudini --set /etc/nova/nova.conf neutron project_domain_name default
crudini --set /etc/nova/nova.conf neutron user_domain_name default
crudini --set /etc/nova/nova.conf neutron region_name RegionOne
crudini --set /etc/nova/nova.conf neutron project_name service
crudini --set /etc/nova/nova.conf neutron username neutron
crudini --set /etc/nova/nova.conf neutron password LiXiaohui
crudini --set /etc/nova/nova.conf neutron service_metadata_proxy True
crudini --set /etc/nova/nova.conf neutron metadata_proxy_shared_secret LiXiaohui
crudini --set /etc/nova/nova.conf vnc enabled true
crudini --set /etc/nova/nova.conf vnc server_listen ' $my_ip'
crudini --set /etc/nova/nova.conf vnc server_proxyclient_address ' $my_ip'
crudini --set /etc/nova/nova.conf vnc novncproxy_base_url http://192.168.8.10:6080/vnc_auto.html
crudini --set /etc/nova/nova.conf glance api_servers http://controller:9292
crudini --set /etc/nova/nova.conf oslo_concurrency lock_path /var/lib/nova/tmp
crudini --set /etc/nova/nova.conf placement region_name RegionOne
crudini --set /etc/nova/nova.conf placement project_domain_name Default
crudini --set /etc/nova/nova.conf placement project_name service
crudini --set /etc/nova/nova.conf placement auth_type password
crudini --set /etc/nova/nova.conf placement user_domain_name Default
crudini --set /etc/nova/nova.conf placement auth_url http://controller:5000/v3
crudini --set /etc/nova/nova.conf placement username placement
crudini --set /etc/nova/nova.conf placement password LiXiaohui
crudini --set /etc/nova/nova.conf libvirt virt_type qemu  #如果是物理机或者支持虚拟化硬件加速，请设置为kvm

```

或者

```ini
[DEFAULT]
...
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:LiXiaohui@controller
my_ip = 192.168.8.20
auth_strategy = keystone
compute_driver = libvirt.LibvirtDriver
log_dir = /var/log/nova
lock_path = /var/lock/nova
state_path = /var/lib/nova

[api]
...
auth_strategy = keystone

[keystone_authtoken]
...
www_authenticate_uri = http://controller:5000/
auth_uri = http://controller:5000
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = LiXiaohui

[service_user]
send_service_user_token = true
auth_url = http://controller:5000/identity
auth_strategy = keystone
auth_type = password
project_domain_name = Default
project_name = service
user_domain_name = Default
username = nova
password = LiXiaohui

[vnc]
...
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://192.168.8.10:6080/vnc_auto.html

[glance]
...
api_servers = http://controller:9292

[oslo_concurrency]
...
lock_path = /var/lib/nova/tmp

[placement]
...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = LiXiaohui
```

### 检测计算节点是否支持虚拟化硬件加速

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

如果此命令返回的值为1或更大，则表示计算节点支持硬件加速，这通常不需要额外的配置。

如果此命令返回值为0，则表示计算节点不支持硬件加速，必须在 `/etc/nova/nova.conf` 中将libvirt配置为使用QEMU而不是KVM。

```ini
[libvirt]
...
virt_type = qemu # 如果是物理机或者支持虚拟化硬件加速，请设置为kvm
```

### 启动nova服务

```bash
systemctl enable libvirtd.service openstack-nova-compute.service neutron-linuxbridge-agent.service iscsid --now
```

## 配置OpenvSwtich方式的网络代理

```bash
yum install openstack-neutron-openvswitch -y
```

### 配置网络通用组件

通用组件的配置包括认证机制、消息队列和插件

`/etc/neutron/neutron.conf`是主服务配置文件

```bash
crudini --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:LiXiaohui@controller
crudini --set /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp
```

或者


```ini
[DEFAULT]
...
transport_url = rabbit://openstack:LiXiaohui@controller

[oslo_concurrency]
...
lock_path = /var/lib/neutron/tmp
```
### 准备OVS网桥

后续的配置中，会用到br-int和br-ex网桥，提前创建出来

```bash
systemctl enable neutron-openvswitch-agent.service --now
```

检查br-int网桥是否创建出来，如果没有，那就自己创建br-int网桥用于虚拟机内部通信，一般来说，启动服务会自动创建

可以看到我的br-int已经自动创建

```bash
[root@controller ~]# ovs-vsctl show
1b37f8e1-ef53-46f1-8d0e-ce60268a0346
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port br-int
            Interface br-int
                type: internal
    ovs_version: "3.3.1"
```

如果没有创建，就执行以下命令即可

```bash
[root@controller ~]# ovs-vsctl add-br br-int
```

确认了br-int之后，需要创建br-ex用于外部网络通信，并添加具有外部通信的网卡进来

```bash
[root@controller ~]# ovs-vsctl add-br br-ex
[root@controller ~]# ovs-vsctl add-port br-ex ens192
```
### 配置OpenvSwitch 代理

`/etc/neutron/plugins/ml2/openvswitch_agent.ini` 是ovs代理配置文件

```bash
crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs bridge_mappings provider:br-ex
crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs local_ip 192.168.8.20
crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini agent tunnel_types vxlan
crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini agent l2_population true
crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini securitygroup enable_security_group true
crudini --set /etc/neutron/plugins/ml2/openvswitch_agent.ini securitygroup firewall_driver openvswitch
```

或

```ini
[ovs]
bridge_mappings = provider:br-ex
local_ip = 192.168.8.20

[agent]
tunnel_types = vxlan
l2_population = true

[securitygroup]
enable_security_group = true
firewall_driver = openvswitch
```

## 在nova服务中使用neutron

`/etc/nova/nova.conf`是nova配置文件

```bash
crudini --set /etc/nova/nova.conf neutron url http://controller:9696
crudini --set /etc/nova/nova.conf neutron auth_url http://controller:5000
crudini --set /etc/nova/nova.conf neutron auth_type password
crudini --set /etc/nova/nova.conf neutron project_domain_name default
crudini --set /etc/nova/nova.conf neutron user_domain_name default
crudini --set /etc/nova/nova.conf neutron region_name RegionOne
crudini --set /etc/nova/nova.conf neutron project_name service
crudini --set /etc/nova/nova.conf neutron username neutron
crudini --set /etc/nova/nova.conf neutron password LiXiaohui
crudini --set /etc/nova/nova.conf neutron service_metadata_proxy True
crudini --set /etc/nova/nova.conf neutron metadata_proxy_shared_secret LiXiaohui
```

或者

```ini
[neutron]
...
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = LiXiaohui
service_metadata_proxy = True
metadata_proxy_shared_secret = LiXiaohui
```
## 启动服务

```bash
systemctl enable openstack-nova-compute.service
systemctl restart openstack-nova-compute.service
```

```bash
systemctl enable neutron-openvswitch-agent.service --now
```


## 将计算节点注册到数据库

**注意这个步骤要在控制节点完成**

```bash
source adminrc.sh
[root@controller ~]# su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
Modules with known eventlet monkey patching issues were imported prior to eventlet monkey patching: urllib3. This warning can usually be ignored if the caller is only importing and not executing nova code.
2024-05-17 14:07:38.326 10888 WARNING oslo_policy.policy [None req-6a01ed55-7538-44ed-9915-34a1c6ca328c - - - - - -] JSON formatted policy_file support is deprecated since Victoria release. You need to use YAML format which will be default in future. You can use ``oslopolicy-convert-json-to-yaml`` tool to convert existing JSON-formatted policy file to YAML-formatted in backward compatible way: https://docs.openstack.org/oslo.policy/latest/cli/oslopolicy-convert-json-to-yaml.html.
2024-05-17 14:07:38.327 10888 WARNING oslo_policy.policy [None req-6a01ed55-7538-44ed-9915-34a1c6ca328c - - - - - -] JSON formatted policy_file support is deprecated since Victoria release. You need to use YAML format which will be default in future. You can use ``oslopolicy-convert-json-to-yaml`` tool to convert existing JSON-formatted policy file to YAML-formatted in backward compatible way: https://docs.openstack.org/oslo.policy/latest/cli/oslopolicy-convert-json-to-yaml.html.
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': a906846b-650e-42c1-8d9e-dcf5cbec6a6e
Found 1 unmapped computes in cell: a906846b-650e-42c1-8d9e-dcf5cbec6a6e
```

添加新计算节点时，必须在控制节点上执行nova-manage cell_v2 discover_hosts命令注册新计算节点，或者在配置文件`/etc/nova/nova.conf`设置一个特定的时间间隔来发现

```bash
[scheduler]
discover_hosts_in_cells_interval = 300
```

# 创建实例时的注意事项

在创建实例时，如果你是在虚拟机上部署openstack而不是物理机，很有可能无法创建虚拟机，原因是你的虚拟机不支持virtio，解决办法是给所有镜像设置一个vga的属性

假设我的openstack镜像为centos9

```bash
openstack image set --property hw_video_model='vga' centos9
```