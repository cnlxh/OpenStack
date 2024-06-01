```text
作者：李晓辉

联系方式：

微信：Lxh_Chat

QQ：939958092

邮箱：xiaohui_li@foxmail.com
```

# 将OpenStack和Ceph集成

|角色|IP|
|-|-|
|openstack控制节点|192.168.8.10|
|openstack计算节点|192.168.8.20|
|ceph|192.168.8.21|

# 准备ssh互信

准备从ceph管理节点到openstack的互信

这一步在ceph上执行

```bash
ssh-keygen -N '' -f /root/.ssh/id_rsa
ssh-copy-id root@192.168.8.10
ssh-copy-id root@192.168.8.20
```

# 创建Ceph池

ceph节点执行

```bash
ceph osd pool create volumes
ceph osd pool create images
ceph osd pool create backups
ceph osd pool create vms
```

初始化rbd功能

```bash
rbd pool init volumes
rbd pool init images
rbd pool init backups
rbd pool init vms
```

# 安装ceph客户端

在所有的openstack节点执行

```bash
yum install epel-release -y
yum install python-rbd ceph-common -y
```


# 准备ceph配置文件

在ceph节点执行，为所有openstack节点准备ceph配置文件

```bash
ssh 192.168.8.10 tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf
ssh 192.168.8.20 tee /etc/ceph/ceph.conf < /etc/ceph/ceph.conf

```

# 准备openstack用户

在ceph节点执行

```bash
ceph auth get-or-create client.glance mon 'profile rbd' osd 'profile rbd pool=images' mgr 'profile rbd pool=images'
ceph auth get-or-create client.cinder mon 'profile rbd' osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd-read-only pool=images' mgr 'profile rbd pool=volumes, profile rbd pool=vms'
ceph auth get-or-create client.cinder-backup mon 'profile rbd' osd 'profile rbd pool=backups' mgr 'profile rbd pool=backups'
```

为glance和cinder准备keyring

```bash
ceph auth get-or-create client.glance | ssh root@192.168.8.10 tee /etc/ceph/ceph.client.glance.keyring
ssh root@192.168.8.10 chown glance:glance /etc/ceph/ceph.client.glance.keyring
ceph auth get-or-create client.cinder | ssh root@192.168.8.10 tee /etc/ceph/ceph.client.cinder.keyring
ssh root@192.168.8.10 chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring
ceph auth get-or-create client.cinder-backup | ssh root@192.168.8.10 tee /etc/ceph/ceph.client.cinder-backup.keyring
ssh root@192.168.8.10 chown cinder:cinder /etc/ceph/ceph.client.cinder-backup.keyring
```

为nova节点准备keyring

```bash
ceph auth get-or-create client.cinder | ssh root@192.168.8.20 tee /etc/ceph/ceph.client.cinder.keyring
```

# 为libvirt准备密钥

在从cinder挂载卷时，libvirt进程需要它来访问集群

先创建一个临时密钥副本，这将在计算节点root家目录生成

```bash
ceph auth get-key client.cinder | ssh root@192.168.8.20 tee client.cinder.key
```

生成uuid

这一步在openstack 计算节点执行

```bash
[root@node ~]# uuidgen
e4355171-b66e-4036-989f-f6af496797a6
````
注意替换下面的UUID为刚生成的

```bash
cat > secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>e4355171-b66e-4036-989f-f6af496797a6</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF
```

```bash
[root@node ~]# virsh secret-define --file secret.xml
Secret e4355171-b66e-4036-989f-f6af496797a6 created
```

```bash
virsh secret-set-value --secret e4355171-b66e-4036-989f-f6af496797a6 --file client.cinder.key && rm client.cinder.key secret.xml
```

# 集成glance服务

在openstack控制节点(glance节点)执行

```bash
yum install crudini -y
```

```bash
crudini --set /etc/glance/glance-api.conf DEFAULT show_image_direct_url True
crudini --set /etc/glance/glance-api.conf DEFAULT enabled_backends rbd:rbd
crudini --set /etc/glance/glance-api.conf glance_store default_backend rbd
crudini --set /etc/glance/glance-api.conf rbd rbd_store_pool images
crudini --set /etc/glance/glance-api.conf rbd rbd_store_user glance
crudini --set /etc/glance/glance-api.conf rbd rbd_store_ceph_conf /etc/ceph/ceph.conf
crudini --set /etc/glance/glance-api.conf rbd rbd_store_chunk_size 8
crudini --set /etc/glance/glance-api.conf paste_deploy flavor keystone

```
或

```ini
[DEFAULT]
enabled_backends = rbd:rbd
show_image_direct_url = True
[glance_store]
default_backend = rbd
[rbd]
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
[paste_deploy]
flavor = keystone
```

```bash
systemctl restart openstack-glance-api
```

# 测试glance集成

创建一个镜像，并查看ceph中是否有数据

openstack控制节点执行

目前没有数据

```bash
[root@controller ~]# rados -p images ls --id glance
rbd_info
```

可以看到我们创建了一个镜像之后，ceph的池中就有数据了，说明集成glance成功

```bash
[root@controller ~]# source adminrc.sh
[root@controller ~]# openstack image create --disk-format raw --file cirros-0.6.2-x86_64-disk.img cirros
+------------------+--------------------------------------------------------------------------------------------------------------------------------------------+
| Field            | Value                                                                                                                                      |
+------------------+--------------------------------------------------------------------------------------------------------------------------------------------+
| container_format | bare                                                                                                                                       |
| created_at       | 2024-05-28T01:24:42Z                                                                                                                       |
| disk_format      | raw                                                                                                                                        |
| file             | /v2/images/cea1e1be-f56f-4d71-ab2a-12fae4a53b39/file                                                                                       |
| id               | cea1e1be-f56f-4d71-ab2a-12fae4a53b39                                                                                                       |
| min_disk         | 0                                                                                                                                          |
| min_ram          | 0                                                                                                                                          |
| name             | cirros                                                                                                                                     |
| owner            | c223c0bf51ab40fd94c3983ca5d04cc5                                                                                                           |
| properties       | os_hidden='False', owner_specified.openstack.md5='', owner_specified.openstack.object='images/cirros', owner_specified.openstack.sha256='' |
| protected        | False                                                                                                                                      |
| schema           | /v2/schemas/image                                                                                                                          |
| status           | queued                                                                                                                                     |
| tags             |                                                                                                                                            |
| updated_at       | 2024-05-28T01:24:42Z                                                                                                                       |
| visibility       | shared                                                                                                                                     |
+------------------+--------------------------------------------------------------------------------------------------------------------------------------------+
[root@controller ~]# rados -p images ls --id glance
rbd_data.ad5a976f67a9.0000000000000002
rbd_data.ad5a976f67a9.0000000000000000
rbd_header.ad5a976f67a9
rbd_directory
rbd_info
rbd_object_map.ad5a976f67a9
rbd_id.cea1e1be-f56f-4d71-ab2a-12fae4a53b39
rbd_object_map.ad5a976f67a9.0000000000000004
rbd_data.ad5a976f67a9.0000000000000001
```

# 集成cinder服务

在openstack控制节点(cinder节点)执行

```bash
crudini --set /etc/cinder/cinder.conf DEFAULT enabled_backends ceph
crudini --set /etc/cinder/cinder.conf DEFAULT glance_api_version 2
crudini --set /etc/cinder/cinder.conf ceph volume_driver cinder.volume.drivers.rbd.RBDDriver
crudini --set /etc/cinder/cinder.conf ceph volume_backend_name ceph
crudini --set /etc/cinder/cinder.conf ceph rbd_pool volumes
crudini --set /etc/cinder/cinder.conf ceph rbd_ceph_conf /etc/ceph/ceph.conf
crudini --set /etc/cinder/cinder.conf ceph rbd_flatten_volume_from_snapshot false
crudini --set /etc/cinder/cinder.conf ceph rbd_max_clone_depth 5
crudini --set /etc/cinder/cinder.conf ceph rbd_store_chunk_size 4
crudini --set /etc/cinder/cinder.conf ceph rados_connect_timeout -1
crudini --set /etc/cinder/cinder.conf ceph rbd_user cinder
crudini --set /etc/cinder/cinder.conf ceph rbd_secret_uuid e4355171-b66e-4036-989f-f6af496797a6 # 这里是前面创uuidgen的UUID

```

或

```ini
[DEFAULT]
enabled_backends = ceph
glance_api_version = 2

[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
rbd_user = cinder
rbd_secret_uuid = e4355171-b66e-4036-989f-f6af496797a6 # 这里是前面创uuidgen的UUID

```

启动服务

```bash
systemctl restart openstack-cinder-api.service openstack-cinder-scheduler.service openstack-cinder-volume.service
```

# 测试cinder集成

现在volumes池是空的

```bash
[root@controller ~]# rados -p volumes ls --id cinder
rbd_info
```

创建一个卷看看是否新增数据

```bash
[root@controller ~]# openstack volume create --size 5 testcinder
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2024-05-28T01:50:33.615859           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 3628cfe5-25e1-4e03-bf8f-59256cbaf8b5 |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | testcinder                           |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 5                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | __DEFAULT__                          |
| updated_at          | None                                 |
| user_id             | 914fc3ad0a434009b59d5dc31115bd03     |
+---------------------+--------------------------------------+
[root@controller ~]# rados -p volumes ls --id cinder
rbd_directory
rbd_object_map.aee97e58d62
rbd_info
rbd_id.volume-3628cfe5-25e1-4e03-bf8f-59256cbaf8b5
rbd_header.aee97e58d62
```

数据正常新增，cinder集成成功

# 测试cinder backup集成

在openstack 控制节点执行

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

# 集成nova


在openstack计算节点(nova节点)执行

执行以下命令向配置文件添加client章节

```bash
cat >> /etc/ceph/ceph.conf <<-'EOF'
[client]
rbd cache = true
rbd cache writethrough until flush = true
rbd concurrent management ops = 20
admin socket = /var/run/ceph/guests/$cluster-$type.$id.$pid.$cctid.asok
log file = /var/log/ceph/qemu-guest-$pid.log
EOF
```

为 admin 套接字和日志文件创建新目录，并将目录权限更改为使用 qemu 用户和 libvirtd 组

```bash
mkdir -p /var/run/ceph/guests/ /var/log/ceph/
chown qemu:libvirt /var/run/ceph/guests /var/log/ceph/
```

在每个 Nova 节点上，编辑 /etc/nova/nova.conf 文件。在 [libvirt] 部分下配置以下设置

```bash
crudini --set /etc/nova/nova.conf libvirt images_type rbd
crudini --set /etc/nova/nova.conf libvirt images_rbd_pool vms
crudini --set /etc/nova/nova.conf libvirt images_rbd_ceph_conf /etc/ceph/ceph.conf
crudini --set /etc/nova/nova.conf libvirt rbd_user cinder
crudini --set /etc/nova/nova.conf libvirt rbd_secret_uuid e4355171-b66e-4036-989f-f6af496797a6 # 这里是前面创uuidgen的UUID
crudini --set /etc/nova/nova.conf libvirt disk_cachemodes "network=writeback"
crudini --set /etc/nova/nova.conf libvirt inject_password false
crudini --set /etc/nova/nova.conf libvirt inject_key false
crudini --set /etc/nova/nova.conf libvirt inject_partition -2
crudini --set /etc/nova/nova.conf libvirt live_migration_flag "VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE,VIR_MIGRATE_PERSIST_DEST,VIR_MIGRATE_TUNNELLED"
crudini --set /etc/nova/nova.conf libvirt hw_disk_discard unmap

```

或

```ini
[libvirt]
images_type = rbd
images_rbd_pool = vms
images_rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
rbd_secret_uuid = 4b5fd580-360c-4f8c-abb5-c83bb9a3f964
disk_cachemodes="network=writeback"
inject_password = false
inject_key = false
inject_partition = -2
live_migration_flag="VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE,VIR_MIGRATE_PERSIST_DEST,VIR_MIGRATE_TUNNELLED"
hw_disk_discard = unmap
```

重启服务

```bash
systemctl restart openstack-nova-compute
```

# 测试nova集成

创建一个服务器

```bash
[root@controller ~]# openstack flavor list
+--------------------------------------+---------+------+------+-----------+-------+-----------+
| ID                                   | Name    |  RAM | Disk | Ephemeral | VCPUs | Is Public |
+--------------------------------------+---------+------+------+-----------+-------+-----------+
| c844c9c6-6560-4045-8409-69930908df49 | flavor1 | 1024 |   10 |         0 |     1 | True      |
+--------------------------------------+---------+------+------+-----------+-------+-----------+
[root@controller ~]# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| cea1e1be-f56f-4d71-ab2a-12fae4a53b39 | cirros | active |
| 429e1184-7a08-4a36-ab25-6d233950f12b | cirros | active |
+--------------------------------------+--------+--------+
[root@controller ~]# openstack network list
+--------------------------------------+----------+--------------------------------------+
| ID                                   | Name     | Subnets                              |
+--------------------------------------+----------+--------------------------------------+
| 955e4b76-1906-4079-b796-3e68b274d732 | external | d457a064-73da-46e3-912c-2ee01e8f1481 |
| a3d5942b-9dd2-4c44-956a-c1e7b1ed558f | internal | cdb54869-a48a-47a4-ba76-f066dc90e09c |
+--------------------------------------+----------+--------------------------------------+
[root@controller ~]# openstack server create --flavor c844c9c6-6560-4045-8409-69930908df49 --image cea1e1be-f56f-4d71-ab2a-12fae4a53b39 --nic net-id=a3d5942b-9dd2-4c44-956a-c1e7b1ed558f testeserver
+-------------------------------------+------------------------------------------------+
| Field                               | Value                                          |
+-------------------------------------+------------------------------------------------+
| OS-DCF:diskConfig                   | MANUAL                                         |
| OS-EXT-AZ:availability_zone         |                                                |
| OS-EXT-SRV-ATTR:host                | None                                           |
| OS-EXT-SRV-ATTR:hypervisor_hostname | None                                           |
| OS-EXT-SRV-ATTR:instance_name       |                                                |
| OS-EXT-STS:power_state              | NOSTATE                                        |
| OS-EXT-STS:task_state               | scheduling                                     |
| OS-EXT-STS:vm_state                 | building                                       |
| OS-SRV-USG:launched_at              | None                                           |
| OS-SRV-USG:terminated_at            | None                                           |
| accessIPv4                          |                                                |
| accessIPv6                          |                                                |
| addresses                           |                                                |
| adminPass                           | hs894qZPZJcf                                   |
| config_drive                        |                                                |
| created                             | 2024-05-28T03:10:24Z                           |
| flavor                              | flavor1 (c844c9c6-6560-4045-8409-69930908df49) |
| hostId                              |                                                |
| id                                  | c638f1e9-f211-49e0-9fef-4ad441c7c544           |
| image                               | cirros (cea1e1be-f56f-4d71-ab2a-12fae4a53b39)  |
| key_name                            | None                                           |
| name                                | testeserver                                    |
| progress                            | 0                                              |
| project_id                          | c223c0bf51ab40fd94c3983ca5d04cc5               |
| properties                          |                                                |
| security_groups                     | name='default'                                 |
| status                              | BUILD                                          |
| updated                             | 2024-05-28T03:10:24Z                           |
| user_id                             | 914fc3ad0a434009b59d5dc31115bd03               |
| volumes_attached                    |                                                |
+-------------------------------------+------------------------------------------------+

```

看看vms池有没有数据

```bash
[root@controller ~]# rados -p vms ls --id cinder
rbd_id.c638f1e9-f211-49e0-9fef-4ad441c7c544_disk
rbd_directory
rbd_header.b16b7863d482
rbd_children
rbd_info
rbd_header.6e4fde594567
rbd_object_map.b16b7863d482
rbd_trash
rbd_object_map.6e4fde594567
rbd_id.ff4d631f-ff23-4ccd-868a-ce394ec073ab_disk
```

正常新增数据，nova集成成功

