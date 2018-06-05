
<p align='center'> <a href='https://github.com/alvinwancn' target="_blank"> <img src='https://github.com/AlvinWanCN/life-record/raw/master/images/etlucency.png' alt='Alvin Wan' width=200></a></p>


## cinder 块存储
#需要准备存储节点,可以使用LVM、NFS、分布式存储等

#本次安装以LVM、NFS为例


## 添加硬盘

这里我们添加了一块300G的硬盘sdb
```
[root@cinder-storage1 ~]# lsblk
NAME                MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                   8:0    0   20G  0 disk
├─sda1                8:1    0  500M  0 part /boot
├─sda2                8:2    0    1G  0 part [SWAP]
└─sda3                8:3    0 18.5G  0 part
  └─vg_root-lv_root 253:0    0 18.5G  0 lvm  /
sdb                   8:16   0  300G  0 dis
```


## 磁盘分区

### fdisk快速分区,新建2个30G分区

```
echo -e 'n\np\n1\n\n+30G\nw' | fdisk /dev/sdb
echo -e 'n\np\n2\n\n+30G\nw' | fdisk /dev/sdb

```

查看一下。
```
[root@cinder-storage1 ~]# fdisk -l /dev/sdb

Disk /dev/sdb: 322.1 GB, 322122547200 bytes, 629145600 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x1d3e73ea

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048    62916607    31457280   83  Linux
/dev/sdb2        62916608   125831167    31457280   83  Linux

```
### 格式化分区
```
mkfs.ext4 /dev/sdb1
mkfs.ext4 /dev/sdb2

mkdir -p /data

mount -t ext4 /dev/sdb1 /data
df -h|grep /dev/sdb1
```

### 开机挂载磁盘

```
echo "mount -t ext4 /dev/sdb1 /data" >>/etc/rc.d/rc.local
tail -1 /etc/rc.d/rc.local
chmod +x /etc/rc.d/rc.local
```

## 安装配置LVM，作为后端存储使用

```
yum install -y lvm2
systemctl enable lvm2-lvmetad.service
systemctl start lvm2-lvmetad.service
#创建LVM物理卷pv与卷组vg
pvcreate /dev/sdb2
vgcreate cinder_lvm01 /dev/sdb2
vgdisplay #查看vg
```


## 安装配置NFS服务，作为后端存储使用

```
useradd cinder
yum install nfs-utils rpcbind -y
mkdir -p /data/{cinder_nfs1,cinder_nfs2}
chown cinder:cinder /data/cinder_nfs1
chmod 777 /data/cinder_nfs1
#echo "/data/cinder_nfs1 *(rw,no_root_squash,sync)">/etc/exports
echo "/data/cinder_nfs1 *(rw,root_squash,sync,anonuid=165,anongid=165)">/etc/exports
exportfs -r
systemctl enable rpcbind nfs-server
systemctl restart rpcbind nfs-server
showmount -e localhost
```

## 安装配置Cinder

### 添加openstack的仓库

```
curl -fsSL https://raw.githubusercontent.com/AlvinWanCN/TechnologyCenter/master/linux/software/yum.repos.d/openstack_pick_centos7.repo > /etc/yum.repos.d/openstack_pick_centos7.repo
```

### 安装配置Cinder
```
yum install -y openstack-cinder targetcli python-keystone lvm2
cp /etc/cinder/cinder.conf{,.bak}
cp /etc/lvm/lvm.conf{,.bak}
```

### 配置LVM过滤，只接收上面配置的lvm设备/dev/sdb2

```
#在devices {  }部分添加 filter = [ "a/sdb2/", "r/.*/"]
sed -i '141a filter = [ "a/sdb2/", "r/.*/"]' /etc/lvm/lvm.conf  #在141行后添加
```



### NFS
```
echo 'cinder-storage1.alv.pub:/data/cinder_nfs1'>/etc/cinder/nfs_shares
chmod 640 /etc/cinder/nfs_shares
chown root:cinder /etc/cinder/nfs_shares
```

### 配置cinder
```
echo '
[DEFAULT]
auth_strategy = keystone
log_dir = /var/log/cinder
state_path = /var/lib/cinder
glance_api_servers = http://glance.alv.pub:9292
transport_url = rabbit://openstack:openstack@rabbitmq1.alv.pub
enabled_backends = lvm,nfs

[database]
connection = mysql+pymysql://cinder:cinder@maxscale.alv.pub/cinder

[keystone_authtoken]
auth_uri = http://keystone1.alv.pub:5000
auth_url = http://keystone1.alv.pub:35357
memcached_servers = keystone1.alv.pub:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = cinder

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp

[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
iscsi_helper = lioadm
iscsi_protocol = iscsi
volume_group = cinder_lvm01
iscsi_ip_address = 192.168.127.83
volumes_dir = $state_path/volumes
volume_backend_name = lvm01

[nfs]
volume_driver = cinder.volume.drivers.nfs.NfsDriver
nfs_shares_config = /etc/cinder/nfs_shares
nfs_mount_point_base = $state_path/mnt
volume_backend_name = nfs01
'>/etc/cinder/cinder.conf

```

```
chmod 640 /etc/cinder/cinder.conf
chgrp cinder /etc/cinder/cinder.conf
```

## #启动Cinder卷服务