## 一、准备工作

### 1.1 安装docker

docker下载地址：https://www.docker.com/get-started

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/db55be9f57045fb2777129669688783d.png)

### 1.2 设置加速器地址

首先注册阿里云容器镜像服务

地址：https://cr.console.aliyun.com/cn-hangzhou/instances/repositories

注册好之后选择镜像加速，操作如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/bc22c448840af1394622b7553086ec66.png)

复制加速地址，然后打开docker的 Preferences -》Docker Engine ，编辑json

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/89ef485d771cf9ba67350b224d1f895c.png)

在原来的json串中添加 registry-mirrors ，然后重新启动docker

### 1.3 创建阿里云镜像仓库

选择镜像仓库 -》创建镜像仓库 -》输入仓库名，描述等信息，如下图：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/03b59db4041cddb5e4b3ce6e15925c2f.png)

点击下一步选择本地仓库，最后点击创建镜像仓库即可

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/e3989a52153a767b4584f68ba574262f.png)


## 二、搭建ftp

我们选择centos作为基础容器，然后在这个基础容器之上安装ftp服务，最后打成一个ftp服务镜像上传到阿里云仓库中

### 2.1 构建ftp docker镜像

在构建镜像之前，我们需要配置以下vsftp的配置文件，我们把配置文件放在宿主机的 /data 目录下，配置文件名为vsftpd.conf，其内容如下：

```powershell
# vsftp默认加载配置文件的位置
# Example config file /etc/vsftpd/vsftpd.conf

# 允许匿名用户登陆
anonymous_enable=YES

# 允许写数据都根目录
allow_writeable_chroot=YES
#
# 是否允许本地用户登陆
local_enable=YES
#
# 是否允许登陆用户有写权限。属于全局设置，默认值为YES
write_enable=YES
#
# 默认的umask码，用户决定用户创建文件时的默人文件权限
local_umask=022
#
# 是否允许匿名用户上传文件，不过前提是write_enable=YES并且匿名用户拥有该文件夹的写入权限
anon_upload_enable=YES
#
# 如果设为YES，则允许匿名登入者有新增目录的权限，只有在write_enable=YES时，此项才有效。当然，匿名用户必须要有对上层目录的写入权。默认值为NO
anon_mkdir_write_enable=YES
anon_other_write_enable=YES
#
# 
dirmessage_enable=YES
#
# The target log file can be vsftpd_log_file or xferlog_file.
# This depends on setting xferlog_std_format parameter
xferlog_enable=YES
#
# Make sure PORT transfer connections originate from port 20 (ftp-data).
connect_from_port_20=YES
#
# If you want, you can arrange for uploaded anonymous files to be owned by
# a different user. Note! Using "root" for uploaded files is not
# recommended!
#chown_uploads=YES
#chown_username=whoever
#
# The name of log file when xferlog_enable=YES and xferlog_std_format=YES
# WARNING - changing this filename affects /etc/logrotate.d/vsftpd.log
#xferlog_file=/var/log/xferlog
#
# Switches between logging into vsftpd_log_file and xferlog_file files.
# NO writes to vsftpd_log_file, YES to xferlog_file
xferlog_std_format=YES
#
# You may change the default value for timing out an idle session.
#idle_session_timeout=600
#
# You may change the default value for timing out a data connection.
#data_connection_timeout=120
#
# It is recommended that you define on your system a unique user which the
# ftp server can use as a totally isolated and unprivileged user.
#nopriv_user=ftpsecure
#
# Enable this and the server will recognise asynchronous ABOR requests. Not
# recommended for security (the code is non-trivial). Not enabling it,
# however, may confuse older FTP clients.
#async_abor_enable=YES
#
# By default the server will pretend to allow ASCII mode but in fact ignore
# the request. Turn on the below options to have the server actually do ASCII
# mangling on files when in ASCII mode.
# Beware that on some FTP servers, ASCII support allows a denial of service
# attack (DoS) via the command "SIZE /big/file" in ASCII mode. vsftpd
# predicted this attack and has always been safe, reporting the size of the
# raw file.
# ASCII mangling is a horrible feature of the protocol.
#ascii_upload_enable=YES
#ascii_download_enable=YES
#
# You may fully customise the login banner string:
#ftpd_banner=Welcome to blah FTP service.
#
# You may specify a file of disallowed anonymous e-mail addresses. Apparently
# useful for combatting certain DoS attacks.
#deny_email_enable=YES
# (default follows)
#banned_email_file=/etc/vsftpd/banned_emails
#
# You may specify an explicit list of local users to chroot() to their home
# directory. If chroot_local_user is YES, then this list becomes a list of
# users to NOT chroot().
#chroot_local_user=YES
#chroot_list_enable=YES
# (default follows)
#chroot_list_file=/etc/vsftpd/chroot_list
#
# You may activate the "-R" option to the builtin ls. This is disabled by
# default to avoid remote users being able to cause excessive I/O on large
# sites. However, some broken FTP clients such as "ncftp" and "mirror" assume
# the presence of the "-R" option, so there is a strong case for enabling it.
#ls_recurse_enable=YES
#
# When "listen" directive is enabled, vsftpd runs in standalone mode and
# listens on IPv4 sockets. This directive cannot be used in conjunction
# with the listen_ipv6 directive.
listen=YES
#
# This directive enables listening on IPv6 sockets. To listen on IPv4 and IPv6
# sockets, you must run two copies of vsftpd with two configuration files.
# Make sure, that one of the listen options is commented !!
#listen_ipv6=YES

pam_service_name=vsftpd
userlist_enable=YES
tcp_wrappers=YES
anon_root=/data/ftpUser
no_anon_password=YES
local_root=/data/ftpUser
ftp_username=ftpUser
# 开启被动模式
pasv_enable=YES
# 被动模式可以开启的端口范围
pasv_min_port=21100
pasv_max_port=21110

```

然后我们准备一个Dockfile文件，放在/data目录下，文件名为Dockfile，其内容如下：

```powershell
# 拉取基础镜像
FROM centos:7
# 维护人
MAINTAINER 951645267@qq.com

# 将vsf配置文件复制到容器的/etc/vsftpd目录下
RUN mkdir -p /etc/vsftpd
ADD vsftpd.conf /etc/vsftpd

# 安装ftp
RUN yum install -y vsftpd
RUN mkdir /data

# 创建用户ftpUser，用户目录为/data/ftpUser
# -s 后面的参数 /bin/bash 表示当用户 ftpUser 登陆时，默认使用的 bash
RUN useradd -d /data/ftpUser -s /bin/bash ftpUser

# 给ftpUser设置密码 123 
RUN echo "ftpUser:123" | chpasswd
RUN chmod 755 /data/ftpUser
# 起声明作用，告诉他人我的服务在这些端口，并不是说我只能访问这些接口
# 接口能否访问由启动容器的命令 -p 决定
EXPOSE 20 21
# 运行容器时启动ftp服务, 但是vsftpd时个后台服务，一执行
# 就会退出容器，所以需要加个 tail -f /dev/null 避免容器退出
ENTRYPOINT /usr/sbin/vsftpd && tail -f /dev/null

# 使用 systemctl 管理服务，关于 systemctl 的使用可参考 https://blog.csdn.net/masteryee/article/details/83689990
# RUN systemctl start vsftp.service

```
准备好 Dockerfile 之后我们就可以构建一个安装了ftp的镜像了（我这里创建的dockerfile文件使用的是默认的名字 Dockerfile）

```powershell

# sudo docker build -t name:tag .
sudo docker build -t registry.cn-hangzhou.aliyuncs.com/wuzh-yun/ftp:1.0.0 .

```

构建的镜像名为 registry.cn-hangzhou.aliyuncs.com/wuzh-yun/ftp，对应的阿里云镜像仓库格式为 域名/命名空间/仓库名，如下为构建过程的输出内容：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ba000f07757ab9a1adfc0e37e9ec28b0.png)

构建完成后，我们查看一下本地仓库是否出现了一个叫做registry.cn-hangzhou.aliyuncs.com/wuzh-yun/ftp:1.0.0的镜像

```powershell

sudo docker images

```

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/6c3ae7e5c6c6593344b19369a1c0ebe1.png)

### 2.2 运行ftp服务

启动容器

```powershell

# 运行镜像，将容器的端口21，20，21100-21110映射到宿主机的21，20，21100-21110
# 将宿主机的/data/ftp挂载到容器的/data/ftpUser目录下，这样我们上传文件到 /data/ftpUser 相当于把文件存在到宿主机的/data/ftp
# 将配置文件vsftpd.conf挂载到容器的/etc/vsftpd/vsftpd.conf，ftp默认加载配置的路径
sudo docker run -dit -p 21:21 -p 20:20 -p 21100-21110:21100-21110 -v /data/ftp:/data/ftpUser --name ftp registry.cn-hangzhou.aliyuncs.com/wuzh-yun/ftp:1.0.0

```

如果挂载目录时报如下错误：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/9e88b7fad1cc563ba015466402b09401.png)

那么请配置共享目录，操作如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/76575bab2c54d67d9a3d4fcd156882ee.png)

重启docker

### 2.3 测试ftp

如果你的mac没有安装ftp，可以执行以下命令：

```powershell
brew install telnet 
brew install inetutils 
brew link --overwrite inetutils

```
如果 brew 也没有安装的话，请参考：https://zhuanlan.zhihu.com/p/111014448。
如果你有VPN的话，直接上官网：https://brew.sh/index_zh-cn.html

执行 ftp ip port，然后输入用户名和密码

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/5565f981b10c7d34af5c0e5a1ccec862.png)

### 2.4 上传镜像到阿里云

首先需要先登陆，命令如下：

```powershell
sudo docker login --username=您的用户名 registry.cn-hangzhou.aliyuncs.com

```
然后输入密码登陆，最后推送镜像至阿里云镜像仓库

```powershell
sudo docker push registry.cn-hangzhou.aliyuncs.com/wuzh-yun/ftp:1.0.0

```



