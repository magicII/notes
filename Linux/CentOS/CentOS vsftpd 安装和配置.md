为方便上传文件到 web 服务器中，一般都需要通过 ftp 方式传输。在 CentOS 中推荐使用 vsftpd 搭建 ftp 服务器。

### 安装 vsftpd

1. 安装 vsftpd

```shell
yum -y install vsftpd
```

2. 设置 vsftpd 服务开机自启动

```shell
chkconfig vsftpd on
```

3. 相关操作指令

```shell
service vsftpd start      # 开启 vsftpd 服务
service vsftpd restart		# 重启 vsftpd 服务
service vsftpd stop			# 关闭 vsftpd 服务
service vsftpd status		# 查看 vsftpd 服务的状态
```

### 基础配置

```shell
vi /etc/vsftpd/vsftpd.conf
```
	
主要的配置项如下(配置完成之后记得需要重启 vsftpd 服务)：
	
```conf
# 禁止匿名用户 anonymous 登录
anonymous_enable=NO
# 允许本地用户登录
local_enable=YES
# 让登录的用户有写权限(上传，删除)
write_enable=YES
# 默认umask
local_umask=022
# 把传输记录的日志保存到 /var/log/vsftpd.log
xferlog_enable=YES
xferlog_file=/var/log/vsftpd.log
xferlog_std_format=NO
# 允许 ASCII 模式上传
ascii_upload_enable=YES 
# 允许 ASCII 模式下载
ascii_download_enable=YES
# 使用20号端口传输数据
connect_from_port_20=YES
	
# **接下来的三条配置很重要**
# `chroot_local_user` 设置为 YES，那么所有的用户默认将被 chroot，
# 也就是用户目录被限制在了自己的 home 下，无法向上改变目录；
# 但是`chroot_list_file` 设置的文件里的用户，是不会被 chroot 的(即，可以向上改变目录)
# 如果`chroot_local_user`设置为 NO，
# 那么`chroot_list_file` 设置的文件里的用户，是会被 chroot 的(即，无法向上改变目录)
# `chroot_list_enable` 设置为 YES，即让 chroot 用户列表有效。
chroot_local_user=YES
chroot_list_enable=YES
chroot_list_file=/etc/vsftpd/chroot_list
# 注意：配置完成之后还需要新建这个 chroot_list 文件
# touch /etc/vsftpd/chroot_list

use_localtime=YES
# 以 standalone 模式在 ipv4 上运行
listen=YES
# PAM 认证服务名，这里默认是 vsftpd，在安装 vsftpd 的时候已经创建了这个 pam 文件，
# 在/etc/pam.d/vsftpd，根据这个 pam 文件里的设置，/etc/vsftpd/ftpusers
# 文件里的用户将禁止登录 ftp 服务器，比如 root 这样敏感的用户，
# 所以你要禁止别的用户登录的时候，也可以把该用户追加到/etc/vsftpd/ftpusers里。
pam_service_name=vsftpd
```

### 添加 ftp 用户
设置好 vsftpd 服务之后，需要设置一个 ftp 账户用来登录 ftp。同时，可以给这个 ftp  账户设定相应的主目录 home，避免改动其他目录。当然，也可以创建多个 ftp 账户，分别管理不同的目录。

用户管理相关命令如下：

* `useradd` 添加用户
* `usermod` 修改用户
* `userdel` 删除用户
* `passwd`  设置用户密码

ftp 用户的时候一般不需要有登录系统的权限，所以创建的时候，可以指定`-s /sbin/nologin`选项。

```shell
# 创建一个 ftp 用户 ftpuser
# -d 指定主目录
# -g 指定所属组
# -s 指定是否有权限登录
# 另外，还可以通过 -u 选项指定用户的 uid
useradd -d /home/wwwroot/magento -g ftp -s /sbin/nologin ftpuser

# 一般建议添加的用户都是 web 服务的守护者，如 www，
# 可以设置用户的 uid 为 www 的 uid(需要使用 -o 选项)
useradd -d /home/wwwroot/test -s /sbin/nologin -g www -o -u 500 test

# 设置密码，之后会提示输入密码并确认重输入
passwd ftpuser

# 如果是先建立的文件夹，然后添加的用户，还需要更改路径权限
chown -R ftpuser /home/wwwroot/magneto 
```

> 关于 ftp 用户的创建，如果对于 web 服务器来说，由于开发者需要不时的上传文件到 web 目录，所以建议将 ftp 用户的 uid 改成和 web 目录的所有者的 uid 相同，这样能避免权限问题。或者，也可以开启 vsftpd 的虚拟用户访问权限，这样就能不必为每个开发者创建一个账号了，具体看下面的“问题及解决”。
> 
> 一般情况下，ftp 是不让 root 用户远程登陆，但只要你修改一个文件就可以登陆了：
> 
> * 去掉或注释掉`/etc/vsftpd/ftpusers`中的 root；
> * 去掉或注释掉`/etc/vsftpd/user_list`中的 root。

### 问题及解决
#### 登录报错：500 Oops
**错误提示**

`500 OOPS: vsftpd: refusing to run with writable root inside chroot ()`

**原因**

当我们限定了用户不能跳出其主目录之后，使用该用户登录 FTP 时往往会遇到这个错误。

这个问题发生在最新 vsftpd 中，是由于下面的更新造成的：

`- Add stronger checks for the configuration error of running with a writeable root directory inside a chroot(). This may bite people who carelessly turned on chroot_local_user but such is life. `

从 2.3.5 之后，vsftpd 增强了安全检查，如果用户被限定在了其主目录下，则该用户的主目录不能再具有写权限了！如果检查发现还有写权限，就会报该错误。

**解决**

要修复这个错误，可以用命令`chmod a-w /home/user`去除用户主目录的写权限(注意把目录替换成你自己的)。

或者可以在 vsftpd 的配置文件中增加一项：`allow_writeable_chroot=YES`。

#### 开启虚拟用户
**问题**

开发人员是没有在 linux 系统中开通相关账户的，只开通了一个 FTP 账户。但是开发人员又要上传代码和修改相关的代码，怎么办呢？

**解决**

这个就需要结合 vsftpd 虚拟名用户来进行设置。

参考：[烂泥：nginx、php-fpm、mysql用户权限解析](http://ilanni.blog.51cto.com/526870/1561097)

在 vsftpd 的配置文件中，加入如下的配置：

```conf
# 启用vsftpd虚拟用户，就是所有登录到FTP的用户在系统都是虚拟用户
guest_enable=YES
# 虚拟用户对应的系统用户为nobody用户
guest_username=nobody
# 启用vsftpd验证
pam_service_name=vsftpd
# vsftpd 虚拟用户的配置目录
user_config_dir=/etc/vsftpd/vu_conf
# 启用vsftpd虚拟用户，并且虚拟用户和本地用户有相同的权限
virtual_use_local_privs=yes
```

然后再配置 vsftpd 虚拟用户的目录，如下：

```shell
vi vu_conf/ilanni
# 设置用户目录为 web 项目的目录
# local_root= /ilanni/a.ilanni.com
```

#### 无法连接 vsftpd 服务器
**问题**

配置好 vsftpd 服务之后，无法连接服务器，提示服务器无法提供连接一类的错误。

**原因**

一般情况下，是由于服务器上开启了防火墙，屏蔽了 ftp 的相关端口。

**解决**
一般情况下，ftp 需要用到的端口是 21 和 20 端口，而且是出入 tcp 连接。只需要在防火墙中开启相关的端口即可。

> 需要注意：CentOS7 默认的防火墙不是 iptables，而是 firewalle。下面是针对 iptables 进行的操作。

参考：[CentOS7 安装 iptables 防火墙](http://www.cnblogs.com/kreo/p/4368811.html)

```shell
# 先检查是否安装了iptables
service iptables status
# 安装iptables
yum install -y iptables
# 升级iptables
yum update iptables 
# 安装iptables-services
yum install iptables-services

# 停止firewalld服务
systemctl stop firewalld
# 禁用firewalld服务
systemctl mask firewalld

# 设置现有规则
# 查看 iptables 现有规则
iptables -L -n
# 先允许所有,不然有可能会杯具
iptables -P INPUT ACCEPT
# 清空所有默认规则
iptables -F
# 清空所有自定义规则
iptables -X
# 所有计数器归0
iptables -Z
# 允许来自于lo接口的数据包(本地访问)
iptables -A INPUT -i lo -j ACCEPT
# 开放20端口入站和出站
iptables -A INPUT -p tcp --dport 20 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 20 -j ACCEPT
# 开放21端口(FTP)入站和出站
iptables -A INPUT -p tcp --dport 21 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 21 -j ACCEPT
# 开放80端口(HTTP)
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
# 开放443端口(HTTPS)
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
# 允许接受本机请求之后的返回数据 RELATED,是为FTP设置的
iptables -A INPUT -m state --state  RELATED,ESTABLISHED -j ACCEPT
# 其他入站一律丢弃
iptables -P INPUT DROP
# 所有转发一律丢弃
iptables -P FORWARD DROP

# 保存上述规则
service iptables save
# 重启 iptables
service iptables restart
```

### 补充说明

#### umask 值的介绍及计算

当我们创建一个文件后，总是有一个默认权限的，那么这个权限是怎么来的呢？这就是`umask`干的事情。vsftpd 中使用`local_umask`来设置用户上传后文件、文件夹的权限。

`umask`设置了用户创建文件的默认权限，它与`chmod`的效果刚好相反，`umask`设置的是权限“补码”，而`chmod`设置的是文件权限码。一般在`/etc/profile`、`$[HOME]/.bash_profile`或`$[HOME]/.profile`中设置`umask`值。

`umask`命令允许你设定文件创建时的缺省模式，对应每一类用户(文件属主、同组用户、其他用户)存在一个相应的`umask`值中的数字。对于文件来说，这一数字的最大值是 6。系统不允许你在创建一个文本文件时就赋予它执行权限，必须在创建后用`chmod`命令增加这一权限。目录则允许设置执行权限，这样针对目录来说，`umask`中各个数字最大可以到 7。

该命令的一般形式为：`umask nnn`。其中`nnn`为`umask`值，范围为：000 – 777。

我们只要记住`umask`是从权限中“拿走”相应的位即可。下表是`umask`值与权限的对照表：

umask | 文件 | 目录
----- | ---- | ----
  0   |   6  |  7
  1   |   6  |  6
  2   |   4  |  5
  3   |   4  |  4
  4   |   2  |  3
  5   |   2  |  2
  6   |   0  |  1
  7   |   0  |  0

Linux 文件系统中：

- r：4（读）
- w：2（写）
- x：1（执行）

示例如下：

* 如果`umask`值为`022`，则默认目录权限为`755`，默认文件权限为`644`；
* 如果`umask`值为`000`，则默认目录权限为`777`，默认文件权限为`666`；
* 如果`umask`值为`047`，则默认目录权限为`730`，默认文件权限为`620`。

> 转摘：[vsftpd中umask值的介绍及计算](http://blog.csdn.net/faye0412/article/details/6280755)。

#### 主动模式和被动模式

FTP 存在两种模式：`PORT(主动)模式`、`PASV(被动)模式`。推荐使用的是被动模式。

两种模式的区别如下：

* `PORT(主动)模式` 所谓主动模式，指的是 FTP 服务器“主动”去连接客户端的数据端口来传输数据，其过程具体来说就是：客户端从一个任意的非特权端口 N（N>1024）连接到 FTP 服务器的命令端口（即 TCP 21 端口），紧接着客户端开始监听端口 N+1，并发送 FTP 命令`port N+1`到 FTP 服务器。然后服务器会从它自己的数据端口（20）“主动”连接到客户端指定的数据端口（N+1），这样客户端就可以和 ftp 服务器建立数据传输通道了。

* `PASV(被动)模式` 所谓被动模式，指的是 FTP 服务器“被动”等待客户端来连接自己的数据端口，其过程具体是：当开启一个 FTP 连接时，客户端打开两个任意的非特权本地端口（N >1024 和N+1）。第一个端口连接服务器的 21 端口，但与主动方式的 FTP 不同，客户端不会提交 PORT 命令并允许服务器来回连它的数据端口，而是提交 PASV 命令。这样做的结果是服务器会开启一个任意的非特权端口（P > 1024），并发送`PORT P`命令给客户端。然后客户端发起从本地端口 N+1 到服务器的端口 P 的连接用来传送数据。（注意此模式下的 FTP 服务器不需要开启 TCP 20 端口了）。

两种模式的比较：

* `PORT(主动)模式`只要开启服务器的 21 和 20 端口，而`PASV(被动)模式`需要开启服务器大于 1024 所有 TCP 端口(其实可以不开启所有的，可以指定端口区间)和 21 端口。

* 从网络安全的角度来看的话似乎 FTP PORT 模式更安全，那么为什么 RFC 要在 FTP PORT 基础再制定一个 FTP PASV 模式呢？其实 RFC 制定 FTP PASV 模式的主要目的是为了数据传输安全角度出发的，因为 FTP port 使用固定 20 端口进行传输数据，那么作为黑客很容使用 sniffer 等探嗅器抓取 FTP 数据，这样一来通过 FTP PORT 模式传输数据很容易被黑客窃取，因此使用 PASV 方式来架设 FTP Server 是最安全绝佳方案。

因此：**如果只是简单的为了文件共享，完全可以禁用 PASV 模式，解除开放大量端口的威胁，同时也为防火墙的设置带来便利。**但是，FTP 工具或者浏览器默认使用的都是 PASV 模式连接 FTP 服务器，因此，必须要使 vsftpd 在开启了防火墙的情况下，也能够支持 PASV 模式进行数据访问。

**开启 vsftpd 的 PASV 模式**

1. 修改`/etc/vsftpd/vsftpd.conf`文件配置，做如下配置：
    
    ```conf
    # 注释掉对 20 端口的监听
    #connect_from_port_20=YES
    # 开启 PASV 模式
    pasv_enable=YES
    # 设置被动模式数据传输的端口范围
    pasv_min_port=6000
    pasv_max_port=7000
    ```

2. 修改防火墙对端口的限制。由于被动模式需要开放一些端口，此时需要修改防火墙，以便允许这些端口的通讯。下面以 Iptables 作为示例：

    ```iptables
    # FTP
    -A INPUT -p tcp -m state --state NEW -m tcp --dport 21 -j ACCEPT
    -A INPUT -p tcp -m state --state NEW -m tcp --dport 6000:7000 -j ACCEPT
    ```
    
修改完成后，需要重启 vsftpd 和 iptables 服务。

> 另外，对于一些 FTP 客户端，可能需要显式的设置使用 PASV 模式链接。


