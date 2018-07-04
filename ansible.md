---
title: ansible 一键安装nginx+php7并部署全栈https
---

## 实现效果

我在本地测试好了网站(nginx+php7)，申请了一台云服务器，一条命令将网站部署到裸机上并实现全站 https.

## ansible 是什么

根据官网的描述:

> Ansible is a radically simple IT automation engine that automates cloud provisioning, configuration management, application deployment, intra-service orchestration, and many other IT needs.

ansible 是一个开源的非常简单的IT自动化运维工具，基于 Python 开发(Python2,Python3均支持)，实现了自动化配置管理，自动化应用部署等很多类似的IT需求。 Ansible 的主要产品有 ansible cli 和 ansible tower (贵，企业土豪专用)。Ansible cli 中常用的有:

* ansible,用来执行一个 ansible 任务，适合极简单的任务或者调试任务
* ansible-playbook，用来执行 playbook 中定义的任务集，配合 ansible role 可以实现复杂的运维功能
* ansible-doc,用来防忘记，查询文档
* ansible-galaxy，用来创建，下载角色

## ansible 解决了什么问题

### 我为什么要用 ansible

在孵化部时候，游戏的官网服务器以及游戏服务器架在公司的北京机房。上线内测前，我们一致觉得公司的机房速度太慢，于是申请了一个月的腾讯云上海试用，并立即在新机器上配置环境(安装nginx,以及一些依赖库)，将服务器迁到腾讯云。腾讯云到期后，我把服务器迁回北京机房。过了几天又得知上海机房有新机器了，于是我再把服务器从北京机房迁到上海机房。每次迁移时候要做的事情都是:

```bash
# 配置源
cp /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
wget http://mirrors.aliyun.com/centos/7/CentOS-Base.repo -O /etc/yum.repos.d/CentOS-Base.repo
yum update
yum install nginx docker ...
# 拷贝nginx配置,网站文件
scp xxx@ip:/var/www/webroot /var/www
# 创建用户
adduser xxx
passwd xxx
# 给用户添加 sudoers 权限
nano /etc/sudoers
xxx ALL=(ALL) ALL
```

同样的命令，同样的流程，敲2遍的时候，都已经快吐了！！敲完第3遍，简直不能忍，于是在网上查运维自动化工具，找到了ansible并且在一个无人的周末，倒腾了一个一键建站的 ansible playbook.申请了一台阿里云，然后使用命令 `ansible-playbook -i ./hosts -v --ask-pass site.yml` 一键搞定可访问的 https 网站。爽！！
<!-- more -->

### ansible 解决的问题

1. 重复敲命令敲到吐
1. 人工敲命令出错(比如更改了Centos-Base.repo但忘记备份，导致yum update失败，而使用ansible的copy模块会自动备份Centos-Base.repo文件；比如敲了100次同样的命令敲成sudo rm -rf /)
1. 模块化(比如下面的需求),现成的功能齐全的模块很多
    ```bash
    # shell脚本
    # 判断一个目录是否存在,不存在则创建
    if [ ! -d "/tmp/log" ]; then
        mkdir /tmp/log
    fi
    ```
    ```yaml
    # ansible 判断目录是否存在,不存在则创建权限为 0755 的目录
    file:
      path: /tmp/log
      state: directory
      mode: 0755
    ```
1. 标准化,简单化,基本上所有功能都是通过 **module** 来实现的，每个 **module** 的使用方法都是在 yaml 中配置
1. Agentless, ansible是基于ssh协议(或rpc等连接协议)与目标机器通信，不需要在目标机器上预先启动 ansible 进程，而自动化运维工具 puppet(saltStack) 则需要在目标机器上安装好 puppet(slatStack) 作为 Agent

## ansible 工作原理

在没有 ansible 之前，想在远程机器上执行一条指令 `ls`，通常的做法是:

> ssh -o User=username 192.168.93.223 ls

Ansible 是基于 paramiko 开发的， paramiko 是一个纯 Python 实现的 ssh 协议库, ansible 本质上也是基于 ssh 协议,在目标机器上执行了一系列的命令。现在的 ansible 已经支持多种协议，除了ssh协议,还有 netconf协议, eos协议(参照 ansible 的 network 模块) 等。

![how ansible works](http://7xp1jz.com1.z0.glb.clouddn.com/blog/ansible/how_ansibe_works.png "how ansible works")

Ansible 在本机上,将 ansible module(一堆python代码文件) 通过 scp 协议推送到目标机器节点上，然后执行这些 python 文件，并在执行完成后删除这些 python 文件。

如果目标机器上么有安装Python怎么办？

Ansible 的 raw/shell 模块是直接基于 ssh连接（netconf等) 在目标机器上执行命令,相当于 `ssh username@host "sudo apt-get install python"`，命令如下：

```bash
ansible -i ansible_hosts remoteserver -m raw -a "test -e /usr/bin/python || (apt -y update && apt install -y python-minimal)"
```

### ansible vs docker

假设要部署一个 https 网站，ansible 和 docker 都可以用：

1. ansible, 将安装 nginx, php7, 更改配置，传文件 的命令，都翻译成 ansible 的任务，依次执行就得到了一个线上可运行的 https 网站；或者在别人写好的 ansible role 基础上，修改成自己的 role 来实现部署功能
1. docker, 选个 ubuntu16.04 作为 base container, 在容器内安装 nginx,php7 等，最后 docker save 保存下镜像,docker run -v 做下磁盘映射，以后部署类似的 https 网站都可以用；或者在别人写好的 Dockerfile 基础上，修改成自己的 Dockerfile

什么时候用 ansible ? 什么时候用 docker?

* 要部署的机器很少(没有明确数量定义，根据测试来决定)，比如只有1台，ansible docker 甚至自己写的 shell 脚本都可以用，哪个顺手哪个来！
* 要部署的机器多，比如50台及以上，用 docker 加载镜像比 ansible 一台台执行命令安装要快得多，此时 ansible 可以用来执行 docker load 指令

什么时候只能用 ansible ?

要配置宿主机环境时，只能用 ansible 或者 ssh 上去手打命令，比如使用 iptables 配置防火墙策略，修改 DNS 解析配置，安装 docker 等。

更多的时候，还是将 ansible 和 docker 结合起来使用，用ansible 来配置 docker 安全运行起来所需要的宿主机环境, 用 docker 来跑应用实例。

## ansible 安装

Ansible 其实就是一个 python 包,怎么安装 python package ? 可以通过系统的包管理直接安装，也可以在 virtualenv 里安装，这里使用的是 virtualenv 在ubuntu 18.04 下安装:

```bash
sudo apt-get install python3-pip
pip3 install virtualenvwrapper
mkdir ~/.virtualenvs

vim ~/.bashrc, 添加如下的行：
export WORKON_HOME=~/.virtualenvs
VIRTUALENVWRAPPER_PYTHON='/usr/bin/python3'
source /usr/local/bin/virtualenvwrapper.sh

source ~/.bashrc

# Create virtualenv
mkvirtualenv ansible
# Activate/switch to a virtualenv
workon ansible
pip3 install ansible
# update ansible
pip3 install -U ansible
# Deactivate virtualenv
deactivate ansible
```

## ansible 基础命令

### ip清单 [inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html)

Ansible 使用ini文件来表示其所管理的机器列表，该ini文件可以分组来配置待管理的机器。一个简单的 inventory 文件如下,文件名可以任意取，一般命名为 ansible_hosts：

```text
192.168.1.1
[ttt]
192.168.1.2

[webserver]
# 冒号加端口号来指定该机器上的 ssh 端口
web.com:2000
# 批量添加类似模式的主机名
web[01:50].com
# 可以给某个host添加变量，该变量可以被 playbook 所使用
host1.com http_port=888 maxRequest=1024

# 可以给整个组添加环境变量
[webserver:vars]
proxy=aaa.com:222

[dbserver]
db-[a-f].com

[allserver]
webserver
dbserver
```

简单的 ping 命令如下：

```bash
$ ansible -i ./ansible_hosts ttt -m ping
192.168.1.2 | success >> {
    "changed": false,
    "ping": "pong"
}
```

### ansible 常用模块举例

ping

```bash
giant@localhost:~$ workon ansible
(ansible) giant@localhost:~$ ansible-doc -s ping
- name: Try to connect to host, verify a usable python and return `pong' on success
  ping:
      data:                  # Data to return for the `ping' return value. If this parameter is set to `crash', the module will cause an exception.
(ansible) giant@localhost:~$ ansible -i ./hosts --connection=local local -m ping 
127.0.0.1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
(ansible) giant@localhost:~$ ansible -i ./hosts --connection=local local -m ping -a "data=areyouok"
127.0.0.1 | SUCCESS => {
    "changed": false,
    "ping": "areyouok"
}
(ansible) giant@localhost:~$ ansible -i ./hosts --connection=local local -m ping -a "data=crash"
An exception occurred during task execution. To see the full traceback, use -vvv. The error was: Exception: boom
127.0.0.1 | FAILED! => {
    "changed": false,
    "module_stderr": "Traceback (most recent call last):\n  File \"/tmp/ansible_70L6IW/ansible_module_ping.py\", line 84, in <module>\n    main()\n  File \"/tmp/ansible_70L6IW/ansible_module_ping.py\", line 74, in main\n    raise Exception(\"boom\")\nException: boom\n",
    "module_stdout": "",
    "msg": "MODULE FAILURE",
    "rc": 1
}
```

shell

```bash
(ansible) giant@localhost:~$ ansible-doc -s shell
- name: Execute commands in nodes.
  shell:
      chdir:                 # cd into this directory before running the command
      creates:               # a filename, when it already exists, this step will *not* be run.
      executable:            # change the shell used to execute the command. Should be an absolute path to the executable.
      free_form:             # (required) The shell module takes a free form command to run, as a string.  There's not an actual option named "free form".  See the examples!
      removes:               # a filename, when it does not exist, this step will *not* be run.
      stdin:                 # Set the stdin of the command directly to the specified value.
      warn:                  # if command warnings are on in ansible.cfg, do not warn about this particular line if set to no/false.
(ansible) giant@localhost:~$ ansible -i ./hosts --connection=local local -m shell -a "ls -l|tail -n 1"
127.0.0.1 | SUCCESS | rc=0 >>
-rw-rw-r--  1 giant giant   65 Jun 29 04:12 test.sh
```

copy

copy 模块有个令人喜欢的 backup 参数，backup=true 时, ansible 会备份将被替代的文件并加上时间戳，方便搞砸了找回原始文件。比如切换 centos 的源为阿里云时候，一般都会执行备份命令 `mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup`. Copy 模块的文档也很长，这里就不粘贴了，试着使用下即可。

```bash
(ansible) giant@localhost:~$ ansible -i ./hosts --connection=local    local -m copy -a "src=./hosts dest=/home/giant/test.sh backup=yes"
127.0.0.1 | SUCCESS => {
    "backup_file": "/home/giant/test.sh.17679.2018-07-02@07:43:56~",
    "changed": true,
    "checksum": "88890b0ced9481b1a8c4f5f9316220d654465e35",
    "dest": "/home/giant/test.sh",
    "gid": 1000,
    "group": "giant",
    "md5sum": "16ce004b12a72177c93546656a54f092",
    "mode": "0664",
    "owner": "giant",
    "size": 18,
    "src": "/home/giant/.ansible/tmp/ansible-tmp-1530517436.197045-222926130794904/source",
    "state": "file",
    "uid": 1000
}
(ansible) giant@localhost:~$ ansible -i ./hosts --connection=local    local -m shell -a "ls /home/giant/test.sh*"
127.0.0.1 | SUCCESS | rc=0 >>
/home/giant/test.sh
/home/giant/test.sh.17679.2018-07-02@07:43:56~
```

apt

apt 模块的文档太长，这里就不单独贴出来了。

```bash
(ansible) giant@localhost:~$ ansible -i ./hosts  -u root  --ask-pass local -m apt -a "name=subversion"
SSH password: 
127.0.0.1 | SUCCESS => {
    "cache_update_time": 1530429044,
    "cache_updated": false,
    "changed": true,
    "stderr": "",
    "stderr_lines": [],
    "stdout": "Reading package lists...\nBuilding dependency tree...\nReading state information...\nThe following packages were automatically installed and are no longer required:\n  libffi-dev libomp-dev libomp5 libtinfo-dev llvm-6.0 llvm-6.0-dev\n  llvm-6.0-runtime\nUse 'apt autoremove' to remove them.\nThe following additional packages will be installed:\n  libapr1 libaprutil1 libserf-1-1 libsvn1\nSuggested packages:\n  db5.3-util libapache2-mod-svn subversion-tools\nThe following NEW packages will be installed:\n  libapr1 libaprutil1 libserf-1-1 libsvn1 subversion\n0 upgraded, 5 newly installed, 0 to remove and 201 not upgraded.\nNeed to get 2237 kB of archives.\nAfter this operation, 9910 kB of additional disk space will be used.\nGet:1 http://mirrors.linode.com/ubuntu bionic/main amd64 libapr1 amd64 1.6.3-2 [90.9 kB]\nGet:2 http://mirrors.linode.com/ubuntu bionic/main amd64 libaprutil1 amd64 1.6.1-2 [84.4 kB]\nGet:3 http://mirrors.linode.com/ubuntu bionic/universe amd64 libserf-1-1 amd64 1.3.9-6 [44.4 kB]\nGet:4 http://mirrors.linode.com/ubuntu bionic/universe amd64 libsvn1 amd64 1.9.7-4ubuntu1 [1183 kB]\nGet:5 http://mirrors.linode.com/ubuntu bionic/universe amd64 subversion amd64 1.9.7-4ubuntu1 [834 kB]\nFetched 2237 kB in 0s (33.6 MB/s)\nSelecting previously unselected package libapr1:amd64.\r\n(Reading database ... \r(Reading database ... 5%\r(Reading database ... 10%\r(Reading database ... 15%\r(Reading database ... 20%\r(Reading database ... 25%\r(Reading database ... 30%\r(Reading database ... 35%\r(Reading database ... 40%\r(Reading database ... 45%\r(Reading database ... 50%\r(Reading database ... 55%\r(Reading database ... 60%\r(Reading database ... 65%\r(Reading database ... 70%\r(Reading database ... 75%\r(Reading database ... 80%\r(Reading database ... 85%\r(Reading database ... 90%\r(Reading database ... 95%\r(Reading database ... 100%\r(Reading database ... 161650 files and directories currently installed.)\r\nPreparing to unpack .../libapr1_1.6.3-2_amd64.deb ...\r\nUnpacking libapr1:amd64 (1.6.3-2) ...\r\nSelecting previously unselected package libaprutil1:amd64.\r\nPreparing to unpack .../libaprutil1_1.6.1-2_amd64.deb ...\r\nUnpacking libaprutil1:amd64 (1.6.1-2) ...\r\nSelecting previously unselected package libserf-1-1:amd64.\r\nPreparing to unpack .../libserf-1-1_1.3.9-6_amd64.deb ...\r\nUnpacking libserf-1-1:amd64 (1.3.9-6) ...\r\nSelecting previously unselected package libsvn1:amd64.\r\nPreparing to unpack .../libsvn1_1.9.7-4ubuntu1_amd64.deb ...\r\nUnpacking libsvn1:amd64 (1.9.7-4ubuntu1) ...\r\nSelecting previously unselected package subversion.\r\nPreparing to unpack .../subversion_1.9.7-4ubuntu1_amd64.deb ...\r\nUnpacking subversion (1.9.7-4ubuntu1) ...\r\nSetting up libapr1:amd64 (1.6.3-2) ...\r\nProcessing triggers for libc-bin (2.27-3ubuntu1) ...\r\nSetting up libaprutil1:amd64 (1.6.1-2) ...\r\nProcessing triggers for man-db (2.8.3-2) ...\r\nSetting up libserf-1-1:amd64 (1.3.9-6) ...\r\nSetting up libsvn1:amd64 (1.9.7-4ubuntu1) ...\r\nSetting up subversion (1.9.7-4ubuntu1) ...\r\nProcessing triggers for libc-bin (2.27-3ubuntu1) ...\r\n"
```

### ansible 命令行例子

ansible \<host-pattern\> [options]

比较常用的选项有 `-m MODULE_NAME -a MODULE_ARGS -i INVENTORY`

ansible 需要指定一个 ansible 模块，如果不指定模块，默认为 command 模块。

记不住每个模块的参数怎么办？使用 ansible-doc -s MODULE_NAME 查看相应模块的文档；记不住模块怎么办？那就 google 下吧。。

比如要安装 nginx, 可以使用 shell 模块来安装，也可以使用 apt 模块来安装：

```bash
# 使用 shell 模块安装 nginx
# --connection=local 告诉 ansible 不需要通过ssh连接来执行命令，直接在本机器运行即可
ansible  localhost --connection=local -b --become-user=root -m shell -a 'apt-get install nginx'

# 在目标机器上安装 nginx
ansible -i ./hosts ttt -b --become-user=root  -m shell -a 'apt-get install nginx'

# 使用 apt 模块安装 nginx
ansible -i ./hosts ttt -b --become-user=root -m apt -a 'name=nginx update_cache=true'

192.168.1.2 | success >> {
    "changed": false
}
# 对于centos则使用yum模块,ansible -i ./hosts ttt -b --become-user=root -m yum -a 'name=nginx update_cache=true'
```

## ansible playbook

Playbook 是 ansible 的配置，部署，自动化语言。Playbook 可以用来描述 你想让远程主机执行的策略，
也可以描述IT运维过程中的一系列步骤。

假如想执行多条指令，可以使用 bash 脚本，每行单独使用 ansible -m -a 来执行，
也可以使用 ansible playbook 将需执行的任务一起放在同一个文件中。
但 playbook 中还可以设置 handler 来执行某项任务完成后的回调，与监控服务器和负载均衡服务器交互，总之使用 playbook 比自己写 bash scripts 要功能强大得多。

Playbook 使用 yaml 格式来描述具体的任务(task)，playbook 中的每个 play 可以有以下参数：

* hosts,指定在哪些机器上执行该 play 中的一系列任务,比如 hosts: webservers:dbservers
* remote_user, 指定该 play 中的任务以什么用户运行，比如 remote_user:root,可在各任务中单独指定
* become,become_method,become_user 可用来指定是否切换用户执行指令，比如 become_user:root, become_method:sudo,可在各任务中单独指定。注意，如果在远程机器上 sudo 需要密码，那么运行 ansible 时需要加参数 --ask-become-pass。

比如安装 nginx，可以使用下面的 playbook(文件名为nginx.yml):

```text
- hosts: ttt
  remote_user : normal_user
  become: yes
  become_user: root
  tasks:
   - name: Install Nginx
     apt:
       name: nginx
       update_cache: true
```

使用 ansible-playbook 执行上面的任务结果如下:

```text
$ ansible-playbook -i ./ansible_hosts nginx.yml

PLAY [ttt] ******************************************************************

GATHERING FACTS ***************************************************************
ok: [192.168.1.2]

TASK: [Install Nginx] *********************************************************
ok: [192.168.1.2]

PLAY RECAP ********************************************************************
192.168.1.2                  : ok=2    changed=0    unreachable=0    failed=0
```

Playbook 中可以定义 handler, handler 和 task 一模一样,但有一点不一样，handler 只能被 task 调用时才会执行。
可以把 handler 认为是事件系统的一部分，只有当 handler 监听的事件发生时， handler 才会执行。
handler 比较适合在运行特定的 task 后执行额外的操作，比如安装完 nginx 后，需要启动 nginx；nginx的网站文件更改后，重新加载 nginx。
比如在安装完 nginx 后重启 nginx 的 playbook 如下:

```text
- hosts: ttt
  become: yes
  become_user: root
  tasks:
   - name: Install Nginx
     apt:
       name: nginx
       update_cache: true
     # 使用 notify 来通知名为 Start Nginx 的 handler
     notify:
      - Start Nginx

  handlers:
   - name: Start Nginx
     service:
       name: nginx
       state: started
```

使用 ansible-playbook 运行上面的 playbook 的结果为：

```text
$ ansible-playbook -i ./hosts nginx.yml

PLAY [ttt] ******************************************************************

GATHERING FACTS ***************************************************************
ok: [192.168.1.2]

TASK: [Install Nginx] *********************************************************
ok: [192.168.1.2]

NOTIFIED: [nginx | Start Nginx] ***********************************************
ok: [192.168.1.2]

PLAY RECAP ********************************************************************
192.168.1.2                  : ok=2    changed=0    unreachable=0    failed=0
```

将安装配置 nginx 的一系列命令，一行行地翻译为 ansible playbook, 容易出错还很累。为何不用 nginx 官方人员写好的呢？？！ Nginx 官方将ubuntu,centos,redhat等安装 nginx 的任务，以及需要的配置文件都集合在一块，组合成了 ansible nginx role.

## 使用 ansible roles 配环境+部署

Ansible 角色(ansible role)是用来（根据既定的文件结构）自动化加载指定变量，文件，任务以及handlers，它很好地将一系列小任务和数据组织在一起。
Ansible 角色也有助于和别人共享角色，更快地实现自动化部署分发。
Ansible 角色的目录结构如下：

```text
site.yml
webservers.yml
fooservers.yml
roles/
   common/
     tasks/
     handlers/
     files/
     templates/
     vars/
     defaults/
     meta/
   webservers/
     tasks/
     defaults/
     meta/
```

Ansible 在角色的每个子目录下，会搜索一个名为 main.yml 的文件，该文件需包含相关的内容，具体如下：

* `tasks`,包含角色要执行的任务列表
* `handlers`,包含handlers,可以被该角色甚至其他角色使用
* `defaults`,该角色的默认变量
* `vars`,该角色要用到的其他变量
* `files`,该角色部署用到的文件
* `templates`,该角色部署用到的模板
* `meta`,定义了该角色的一些元数据，比如作者，描述等..

### 创建角色

使用命令 `ansible-galaxy` 可以创建新角色，比如现在要创建一个一键部署https的角色，可以使用命令 `ansible-galaxy init https1` 来新建一个 https1 的角色。

```bash
(ansible) giant@localhost:~/ansible_test/roles$ mkdir ~/ansible_test
(ansible) giant@localhost:~/ansible_test/roles$ cd ~/ansible_test && mkdir roles && cd roles
(ansible) giant@localhost:~/ansible_test/roles$ ansible-galaxy init https1
(ansible) giant@localhost:~/ansible_test/roles$ ansible-galaxy init https1
- https1 was created successfully
(ansible) giant@localhost:~/ansible_test/roles$ ls https1/ -l
total 36
drwxrwxr-x 2 giant giant 4096 Jun 25 09:41 defaults
drwxrwxr-x 2 giant giant 4096 Jun 25 09:41 files
drwxrwxr-x 2 giant giant 4096 Jun 25 09:41 handlers
drwxrwxr-x 2 giant giant 4096 Jun 25 09:41 meta
-rw-rw-r-- 1 giant giant 1328 Jun 25 09:41 README.md
drwxrwxr-x 2 giant giant 4096 Jun 25 09:41 tasks
drwxrwxr-x 2 giant giant 4096 Jun 25 09:41 templates
drwxrwxr-x 2 giant giant 4096 Jun 25 09:41 tests
drwxrwxr-x 2 giant giant 4096 Jun 25 09:41 vars
```

### 使用角色

**角色哪里来？** 可以按上述方法自己创建一个角色。也可以从 ansible galaxy 网站上下载别人写好的角色，然后自己改改变量的默认值，加些任务就可用了。

**角色目录放在哪里？** Ansible 将会按照下列顺序搜寻角色：

1. 从 playbook 同级的 roles 目录下搜索角色
1. 从 /etc/ansible/roles 目录下搜索角色

Ansible 1.4 及以后还支持自定义 ansible 角色搜寻路径，在 /etc/ansible/ansible.cfg 中配置 roles_path.

**角色的使用方法是什么？** 在 playbook 的 yaml 中直接填写 roles 属性，示例如下：

```text
- name: Configure app server(s)
  hosts: webserver
  remote_user: root  

  roles:
    - { role: install_nginx }
    - { role: install_php }
```

**Playbook 中添加了 roles 后执行顺序**(参照[官网](https://docs.ansible.com/ansible/2.5/user_guide/playbooks_reuse_roles.html#using-roles)):

1. Playbook 中定义的所有 **pre_tasks**
1. 所有被触发的 handlers
1. Each role listed in roles will execute in turn. Any role dependencies defined in the roles meta/main.yml will be run first, subject to tag filtering and conditionals.
1. Any tasks defined in the play.
1. Any handlers triggered so far will be run.
1. Any post_tasks defined in the play.
1. Any handlers triggered so far will be run.

### ansible 变量,定义与使用

ansible 变量是由 字母，数字和下划线 组成的字符序列，必须以字母开头。变量可以像上文一样在 inventory 中定义，也可以在 playbook 中定义。在角色的 vars/main.yml 中可按如下的方式定义变量：

```text  
software_name: nginx
foo:
  field1: true
  field2: latest
```

使用的时候，在 tasks/main.yml 中，可以按如下方式使用：

```text
- name: 安装nginx
  yum:
    name: "{{software_name}}"
    state: "{{foo['field2']}}"
    update_cache: "{{foo.field1}}"
```

加引号的原因是 yaml 会把以大括号开头的属性，识别为 yaml 里的字典结构。
可以使用中括号加属性名来索引，可以使用 点属性名 来索引；当使用 foo.field1 方式索引时，要避免和 python 对象的属性冲突。
如果不确定是否冲突，那最好使用中括号+属性名来索引。

```text
{
    "ansible_all_ipv4_addresses": [
        "REDACTED IP ADDRESS"
    ],
    "ansible_all_ipv6_addresses": [
        "REDACTED IPV6 ADDRESS"
    ],
    "ansible_architecture": "x86_64",
    "ansible_bios_date": "09/20/2012",
    "ansible_bios_version": "6.00",
    "ansible_cmdline": {
        "BOOT_IMAGE": "/boot/vmlinuz-3.5.0-23-generic",
        "quiet": true,
        "ro": true,
        "root": "UUID=4195bff4-e157-4e41-8701-e93f0aec9e22",
        "splash": true
    },
```

上面是运行命令 `ansible localhost -m setup`的部分结果，返回的属性字典被称为 ansible facts,是 ansible 和主机通信返回的信息。该字典的每个 key 都可以直接在 ansible playbook 中使用， 比如获取系统的架构可以使用 `{{ansible_architecture}}`。

### ansible 变量优先级

不用全部优先级都记住，到用的时候去查[官方文档](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable)即可。重点关注下： Playbook 中定义的变量优先级高于 roles/default/main.yml 中的变量。这样可以在 playbook 中重载 roles 中定义的变量。

### ansible templates

Ansible 使用 jinja2 模板引擎来支持动态表达式和变量访问。 Ansible 扩展了 jinja2 filters 的数目，并且增加了一个新类型 lookup，但这些我们不需要特别关心，也不需要记住，用的时候 google 下特定功能的 filter 即可吗，或者去 [ansible templating](https://docs.ansible.com/ansible/2.5/user_guide/playbooks_templating.html) 页面去查询。

Ansible 模板解析发生在将任务发送至目标机器之前，这是为了降低目标机器门槛，目标机器不需要安装 jinja2.

### https 证书创建(privkey.pem,fullchain.pem)

在尝试自己写创建 https 证书的 ansible playbook 之前，记得先在 google 上输入 ansible letsencrypt 查看别人已经写好的 ansible roles, 如果有满足需求的，那就直接拿来用。

假设查不到满足需求的 ansible role, 那么自己如何写呢？

**Update:20180702 搜索了下，[github](https://github.com/systemli/ansible-role-letsencrypt/blob/master/README.md) 上已经有别人写好的 ansible 使用 http challenge 申请 lets encrypt 的角色**

https证书创建使用 certbot 的 standalone 模式，需要在 godaddy 之类的网站DNS管理界面，在 host records 中添加一条 A Record, Host=@,ip=目标机器IP,如下图:

![dns manage](http://7xp1jz.com1.z0.glb.clouddn.com/blog/ansible/dnsM.png "dns manage")

参考 Digital Ocean 的 [certbot standalone certificate retrieve](https://www.digitalocean.com/community/tutorials/how-to-use-certbot-standalone-mode-to-retrieve-let-s-encrypt-ssl-certificates)教程，整篇文章的步骤汇总如下：

```bash
# 安装certbot
giant@localhost$ sudo apt-get update
giant@localhost$ sudo apt-get install software-properties-common
giant@localhost$ sudo add-apt-repository ppa:certbot/certbot
giant@localhost$ sudo apt-get update
giant@localhost$ sudo apt-get install python-certbot-nginx
# 打开防火墙 80 443,运行certbot,certbot将会使用内置的web服务器
giant@localhost$ sudo ufw allow 80
giant@localhost$ sudo ufw allow 443
giant@localhost$ sudo certbot certonly --standalone --preferred-challenges http -d example.com
# certbot 在ubuntu上安装的时候，已经自动添加了一个 renew 脚本在 /etc/cron.d/ 目录下
# 当90天过期后，https证书将会被重新renew，这里创建一个 renew_hook 脚本，在 renew 后自动重新加载 nginx 的脚本
giant@localhost$ sudo nano /etc/letsencrypt/renewal/example.com.conf
# 在 example.com.conf 的最后一行加上
renew_hook = systemctl reload nginx
# 最后使用 dry-run 测试下 renew 功能
giant@localhost$ sudo certbot renew --dry-run
# 测试成功后，也可以不使用ubuntu官方的脚本，自己写个shell脚本测试,参照 [StackOverflow](https://stackoverflow.com/questions/42300579/letsencrypt-certbot-multiple-renew-hooks)
#!/bin/bash
#sudo certbot renew --renew-hook "service postfix reload" --renew-hook "service dovecot restart" --renew-hook "service apache2 reload" -q >> /var/log/certbot-renew.log | mail -s "CERTBOT Renewals" me@myemail.com  < /var/log/certbot-renew.log
```

将上述指令，依次翻译成 ansible 的模块语言, 可以翻译为:

```text
- hosts : local
  remote_user : root
  vars:
      domain: cxz.sh
  tasks:
    - name : install software-properties-common
      apt:
          name : software-properties-common
          state : latest
          update_cache : yes
    - name : add certbot ppa
      apt_repository :
        repo : ppa:certbot/certbot
    - name : install python certbot
      apt:
          name : python-certbot-nginx
          state : latest
          update_cache : yes
    - name : ufw allow 80 443 for verifying
      ufw:
          rule : allow
          port : '{{item}}'
      with_items:
          - 80
          - 443
    - name : run certbot http challenge
      command : certbot certonly --standalone --preferred-challenges http -d {{domain}}
    - name : append renew hook task to end of renewal file
      lineinfile:
          path : /etc/letsencrypt/renewal/{{domain}}.conf
          line : 'renew_hook = systemctl reload nginx'
```

最终得到一个可以名为 create_lets.yaml 的 playbook，该 playbook 的功能是申请免费的 LetsEncrypt https 证书，并添加定时任务来 renew 证书并重启nginx。

### 安装 nginx+php7

想在目标服务器上运行起 https 服务器，需要下列步骤：

1. 目标机器上环境配置,比如安装 nginx,php等
1. 将网站文件拷贝过去

安装 nginx 推荐使用 nginx 官方提供的 [ansible role](https://github.com/nginxinc/ansible-role-nginx)，当然也可以自己尝试将以前安装 nginx 的命令翻译成 ansible playbook。
推荐使用 nginx 官方提供的角色，功能全，内容多。记住根据自己的需要，重载默认变量。

安装 php7 可以直接按照 google 出的教程安装，也可以使用别人已经测试好的 ansible 角色直接安装。同样推荐使用写好的角色安装，一是别人写好的角色可能功能更全，二是省得自己手写一遍功能类似的 playbook。

将网站目录整个拷贝过去，可以直接使用 copy 模块：

```bash
- name: create root directory of website on remote server
  file:
      path: '{{ remote_website_root }}'
      mode: 0755
      state : directory
      owner : '{{ website_user }}'
      group : '{{ website_user }}'
- name: copy contents of local website root directory to remote server
  copy:
      src: '{{ local_website_root }}'
      dest : '{{ remote_website_root }}'
      owner : '{{ website_user }}'
      group : '{{ website_user }}'
```

## conclusion

之前，每次换服务器，要自己 root 登录上去，配置源，安装 nginx,php7,修改配置，拷贝网站，重新启动，很烦。

而现在：当网站要换服务器时，我只需要去 godaddy 的 DNS 管理界面将 A Record 改为新服务器地址，敲一行命令 `ansible-playbook -i ./hosts --ask-pass deploy_website.yml`，然后开始喝茶等着 https 网站部署好。

## references

1. [ansible解决了什么](https://showme.codes/2017-06-12/ansible-introduce/)
1. [how ansible works](https://www.ansible.com/overview/how-ansible-works)
1. [Ansible architecture ppt](http://afnog.github.io/sse/ansible/Ansible_Presentation.pptx)
1. [an ansible 2 tutorial](https://serversforhackers.com/c/an-ansible2-tutorial)
1. [certbot standalone certificate retrieve](https://www.digitalocean.com/community/tutorials/how-to-use-certbot-standalone-mode-to-retrieve-let-s-encrypt-ssl-certificates)
1. [StackOverflow](https://stackoverflow.com/questions/42300579/letsencrypt-certbot-multiple-renew-hooks)
1. [ansible vs docker](https://www.ansible.com/integrations/containers/docker)
1. [INTEGRATION:ANSIBLE AND DOCKER](https://www.ansible.com/integrations/containers/docker)
1. [SO,difference between ansible playbook and docker](https://stackoverflow.com/questions/30550378/what-is-the-difference-between-a-docker-container-and-an-ansible-playbook)
1. [Quora,difference between ansible playbook and docker](https://www.quora.com/What-is-the-difference-between-Ansible-and-Docker)
1. [20分钟入门ansible](https://mp.weixin.qq.com/s/2G4LrSOo2xlT_5b3j02PZw)
1. [变量优先级](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable)
1. [playbook中角色执行顺序](https://docs.ansible.com/ansible/2.5/user_guide/playbooks_reuse_roles.html#using-roles)
