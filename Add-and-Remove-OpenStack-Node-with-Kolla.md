```textile
作者：李晓辉

微信：Lxh_Chat

邮箱：939958092@qq.com
```

# 准备hosts文件

我这里新增了control4和compute3，预计新增一个control和一个compute

```bash
cat > /etc/hosts <<EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.50.10 kolla
172.16.50.101 control1
172.16.50.102 control2
172.16.50.103 control3
172.16.50.106 control4
172.16.50.104 compute1
172.16.50.105 compute2
172.16.50.107 compute3
172.16.50.253 registry.xiaohui.cn
EOF
```

# 时间同步

```bash
for server in control4 compute3 ; do sshpass -p 1 ssh -A -g -o StrictHostKeyChecking=no root@$server 'dnf install chrony -y && systemctl enable chronyd --now';done
```

# 添加控制节点

## 为Cinder创建VG

这一步仅限于稍后将cinder后端设置为lvm时才需要做

由于只在control节点规划了cinder，所以这个只需要在control4上完成

```bash
sshpass -p 1 ssh -A -g -o StrictHostKeyChecking=no root@control4 vgcreate cinder-volumes /dev/sdb
```

## 修改清单文件

在control中新增了control4

```bash
[control]
control1 ansible_become=true
control2 ansible_become=true
control3 ansible_become=true
control4 ansible_become=true
```

## 准备先决条件

在kolla上执行

```bash
sshpass -p 1 ssh-copy-id -o StrictHostKeyChecking=no root@control4
kolla-ansible bootstrap-servers -i ./multinode --limit control4
```

## 下载镜像

```bash
kolla-ansible -i ./multinode pull --limit control4
```

## 将容器部署到节点

这里需要注意，不能只--limit control4，要把所有control都包含在内，如果怕都重启造成业务中断，可以-e kolla_serial=30%

```bash
kolla-ansible -i ./multinode deploy --limit control
```

# 添加计算节点

## 修改清单文件

在compute中新增了control3

```bash
[compute]
compute1 ansible_become=true
compute2 ansible_become=true
compute3 ansible_become=true
```

## 准备先决条件

在kolla上执行

```bash
sshpass -p 1 ssh-copy-id -o StrictHostKeyChecking=no root@compute3 
kolla-ansible bootstrap-servers -i ./multinode --limit compute3 
```

## 下载镜像

```bash
kolla-ansible -i ./multinode pull --limit compute3 
```

## 将容器部署到节点

```bash
kolla-ansible -i ./multinode deploy --limit compute3 
```

## 测试迁移

```bash
openstack server migrate <server> --live-migration --host <target host>
```

# 删除控制节点

不要轻易删除控制节点，哪怕要删除，也要确保删除后还有足够多的投票者，这是为了高可用性，比如3节点的控制节点，就最多只能删除一个

## 禁用资源

在删除节点之前，推荐删除其上的资源，这里举例删除网络agent

我打算删除control2

禁用L3 agent

```bash
l3_id=$(openstack network agent list --host control2 --agent-type l3 -f value -c ID)
target_l3_id=$(openstack network agent list --host control2 --agent-type l3 -f value -c ID)
openstack router list --agent $l3_id -f value -c ID | while read router; do 
openstack network agent remove router $l3_id $router --l3 
openstack network agent add router $target_l3_id $router --l3 
done
openstack network agent set $l3_id --disable
```

禁用DHCP agent

```bash
dhcp_id=$(openstack network agent list --host control2 --agent-type dhcp -f value -c ID)
target_dhcp_id=$(openstack network agent list --host <target host> --agent-type dhcp -f value -c ID)
openstack network list --agent $dhcp_id -f value -c ID | while read network; do
openstack network agent remove network $dhcp_id $network --dhcp
openstack network agent add network $target_dhcp_id $network --dhcp
done
```

## 停止服务

```bash
kolla-ansible -i ./multinode stop --yes-i-really-really-mean-it --limit control2
```

## 从主机清单删除

已经没有control2

```bash
[control]
control1 ansible_become=true
control3 ansible_become=true
control4 ansible_become=true
```

## 重新配置controller

```bash
kolla-ansible -i ./multinode deploy --limit control
```

## 清理服务

```bash
openstack network agent list --host control2 -f value -c ID | while read id; do
openstack network agent delete $id
done
openstack compute service list --os-compute-api-version 2.53 --host control2 -f value -c ID | while read id; do
openstack compute service delete --os-compute-api-version 2.53 $id
done
```

# 删除计算节点

删除计算节点之前，要考虑删除后，剩下的资源是否还有容量承载现有业务，还需要将其上的实例迁移至其他节点上

我打算删除compute2

## 禁止新的调度

```bash
openstack compute service set compute2 nova-compute --disable
```

## 迁移实例到其他节点

```bash
openstack server list --all-projects --host compute2 -f value -c ID | while read server; do
openstack server migrate --block-migration --live-migration --os-compute-api-version 2.30 $server
done
```

## 确认迁移情况

```bash
openstack server list --all-projects
```

## 停止服务

```bash
 kolla-ansible -i ./multinode stop --yes-i-really-really-mean-it --limit compute2
```

## 从主机清单删除

已经没有compute2

```bash
[compute]
compute1 ansible_become=true
compute3 ansible_become=true
```

## 清理服务

```bash
openstack network agent list --host compute2 -f value -c ID | while read id; do
openstack network agent delete $id
done
openstack compute service list --os-compute-api-version 2.53 --host compute2 -f value -c ID | while read id; do
openstack compute service delete --os-compute-api-version 2.53 $id
done
```
