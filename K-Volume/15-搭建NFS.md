# Kubernetes存储之搭建NFS

2021-0802

## 概述

- 平台：VMware® Workstation 16 Pro
- nfs服务端IP：192.168.43.212 主机名为nfs-server
- nfs客户端IP：192.168.43.187 主机名为k8s-node02

CentOS-7版本启动NFS server之前，首先要启动RPC服务，完成NFS向RPC服务的注册。如果RPC服务重新启动，原来已经注册好的NFS端口数据就会丢失。因此，此时RPC服务管理的NFS程序也需要重新启动以重新向RPC注册。

说明：一般修改NFS配置文件后，是不需要重启NFS的，直接在命令行执行`systemctl reload nfs`或`exportfs -rv`即可使修改的`/etc/exports`生效。

![nfs01](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/nfs01.png)

注意：一台机器不要同时做NFS的服务端和NFS的客户端。如果同时作了服务端和客户端，那么在关机的时候，会一直卡住，可能十分钟之后甚至更久才能关闭成功。

## 安装NFS和RPC服务(服务端和客户端均安装)

```shell
# nfs-server和nfs-client均需要安装，此处以nfs-server为例
# 安装NFS和RPC
[root@nfs-server ~]# yum install nfs-utils rpcbind -y
# 查看
[root@nfs-server ~]# rpm -qa nfs-utils rpcbind
nfs-utils-1.3.0-0.68.el7.2.x86_64
rpcbind-0.2.0-49.el7.x86_64
# 安装成功后自动创建了三个相关用户
[root@nfs-server ~]# tail -n 3 /etc/passwd
rpc:x:32:32:Rpcbind Daemon:/var/lib/rpcbind:/sbin/nologin
rpcuser:x:29:29:RPC Service User:/var/lib/nfs:/sbin/nologin
nfsnobody:x:65534:65534:Anonymous NFS User:/var/lib/nfs:/sbin/nologin
```

## 配置NFS服务端

```shell
[root@nfs-server ~]# mkdir /data
[root@nfs-server ~]# chown -R nfsnobody.nfsnobody /data/
[root@nfs-server ~]# ll -d /data/
drwxr-xr-x 2 nfsnobody nfsnobody 6 Mar 16 14:29 /data/
# 配置nfs工作网段权限
[root@nfs-server ~]# vim /etc/exports
/data   192.168.43.0/24(rw,sync)
# 启动查看rpcbind
[root@nfs-server ~]# systemctl start rpcbind
[root@nfs-server ~]# rpcinfo -p localhost
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
[root@nfs-server ~]# netstat -lntp | grep rpc
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      1597/rpcbind        
tcp6       0      0 :::111                  :::*                    LISTEN      1597/rpcbind  
# 启动NFS
[root@nfs-server ~]# systemctl start nfs
[root@nfs-server ~]# ps -ef | grep nfs
root       1659      2  0 14:34 ?        00:00:00 [nfsd]
root       1660      2  0 14:34 ?        00:00:00 [nfsd]
root       1661      2  0 14:34 ?        00:00:00 [nfsd]
root       1662      2  0 14:34 ?        00:00:00 [nfsd]
root       1663      2  0 14:34 ?        00:00:00 [nfsd]
root       1664      2  0 14:34 ?        00:00:00 [nfsd]
root       1665      2  0 14:34 ?        00:00:00 [nfsd]
root       1666      2  0 14:34 ?        00:00:00 [nfsd]
[root@nfs-server ~]# netstat -lntp | grep rpc
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      1597/rpcbind        
tcp        0      0 0.0.0.0:20048           0.0.0.0:*               LISTEN      1651/rpc.mountd     
tcp        0      0 0.0.0.0:49275           0.0.0.0:*               LISTEN      1627/rpc.statd      
tcp6       0      0 :::52143                :::*                    LISTEN      1627/rpc.statd      
tcp6       0      0 :::111                  :::*                    LISTEN      1597/rpcbind        
tcp6       0      0 :::20048                :::*                    LISTEN      1651/rpc.mountd  
[root@nfs-server ~]# rpcinfo -p localhost
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
    100024    1   udp  47016  status
    100024    1   tcp  49275  status
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
    100227    3   udp   2049  nfs_acl
    100021    1   udp  52915  nlockmgr
    100021    3   udp  52915  nlockmgr
    100021    4   udp  52915  nlockmgr
    100021    1   tcp  43557  nlockmgr
    100021    3   tcp  43557  nlockmgr
    100021    4   tcp  43557  nlockmgr
# 加入开机启动项
[root@nfs-server ~]# systemctl enable nfs
[root@nfs-server ~]# systemctl enable rpcbind
# 确认两个服务的状态
[root@nfs-server ~]# systemctl status rpcbind
● rpcbind.service - RPC bind service
   Loaded: loaded (/usr/lib/systemd/system/rpcbind.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2022-03-16 14:31:15 CST; 6min ago
 Main PID: 1597 (rpcbind)
   CGroup: /system.slice/rpcbind.service
           └─1597 /sbin/rpcbind -w

Mar 16 14:31:15 cicd-dest systemd[1]: Starting RPC bind service...
Mar 16 14:31:15 cicd-dest systemd[1]: Started RPC bind service.
[root@nfs-server ~]# systemctl status nfs
● nfs-server.service - NFS server and services
   Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; vendor preset: disabled)
  Drop-In: /run/systemd/generator/nfs-server.service.d
           └─order-with-mounts.conf
   Active: active (exited) since Wed 2022-03-16 14:34:42 CST; 2min 58s ago
 Main PID: 1654 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/nfs-server.service

Mar 16 14:34:41 cicd-dest systemd[1]: Starting NFS server and services...
Mar 16 14:34:42 cicd-dest systemd[1]: Started NFS server and services.

# 查看nfs相关配置参数
[root@nfs-server ~]# cat /var/lib/nfs/etab
/data   192.168.43.0/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,no_all_squash,no_subtree_check,secure_locks,acl,no_pnfs,anonuid=65534,anongid=65534,sec=sys,rw,secure,root_squash,no_all_squash)
```

参数说明:

- ro：只读设置,这样 NFS 客户端只能读、不能写(默认设置)
- rw：读写设置,NFS 客户端可读写
- sync：将数据同步写入磁盘中,效率低,但可以保证数据的一致性(默认设置)
- async：将数据先保存在内存缓冲区中,必要时才写入磁盘;如果服务器重新启动,这种行为可能会导致数据损坏,但效率高
- root_squash：当客户端用 root 用户访问该共享文件夹时,将 root 用户映射成匿名用户(默认设置)
- no_root_squash：客户端的 root 用户不映射.这样客户端的 root 用户与服务端的 root 用户具有相同的访问权限,这可能会带来严重的安全影响.没有充分的理由,不应该指定此选项
- all_squash：客户端所有普通用户及所属组都映射为匿名用户及匿名用户组;(推荐设置)
- no_all_squash：客户端所有普通用户及所属组不映射(默认设置)
- subtree_check：如果共享,如:/usr/bin之类的子目录时,强制NFS检查父目录的权限
- no_subtree_check：即使共享 NFS 服务端的子目录时,nfs服务端也不检查其父目录的权限,这样可以提高效率(默认设置)
- secure：限制客户端只能从小于1024的tcp/ip端口连接nfs服务器(默认设置)
- insecure：允许客户端从大于1024的tcp/ip端口连接服务器
- wdelay：检查是否有相关的写操作,如果有则将这些写操作一起执行,这样可以提高效率(默认设置)
- no_wdelay：若有写操作则立即执行,当使用async时,无需此设置
- anonuid=xxx：将远程访问的所有用户主都映射为匿名用户主账户,并指定该匿名用户主为本地用户主(UID=xxx)
- anongid=xxx：将远程访问的所有用户组都映射为匿名用户组账户,并指定该匿名用户组为本地用户组(GID=xxx)

```shell
# 查看服务端NFS已经开放/data 192.168.43.212为服务端IP
[root@nfs-server ~]# showmount -e 192.168.43.212
Export list for 192.168.43.212:
/data 192.168.43.0/24

# 防火墙永久开放 nfs  rpc-bind  mountd 三个服务, 临时开放把--permanent去掉即可(重启失效)
# 若不开放, 客户端访问时会报错, 类似clnt_create: RPC: Port mapper failure - Unable to receive: errno 113 (No route to host)
# 重启后, showmount若报错"clnt_create: RPC: Program not registered", 可尝试先重启rpcbind再重启nfs
[root@nfs-server ~]# firewall-cmd --add-service=nfs --permanent
success
[root@nfs-server ~]# firewall-cmd --add-service=rpc-bind --permanent
success
[root@nfs-server ~]# firewall-cmd --add-service=mountd --permanent
success
# 重新加载防火墙规则使之生效
[root@nfs-server ~]#  firewall-cmd --reload
success
```

## 配置NFS客户端

```shell
# 启动rpcbind
[root@k8s-node02 ~]# systemctl start rpcbind
# 查看服务端NFS服务共享信息, 192.168.43.212为服务端IP
# 如果报错no route, 检查是否开放服务或是否reload防火墙
# 若配置更新还不能访问, 尝试重启服务端nfs和rpcbind即可
[root@k8s-node02 ~]# showmount -e 192.168.43.212
Export list for 192.168.43.212:
/data 192.168.43.0/24
# 挂载NFS并查看
[root@k8s-node02 ~]# mount -t nfs 192.168.43.212:/data /mnt
[root@k8s-node02 ~]# df -h | grep mnt
192.168.43.212:/data                  17G  1.8G   16G  11% /mnt
# 推荐用下面命令查看挂载, 比较详细
[root@k8s-node02 ~]# cat /proc/mounts | grep mnt
192.168.43.212:/data /mnt nfs4 rw,relatime,vers=4.1,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.43.187,local_lock=none,addr=192.168.43.212 0 0
```

## 测试

```shell
# 1.客户端创建文件夹或创建文件并且输入数据，在服务端查看同步结果
# 客户端
[root@k8s-node02 ~]# cd /mnt/
[root@k8s-node02 mnt]# echo "Message from nfs-client: k8s-node02" > client.log
[root@k8s-node02 mnt]# mkdir client
# 服务端
[root@nfs-server ~]# cd /data/
[root@nfs-server data]# ls
client  client.log
[root@nfs-server data]# cat client.log 
Message from nfs-client: k8s-node02

# 2.服务端创建文件夹或创建文件并且输入数据,在客户端查看
# 服务端
[root@nfs-server data]# mkdir server
[root@nfs-server data]# echo "Message from nfs-server: nfs-server" > server.log
# 客户端
[root@k8s-node02 mnt]# ls
client  client.log  server  server.log
[root@k8s-node02 mnt]# cat server.log 
Message from nfs-server: nfs-server
```

## 重启自动挂载

配置客户端重启自动挂载NFS，若是下面方式1，必须保证`/etc/rc.d/rc.local`文件具有可执行权限，否则该脚本不会执行也不会生效。推荐方式2。

```shell
# 开机自启动方式1, 挂载信息写入系统启动加载文件
[root@k8s-node02 mnt]# ll /etc/rc.local 
lrwxrwxrwx. 1 root root 13 Mar  5 15:12 /etc/rc.local -> rc.d/rc.local
[root@k8s-node02 mnt]# ll /etc/rc.d/rc.local 
-rw-r--r--. 1 root root 473 Jan 14 00:54 /etc/rc.d/rc.local
[root@k8s-node02 mnt]# chmod +x /etc/rc.d/rc.local 
[root@k8s-node02 mnt]# vim /etc/rc.local 
# 最后一行加入
mount -t nfs 192.168.43.212:/data /mnt

# 开机自启动方式2, 写入挂载文件（推荐）
[root@k8s-node02 mnt]# vim /etc/fstab 
# 最后一行加入
192.168.43.212:/data    /mnt                    nfs     defaults        0 0
```

## 问题分析

nfs相关服务加入了开机自启动，当重启NFS客户端机器后，如果此时NFS服务端机器已关机，或者网络存在问题(网络断连或IP变更)等等。使NFS客户端连接NFS服务端失败，那么此时会造成NFS客户端机器启动很慢（因为检查/etc/rc.local或/etc/fstab挂载无法找到目标服务端机器）的情况。为了避免该情况发生，不建议机器开机自启动就挂载NFS。

如果一台机器必须挂载 NFS，可以做监控。当该机器未挂载 NFS 时就告警，然后使用命令`mount -t nfs 192.168.43.212:/data /mnt`去手动挂载。

当然如果实际环境中NFS服务极其稳定，且几乎不再改变NFS服务端IP地址，那么此时也可以加入开机自启动。具体问题具体分析。
