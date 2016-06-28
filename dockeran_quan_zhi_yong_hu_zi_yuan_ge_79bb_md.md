## docker进行资源隔离的6种namespace

|namespace|隔离内容|内核版本|
|---|---|---|
|UTS|主机名与域名|Linux 2.6.19|
|IPC|信号量，消息队列和共享内存|Linux 2.6.19|
|PID|进程编号|Linux 2.6.24|
|Network|网络设备,网络栈，端口|始于Linux 2.6.24 完成于 Linux 2.6.29|
|Mount|文件挂载|Linux 2.4.19|
|User|用户用户组|	始于 Linux 2.6.23 完成于 Linux 3.8|

**其中User namespace是从docker1.10开始被支持,并且不是默认开启的.**

以上内容不是这篇文章的重点，此篇文章主要是介绍user namespace

## 一.User namespace的作用

docker 使用namespace进行资源隔离,其中一种是user namespace.user namespace主要隔离了安全相关的标识符和属性，包括用户ID,用户组Id，root目录，key(密钥)以及特殊权限.

默认的情况下，docker容器使用的root用户和宿主机的root用户是同一个用户，尽管可以限制容器内root用户的权限(capability)，但本质上仍然和宿主机root用户是同一个用户.

有了user namespace之后，我们就可以将宿主机上的普通用户映射为容器的root用户,这样容器中的实际用户为普通用户权限，可以将容器的安全程度提高一个等级！

### 实验一：不使用user namespace进行资源隔离

* 运行一个容器
```
docker run -it ubuntu:14.04 top
```

* 另外开一个终端，查看该容器进程在宿主机上的用户
```
~$ ps -aux|grep top
root     18724  0.2  0.0  19848  2400 pts/15   Ss+  14:16   0:00 top
```
可以看到，运行top命令的用户是root,即容器中的root用户就是宿主机的root用户

### 实验二：使用user namespace进行资源隔离

#### 配置实现
* 运行docker deamon进程的时候加入参数`--userns-remap=default`，如：ubuntu中是修改/etc/default/docker中的DOCKER_OPTS,追加配置`--userns-remap=default`
* 重启docker deamon，如：ubuntu中是使用`service docker restart`

#### 实验内容

* 运行一个容器
```
docker run -it ubuntu:14.04 top
```

* 另外开一个终端，查看该容器进程在宿主机上的用户
```
~$ ps -aux|grep top
165536   19347  0.1  0.0  19848  2424 pts/15   Ss+  14:32   0:00 top
```
可以看到，在宿主机上top命令的执行使用的用户是165536(uid)，不是root

* 看看容器内的top命令的输出
```
   1 root      20   0   19848   2424   2108 R   0.0  0.0   0:00.07 top  
```
容器内，看上去仍然是root用户.即：有了user namespace之后，我们就可以将宿主机上的普通用户映射为容器的root用户.
那么，有一个问题？这个普通用户是谁？


## 二.user namespace的默认映射用户
上面的实验中，我们已经使用了user namespace的最简化配置.即：`--userns-remap=default`

实际上，docker新建了一个用户和用户组都叫做dockremap，容器内的root用户映射到宿主机的这个dockremap用户上.
```
$ cat /etc/passwd
……
dockremap:x:10000:10000:,,,:/home/dockremap:/bin/false
$ cat /etc/subuid
……
dockremap:165536:65536
$ cat /etc/subgid
……
dockremap:165536:65536
```

## 三.自定义映射用户
首先在宿主机上创建用户及用户组，在启动docker deamon的时候传入如下参数.

* --userns-remap=<uid>
* --userns-remap=<uid>:<gid>
* --userns-remap=<username>
* --userns-remap=<username>:<groupname>








