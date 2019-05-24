---
layout:     post
title:      "OpenStack手动部署指南 "
subtitle:   ""
date:       2019-05-23 09:14:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
    - OpenStack
---


## 一.服务器基础配置

1.如果是新的服务器,先安装常用软件,配制yum源,关闭防火墙和selinux等,基本操作如下

```
yum install lrzsz net-tools ntp vim tree wget -y
```

```
systemctl stop firewalld && systemctl disable firewalld
```

```
setenforce 0
```

```
vim /etc/sysconfig/selinux
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled  ##这里改为disabled
# SELINUXTYPE= can take one of these two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

配置epel的yum源

```
rpm -Uvh http://mirrors.zju.edu.cn/epel/7Server/x86_64/e/epel-release-7-10.noarch.rpm  ##直接运行这条命令,经过我自己的测试,这个epel源的速度还是比较快的
yum install yum-plugin-fastestmirror -y   ##安装这个yum插件,可以在yum安装软件的时候选择速度比较快的源
```

yum源配置结束后可以使用下面的命令,检测并重新生成yum缓存

```
yum repolist
yum clean all && yum makecache
```

基础配置完成后最好能够重启一下服务器  
下面我来说明一下本次安装openstack的拓扑图(很粗糙,不要见笑)

![](https://upload-images.jianshu.io/upload_images/7715809-86ed0ab355b93097.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/747/format/webp)





从上面的拓扑图可以看出来,OpenStack并不是一个软件,它是由多个工具组合而成的一个集合体,这也是很多初学者不能很快搭建出OpenStack的原因.个人觉得,多搭建几遍,了解它每个工具的作用,然后很多东西就自然而然的通了.  
以上的基础操作,是需要在管理节点和计算节点都要操作的,下面进入正式的安装

## 二.OpenStack安装

###### 1. 在两个服务器节点安装OpenStack的软件仓库,安装OpenStack的客户端和selinux的管理工具

```
[root@Marvin-OpenStack ~]# yum install http://mirrors.zju.edu.cn/centos/7.3.1611/cloud/x86_64/openstack-newton/centos-release-openstack-newton-1-1.el7.noarch.rpm -y  ## 安装OpenStack的软件仓库
[root@Marvin-OpenStack ~]#  yum install python-openstackclient -y  ## 安装Openstack客户端
[root@Marvin-Compute ~]# yum install openstack-selinux -y  ## 安装selinux管理工具
```

其实上面这一步应该也算做基础配置当中,下面进行正式安装管理节点的软件

##### 2.在管理节点安装MariaDB,并修改配置做初始化操作

```
[root@Marvin-OpenStack ~]# yum install mariadb mariadb-server python2-PyMySQL -y
在目录/etc/my.cnf.d/下创建openstack.cnf文件
[root@Marvin-OpenStack ~]# vim /etc/my.cnf.d/openstack.cnf
[mysqld]
bind-address = 10.0.0.56
default-storage-engine = innodb
innodb_file_per_table
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
[root@Marvin-OpenStack ~]# systemctl enable mariadb && systemctl start mariadb ## 启动mariadb并设置开机自启
Created symlink from /etc/systemd/system/multi-user.target.wants/mariadb.service to /usr/lib/systemd/system/mariadb.service.
[root@Marvin-OpenStack ~]# lsof -i:3306  ## 查看数据库是否启动
COMMAND  PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
mysqld  4299 mysql   17u  IPv4  33131      0t0  TCP Marvin-OpenStack:mysql (LISTEN)
[root@Marvin-OpenStack ~]# mysql_secure_installation  ## 初始化数据库

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none): 
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] y   ##这里设置数据库密码,在学习阶段密码尽可能的简单,我这里是123456
New password: 
Re-enter new password: 
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] 
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] 
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] 
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] 
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
在初始化数据库完成后,一定要使用root用户来登录,验证数据库是否可以正常登录
[root@Marvin-OpenStack ~]# mysql -uroot -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 10
Server version: 10.1.20-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> exit
Bye
```

数据库安装完成!!!

##### 3.管理节点安装rabbitmq消息队列

```
[root@Marvin-OpenStack ~]# yum install rabbitmq-server -y  ## 安装
[root@Marvin-OpenStack ~]# systemctl enable rabbitmq-server && systemctl start rabbitmq-server  ## 启动
Created symlink from /etc/systemd/system/multi-user.target.wants/rabbitmq-server.service to /usr/lib/systemd/system/rabbitmq-server.service.
创建openstack用户,并附加权限
[root@Marvin-OpenStack ~]# rabbitmqctl add_user openstack openstack  ## 创建用户
Creating user "openstack" ...
[root@Marvin-OpenStack ~]# rabbitmqctl set_permissions openstack ".*" ".*" ".*"   ## 附加权限
Setting permissions for user "openstack" in vhost "/" ...
[root@Marvin-OpenStack ~]# rabbitmq-plugins enable rabbitmq_management  ## 启动web管理界面,使用rabbitmq-plugins list可以列出rabbit所有的可用插件
The following plugins have been enabled:
  mochiweb
  webmachine
  rabbitmq_web_dispatch
  amqp_client
  rabbitmq_management_agent
  rabbitmq_management

Applying plugin configuration to rabbit@Marvin-OpenStack... started 6 plugins.
[root@Marvin-OpenStack ~]# lsof -i:15672  ## 养成习惯,查看服务是否启动,rabbitmq的服务端口是15672
COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
beam.smp 4507 rabbitmq   54u  IPv4  37049      0t0  TCP *:15672 (LISTEN)
```

使用web登录验证,登录地址: http://10.0.0.56:15672 用户名密码均为:guest

![](//upload-images.jianshu.io/upload_images/7715809-ba615315e25cbd11.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

Paste_Image.png

rabbitmq消息队列安装完成!!!

##### 4.管理节点安装Keystone认证服务

在安装keystone服务之前,应该创建相应的数据库,并且授权;  
为了方便,我们一次性将所有用到的数据库都创建完成,分别是:  
keystone, glance, nova, nova_api, neutron, cinder  
需要注意的是,在对数据库进行授权的时候,我这里默认密码也和数据库名称相同,方便记忆也不会出错

```
[root@Marvin-OpenStack ~]# mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 11
Server version: 10.1.20-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> create database keystone;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> grant all on keystone.* to 'keystone'@'localhost' identified by 'keystone';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> grant all on keystone.* to 'keystone'@'%' identified by 'keystone';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> create database glance;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> grant all on glance.* to 'glance'@'localhost' identified by 'glance';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> grant all on glance.* to 'glance'@'%' identified by 'glance';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> create database nova;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> grant all on nova.* to 'nova'@'%' identified by 'nova';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> grant all on nova.* to 'nova'@'localhost' identified by 'nova';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> create database nova_api;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> grant all on nova_api.* to 'nova'@'localhost' identified by 'nova';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> grant all on nova_api.* to 'nova'@'%' identified by 'nova';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> create database neutron;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> grant all on neutron.* to 'neutron'@'localhost' identified by 'neutron';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> grant all on neutron.* to 'neutron'@'%' identified by 'neutron'; 
Query OK, 0 rows affected (0.01 sec)

MariaDB [(none)]> create database cinder;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> grant all on cinder.* to 'cinder'@'localhost' identified by 'cinder';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> grant all on cinder.* to 'cinder'@'%' identified by 'cinder'; 
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| cinder             |
| glance             |
| information_schema |
| keystone           |
| mysql              |
| neutron            |
| nova               |
| nova_api           |
| performance_schema |
+--------------------+
9 rows in set (0.00 sec)

MariaDB [(none)]> exit
Bye
```

创建数据库授权是件需要很细心的工作,一定不能出错,否走后面会出现很多你意想不到的错误,下面去安装配置keystone

```
[root@Marvin-OpenStack ~]# yum install openstack-keystone httpd mod_wsgi -y  ## 安装
[root@Marvin-OpenStack ~]# vim /etc/keystone/keystone.conf  ## 修改配置文件
[database]  ## 模块名称
connection = mysql+pymysql://keystone:keystone@10.0.0.56/keystone   ## 大约在640行
[memcache]
servers = 10.0.0.56:11211 ## 大约在1476行
[token]
provider = fernet  ## 大约在2659行
driver = memcache  ## 大约在2669行
[root@Marvin-OpenStack ~]# grep '^[a-z]' /etc/keystone/keystone.conf   ## 可以看看一共修改了的内容
connection = mysql+pymysql://keystone:keystone@10.0.0.56/keystone   ## 数据库
servers = 10.0.0.56:11211    ## memcache
provider = fernet          ## token的提供者
driver = memcache      ## token的存放位置
可以看出来我们都有配制memcache,但是并没有安装memcache,现在安装
[root@Marvin-OpenStack ~]# yum install memcached python-memcached -y
[root@Marvin-OpenStack ~]# vim /etc/sysconfig/memcached
PORT="11211"
USER="memcached"
MAXCONN="1024"
CACHESIZE="64"
OPTIONS="-l 10.0.0.56,::1"  ## 这里修改成自己主机的IP地址
[root@Marvin-OpenStack ~]# systemctl enable memcached && systemctl start memcached  ## 启动memcache
Created symlink from /etc/systemd/system/multi-user.target.wants/memcached.service to /usr/lib/systemd/system/memcached.service.
[root@Marvin-OpenStack ~]# lsof -i:11211  ## 检查
COMMAND    PID      USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
memcached 5672 memcached   26u  IPv4  40367      0t0  TCP Marvin-OpenStack:memcache (LISTEN)
memcached 5672 memcached   27u  IPv6  40368      0t0  TCP localhost:memcache (LISTEN)
memcached 5672 memcached   28u  IPv4  40369      0t0  UDP Marvin-OpenStack:memcache 
memcached 5672 memcached   29u  IPv6  40370      0t0  UDP localhost:memcache
现在来同步keystone的数据库信息
[root@Marvin-OpenStack ~]# su -s /bin/sh -c "keystone-manage db_sync" keystone  ## 同步
[root@Marvin-OpenStack ~]# mysql -h 10.0.0.56 -ukeystone -pkeystone -e "use keystone;show tables;"  ## 验证
+------------------------+
| Tables_in_keystone     |
+------------------------+
| access_token           |
| assignment             |
| config_register        |
| consumer               |
| credential             |
| endpoint               |
| endpoint_group         |
| federated_user         |
| federation_protocol    |
| group                  |
| id_mapping             |
| identity_provider      |
| idp_remote_ids         |
| implied_role           |
| local_user             |
| mapping                |
| migrate_version        |
| nonlocal_user          |
| password               |
| policy                 |
| policy_association     |
| project                |
| project_endpoint       |
| project_endpoint_group |
| region                 |
| request_token          |
| revocation_event       |
| role                   |
| sensitive_config       |
| service                |
| service_provider       |
| token                  |
| trust                  |
| trust_role             |
| user                   |
| user_group_membership  |
| whitelisted_config     |
+------------------------+
看到数据库的表,就说明OK了,现在去注册keystone的相关服务
[root@Marvin-OpenStack ~]# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
[root@Marvin-OpenStack ~]# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
[root@Marvin-OpenStack ~]# keystone-manage bootstrap --bootstrap-password admin \
>     --bootstrap-admin-url http://10.0.0.56:35357/v3/ \
>     --bootstrap-internal-url http://10.0.0.56:35357/v3/ \
>     --bootstrap-public-url http://10.0.0.56:5000/v3/ \
>     --bootstrap-region-id RegionOn
完成后登录数据库,检查注册是否成功
[root@Marvin-OpenStack ~]# mysql -ukeystone -pkeystone
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 15
Server version: 10.1.20-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> use keystone;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [keystone]> show tables;
+------------------------+
| Tables_in_keystone     |
+------------------------+
| access_token           |
| assignment             |
| config_register        |
| consumer               |
| credential             |
| endpoint               |
| endpoint_group         |
| federated_user         |
| federation_protocol    |
| group                  |
| id_mapping             |
| identity_provider      |
| idp_remote_ids         |
| implied_role           |
| local_user             |
| mapping                |
| migrate_version        |
| nonlocal_user          |
| password               |
| policy                 |
| policy_association     |
| project                |
| project_endpoint       |
| project_endpoint_group |
| region                 |
| request_token          |
| revocation_event       |
| role                   |
| sensitive_config       |
| service                |
| service_provider       |
| token                  |
| trust                  |
| trust_role             |
| user                   |
| user_group_membership  |
| whitelisted_config     |
+------------------------+
37 rows in set (0.00 sec)

MariaDB [keystone]> select * from user\G
*************************** 1. row ***************************
                id: 4799fa14964b4703bd84a93d3b792660
             extra: {}
           enabled: 1
default_project_id: NULL
        created_at: 2017-09-02 02:36:15
    last_active_at: NULL
1 row in set (0.00 sec)

MariaDB [keystone]> select * from role\G
*************************** 1. row ***************************
       id: 097281195831406fa3e3782465aa922a
     name: admin
    extra: {}
domain_id: <<null>>
*************************** 2. row ***************************
       id: 9fe2ff9ee4384b1894a90878d3e92bab
     name: _member_
    extra: {}
domain_id: <<null>>
2 rows in set (0.00 sec)

MariaDB [keystone]> select * from endpoint\G
*************************** 1. row ***************************
                id: 41838de3d5204c9dbf173321b8bc05d7
legacy_endpoint_id: NULL
         interface: internal
        service_id: 1575e305f3aa4750b10304f38aa54b5e
               url: http://10.0.0.56:35357/v3/
             extra: {}
           enabled: 1
         region_id: RegionOne
*************************** 2. row ***************************
                id: d8b9d6bf135b482e852c6483ce30e4f9
legacy_endpoint_id: NULL
         interface: admin
        service_id: 1575e305f3aa4750b10304f38aa54b5e
               url: http://10.0.0.56:35357/v3/
             extra: {}
           enabled: 1
         region_id: RegionOne
*************************** 3. row ***************************
                id: e83d3ee63f2a473e809931900ec71174
legacy_endpoint_id: NULL
         interface: public
        service_id: 1575e305f3aa4750b10304f38aa54b5e
               url: http://10.0.0.56:5000/v3/
             extra: {}
           enabled: 1
         region_id: RegionOne
3 rows in set (0.00 sec)

MariaDB [keystone]> exit
Bye
keystone就配置好了,去开启apache服务了
[root@Marvin-OpenStack ~]# vim /etc/httpd/conf/httpd.conf
95 ServerName 10.0.0.56:80  ## 第95行修改apache的服务地址
[root@Marvin-OpenStack ~]# ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/  ## 将keystone配置文件链接到apache默认目录下
[root@Marvin-OpenStack ~]# systemctl enable httpd && systemctl start httpd  ## 启动
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
[root@Marvin-OpenStack ~]# lsof -i:80  ## 验证
COMMAND  PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
httpd   5886   root    4u  IPv6  43065      0t0  TCP *:http (LISTEN)
httpd   5897 apache    4u  IPv6  43065      0t0  TCP *:http (LISTEN)
httpd   5898 apache    4u  IPv6  43065      0t0  TCP *:http (LISTEN)
httpd   5899 apache    4u  IPv6  43065      0t0  TCP *:http (LISTEN)
httpd   5900 apache    4u  IPv6  43065      0t0  TCP *:http (LISTEN)
httpd   5901 apache    4u  IPv6  43065      0t0  TCP *:http (LISTEN)
验证性息
[root@Marvin-OpenStack ~]# openstack user list
Missing value auth-url required for auth plugin password  ## 这里不成功,是因为环境变量没有配制
配置环境变量
[root@Marvin-OpenStack ~]# export OS_USERNAME=admin
[root@Marvin-OpenStack ~]# export OS_PASSWORD=admin
[root@Marvin-OpenStack ~]# export OS_PROJECT_NAME=admin
[root@Marvin-OpenStack ~]# export OS_USER_DOMAIN_NAME=default
[root@Marvin-OpenStack ~]# export OS_PROJECT_DOMAIN_NAME=default
[root@Marvin-OpenStack ~]# export OS_AUTH_URL=http://10.0.0.56:35357/v3
[root@Marvin-OpenStack ~]# export OS_IDENTITY_API_VERSION=3
[root@Marvin-OpenStack ~]# openstack user  list
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| 4799fa14964b4703bd84a93d3b792660 | admin |
+----------------------------------+-------+
```

keystone配置完成

##### 5.创建项目,用户,角色,并为各用户赋予角色和项目,按照要求配置环境变量

```
## 创建service项目
[root@Marvin-OpenStack ~]# openstack project create --domain default \
>     --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 692ff9bb43a14bbca347d1151d9bb996 |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
+-------------+----------------------------------+
## 创建demo项目
[root@Marvin-OpenStack ~]# openstack project create --domain default \
>      --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 9e6ed2044de448c5b6064da5e61108f3 |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | default                          |
+-------------+----------------------------------+
## 创建demo用户,密码也为demo
[root@Marvin-OpenStack ~]# openstack user create --domain default \
>      --password-prompt demo
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | bccfde0f0711448c8e3855ac4dcb8e19 |
| name                | demo                             |
| password_expires_at | None                             |
+---------------------+----------------------------------+
## 创建user角色
[root@Marvin-OpenStack ~]# openstack role create user
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | f2c217d721874682b7987d5e6a363905 |
| name      | user                             |
+-----------+----------------------------------+
## 把demo用户加入到demo项目,并赋予user角色的权限
[root@Marvin-OpenStack ~]# openstack role add --project demo --user demo user
## 用同样的方法建立glance, nova, neutron, cinder 用户,并将他们加入service项目,赋予admin角色权限
[root@Marvin-OpenStack ~]# openstack user create --domain default   --password-prompt glance  ## 创建glance用户
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 7f9ad6700c9a4c5b9c2a9b788dc623e1 |
| name                | glance                           |
| password_expires_at | None                             |
+---------------------+----------------------------------+
[root@Marvin-OpenStack ~]# openstack role add --project service --user glance admin  ## 加入项目和角色
[root@Marvin-OpenStack ~]# openstack user create --domain default   --password-prompt nova
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 091ccd11a04c40c0a764f7ab988fe572 |
| name                | nova                             |
| password_expires_at | None                             |
+---------------------+----------------------------------+
[root@Marvin-OpenStack ~]# openstack role add --project service --user nova admin
[root@Marvin-OpenStack ~]# openstack user create --domain default   --password-prompt neutron
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 02cb936cd3c744d3aabf5a775e8d8f92 |
| name                | neutron                          |
| password_expires_at | None                             |
+---------------------+----------------------------------+
[root@Marvin-OpenStack ~]# openstack role add --project service --user neutron admin
[root@Marvin-OpenStack ~]# openstack user create --domain default   --password-prompt cinder
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 4a1a121dd6194784a625ce842aa96b56 |
| name                | cinder                           |
| password_expires_at | None                             |
+---------------------+----------------------------------+
[root@Marvin-OpenStack ~]# openstack role add --project service --user cinder admin
## 验证admin和demo是否可以申请到token的令牌,需要注意的是:admin和demo请求的端口是不一样的,admin是35357端口,demo使用的是5000端口
[root@Marvin-OpenStack ~]# unset OS_AUTH_URL OS_PASSWORD
[root@Marvin-OpenStack ~]# openstack --os-auth-url http://10.0.0.56:35357/v3 \
>     --os-project-domain-name default --os-user-domain-name default \
>     --os-project-name admin --os-username admin token issue
Password:   ## 输入admin密码
+------------+--------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                        |
+------------+--------------------------------------------------------------------------------------------------------------+
| expires    | 2017-09-02 14:12:33+00:00                                                                                    |
| id         | gAAAAABZqq5BuqekQMg2ngLYfbk-uePRrhgk9MmI2Z2J9-vQ6NeGhlwzkOPA__A2byWM-                                        |
|            | JN68q7CngGcCvJsHnefk1FOSrQgHrEOOkoJm7_oEqGHmL2_yH09h-TWv148ug78dJA4xmcJA8glI7-HrYe_D-                        |
|            | wJCCL9eZfh9gylPCtFDt05rISeK6k                                                                                |
| project_id | 8e8448db75034b1e8be0f7d6931be2d4                                                                             |
| user_id    | 4799fa14964b4703bd84a93d3b792660                                                                             |
+------------+--------------------------------------------------------------------------------------------------------------+
[root@Marvin-OpenStack ~]# openstack --os-auth-url http://10.0.0.56:5000/v3 \
>     --os-project-domain-name default --os-user-domain-name default \
>     --os-project-name demo --os-username demo token issue
Password:   ## 输入demo密码
+------------+--------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                        |
+------------+--------------------------------------------------------------------------------------------------------------+
| expires    | 2017-09-02 14:13:29+00:00                                                                                    |
| id         | gAAAAABZqq55u-TC1n7b3w2uRjuJn3GIRcHbXyznI9slRgSx-hfsuQhfJsQR4M91UxZoObyH7UXxafE0_MHGe8_y1w-_0M4yxJL6Q5uf-    |
|            | nmwTAbB1Vj4y_cQKq-w6ldAlV9UeW2ijP4BZVEg9JVqaBmvs-YaP0yEcq9U9S9fA6jrtHpsX1ScUOQ                               |
| project_id | 9e6ed2044de448c5b6064da5e61108f3                                                                             |
| user_id    | bccfde0f0711448c8e3855ac4dcb8e19                                                                             |
+------------+--------------------------------------------------------------------------------------------------------------+
## 验证是没有问题的,现在配制admin和demo的环境变量脚本
[root@Marvin-OpenStack ~]# vim admin-openstack
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_AUTH_URL=http://10.0.0.56:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
[root@Marvin-OpenStack ~]# vim demo-openstack
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=demo
export OS_AUTH_URL=http://10.0.0.56:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
## 验证脚本
[root@Marvin-OpenStack ~]# source admin-openstack   ## 切换到admin变量
[root@Marvin-OpenStack ~]# openstack token issue   ## 执行命令成功
+------------+--------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                        |
+------------+--------------------------------------------------------------------------------------------------------------+
| expires    | 2017-09-02 14:18:09+00:00                                                                                    |
| id         | gAAAAABZqq-Rbg_PBi20TjUrf-U0tj0aqQR1ukbIBT_Evki13y0nZqMKBBnjnE3tsmLbtVUCoHP4w33MLWhGzFPgPoP-                 |
|            | GwhqiJxNUAsVRD0fi7tvtzqYelkqsCy5kkxggjLlfX54V0xzsVLqpgD2QJyZIiRJgk_FzunnTK4X5A9cjbk9MTqjo4A                  |
| project_id | 8e8448db75034b1e8be0f7d6931be2d4                                                                             |
| user_id    | 4799fa14964b4703bd84a93d3b792660                                                                             |
+------------+--------------------------------------------------------------------------------------------------------------+
[root@Marvin-OpenStack ~]# source demo-openstack   ## 切换到demo变量
[root@Marvin-OpenStack ~]# openstack token issue   ## 执行命令成功
+------------+--------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                        |
+------------+--------------------------------------------------------------------------------------------------------------+
| expires    | 2017-09-02 14:18:30+00:00                                                                                    |
| id         | gAAAAABZqq-nLZ-rhs6GseaHcVX22mxB2JkcJ19xL_CQUuC5UeEVMzUyyplqtbEAgs4ulvAb8zKcxcj3Pe3HYyttJSrtVpDxIiEZfGQBaLfl |
|            | OUQvEvkG9BBbXoVv2_dZ259CLjNE0hILzsu6e7b59RSgqdtbyOeLGPITHSbxFgS8aq6mpXidDvE                                  |
| project_id | 9e6ed2044de448c5b6064da5e61108f3                                                                             |
| user_id    | bccfde0f0711448c8e3855ac4dcb8e19                                                                             |
+------------+--------------------------------------------------------------------------------------------------------------+
```

用户,角色,变量配置完成

##### 6.安装配置glance镜像服务

###### 6.1 创建image服务实体

```
[root@Marvin-OpenStack ~]# source admin-openstack  ## 切换到admin变量
[root@Marvin-OpenStack ~]# openstack service create --name glance   --description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 23c1ddf9a59048aea91fcce887ca97dc |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```

###### 6.2创建glance的API端口,分别创建public,internal,admin

```
[root@Marvin-OpenStack ~]# openstack endpoint create --region RegionOne \
>     image public http://10.0.0.56:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 0e28a8c975904c28b5a2a9bc180efe02 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 23c1ddf9a59048aea91fcce887ca97dc |
| service_name | glance                           |
| service_type | image                            |
| url          | http://10.0.0.56:9292            |
+--------------+----------------------------------+
[root@Marvin-OpenStack ~]# openstack endpoint create --region RegionOne     image internal http://10.0.0.56:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 2cf91a5b1e1e4d19830627a1d1088b40 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 23c1ddf9a59048aea91fcce887ca97dc |
| service_name | glance                           |
| service_type | image                            |
| url          | http://10.0.0.56:9292            |
+--------------+----------------------------------+
[root@Marvin-OpenStack ~]# openstack endpoint create --region RegionOne     image admin http://10.0.0.56:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 806d8539a4f44c66b14488ee1979190d |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 23c1ddf9a59048aea91fcce887ca97dc |
| service_name | glance                           |
| service_type | image                            |
| url          | http://10.0.0.56:9292            |
+--------------+----------------------------------+
## 查看创建信息
[root@Marvin-OpenStack ~]# openstack service list
+----------------------------------+----------+----------+
| ID                               | Name     | Type     |
+----------------------------------+----------+----------+
| 1575e305f3aa4750b10304f38aa54b5e | keystone | identity |
| 23c1ddf9a59048aea91fcce887ca97dc | glance   | image    |
+----------------------------------+----------+----------+
[root@Marvin-OpenStack ~]# openstack endpoint list
+-----------------------------+-----------+--------------+--------------+---------+-----------+----------------------------+
| ID                          | Region    | Service Name | Service Type | Enabled | Interface | URL                        |
+-----------------------------+-----------+--------------+--------------+---------+-----------+----------------------------+
| 0e28a8c975904c28b5a2a9bc180 | RegionOne | glance       | image        | True    | public    | http://10.0.0.56:9292      |
| efe02                       |           |              |              |         |           |                            |
| 2cf91a5b1e1e4d19830627a1d10 | RegionOne | glance       | image        | True    | internal  | http://10.0.0.56:9292      |
| 88b40                       |           |              |              |         |           |                            |
| 41838de3d5204c9dbf173321b8b | RegionOne | keystone     | identity     | True    | internal  | http://10.0.0.56:35357/v3/ |
| c05d7                       |           |              |              |         |           |                            |
| 806d8539a4f44c66b14488ee197 | RegionOne | glance       | image        | True    | admin     | http://10.0.0.56:9292      |
| 9190d                       |           |              |              |         |           |                            |
| d8b9d6bf135b482e852c6483ce3 | RegionOne | keystone     | identity     | True    | admin     | http://10.0.0.56:35357/v3/ |
| 0e4f9                       |           |              |              |         |           |                            |
| e83d3ee63f2a473e809931900ec | RegionOne | keystone     | identity     | True    | public    | http://10.0.0.56:5000/v3/  |
| 71174                       |           |              |              |         |           |                            |
+-----------------------------+-----------+--------------+--------------+---------+-----------+----------------------------+
```

###### 6.3 glance安装配置,并同步数据信息

```
[root@Marvin-OpenStack ~]# yum install openstack-glance -y
[root@Marvin-OpenStack ~]# vim /etc/glance/glance-api.conf
[database]
connection = mysql+pymysql://glance:glance@10.0.0.56/glance  ## 大约在1748行修改数据库
[root@Marvin-OpenStack ~]# vim /etc/glance/glance-registry.conf
[database]
connection = mysql+pymysql://glance:glance@10.0.0.56/glance  ## 大约在1038行修改数据
[root@Marvin-OpenStack ~]# su -s /bin/sh -c "glance-manage db_sync" glance  ## 导入数据库
Option "verbose" from group "DEFAULT" is deprecated for removal.  Its value may be silently ignored in the future.
/usr/lib/python2.7/site-packages/oslo_db/sqlalchemy/enginefacade.py:1171: OsloDBDeprecationWarning: EngineFacade is deprecated; please use oslo_db.sqlalchemy.enginefacade
  expire_on_commit=expire_on_commit, _conf=conf)
/usr/lib/python2.7/site-packages/pymysql/cursors.py:166: Warning: (1831, u'Duplicate index `ix_image_properties_image_id_name`. This is deprecated and will be disallowed in a future release.')
  result = self._query(query)  ## 这里的警告忽略掉
[root@Marvin-OpenStack ~]# mysql -h 10.0.0.56 -uglance -pglance -e "use glance;show tables;"  ## 验证是否导入成功
+----------------------------------+
| Tables_in_glance                 |
+----------------------------------+
| artifact_blob_locations          |
| artifact_blobs                   |
| artifact_dependencies            |
| artifact_properties              |
| artifact_tags                    |
| artifacts                        |
| image_locations                  |
| image_members                    |
| image_properties                 |
| image_tags                       |
| images                           |
| metadef_namespace_resource_types |
| metadef_namespaces               |
| metadef_objects                  |
| metadef_properties               |
| metadef_resource_types           |
| metadef_tags                     |
| migrate_version                  |
| task_info                        |
| tasks                            |
+----------------------------------+
## 继续修改glance的配置文件,配置认证访问服务
[root@Marvin-OpenStack ~]# vim /etc/glance/glance-api.conf
[keystone_authtoken]  ## 在keystone_authtoken模块下添加一下内容,大约在3178行
auth_uri = http://10.0.0.56:5000
auth_url = http://10.0.0.56:35357
memcached_servers = 10.0.0.56:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance
-----------------------------------------------
[paste_deploy]
flavor = keystone   ##  3990行取消注释
[glance_store]
stores = file,http   ##  1864行取消注释
default_store = file  ## 1896行取消注释
filesystem_store_datadir = /var/lib/glance/images   ## 2196行取消注释
[root@Marvin-OpenStack ~]# vim /etc/glance/glance-registry.conf
[keystone_authtoken]
auth_uri = http://10.0.0.56:5000
auth_url = http://10.0.0.56:35357
memcached_servers = 10.0.0.56:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance
-----------------------------------------------
[paste_deploy]
flavor = keystone   ##  1910行取消注释
## 查看glance-api.cnf修改的内容
[root@Marvin-OpenStack ~]# grep '^[a-z]' /etc/glance/glance-api.conf 
connection = mysql+pymysql://glance:glance@10.0.0.56/glance
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images
auth_uri = http://10.0.0.56:5000
auth_url = http://10.0.0.56:35357
memcached_servers = 10.0.0.56:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance
flavor = keystone
## 查看glance-registry.conf修改的内容
connection = mysql+pymysql://glance:glance@10.0.0.56/glance
auth_uri = http://10.0.0.56:5000
auth_url = http://10.0.0.56:35357
memcached_servers = 10.0.0.56:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = glance
flavor = keystone
## 启动glance并设置开机自启
[root@Marvin-OpenStack ~]# systemctl enable openstack-glance-api.service \
>     openstack-glance-registry.service
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-glance-api.service to /usr/lib/systemd/system/openstack-glance-api.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-glance-registry.service to /usr/lib/systemd/system/openstack-glance-registry.service.
[root@Marvin-OpenStack ~]# systemctl start  openstack-glance-api.service     openstack-glance-registry.service
## 下载官方cirros镜像,测试
镜像下载:  http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
[root@Marvin-OpenStack ~]# ll
total 12992
-rw-r--r--  1 root root      260 Sep  2 21:16 admin-openstack
-rw-------. 1 root root      930 Sep  2 06:54 anaconda-ks.cfg
-rw-r--r--  1 root root 13287936 Sep  2 22:01 cirros-0.3.4-x86_64-disk.img   ## 上传的镜像
-rw-r--r--  1 root root      256 Sep  2 21:17 demo-openstack
## 创建一个cirros的镜像,属性是公开的
[root@Marvin-OpenStack ~]# openstack image create "cirros" \   
>      --file cirros-0.3.4-x86_64-disk.img \
>      --disk-format qcow2 --container-format bare \
>      --public
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6                     |
| container_format | bare                                                 |
| created_at       | 2017-09-02T14:02:46Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/a131bcba-8fa7-4a0e-8333-a36046c31a9c/file |
| id               | a131bcba-8fa7-4a0e-8333-a36046c31a9c                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | 8e8448db75034b1e8be0f7d6931be2d4                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 13287936                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2017-09-02T14:02:47Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
[root@Marvin-OpenStack ~]# openstack image list  ## 查看镜像
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| a131bcba-8fa7-4a0e-8333-a36046c31a9c | cirros | active |
+--------------------------------------+--------+--------+
[root@Marvin-OpenStack ~]# glance image-list
+--------------------------------------+--------+
| ID                                   | Name   |
+--------------------------------------+--------+
| a131bcba-8fa7-4a0e-8333-a36046c31a9c | cirros |
+--------------------------------------+--------+
```

glance服务安装完成

#### 7. nova安装,nova的安装是比较复杂的,是需要在管理节点和计算节点分别安装的

###### 7.1 管理节点安装配置nova

```
[root@Marvin-OpenStack ~]# yum install openstack-nova-api openstack-nova-conductor   openstack-nova-console openstack-nova-novncproxy   openstack-nova-scheduler -y
[root@Marvin-OpenStack ~]# cd /etc/nova
[root@Marvin-OpenStack nova]# ll
total 300
-rw-r----- 1 root nova   2717 May 31 00:07 api-paste.ini
-rw-r----- 1 root nova 289748 Aug  3 17:52 nova.conf
-rw-r----- 1 root nova      4 May 31 00:07 policy.json
-rw-r--r-- 1 root root     64 Aug  3 17:52 release
-rw-r----- 1 root nova    966 May 31 00:07 rootwrap.conf
[root@Marvin-OpenStack nova]# vim nova.conf
 [DEFAULT]
auth_strategy=keystone  ## 14行,开启验证方式
use_neutron=true   ## 2062 开启注释,并修改为true
enabled_apis=osapi_compute,metadata  ## 3052行,开启注释
firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver  ## 3265 开启注释
transport_url=rabbit://openstack:openstack@10.0.0.56   ## 3601 开启注释,并写入消息验证的地址
[api_database]
connection=mysql+pymysql://nova:nova@10.0.0.56/nova_api  ## 3661 开启注释,并修改数据库地址
[database]
connection=mysql+pymysql://nova:nova@10.0.0.56/nova  ## 4678 开启注释,并修改数据库地址
[keystone_authtoken]  ## 5429行,在这个下面添加nova的认证信息
auth_uri = http://10.0.0.56:5000
auth_url = http://10.0.0.56:35357
memcached_servers = 10.0.0.56:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova 
password = nova
[vnc]
vncserver_listen=0.0.0.0  ## 8384行,取消注释,并修改为0.0.0.0
vncserver_proxyclient_address=10.0.0.56   ## 8396 取消注释,修改为自己的主机地址
[glance]
api_servers=10.0.0.56:9292  ## 4813  取消注释,并修改地址
[oslo_concurrency]
lock_path=/var/lib/nova/tmp   ## 6705行,取消注释
## 查看nova.conf一共修改了多少内容
[root@Marvin-OpenStack nova]# grep '^[a-Z]' nova.conf
auth_strategy=keystone
use_neutron=true
enabled_apis=osapi_compute,metadata
firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
transport_url=rabbit://openstack:openstack@10.0.0.56
connection=mysql+pymysql://nova:nova@10.0.0.56/nova_api
connection=mysql+pymysql://nova:nova@10.0.0.56/nova
api_servers=10.0.0.56:9292
auth_uri = http://10.0.0.56:5000
auth_url = http://10.0.0.56:35357
memcached_servers = 10.0.0.56:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova 
password = nova
lock_path=/var/lib/nova/tmp
vncserver_listen=0.0.0.0
vncserver_proxyclient_address=10.0.0.56
## 导入数据库信息并验证
[root@Marvin-OpenStack nova]# su -s /bin/sh -c "nova-manage api_db sync" nova
[root@Marvin-OpenStack nova]# su -s /bin/sh -c "nova-manage db sync" nova
WARNING: cell0 mapping not found - not syncing cell0.
/usr/lib/python2.7/site-packages/pymysql/cursors.py:166: Warning: (1831, u'Duplicate index `block_device_mapping_instance_uuid_virtual_name_device_name_idx`. This is deprecated and will be disallowed in a future release.')
  result = self._query(query)
/usr/lib/python2.7/site-packages/pymysql/cursors.py:166: Warning: (1831, u'Duplicate index `uniq_instances0uuid`. This is deprecated and will be disallowed in a future release.')
  result = self._query(query   ### 警告可以忽略
[root@Marvin-OpenStack nova]# mysql -h 10.0.0.56 -unova -pnova -e "use nova;show tables;"   ## 验证,记住nova有两个库,一个nova,一个是nova_api
+--------------------------------------------+
| Tables_in_nova                             |
+--------------------------------------------+
| agent_builds                               |
| aggregate_hosts                            |
| aggregate_metadata                         |
| aggregates                                 |
| allocations                                |
| block_device_mapping                       |
| bw_usage_cache                             |
| cells                                      |
| certificates                               |
| compute_nodes                              |
| console_auth_tokens                        |
| console_pools                              |
| consoles                                   |
| dns_domains                                |
| fixed_ips                                  |
| floating_ips                               |
| instance_actions                           |
| instance_actions_events                    |
| instance_extra                             |
| instance_faults                            |
| instance_group_member                      |
| instance_group_policy                      |
| instance_groups                            |
| instance_id_mappings                       |
| instance_info_caches                       |
| instance_metadata                          |
| instance_system_metadata                   |
| instance_type_extra_specs                  |
| instance_type_projects                     |
| instance_types                             |
| instances                                  |
| inventories                                |
| key_pairs                                  |
| migrate_version                            |
| migrations                                 |
| networks                                   |
| pci_devices                                |
| project_user_quotas                        |
| provider_fw_rules                          |
| quota_classes                              |
| quota_usages                               |
| quotas                                     |
| reservations                               |
| resource_provider_aggregates               |
| resource_providers                         |
| s3_images                                  |
| security_group_default_rules               |
| security_group_instance_association        |
| security_group_rules                       |
| security_groups                            |
| services                                   |
| shadow_agent_builds                        |
| shadow_aggregate_hosts                     |
| shadow_aggregate_metadata                  |
| shadow_aggregates                          |
| shadow_block_device_mapping                |
| shadow_bw_usage_cache                      |
| shadow_cells                               |
| shadow_certificates                        |
| shadow_compute_nodes                       |
| shadow_console_pools                       |
| shadow_consoles                            |
| shadow_dns_domains                         |
| shadow_fixed_ips                           |
| shadow_floating_ips                        |
| shadow_instance_actions                    |
| shadow_instance_actions_events             |
| shadow_instance_extra                      |
| shadow_instance_faults                     |
| shadow_instance_group_member               |
| shadow_instance_group_policy               |
| shadow_instance_groups                     |
| shadow_instance_id_mappings                |
| shadow_instance_info_caches                |
| shadow_instance_metadata                   |
| shadow_instance_system_metadata            |
| shadow_instance_type_extra_specs           |
| shadow_instance_type_projects              |
| shadow_instance_types                      |
| shadow_instances                           |
| shadow_key_pairs                           |
| shadow_migrate_version                     |
| shadow_migrations                          |
| shadow_networks                            |
| shadow_pci_devices                         |
| shadow_project_user_quotas                 |
| shadow_provider_fw_rules                   |
| shadow_quota_classes                       |
| shadow_quota_usages                        |
| shadow_quotas                              |
| shadow_reservations                        |
| shadow_s3_images                           |
| shadow_security_group_default_rules        |
| shadow_security_group_instance_association |
| shadow_security_group_rules                |
| shadow_security_groups                     |
| shadow_services                            |
| shadow_snapshot_id_mappings                |
| shadow_snapshots                           |
| shadow_task_log                            |
| shadow_virtual_interfaces                  |
| shadow_volume_id_mappings                  |
| shadow_volume_usage_cache                  |
| snapshot_id_mappings                       |
| snapshots                                  |
| tags                                       |
| task_log                                   |
| virtual_interfaces                         |
| volume_id_mappings                         |
| volume_usage_cache                         |
+--------------------------------------------+
[root@Marvin-OpenStack nova]# mysql -h 10.0.0.56 -unova -pnova -e "use nova_api;show tables;"
+------------------------------+
| Tables_in_nova_api           |
+------------------------------+
| aggregate_hosts              |
| aggregate_metadata           |
| aggregates                   |
| allocations                  |
| build_requests               |
| cell_mappings                |
| flavor_extra_specs           |
| flavor_projects              |
| flavors                      |
| host_mappings                |
| instance_group_member        |
| instance_group_policy        |
| instance_groups              |
| instance_mappings            |
| inventories                  |
| key_pairs                    |
| migrate_version              |
| request_specs                |
| resource_provider_aggregates |
| resource_providers           |
+------------------------------+
## 验证成功后启动nova服务
[root@Marvin-OpenStack nova]# systemctl enable openstack-nova-api.service \
>      openstack-nova-consoleauth.service openstack-nova-scheduler.service \
>      openstack-nova-conductor.service openstack-nova-novncproxy.service
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-nova-api.service to /usr/lib/systemd/system/openstack-nova-api.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-nova-consoleauth.service to /usr/lib/systemd/system/openstack-nova-consoleauth.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-nova-scheduler.service to /usr/lib/systemd/system/openstack-nova-scheduler.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-nova-conductor.service to /usr/lib/systemd/system/openstack-nova-conductor.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-nova-novncproxy.service to /usr/lib/systemd/system/openstack-nova-novncproxy.service.
[root@Marvin-OpenStack nova]# systemctl start openstack-nova-api.service     openstack-nova-consoleauth.service openstack-nova-scheduler.service     openstack-nova-conductor.service openstack-nova-novncproxy.service
## 创建compute服务
[root@Marvin-OpenStack ~]# openstack service create --name nova \
>      --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | d1559df37465470eb7fafbd93bffb183 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
## 添加endpoint记录
[root@Marvin-OpenStack ~]# openstack endpoint create --region RegionOne \
>     compute public http://10.0.0.56:8774/v2.1/%\(tenant_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | a0f3ffe43f7a44e58261355c9ebad61a         |
| interface    | public                                   |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | d1559df37465470eb7fafbd93bffb183         |
| service_name | nova                                     |
| service_type | compute                                  |
| url          | http://10.0.0.56:8774/v2.1/%(tenant_id)s |
+--------------+------------------------------------------+
[root@Marvin-OpenStack ~]# openstack endpoint create --region RegionOne     compute internal http://10.0.0.56:8774/v2.1/%\(tenant_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | 7532f46cca5540bfae8d9340a62644af         |
| interface    | internal                                 |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | d1559df37465470eb7fafbd93bffb183         |
| service_name | nova                                     |
| service_type | compute                                  |
| url          | http://10.0.0.56:8774/v2.1/%(tenant_id)s |
+--------------+------------------------------------------+
[root@Marvin-OpenStack ~]# openstack endpoint create --region RegionOne     compute admin http://10.0.0.56:8774/v2.1/%\(tenant_id\)s
+--------------+------------------------------------------+
| Field        | Value                                    |
+--------------+------------------------------------------+
| enabled      | True                                     |
| id           | d0470f3f77f040b086b34b6641b5e280         |
| interface    | admin                                    |
| region       | RegionOne                                |
| region_id    | RegionOne                                |
| service_id   | d1559df37465470eb7fafbd93bffb183         |
| service_name | nova                                     |
| service_type | compute                                  |
| url          | http://10.0.0.56:8774/v2.1/%(tenant_id)s |
+--------------+------------------------------------------+
## 验证管理节点的nova是否配置成功
[root@Marvin-OpenStack ~]# openstack host list
+------------------+-------------+----------+
| Host Name        | Service     | Zone     |
+------------------+-------------+----------+
| Marvin-OpenStack | scheduler   | internal |
| Marvin-OpenStack | consoleauth | internal |
| Marvin-OpenStack | conductor   | internal |
+------------------+-------------+----------+
[root@Marvin-OpenStack ~]# nova service-list
+----+------------------+------------------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host             | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+------------------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-scheduler   | Marvin-OpenStack | internal | enabled | up    | 2017-09-03T08:52:25.000000 | -               |
| 2  | nova-consoleauth | Marvin-OpenStack | internal | enabled | up    | 2017-09-03T08:52:25.000000 | -               |
| 3  | nova-conductor   | Marvin-OpenStack | internal | enabled | up    | 2017-09-03T08:52:27.000000 | -               |
+----+------------------+------------------+----------+---------+-------+----------------------------+-----------------+
[root@Marvin-OpenStack ~]# openstack endpoint list
+-----------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------+
| ID                          | Region    | Service Name | Service Type | Enabled | Interface | URL                         |
+-----------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------+
| 0e28a8c975904c28b5a2a9bc180 | RegionOne | glance       | image        | True    | public    | http://10.0.0.56:9292       |
| efe02                       |           |              |              |         |           |                             |
| 2cf91a5b1e1e4d19830627a1d10 | RegionOne | glance       | image        | True    | internal  | http://10.0.0.56:9292       |
| 88b40                       |           |              |              |         |           |                             |
| 41838de3d5204c9dbf173321b8b | RegionOne | keystone     | identity     | True    | internal  | http://10.0.0.56:35357/v3/  |
| c05d7                       |           |              |              |         |           |                             |
| 7532f46cca5540bfae8d9340a62 | RegionOne | nova         | compute      | True    | internal  | http://10.0.0.56:8774/v2.1/ |
| 644af                       |           |              |              |         |           | %(tenant_id)s               |
| 806d8539a4f44c66b14488ee197 | RegionOne | glance       | image        | True    | admin     | http://10.0.0.56:9292       |
| 9190d                       |           |              |              |         |           |                             |
| a0f3ffe43f7a44e58261355c9eb | RegionOne | nova         | compute      | True    | public    | http://10.0.0.56:8774/v2.1/ |
| ad61a                       |           |              |              |         |           | %(tenant_id)s               |
| d0470f3f77f040b086b34b6641b | RegionOne | nova         | compute      | True    | admin     | http://10.0.0.56:8774/v2.1/ |
| 5e280                       |           |              |              |         |           | %(tenant_id)s               |
| d8b9d6bf135b482e852c6483ce3 | RegionOne | keystone     | identity     | True    | admin     | http://10.0.0.56:35357/v3/  |
| 0e4f9                       |           |              |              |         |           |                             |
| e83d3ee63f2a473e809931900ec | RegionOne | keystone     | identity     | True    | public    | http://10.0.0.56:5000/v3/   |
| 71174                       |           |              |              |         |           |                             |
+-----------------------------+-----------+--------------+--------------+---------+-----------+-----------------------------+
```

###### 7.2 计算节点安装配置nova

```
[root@Marvin-Compute ~]#  yum install openstack-nova-compute -y
[root@Marvin-Compute ~]# cd /etc/nova/
[root@Marvin-Compute nova]# ll
total 300
-rw-r----- 1 root nova   2717 May 31 00:07 api-paste.ini
-rw-r----- 1 root nova 289748 Aug  3 17:52 nova.conf
-rw-r----- 1 root nova      4 May 31 00:07 policy.json
-rw-r--r-- 1 root root     64 Aug  3 17:52 release
-rw-r----- 1 root nova    966 May 31 00:07 rootwrap.conf
## 修改nova.conf的配置,我们直接从管理节点拷贝一个过来修改
[root@Marvin-Compute nova]# mv nova.conf nova.conf.marvin20170902
[root@Marvin-Compute nova]# scp 10.0.0.56:/etc/nova/nova.conf .
The authenticity of host '10.0.0.56 (10.0.0.56)' can't be established.
ECDSA key fingerprint is 6c:10:78:9b:94:7e:90:93:fb:65:37:3c:98:9b:83:d1.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.0.0.56' (ECDSA) to the list of known hosts.
root@10.0.0.56's password: 
nova.conf                                                                                  100%  283KB 283.3KB/s   00:00    
[root@Marvin-Compute nova]# ll
total 584
-rw-r----- 1 root nova   2717 May 31 00:07 api-paste.ini
-rw-r----- 1 root root 290057 Sep  3 17:26 nova.conf
-rw-r----- 1 root nova 289748 Aug  3 17:52 nova.conf.marvin20170902
-rw-r----- 1 root nova      4 May 31 00:07 policy.json
-rw-r--r-- 1 root root     64 Aug  3 17:52 release
-rw-r----- 1 root nova    966 May 31 00:07 rootwrap.conf
[root@Marvin-Compute nova]# chgrp nova nova.conf
[root@Marvin-Compute nova]# ll
total 584
-rw-r----- 1 root nova   2717 May 31 00:07 api-paste.ini
-rw-r----- 1 root nova 290057 Sep  3 17:26 nova.conf
-rw-r----- 1 root nova 289748 Aug  3 17:52 nova.conf.marvin20170902
-rw-r----- 1 root nova      4 May 31 00:07 policy.json
-rw-r--r-- 1 root root     64 Aug  3 17:52 release
-rw-r----- 1 root nova    966 May 31 00:07 rootwrap.conf
## 修改nova.conf
[root@Marvin-Compute nova]# vim nova.conf
[api_database]
connection=mysql+pymysql://nova:nova@10.0.0.56/nova_api   ## 3661行,直接删除
[database]
connection=mysql+pymysql://nova:nova@10.0.0.56/nova  ##  4677行,直接删除
[vnc]
vncserver_proxyclient_address=10.0.0.57  ## 8394行地址改为本机的地址
novncproxy_base_url=http://10.0.0.56:6080/vnc_auto.html  ## 8413行取消注释,并修改
[libvirt]
virt_type=kvm   ##  5672行取消注释,如果环境不支持虚拟化,需要改为qemu
enabled=true  ## 8359取消注释
keymap=en-us  ## 8375取消注释
## 查看nova.conf总共修改的内容
[root@Marvin-Compute nova]# grep '^[a-z]' nova.conf
auth_strategy=keystone
use_neutron=true
enabled_apis=osapi_compute,metadata
firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
transport_url=rabbit://openstack:openstack@10.0.0.56
api_servers=10.0.0.56:9292
auth_uri = http://10.0.0.56:5000
auth_url = http://10.0.0.56:35357
memcached_servers = 10.0.0.56:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova 
password = nova
virt_type=kvm
lock_path=/var/lib/nova/tmp
enabled=true
keymap=en-us
vncserver_listen=0.0.0.0
vncserver_proxyclient_address=10.0.0.57
novncproxy_base_url=http://10.0.0.56:6080/vnc_auto.html
## 查看自己的计算节点是否支持虚拟化
[root@Marvin-Compute nova]# egrep \ '(vmx|svm)' /proc/cpuinfo
flags       : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts nopl xtopology tsc_reliable nonstop_tsc aperfmperf eagerfpu pni pclmulqdq vmx ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch ida arat epb xsaveopt pln pts dtherm tpr_shadow vnmi ept vpid fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid rdseed adx smap
flags       : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts nopl xtopology tsc_reliable nonstop_tsc aperfmperf eagerfpu pni pclmulqdq vmx ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch ida arat epb xsaveopt pln pts dtherm tpr_shadow vnmi ept vpid fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid rdseed adx smap
flags       : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts nopl xtopology tsc_reliable nonstop_tsc aperfmperf eagerfpu pni pclmulqdq vmx ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch ida arat epb xsaveopt pln pts dtherm tpr_shadow vnmi ept vpid fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid rdseed adx smap
flags       : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts nopl xtopology tsc_reliable nonstop_tsc aperfmperf eagerfpu pni pclmulqdq vmx ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch ida arat epb xsaveopt pln pts dtherm tpr_shadow vnmi ept vpid fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid rdseed adx smap
[root@Marvin-Compute nova]# egrep -c '(vmx|svm)' /proc/cpuinfo
4
## 执行以上命令,如果没有反馈或者反馈数字为0,那么virt_type的值就要改为qemu
## 启动计算节点的nova程序
[root@Marvin-Compute nova]# systemctl enable libvirtd.service openstack-nova-compute.service
Created symlink from /etc/systemd/system/multi-user.target.wants/openstack-nova-compute.service to /usr/lib/systemd/system/openstack-nova-compute.service.
[root@Marvin-Compute nova]# systemctl start libvirtd.service openstack-nova-compute.service
## 在管理节点上验证
[root@Marvin-OpenStack ~]# nova service-list
+----+------------------+------------------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host             | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+------------------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-scheduler   | Marvin-OpenStack | internal | enabled | up    | 2017-09-03T09:56:36.000000 | -               |
| 2  | nova-consoleauth | Marvin-OpenStack | internal | enabled | up    | 2017-09-03T09:56:36.000000 | -               |
| 3  | nova-conductor   | Marvin-OpenStack | internal | enabled | up    | 2017-09-03T09:56:38.000000 | -               |
| 6  | nova-compute     | Marvin-Compute   | nova     | enabled | up    | 2017-09-03T09:56:35.000000 | -               |
+----+------------------+------------------+----------+---------+-------+----------------------------+-----------------+
[root@Marvin-OpenStack ~]# openstack compute service list
+----+------------------+------------------+----------+---------+-------+----------------------------+
| ID | Binary           | Host             | Zone     | Status  | State | Updated At                 |
+----+------------------+------------------+----------+---------+-------+----------------------------+
|  1 | nova-scheduler   | Marvin-OpenStack | internal | enabled | up    | 2017-09-03T09:56:56.000000 |
|  2 | nova-consoleauth | Marvin-OpenStack | internal | enabled | up    | 2017-09-03T09:56:56.000000 |
|  3 | nova-conductor   | Marvin-OpenStack | internal | enabled | up    | 2017-09-03T09:56:58.000000 |
|  6 | nova-compute     | Marvin-Compute   | nova     | enabled | up    | 2017-09-03T09:56:55.000000 |
+----+------------------+------------------+----------+---------+-------+----------------------------+
可以看到已经发现了我们的计算节点
```

到此,管理节点和计算节点nova就配置完成了

#### 8. 安装配置neutron服务,neutron也是需要在两个节点安装的服务

###### 8.1 管理节点安装配置neutron服务

```
[root@Marvin-OpenStack ~]# yum install openstack-neutron openstack-neutron-ml2    openstack-neutron-linuxbridge ebtables -y
[root@Marvin-OpenStack ~]# cd /etc/neutron/
[root@Marvin-OpenStack neutron]# ll
total 120
drwxr-xr-x 11 root root     4096 Sep  3 18:14 conf.d
-rw-r-----  1 root neutron  8592 Aug 16 02:08 dhcp_agent.ini
-rw-r-----  1 root neutron 11119 Aug 16 02:09 l3_agent.ini
-rw-r-----  1 root neutron 10140 Aug 16 02:09 metadata_agent.ini
-rw-r-----  1 root neutron 63378 Aug 16 02:09 neutron.conf
drwxr-xr-x  3 root root       16 Sep  3 18:14 plugins
-rw-r-----  1 root neutron 10148 Jun  1 23:39 policy.json
-rw-r--r--  1 root root     1195 Jun  1 23:39 rootwrap.conf
[root@Marvin-OpenStack neutron]# vim neutron.conf
[database]
connection = mysql+pymysql://neutron:neutron@10.0.0.56/neutron  ##  722行修改数据库信息
[DEFAULT]
auth_strategy = keystone  ## 27行取消注释
core_plugin = ml2     ##  30行取消注释,并修改为ml2
service_plugins =     ##  33行取消注释
transport_url = rabbit://openstack:openstack@10.0.0.56  ## 530行修改消息队列
notify_nova_on_port_status_changes = true  ## 118行取消注释
notify_nova_on_port_data_changes = true  ## 122行取消注释
[keystone_authtoken]  ## 802行,模块下加入下面的信息
auth_uri = http://10.0.0.56:5000
auth_url = http://10.0.0.56:35357
memcached_servers = 10.0.0.56:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron
[nova]  ## 1001行nova模块下加入下面的信息
auth_url = http://10.0.0.56:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova 
password = nova
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp ## 1123行取消注释并修改
查看neutron.conf修改的内容
[root@Marvin-OpenStack neutron]# grep '^[a-z]' neutron.conf
auth_strategy = keystone
core_plugin = ml2
service_plugins =
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
transport_url = rabbit://openstack:openstack@10.0.0.56
connection = mysql+pymysql://neutron:neutron@10.0.0.56/neutron
auth_uri = http://10.0.0.56:5000
auth_url = http://10.0.0.56:35357
memcached_servers = 10.0.0.56:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron
auth_url = http://10.0.0.56:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = nova
lock_path = /var/lib/neutron/tmp
## 修改/etc/neutron/plugins/ml2/ml2_conf.ini文件
[root@Marvin-OpenStack neutron]# cd plugins/ml2/
[root@Marvin-OpenStack ml2]# vim ml2_conf.ini
109 type_drivers = flat,vlan,gre,vxlan,geneve
114 tenant_network_types = flat,vlan,gre,vxlan,geneve
118 mechanism_drivers = linuxbridge
123 extension_drivers = port_security
159 flat_networks = public
236 enable_ipset = true
## ml2_conf.ini配置文件修改简单,按照上面的行数对照进行修改就好了
[root@Marvin-OpenStack ml2]# grep '^[a-z]' ml2_conf.ini 
type_drivers = flat,vlan,gre,vxlan,geneve
tenant_network_types = flat,vlan,gre,vxlan,geneve
mechanism_drivers = linuxbridge
extension_drivers = port_security
flat_networks = public
enable_ipset = true
##  配置Linuxbridge代理
[root@Marvin-OpenStack ml2]# vim linuxbridge_agent.ini
143 physical_interface_mappings = public:eth0   ## 这里和前面flat_networks的名称对应,eth0是网卡的名称,有的网卡并不一定都是eth0,记得修改成自己的网卡名称
156 firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
161 enable_security_group = true
176 enable_vxlan = false
[root@Marvin-OpenStack ml2]# grep '^[a-z]' linuxbridge_agent.ini    ## 这个文件修改也简单,按照行数对照修改
physical_interface_mappings = public:eth0
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
enable_security_group = true
enable_vxlan = false
## 配置DHCP代理
[root@Marvin-OpenStack ml2]# cd /etc/neutron/
[root@Marvin-OpenStack neutron]# vim dhcp_agent.ini   ## 还是一样的,根据行号修改
16 interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
32 dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
41 enable_isolated_metadata = true
[root@Marvin-OpenStack neutron]# grep '^[a-z]' dhcp_agent.ini 
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
## 配置元数据代理
[root@Marvin-OpenStack neutron]# vim metadata_agent.ini
22 nova_metadata_ip = 10.0.0.56
34 metadata_proxy_shared_secret = marvin   ##共享秘钥
## 配制nova服务来使用网络
[root@Marvin-OpenStack neutron]# cd /etc/nova
[root@Marvin-OpenStack nova]# vim nova.conf
[neutron]  ## 6469行neutron模块下添加如下信息
url = http://10.0.0.56:9696
auth_url = http://10.0.0.56:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
service_metadata_proxy = True
metadata_proxy_shared_secret = marvin
## 设置ml2的软链接
[root@Marvin-OpenStack ~]# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
## 同步数据库
[root@Marvin-OpenStack ~]# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
>      --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
  Running upgrade for neutron ...
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> kilo, kilo_initial
INFO  [alembic.runtime.migration] Running upgrade kilo -> 354db87e3225, nsxv_vdr_metadata.py
INFO  [alembic.runtime.migration] Running upgrade 354db87e3225 -> 599c6a226151, neutrodb_ipam
INFO  [alembic.runtime.migration] Running upgrade 599c6a226151 -> 52c5312f6baf, Initial operations in support of address scopes
INFO  [alembic.runtime.migration] Running upgrade 52c5312f6baf -> 313373c0ffee, Flavor framework
INFO  [alembic.runtime.migration] Running upgrade 313373c0ffee -> 8675309a5c4f, network_rbac
INFO  [alembic.runtime.migration] Running upgrade 8675309a5c4f -> 45f955889773, quota_usage
INFO  [alembic.runtime.migration] Running upgrade 45f955889773 -> 26c371498592, subnetpool hash
INFO  [alembic.runtime.migration] Running upgrade 26c371498592 -> 1c844d1677f7, add order to dnsnameservers
INFO  [alembic.runtime.migration] Running upgrade 1c844d1677f7 -> 1b4c6e320f79, address scope support in subnetpool
INFO  [alembic.runtime.migration] Running upgrade 1b4c6e320f79 -> 48153cb5f051, qos db changes
INFO  [alembic.runtime.migration] Running upgrade 48153cb5f051 -> 9859ac9c136, quota_reservations
INFO  [alembic.runtime.migration] Running upgrade 9859ac9c136 -> 34af2b5c5a59, Add dns_name to Port
INFO  [alembic.runtime.migration] Running upgrade 34af2b5c5a59 -> 59cb5b6cf4d, Add availability zone
INFO  [alembic.runtime.migration] Running upgrade 59cb5b6cf4d -> 13cfb89f881a, add is_default to subnetpool
INFO  [alembic.runtime.migration] Running upgrade 13cfb89f881a -> 32e5974ada25, Add standard attribute table
INFO  [alembic.runtime.migration] Running upgrade 32e5974ada25 -> ec7fcfbf72ee, Add network availability zone
INFO  [alembic.runtime.migration] Running upgrade ec7fcfbf72ee -> dce3ec7a25c9, Add router availability zone
INFO  [alembic.runtime.migration] Running upgrade dce3ec7a25c9 -> c3a73f615e4, Add ip_version to AddressScope
INFO  [alembic.runtime.migration] Running upgrade c3a73f615e4 -> 659bf3d90664, Add tables and attributes to support external DNS integration
INFO  [alembic.runtime.migration] Running upgrade 659bf3d90664 -> 1df244e556f5, add_unique_ha_router_agent_port_bindings
INFO  [alembic.runtime.migration] Running upgrade 1df244e556f5 -> 19f26505c74f, Auto Allocated Topology - aka Get-Me-A-Network
INFO  [alembic.runtime.migration] Running upgrade 19f26505c74f -> 15be73214821, add dynamic routing model data
INFO  [alembic.runtime.migration] Running upgrade 15be73214821 -> b4caf27aae4, add_bgp_dragent_model_data
INFO  [alembic.runtime.migration] Running upgrade b4caf27aae4 -> 15e43b934f81, rbac_qos_policy
INFO  [alembic.runtime.migration] Running upgrade 15e43b934f81 -> 31ed664953e6, Add resource_versions row to agent table
INFO  [alembic.runtime.migration] Running upgrade 31ed664953e6 -> 2f9e956e7532, tag support
INFO  [alembic.runtime.migration] Running upgrade 2f9e956e7532 -> 3894bccad37f, add_timestamp_to_base_resources
INFO  [alembic.runtime.migration] Running upgrade 3894bccad37f -> 0e66c5227a8a, Add desc to standard attr table
INFO  [alembic.runtime.migration] Running upgrade 0e66c5227a8a -> 45f8dd33480b, qos dscp db addition
INFO  [alembic.runtime.migration] Running upgrade 45f8dd33480b -> 5abc0278ca73, Add support for VLAN trunking
INFO  [alembic.runtime.migration] Running upgrade 5abc0278ca73 -> d3435b514502, Add device_id index to Port
INFO  [alembic.runtime.migration] Running upgrade d3435b514502 -> 30107ab6a3ee, provisioning_blocks.py
INFO  [alembic.runtime.migration] Running upgrade 30107ab6a3ee -> c415aab1c048, add revisions table
INFO  [alembic.runtime.migration] Running upgrade c415aab1c048 -> a963b38d82f4, add dns name to portdnses
INFO  [alembic.runtime.migration] Running upgrade a963b38d82f4 -> 3d0e74aa7d37, Add flavor_id to Router
INFO  [alembic.runtime.migration] Running upgrade 3d0e74aa7d37 -> 030a959ceafa, uniq_routerports0port_id
INFO  [alembic.runtime.migration] Running upgrade 030a959ceafa -> a5648cfeeadf, Add support for Subnet Service Types
INFO  [alembic.runtime.migration] Running upgrade a5648cfeeadf -> 0f5bef0f87d4, add_qos_minimum_bandwidth_rules
INFO  [alembic.runtime.migration] Running upgrade 0f5bef0f87d4 -> 67daae611b6e, add standardattr to qos policies
INFO  [alembic.runtime.migration] Running upgrade kilo -> 30018084ec99, Initial no-op Liberty contract rule.
INFO  [alembic.runtime.migration] Running upgrade 30018084ec99 -> 4ffceebfada, network_rbac
INFO  [alembic.runtime.migration] Running upgrade 4ffceebfada -> 5498d17be016, Drop legacy OVS and LB plugin tables
INFO  [alembic.runtime.migration] Running upgrade 5498d17be016 -> 2a16083502f3, Metaplugin removal
INFO  [alembic.runtime.migration] Running upgrade 2a16083502f3 -> 2e5352a0ad4d, Add missing foreign keys
INFO  [alembic.runtime.migration] Running upgrade 2e5352a0ad4d -> 11926bcfe72d, add geneve ml2 type driver
INFO  [alembic.runtime.migration] Running upgrade 11926bcfe72d -> 4af11ca47297, Drop cisco monolithic tables
INFO  [alembic.runtime.migration] Running upgrade 4af11ca47297 -> 1b294093239c, Drop embrane plugin table
INFO  [alembic.runtime.migration] Running upgrade 1b294093239c -> 8a6d8bdae39, standardattributes migration
INFO  [alembic.runtime.migration] Running upgrade 8a6d8bdae39 -> 2b4c2465d44b, DVR sheduling refactoring
INFO  [alembic.runtime.migration] Running upgrade 2b4c2465d44b -> e3278ee65050, Drop NEC plugin tables
INFO  [alembic.runtime.migration] Running upgrade e3278ee65050 -> c6c112992c9, rbac_qos_policy
INFO  [alembic.runtime.migration] Running upgrade c6c112992c9 -> 5ffceebfada, network_rbac_external
INFO  [alembic.runtime.migration] Running upgrade 5ffceebfada -> 4ffceebfcdc, standard_desc
INFO  [alembic.runtime.migration] Running upgrade 4ffceebfcdc -> 7bbb25278f53, device_owner_ha_replicate_int
INFO  [alembic.runtime.migration] Running upgrade 7bbb25278f53 -> 89ab9a816d70, Rename ml2_network_segments table
INFO  [alembic.runtime.migration] Running upgrade 89ab9a816d70 -> c879c5e1ee90, Add segment_id to subnet
INFO  [alembic.runtime.migration] Running upgrade c879c5e1ee90 -> 8fd3918ef6f4, Add segment_host_mapping table.
INFO  [alembic.runtime.migration] Running upgrade 8fd3918ef6f4 -> 4bcd4df1f426, Rename ml2_dvr_port_bindings
INFO  [alembic.runtime.migration] Running upgrade 4bcd4df1f426 -> b67e765a3524, Remove mtu column from networks.
INFO  [alembic.runtime.migration] Running upgrade b67e765a3524 -> a84ccf28f06a, migrate dns name from port
INFO  [alembic.runtime.migration] Running upgrade a84ccf28f06a -> 7d9d8eeec6ad, rename tenant to project
INFO  [alembic.runtime.migration] Running upgrade 7d9d8eeec6ad -> a8b517cff8ab, Add routerport bindings for L3 HA
INFO  [alembic.runtime.migration] Running upgrade a8b517cff8ab -> 3b935b28e7a0, migrate to pluggable ipam
INFO  [alembic.runtime.migration] Running upgrade 3b935b28e7a0 -> b12a3ef66e62, add standardattr to qos policies
INFO  [alembic.runtime.migration] Running upgrade b12a3ef66e62 -> 97c25b0d2353, Add Name and Description to the networksegments table
INFO  [alembic.runtime.migration] Running upgrade 97c25b0d2353 -> 2e0d7a8a1586, Add binding index to RouterL3AgentBinding
INFO  [alembic.runtime.migration] Running upgrade 2e0d7a8a1586 -> 5c85685d616d, Remove availability ranges.
INFO  [alembic.runtime.migration] Running upgrade 67daae611b6e -> 6b461a21bcfc, uniq_floatingips0floating_network_id0fixed_port_id0fixed_ip_addr
INFO  [alembic.runtime.migration] Running upgrade 6b461a21bcfc -> 5cd92597d11d, Add ip_allocation to port
  OK
## 重启nova-api服务,启动neutron服务,并设置开机自启
[root@Marvin-OpenStack ~]# systemctl restart openstack-nova-api.service
[root@Marvin-OpenStack ~]# systemctl enable neutron-server.service \
>     neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
>     neutron-metadata-agent.service
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-server.service to /usr/lib/systemd/system/neutron-server.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-linuxbridge-agent.service to /usr/lib/systemd/system/neutron-linuxbridge-agent.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-dhcp-agent.service to /usr/lib/systemd/system/neutron-dhcp-agent.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-metadata-agent.service to /usr/lib/systemd/system/neutron-metadata-agent.service.
[root@Marvin-OpenStack ~]# systemctl start neutron-server.service     neutron-linuxbridge-agent.service neutron-dhcp-agent.service     neutron-metadata-agent.service
## 创建Neutron服务实体
[root@Marvin-OpenStack ~]# openstack service create --name neutron \
>     --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | 4ae2380a5bb9497e9f84899de97778d6 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
## 注册endpoint
[root@Marvin-OpenStack ~]# openstack endpoint create --region RegionOne \
>     network public http://10.0.0.56:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 12c323c8747947999471026d2f1dec0a |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 4ae2380a5bb9497e9f84899de97778d6 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://10.0.0.56:9696            |
+--------------+----------------------------------+
[root@Marvin-OpenStack ~]# openstack endpoint create --region RegionOne     network internal http://10.0.0.56:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 9c3cdef26ff543dc8ebb4a206a92bb47 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 4ae2380a5bb9497e9f84899de97778d6 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://10.0.0.56:9696            |
+--------------+----------------------------------+
[root@Marvin-OpenStack ~]# openstack endpoint create --region RegionOne     network admin http://10.0.0.56:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 344c8833f7194d669d1837120f7a94ec |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 4ae2380a5bb9497e9f84899de97778d6 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://10.0.0.56:9696            |
+--------------+----------------------------------+
## 验证管理端的neutrun配置
[root@Marvin-OpenStack ~]# neutron agent-list
+------------------+------------------+------------------+-------------------+-------+----------------+----------------------+
| id               | agent_type       | host             | availability_zone | alive | admin_state_up | binary               |
+------------------+------------------+------------------+-------------------+-------+----------------+----------------------+
| 01efe4d9-53f5-48 | Metadata agent   | Marvin-OpenStack |                   | :-)   | True           | neutron-metadata-    |
| 6f-8869-40546a68 |                  |                  |                   |       |                | agent                |
| 8fd1             |                  |                  |                   |       |                |                      |
| 41127c68-fb5d-46 | DHCP agent       | Marvin-OpenStack | nova              | :-)   | True           | neutron-dhcp-agent   |
| 41-acdd-         |                  |                  |                   |       |                |                      |
| 1fba6b78735d     |                  |                  |                   |       |                |                      |
| 88e9488b-18cf-4c | Linux bridge     | Marvin-OpenStack |                   | :-)   | True           | neutron-linuxbridge- |
| a4-b0a8-8a916231 | agent            |                  |                   |       |                | agent                |
| 1315             |                  |                  |                   |       |                |                      |
+------------------+------------------+------------------+-------------------+-------+----------------+----------------------+
```

管理端的neutron配置完成

###### 8.2 计算节点安装配置neutrun服务

```
[root@Marvin-Compute ~]# yum install openstack-neutron-linuxbridge ebtables ipset -y
[root@Marvin-Compute ~]# vim /etc/neutron/neutron.conf
[DEFAULT]
27 auth_strategy = keystone
530 transport_url = rabbit://openstack:openstack@10.0.0.56
[keystone_authtoken]   ##  802行,添加如下内容
auth_uri = http://10.0.0.56:5000
auth_url = http://10.0.0.56:35357
memcached_servers = 10.0.0.56:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron
[oslo_concurrency]
1115 lock_path = /var/lib/neutron/tmp
[root@Marvin-Compute ~]# grep '^[a-z]' /etc/neutron/neutron.conf 
auth_strategy = keystone
transport_url = rabbit://openstack:openstack@10.0.0.56
auth_uri = http://10.0.0.56:5000
auth_url = http://10.0.0.56:35357
memcached_servers = 10.0.0.56:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron
lock_path = /var/lib/neutron/tmp
##  配置linuxbridge代理,因为管理节点和计算节点修改的内容一样,我们直接在管理节点拷贝过来,修改权限即可
[root@Marvin-Compute ~]# cd /etc/neutron/plugins/ml2/
[root@Marvin-Compute ml2]# mv linuxbridge_agent.ini  linuxbridge_agent.ini.marvin20170902
[root@Marvin-Compute ml2]# scp 10.0.0.56:/etc/neutron/plugins/ml2/linuxbridge_agent.ini .
root@10.0.0.56's password: 
linuxbridge_agent.ini                                                                      100% 8376     8.2KB/s   00:00    
[root@Marvin-Compute ml2]# ll
total 24
-rw-r----- 1 root root    8376 Sep  3 19:11 linuxbridge_agent.ini
-rw-r----- 1 root neutron 8313 Aug 16 02:09 linuxbridge_agent.ini.marvin20170902
[root@Marvin-Compute ml2]# chgrp neutron linuxbridge_agent.ini
[root@Marvin-Compute ml2]# ll
total 24
-rw-r----- 1 root neutron 8376 Sep  3 19:11 linuxbridge_agent.ini
-rw-r----- 1 root neutron 8313 Aug 16 02:09 linuxbridge_agent.ini.marvin20170902
## 在计算节点的nova中配置网络信息
[root@Marvin-Compute ml2]# cd /etc/nova
[root@Marvin-Compute nova]# vim nova.conf
[neutron]  ## 6467行,添加如下内容
url = http://10.0.0.56:9696
auth_url = http://10.0.0.56:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
## 重启nova服务,启动linuxbridge代理,并设置开机自启
[root@Marvin-Compute nova]# systemctl restart openstack-nova-compute.service
[root@Marvin-Compute nova]# systemctl enable neutron-linuxbridge-agent.service
Created symlink from /etc/systemd/system/multi-user.target.wants/neutron-linuxbridge-agent.service to /usr/lib/systemd/system/neutron-linuxbridge-agent.service.
[root@Marvin-Compute nova]# systemctl start neutron-linuxbridge-agent.service
## 管理节点上进行验证
[root@Marvin-OpenStack ~]# neutron agent-list
+------------------+------------------+------------------+-------------------+-------+----------------+----------------------+
| id               | agent_type       | host             | availability_zone | alive | admin_state_up | binary               |
+------------------+------------------+------------------+-------------------+-------+----------------+----------------------+
| 01efe4d9-53f5-48 | Metadata agent   | Marvin-OpenStack |                   | :-)   | True           | neutron-metadata-    |
| 6f-8869-40546a68 |                  |                  |                   |       |                | agent                |
| 8fd1             |                  |                  |                   |       |                |                      |
| 41127c68-fb5d-46 | DHCP agent       | Marvin-OpenStack | nova              | :-)   | True           | neutron-dhcp-agent   |
| 41-acdd-         |                  |                  |                   |       |                |                      |
| 1fba6b78735d     |                  |                  |                   |       |                |                      |
| 88e9488b-18cf-4c | Linux bridge     | Marvin-OpenStack |                   | :-)   | True           | neutron-linuxbridge- |
| a4-b0a8-8a916231 | agent            |                  |                   |       |                | agent                |
| 1315             |                  |                  |                   |       |                |                      |
| fdc0270e-3b6f-47 | Linux bridge     | Marvin-Compute   |                   | :-)   | True           | neutron-linuxbridge- |
| 53-8e05-428a4f3a | agent            |                  |                   |       |                | agent                |
| 0e2a             |                  |                  |                   |       |                |                      |
+------------------+------------------+------------------+-------------------+-------+----------------+----------------------+
[root@Marvin-OpenStack ~]# nova service-list
+----+------------------+------------------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host             | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+------------------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-scheduler   | Marvin-OpenStack | internal | enabled | up    | 2017-09-03T11:17:37.000000 | -               |
| 2  | nova-consoleauth | Marvin-OpenStack | internal | enabled | up    | 2017-09-03T11:17:38.000000 | -               |
| 3  | nova-conductor   | Marvin-OpenStack | internal | enabled | up    | 2017-09-03T11:17:39.000000 | -               |
| 6  | nova-compute     | Marvin-Compute   | nova     | enabled | up    | 2017-09-03T11:17:34.000000 | -               |
+----+------------------+------------------+----------+---------+-------+----------------------------+-----------------+
```

到此,neutrun的配置全部完成

#### 9. 创建云主机

###### 9.1 创建提供者网络

```
[root@Marvin-OpenStack ~]# neutron net-create --shared --provider:physical_network public \
>    --provider:network_type flat public
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2017-09-03T11:19:56Z                 |
| description               |                                      |
| id                        | 11a77b36-fc41-4fcd-babc-1bd3051ee064 |
| ipv4_address_scope        |                                      |
| ipv6_address_scope        |                                      |
| mtu                       | 1500                                 |
| name                      | public                               |
| port_security_enabled     | True                                 |
| project_id                | 8e8448db75034b1e8be0f7d6931be2d4     |
| provider:network_type     | flat                                 |
| provider:physical_network | public                               |
| provider:segmentation_id  |                                      |
| revision_number           | 3                                    |
| router:external           | False                                |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| tenant_id                 | 8e8448db75034b1e8be0f7d6931be2d4     |
| updated_at                | 2017-09-03T11:19:56Z                 |
+---------------------------+--------------------------------------+
## 创建子网
[root@Marvin-OpenStack ~]# openstack subnet create --network public \
>     --allocation-pool start=10.0.0.100,end=10.0.0.200 \
>     --dns-nameserver 10.0.0.2 --gateway 10.0.0.2 \
>     --subnet-range 10.0.0.0/24 public-subnet
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 10.0.0.100-10.0.0.200                |
| cidr              | 10.0.0.0/24                          |
| created_at        | 2017-09-03T11:22:17Z                 |
| description       |                                      |
| dns_nameservers   | 10.0.0.2                             |
| enable_dhcp       | True                                 |
| gateway_ip        | 10.0.0.2                             |
| headers           |                                      |
| host_routes       |                                      |
| id                | 5d749512-4d93-4b97-b09d-ff13029b999f |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| name              | public-subnet                        |
| network_id        | 11a77b36-fc41-4fcd-babc-1bd3051ee064 |
| project_id        | 8e8448db75034b1e8be0f7d6931be2d4     |
| project_id        | 8e8448db75034b1e8be0f7d6931be2d4     |
| revision_number   | 2                                    |
| service_types     | []                                   |
| subnetpool_id     | None                                 |
| updated_at        | 2017-09-03T11:22:17Z                 |
+-------------------+--------------------------------------+
## 查看网络信息
[root@Marvin-OpenStack ~]# neutron net-list
+--------------------------------------+--------+--------------------------------------------------+
| id                                   | name   | subnets                                          |
+--------------------------------------+--------+--------------------------------------------------+
| 11a77b36-fc41-4fcd-babc-1bd3051ee064 | public | 5d749512-4d93-4b97-b09d-ff13029b999f 10.0.0.0/24 |
+--------------------------------------+--------+--------------------------------------------------+
[root@Marvin-OpenStack ~]# neutron subnet-list
+--------------------------------------+---------------+-------------+----------------------------------------------+
| id                                   | name          | cidr        | allocation_pools                             |
+--------------------------------------+---------------+-------------+----------------------------------------------+
| 5d749512-4d93-4b97-b09d-ff13029b999f | public-subnet | 10.0.0.0/24 | {"start": "10.0.0.100", "end": "10.0.0.200"} |
+--------------------------------------+---------------+-------------+----------------------------------------------+
[root@Marvin-OpenStack ~]# openstack network list
+--------------------------------------+--------+--------------------------------------+
| ID                                   | Name   | Subnets                              |
+--------------------------------------+--------+--------------------------------------+
| 11a77b36-fc41-4fcd-babc-1bd3051ee064 | public | 5d749512-4d93-4b97-b09d-ff13029b999f |
+--------------------------------------+--------+--------------------------------------+
```

###### 9.2 创建云主机类型,生成一个键值对

```
[root@Marvin-OpenStack ~]# openstack flavor create --id 0 --vcpus 1 --ram 64 --disk 1 m1.nano
+----------------------------+---------+
| Field                      | Value   |
+----------------------------+---------+
| OS-FLV-DISABLED:disabled   | False   |
| OS-FLV-EXT-DATA:ephemeral  | 0       |
| disk                       | 1       |
| id                         | 0       |
| name                       | m1.nano |
| os-flavor-access:is_public | True    |
| properties                 |         |
| ram                        | 64      |
| rxtx_factor                | 1.0     |
| swap                       |         |
| vcpus                      | 1       |
+----------------------------+---------+
[root@Marvin-OpenStack ~]# source demo-openstack 
[root@Marvin-OpenStack ~]# ssh-keygen -q -N ""
Enter file in which to save the key (/root/.ssh/id_rsa): 
[root@Marvin-OpenStack ~]# openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey  ## 将生成的键值对上传到openstack
+-------------+-------------------------------------------------+
| Field       | Value                                           |
+-------------+-------------------------------------------------+
| fingerprint | fa:bd:6d:4e:24:56:41:c6:37:11:65:81:7f:04:41:53 |
| name        | mykey                                           |
| user_id     | bccfde0f0711448c8e3855ac4dcb8e19                |
+-------------+-------------------------------------------------+
## 增加安全组规则,开发22端口
[root@Marvin-OpenStack ~]# openstack security group rule create --proto icmp default
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| created_at        | 2017-09-03T11:28:19Z                 |
| description       |                                      |
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| headers           |                                      |
| id                | 79d3f638-7efc-422f-9db2-4e739de3814c |
| port_range_max    | None                                 |
| port_range_min    | None                                 |
| project_id        | 9e6ed2044de448c5b6064da5e61108f3     |
| project_id        | 9e6ed2044de448c5b6064da5e61108f3     |
| protocol          | icmp                                 |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| revision_number   | 1                                    |
| security_group_id | e08a5e52-71df-4349-92e3-412b5aee27c5 |
| updated_at        | 2017-09-03T11:28:19Z                 |
+-------------------+--------------------------------------+
[root@Marvin-OpenStack ~]# openstack security group rule create --proto tcp --dst-port 22 default
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| created_at        | 2017-09-03T11:28:37Z                 |
| description       |                                      |
| direction         | ingress                              |
| ethertype         | IPv4                                 |
| headers           |                                      |
| id                | 17ac408e-f270-4bd6-ab18-7f926246ac76 |
| port_range_max    | 22                                   |
| port_range_min    | 22                                   |
| project_id        | 9e6ed2044de448c5b6064da5e61108f3     |
| project_id        | 9e6ed2044de448c5b6064da5e61108f3     |
| protocol          | tcp                                  |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| revision_number   | 1                                    |
| security_group_id | e08a5e52-71df-4349-92e3-412b5aee27c5 |
| updated_at        | 2017-09-03T11:28:37Z                 |
+-------------------+--------------------------------------+
```

###### 9.3 创建云主机需要验证的信息

```
[root@Marvin-OpenStack ~]# openstack flavor list   ## 云主机类型
+----+---------+-----+------+-----------+-------+-----------+
| ID | Name    | RAM | Disk | Ephemeral | VCPUs | Is Public |
+----+---------+-----+------+-----------+-------+-----------+
| 0  | m1.nano |  64 |    1 |         0 |     1 | True      |
+----+---------+-----+------+-----------+-------+-----------+
[root@Marvin-OpenStack ~]# openstack image list  ## 镜像
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| a131bcba-8fa7-4a0e-8333-a36046c31a9c | cirros | active |
+--------------------------------------+--------+--------+
[root@Marvin-OpenStack ~]# openstack network list  ## 网络
+--------------------------------------+--------+--------------------------------------+
| ID                                   | Name   | Subnets                              |
+--------------------------------------+--------+--------------------------------------+
| 11a77b36-fc41-4fcd-babc-1bd3051ee064 | public | 5d749512-4d93-4b97-b09d-ff13029b999f |
+--------------------------------------+--------+--------------------------------------+
[root@Marvin-OpenStack ~]# openstack security group list   ## 安全策略
+--------------------------------------+---------+------------------------+----------------------------------+
| ID                                   | Name    | Description            | Project                          |
+--------------------------------------+---------+------------------------+----------------------------------+
| e08a5e52-71df-4349-92e3-412b5aee27c5 | default | Default security group | 9e6ed2044de448c5b6064da5e61108f3 |
+--------------------------------------+---------+------------------------+----------------------------------+
```

###### 9.4 创建云主机

```
[root@Marvin-OpenStack ~]# source demo-openstack 
[root@Marvin-OpenStack ~]# openstack server create --flavor m1.nano --image cirros \
>     --nic net-id=11a77b36-fc41-4fcd-babc-1bd3051ee064 --security-group default \
>     --key-name mykey marvinopen1-instance
+--------------------------------------+-----------------------------------------------+
| Field                                | Value                                         |
+--------------------------------------+-----------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                        |
| OS-EXT-AZ:availability_zone          |                                               |
| OS-EXT-STS:power_state               | NOSTATE                                       |
| OS-EXT-STS:task_state                | scheduling                                    |
| OS-EXT-STS:vm_state                  | building                                      |
| OS-SRV-USG:launched_at               | None                                          |
| OS-SRV-USG:terminated_at             | None                                          |
| accessIPv4                           |                                               |
| accessIPv6                           |                                               |
| addresses                            |                                               |
| adminPass                            | k2HpAW2f5ELA                                  |
| config_drive                         |                                               |
| created                              | 2017-09-03T11:34:24Z                          |
| flavor                               | m1.nano (0)                                   |
| hostId                               |                                               |
| id                                   | c75fa09b-cb20-4e16-9def-116bfc39a4ec          |
| image                                | cirros (a131bcba-8fa7-4a0e-8333-a36046c31a9c) |
| key_name                             | mykey                                         |
| name                                 | marvinopen1-instance                          |
| os-extended-volumes:volumes_attached | []                                            |
| progress                             | 0                                             |
| project_id                           | 9e6ed2044de448c5b6064da5e61108f3              |
| properties                           |                                               |
| security_groups                      | [{u'name': u'default'}]                       |
| status                               | BUILD                                         |
| updated                              | 2017-09-03T11:34:32Z                          |
| user_id                              | bccfde0f0711448c8e3855ac4dcb8e19              |
+--------------------------------------+-----------------------------------------------+
[root@Marvin-OpenStack ~]# openstack server list
+--------------------------------------+----------------------+--------+-------------------+------------+
| ID                                   | Name                 | Status | Networks          | Image Name |
+--------------------------------------+----------------------+--------+-------------------+------------+
| c75fa09b-cb20-4e16-9def-116bfc39a4ec | marvinopen1-instance | ACTIVE | public=10.0.0.107 | cirros     |
+--------------------------------------+----------------------+--------+-------------------+------------+
[root@Marvin-OpenStack ~]# ping 10.0.0.107 -c4
PING 10.0.0.107 (10.0.0.107) 56(84) bytes of data.
64 bytes from 10.0.0.107: icmp_seq=1 ttl=64 time=0.878 ms
64 bytes from 10.0.0.107: icmp_seq=2 ttl=64 time=1.40 ms
64 bytes from 10.0.0.107: icmp_seq=3 ttl=64 time=0.898 ms
64 bytes from 10.0.0.107: icmp_seq=4 ttl=64 time=1.01 ms

--- 10.0.0.107 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 0.878/1.050/1.408/0.215 ms
## 因为前面配置了键值对,可以直接ssh登录,不需要密码
[root@Marvin-OpenStack ~]# ssh cirros@10.0.0.107
The authenticity of host '10.0.0.107 (10.0.0.107)' can't be established.
RSA key fingerprint is a4:cf:41:2f:4e:eb:6f:1e:84:e7:23:f7:46:a6:4a:af.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.0.0.107' (RSA) to the list of known hosts.
$ ifconfig
eth0      Link encap:Ethernet  HWaddr FA:16:3E:C0:83:39  
          inet addr:10.0.0.107  Bcast:10.0.0.255  Mask:255.255.255.0
          inet6 addr: fe80::f816:3eff:fec0:8339/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:255 errors:0 dropped:0 overruns:0 frame:0
          TX packets:184 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:31309 (30.5 KiB)  TX bytes:20644 (20.1 KiB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
## 第一台云主机就这样建成了,美不美呢,哈哈!!!
## 查看VNC的url地址
[root@Marvin-OpenStack ~]# openstack console url show marvinopen1-instance
+-------+--------------------------------------------------------------------------------+
| Field | Value                                                                          |
+-------+--------------------------------------------------------------------------------+
| type  | novnc                                                                          |
| url   | http://10.0.0.56:6080/vnc_auto.html?token=3a4766ca-443a-469b-b2a7-328d95ad7109 |
+-------+--------------------------------------------------------------------------------+
```

下面可以使用vnc的地址去直接访问浏览器,就可以直接控制虚拟机了  
浏览器访问: [http://10.0.0.56:6080/vnc_auto.html?token=3a4766ca-443a-469b-b2a7-328d95ad7109](https://link.jianshu.com?t=http://10.0.0.56:6080/vnc_auto.html?token=3a4766ca-443a-469b-b2a7-328d95ad7109)

![](https://upload-images.jianshu.io/upload_images/7715809-1c809a80b7df85ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

Paste_Image.png

可以看到虚拟机正在运行,输入帐号cirros,密码: cubswin:) 可以登录到服务器

```
nova stop 云主机的ID  == 停止云主机
nova start 云主机的ID  == 启动云主机
nova delete 云主机的ID  == 删除云主机
```

### OpenStack的web管理界面

#### 1.Horizon

Horizon提供了一个web界面来操作openstack,它是基于api进行开发的,不需要连接数据库,支持群集.在我们所搭建的openstack所有的组件中,都直接或者间接的和keystone有关系,所以Horizon的安装还是让它去注册到keystone就可以了.  
我们需要注意的是,openstack创建云主机的方式有三种:命令行, api, Horizon  
我这里将Horizon安装在计算节点上,因为在管理节点安装keystone的时候已经安装了apache服务,避免冲突.  
在计算节点上安装Horizon,就要保持计算节点和管理节点的时间一致

```
[root@Marvin-Compute ~]# yum install openstack-dashboard -y
[root@Marvin-Compute openstack-dashboard]# ll
total 108
-rw-r----- 1 root apache   201 Jun 21 14:12 ceilometer_policy.json
-rw-r----- 1 root apache  5391 Jun 21 14:12 cinder_policy.json
-rw-r----- 1 root apache  1244 Jun 21 14:12 glance_policy.json
-rw-r----- 1 root apache  4544 Jun 21 14:12 heat_policy.json
-rw-r----- 1 root apache  9699 Jun 21 14:12 keystone_policy.json
-rw-r----- 1 root apache 30940 Sep  4  2017 local_settings
-rw-r----- 1 root apache 10476 Jun 21 14:12 neutron_policy.json
-rw-r----- 1 root apache 28383 Jun 21 14:12 nova_policy.json
[root@Marvin-Compute openstack-dashboard]# vim local_settings
29 ALLOWED_HOSTS = ['*',]
55 OPENSTACK_API_VERSIONS = {
 56     "data-processing": 1.1,
 57     "identity": 3,
 58     "image": 2,
 59     "volume": 2,
 60     "compute": 2,
 61 }
66 OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
74 OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'default'
160 OPENSTACK_HOST = "10.0.0.56"
161 OPENSTACK_KEYSTONE_URL = "http://%s:5000/v2.0" % OPENSTACK_HOST
162 OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
273 OPENSTACK_NEUTRON_NETWORK = {
274     'enaele_router': False,
275     'enable_quotas': False,
276     'enable_ipv6': False,
277     'enable_distributed_router': False,
278     'enable_ha_router': False,
279     'enable_lb': False,
280     'enable_firewall': False,
281     'enable_vpn': False,
282     'enable_fip_topology_check': False,
408 TIME_ZONE = "Asia/Shanghai"
## 启动
[root@Marvin-Compute openstack-dashboard]# systemctl enable httpd
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
[root@Marvin-Compute openstack-dashboard]# systemctl start httpd
[root@Marvin-Compute openstack-dashboard]# lsof -i:80
COMMAND   PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
httpd   10950   root    4u  IPv6 154317      0t0  TCP *:http (LISTEN)
httpd   10952 apache    4u  IPv6 154317      0t0  TCP *:http (LISTEN)
httpd   10953 apache    4u  IPv6 154317      0t0  TCP *:http (LISTEN)
httpd   10954 apache    4u  IPv6 154317      0t0  TCP *:http (LISTEN)
httpd   10955 apache    4u  IPv6 154317      0t0  TCP *:http (LISTEN)
httpd   10956 apache    4u  IPv6 154317      0t0  TCP *:http (LISTEN)
[root@Marvin-Compute openstack-dashboard]# netstat -ntal
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:5900            0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 10.0.0.57:59237         10.0.0.56:5672          ESTABLISHED
tcp        0      0 10.0.0.57:59233         10.0.0.56:5672          ESTABLISHED
tcp        0      0 10.0.0.57:59230         10.0.0.56:5672          ESTABLISHED
tcp        0      0 127.0.0.1:47713         127.0.0.1:47750         ESTABLISHED
tcp        0      0 10.0.0.57:59229         10.0.0.56:5672          ESTABLISHED
tcp        0      0 10.0.0.57:59226         10.0.0.56:5672          ESTABLISHED
tcp        0      0 10.0.0.57:59234         10.0.0.56:5672          ESTABLISHED
tcp        0      0 10.0.0.57:59232         10.0.0.56:5672          ESTABLISHED
tcp        0      0 10.0.0.57:59231         10.0.0.56:5672          ESTABLISHED
tcp        0      0 10.0.0.57:59235         10.0.0.56:5672          ESTABLISHED
tcp        0      0 10.0.0.57:59236         10.0.0.56:5672          ESTABLISHED
tcp        0     52 10.0.0.57:22            10.0.0.13:53736         ESTABLISHED
tcp        0      0 127.0.0.1:47750         127.0.0.1:47713         ESTABLISHED
tcp6       0      0 ::1:25                  :::*                    LISTEN     
tcp6       0      0 :::111                  :::*                    LISTEN     
tcp6       0      0 :::80                   :::*                    LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN
```

这里配置完成,开始验证,浏览器输入[http://10.0.0.57/dashboard](https://link.jianshu.com?t=http://10.0.0.57/dashboard)



![](https://upload-images.jianshu.io/upload_images/7715809-182ecdf42f820085.png?imageMogr2/auto-orient/)





![](https://upload-images.jianshu.io/upload_images/7715809-1222e24d07157dcf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)



![](https://upload-images.jianshu.io/upload_images/7715809-5935887b5c3b85b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)


































































