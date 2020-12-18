# Kubernetes存储之搭建NFS

## NFS工作原理简介

CentOS 7 版本启动 NFS SERVER 之前,首先要启动 RPC 服务,否则 NFS SERVER 就无法向 RPC 服务注册.

另外,如果 RPC 服务重新启动,原来已经注册好的NFS端口数据就会丢失,因此,此时 RPC 服务管理的NFS程序也需要重新启动以重新向RPC注册.

说明:一般修改NFS配置文件后,是不需要重启NFS的,直接在命令行执行 systemctl reload nfs.service 「针对CentOS 7.x」 或 exportfs -rv 即可使修改的 /etc/exports 生效.

![nfs01](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/nfs01.png)

注意:一台机器不要同时做 NFS 的服务端和 NFS 的客户端.如果同时作了 NFS 的服务端和客户端,那么在关机的时候,会一直夯住,可能十分钟之后甚至更久才能关闭成功.

## 安装NFS和RPC服务(服务端和客户端均安装)

```shell
# 安装NFS和RPC
[root@nfs ~]# yum install nfs-utils rpcbind -y
[root@nfs ~]# rpm -qa nfs-utils rpcbind
rpcbind-0.2.0-49.el7.x86_64
nfs-utils-1.3.0-0.68.el7.x86_64
[root@nfs ~]# rpm -qa nfs-utils rpcbind
rpcbind-0.2.0-49.el7.x86_64
nfs-utils-1.3.0-0.68.el7.x86_64
# 安装成功后自动创建了三个用户
[root@nfs ~]# tail /etc/passwd
...
rpc:x:32:32:Rpcbind Daemon:/var/lib/rpcbind:/sbin/nologin
rpcuser:x:29:29:RPC Service User:/var/lib/nfs:/sbin/nologin
nfsnobody:x:65534:65534:Anonymous NFS User:/var/lib/nfs:/sbin/nologin
```

## 配置NFS服务端

```shell
[root@nfs ~]# mkdir /data
[root@nfs ~]# chown -R nfsnobody.nfsnobody /data/
[root@nfs ~]# ll -d /data/
drwxr-xr-x. 2 nfsnobody nfsnobody 6 Dec 16 14:14 /data/
# 先确认IP所在网段
[root@nfs ~]# ifconfig | grep -A 8 ens | grep -w inet
        inet 192.168.43.136  netmask 255.255.255.0  broadcast 192.168.43.255
[root@nfs ~]# vi /etc/exports
/data   192.168.43.0/24(rw,sync)
# 启动查看rpcbind
[root@nfs ~]# systemctl start rpcbind.service
[root@nfs ~]# ps -ef | grep nfs
root       2097      2  0 14:24 ?        00:00:00 [nfsd4_callbacks]
root       2103      2  0 14:24 ?        00:00:00 [nfsd]
root       2104      2  0 14:24 ?        00:00:00 [nfsd]
root       2105      2  0 14:24 ?        00:00:00 [nfsd]
root       2106      2  0 14:24 ?        00:00:00 [nfsd]
root       2107      2  0 14:24 ?        00:00:00 [nfsd]
root       2108      2  0 14:24 ?        00:00:00 [nfsd]
root       2109      2  0 14:24 ?        00:00:00 [nfsd]
root       2110      2  0 14:24 ?        00:00:00 [nfsd]
[root@nfs ~]# netstat -pantu | grep rpc
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      2035/rpcbind        
tcp6       0      0 :::111                  :::*                    LISTEN      2035/rpcbind        
udp        0      0 0.0.0.0:938             0.0.0.0:*                           2035/rpcbind        
udp        0      0 0.0.0.0:111             0.0.0.0:*                           2035/rpcbind        
udp6       0      0 :::938                  :::*                                2035/rpcbind        
udp6       0      0 :::111                  :::*                                2035/rpcbind        
[root@nfs ~]# rpcinfo -p localhost
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
# 启动NFS
[root@nfs ~]# systemctl start nfs.service
[root@nfs ~]# rpcinfo -p localhost
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
    100024    1   udp  56546  status
    100024    1   tcp  56040  status
    100005    1   udp  20048  mountd
    100005    1   tcp  20048  mountd
    100005    2   udp  20048  mountd
    100005    2   tcp  20048  mountd
    100005    3   udp  20048  mountd
    100005    3   tcp  20048  mountd
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100227    3   tcp   2049  nfs_acl
    100003    3   udp   2049  nfs
    100003    4   udp   2049  nfs
    100227    3   udp   2049  nfs_acl
    100021    1   udp  40602  nlockmgr
    100021    3   udp  40602  nlockmgr
    100021    4   udp  40602  nlockmgr
    100021    1   tcp  45707  nlockmgr
    100021    3   tcp  45707  nlockmgr
    100021    4   tcp  45707  nlockmgr
# 加入开机启动项
[root@nfs ~]# systemctl enable rpcbind.service
[root@nfs ~]# systemctl enable nfs.service
# 确认两个服务的状态
[root@nfs ~]# systemctl status rpcbind.service
● rpcbind.service - RPC bind service
   Loaded: loaded (/usr/lib/systemd/system/rpcbind.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2020-12-16 14:18:19 CST; 7min ago
 Main PID: 2035 (rpcbind)
   CGroup: /system.slice/rpcbind.service
           └─2035 /sbin/rpcbind -w

Dec 16 14:18:18 nfs systemd[1]: Starting RPC bind service...
Dec 16 14:18:19 nfs systemd[1]: Started RPC bind service.
[root@nfs ~]# systemctl status nfs.service
● nfs-server.service - NFS server and services
   Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; vendor preset: disabled)
  Drop-In: /run/systemd/generator/nfs-server.service.d
           └─order-with-mounts.conf
   Active: active (exited) since Wed 2020-12-16 14:24:06 CST; 1min 33s ago
 Main PID: 2096 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/nfs-server.service

Dec 16 14:24:06 nfs systemd[1]: Starting NFS server and services...
Dec 16 14:24:06 nfs systemd[1]: Started NFS server and services.

# 查看nfs相关配置参数
[root@nfs ~]# cat /var/lib/nfs/etab
/data   192.168.43.0/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,no_all_squash,no_subtree_check,secure_locks,acl,no_pnfs,anonuid=65534,anongid=65534,sec=sys,rw,secure,root_squash,no_all_squash)
```

参数说明:

- ro:只读设置,这样 NFS 客户端只能读、不能写(默认设置);
- rw:读写设置,NFS 客户端可读写;
- sync:将数据同步写入磁盘中,效率低,但可以保证数据的一致性(默认设置);
- async:将数据先保存在内存缓冲区中,必要时才写入磁盘;如果服务器重新启动,这种行为可能会导致数据损坏,但效率高;
- root_squash:当客户端用 root 用户访问该共享文件夹时,将 root 用户映射成匿名用户(默认设置);
- no_root_squash:客户端的 root 用户不映射.这样客户端的 root 用户与服务端的 root 用户具有相同的访问权限,这可能会带来严重的安全影响.没有充分的理由,不应该指定此选项;
- all_squash:客户端所有普通用户及所属组都映射为匿名用户及匿名用户组;(推荐设置)
- no_all_squash:客户端所有普通用户及所属组不映射(默认设置);
- subtree_check:如果共享,如:/usr/bin之类的子目录时,强制NFS检查父目录的权限;
- no_subtree_check:即使共享 NFS 服务端的子目录时,nfs服务端也不检查其父目录的权限,这样可以提高效率(默认设置);
- secure:限制客户端只能从小于1024的tcp/ip端口连接nfs服务器(默认设置);
- insecure:允许客户端从大于1024的tcp/ip端口连接服务器;
- wdelay:检查是否有相关的写操作,如果有则将这些写操作一起执行,这样可以提高效率(默认设置);
- no_wdelay:若有写操作则立即执行,当使用async时,无需此设置;
- anonuid=xxx:将远程访问的所有用户主都映射为匿名用户主账户,并指定该匿名用户主为本地用户主(UID=xxx);
- anongid=xxx:将远程访问的所有用户组都映射为匿名用户组账户,并指定该匿名用户组为本地用户组(GID=xxx);

```shell
# 查看服务端NFS已经开放/data 192.168.43.136为服务端IP
[root@nfs ~]# showmount -e 192.168.43.136
Export list for 192.168.43.136:
/data 192.168.43.0/24

# 防火墙永久开放 nfs  rpc-bind  mountd 三个服务, 临时开放把--permanent去掉即可(重启失效)
# 若不开放, 客户端访问时会报错, 类似clnt_create: RPC: Port mapper failure - Unable to receive: errno 113 (No route to host)
# 重启后, showmount若报错"clnt_create: RPC: Program not registered", 可尝试先重启rpcbind再重启nfs
[root@nfs ~]# firewall-cmd --add-service=nfs --permanent
[root@nfs ~]# firewall-cmd --add-service=rpc-bind --permanent
[root@nfs ~]# firewall-cmd --add-service=mountd --permanent

```

## 配置NFS客户端

```shell
# 启动rpcbind
[root@k8s-node1 ~]# systemctl start rpcbind.service
# 查看服务端NFS服务共享信息, 192.168.43.136为服务端IP
# 如果报错no route, 检查是否开放服务
# 若配置更新还不能访问, 尝试重启服务端nfs和rpcbind即可
[root@k8s-node1 ~]# showmount -e 192.168.43.136
Export list for 192.168.43.136:
/data 192.168.43.0/24
# 挂载NFS并查看
[root@k8s-node1 ~]# mount -t nfs 192.168.43.136:/data /mnt
[root@k8s-node1 ~]# df -h | grep /mnt
Filesystem               Size  Used Avail Use% Mounted on
192.168.43.136:/data      37G  2.2G   35G   6% /mnt
# 推荐用下面命令查看挂载, 比较详细
[root@k8s-node1 ~]# cat /proc/mounts
192.168.43.136:/data /mnt nfs4 rw,relatime,vers=4.1,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.43.20,local_lock=none,addr=192.168.43.136 0 0
```

## 测试

客户端: k8s-master; k8s-node1
服务端: nfs

```shell
# 1.客户端创建文件夹或创建文件并且输入数据,在服务端查看
# 客户端
[root@k8s-node1 /]# cd /mnt/
[root@k8s-node1 mnt]# mkdir nfs_client
[root@k8s-node1 mnt]# ll
total 0
drwxr-xr-x 2 nfsnobody nfsnobody 6 Dec 16 15:14 nfs_client
-rw-r--r-- 1 nfsnobody nfsnobody 0 Dec 16 15:16 nfs_c.txt
[root@k8s-node1 mnt]# echo nfs_c >> nfs_c.txt
# 服务端
[root@nfs data]# ll
total 0
drwxr-xr-x. 2 nfsnobody nfsnobody 6 Dec 16 15:14 nfs_client
-rw-r--r--. 1 nfsnobody nfsnobody 0 Dec 16 15:16 nfs_c.txt
[root@nfs data]# cat nfs_c.txt 
nfs_c

# 2.服务端创建文件夹或创建文件并且输入数据,在客户端查看
# 服务端
[root@nfs data]# mkdir nfs_server
[root@nfs data]# touch nfs_s.txt
[root@nfs data]# echo nfs_s >> nfs_s.txt
# 客户端
[root@k8s-node1 mnt]# ll
total 8
drwxr-xr-x 2 nfsnobody nfsnobody 6 Dec 16 15:14 nfs_client
-rw-r--r-- 1 nfsnobody nfsnobody 6 Dec 16 15:39 nfs_c.txt
drwxr-xr-x 2 root      root      6 Dec 16 15:41 nfs_server
-rw-r--r-- 1 root      root      6 Dec 16 15:43 nfs_s.txt
[root@k8s-node1 mnt]# cat nfs_s.txt 
nfs_s

# 3.在客户端之间互相编辑文件, 查看是否生效
[root@k8s-master mnt]# echo "k8s-master" >> nfs_c.txt
[root@k8s-node1 mnt]# echo "k8s-node1" >> nfs_c.txt
[root@nfs data]# cat nfs_c.txt 
nfs_c
k8s-master
k8s-node1
```

## 是否加入开机自启动

如果客户端需要开机自动挂载NFS, 若是方式1, 必须保证 /etc/rc.d/rc.local 文件具有可执行权限，否则该脚本不会执行也不会生效。

```shell
# 开机自启动方式1, 挂载信息写入系统启动加载文件
[root@k8s-node1 mnt]# ll /etc/rc.local
lrwxrwxrwx. 1 root root 13 Oct 21 13:51 /etc/rc.local -> rc.d/rc.local
[root@k8s-node1 mnt]# vi /etc/rc.local 
#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
mount -t nfs 192.168.43.136:/data /mnt

# 开机自启动方式2, 写入挂载文件
[root@k8s-node1 mnt]# vi /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Wed Oct 21 01:34:22 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
192.168.43.136:/data    /mnt                    nfs     defaults        0 0
```

## 存在问题

加入了开机自启动，当重启 NFS 客户端机器时，如果此时 NFS 服务端机器已关机，或者网络存在问题(网络断连或IP变更)等等。使 NFS 客户端连接 NFS 服务端失败，那么此时会造成 NFS 客户端机器起不来的情况。

因此为了避免该情况发生，不建议机器开机自启动就挂载 NFS。

如果一台机器必须挂载 NFS，那么我们就做好监控。当该机器未挂载 NFS 时就告警给我们，然后我们去手动挂载。

当然如果实际环境中你们的 NFS 服务极其稳定，且几乎不再改变 NFS 服务端IP地址，那么此时你也可以加入开机自启动。

具体问题具体分析
