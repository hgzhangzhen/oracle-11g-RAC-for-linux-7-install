# oracle 11g rac for Linux 7 install 

## 1.环境描述

宿主机esxi 7.0

| 主机名 | 操作系统                                           | hostname  | 地址说明 | 网卡名  | Ip地址          |
| ------ | -------------------------------------------------- | --------- | -------- | ------- | --------------- |
| db01   | oracle Enterprise Linux Server release 7.9 (Maipo) | db01      | 物理IP   | ens 192 | 172.16.50.31    |
|        | 原版内核：5.4.17-2102.201.3.el7uek.x86_64          | db01-priv | 心跳IP   | ens 224 | 192.168.100.101 |
|        | 升级之后内核：5.4.17-2136.309.4.el7uek.x86_64      | db01-vip  | vip      | ens 192 | 172.16.50.33    |
| db02   | oracle Enterprise Linux Server release 7.9 (Maipo) | db02      | 物理IP   | ens 192 | 172.16.50.32    |
|        | 原版内核：5.4.17-2102.201.3.el7uek.x86_64          | db02-priv | 心跳IP   | ens 224 | 192.168.100.102 |
|        | 升级之后内核：5.4.17-2136.309.4.el7uek.x86_64      | db02-vip  | vip      | ens 192 | 172.16.50.34    |
|        |                                                    | db-scan   | scanIP   | ens 192 | 172.16.50.35    |

## 2.安装程序准备

### 2.1 安装程序下载与兼容性查询

操作系统安装不赘述，物理服务器安装linux系统需要注意服务器与操作系统的兼容性。

Redhat 提供的兼容性查询：

https://catalog.redhat.com/platform/red-hat-enterprise-linux/hardware

例如 :  服务器为Lenovo Thinksystem 650 v2,查看它的操作系统兼容性。

https://catalog.redhat.com/hardware/servers/detail/6040821?platforms=Red%20Hat%20Enterprise%20Linux%207

这里可以查到Lenovo Thinksystem 650 v2最低支持Redhat 7.9， 那么服务器也支持Centos 7.9与oracle Linux 7.9，根据实际情况，如果公司有买Redhat 的授权那就安装Redhat 系统 .

### 2.2 linux 7下载地址

| Redhat 7       | https://developers.redhat.com/products/rhel/download |
| -------------- | ---------------------------------------------------- |
| Centos 7       | http://isoredirect.centos.org/centos/7/isos/x86_64/  |
| Oracle linux 7 | https://yum.oracle.com/oracle-linux-isos.html        |

### 2.3 oracle  11g 下载地址

oracle 官方下载 地址：https://support.oracle.com（需要MOS帐号才能下载 ）

百度链接：https://pan.baidu.com/s/1PYksOiQBKQXrrGHeod7Cwg 提取码：uqzf 

Oracle Grid/RAC 11.2.0.4 on Oracle Linux 7官方安装文档：https://support.oracle.com/knowledge/Oracle%20Database%20Products/1951613_1.html#aref_section32

在官方文档中可以看到需要以下补订包，补订包上面百度网盘里面有。

```bash
	Oracle Grid Infrastructure - Installation Notes
 	Patch 19404309
 	Patch 18370031
 	Oracle Database/RAC - Installation Notes
 	Patch 19404309
 	Patch 19692824
```

### 2.4 更新Linux 系统补订

笔者以oracle  linux 7.9 为例

```bash

echo "nameserver 114.114.114.114" >>/etc/resolv.conf
[root@localhost ~]# uname -a   #查看当前版本
Linux localhost.localdomain 5.4.17-2102.201.3.el7uek.x86_64 #2 SMP Fri Apr 23 09:05:55 PDT 2021 x86_64 x86_64 x86_64 GNU/Linux
[root@localhost ~]#yum update -y  #更新系统补订
[root@localhost ~]# uname -a  # 更新完之后的内核版本
Linux localhost.localdomain 5.4.17-2102.201.3.el7uek.x86_64 #2 SMP Fri Apr 23 09:05:55 PDT 2021 x86_64 x86_64 x86_64 GNU/Linux
```

## 3. grid 环境准备

### 3.1 配置网络

```bash
[root@localhost ~]# nmcli connection show
NAME    UUID                                  TYPE      DEVICE 
ens192  14816990-8914-4730-9e29-dfcdbf912611  ethernet  ens192 
ens224  8115423f-23e4-472c-b68d-7bb1b33fdd51  ethernet  ens224 
virbr0  b09981a6-a4f1-4466-901c-4cf61f389b6e  bridge    virbr0 

[root@localhost ~]# nmcli connection modify ens192 ipv4.addresses 172.16.50.31/24 ipv4.gateway 172.16.50.1 ipv4.method manual autoconnect yes 
[root@localhost ~]# nmcli connection modify ens224 ipv4.addresses 192.168.100.101/24  ipv4.method manual autoconnect yes 
[root@localhost ~]# nmcli connection up ens192 
[root@localhost ~]# nmcli connection up ens224
```

### 3.2 调整ssh 连接

```bash
sed -i 's/#UseDNS yes/UseDNS no' /etc/ssh/sshd_config 
```

### 3.3 修改hostname文件 

```bash
cat <<EOF>>/etc/hosts
##public ip
172.16.50.31    db01
172.16.50.32    db02
##private ip
192.168.100.101 db01-priv
192.168.100.102 db02-priv
##vip ip
172.16.50.33    db01-vip
172.16.50.34    db02-vip
##scan ip
172.16.50.35    db-scan
EOF
```

### 3.4 配置网络防火墙与selinux

```bash
systemctl disable firewalld --now

sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
```

3. ### 5 禁用ntp服务与ntp时间同步

```bash
timedatectl set-timezone Asia/Shanghai
systemctl disable chronyd.service --now

ntpdate -u 172.16.50.1
hwclock -w
```

### 3.6 Linux 7 配置内核文件，关闭透明大页和numa：

```
sed -i 's/quiet/quiet transparent_hugepage=never numa=off/' /etc/default/grub
grub2-mkconfig -o /boot/grub2/grub.cfg
##重启后检查是否生效
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /proc/cmdline
```

### 3.7  avahi-daemon 功能配置

主机安装选择最小化安装，没有安装 avahi-daemon 功能，建议安装之后禁用，防止以后误操作导致出问题：

```
yum install -y avahi*
systemctl disable avahi-daemon.socket --now
systemctl disable avahi-daemon.service --now
```

### 3.8 配置 NOZEROCONF：

```
cat <<EOF>>/etc/sysconfig/network
NOZEROCONF=yes
EOF
```

### 3.9 配置pam.d/login

```
cat <<EOF>>/etc/pam.d/login
session required pam_limits.so 
session required /lib64/security/pam_limits.so
EOF
```

### 3.10 创建oracle grid用户与组

```bash
groupadd --gid 54321 oinstall
groupadd --gid 54322 dba
groupadd --gid 54323 asmdba
groupadd --gid 54324 asmoper
groupadd --gid 54325 asmadmin
groupadd --gid 54326 oper
groupadd --gid 54327 backupdba
groupadd --gid 54328 dgdba
groupadd --gid 54329 kmdba
groupadd --gid 54330 racdba

useradd --uid 54321 --gid oinstall --groups dba,oper,asmdba,racdba,backupdba,dgdba,kmdba oracle
useradd --uid 54322 --gid oinstall --groups dba,asmadmin,asmdba,asmoper,racdba grid
echo "oracle" |passwd oracle --stdin
echo "oracle" |passwd grid --stdin
```

### 3.11 创建安装所需要的目录 

```bash
mkdir -p /u01/grid
mkdir -p /u01/11.2.0/grid
chown -R grid:oinstall /u01
mkdir -p /u01/oracle 
chown -R oracle:oinstall /u01/oracle
chmod -R 775 /u01
```

### 3.12 配置环境变量

```bash
[oracle@orcdb1 ]# vi /home/oracle/.bash_profile
export ORACLE_BASE=/u01/oracle
export ORACLE_HOME=$ORACLE_BASE/product/11.2.0/db_1
export ORACLE_SID=oracl1
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib:$ORACLE_HOME/rdbms/lib
export PATH=$PATH:$ORA_CRS_HOME/bin:$ORACLE_HOME/bin
export CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib:$ORACLE_HOME/network/jlib
export NLS_LANG=AMERICAN_AMERICA.UTF8
export NLS_DATE_FORMAT='yyyy-mm-dd hh24:mi:ss'  
export TNS_ADMIN=/u01/11.2.0/grid/network/admin
 
[grid@orcdb1 ]# vi /home/grid/.bash_profile
export ORACLE_BASE=/u01/grid
export ORACLE_HOME=/u01/11.2.0/grid
export ORACLE_SID=+ASM1
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib:$ORACLE_HOME/rdbms/lib
export PATH=$PATH:$ORA_CRS_HOME/bin:$ORACLE_HOME/bin
export CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib:$ORACLE_HOME/network/jlib
 export NLS_LANG=AMERICAN_AMERICA.UTF8
export NLS_DATE_FORMAT='yyyy-mm-dd hh24:mi:ss'  
export TNS_ADMIN=/u01/11.2.0/grid/network/admin
```

### 3.13 添加其它用户可以访问grid程序环境变量

```bash
vi /etc/profile
PATH=$PATH:/u01/11.2.0/grid/bin
#export PATH USER LOGNAME MAIL HOSTNAME HISTSIZE HISTCONTROL
```

3. ### 14安装oracle linux 自带的程序包

```bash
[root@localhost ~]# yum list |grep oracle-rdbms
oracle-rdbms-server-11gR2-preinstall.x86_64
oracle-rdbms-server-12cR1-preinstall.x86_64
yum install oracle-rdbms-server-11gR2-preinstall.x86_64 -y
```

### 3.15 配置grid用户限制

```bash
cat <<EOF>>/etc/security/limits.conf
grid soft nofile 1024
grid hard nofile 65536
grid soft stack 10240
grid hard stack 32768
grid soft nproc 2047
grid hard nproc 16384
EOF
```

### 3.16 配置vncserver

```bash
[root@localhost ~]# yum list |grep vncserver
libvncserver.i686                      0.9.9-14.el7_8.1            ol7_latest   
libvncserver.x86_64                    0.9.9-14.el7_8.1            ol7_latest   
[root@localhost ~]# yum list |grep vnc
tigervnc-server.x86_64                 1.8.0-22.el7                ol7_latest   
[root@localhost ~]# yum install tigervnc-server.x86_64  -y
[root@db01 ~]# su - grid
Last login: Sun Jul 17 17:02:58 CST 2022 on pts/1
[grid@db01 ~]$ vncserver :2
```

## 4. 替换补订包安装程序，安装  grid 

##  4.1解压grid安装文件 

```bash
[root@db01 software]# unzip p13390677_112040_Linux-x86-64_3of7.zip 
## 解压必备的补订包
[root@db01 software]# unzip p19404309_112040_Linux-x86-64.zip
```

### 4.2更新补订包
```bash
cp /software/b19404309/grid/cvu_prereq.xml /software/grid/stage/cvu/cvu_prereq.xml
[root@db01 software]# chown grid:oinstall -R grid && chmod -R 775 grid;
```

cd /software/grid##执行安装程序开始安装，加上jar包防止弹窗不显示问题
./runInstaller -jreLoc /etc/alternatives/jre_1.8.0



![image-20220717172116673](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717172116673.png)



![image-20220717172759870](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717172759870.png)

![image-20220717172826370](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717172826370.png)

![image-20220717172845446](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717172845446.png)



![image-20220717172901176](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717172901176.png)

![](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717172913021.png)



![image-20220717173008372](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717173008372.png)



![image-20220717173043917](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717173043917.png)

![image-20220717173119146](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717173119146.png)



安装所需要的包。pdksh 可以忽略。

```
执行/tmp/CVU_11.2.0.4.0/runfixup.sh
[root@db01 CVU_11.2.0.4.0_grid]# yum install elfutils-libelf-devel.i686  
```

![image-20220717173305641](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717173305641.png)



![image-20220717174207679](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717174207679.png)

![image-20220717174353863](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717174353863.png)

root用户执行两个脚本 



![image-20220717174511740](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717174511740.png)

grid 安装完成， 

### 4.3 grid程序打补订

```
[root@db01 software]# su - grid
Last login: Sun Jul 17 17:18:38 CST 2022 on pts/0
[grid@db01 ~]$ $ORACLE_HOME/OPatch/opatch napply -oh $ORACLE_HOME -local /software/18370031 -silent 

查看补订是否打上
[grid@db01 ~]$ $ORACLE_HOME/OPatch/opatch  lsinventory
```

### 4.4 虚拟机克隆节点二

 打完补订，现在直接通过安装好的虚拟机克隆db02

![image-20220717175428237](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717175428237.png)



克隆完之后db01新建共享磁盘

### 4.5 共享磁盘配置

![image-20220717175629486](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717175629486.png)

![image-20220717175939807](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717175939807.png)



克隆完之后db02添加现有磁盘

![image-20220717180154773](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717180154773.png)



![image-20220717180454246](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717180454246.png)

![image-20220717180556182](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717180556182.png)

### 4.6 节点二网络配置

删掉db01网卡信息,并重新配置IP地址

```bash
  [root@db01 ~]# nmcli connection delete ens192 
  [root@db01 ~]# nmcli connection delete ens224 
  [root@db01 ~]# nmcli connection add con-name ens192 ifname ens192 type ethernet ipv4.addresses 172.16.50.32/24 ipv4.method manual ipv4.gateway 172.16.50.1 autoconnect yes 
  [root@db01 ~]# nmcli connection add con-name ens224 ifname ens224 type ethernet ipv4.addresses 192.168.100.102/24 ipv4.method manual  autoconnect yes 
  [root@db01 ~]#  nmcli connection up ens192 
  [root@db01 ~]#  nmcli connection up ens224
  [root@db01 ~]# hostnamectl set-hostname db02
```

### 4.7 安装oracle asm 磁盘工具包

```bash
yum install kmod-oracleasm.x86_64  -y
yum install oracleasm-support.x86_64  -y
[root@db01 ~]# oracleasm init
Creating /dev/oracleasm mount point: /dev/oracleasm
Loading module "oracleasm": oracleasm
Configuring "oracleasm" to use device physical block size
Mounting ASMlib driver filesystem: /dev/oracleasm
[root@db02 ~]# oracleasm configure -i
Configuring the Oracle ASM library driver.

This will configure the on-boot properties of the Oracle ASM library
driver.  The following questions will determine whether the driver is
loaded on boot and what permissions it will have.  The current values
will be shown in brackets ('[]').  Hitting <ENTER> without typing an
answer will keep that current value.  Ctrl-C will abort.

Default user to own the driver interface []: grid
Default group to own the driver interface []: asmadmin
Start Oracle ASM library driver on boot (y/n) [n]: y
Scan for Oracle ASM disks on boot (y/n) [y]: 
Writing Oracle ASM library driver configuration: done
[root@db02 ~]# oracleasm configure
ORACLEASM_ENABLED=true
ORACLEASM_UID=grid
ORACLEASM_GID=asmadmin
ORACLEASM_SCANBOOT=true
ORACLEASM_SCANORDER=""
ORACLEASM_SCANEXCLUDE=""
ORACLEASM_SCAN_DIRECTORIES=""
ORACLEASM_USE_LOGICAL_BLOCK_SIZE="false"
```

### 4.8 创建asm磁盘

```bash
[root@db01 ~]#parted /dev/sdb mklabel gpt mkpart primary "1 -1"
[root@db01 ~]#parted /dev/sdc mklabel gpt mkpart primary "1 -1"
[root@db01 ~]#parted /dev/sdd mklabel gpt mkpart primary "1 -1"
[root@db01 ~]#parted /dev/sde mklabel gpt mkpart primary "1 -1"

[root@db01 ~]#oracleasm createdisk asmdisk1 /dev/sdb1
[root@db01 ~]#oracleasm createdisk asmdisk2 /dev/sdc1
[root@db01 ~]#oracleasm createdisk asmdisk3 /dev/sdd1
[root@db01 ~]#oracleasm createdisk asmdisk4 /dev/sde1

[root@db02 ~]# oracleasm scandisks;
Reloading disk partitions: done
Cleaning any stale ASM disks...
Scanning system for ASM disks...
Instantiating disk "ASMDISK2"
Instantiating disk "ASMDISK1"
Instantiating disk "ASMDISK3"
Instantiating disk "ASMDISK4"
[root@db02 ~]# oracleasm listdisks
ASMDISK1
ASMDISK2
ASMDISK3
ASMDISK4
```

### 4.9 配置grid 集群 

```bash
$ORACLE_HOME/crs/config/config.sh -jreLoc /etc/alternatives/jre_1.8.0
```

![image-20220717190756740](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717190756740.png)



![image-20220717190912730](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717190912730.png)

### 4.10 配置ssh 互信

```bash
su - grid
[grid@db01 ~]$ ssh-keygen -t dsa
[grid@db01 ~]$ ssh-keygen -t rsa
 
[grid@db01 ~]$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
[grid@db01 ~]$ cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
 
提示：下列命令会提示你输入db02 的grid密码，按照提示输入即可，如果失败可重新尝试执行命令。
db01 节点：
[grid@db01 ~]$ scp ~/.ssh/authorized_keys db02:~/.ssh/authorized_keys
 
db02节点：
[grid@db02 ~]$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
[grid@db02 ~]$ cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
[grid@db02 ~]$ scp ~/.ssh/authorized_keys db01:~/.ssh/authorized_keys
```

![image-20220717191603900](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717191603900.png)

![](https://gitee.com/hgzhangzhen/imgs/raw/bf2a1531fafa94e2894f39d362b3a3bd4cc26d32/image-20220717191630022.png)





![image-20220717191705372](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717191705372.png)

![image-20220717191757765](D:\oracle rac 11g rac for linux 7 install\oracle 11g rac for Linux 7 install.assets\image-20220717191757765.png)

![image-20220717191827410](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717191827410.png)

![image-20220717192116317](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717192116317.png)

![image-20220717192147057](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717192147057.png)

![image-20220717192215857](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717192215857.png)

![image-20220717192255912](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717192255912.png)

在两个节点上分别以root用户执行以上脚本 

```bash
[root@db01 ~]# /u01/11.2.0/grid/root.sh 
Performing root user operation for Oracle 11g 

The following environment variables are set as:
    ORACLE_OWNER= grid
    ORACLE_HOME=  /u01/11.2.0/grid

Enter the full pathname of the local bin directory: [/usr/local/bin]: 
The contents of "dbhome" have not changed. No need to overwrite.
The contents of "oraenv" have not changed. No need to overwrite.
The contents of "coraenv" have not changed. No need to overwrite.

Entries will be added to the /etc/oratab file as needed by
Database Configuration Assistant when a database is created
Finished running generic part of root script.
Now product-specific root actions will be performed.
Relinking oracle with rac_on option
Using configuration parameter file: /u01/11.2.0/grid/crs/install/crsconfig_params
Creating trace directory
User ignored Prerequisites during installation
Installing Trace File Analyzer
OLR initialization - successful
  root wallet
  root wallet cert
  root cert export
  peer wallet
  profile reader wallet
  pa wallet
  peer wallet keys
  pa wallet keys
  peer cert request
  pa cert request
  peer cert
  pa cert
  peer root cert TP
  profile reader root cert TP
  pa root cert TP
  peer pa cert TP
  pa peer cert TP
  profile reader pa cert TP
  profile reader peer cert TP
  peer user cert
  pa user cert
Adding Clusterware entries to oracle-ohasd.service
CRS-2672: Attempting to start 'ora.mdnsd' on 'db01'
CRS-2676: Start of 'ora.mdnsd' on 'db01' succeeded
CRS-2672: Attempting to start 'ora.gpnpd' on 'db01'
CRS-2676: Start of 'ora.gpnpd' on 'db01' succeeded
CRS-2672: Attempting to start 'ora.cssdmonitor' on 'db01'
CRS-2672: Attempting to start 'ora.gipcd' on 'db01'
CRS-2676: Start of 'ora.cssdmonitor' on 'db01' succeeded
CRS-2676: Start of 'ora.gipcd' on 'db01' succeeded
CRS-2672: Attempting to start 'ora.cssd' on 'db01'
CRS-2672: Attempting to start 'ora.diskmon' on 'db01'
CRS-2676: Start of 'ora.diskmon' on 'db01' succeeded
CRS-2676: Start of 'ora.cssd' on 'db01' succeeded

ASM created and started successfully.

Disk Group CRS created successfully.

clscfg: -install mode specified
Successfully accumulated necessary OCR keys.
Creating OCR keys for user 'root', privgrp 'root'..
Operation successful.
CRS-4256: Updating the profile
Successful addition of voting disk 1c514b076ced4f17bf1a814ca4e5a29b.
Successful addition of voting disk 1b18d5e2b47c4f32bfd98f7f1a5a3dfd.
Successful addition of voting disk 0f34576d9f854fe6bfb10be726acc940.
Successfully replaced voting disk group with +CRS.
CRS-4256: Updating the profile
CRS-4266: Voting file(s) successfully replaced
##  STATE    File Universal Id                File Name Disk group
--  -----    -----------------                --------- ---------
 1. ONLINE   1c514b076ced4f17bf1a814ca4e5a29b (/dev/oracleasm/disks/ASMDISK1) [CRS]
 2. ONLINE   1b18d5e2b47c4f32bfd98f7f1a5a3dfd (/dev/oracleasm/disks/ASMDISK2) [CRS]
 3. ONLINE   0f34576d9f854fe6bfb10be726acc940 (/dev/oracleasm/disks/ASMDISK3) [CRS]
Located 3 voting disk(s).
CRS-2672: Attempting to start 'ora.asm' on 'db01'
CRS-2676: Start of 'ora.asm' on 'db01' succeeded
CRS-2672: Attempting to start 'ora.CRS.dg' on 'db01'
CRS-2676: Start of 'ora.CRS.dg' on 'db01' succeeded
Configure Oracle Grid Infrastructure for a Cluster ... succeeded
```

![image-20220717194040664](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717194040664.png)

![image-20220717194237743](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717194237743.png)

安装过程有告警，查看日志得知是scnIP dns 解析问题，由于笔者环境中没有使用到DNS服务器，可以直接忽略。

```
[root@db01 ~]# tail -1000 /u01/oraInventory/logs/configActions2022-07-17_07-07-07-PM.log

INFO: Checking integrity of name service switch configuration file "/etc/nsswitch.conf" ...
Jul 17, 2022 7:41:40 PM oracle.install.commons.util.LogStream log
INFO: All nodes have same "hosts" entry defined in file "/etc/nsswitch.conf"
Jul 17, 2022 7:41:40 PM oracle.install.commons.util.LogStream log
INFO: Check for integrity of name service switch configuration file "/etc/nsswitch.conf" passed
Jul 17, 2022 7:41:40 PM oracle.install.commons.util.LogStream log
INFO: ERROR: 
Jul 17, 2022 7:41:40 PM oracle.install.commons.util.LogStream log
INFO: PRVG-1101 : SCAN name "db-scan" failed to resolve
Jul 17, 2022 7:41:40 PM oracle.install.commons.util.LogStream log
INFO: ERROR: 
```

![image-20220717194851037](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717194851037.png)

## 5.安装oracle software

### 5.1 解压安装包与替换cuv补订文件

```bash
[root@db01 software]# unzip p13390677_112040_Linux-x86-64_1of7.zip && unzip p13390677_112040_Linux-x86-64_2of7.zip 
cp /software/b19404309/database/cvu_prereq.xml  software/database/stage/cvu/cvu_prereq.xml
[root@db01 software]# chown oracle:oinstall ./database -R
[root@db01 software]# chmod 775 -R ./database/
```

### 5.2 配置vncServer

```bash
[oracle@db01 ~]$ vncserver :1

You will require a password to access your desktops.

Password:
Verify:
Would you like to enter a view-only password (y/n)? n
A view-only password is not used
xauth:  file /home/oracle/.Xauthority does not exist

New 'db01:1 (oracle)' desktop is db01:1

Creating default startup script /home/oracle/.vnc/xstartup
Creating default config /home/oracle/.vnc/config
Starting applications specified in /home/oracle/.vnc/xstartup
Log file is /home/oracle/.vnc/db01:1.log

```

![image-20220717200230927](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717200230927.png)



### 5.3 安装oracle 数据库软件

```bash
./runInstaller -jreLoc /etc/alternatives/jre_1.8.0
```

![image-20220717200408342](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717200408342.png)



![image-20220717200615937](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717200615937.png)

![image-20220717200646742](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717200646742.png)

![image-20220717200824521](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717200824521.png)

![image-20220717201008254](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717201008254.png)

![image-20220717201023850](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717201023850.png)

![image-20220717201040201](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717201040201.png)

![image-20220717201134564](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717201134564.png)

![image-20220717201151207](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717201151207.png)

![image-20220717201205232](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717201205232.png)

​       



![image-20220717201418965](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717201418965.png)

这个报错一个BUG 执行下面一段命令

```
sed -i 's/^\(\s*\$(MK_EMAGENT_NMECTL)\)\s*$/\1 -lnnz11/g' "$ORACLE_HOME/sysman/lib/ins_emagent.mk"
```

再点Retry

![image-20220717212132464](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717212132464.png)

分别在两个节点上以root执行这个脚本，到此oracle software 安装完成。

##  6. 安装数据库补订程序 



```bash
[root@db01 software]# unzip p19692824_112040_Linux-x86-64.zip 
   
[root@db01 software]# chmod 777 -R ./19692824
[root@db01 software]# 
[root@db01 software]# 
[root@db01 software]# su - oracle
[oracle@db01 ~]$ cd /software/19692824/
[oracle@db01 19692824]$ $ORACLE_HOME/OPatch/opatch apply

[oracle@db01 19692824]$ $ORACLE_HOME/OPatch/opatch  lsinventory
Oracle Interim Patch Installer version 11.2.0.3.16
Copyright (c) 2022, Oracle Corporation.  All rights reserved.
Oracle Home       : /u01/oracle/product/11.2.0/db_1
Central Inventory : /u01/oraInventory
   from           : /u01/oracle/product/11.2.0/db_1/oraInst.loc
OPatch version    : 11.2.0.3.16
OUI version       : 11.2.0.4.0
Log file location : /u01/oracle/product/11.2.0/db_1/cfgtoollogs/opatch/opatch2022-07-17_21-46-39PM_1.log

Lsinventory Output file location : /u01/oracle/product/11.2.0/db_1/cfgtoollogs/opatch/lsinv/lsinventory2022-07-17_21-46-39PM.txt
--------------------------------------------------------------------------------
Local Machine Information::
Hostname: db01
ARU platform id: 226
ARU platform description:: Linux x86-64

Installed Top-level Products (1): 

Oracle Database 11g                                                  11.2.0.4.0
There are 1 products installed in this Oracle Home.
Interim patches (1) :
Patch  19692824     : applied on Sun Jul 17 21:45:16 CST 2022
Unique Patch ID:  18333234
   Created on 1 Dec 2014, 10:00:32 hrs GMT
   Bugs fixed:
     19692824
--------------------------------------------------------------------------------
OPatch succeeded.
```

## 7. 创建oracle 实例

### 7.1 创建数据磁盘vnc连接到grid 用户下，打开终端执行asmca

![image-20220717215341382](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717215341382.png)

![image-20220717215510877](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717215510877.png)

![image-20220717215605003](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717215605003.png)

### 7.2 vnc连接到oracle 用户下，打开终端执行dbca

![image-20220717214936211](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717214936211.png)

![image-20220717215014729](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717215014729.png)

![image-20220717215049515](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717215049515.png)

![image-20220717215114082](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717215114082.png)

![image-20220717215647620](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717215647620.png)

![image-20220717215735844](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717215735844.png)

![image-20220717215800861](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717215800861.png)

![image-20220717215838895](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717215838895.png)

![image-20220717220204873](https://gitee.com/hgzhangzhen/imgs/raw/master/image-20220717220204873.png)

修改db01环境变量实例名

```bash
[root@db02 ~]# sed -i 's/export ORACLE_SID=oracl2/export ORACLE_SID=orcl2/g' /home/oracle/.bash_profile 
[root@db02 ~]# sed -i 's/export ORACLE_SID=+ASM1/export ORACLE_SID=+ASM2/g' /home/grid/.bash_profile  
```

### 7.3 测试RAC 读写

```bash
su - oracle
sqlplus / as sysdba;
SQL> select HOST_NAME,instance_name,status from gv$instance;

HOST_NAME            INSTANCE_NAME                                    STATUS
-------------------- ------------------------------------------------ ------------------------------------
db02                 orcl2                                            OPEN
db01                 orcl1                                            OPEN
SQL> create table t1 (id int,first_name varchar2(20),last_name varchar2(20));

Table created
SQL> create table t1 (id int,first_name varchar2(20),last_name varchar2(20));

Table created.

SQL> insert into t1 (id,last_name) values (100,'zhang');

1 row created.

SQL> insert into t1 values(1,'li','si');

1 row created.
SQL> select * from t1;

        ID FIRST_NAME           LAST_NAME
---------- -------------------- ------------------------------------------------------------
       100                      zhang
         1 li                   si
在节点2上查询并更新表t1;
SQL> update t1 set last_name='san',first_name='zhang' where id=100;

1 row updated.

SQL> col FIRST_NAME for a20;
SQL> select * from t1;

        ID FIRST_NAME           LAST_NAME
---------- -------------------- ------------------------------------------------------------
       100 zhang                san
         1 li                   si
```

本篇完！
