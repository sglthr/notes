# RHCSA9

## 环境准备

```shell
# 登录 servera
root/flectrag

# 允许 root 远程登录，如果修改这个文件不生效，则检查/etc/ssh/sshd_config.d目录下是否有相关配置
vim /etc/ssh/sshd_config

PermitRootLogin yes

# 重启sshd
systemctl restart sshd

# 查看网卡
nmcli connection show

# 配置静态ip
nmcli connection modify "Wired connection 1" ipv4.method manual ipv4.addresses 172.25.250.100/24 ipv4.gateway 172.25.250.254 ipv4.dns 172.25.250.254 autoconnect yes

# 重启网卡
nmcli connection up "Wired connection 1"

# 检查
ip a
```



## 1、配置网络设置

```shell
# 用物理机远程到node1
ssh root@node1

# 检查
ip a

# 修改主机名
hostnamectl set-hostname node1.domain250.example.com

# 检查
hostname
```



## 2、配置默认存储库

```shell
cat > /etc/yum.repos.d/rhcsa.repo <<'EOF'
[BaseOS]
name = BaseOS
baseurl = http://content/rhel9.0/x86_64/dvd/BaseOS
gpgcheck = false
enabled = true

[AppStream]
name = AppStream
baseurl = http://content/rhel9.0/x86_64/dvd/AppStream
gpgcheck = false
enabled = true
EOF

# 验证
yum repoinfo
yum install -y vsftpd
# 或者 dnf install -y vsftpd
```



## 3、调试SELinux

```shell
# 检查
systemctl status httpd
vim /etc/httpd/conf/httpd.conf

# 查找必要软件包并安装
yum provides "*/semanage"
yum install -y policycoreutils-python-utils

# 修改文件上下文
ll -Z /var/www/html/
semanage fcontext -mt httpd_sys_content_t /var/www/html/file1

# 恢复安全上下文
restorecon -Rv /var/www/html/
ll -Z /var/www/html/

# 将 82 端口添加到白名单
semanage port -at http_port_t -p tcp 82
systemctl enable httpd --now

# 添加防火墙策略
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-port=82/tcp
firewall-cmd --reload

# 检查selinux状态
getenforce
cat /etc/selinux/config

# 验证
curl http://node1:82/file1
curl http://node1:82/file2
curl http://node1:82/file3
```



## 4、创建用户账户

```shell
groupadd sysmgrs

# -G代表次要组
useradd -G sysmgrs natasha
useradd -G sysmgrs harry

# 无权访问交互式shell
useradd -s /bin/false sarah

# 修改密码，注意是passwd，不是password
echo flectrag | passwd --stdin natasha
echo flectrag | passwd --stdin harry
echo flectrag | passwd --stdin sarah
```



## 5、配置cron作业

```shell
# 检查服务状态
systemctl status crond
systemctl enable crond --now

# 配置cron作业
crontab -e -u harry
* * * * * /usr/bin/echo hello
23 14 * * * /usr/bin/echo hello

# 检查
crontab -l -u harry
```



## 6、创建协作目录

```shell
mkdir /home/managers
chgrp sysmgrs /home/managers
# SUID=4,SGID=2,SBIT=1，2即设置该目录下的文件自动继承该目录所属组
chmod 2770 /home/managers

# 检查
ll -Z /home
drwxrws---. 2 root        sysmgrs     unconfined_u:object_r:user_home_dir_t:s0  6 Jun 11 03:55 managers
```



## 7、配置NTP

```shell
yum install -y chrony

# 加一行
vim /etc/chrony.conf
server materials.example.com iburst

# 验证
date
date -s "2000-1-1"
systemctl restart chronyd
systemctl enable chronyd --now
date
```



## 8、配置autofs

```shell
yum install -y nfs-utils
yum install -y autofs

# 配置映射文件
vim /etc/auto.master
/rhome /etc/auto.rhome

# 挂载配置（挂载点 挂载方式 源目录）
vim /etc/auto.rhome
remoteuser1 -rw materials.example.com:/rhome/remoteuser1

systemctl enable autofs --now

# 验证
ll /rhome/

ssh remoteuser1@localhost
remoteuser1@localhost's password: 'flectrag'

# 查看是否自动挂载家目录
pwd
/rhome/remoteuser1

# 验证写权限
touch test

# 查看挂载点是否包含以下内容
mount | grep rhome 
... 
materials.example.com:/rhome/remoteuser1 on /rhome/remoteuser1 type nfs4 
(`rw`,relatime,vers=4.2,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,timeo=600,r 
etrans=2,sec=sys,clientaddr=172.25.250.100,local_lock=none,addr=172.25.254.254)
```



## 9、配置用户账号

```shell
useradd -u 3533 manalo

echo flectrag | passwd --stdin manalo

# 验证
id manalo
```



## 10、查找文件

```shell
mkdir /root/findfiles

# 尝试查找，查看是否有相应文件
find / -user jacques

# -exec表示进入该进程执行命令，cp -a 保留所有属性，{}代表查找到的所有内容，\;是固定写法
find / -user jacques -exec cp -a {} /root/findfiles \;

# 验证
ll /root/findfiles/
```



## 11、查找字符串

```shell
# 尝试
grep ng /usr/share/xml/iso-codes/iso_639_3.xml

grep ng /usr/share/xml/iso-codes/iso_639_3.xml > /root/list

# 验证
cat /root/list
```



## 12、创建归档文件

```shell
yum install -y bzip2 gzip

# 根据题意，如果要求采用bzip2压缩则使用-cjf参数，如果是gzip压缩则采用-czf参数，文件后缀不重要
tar -cjvf /root/backup.tar.bz2 /usr/local
# tar -czvf /root/backup.tar.gz /usr/local

# 验证
file backup.tar.bz2 backup.tar.gz
backup.tar.bz2: bzip2 compressed data, block size = 900k
backup.tar.gz:  gzip compressed data, from Unix, original size modulo 2^32 51200
```



## 13、创建容器镜像

```shell
dnf install -y container-tools

# 必须用ssh，不能用su
ssh wallah@node1
wallah@node1's password: 'flectrag'

# 下载文件
wget http://classroom/Containerfile

# 构建镜像
podman login registry.lab.example.com -u admin -p redhat321
podman build -t pdf .
```



## 14、将容器配置为服务

```shell
# <ctrl + d> 切回root
mkdir /opt/{file,progress}
chown wallah:wallah /opt/{file,progress}

ssh wallah@node1

# 运行容器
podman run -d --name ascii2pdf -v /opt/file:/dir1:Z -v /opt/progress:/dir2:Z pdf

podman ps

# 创建系统服务
mkdir -p ~/.config/systemd/user
cd ~/.config/systemd/user/
podman generate systemd -n ascii2pdf -f --new
ls

# 验证系统服务
podman stop ascii2pdf
podman rm ascii2pdf
podman ps

systemctl daemon-reload --user
systemctl enable container-ascii2pdf --now --user
systemctl status container-ascii2pdf --user
podman ps

# 确保该服务在系统启动时启动
loginctl enable-linger
loginctl show-user wallah

# 验证
<ctrl + d> 切回root
reboot
ssh root@node1
ssh wallah@node1

# 查看容器启动时间不止几秒钟就是正确的
podman ps
```



## 15、添加sudo免密操作

```shell
<ctrl + d> 切回root
visudo
%sysmgrs        ALL=(ALL)       NOPASSWD: ALL

# 验证
su - natasha
id
sudo cat /etc/shadow
```



## ------  附加题  ------

## 16、创建共享目录

```shell
groupadd admins
mkdir /home/test
chgrp admins /home/test/
chmod 2770 /home/test/

# 验证
ll -d /home/test/
drwxrws---. 2 root admins 6 Jun 11 06:09 /home/test/
```



## 17、设置默认密码策略

```shell
vim /etc/login.defs
PASS_MAX_DAYS   25

# 验证
useradd test1
chage -l test1
```



## 18、创建系统监控脚本

```shell
vim /usr/local/bin/systeminfo
ps -xao user,pid,vsz,rss,%cpu --sort=%cpu

chmod +x /usr/local/bin/systeminfo

# 验证
systeminfo
```



## 19、创建查找文件脚本

```shell
mkdir /root/myfiles

# 创建脚本，-2000表示只关心第一位2，去掉-则表示权限必须完全符合2000
vim /usr/local/bin/myresearch
#!/bin/bash
find /usr -type f -and -size -10M -and -perm -2000 -exec cp -a {} /root/myfiles \;

chmod +x /usr/local/bin/myresearch

# 验证
myresearch
ll /root/myfiles/
```



## 20、设置用户默认创建文件UMASK

```shell
su - natasha
touch file1
mkdir dir1
ll

echo 'umask 222' >> .bashrc
cat .bashrc
source .bashrc

# 验证
touch file2
mkdir dir2
ll
```



## 21、配置一个应用

```shell
su - natasha

# 注意这里的等号两边不能有空格
vim .bashrc
alias rhcsa='echo This is a rhcsa'

source .bashrc
```



## ------  node2 重置密码  ------

```shell
# 右键点击serverb，选择Shut Down --> Force Reset
# 开机后，选择rescue模式，按e进入后，在内核一行末尾加上rw rd.break，最后<ctrl + x>，等待重启
chroot /sysroot
echo flectrag | passwd --stdin root
touch .autorelabel
ls -a
sync
exit
reboot

# 重启后选择正常的内核，需要重启两次。
```



## 23、配置默认存储库

```shell
scp root@node1:/etc/yum.repo.d/rhcsa.repo /etc/yum.repo.d

yum repoinfo

yum install -y ftp
```



## 24、调整逻辑卷大小

```shell
lvscan
vgs

# 调整逻辑卷大小
lvextend -L 230M /dev/myvol/vo

# 调整文件系统大小，-T查看文件系统格式，如果是xfs则使用xfs_growfs，如果是ext4则使用resize2fs
df -Th
resize2fs /dev/myvol/vo
xfs_growfs /dev/myvol/vo

# 检查
df -Th
```



## 25、添加交换分区

```shell
lsblk

# 如果vdb还有空间就用vdb做，否则可能影响后续题目（创建逻辑卷）操作
fdisk /dev/vdb

Welcome to fdisk (util-linux 2.37.4).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

This disk is currently in use - repartitioning is probably a bad idea.
It's recommended to umount all file systems, and swapoff all swap
partitions on this disk.


Command (m for help): n		# 新建分区
Partition number (3-128, default 3):    # 回车，默认编号3
First sector (1476608-10485726, default 1476608): 		# 回车，默认起始位置
Last sector, +/-sectors or +/-size{K,M,G,T,P} (1476608-10485726, default 10485726): +512M
	# 输入分区大小
Created a new partition 3 of type 'Linux filesystem' and of size 512 MiB.

Command (m for help): w		# 保存
The partition table has been altered.
Syncing disks.

# 格式化分区
lsblk
mkswap /dev/vdb3

# 挂载
vim /etc/fstab
/dev/vdb3 swap swap defaults 0 0

swapon -a

# 检查
swapon
NAME      TYPE      SIZE USED PRIO
/dev/vdb2 partition 243M   0B   -2
/dev/vdb3 partition 512M   0B   -3
```



## 26、创建逻辑卷

```shell
# 考试时如果没有vdc、vdd，则继续使用vdb（先计算空间是否足够）
lsblk

# 创建新分区
fdisk /dev/vdb
Welcome to fdisk (util-linux 2.37.4).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): n    # 新建分区
Partition type
   p   primary (1 primary, 0 extended, 3 free)    # 主分区
   e   extended (container for logical partitions)    # 扩展分区
Select (default p):     # 回车
Partition number (2-4, default 2): 		# 回车
First sector (1026048-20971519, default 1026048): 	  # 回车

***此处，分区大小，必须大于总大小，不可以刚好等于，建议多200M
Last sector, +/-sectors or +/-size{K,M,G,T,P} (1026048-20971519, default 20971519): +1200M  # 比题目中要求的多给200M

Created a new partition 2 of type 'Linux' and of size 1200 MiB.

Command (m for help): w    # 保存退出
    The partition table has been altered.
Syncing disks.

# 创建卷组和逻辑卷
lsblk
vgcreate -s 16M qagroup /dev/vdb4
lvcreate -l 60 -n qa qagroup

# 格式化分区，如果题目要求是vfat，则需要安装包，其他格式直接mkfs.ext4或者mkfs.xfs等，看题目要求
yum provides '*/mkfs.vfat'
yum install -y dosfstools

mkfs.vfat /dev/qagroup/qa

# 配置挂载
mkdir /mnt/qa
vim /etc/fstab
/dev/qagroup/qa /mnt/qa vfat defaults 0 0

mount -a

# 检查
df -Th
/dev/mapper/qagroup-qa vfat      959M  4.0K  959M   1% /mnt/qa
```



## 27、配置系统调优

```shell
yum install -y tuned
systemctl enable tuned --now
systemctl restart tuned
systemctl status tuned

# 查看当前策略
tuned-adm active

# 查看推荐策略
tuned-adm recommend

# 修改为推荐策略
tuned-adm profile virtual-guest

# 检查
tuned-adm active
```

