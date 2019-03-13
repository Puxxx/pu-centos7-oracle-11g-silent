# 前言 (introduction)
主机名称(localhost)、主机域名(localdomain) below should change to the right name for localhost.localdomain.
/u01/app could change to anywhere you like，db_1 also can be remove. 

## 1.配置host等信息：
[root@CentOS tmp]# vim /etc/hosts
文件尾添加192.168.xx.xx 主机名称.主机域名

#修改内核参数

vim /etc/sysctl.conf

#添加以下内容

fs.aio-max-nr = 1048576

fs.file-max = 6815744

kernel.shmall = 2097152

kernel.shmmax = 1073741824

kernel.shmmni = 4096

kernel.sem = 250 32000 100 128

net.ipv4.ip_local_port_range = 9000 65500

net.core.rmem_default = 262144

net.core.rmem_max = 4194304

net.core.wmem_default = 262144

net.core.wmem_max = 1048576

#使内核新配置生效

sysctl -p

#修改用户限制

vim /etc/security/limits.conf

#添加以下内容

oracle           soft    nproc           2047

oracle           hard    nproc           16384

oracle           soft    nofile          1024

oracle           hard    nofile          65536

oracle           soft    stack           10240

#修改/etc/pam.d/login文件

vim /etc/pam.d/login

#添加以下内容

session  required   /lib64/security/pam_limits.so

session  required   pam_limits.so

#修改/etc/profile文件

vim /etc/profile

#添加以下内容

if [ $USER = "oracle" ]; then

  if [ $SHELL = "/bin/ksh" ]; then
  
   ulimit -p 16384
   
   ulimit -n 65536
   
  else
  
   ulimit -u 16384 -n 65536
   
  fi
  
fi

## 2.防火墙设置：

firewall-cmd --zone=public --add-port=1521/tcp --add-port=5500/tcp --add-port=5520/tcp --add-port=3938/tcp --permanent

firewall-cmd --reload

firewall-cmd --list-ports

或者

vim /etc/selinux/config

设置SELINUX=disabled

setenforce 0

service iptables stop

systemctl stop firewalld

systemctl disable firewalld


## 3.安装依赖包-二选一：(choose one way to execute)

yum install -y  binutils compat-libstdc++-33 elfutils-libelf elfutils-libelf-devel elfutils-libelf-devel-static gcc gcc-c++ glibc glibc-common glibc-devel glibc-headers glibc-static kernel-headers pdksh libaio libaio-devel libgcc libgomp libstdc++ libstdc++-devel libstdc++-static make numactl-devel sysstat unixODBC unixODBC-devel

yum -y install binutils compat-libcap1 compat-libstdc++-33 compat-libstdc++-33*i686 gcc gcc-c++ glibc glibc*.i686 glibc-devel glibc-devel*.i686 ksh libaio libaio*.i686 libaio-devel libgcc libgcc*.i686 libstdc++ libstdc++*.i686 libstdc++-devel libXi libXi*.i686 libXtst libXtst*.i686 make sysstat unixODBC unixODBC*.i686 unixODBC-devel unixODBC-devel*.i686

## 4.创建用户

groupadd oinstall

groupadd dba

useradd -g oinstall -g dba -m oracle

passwd oracle

## 5.创建目录

mkdir -p /u01/app/oracle

mkdir -p /u01/app/oraInventory

mkdir -p /u01/app/database

mkdir -p /u01/app/oracle/product/11.2.0/db_1

mkdir -p /u01/app/oracle/oradata

mkdir -p /u01/app/oracle/flash_recovery_area

chown -R oracle:oinstall /u01/app/oracle

chown -R oracle:oinstall /u01/app/oracle/oradata

chown -R oracle:oinstall /u01/app/oraInventory

chown -R oracle:oinstall /u01/app/database

chmod -R 775 /u01/app/oracle

## 6.配置环境：

su - oracle

vim .bash_profile  添加如下内容：


ORACLE_HOSTNAME=主机名称.主机域名; export ORACLE_HOSTNAME

ORACLE_UNQNAME=数据库名称; export ORACLE_UNQNAME

ORACLE_BASE=/u01/app/oracle; export ORACLE_BASE

ORACLE_HOME=$ORACLE_BASE/product/11.2.0/db_1; export ORACLE_HOME

ORACLE_SID=数据库名称;; export ORACLE_SID

ORACLE_TERM=xterm; export ORACLE_TERM

PATH=/usr/sbin:$PATH; export PATH

PATH=$ORACLE_HOME/bin:$PATH; export PATH

LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib; export LD_LIBRARY_PATH

CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib; export CLASSPATH

## 7.安装数据库

cd ~

cp -R /u01/app/database/response/ . && cd response/

vim db_install.rsp

oracle.install.option=INSTALL_DB_SWONLY

ORACLE_HOSTNAME=主机名称

UNIX_GROUP_NAME=oinstall

INVENTORY_LOCATION=/u01/app/oracle/inventory

SELECTED_LANGUAGES=en,zh_CN

ORACLE_HOME=/u01/app/oracle/product/11.2.0/db_1

ORACLE_BASE=/u01/app/oracle

oracle.install.db.InstallEdition=EE

oracle.install.db.DBA_GROUP=dba

oracle.install.db.OPER_GROUP=dba

DECLINE_SECURITY_UPDATES=true

#cd 数据库安装文件所在目录

./runInstaller -silent -responseFile /home/oracle/response/db_install.rsp -ignorePrereq

#可查看日志

tail -f /data/oracle/inventory/logs/installActions2015-06-08_04-00-25PM.log

## 8.启动监听

#注意此处，必须使用/silent /responseFile格式，而不是-silent -responseFile，因为是静默安装

netca /silent /responseFile /home/oracle/response/netca.rsp

#监听操作

lsnrctl status

## 9.建立新库：

vim /home/oracle/response/dbca.rsp

GDBNAME= "数据库名称"

SID =" 数据库名称"

SYSPASSWORD= " password"

SYSTEMPASSWORD= "password"

SYSMANPASSWORD= " password"

DBSNMPPASSWORD= " password"

DATAFILEDESTINATION=/data/oracle/oradata

RECOVERYAREADESTINATION=/data/oracle/fast_recovery_area

CHARACTERSET= "ZHS16GBK"

#内存随便设置

TOTALMEMORY= "3000"

dbca -silent -responseFile /home/oracle/response/dbca.rsp

## 10.设置开机自启动

vim /u01/app/oracle/product/11.2.0/db_1/bin/dbstart

#将ORACLE_HOME_LISTNER=$1修改为ORACLE_HOME_LISTNER=$ORACLE_HOME

vim /u01/app/oracle/product/11.2.0/db_1/bin/dbshut

#将ORACLE_HOME_LISTNER=$1修改为ORACLE_HOME_LISTNER=$ORACLE_HOME

vim /etc/oratab

#将orcl:/data/oracle/product/11.2.0:N中最后的N改为Y，成为orcl:/data/oracle/product/11.2.0:Y


