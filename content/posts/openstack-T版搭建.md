+++
title = 'openstack-T版搭建'
date = 2024-06-11T15:05:44+08:00
draft = false
categories = ["云计算"]
tags = ["openstack"]
+++

# 	（以下所有密码统一为 **feng**）

# 搭建前基础配置



## 搭建环境



| 节点       | eth0, nat 网络     | eth1, 仅主机模式 | vcpu | memory | disk1 | disk2 |
| ---------- | ---------------- | --------------- | ---- | ------ | ----- | ----- |
| controller | 192.168.10.10/24 | 10.0.0.10/24    | 4 核  | 6G     | 50G   | 无    |
| compute    | 192.168.10.20/24 | 10.0.0.20/24    | 4 核  | 4G     | 50G   | 50G   |



## 修改主机名

```bash
控制节点
hostnamectl  set-hostname controller

# 计算节点
hostnamectl set-hostname compute
```



## 修改 hots 文件

```bash
所有节点
vim /etc/hosts

192.168.10.10	controller
192.168.10.20	compute
```



## 配置网络关闭 selinux 和防火墙

```bash
控制节点
vim /etc/sysconfig/network-scripts/ifcfg-eth0

BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.10.10
NETMASK=255.255.255.0
GATEWAY=192.168.10.1
DNS=8.8.8.8

vi /etc/sysconfig/network-scripts/ifcfg-eth1

BOOTPROTO=static
ONBOOT=yes
IPADDR=10.0.0.10
NETMASK=255.255.255.0

# 重载配置文件
systemctl restart network

# 关闭 selinux
setenforce 0

vi /etc/selinux/config

SELINUX=disabled


# 关闭防火墙
systemctl disable firewalld --now ;systemctl disable NetworkManager --now


# 计算节点
vim /etc/sysconfig/network-scripts/ifcfg-eth0

BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.10.20
NETMASK=255.255.255.0
GATEWAY=192.168.10.1
DNS=8.8.8.8

vim /etc/sysconfig/network-scripts/ifcfg-eth1

BOOTPROTO=static
ONBOOT=yes
IPADDR=10.0.0.20
NETMASK=255.255.255.0

# 重载配置文件
systemctl restart network

# 关闭 selinux
setenforce 0

vi /etc/selinux/config

SELINUX=disabled


# 关闭防火墙
systemctl disable firewalld --now ;systemctl disable NetworkManager --now
```



 ## 配置 ntp 时间同步服务器

```bash
控制节点

##修改配置文件
vim /etc/chrony.conf

server ntp3.aliyun.com iburst

allow all					## 允许所有网段连接
local stratum 10 			## 允许十个用户同步
## 重启服务
systemctl restart chronyd.service


# 计算节点
vi /etc/chrony.conf

server controller iburst

## 重启服务
systemctl restart chronyd

chronyc sources -v  # 查看同步情况
```

## 安装 openstack 软件源

```bash
所有节点

yum install centos-release-openstack-train -y
yum upgrade -y

# 安装 openstack 客户端
yum install python-openstackclient openstack-selinux -y
```

## 安装并配置数据库

```bash
仅控制节点

## 安装数据库和 python 模块
yum install mariadb mariadb-server python2-PyMySQL -y

## 编辑 openstack 配置文件
vim /etc/my.cnf.d/openstack.cnf
[mysqld]
bind-address = 192.168.10.10
default-storage-engine = innodb
innodb-file-per-table = on
max-connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8

## 启动 sql 服务并设置开机自启
systemctl enable mariadb --now

## 初始化 mysql 配置
mysql_secure_installation 

Enter
是否需要设置密码 y  # 密码设置为 feng
y
n
移除 test y
移除 table y
```

## 配置消息队列

```bash
仅控制节点


## 安装消息队列
yum install rabbitmq-server -y

## 启动自启
systemctl enable rabbitmq-server --now

## 添加 openstack 用户
rabbitmqctl add_user openstack feng   

## 给 openstack 用户赋予权限
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

## 配置缓存服务

```bash
仅控制节点


## 安装缓存服务和 python 模块
yum install memcached python-memcached -y

## 编辑配置文件
vim /etc/sysconfig/memcached

CACHSIZE = 1024
OPTIONS = "-l 127.0.0.1,::1,controller"

## 启动自启
systemctl enable memcached --now


```





# keystone 配置



## 创库授权

```sql
# 连接数据库
mysql -uroot -pfeng

# 创建数据库
create database keystone;

# 创建用户并赋予权限
 GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'feng';

GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'feng';
```

##  keystone 安装和配置

```bash
安装 keystone
yum install openstack-keystone httpd mod_wsgi -y

# 编辑配置文件
vim /etc/keystone/keystone.conf

[database]
connection = mysql+pymysql://keystone:feng@controller/keystone

[token]
provider = fernet
```

## 同步数据库

```bash
 su -s /bin/sh -c "keystone-manage db_sync" keystone
```

## 创建验证令牌

```bash

# 创建令牌
 keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
 keystone-manage credential_setup --keystone-user keystone --keystone-group keystone


keystone-manage bootstrap --bootstrap-password feng \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

## apched 配置

```bash

# 配置 apched
vim /etc/httpd/conf/httpd.conf 

ServerName controller

# 创建软连接
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/

# 启动自启
systemctl enable httpd --now

# 创建环境变量
export OS_USERNAME=admin
export OS_PASSWORD=feng
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3

```



## 创建域、项目、用户和角色

```bash
创建域
openstack domain create --description "An Example Domain" example

# 创建项目
openstack project create --domain default  --description "Service Project" service
openstack project create --domain default --description "Demo Project" myproject

# 创建用户
openstack user create --domain default --password-prompt myuser
  
# 创建角色
openstack role create myrole

# 将角色添加到项目和用户
openstack role add --project myproject --user myuser myrole
```

##  验证操作

```bash
 unset OS_AUTH_URL OS_PASSWORD
 
 openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue
  
  openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name myproject --os-username myuser token issue
```

## 创建脚本

```bash

vim admin.sh

#!/bin/bash
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=feng
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2


vim demo.sh

#!/bin/bash
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=feng
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2

```

# glance 配置





##  创库授权

```bash
连接数据库
 mysql -u root -pfeng

# 创建数据库
 CREATE DATABASE glance;
 
#创建用户并配置权限
 GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'feng';
 GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%'  IDENTIFIED BY 'feng'; 
```

## 创建 glance 用户

```bash
获得 admin 用户 cli 的权限
source ~/admin.sh

# 创建 glance 用户
openstack user create --domain default --password-prompt glance

# 将 glance 用户添加到具有管理员权限的 admin 角色和 service 项目中
openstack role add  --project service --user glance admin
```



## 创建 glance 服务实体

```bash
openstack service create --name glance  --description  "openstack image" image
```

##  创建 glance 服务 API 端点

```bash
openstack endpoint create --region RegionOne  image public http://controller:9292
openstack endpoint create --region RegionOne  image internal http://controller:9292
openstack endpoint create --region RegionOne  image admin http://controller:9292
```

## 安装和配置

```bash
yum install openstack-glance -y

vim /etc/glance/glance-api.conf

[database]

connection = mysql+pymysql://glance:feng@controller/glance


[keystone_authtoken]

www_authenticate_uri  = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = feng

[paste_deploy]

flavor = keystone

[glance_store]

stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/


```

## 同步数据库并设置自启

```bash
su -s /bin/sh -c "glance-manage db_sync" glance

# 设置服务自启
systemctl enable openstack-glance-api.service --now
```



## 测试

```bash
glance image-create --name "cirros" --file cirros-0.4.0-x86_64-disk.img \
--disk-format qcow2 --container-format bare --visibility public


glance image-list
```

#  placement 配置

## 创库授权

```bash
连接数据库
mysql -u root -pfeng

# 创建 placement 数据库
CREATE DATABASE placement;

# 创建 placement 用户并赋予权限
 GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'feng';
 GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%'  IDENTIFIED BY 'feng';
  

```

## 创建 placement 用户

``` bash
# 创建placement用户
openstack user create --domain default --password-prompt placement # 密码统一为feng

# 将Placement用户添加到具有管理员角色的服务项目中
openstack role add --project service --user placement admin
```

## 创建 placement 服务实体

``` bash
openstack service create --name placement --description "Placement API" placement
```

## 创建 placement API 服务端点

```bash
openstack endpoint create --region RegionOne  placement public http://controller:8778
openstack endpoint create --region RegionOne  placement internal http://controller:8778
openstack endpoint create --region RegionOne  placement admin http://controller:8778
```

## 安装和配置

```bash
安装 placement
yum install openstack-placement-api -y

# 编辑配置文件
vim /etc/placement/placement.conf

[placement_database]
connection = mysql+pymysql://placement:feng@controller/placement


[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = feng

```

## 同步 placement 数据库

```bash
su -s /bin/sh -c "placement-manage db sync" placement
```

## 解决 BUG

```bash
apched 版本大于或等于 2.4 需要添加以下信息
vim /etc/httpd/conf.d/00-placement-api.conf

<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>

# 重启 apched
systemctl restart httpd


# 验证
placement-status upgrade check
```

# nova 配置

## 控制节点

### 创库授权

```bash
连接数据库
mysql -uroot -pfeng

# 创建数据库
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;

# 创建用户并授权
   GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'feng';
   GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'feng';
  
  GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'feng';
  GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'feng';
  
  GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'feng';
  GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'feng';
```



### 创建 nova 用户

```bash
获取 admin 凭证以获取配置权限
 source ~/admin.sh

# 创建 nova 用户
openstack user create --domain default --password-prompt nova

# 将 nova 用户加入 admin 角色
 openstack role add --project service --user nova admin

```

### 创建 nova 服务实体

```bash
openstack service create --name nova --description "OpenStack Compute" compute
```

### 创建 nova API 服务端点

```bash
openstack endpoint create --region RegionOne  compute public http://controller:8774/v2.1
openstack endpoint create --region RegionOne  compute internal http://controller:8774/v2.1
openstack endpoint create --region RegionOne  compute admin http://controller:8774/v2.1
```

### 安装和配置

```bash
yum install openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler -y

vim /etc/nova/nova.conf

[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:feng@controller:5672/
my_ip = 192.168.10.10
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
connection = mysql+pymysql://nova:feng@controller/nova_api


[database]
connection = mysql+pymysql://nova:feng@controller/nova


[api]
auth_strategy = keystone


[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = feng



[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip



[glance]
api_servers = http://controller:9292



[oslo_concurrency]
lock_path = /var/lib/nova/tmp



[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = feng
```

### 同步 nova 数据库

```bash
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
su -s /bin/sh -c "nova-manage db sync" nova

# 验证
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

### 启动服务

```bash
systemctl enable openstack-nova-api.service  openstack-nova-scheduler.service openstack-nova-conductor.service \
openstack-nova-novncproxy.service --now
```









## 计算节点

###  安装并配置

```bash
安装服务
yum install openstack-nova-compute -y

# 编辑配置文件
vim /etc/nova/nova.conf

[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:feng@controller
my_ip = 192.168.10.20
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver



[api]
auth_strategy = keystone


[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = feng



[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html


[glance]
api_servers = http://controller:9292


[oslo_concurrency]
lock_path = /var/lib/nova/tmp


[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = feng



# 启动服务并设置自启
systemctl enable libvirtd.service openstack-nova-compute.service --now
```

### 主机发现（控制节点配置）

```bash
控制节点配置

## 首先获取 admin 权限
source ~/admin.sh

# 发现新的计算机
 su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
 
 # 如果需要自动按时，需配置/etc/nova/nova.conf
 
 [scheduler]
 discover_hosts_in_cells_interval = 300 
 
 # 重启服务
 systemctl restart openstack-nova-api.service  openstack-nova-scheduler.service openstack-nova-conductor.service \
openstack-nova-novncproxy.service 
```



# neutron 配置





## 控制节点



### 创库授权

```bash
连接数据库
mysql -uroot -pfeng

# 创建数据库
CREATE DATABASE neutron;

# 授权
 GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'feng';
  GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'feng';
```

### 创建 neutron用户

```bash
source admin.sh

# 创建 neutron 用户
openstack user create --domain default --password-prompt neutron

# 添加 admin 角色到 neutron
openstack role add --project service --user neutron admin
```

### 创建 neutron 服务实体

```bash
openstack service create --name neutron --description "OpenStack Networking" network
```

### 创建 neutron API 端点

```bash
openstack endpoint create --region RegionOne  network public http://controller:9696

openstack endpoint create --region RegionOne  network internal http://controller:9696

openstack endpoint create --region RegionOne  network admin http://controller:9696
```

### 安装和配置

```bash
安装 neutron 及依赖
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y


yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch ebtables -y 

#编辑配置文件
vim /etc/neutron/neutron.conf

[database]
connection = mysql+pymysql://neutron:feng@controller/neutron


[DEFAULT]
core_plugin = ml2
service_plugins = router
transport_url = rabbit://openstack:feng@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true



[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = feng



[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = feng



[oslo_concurrency]
lock_path = /var/lib/neutron/tmp










vim /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
type_drivers = flat,vlan
tenant_network_types =
mechanism_drivers = linuxbridge
extension_drivers = port_security

[ml2_type_flat]
flat_networks = extnet

[securitygroup]
enable_ipset = true








vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[linux_bridge]
physical_interface_mappings = extnet:eth0

[vxlan]
enable_vxlan = false

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver




#===== ==== ==== ==== ==== ==== ==== ==== ==== ====#
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

modprobe br_netfilter
#===== ==== ==== ==== ==== ==== ==== ==== ==== ====#


vim /etc/neutron/dhcp_agent.ini

[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true

```



### 配置元数据代理

```bash
vim /etc/neutron/metadata_agent.ini

[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = feng
```



### 配置 nova

```bash
vim /etc/nova/nova.conf

[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = feng
service_metadata_proxy = true
metadata_proxy_shared_secret = feng


# 创建软连接
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini

```



### 同步数据库

```bash
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```



### 启动服务

```bash
重启 nova 服务
 systemctl restart openstack-nova-api.service

## 启动网络服务
systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
neutron-metadata-agent.service --now
```



 







## 计算节点



### 安装和配置

```bash
安装
yum install openstack-neutron-linuxbridge ebtables ipset -y

vim /etc/neutron/neutron.conf

[DEFAULT]
transport_url = rabbit://openstack:feng@controller
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = feng

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp




# 配置桥接代理
vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[linux_bridge]
physical_interface_mappings = extnet:eth0

[vxlan]
enable_vxlan = false

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

#===== ==== ==== ==== ==== ==== ==== ==== ====#
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

modprobe br_netfilter
#===== ==== ==== ==== ==== ==== ==== ==== ==== =#


#配置 nova 服务
vim /etc/nova/nova.conf

[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = feng
```



### 启动服务

```bash
重启计算服务
systemctl restart openstack-nova-compute.service

# 启动服务
systemctl enable neutron-linuxbridge-agent.service --now
```



## 验证操作

```bash
openstack extension list --network

openstack network agent list

# 创建网络
openstack network create  --share --external \
  --provider-physical-network extnet \
  --provider-network-type flat flat-extnet
  
# 创建子网
openstack subnet create --network flat-extnet \
  --allocation-pool start=192.168.10.50,end=192.168.10.100 \
  --dns-nameserver 8.8.8.8 --gateway 192.168.10.1 \
  --subnet-range 192.168.10.0/24 flat-subnet


# 创建实例
openstack flavor create --id 0 --vcpus 1 --ram 64 --disk 1 m1.nano


# 生成密钥对并添加公钥
ssh-keygen -q -N ""
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey

# 验证密钥对是否添加
openstack keypair list


# 添加安全组规则
openstack security group rule create --proto icmp default
openstack security group rule create --proto tcp --dst-port 22 default

# 启动实例
openstack server create --flavor m1.nano --image cirros \
  --nic net-id=a06e44f3-0df0-4ac9-8d47-9d914f6e0b7c --security-group default \
  --key-name mykey vm1

# 查看实例状态
openstack server list

+--------------------------------------+------+--------+---------------------------+--------+---------+
| ID                                   | Name | Status | Networks                  | Image  | Flavor  |
+--------------------------------------+------+--------+---------------------------+--------+---------+
| 01e56cad-5c3d-47bf-b4ee-1e55878138ad | vm2  | ACTIVE | flat-extnet=192.168.10.86 | cirros | m1.nano |
| 4aea631a-5638-4639-9d26-59d9a4886864 | vm1  | ERROR  |                           | cirros | m1.nano |
+--------------------------------------+------+--------+---------------------------+--------+---------+

# vnc 访问
openstack console url show vm2

+-------+-------------------------------------------------------------------------------------------+
| Field | Value                                                                                     |
+-------+-------------------------------------------------------------------------------------------+
| type  | novnc                                                                                     |
| url   | http://controller:6080/vnc_auto.html?path=%3Ftoken%3D5cd92fec-30ff-4b6e-b9e4-4d442997bc8b |
+-------+-------------------------------------------------------------------------------------------+


[root@controller ~]# ssh cirros@192.168.10.86
The authenticity of host '192.168.10.86 (192.168.10.86)' can't be established.
ECDSA key fingerprint is SHA256:chcB+7jnb24JxgJEa6f6WMiYvqTp7f/5Ho+nWOfjBRw.
ECDSA key fingerprint is MD5:4e:cd:34:3b:ef:4b:3d:0f:ca:8d:c8:d6:25:1d:be:f6.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.10.86' (ECDSA) to the list of known hosts.
$ sudo su - root
#
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether fa:16:3e:a1:f8:02 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.86/24 brd 192.168.10.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fea1:f802/64 scope link
       valid_lft forever preferred_lft forever
# ping 192.168.10.10
PING 192.168.10.10 (192.168.10.10): 56 data bytes
64 bytes from 192.168.10.10: seq=0 ttl=64 time=0.647 ms
64 bytes from 192.168.10.10: seq=1 ttl=64 time=1.060 ms
^C
--- 192.168.10.10 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.647/0.853/1.060 ms
#

```











```bash
https://docs.openstack.org/ocata/config-reference/networking/samples/ml2_conf.ini.html
```

```bash
https://docs.openstack.org/ocata/config-reference/networking/samples/linuxbridge_agent.ini.html
```



# dashboard 配置



## 安装和配置

```bash
 # 安装软件包
 yum install openstack-dashboard -y 
 
vim /etc/openstack-dashboard/local_settings 

OPENSTACK_HOST = "controller"
 
 ALLOWED_HOSTS = ["*"]
 
 SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
}

TIME_ZONE = "Asia/Shanghai"


vim /etc/httpd/conf.d/openstack-dashboard.conf

WSGIApplicationGroup %{GLOBAL}


# 重启服务
systemctl restart httpd.service memcached.service/
```





