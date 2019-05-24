---
layout:     post
title:      "OpenStack手动部署错误整理 "
subtitle:   ""
date:       2019-05-23 10:04:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
    - OpenStack
---


### 1、 Openstack nova service-list检查到服务状态异常

* 问题描述

  Openstack nova service-list检查到服务状态异常   

* 解决方案

nova service-list检查到服务状态异常，通常有三种原因（以nova-compute为例）：

一、**进程故障。**

**排查方法：**用cps template-instance-list –service nova  nova-compute命令检查组件状态：

如果是active，证明组件进程正常；

如果是fault，在novaControl日志中搜索ERROR信息。**可能原因**：keystone异常，存储多路径异常，openvswich_agent异常。

二、**公共服务异常（****gaussdb****、****rabbitmq****等）导致组件更新的消息无法及时写入数据库。**

**排查方法：**登陆FM查看告警，是否存在gaussdb或者rabbitmq相关告警：

如果没有，进入nova-compute日志搜索ERROR信息进行分析；

如果有告警，按照告警指导恢复后再检查是否ok。**可能原因：**rabbitmq异常导致消息无法正常发送，gaussdb异常导致无法更新数据库信息。

三、**环境问题（节点故障，时间不同步等）**

排查方法：登陆FM检查是否存在对应告警。或者：

1、在后台执行cps host-list检查是否有fault的节点（单板故障的表现现象为当前节点上所有服务都会异常）。

2、在后台执行ntp time-detal –host all命令，检查result一栏是否有非常大的数字（该数字为对应节点与主节点的时间差）。（时间不同步的表现现象为某个或某几个节点上的服务状态一会好一会坏）。





### 2、 openstack compute service list报错(HTTP 503)

[root@controller ~]# openstack compute service list
Unknown Error (HTTP 503) (Request-ID: req-b4bedf97-115c-401d-9775-b97ec09d9d20)
Unknown Error (HTTP 503)

未知错误（HTTP 503）

/var/log/nova/nova-api.log错误日志显示

2017-11-16 11:54:46.259 16440 ERROR keystonemiddleware.auth_token [-] Bad response code while validating token: 400
2017-11-16 11:54:46.259 16440 WARNING keystonemiddleware.auth_token [-] Identity response: {"error": {"message": "Expecting to find username or userId in passwordCredentials - the server could not comply with the request since it is either malformed or otherwise incorrect. The client is assumed to be in error.", "code": 400, "title": "Bad Request"}}
2017-11-16 11:54:46.260 16440 CRITICAL keystonemiddleware.auth_token [-] Unable to validate token: Failed to fetch token data from identity server
2017-11-16 11:54:46.263 16440 INFO nova.osapi_compute.wsgi.server [-] 192.168.100.10 "GET /v2.1/af24a3c94886470183c864ef0f161b4c/os-services HTTP/1.1" status: 503 len: 323 time: 0.0119650

说是验证令牌问题
检查配置文件中[keystone_authtoken]设置是否有误

确认无误后重启服务

修改配置文件[keystone_autoken]确认是否无误
[root@controller ~]# vi /etc/nova/nova.conf 
重启服务
[root@controller ~]# systemctl restart openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service   openstack-nova-conductor.service openstack-nova-novncproxy.service
显示正常
[root@controller ~]# openstack compute service list
+----+------------------+------------+----------+---------+-------+----------------------------+
| Id | Binary           | Host       | Zone     | Status  | State | Updated At                 |
+----+------------------+------------+----------+---------+-------+----------------------------+
|  1 | nova-scheduler   | controller | internal | enabled | up    | 2017-11-16T16:59:41.000000 |
|  2 | nova-conductor   | controller | internal | enabled | up    | 2017-11-16T16:59:41.000000 |
|  3 | nova-consoleauth | controller | internal | enabled | up    | 2017-11-16T16:59:46.000000 |
|  6 | nova-compute     | compute    | nova     | enabled | up    | 2017-11-16T16:59:42.000000 |
+----+------------------+------------+----------+---------+-------+----------------------------+

---



### 3、控制节点安装NTP****时报错：**Could not resolve host: mirrorlist.centos.org

解决办法：DNS配置错误，重新按照实际环境配置DNS。



### 4、计算节点无法同步控制节点NTP****服务。**

解决办法：原因在于控制节点防火墙没关。这里也提醒一下，配置完网络后建议关闭防火墙和selinux。



### 5、创建域、项目、用户、角色时无法创建成功。

实际控制节点主机名为controller1，而配置文件中写成了controller，最终卸载了keystone、httpd、wsgi，重新配置后执行正确。



这里也提醒一下各位，配置文件中有很多地方需要配置主机名，例如：

> In the[keystone_authtoken]and[paste_deploy]sections, configure Identity service access:

[keystone_authtoken]

...

auth_uri=http://controller:5000

auth_url=http://controller:35357

memcached_servers=controller:11211

auth_type=password

project_domain_name=Default

user_domain_name=Default

project_name=service

username=glance

password=GLANCE_PASS

[paste_deploy]

...

flavor=keystone

这里面就需要根据实际主机名来替换掉官方配置，或者干脆直接换成IP地址。类似这样的配置有很多，需要注意。



### 6、配置镜像服务出错，无法同步glance****数据库，无法上传镜像。**

解决办法：查看openstack-glance-api.service状态时看到没有启动，提示启动太频繁。

根据网上的经验多方尝试，删除了glance数据库，重新同步，依然没有成功；后又将镜像保存位置权限修改为777，未果。使用su -s /bin/sh -c "glance-manage db_sync" glance命令同步（之前没有用su,因为一直是root用户）后发现IOError: [Errno 13] Permission denied: '/var/log/glance/api.log'

查看权限后发现有问题，# ll /var/log/glance/api.log，将其修改为glance组，glance用户，再次同步数据库，成功。

但是这里还是不能上传镜像，报错：410 Gone: Error in store configuration. Adding images to store is disabled. (HTTP N/A)通过在网上查找相关资料，有人在安装J版本时遇到过这个问题，在这个文件中，/etc/glance/glance-api.conf，选项[glance_store]中有一项default_store = file，最终注释掉这个参数。上传镜像成功！





### 7、openstack 缺少 placement Service 解决办法

根据官方文档安装，当启动nova-compute时会报错

PlacementNotConfigured: This compute is not configured to talk to the placement service

原因：官方文档中遗漏了-nova-placement-api的安装

我总结的安装步骤

1、控制节点

yum install openstack-nova-placement-api

openstack service create --name placement --description "OpenStack Placement" placement

openstack endpoint create --region RegionOne placement public[http://<ip>:8778](https://blog.51cto.com/superbigsea/1901216)

openstack endpoint create --region RegionOne placement admin [http://<ip>:8778](https://blog.51cto.com/superbigsea/1901216)

openstack endpoint create --region RegionOne placement intenal[http://<ip>:8778](https://blog.51cto.com/superbigsea/1901216)

systemctl restart httpd  

2、计算节点

编辑 /etc/nova/nova.conf

增加

[placement]

auth_uri = http://controller:5000

auth_url = http://controller:35357

memcached_servers = controller:11211

auth_type = password

project_domain_name = default

user_domain_name = default

project_name = service

username = nova

password = ******

os_region_name = RegionOne  

重启 systemctl restart openstack-nova-compute.service



### 8、 关于neutron.service启动不成功

neutron.service启动不成功一般有三种错误可能

前两种是启动时，长时间卡主无反应，第三种为启动失败报错

1.neutron.service配置文件中core_plugin = xxx 的设置错误，错误和没有写都会导致启动时卡主

第一种日志无明显报错

2.neutron.service配置文件中[oslo_messaging_rabbit]下rabbit_password=xxx的密码设置错误，此项错误会导致启动时卡主

第二种报错AMQP5672

3.如果你没有设置网络服务初始化脚本的超链接将导致neutron.service启动失败

[root@controller ~]# systemctl restart neutron-server.service   
Job for neutron-server.service failed because the control process exited with error code. See "systemctl status neutron-server.service" and "journalctl -xe" for details.
设置后启动成功

[root@controller ~]# ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
[root@controller ~]# systemctl restart neutron-server.service 

——————————————————————————————————————————————————————————————————

注意：不只是neutron.service写错了会导致失败

如果neutron-linuxbridge-agent.service配置文件中[ml2]配置下tenant_network_types = vxlan写错了也会导致neutron.service启动失败。

有一个神奇的现象就是，上述那个写错了，neutron-linuxbridge-agent.service启动居然是成功，反而是neutron.service的报错.....这就很气

[root@controller ~]# systemctl restart neutron-server.service
Job for neutron-server.service failed because the control process exited with error code. See "systemctl status neutron-server.service" and "journalctl -xe" for details.
[root@controller ~]# systemctl restart neutron-linuxbridge-agent.service
需要特别注意，写错了neutron-linuxbridge-agent.service的配置居然导致neutron.service是启动失败，错了的反而启动正常

—————————————————————————————————————————————————————————

有一种没有报错的情况需要注意[keystone_authtoken]下密码错误，这是会启动成功的，但是在neutron agent-list中会报错

503 Service Unavailable

The server is currently unavailable. Please try again at a later time.

Neutron server returns request_ids: ['req-2c1666fe-e70f-45e2-b5fd-98e154050f74']



503的报错大多是keystone认证问题



### 9、关于neutron服务模块启动失败的解决方案

1.执行启动neutron服务报错[root@localhost ~]# systemctl start neutron-server.service  
Job for neutron-server.service failed. See 'systemctl status neutron-server.service' and 'journalctl -xn' for details.  

2.查看/var/log/neutron/server.log  
2015-06-23 22:56:35.527 5297 INFO oslo.messaging._drivers.impl_rabbit [-] Connecting to AMQP server on localhost:5672  
2015-06-23 22:56:43.548 5297 ERROR oslo.messaging._drivers.impl_rabbit [-] AMQP server localhost:5672 closed the connection. Check login credentials: Socket closed  

发现是rabbitmq连接不上的问题  

3.在/etc/rabbitmq/下创建rabbitmq.config包括如下内容：  
    [  
      {rabbit, [  
        {default_user, <<"guest">>},  
        {default_pass, <<"guest">>}  
      ]},  
      {kernel, [  
      ]}  
    ].  

% EOF  

问题得以解决。neutron网络服务模块启动成功。



### 10、openstack访问DashBoard 500错误

安装官方文档一步一步安装openstack 每一步都验证，没有问题  
安装了keystone、glance、nova、neutron、验证都没有问题  
安装horizon之后，重启httpd  
访问http://controller/dashboard 访问很久之后返回500  
按照网上的解决方法，在/etc/httpd/conf.d/openstack-dashboard.conf   加入 WSGIApplicationGroup %{GLOBAL}








