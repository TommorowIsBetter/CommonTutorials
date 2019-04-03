# 制作OpenStack下的Java WebApp镜像 #
----------

##1. 准备用来制作镜像的虚拟机

在VMWare虚拟机下进行镜像制作。VirtualBox不能有效支持KVM的运行。

### 检查CPU虚拟化支持
运行 `grep -E -o 'vmx|svm' /proc/cpuinfo` ，显示`vmx`则表示当前系统支持KVM。

### 安装软件   
sudo apt-get install libvirt-bin kvm qemu qemu-kvm virt-manager bridge-utils gvncviewer

### 检查kvm的运行  
lsmod | grep kvm  
如果KVM没有运行，则将其激活   
modprobe kvm

### 安装镜像系统

准备磁盘(<2G）  
qemu-img create -f qcow2 ~/webapp.qcow2 1960M      

启动 `CentOS 7.3 x64 Minimal install` 系统的安装  
sudo virt-install --virt-type kvm --name webapp --ram 1024 --disk /home/plasedev/webapp.qcow2,format=qcow2 --network network=default --graphics vnc,listen=0.0.0.0 --noautoconsole --os-type=linux --os-variant=centos7.0 --location=/home/plasedev/CentOS-7-x86_64-Minimal-1611.iso 

安装注意事项     
- 分区： 分配`/boot`(200M)和`/`两个分区，<font color="red">采用ext4文件系统，不要xfs。xfs不能方便地调整。  </font>  
- 网络设置： 确保你的网卡eth0是DHCP状态的，而且请务必勾上”auto connect”的对勾

安装后，可以打开KVM虚拟机管理程序，可在GUI界面下完成虚拟机控制  
sudo virt-manager 

或者检查KVM的运行，并利用VNC链接到虚拟机  
sudo virsh list  
gvncviewer 127.0.0.1:0

安装完成后，系统会关机。可用如下命令重新启动虚拟机。  
sudo virsh start webapp
 
镜像系统账号:  
root:webapp   
webapp:webapp

##2. 准备WebApp的运行环境

用SSH登陆到镜像虚拟机，开始进行一系列软件安装和环境配置。

ssh webapp@192.168.122.171

### 安装基本工具

`vi /etc/sudoers`文件，在`root ALL=(ALL) ALL`下添加`webapp ALL=(ALL) ALL`，然后`wq!`保存退出，赋给webapp用户sudo的权限。
 
sudo yum install net-tools tar bzip2 wget telnet

安装acpid工具，以方便远程关闭机器  
sudo yum install acpid  
sudo systemctl enable acpid

### 安装Java环境

传递几个大文件到镜像虚拟机（下载很慢）。  采用SSH方式：  
scp /home/plasedev/Downloads/jdk-8u144-linux-x64.rpm webapp@192.168.122.171:/home/webapp/jdk-8u144-linux-x64.rpm   

或者镜像系统关机状态下使用KVM工具：  
sudo virt-copy-in -d webapp /home/plasedev/Downloads/jdk-8u144-linux-x64.rpm /home/webapp 

安装Java：  
sudo rpm -ivh jdk-8u144-linux-x64.rpm  
rm jdk-8u144-linux-x64.rpm
 
### 安装Tomcat容器到 `/opt/tomcat` 目录下

scp -rf /mnt/hgfs/CloudTest/PerfCloud/WebApp/tomcat root@192.168.122.171:/opt 

### 设置tomcat程序开机启动

添加脚本可执行权限  
sudo chmod +x /opt/tomcat/bin/catalina.sh

添加rc.local可执行权限   
sudo chmod +x /etc/rc.d/rc.local

sudo vi /etc/rc.d/rc.local   
在rc.local中添加一行  
/opt/tomcat/bin/catalina.sh start & 


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
sudo virsh list  
如果未关闭，可以强制关闭  
sudo virsh shutdown webapp

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
 
   