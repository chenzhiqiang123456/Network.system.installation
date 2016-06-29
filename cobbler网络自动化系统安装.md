#CObbler全自动批量安装部署Linux系统


## 什么是cobbler

Cobbler是一个Linux服务器快速网络安装的服务，可以通过网络启动(PXE)的方式来快速安装、重装物理服务器和虚拟机，同时还可以管理DHCP，DNS，TFTP、RSYNC以及yum仓库、构造系统ISO镜像

Cobbler可以使用命令行方式管理，也提供了基于Web的界面管理工具(cobbler-web)，还提供了API接口，可以方便二次开发使用。

Cobbler是较早前的kickstart的升级版，优点是比较容易配置，还自带web界面比较易于管理。

Cobbler内置了一个轻量级配置管理系统，但它也支持和其它配置管理系统集成，如Puppet，暂时不支持SaltStack。

Cobbler客户端Koan支持虚拟机安装和操作系统重新安装，使重装系统更便捷。

##Cobbler必须启动的服务
<pre>
   systemctl start cobblerd
   systemctl start httpd
可以开机自启动
systemctl enable cobblerd
systemctl enable httpd
</pre>

##cobbler 服务安装
###**一、安装cobbler包**
<pre>
yum install cobbler cober-web dhcp tftp-server pykickstart httpd xinetd -y
</pre>
###**二、启动cobbler服务及http服务**
<pre>
   systemctl start cobblerd
   systemctl start httpd
</pre>
###**三、根据cobbler检查对内容修改配置文件**
<pre>
 cobbler check 
</pre> 
###**四、配置cobbler**
**<1>、主机地址及TFTP地址的配置** 
<pre>
vim /etc/cobbler/settings
server: 192.168.56.3 ### 主机地址
next_server： 192.168.56.3 ###tftp的服务器地址
manage_dhcp： 1 ###设置为1表示cobbler有dhcp管理功能
manage_tftpd: 1 ###设置为1表示cobbler管理TFTP设置为0表示自行管理
</pre>
**<2>、**
<pre>
cobbler get-loaders
</pre>

**<3>、设置rsyncd为开机启动**
<pre>
systemctl enable rsyncd
</pre>

**<4>、配置WEB密码，并修改配置文件中的密码**
<pre>
openssl passwd -l -salt 'cobbler' 'cobbler'
/etc/cobbler/settings 第101行密码修改

修改用户
在Cobbler组添加chen用户，提示输入2遍密码确认
htdigest /etc/cobbler/users.digest "Cobbler" chen
cobbler sync
</pre>

**<5>、修改cobbler的dhcp模块，这个模块会覆盖dhcp本身的配置文件**
<pre>
vim /etc/cobbler/dhcp.template
subnet 192.168.56.0 netmask 255.255.255.0 {
     option routers             192.168.56.2;
     option domain-name-servers 192.168.56.2;
     option subnet-mask         255.255.255.0;
     range dynamic-bootp        192.168.56.0 192.168.56.254;
     default-lease-time         21600;
     max-lease-time             43200;
</pre>

**<6>、修改tftp将disable改为no**
<pre>
vim /etc/xinetd.d/tftp
disable                 = no
</pre>

**<7>、同步cobbler配置文件**
<pre>
cobbler sync
</pre>


##二、系统挂载及镜像导入及cobbler项目配置的修改
**<1>、挂载系统文件**
<pre>
mount /dev/cdrom /mnt
</pre>

**<2>、导入镜像cobbler import**
<pre>
 cobbler import --path=/mnt/ --name=CentOS-7-x86_64 --arch=x86_64
--path=/mnt 镜像文件目录
--name 项目名称(PS:就是自己给系统起的别名（版本）)
--arch  架构属性 （PS：就是系统是多少位的）

* 导入后镜像存在的地址为  *
 /var/www/cobbler/ks_mirror
 
</pre>

**<3>、查看cobbler中项目**
<pre>
cobbler list
</pre>

**<4>、查看cobbler项目的配置文件**
<pre>
cobbler profile report
</pre>

**<5>、修改kickstart 配置文件**
<pre>
cobbler profile edit --name=CentOS-7-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg
</pre>
**<6>、修改启动内核参数（修改网卡为eth0）**
<pre>
cobbler profile edit --name=CentOS-7-x86_64  --kopts='net.ifnames=0 biosdevname=0'  
</pre>

**<7>、修改pxe启动提示**
<pre>
vim /etc/cobbler/pxe/pxedefault.template
MENU TITLE Cobbler | QQ:530486014
cobbler sync
</pre>

##三、cobbler指定mac及IP等信息
<pre>
cobbler system add --name=chen  --mac=00:0C:29:51:30:00 --profile=CentOS-7-x86_64 \
--ip-address=192.168.56.7 --subnet=255.255.255.0 --gateway=192.168.56.2 --interface=eth0 \
--static=1 --hostname=linux-node2 --name-servers="192.168.56.2" \
--kickstart=/var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg
</pre>
##四、使用cobbler制作openstack的repo

###**<1>、添加repo**
<pre>
cobbler repo add --name=openstack-mitaka --mirror=http://mirrors.aliyun.com/centos/7.2.1511/cloud/x86_64/openstack-mitaka/ --arch=x86_64 --breed=yum
</pre>
###**<2>、同步repo**
<pre>
cobbler reposync
</pre>
###**<3>、添加repo到对应的profile项目中**
<pre>
 cobbler profile edit --name=Centos-7-x86_64  --repos=http://mirrors.aliyun.com/centos/7.2.1511/cloud/x86_64/openstack-mitaka/ 
</pre>
###**<4>、修改ks.cfg文件添加到%post %end中间**
<pre>
systemctl disable postfix.service

$yum_config_stanza
%end
</pre>
###**<5>、添加定时任务，定时同步repo**
<pre>
echo "1 3 * * * /usr/bin/cobbler reposync --tries=3 --no-fail" >> /var/spool/cron/root
</pre>
##五、客户端自动重装系统
<pre>
自动重装系统
yum install koan -y
koan --server=192.168.56.3  --list=profiles
指定要重装的系统
koan replace-self --server=192.168.3 --list=profile=CentOS-6-x86_64
</pre>
