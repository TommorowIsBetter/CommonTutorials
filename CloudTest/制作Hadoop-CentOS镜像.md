# 制作OpenStack下的Hadoop集群使用的系统镜像 #
----------
##1. 准备用来制作镜像的虚拟机
此环境是在VMWare虚拟机下进行镜像制作。通过在VMWare下面安装CentOS7系统虚拟机，CenOS7为Minimal版本，然后在CentOS7宿主机下面进行镜像制作。安装CentOS虚拟机系统的时候，要勾选VNC连接，开启VNC连接后面会用VNC Viewer才可以在Windows系统下面使用VNC Viewer连接CentOS7下的镜像系统，开启VNC连接在VMWare下系统镜像的设置-》选项里面可以看到。
### 检查CPU虚拟化支持
运行 `grep -E -o 'vmx|svm' /proc/cpuinfo` ，显示`vmx`则表示当前系统支持KVM。
### 安装软件   
yum -y install libvirt-bin kvm qemu qemu-kvm virt-manager bridge-utils gvncviewer tigervnc-server virt-install libvirt 
安装tigervnc-server是为了VNC Viewer可以连接到镜像系统。安装virt-install是为了安装镜像系统的时候需要，安装libvirt是为了启动libvirt这个服务，
virt-install才可以正常运行。
### 检查kvm的运行  
lsmod | grep kvm  
如果KVM没有运行，则将其激活   
modprobe kvm
### 安装镜像系统

准备磁盘 
qemu-img create -f qcow2 /home/plasedev/webapp.qcow2 4096M      

启动 `CentOS 7.3 x64 Minimal install` 系统的安装  
virt-install --virt-type kvm --name webapp --ram 1024 --disk /home/plasedev/webapp.qcow2,format=qcow2 --network network=default --graphics vnc,listen=0.0.0.0 --noautoconsole --os-type=linux --os-variant=centos7.0 --location=/home/plasedev/CentOS-7-x86_64-Minimal-1804.iso 

运行virt-install命令之前要启动libvirt服务,不启动的话，vir-install命令运行会失败。libvirt服务启动命令：systemctl start libvirtd.service

如果报出同门错误：ERROR  Guest name ‘webapp’ is already in use.解决办法：#virsh undefine webapp

镜像系统开始安装后，可以用VNC Viewer工具连接镜像系统，完成安装步骤。方法为打开VNC Viewer工具，输入Centos7的ip地址和5900端口后即可连接，5900端口号在VMWare的选项设置里面可以看到。 CentOS7宿主机IP::5900,这个IP是宿主机的IP

安装注意事项     
- 分区： 分配`/boot`(200M)和`/`两个分区，<font color="red">采用ext4文件系统，不要xfs。xfs不能方便地调整。选择Standard Partition分别。  </font>  
- 网络设置： 确保你的网卡eth0是DHCP状态的，而且请务必勾上”auto connect”的对勾

安装完成后，点击reboot此时镜像系统不会重启，系统会关机。可用如下命令重新启动虚拟机。  
virsh start webapp




 
镜像系统账号:  
root:webapp   
webapp:webapp

##2. 准备Hadoop的运行环境

利用VNC Viewer工具连接到镜像系统进行hadoop需要的环境配置

### 安装基本工具

`vi /etc/sudoers`文件，在`root ALL=(ALL) ALL`下添加`webapp ALL=(ALL) ALL`，然后`wq!`保存退出，赋给webapp用户sudo的权限。
 
sudo yum install net-tools tar bzip2 wget telnet

安装acpid工具，以方便远程关闭机器  
sudo yum install acpid  
sudo systemctl enable acpid

安装hadoop的细节较多，详细步骤此处贴上链接地址：https://blog.csdn.net/pucao_cug/article/details/71698903

### 1.主机名配置
分别给三台集群系统的机器命名，然后在每台机器的/etc/hosts 写入机器名与IP地址的映射关系，后面的配置即可通过主机名完成，而不需要填写IP地址。使三台
机器可以通过主机名可以互相ping通。制作镜像的时候没有试验ping，而是在openstack里面才开始试验ping通。

### 2.无密码登陆设置
就是使用SSH进行无密码登陆设置，通过设置公钥放在目标主机上即可完成无密码登陆。使三台机器都可以都可以互相无密码登陆。在openstack里面进行试验无密码登陆。

### 3.安装JDK
安装JDK的安装路径要安装在/opt/java/文件夹下，然后在/etc/profile进行环境变量的设置。安装完成后检查是否安装成功。
 
### 4.安装Hadoop
安装hadoop的路径在/opt/hadoop/,修改相应的配置文件。我这里是把五个相应的配置文件保存下来，然后进行分个传输到具体的系统。hadoop设置好之后，在openstack里面进行HDFS文件系统格式化。

### 5.安装Tomcat
安装Tomcat的路径在/opt/tomcat/

### 防火墙安全策略设置
 
关闭防火墙   
>     sudo systemctl stop firewalld.service #停止firewall   
>     sudo systemctl disable firewalld.service #禁止firewall开机启动

关闭SELINUX
>     #修改SELinux配置并检查
>     sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config  
>     grep SELINUX=disabled /etc/selinux/config 
>     # 关闭selinux防火墙 
>     sudo setenforce 0    

### 设置SSH访问权限

修改`sudo vi /etc/ssh/sshd_config`文件中的`PermitRootLogin yes`


##3. 制作镜像
<font color="red">先关闭镜像系统并备份qcow2文件，此处安装和配置后虚拟机可能不能再直接启动</font>

检查是否完全关机  
virsh list  
如果未关闭，可以强制关闭  
virsh shutdown webapp

###设置镜像的虚拟机的网络

* 删除网卡MAC信息  
  修改文件 vi /etc/sysconfig/network-scripts/ifcfg-eth0   (或其它网卡名称)   
  删除UUID和HWADDR行   
  修改NM_CONTROLLED=no
  
* 删除已生成的网络设备规则，否则制作的镜像不能上网  
  mv /etc/udev/rules.d/70-persistent-ipoib.rules  /root   

* 禁用ZERCONF服务
  增加一行到 /etc/sysconfig/network 中  
  echo "NOZEROCONF=yes" >> /etc/sysconfig/network

### 安装cloud-init工具
<font color="red">此处安装后虚拟机可能就无法正常从VirtualBox下启动了</font>

yum install cloud-init cloud-utils-growpart

安装后需配置Cloud-Init选项，允许用用户名和密码登录（否则用publicKey登陆，略麻烦）   
vi /etc/cloud/cloud.cfg

将以下行  
`ssh_pwauth：  0`  
改成  
`ssh_pwauth：  1`


### 关机并导出镜像
>     init 0

现在镜像文件webapp.qcow2即可以开始使用了。

WebApp的镜像启动时间较长（特别是内存不充足的情况，可能要10分钟），需耐心等待，直到Console中可以看到系统完成了初始化。

##5. 在OpenStack中启动WebApp镜像
三个镜像在openstack中创建镜像然后创建实例，并启动实例。需要做如下工作：
1.分别给集群中三台机器命名，命令如下：
hostname hserver1
hostname hserver2
hostname hserver3

2.分别给三台机器修改/etc/hosts,目的是可以通过主机名访问。在/etc/hosts添加如下部分：
10.2.0.13 hserver1
10.2.0.14 hserver2
10.2.0.11 hserver3

3.修改/etc/profile,目的是给hadoop设置环境变量，这样在Centos中才可以输入HDFS的命令。在/etc/profile添加如下部分：
export HADOOP_HOME=/opt/hadoop/hadoop-2.8.5
export PATH=$PATH:$HADOOP_HOME/bin

4.格式化HDFS文件系统(执行这条命令之前可以通过ssh hserver2 ,ssh hserver3来查看是否可以通过SSH无密码登陆，其它两台机器亦是如此)
cd /opt/hadoop/hadoop-2.8.5/bin
./hadoop namenode -format

5.启动Hadoop
cd /opt/hadoop/hadoop-2.8.5/sbin
./start-all.sh

创建实例的时候，安全组策略要开放50070,50075,9000,8088四个端口

### 最低资源需求

内存： 1G  
磁盘： Volumn 3G，Root Disk可以为0  
网络： 必须分配Floating IP

**注意**：OpenStack的防火墙设置中，必须允许到实例80端口的连接

 
##6. 更新镜像
 
### 开机更新（安装cloud-init后无法开机）
scp /mnt/hgfs/CloudTest/PerfCloud/TestAgent/testagent.jar root@192.168.122.180:/opt/testagent/testagent.jar 

### 关机情况下的更新
sudo virt-edit -d testagent /opt/testagent/config.json  
sudo virt-copy-in -d testagent /mnt/hgfs/CloudTest/PerfCloud/TestAgent/image/config.json /opt/testagent 

### 首先检查目前的磁盘使用情况（在宿主机中）
sudo apt-get install libguestfs-tools  
sudo virt-filesystems --long --parts --blkdevs -h -a webapp.qcow2   
sudo virt-df -d webapp -h

### 清理系统
查看安装的包  
rpm -qa
rpm -e --allmatches jdk1.8
 
   