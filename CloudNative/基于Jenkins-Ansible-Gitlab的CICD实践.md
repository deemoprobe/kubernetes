# 基于Jenkins-Ansible-Gitlab的CICD实践

提示：文中大部分图片设置了点击放大功能，细节信息可以放大图片查看，点击图片以外的空白区域即可返回阅读。

## 概要

CI/CD（Continuous Integration/Continuous Delivery）指持续集成和持续交付。

CI属于开发人员的自动化流程，成功的CI意味着：由仓库不同分支合并过来的应用代码会被定期构建、测试。持续集成中代码构建、单元测试和集成测试的自动化流程后，持续交付可自动将已验证的代码发布。持续交付的目标是拥有一个可随时部署到生产环境的代码库。

下图是敏捷开发（Agile Development）、CI、CD和DevOps几个开发模式所对应的周期：

![ADvsCICDvsDevOps](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/ADvsCICDvsDevOps.png)

本文中CICD流程概括如图：

![cicd](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/cicd.png)

## 环境说明

- 宿主机：Windows10 21H2
- 虚拟机平台：VMware® Workstation 16 Pro
- 操作用户：root
- IP分配：192.168.43.210-213

> 虚拟机安装CentOS-7系统配置静态IP以及常用工具的安装请参考另一篇博客：[LINUX之VMWARE WORKSTATION安装CENTOS-7](https://www.deemoprobe.com/standard/vmware-centos7/)

IP和主机域名分配：

| 主机名               |       IP       | 域名                   | 功能描述                      |
| -------------------- | :------------: | ---------------------- | ----------------------------- |
| cicd-gitlab          | 192.168.43.210 | gitlab.deemoprobe.com  | 配置Gitlab服务                |
| cicd-jenkins-ansible | 192.168.43.211 | jenkins.deemoprobe.com | 配置Jenkins和ansible服务      |
| cicd-dest            | 192.168.43.212 | dest.deemoprobe.com    | 模拟交付的目标                |
| Windows本机          | 192.168.43.213 | 无需配置               | 通过SSH对CICD主机进行配置管理 |

先在Windows主机上添加一下cicd机器的域名解析记录：

```bash
# 编辑C:\Windows\System32\drivers\etc\hosts文件
# 如果编辑保存时提示文件只读没有权限，复制hosts文件内容后重命名该文件
# 在桌面写一个hosts文件（不要有后缀名）粘贴原来文件的配置后写入自己的配置
# 最后复制桌面的hosts到C:\Windows\System32\drivers\etc\目录
# 或者使用火绒安全-安全工具里面的“修改HOSTS文件”功能可直接修改
192.168.43.210	gitlab.deemoprobe.com
192.168.43.211	jenkins.deemoprobe.com
192.168.43.212	dest.deemoprobe.com
```

## Gitlab实践

说明：生产环境请权衡防火墙和SELinux设置，在没有外层防火墙的情况下不建议禁用，按需开放所需端口策略即可。

```bash
# 禁用防火墙
[root@cicd-gitlab ~]# systemctl stop firewalld
[root@cicd-gitlab ~]# systemctl disable firewalld
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.

# 禁用Selinux
[root@cicd-gitlab ~]# setenforce 0
[root@cicd-gitlab ~]# sed -i "s/=enforcing/=disabled/g" /etc/selinux/config

# 安装依赖工具
[root@cicd-gitlab ~]# yum install -y curl policycoreutils-python openssh-server perl

# 安装启动邮件服务
[root@cicd-gitlab ~]# yum install postfix
[root@cicd-gitlab ~]# systemctl start postfix
[root@cicd-gitlab ~]# systemctl enable postfix

# 下载gitlab-ee仓库，ee是Enterprise Edition的意思，包含所有分支功能，ce版本仅包含社区版本
[root@cicd-gitlab ~]# curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.rpm.sh | sudo bash
# 安装gitlab-ee并设置URL为https://gitlab.deemoprobe.com
# 如果安装速度太慢，在 https://packages.gitlab.com/gitlab/gitlab-ee 页面下载下面命令执行后对应的版本
# 下载后上传到服务器，下载页面有详细的安装说明
[root@cicd-gitlab ~]# EXTERNAL_URL="https://gitlab.deemoprobe.com" yum install -y gitlab-ee

# 安装时指定EXTERNAL_URL，会在安装后同时配置好秘钥文件，存放在/etc/gitlab/ssl/
[root@cicd-gitlab ~]# ls /etc/gitlab/ssl/
gitlab.deemoprobe.com.crt  gitlab.deemoprobe.com.key  gitlab.deemoprobe.com.key-staging
# 使用key秘钥文件生成csr证书文件
[root@cicd-gitlab ~]# openssl req -new -key "/etc/gitlab/ssl/gitlab.deemoprobe.com.key" -out "/etc/gitlab/ssl/gitlab.deemoprobe.com.csr"
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:cn
State or Province Name (full name) []:sh
Locality Name (eg, city) [Default City]:sh
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server hostname) []:gitlab.deemoprobe.com
Email Address []:deemoprobe@gmail.com
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:123456 
An optional company name []:
# 根据csr和key文件重新生成crt秘钥
[root@cicd-gitlab ~]# openssl x509 -req -days 365 -in "/etc/gitlab/ssl/gitlab.deemoprobe.com.csr" -signkey "/etc/gitlab/ssl/gitlab.deemoprobe.com.key" -out "/etc/gitlab/ssl/gitlab.deemoprobe.com.crt"
Signature ok
subject=/C=cn/ST=sh/L=sh/O=Default Company Ltd/CN=gitlab.deemoprobe.com/emailAddress=deemoprobe@gmail.com
Getting Private key
# 创建pem证书
[root@cicd-gitlab ~]# openssl dhparam -out /etc/gitlab/ssl/dhparams.pem 2048
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
...............................+...........................................................................................................................++*++*
# 更改证书权限
[root@cicd-gitlab ssl]# chmod 600 *
[root@cicd-gitlab ssl]# ll
total 20
-rw-------. 1 root root  424 Mar  2 21:11 dhparams.pem
-rw-------. 1 root root 1298 Mar  2 21:08 gitlab.deemoprobe.com.crt
-rw-------. 1 root root 1082 Mar  2 21:04 gitlab.deemoprobe.com.csr
-rw-------. 1 root root 1675 Mar  2 20:52 gitlab.deemoprobe.com.key
-rw-------. 1 root root 1675 Mar  2 20:52 gitlab.deemoprobe.com.key-staging

# 配置gitlab配置文件
[root@cicd-gitlab ssl]# vim /etc/gitlab/gitlab.rb
nginx['redirect_http_to_https'] = true #取消注释并把false改为true
...             # 下面两行取消注释并更改秘钥名称为之前生成的秘钥
nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.deemoprobe.com.crt" 
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.deemoprobe.com.key"
...             # 下面一行取消注释并把nil改为自己的dhparams.pem证书，记得加上引号，否则初始化会报错
nginx['ssl_dhparam'] = "/etc/gitlab/ssl/dhparams.pem"

# 初始化gitlab配置，第一次需要一段时间，最后几行出现下面提示表示初始化成功
[root@cicd-gitlab ssl]# gitlab-ctl reconfigure
...
Running handlers:
Running handlers complete
Chef Infra Client finished, 1/847 resources updated in 15 seconds
gitlab Reconfigured!

# 编辑nginx配置，server_name这一行后添加一行
[root@cicd-gitlab ~]# vim /var/opt/gitlab/nginx/conf/gitlab-http.conf
  server_name gitlab.deemoprobe.com;
  rewrite ^(.*)$ https://$host$1 permanent;
# 重启
[root@cicd-gitlab ~]# gitlab-ctl restart
ok: run: alertmanager: (pid 3477) 0s
ok: run: gitaly: (pid 3485) 0s
ok: run: gitlab-exporter: (pid 3498) 1s
ok: run: gitlab-kas: (pid 3505) 0s
ok: run: gitlab-workhorse: (pid 3514) 1s
ok: run: grafana: (pid 3522) 0s
ok: run: logrotate: (pid 3533) 0s
ok: run: nginx: (pid 3539) 1s
ok: run: node-exporter: (pid 3545) 0s
ok: run: postgres-exporter: (pid 3550) 1s
ok: run: postgresql: (pid 3565) 0s
ok: run: prometheus: (pid 3574) 1s
ok: run: puma: (pid 3591) 0s
ok: run: redis: (pid 3596) 0s
ok: run: redis-exporter: (pid 3603) 0s
ok: run: sidekiq: (pid 3689) 0s
```

- 浏览器输入`gitlab.deemoprobe.com`，进入gitlab登陆页面

[![1][1]][1]

[1]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220302224619.png

```bash
# 默认用户root，密码位于/etc/gitlab/initial_root_password文件中，该文件会在24小时候自动删除
[root@cicd-gitlab ~]# cat /etc/gitlab/initial_root_password 
...
Password: 此处为密码
# NOTE: This file will be automatically deleted in the first reconfigure run after 24 hours.
```

[![2][2]][2]

[2]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220302230808.png

### 应用

- 新建两个账户：dev-开发人员使用；master-项目经理使用。同时为两个账户设置控制台登陆密码（如果使用该密码`git clone`操作验证被禁止，则可以为用户创建token）

[![3][3]][3]

[![4][4]][4]

[![5][5]][5]

[![6][6]][6]

[3]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303103209.png

[4]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303103356.png

[5]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303103505.png

[6]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303105039.png

- 新建项目仓库，为这两个用户赋权，dev-开发者权限，master-管理权限

![20220303104105](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303104105.png)

[![7][7]][7]

[7]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303104400.png

![20220303104337](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303104337.png)

![20220303104434](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303104434.png)

[![8][8]][8]

[8]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303104507.png

- 项目左侧settings里创建dev的Access Tokens，用以dev身份克隆项目登陆git，添加开发分支并同步到远程gitlab代码仓库，git操作可以参考博客：[GitHub使用手册](http://www.deemoprobe.com/standard/github/)

[![9][9]][9]

[9]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303110544.png

```bash
# 以下是Windows中的Git bash中的操作
deemoprobe@deemoprobe MINGW64 /d/DevOps
$ mkdir cicd
deemoprobe@deemoprobe MINGW64 /d/DevOps
$ cd cicd/
deemoprobe@deemoprobe MINGW64 /d/DevOps/cicd
$ git clone https://gitlab.deemoprobe.com/root/golang.git
Cloning into 'golang'... # Security Warning忽略
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (3/3), done.
deemoprobe@deemoprobe MINGW64 /d/DevOps/cicd
$ cd golang/
deemoprobe@deemoprobe MINGW64 /d/DevOps/cicd/golang (main)
$ git checkout -b version-1.0
Switched to a new branch 'version-1.0'
deemoprobe@deemoprobe MINGW64 /d/DevOps/cicd/golang (version-1.0)
$ vim main.go
package main

import (
    "fmt"
)

func main(){
    fmt.Println("Change from dev, version is 1.0")
}
deemoprobe@deemoprobe MINGW64 /d/DevOps/cicd/golang (version-1.0)
$ git add main.go
deemoprobe@deemoprobe MINGW64 /d/DevOps/cicd/golang (version-1.0)
$ git commit -m "version-1.0"
deemoprobe@deemoprobe MINGW64 /d/DevOps/cicd/golang (version-1.0)
$ git push origin version-1.0
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 8 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 359 bytes | 359.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
remote:
remote: To create a merge request for version-1.0, visit:
remote:   https://gitlab.deemoprobe.com/root/golang/-/merge_requests/new?merge_request%5Bsource_branch%5D=version-1.0
remote:
To https://gitlab.deemoprobe.com/root/golang.git
 * [new branch]      version-1.0 -> version-1.0
```

[![10][10]][10]

[10]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303113144.png

- dev用户登陆gitlab界面，创建代码合并申请

[![11][11]][11]

[11]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303113638.png

[![12][12]][12]

[12]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303114211.png

- master用户登陆，查看并批准申请

[![13][13]][13]

[13]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303114602.png

![20220303114632](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303114632.png)

- 返回项目界面，可见主分支已经合并了version-1.0的代码

[![14][14]][14]

[14]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220303114807.png

- 至此，Gitlab实践部分已完成

## Ansible实践

GitHub上面的Ansible项目如果直接拉到本地速度很慢，可以通过URL形式上传到gitee再拉取。

```bash
# 禁用防火墙和SELinux
[root@cicd-jenkins-ansible ~]# systemctl stop firewalld
[root@cicd-jenkins-ansible ~]# systemctl disable firewalld
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
[root@cicd-jenkins-ansible ~]# setenforce 0
[root@cicd-jenkins-ansible ~]# sed -i "s/=enforcing/=disabled/g" /etc/selinux/config

# 安装必要的工具包
[root@cicd-jenkins-ansible Python-3.10.2]# yum install gcc* make zlib* -y

# 升级OpenSSL
[root@cicd-jenkins-ansible ~]# wget https://www.openssl.org/source/openssl-1.1.1a.tar.gz
[root@cicd-jenkins-ansible ~]# tar -zxvf openssl-1.1.1a.tar.gz 
[root@cicd-jenkins-ansible ~]# cd openssl-1.1.1a
[root@cicd-jenkins-ansible openssl-1.1.1a]# ./config --prefix=/usr/local/openssl
[root@cicd-jenkins-ansible openssl-1.1.1a]# make && make install
[root@cicd-jenkins-ansible openssl-1.1.1a]# mv /usr/bin/openssl /usr/bin/openssl.bak
[root@cicd-jenkins-ansible openssl-1.1.1a]# ln -sf /usr/lcoal/openssl/bin/openssl /usr/bin/openssl
[root@cicd-jenkins-ansible openssl-1.1.1a]# cp /etc/ld.so.conf /etc/ld.so.conf.bak
[root@cicd-jenkins-ansible openssl-1.1.1a]# echo "/usr/local/openssl/lib" >> /etc/ld.so.conf
[root@cicd-jenkins-ansible openssl-1.1.1a]# ldconfig -v

# 安装Python，安装包已经提前下载并上传到了主机
[root@cicd-jenkins-ansible ~]# tar -zxf Python-3.7.7.tgz
[root@cicd-jenkins-ansible ~]# cd Python-3.7.7
[root@cicd-jenkins-ansible Python-3.7.7]# ./configure --prefix=/usr/local/ --with-ensurepip=install --with-openssl=/usr/local/openssl --enable-shared LDFLAGS="-Wl,-rpath /usr/local/lib"
[root@cicd-jenkins-ansible Python-3.7.7]# make && make altinstall

# pip3.7软链接，之后安装virtualenv
[root@cicd-jenkins-ansible ~]# ln -s /usr/local/bin/pip3.7 /usr/local/bin/pip
# 配置pip国内源
[root@cicd-jenkins-ansible Python-3.7.7]# mkdir ~/.pip
[root@cicd-jenkins-ansible Python-3.7.7]# vim ~/.pip/pip.conf
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host = https://pypi.tuna.tsinghua.edu.cn
# 更新pip（可选）
[root@cicd-jenkins-ansible Python-3.7.7]# pip install --upgrade pip
# 安装virtualenv
[root@cicd-jenkins-ansible Python-3.7.7]# pip install virtualenv

# 创建普通用户deploy并登陆配置virtualenv环境
[root@cicd-jenkins-ansible Python-3.7.7]# cd ~
[root@cicd-jenkins-ansible ~]# useradd deploy
[root@cicd-jenkins-ansible ~]# su - deploy
[deploy@cicd-jenkins-ansible ~]$ virtualenv -p /usr/local/bin/python3.7 ansible
created virtual environment CPython3.7.7.final.0-64 in 448ms
  creator CPython3Posix(dest=/home/deploy/ansible, clear=False, no_vcs_ignore=False, global=False)
  seeder FromAppData(download=False, pip=bundle, setuptools=bundle, wheel=bundle, via=copy, app_data_dir=/home/deploy/.local/share/virtualenv)
    added seed packages: pip==22.0.3, setuptools==60.9.3, wheel==0.37.1
  activators BashActivator,CShellActivator,FishActivator,NushellActivator,PowerShellActivator,PythonActivator
[deploy@cicd-jenkins-ansible ~]$ ls
ansible
# 拉取Ansible项目
[deploy@cicd-jenkins-ansible ~]$ cd ansible/
[deploy@cicd-jenkins-ansible ansible]$ git clone https://gitee.com/deemoprobe/ansible.git
Cloning into 'ansible'...
remote: Enumerating objects: 572745, done.
remote: Counting objects: 100% (572745/572745), done.
remote: Compressing objects: 100% (136116/136116), done.
remote: Total 572745 (delta 384427), reused 572745 (delta 384427), pack-reused 0
Receiving objects: 100% (572745/572745), 199.74 MiB | 11.11 MiB/s, done.
Resolving deltas: 100% (384427/384427), done.
# 加载ansible环境
[deploy@cicd-jenkins-ansible ansible]$ source ~/ansible/bin/activate
# 为deploy用户配置pip源
(ansible) [deploy@cicd-jenkins-ansible ansible]$ mkdir ~/.pip
(ansible) [deploy@cicd-jenkins-ansible ansible]$ vim ~/.pip/pip.conf
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host = https://pypi.tuna.tsinghua.edu.cn
# 安装依赖
(ansible) [deploy@cicd-jenkins-ansible ansible]$ pip install paramiko PyYAML jinja2
(ansible) [deploy@cicd-jenkins-ansible ansible]$ cd ansible/
# 切换到2.10版本分支
(ansible) [deploy@cicd-jenkins-ansible ansible]$ git checkout stable-2.10
# 加载ansible-2.10
(ansible) [deploy@cicd-jenkins-ansible ansible]$ source ~/ansible/ansible/hacking/env-setup -q
(ansible) [deploy@cicd-jenkins-ansible ansible]$ ansible --version
ansible 2.10.17.post0
  config file = None
  configured module search path = ['/home/deploy/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/deploy/ansible/ansible/lib/ansible
  executable location = /home/deploy/ansible/ansible/bin/ansible
  python version = 3.7.7 (default, Mar  3 2022, 14:46:38) [GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]

# 为deploy用户配置ansible环境变量，即每次登入deploy用户自动加载其环境变量，进入ansible-2.10环境
(ansible) [deploy@cicd-jenkins-ansible ~]$ vim ~/.bashrc 
...             # 配置文件末尾加入下面两行
source ~/ansible/bin/activate
source ~/ansible/ansible/hacking/env-setup -q
```

- 至此，完成了Python3.7.7、virtualenv、ansible-2.10的安装

> 更高版本的Ansible需要更高版本的Python环境提供支持，比如：Ansible-2.12依赖Python3.8+

### Playbooks规则

Playbooks结构：

- inventory：目录，server清单目录
  - testenv：文件，server清单和变量文件
- roles：目录，roles任务目录
  - testbox：目录，具体任务目录
    - tasks：目录，存放任务文件
      - main.yml：文件，主任务文件
- deploy.yml：文件，playbook任务入口文件

```bash
# testenv文件
[testservers] # server清单字段
dest.deemoprobe.com # 具体的主机域名或IP
[testservers:vars] # 主机参数字段
server_name=dest.deemoprobe.com # 目标主机的参数
user=root
output=/root/dest.txt

# main.yml文件
- name: Print user and server_name # 任务名称
remote testbox # 具体任务
  shell: "echo 'User is {{ user }}, server_name is {{ server_name }}' > {{ output }}" # 调用testenv

# deploy.yml文件
- hosts "testservers" # 对应server清单字段名
  gather_facts: true # 获取server基本信息
  remote_user: root # 指定目标主机用户
  roles: # 指定roles/testbox下面的任务
    - testbox

[root@cicd-jenkins-ansible ~]# vim /etc/hosts
... # 加入域名解析记录
192.168.43.212    dest.deemoprobe.com

# 创建playbook，yml文件只能使用空格缩进，使用tab键会报错
[root@cicd-jenkins-ansible ~]# su - deploy
(ansible) [deploy@cicd-jenkins-ansible ~]$ mkdir testplaybooks
(ansible) [deploy@cicd-jenkins-ansible ~]$ cd testplaybooks/
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ mkdir inventory
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ mkdir -p roles/testbox/tasks
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ vim inventory/testenv
[testservers] 
dest.deemoprobe.com 
[testservers:vars] 
server_name=dest.deemoprobe.com 
user=root
output=/root/dest.txt
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ vim roles/testbox/tasks/main.yml
- name: Print user and server name
  shell: "echo 'User is {{ user }} server name is {{ server_name }}' > {{ output }}"
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ vim deploy.yml
- hosts: "testservers" 
  gather_facts: true 
  remote_user: root 
  roles: 
    - testbox
# 查看目录结构
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ cd ..
(ansible) [deploy@cicd-jenkins-ansible ~]$ tree testplaybooks/
testplaybooks/
├── deploy.yml
├── inventory
│   └── testenv
└── roles
    └── testbox
        └── tasks
            └── main.yml
4 directories, 3 files

# 配置SSH免密登陆ansible目标主机
(ansible) [deploy@cicd-jenkins-ansible ~]$ ssh-keygen -t rsa
(ansible) [deploy@cicd-jenkins-ansible ~]$ ssh-copy-id -i /home/deploy/.ssh/id_rsa.pub root@dest.deemoprobe.com
# 执行playbook
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ ansible-playbook -i inventory/testenv ./deploy.yml 

PLAY [testservers] ****************************************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************************
ok: [dest.deemoprobe.com]

TASK [testbox : Print user and server name] ***************************************************************************************************************
changed: [dest.deemoprobe.com]

PLAY RECAP ************************************************************************************************************************************************
dest.deemoprobe.com        : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
# 到目标主机查看操作结果
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ ssh root@dest.deemoprobe.com
Last login: Thu Mar  3 20:26:15 2022 from 192.168.43.211
[root@cicd-dest ~]# cat dest.txt 
User is root server name is dest.deemoprobe.com
```

### Playbooks常用模块

```bash
# file模块
- name: Create a file # 名称
  file: 'path=/root/test.txt state=touch mode=0755 owner=deploy group=deploy' # 具体操作

# copy模块
- name: Copy a file
  copy: 'remote_src=no src=roles/testbox/files/test.txt dest=/root/test.txt mode=0644 force=yes'

# stat模块
- name: Check file state
  stat: 'path=/root/test.txt'
  register: file_stat # 将文件状态信息传给变量file_stat

# debug模块
- debug: msg="test.txt exists"
  when: file_stat.stat.exists # 通过变量判断文件是否存在

# shell模块
- name: Run a command
  shell: "echo 'test' > /root/test.txt"

# template模块（配置文件jinja2模板，模板会调用testenv文件里定义的变量）
- name: Write nginx conf
  template: src=roles/testbox/templates/nginx.conf.j2 dest=/etc/nginx/nginx.conf

# packaging模块，调用包管理工具（如yum）
- name: Ensure nginx is the latest version
  yum: pkg=nginx state=latest

# service模块，调用系统service或systemctl命令
- name: Start nginx service
  service: name=nginx state=started

# 创建综合的模块实例playbook，在上面的testplaybooks基础上
(ansible) [deploy@cicd-jenkins-ansible ~]$ cd testplaybooks/
# 创建脚本文件
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ mkdir -p roles/testbox/files
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ vim roles/testbox/files/test.sh
echo "Ansible script test"
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ vim inventory/testenv 
[testservers]
dest.deemoprobe.com
[testservers:vars]
server_name=dest.deemoprobe.com
user=root
output=/root/dest.txt
port=80
worker_processes=4
max_open_file=65500
# 创建模板文件
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ mkdir roles/testbox/templates
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ vim roles/testbox/templates/nginx.conf.j2
worker_processes {{ worker_processes }};
error_log /var/log/nginx/error.log error;
pid /var/run/nginx.pid;
events {
    use epoll;
    worker_connections {{ max_open_file }};
}
http {
    include mime.types;
    default_type application/octet-stream;
    log_format access.log  '$remote_addr - [$time_local] - "$request" - "$http_user_agent"';
    sendfile        on;
    keepalive_timeout  120;
    server {
        listen       {{ port }};
        server_name  {{ server_name }};
        access_log /var/log/nginx/access_log;
        location / {
            root   /www;
            index  index.html;
        }
        error_page 404 /404.html;
    }
}
# 编辑主任务文件
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ vim roles/testbox/tasks/main.yml 
- name: Print user and server name
  shell: "echo 'User is {{ user }} server name is {{ server_name }}' > {{ output }}"
- name: Create test.txt
  file: 'path=/root/test.txt state=touch mode=0755 owner=root group=root'
- name: Copy test.sh
  copy: 'remote_src=no src=roles/testbox/files/test.sh dest=/root/test.sh mode=0644 force=yes'
- name: Check test.sh exists
  stat: 'path=/root/test.sh'
  register: script_stat
- debug: msg="test.sh exists"
  when: script_stat.stat.exists
- name: Run script test.sh
  shell: 'sh /root/test.sh'
- name: Add nginx repo
  yum_repository: name=nginx description="nginx repo" baseurl="http://nginx.org/packages/centos/7/$basearch/" gpgcheck=no enabled=1
- name: Install and ensure nginx is the latest version
  yum: name=nginx state=latest
- name: Write nginx conf file
  template: src=roles/testbox/templates/nginx.conf.j2 dest=/etc/nginx/nginx.conf
- name: Start nginx service
  service: name=nginx state=started
# 执行playbook
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ ansible-playbook -i inventory/testenv ./deploy.yml 

PLAY [testservers] ****************************************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************************
ok: [dest.deemoprobe.com]

TASK [testbox : Print user and server name] ***************************************************************************************************************
changed: [dest.deemoprobe.com]

TASK [testbox : Create test.txt] **************************************************************************************************************************
changed: [dest.deemoprobe.com]

TASK [testbox : Copy test.sh] *****************************************************************************************************************************
ok: [dest.deemoprobe.com]

TASK [testbox : Check test.sh exists] *********************************************************************************************************************
ok: [dest.deemoprobe.com]

TASK [testbox : debug] ************************************************************************************************************************************
ok: [dest.deemoprobe.com] => {
    "msg": "test.sh exists"
}

TASK [testbox : Run script test.sh] ***********************************************************************************************************************
changed: [dest.deemoprobe.com]

TASK [testbox : Add nginx repo] ***************************************************************************************************************************
changed: [dest.deemoprobe.com]

TASK [testbox : Install and ensure nginx is the latest version] *******************************************************************************************
changed: [dest.deemoprobe.com]

TASK [testbox : Write nginx conf file] ********************************************************************************************************************
changed: [dest.deemoprobe.com]

TASK [testbox : Start nginx service] **********************************************************************************************************************
changed: [dest.deemoprobe.com]

PLAY RECAP ************************************************************************************************************************************************
dest.deemoprobe.com        : ok=11   changed=7    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 

# 登入目标主机，查看情况
(ansible) [deploy@cicd-jenkins-ansible testplaybooks]$ ssh root@dest.deemoprobe.com
[root@cicd-dest ~]# ls -al /root/test*
-rw-r--r--. 1 root root 27 Mar  3 21:36 /root/test.sh
-rwxr-xr-x. 1 root root  0 Mar  3 21:46 /root/test.txt
[root@cicd-dest ~]# cat /root/test.sh 
echo "Ansible script test"
[root@cicd-dest ~]# cat /etc/nginx/nginx.conf 
worker_processes 4;
error_log /var/log/nginx/error.log error;
pid /var/run/nginx.pid;
events {
    use epoll;
    worker_connections 65500;
}
http {
    include mime.types;
    default_type application/octet-stream;
    log_format access.log  '$remote_addr - [$time_local] - "$request" - "$http_user_agent"';
    sendfile        on;
    keepalive_timeout  120;
    server {
        listen       80;
        server_name  dest.deemoprobe.com;
        access_log /var/log/nginx/access_log;
        location / {
            root   /www;
            index  index.html;
        }
        error_page 404 /404.html;
    }
}
# 查看进程和监听端口
[root@cicd-dest ~]# ps -ef | grep nginx
root      14859      1  0 21:48 ?        00:00:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
nginx     14860  14859  0 21:48 ?        00:00:00 nginx: worker process
nginx     14861  14859  0 21:48 ?        00:00:00 nginx: worker process
nginx     14862  14859  0 21:48 ?        00:00:00 nginx: worker process
nginx     14863  14859  0 21:48 ?        00:00:00 nginx: worker process
root      14900  14880  0 21:55 pts/1    00:00:00 grep --color=auto nginx
[root@cicd-dest ~]# ss -lntp | grep 80
LISTEN     0      128          *:80                       *:*                   users:(("nginx",pid=14863,fd=6),("nginx",pid=14862,fd=6),("nginx",pid=14861,fd=6),("nginx",pid=14860,fd=6),("nginx",pid=14859,fd=6))
```

以上nginx仅用于Ansible Playbook实践测试，实际生产环境`nginx.conf`文件还需要更多的配置和优化。由于本文重点不在nginx，未一一列举。配置文件详解可参考博客：[Nginx配置文件详解](https://www.deemoprobe.com/yunv/nginxconf/)

## Jenkins实践

Jenkins和Ansible使用的是同一台服务器，Ansible安装时Linux环境禁用防火墙和SELinux已操作，如果选择新机器安装Jenkins，需要重新操作

```bash
# 导入Jenkins repo和key
[root@cicd-jenkins-ansible ~]# wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
[root@cicd-jenkins-ansible ~]# rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
# 安装jdk
[root@cicd-jenkins-ansible ~]# yum install epel-release
[root@cicd-jenkins-ansible ~]#  yum install java-11-openjdk-devel
[root@cicd-jenkins-ansible ~]# java -version
openjdk version "11.0.14.1" 2022-02-08 LTS
OpenJDK Runtime Environment 18.9 (build 11.0.14.1+1-LTS)
OpenJDK 64-Bit Server VM 18.9 (build 11.0.14.1+1-LTS, mixed mode, sharing)
# 安装Jenkins最新版
[root@cicd-jenkins-ansible ~]# yum install jenkins -y
# Jenkins用户改为deploy，此处deploy和ansible一致
[root@cicd-jenkins-ansible ~]# vim /etc/sysconfig/jenkins
JENKINS_USER="deploy"
# 更改Jenkins家目录、日志目录和缓存目录权限为deploy
[root@cicd-jenkins-ansible ~]# chown -R deploy. /var/lib/jenkins/
[root@cicd-jenkins-ansible ~]# chown -R deploy. /var/log/jenkins/
[root@cicd-jenkins-ansible ~]# chown -R deploy. /var/cache/jenkins/
# 启动Jenkins
[root@cicd-jenkins-ansible jenkins]# systemctl restart jenkins
[root@cicd-jenkins-ansible jenkins]# systemctl status jenkins
● jenkins.service - LSB: Jenkins Automation Server
   Loaded: loaded (/etc/rc.d/init.d/jenkins; bad; vendor preset: disabled)
   Active: active (running) since Fri 2022-03-04 11:12:29 CST; 37s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 2029 ExecStop=/etc/rc.d/init.d/jenkins stop (code=exited, status=0/SUCCESS)
  Process: 2040 ExecStart=/etc/rc.d/init.d/jenkins start (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/jenkins.service
           └─2046 /etc/alternatives/java -Djava.awt.headless=true -DJENKINS_HOME=/var/lib/jenkins -jar /usr/lib/jenkins/jenkins.war --logfile=/var/log/j...

Mar 04 11:12:29 cicd-jenkins-ansible systemd[1]: Starting LSB: Jenkins Automation Server...
Mar 04 11:12:29 cicd-jenkins-ansible systemd[1]: Started LSB: Jenkins Automation Server.
Mar 04 11:12:29 cicd-jenkins-ansible jenkins[2040]: Starting Jenkins [  OK  ]
# 查看端口
[root@cicd-jenkins-ansible jenkins]# ss -lntp | grep 8080
LISTEN     0      50        [::]:8080                  [::]:*                   users:(("java",pid=2046,fd=115))
```

- 浏览器输入：<http://jenkins.deemoprobe.com:8080/>，几十秒后进入`Unlock Jenkins`界面

![20220304111621](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304111621.png)

![20220304111738](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304111738.png)

- 按照提示去服务器找到`/var/lib/jenkins/secrets/initialAdminPassword`文件中的密码，授权后进入插件安装界面

![20220304112105](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304112105.png)

- 配置阿里更新源<https://mirrors.aliyun.com/jenkins/updates/update-center.json>

```bash
[root@cicd-jenkins-ansible jenkins]# pwd
/var/lib/jenkins
[root@cicd-jenkins-ansible jenkins]# cp hudson.model.UpdateCenter.xml hudson.model.UpdateCenter.xml.bak
[root@cicd-jenkins-ansible jenkins]# vim hudson.model.UpdateCenter.xml
<?xml version='1.1' encoding='UTF-8'?>
<sites>
  <site>
    <id>default</id>
    <url>https://mirrors.aliyun.com/jenkins/updates/update-center.json</url>
  </site>
</sites>
```

- 安装建议的插件，十分钟左右可以装完，进入创建用户界面

![20220304114205](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304114205.png)

- 用户创建后确认URL正确，完成即可进入控制台界面

[![15][15]][15]

[15]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304114619.png

- 配置插件管理URL为阿里源

[![16][16]][16]

[16]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304114955.png

- 重启，<http://jenkins.deemoprobe.com:8080/restart>

![20220304115053](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304115053.png)

- 输入前面设置的账户密码，登陆

![20220304115214](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304115214.png)

```bash
[root@cicd-jenkins-ansible jenkins]# vim /etc/hosts
...     # 添加gitlab server的解析记录
192.168.43.210    gitlab.deemoprobe.com
# 配置git sslVerify
[root@cicd-jenkins-ansible jenkins]# git config --system http.sslVerify false
```

- Jenkins控制台配置git global user和email

![20220304145750](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304145750.png)

- 添加Git Credential，用户和密码是gitlab的账户和密码

![20220304145450](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304145450.png)

### FreestyleJob

- 创建test-freestyle-job，New Item-->Freestyle Project

[![31][31]][31]

[31]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304145915.png

![2022-03-04_1515252](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/2022-03-04_1515252.png)

```bash
# build构建脚本内容

#!/bin/bash

export PATH="/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin"

echo "[INFO] Print env"
echo "Current env is $deploy_env"  >> test.properties
echo "Build version is $version" >> test.properties
echo "[INFO] Done..."

echo "[INFO] Check test.properties"
if [ -s test.properties ]
then
  cat test.properties
  echo "[INFO] Done..."
else
  echo "test.properties is empty"
fi

echo "[INFO] Build finished..."
```

- 构建，Build with Parameters

![20220304152409](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304152409.png)

![20220304152513](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304152513.png)

- 左下角可见构建成功

![20220304152601](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304152601.png)

- 查看"#1"这个构建记录，

[![32][32]][32]

[32]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304152711.png

```bash
# 构建记录控制台输出
Started by user deemoprobe
Running as SYSTEM
Building in workspace /var/lib/jenkins/workspace/test-freestyle-job
The recommended git tool is: NONE
using credential c7567e8e-a9a3-442e-b033-5486c6d3170f
Cloning the remote Git repository
Cloning repository https://gitlab.deemoprobe.com/root/golang.git
 > git init /var/lib/jenkins/workspace/test-freestyle-job # timeout=10
Fetching upstream changes from https://gitlab.deemoprobe.com/root/golang.git
 > git --version # timeout=10
 > git --version # 'git version 1.8.3.1'
using GIT_ASKPASS to set credentials 
 > git fetch --tags --progress https://gitlab.deemoprobe.com/root/golang.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git config remote.origin.url https://gitlab.deemoprobe.com/root/golang.git # timeout=10
 > git config --add remote.origin.fetch +refs/heads/*:refs/remotes/origin/* # timeout=10
Avoid second fetch
 > git rev-parse refs/remotes/origin/main^{commit} # timeout=10
Checking out Revision ed2e5ade521de619582167ae5542e8111a38aa16 (refs/remotes/origin/main)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f ed2e5ade521de619582167ae5542e8111a38aa16 # timeout=10
Commit message: "Merge branch 'version-1.0' into 'main'"
First time build. Skipping changelog.
[test-freestyle-job] $ /bin/bash /tmp/jenkins8051294503775227285.sh
[INFO] Print env
[INFO] Done...
[INFO] Check test.properties
Current env is prod
Build version is 1.0.0
[INFO] Done...
[INFO] Build finished...
Finished: SUCCESS

# 根据控制台输出查看这个job存放的路径，golang这个项目已经拉取，并且创建了文件test.properties
[root@cicd-jenkins-ansible jenkins]# cd /var/lib/jenkins/workspace/test-freestyle-job
[root@cicd-jenkins-ansible test-freestyle-job]# ll
total 16
-rw-r--r-- 1 deploy deploy  102 Mar  4 15:25 main.go
-rw-r--r-- 1 deploy deploy 6218 Mar  4 15:25 README.md
-rw-r--r-- 1 deploy deploy   43 Mar  4 15:25 test.properties
```

### PipelineJob

```bash
# Pipeline语法结构
pipeline {
  agent any # 构建主机，任意一台
  environment { # 环境变量 key=value
    host='dest.deemoprobe.com'
    user='deploy'
  }
  stages {
    stage('build') { # 管道模块
      steps { # 具体操作
        sh "cat $host"
        echo $deploy
      }
    }
  }
}
```

- 创建test-pipeline-job，New Item-->Pipeline

![20220304154408](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304154408.png)

- 获取Git Credential ID，编写Pipeline Groovy脚本

![20220304155554](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304155554.png)

```bash
#!groovy

pipeline {
  agent { node {label 'master'}}
  environment{
    PATH="/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin"
  }
  parameters {
    choice(
      choices: 'dev\nprod',
      description: 'Choose env',
      name: 'deploy_env'
    )
    string (name: 'version', defaultValue: '1.0.0', description: 'Build version')
  }
  stages {
    stage("Checkout test repo") {
      steps {
        sh 'git config --global http.sslVerify false'
        dir ("${env.WORKSPACE}") {
          git branch: 'main', credentialsId: "c7567e8e-a9a3-442e-b033-5486c6d3170f", url: 'https://gitlab.deemoprobe.com/root/golang.git'
        }
      }
    }
    stage("Print env") {
      steps {
        dir ("${env.WORKSPACE}") {
          sh """
          echo "[INFO] Print env"
          echo "Current env is $deploy_env"  >> test.properties
          echo "Build version is $version" >> test.properties
          echo "[INFO] Done..."
          """
        }
      }
    }
    stage("Check test.properties") {
      steps {
        dir ("${env.WORKSPACE}") {
          sh """
          echo "[INFO] Check test.properties"
          if [ -s test.properties ]
          then
            cat test.properties
            echo "[INFO] Done..."
          else
            echo "test.properties is empty"
          fi
          """
          echo "[INFO] Build finished..."
        }
      }
    }
  }
}
```

- 构建，Build Now，如果设置了参数，首次构建会加载参数，构建提示失败，之后`Build with Parameters`选择参数后即可继续构建，如果依旧没有成功查看构建记录分析即可，一般是语法问题

![20220304161045](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304161045.png)

- 查看构建记录，由于没找到标签为master的主机，所以无法继续进行，脚本内容中`agent { node {label 'master'}}`改为`agent any`

![17](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/2022-03-04_1647082.png)

- 重新构建`Build with Parameters`，构建成功后查看构建记录

[![18][18]][18]

[18]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304162519.png

```bash
# 构建记录
Started by user deemoprobe
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /var/lib/jenkins/workspace/test-pipeline-job
[Pipeline] {
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Checkout test repo)
[Pipeline] sh
+ git config --global http.sslVerify false
[Pipeline] dir
Running in /var/lib/jenkins/workspace/test-pipeline-job
[Pipeline] {
[Pipeline] git
The recommended git tool is: NONE
using credential c7567e8e-a9a3-442e-b033-5486c6d3170f
 > git rev-parse --resolve-git-dir /var/lib/jenkins/workspace/test-pipeline-job/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://gitlab.deemoprobe.com/root/golang.git # timeout=10
Fetching upstream changes from https://gitlab.deemoprobe.com/root/golang.git
 > git --version # timeout=10
 > git --version # 'git version 1.8.3.1'
using GIT_ASKPASS to set credentials 
 > git fetch --tags --progress https://gitlab.deemoprobe.com/root/golang.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/main^{commit} # timeout=10
Checking out Revision 2aaab90a5535e5444f8513665d2b6e89d7551308 (refs/remotes/origin/main)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 2aaab90a5535e5444f8513665d2b6e89d7551308 # timeout=10
 > git branch -a -v --no-abbrev # timeout=10
 > git branch -D main # timeout=10
 > git checkout -b main 2aaab90a5535e5444f8513665d2b6e89d7551308 # timeout=10
Commit message: "Delete test.properties"
 > git rev-list --no-walk 2aaab90a5535e5444f8513665d2b6e89d7551308 # timeout=10
[Pipeline] }
[Pipeline] // dir
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Print env)
[Pipeline] dir
Running in /var/lib/jenkins/workspace/test-pipeline-job
[Pipeline] {
[Pipeline] sh
+ echo '[INFO] Print env'
[INFO] Print env
+ echo 'Current env is dev'
+ echo 'Build version is 1.0.0'
+ echo '[INFO] Done...'
[INFO] Done...
[Pipeline] }
[Pipeline] // dir
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Check test.properties)
[Pipeline] dir
Running in /var/lib/jenkins/workspace/test-pipeline-job
[Pipeline] {
[Pipeline] sh
+ echo '[INFO] Check test.properties'
[INFO] Check test.properties
+ '[' -s test.properties ']'
+ cat test.properties
Current env is dev
Build version is 1.0.0
+ echo '[INFO] Done...'
[INFO] Done...
[Pipeline] echo
[INFO] Build finished...
[Pipeline] }
[Pipeline] // dir
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
# 查看test-pipeline-job目录
[root@cicd-jenkins-ansible test-freestyle-job]# cd /var/lib/jenkins/workspace/test-pipeline-job
[root@cicd-jenkins-ansible test-pipeline-job]# ls
main.go  null  null@tmp  README.md  test.properties
[root@cicd-jenkins-ansible test-pipeline-job]# cat test.properties 
Current env is dev
Build version is 1.0.0
```

- 构建后文件保存在本地，并没有同步到gitlab，此处以上传`test.properties`文件到远程gitlab仓库为例

```bash
[root@cicd-jenkins-ansible test-pipeline-job]# git config --global user.email "deemoprobe@gmail.com"
[root@cicd-jenkins-ansible test-pipeline-job]# git config --global user.name "deemoprobe"
[root@cicd-jenkins-ansible test-pipeline-job]# git add test.properties
[root@cicd-jenkins-ansible test-pipeline-job]# git commit -m "Add file test.properties"
[main 16503ac] Add file test.properties
 1 file changed, 2 insertions(+)
 create mode 100644 test.properties
[root@cicd-jenkins-ansible test-pipeline-job]# git push origin main
Username for 'https://gitlab.deemoprobe.com': root # 输入gitlab账户密码确认
Password for 'https://root@gitlab.deemoprobe.com': 
Counting objects: 4, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 357 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To https://gitlab.deemoprobe.com/root/golang.git
   ed2e5ad..16503ac  main -> main
```

- 为主机打标签，控制台路径：Manage Jenkins-->Manage Nodes and Clouds-->点击机器后面设置按钮-->填写标签名master

[![19][19]][19]

[![20][20]][20]

[19]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304173034.png

[20]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304173101.png

- 创建`test-pipeline-agent`，执行带有`agent { node {label 'master'}}`脚本

[![21][21]][21]

[![22][22]][22]

[21]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304173800.png

[22]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304173738.png

- 查看控制台输出

```bash
Started by user deemoprobe
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /var/lib/jenkins/workspace/test-pipeline-agent
[Pipeline] {
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Checkout test repo)
[Pipeline] sh
+ git config --global http.sslVerify false
[Pipeline] dir
Running in /var/lib/jenkins/workspace/test-pipeline-agent
[Pipeline] {
[Pipeline] git
The recommended git tool is: NONE
using credential c7567e8e-a9a3-442e-b033-5486c6d3170f
 > git rev-parse --resolve-git-dir /var/lib/jenkins/workspace/test-pipeline-agent/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://gitlab.deemoprobe.com/root/golang.git # timeout=10
Fetching upstream changes from https://gitlab.deemoprobe.com/root/golang.git
 > git --version # timeout=10
 > git --version # 'git version 1.8.3.1'
using GIT_ASKPASS to set credentials 
 > git fetch --tags --progress https://gitlab.deemoprobe.com/root/golang.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/main^{commit} # timeout=10
Checking out Revision 7dcbc535a683e37abf1312773cb66cc912d1a664 (refs/remotes/origin/main)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 7dcbc535a683e37abf1312773cb66cc912d1a664 # timeout=10
 > git branch -a -v --no-abbrev # timeout=10
 > git branch -D main # timeout=10
 > git checkout -b main 7dcbc535a683e37abf1312773cb66cc912d1a664 # timeout=10
Commit message: "Add test.properties"
 > git rev-list --no-walk 7dcbc535a683e37abf1312773cb66cc912d1a664 # timeout=10
[Pipeline] }
[Pipeline] // dir
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Print env)
[Pipeline] dir
Running in /var/lib/jenkins/workspace/test-pipeline-agent
[Pipeline] {
[Pipeline] sh
+ echo '[INFO] Print env'
[INFO] Print env
+ echo 'Current env is dev'
+ echo 'Build version is 1.0.0'
+ echo '[INFO] Done...'
[INFO] Done...
[Pipeline] }
[Pipeline] // dir
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Check test.properties)
[Pipeline] dir
Running in /var/lib/jenkins/workspace/test-pipeline-agent
[Pipeline] {
[Pipeline] sh
+ echo '[INFO] Check test.properties'
[INFO] Check test.properties
+ '[' -s test.properties ']'
+ cat test.properties
Current env is dev
Build version is 1.0.0
Current env is dev
Build version is 1.0.0
+ echo '[INFO] Done...'
[INFO] Done...
[Pipeline] echo
[INFO] Build finished...
[Pipeline] }
[Pipeline] // dir
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```

上面使用了Shell集成、Git集成和参数集成

### Jenkins集成Maven

官方下载点：<https://maven.apache.org/download.cgi> 下载apache-maven-3.8.4-bin.tar.gz上传到服务器安装

```bash
# 解压安装至/usr/local，查看Maven home和Java home（runtime）
[root@cicd-jenkins-ansible ~]# tar -zxvf apache-maven-3.8.4-bin.tar.gz -C /usr/local/
[root@cicd-jenkins-ansible ~]# cd /usr/local/apache-maven-3.8.4/
[root@cicd-jenkins-ansible apache-maven-3.8.4]# cd bin/
[root@cicd-jenkins-ansible bin]# ./mvn --version
Apache Maven 3.8.4 (9b656c72d54e5bacbed989b64718c159fe39b537)
Maven home: /usr/local/apache-maven-3.8.4
Java version: 11.0.14.1, vendor: Red Hat, Inc., runtime: /usr/lib/jvm/java-11-openjdk-11.0.14.1.1-1.el7_9.x86_64
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "4.19.12-1.el7.elrepo.x86_64", arch: "amd64", family: "unix"

# 在gitlab创建Java-war的新项目，克隆到本地
[root@cicd-jenkins-ansible ~]# git clone https://gitlab.deemoprobe.com/root/java-war.git
# 创建Maven实例项目，上传至gitlab
[root@cicd-jenkins-ansible ~]# mkdir -p java-war/src/main/webapp/WEB-INF/;cd java-war/
# pom.xml内容
[root@cicd-jenkins-ansible java-war]# vim pom.xml 
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.jenkins.demo</groupId>
  <artifactId>Java-war</artifactId>
  <packaging>war</packaging>
  <version>1.0.15-SNAPSHOT</version>
  <name>Java-war Maven Webapp</name>
  <url>http://maven.apache.org</url>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
  <build>
    <finalName>Java-war</finalName>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-release-plugin</artifactId>
        <version>2.5</version>
        <configuration>
          <autoVersionSubmodules>true</autoVersionSubmodules>
        </configuration>
      </plugin>
    </plugins>
  </build>
  <distributionManagement>
    <snapshotRepository>
      <id>releases</id>
      <name>Internal Releases</name>
      <url>http://nexus.xxx.com/repository/Java-war-tomcat/</url> 
    </snapshotRepository>
  </distributionManagement>
</project>
# index.jsp内容
[root@cicd-jenkins-ansible java-war]# vim src/main/webapp/index.jsp 
<html>
<head>
<title>Hello World!!</title>
</head>
<body>
  <h2>Awesome, Jenkins Successfully Deployed War File!</h2>
  <p>
    It is now
    <%= new java.util.Date() %></p>
  <p>
    You are coming from 
    <%= request.getRemoteAddr()  %></p>
</body>
# web.xml内容
[root@cicd-jenkins-ansible java-war]# vim src/main/webapp/WEB-INF/web.xml 
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >
<web-app>
  <display-name>Java-war Web Application</display-name>
</web-app>
[root@cicd-jenkins-ansible java-war]# git add -A
[root@cicd-jenkins-ansible java-war]# git commit -m "add file"
[main b2f6b64] add file
 3 files changed, 23 insertions(+), 24 deletions(-)
[root@cicd-jenkins-ansible java-war]# git push origin main
Username for 'https://gitlab.deemoprobe.com': root
Password for 'https://root@gitlab.deemoprobe.com': 
Counting objects: 17, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (6/6), done.
Writing objects: 100% (9/9), 915 bytes | 0 bytes/s, done.
Total 9 (delta 2), reused 0 (delta 0)
To https://gitlab.deemoprobe.com/root/java-war.git
   1bf422c..b2f6b64  main -> main
[root@cicd-jenkins-ansible ~]# tree java-war/
java-war/
├── pom.xml
├── README.md
└── src
    └── main
        └── webapp
            ├── index.jsp
            └── WEB-INF
                └── web.xml
```

![20220304201406](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304201406.png)

- 配置JDK和Maven集成，Manage Jenkins-->Global Tool Configuration

[![23][23]][23]

[![24][24]][24]

[23]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304202148.png

[24]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304202203.png

- 创建Maven Freestyle job

![192](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/2022-03-04_2038192.png)

- 首次构建Maven会去中央库下载很多依赖，大约需要五分钟，之后才会执行构建，下方截图已省略中央库依赖下载过程

[![26][26]][26]

[![27][27]][27]

[26]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304203549.png

[27]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304203711.png

- 可到服务器查看`maven-freestyle-job`文件

```bash
[root@cicd-jenkins-ansible ~]# cd /var/lib/jenkins/workspace/maven-freestyle-job
[root@cicd-jenkins-ansible maven-freestyle-job]# ls
pom.xml  README.md  src  target
[root@cicd-jenkins-ansible maven-freestyle-job]# cd target/
[root@cicd-jenkins-ansible target]# ls
Java-war  Java-war.war  maven-archiver
```

### Jenkins集成Ansible

```bash
# 编写ansible主机文件
[root@cicd-jenkins-ansible ~]# su - deploy
(ansible) [deploy@cicd-jenkins-ansible ~]$ vim testservers
[testserver]
dest.deemoprobe.com ansible_user=root
```

- 创建ansible-freestyle-job，使用shell脚本的形式构建

[![28][28]][28]

[28]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304205404.png

```bash
#!/bin/bash

set +x
source /home/deploy/ansible/bin/activate
source /home/deploy/ansible/ansible/hacking/env-setup -q

cd /home/deploy
ansible --version

cat testservers

ansible -i testservers testserver -m command -a "ip addr"
set -x
```

- 构建日志输出

![20220304205506](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304205506.png)

- 至此，Jenkins实践部分已完成

## 综合实践

综合使用Jenkins-Ansible-Gitlab进行实践，环境部署如下图：

![cicd](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/cicd.png)

- 创建gitlab仓库`nginx-playbooks`，克隆到本地后添加项目文件后再提交到远程gitlab仓库，最终结果如图

[![29][29]][29]

[29]: https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304212123.png

```bash
# ansible项目nginx-playbooks结构
[root@cicd-jenkins-ansible ~]# tree nginx-playbooks/
nginx-playbooks/
├── deploy.retry
├── deploy.yml
├── inventory
│   ├── dev
│   └── prod
├── README.md
└── roles
    └── nginx
        ├── files
        │   ├── health_check.sh
        │   └── index.html
        ├── tasks
        │   └── main.yml
        └── templates
            └── nginx.conf.j2
[root@cicd-jenkins-ansible ~]# cd nginx-playbooks/
# 重试主机列表
[root@cicd-jenkins-ansible nginx-playbooks]# cat deploy.retry 
dest.deemoprobe.com
# 部署文件
[root@cicd-jenkins-ansible nginx-playbooks]# cat deploy.yml 
- hosts: "nginx"
  gather_facts: true
  remote_user: root
  roles:
    - nginx
# 主机和参数，对于dev开发环境
[root@cicd-jenkins-ansible nginx-playbooks]# cat inventory/dev 
[nginx]
dest.deemoprobe.com
[nginx:vars]
server_name=dest.deemoprobe.com
port=80
user=deploy
worker_processes=4
max_open_file=65500
root=/www
# 主机和参数，对于prod生产环境
[root@cicd-jenkins-ansible nginx-playbooks]# cat inventory/prod 
[nginx]
dest.deemoprobe.com
[nginx:vars]
server_name=dest.deemoprobe.com
port=80
user=deploy
worker_processes=4
max_open_file=65500
root=/www
# 健康检查脚本
[root@cicd-jenkins-ansible nginx-playbooks]# cat roles/nginx/files/health_check.sh 
#!/bin/sh
URL=$1
curl -Is http://$URL > /dev/null && echo "The remote side is healthy" || echo "The remote side is failed, please check"
# nginx首页文件
[root@cicd-jenkins-ansible nginx-playbooks]# cat roles/nginx/files/index.html 
This is my first website
# jinja2-nginx模板文件
[root@cicd-jenkins-ansible nginx-playbooks]# cat roles/nginx/templates/nginx.conf.j2 
# For more information on configuration, see: 
user              {{ user }};  
worker_processes  {{ worker_processes }};  
  
error_log  /var/log/nginx/error.log;  
  
pid        /var/run/nginx.pid;  
  
events {  
    worker_connections  {{ max_open_file }};  
}  
  
  
http {  
    include       /etc/nginx/mime.types;  
    default_type  application/octet-stream;  
  
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '  
                      '$status $body_bytes_sent "$http_referer" '  
                      '"$http_user_agent" "$http_x_forwarded_for"';  
  
    access_log  /var/log/nginx/access.log  main;  
  
    sendfile        on;  
    #tcp_nopush     on;  
  
    #keepalive_timeout  0;  
    keepalive_timeout  65;  
  
    #gzip  on;  
      
    # Load config files from the /etc/nginx/conf.d directory  
    # The default server is in conf.d/default.conf  
    #include /etc/nginx/conf.d/*.conf;  
    server {  
        listen       {{ port }} default_server;  
        server_name  {{ server_name }};  
  
        #charset koi8-r;  
  
        #access_log  logs/host.access.log  main;  
  
        location / {  
            root   {{ root }};  
            index  index.html index.htm;  
        }  
  
        error_page  404              /404.html;  
        location = /404.html {  
            root   /usr/share/nginx/html;  
        }  
  
        # redirect server error pages to the static page /50x.html  
        #  
        error_page   500 502 503 504  /50x.html;  
        location = /50x.html {  
            root   /usr/share/nginx/html;  
        }  
  
    }  
  
}
# ansible主任务文件
[root@cicd-jenkins-ansible nginx-playbooks]# cat roles/nginx/tasks/main.yml 
- name: Permit traffic in default zone for http service
  ansible.posix.firewalld:
    port: 80/tcp
    zone: public
    permanent: yes
    immediate: yes
    state: enabled

- name: Disable SELinux
  ansible.posix.selinux:
    state: disabled

- name: Create user deploy
  user: name=deploy

- name: Setup nginx yum source
  yum: pkg=epel-release state=latest

- name: Write then nginx config file
  template: src=roles/nginx/templates/nginx.conf.j2 dest=/etc/nginx/nginx.conf

- name: Create nginx root folder
  file: 'path={{ root }} state=directory owner={{ user }} group={{ user }} mode=0755'

- name: Copy index.html to remote
  copy: 'remote_src=no src=roles/nginx/files/index.html dest=/www/index.html mode=0755'

- name: Restart nginx service
  service: name=nginx state=restarted

- name: Run the health check locally
  shell: "sh roles/nginx/files/health_check.sh {{ server_name }}"
  delegate_to: localhost
  register: health_status

- debug: msg="{{ health_status.stdout }}"
```

```bash
# 实测CentOS7-ansible 2.10.17无法直接使用firewall的和selinux插件
# ansible-doc -l | grep selinux  或firewalld  查不到相关记录 
# 从官方最新版本文档<https://docs.ansible.com/ansible/latest/>可以查到
# 使用selinux和firewalld需要安装ansible-posix插件
# 官方推荐的安装方式
ansible-galaxy collection list
ansible-galaxy collection install ansible.posix
# 但报错如下，验证不通过，无法安装
(ansible) [deploy@cicd-jenkins-ansible ~]$ ansible-galaxy collection install ansible.posix
Starting galaxy collection install process
Process install dependency map
ERROR! Unknown error when attempting to call Galaxy at 'https://galaxy.ansible.com/api/': <urlopen error [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate (_ssl.c:1076)>

# 最终在阿里云镜像库找到了CentOS7版本的ansible.posix RPM包，上传到服务器后root安装
[root@cicd-jenkins-ansible ~]# yum install ansible-collection-ansible-posix-1.2.0-1.el7.noarch.rpm
# 进入deploy用户加载ansible后确认，posix插件安装成功。本文主要用到firewalld和selinux
[root@cicd-jenkins-ansible ~]# su - deploy
Last login: Fri Mar  4 23:35:10 CST 2022 on pts/0
(ansible) [deploy@cicd-jenkins-ansible ~]$ ansible-doc -l | grep posix
[WARNING]: packaging Python module unavailable; unable to validate collection
Ansible version requirements
ansible.posix.acl            Set and retrieve file ACL information         
ansible.posix.at             Schedule the execution of a command or script ...
ansible.posix.authorized_key Adds or removes an SSH authorized key         
ansible.posix.firewalld      Manage arbitrary ports/services with firewalld
ansible.posix.mount          Control active and configured mount points    
ansible.posix.patch          Apply patch files using the GNU patch tool    
ansible.posix.seboolean      Toggles SELinux booleans                      
ansible.posix.selinux        Change policy and state of SELinux            
ansible.posix.synchronize    A wrapper around rsync to make common tasks in...
ansible.posix.sysctl         Manage entries in sysctl.conf 
```

> [ansible.posix阿里云镜像库地址](https://developer.aliyun.com/packageSearch?word=ansible.posix)
> [个人仓库备份ansible-collection-ansible-posix-1.2.0-1.el7.noarch.rpm下载链接](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/repo/ansible-collection-ansible-posix-1.2.0-1.el7.noarch.rpm?versionId=CAEQNBiBgID.1JjW.hciIDAwYTQ4OTkxM2IzYzQ5NTc5Y2JiNGUyYTc0MGQ3NmU2)

- Jenkins新建`ansible-nginx-freestyle`任务

![30](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/2022-03-04_2211322.png)

```bash
# 脚本内容
#/bin/sh

set +x
source /home/deploy/ansible/bin/activate
source /home/deploy/ansible/ansible/hacking/env-setup -q

cd $WORKSPACE
ansible --version
ansible-playbook --version

ansible-playbook -i inventory/$deploy_env ./deploy.yml -e project=nginx -e branch=$branch -e env=$deploy_env
```

- Build with Parameters

![20220304214156](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304214156.png)

```bash
# 构建日志信息
Started by user deemoprobe
Running as SYSTEM
Building in workspace /var/lib/jenkins/workspace/ansible-nginx-freestyle
The recommended git tool is: NONE
using credential c7567e8e-a9a3-442e-b033-5486c6d3170f
 > git rev-parse --resolve-git-dir /var/lib/jenkins/workspace/ansible-nginx-freestyle/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://gitlab.deemoprobe.com/root/nginx-playbooks.git # timeout=10
Fetching upstream changes from https://gitlab.deemoprobe.com/root/nginx-playbooks.git
 > git --version # timeout=10
 > git --version # 'git version 1.8.3.1'
using GIT_ASKPASS to set credentials 
 > git fetch --tags --progress https://gitlab.deemoprobe.com/root/nginx-playbooks.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/main^{commit} # timeout=10
Checking out Revision 2999bfb83ee36238344b72fde7b38733a63e6251 (refs/remotes/origin/main)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 2999bfb83ee36238344b72fde7b38733a63e6251 # timeout=10
Commit message: "Update main.yml"
 > git rev-list --no-walk b2c016d6e71aa2e8abd1eaa5131918a1402aeec7 # timeout=10
[ansible-nginx-freestyle] $ /bin/sh -xe /tmp/jenkins3169245191854428687.sh
+ set +x
ansible 2.10.17.post0
  config file = None
  configured module search path = ['/home/deploy/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/deploy/ansible/ansible/lib/ansible
  executable location = /home/deploy/ansible/ansible/bin/ansible
  python version = 3.7.7 (default, Mar  3 2022, 14:46:38) [GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]
ansible-playbook 2.10.17.post0
  config file = None
  configured module search path = ['/home/deploy/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/deploy/ansible/ansible/lib/ansible
  executable location = /home/deploy/ansible/ansible/bin/ansible-playbook
  python version = 3.7.7 (default, Mar  3 2022, 14:46:38) [GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]
[WARNING]: packaging Python module unavailable; unable to validate collection
Ansible version requirements

PLAY [nginx] *******************************************************************

TASK [Gathering Facts] *********************************************************
ok: [dest.deemoprobe.com]

TASK [nginx : Permit traffic in default zone for http service] *****************
changed: [dest.deemoprobe.com]

TASK [nginx : Disable SELinux] *************************************************
[WARNING]: SELinux state change will take effect next reboot
ok: [dest.deemoprobe.com]

TASK [nginx : Create user deploy] **********************************************
ok: [dest.deemoprobe.com]

TASK [nginx : Setup nginx yum source] ******************************************
ok: [dest.deemoprobe.com]

TASK [nginx : Write then nginx config file] ************************************
ok: [dest.deemoprobe.com]

TASK [nginx : Create nginx root folder] ****************************************
ok: [dest.deemoprobe.com]

TASK [nginx : Copy index.html to remote] ***************************************
ok: [dest.deemoprobe.com]

TASK [nginx : Restart nginx service] *******************************************
changed: [dest.deemoprobe.com]

TASK [nginx : Run the health check locally] ************************************
changed: [dest.deemoprobe.com -> localhost]

TASK [nginx : debug] ***********************************************************
ok: [dest.deemoprobe.com] => {
    "msg": "The remote side is healthy"
}

PLAY RECAP *********************************************************************
dest.deemoprobe.com        : ok=11   changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

Finished: SUCCESS

# 查看任务目录
[root@cicd-jenkins-ansible ~]# cd /var/lib/jenkins/workspace/ansible-nginx-freestyle
[root@cicd-jenkins-ansible ansible-nginx-freestyle]# ls
deploy.retry  deploy.yml  inventory  README.md  roles
[root@cicd-jenkins-ansible ansible-nginx-freestyle]# tree ./
./
├── deploy.retry
├── deploy.yml
├── inventory
│   ├── dev
│   └── prod
├── README.md
└── roles
    └── nginx
        ├── files
        │   ├── health_check.sh
        │   └── index.html
        ├── tasks
        │   └── main.yml
        └── templates
            └── nginx.conf.j2
# 访问目标主机的nginx服务
(ansible) [deploy@cicd-jenkins-ansible ~]$ curl dest.deemoprobe.com
This is my first website
(ansible) [deploy@cicd-jenkins-ansible ~]$ curl -I dest.deemoprobe.com
HTTP/1.1 200 OK
Server: nginx/1.20.2
Date: Fri, 04 Mar 2022 15:45:46 GMT
Content-Type: text/html
Content-Length: 25
Last-Modified: Fri, 04 Mar 2022 14:03:23 GMT
Connection: keep-alive
ETag: "62221c2b-19"
Accept-Ranges: bytes
```

- Windows浏览器访问

![20220304234504](https://deemoprobe.oss-cn-shanghai.aliyuncs.com/images/20220304234504.png)

至此，基于Jenkins-Ansible-Gitlab的CICD全流程实践完成，总体感受是知识概念都不是很难，但需要关注的细节很多，有所收获。此文纪念。
