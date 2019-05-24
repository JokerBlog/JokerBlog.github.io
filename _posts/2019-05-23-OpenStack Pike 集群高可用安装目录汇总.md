---
layout:     post
title:      "OpenStack Pike集群高可用安装目录汇总 "
subtitle:   ""
date:       2019-05-23 09:39:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
    - OpenStack
---




**[openstack pike 单机 一键安装 shell](http://dwz.cn/openstackinstall)**

############################openstack控制节点单独安装

**#------控制节点安装------**  
[##1.Centos7环境准备](http://www.cnblogs.com/elvi/p/7614035.html)  
[##2.基础服务(MysqlSQL,RabbitMQ)](http://www.cnblogs.com/elvi/p/7614057.html)  
[##3.Keystone 验证服务](http://www.cnblogs.com/elvi/p/7614067.html)  
[##4.Glance镜像服务](http://www.cnblogs.com/elvi/p/7614072.html)  
[##5.1 Nova控制节点](http://www.cnblogs.com/elvi/p/7614473.html)  
[##6.1 Neutron控制节点](http://www.cnblogs.com/elvi/p/7614490.html)  
[##7.Dashboard web管理界面](http://www.cnblogs.com/elvi/p/7614497.html)

**#------计算节点安装------**  
[##1.Centos7环境准备](http://www.cnblogs.com/elvi/p/7614035.html)  
[##5.2 Nova计算节点](http://www.cnblogs.com/elvi/p/7614478.html)  
[##6.2 Neutron计算节点](http://www.cnblogs.com/elvi/p/7614494.html)

**#------cinder块存储------**  
[##1.cinder存储节点](http://www.cnblogs.com/elvi/p/7735881.html)  
[##2.cinder控制节点](http://www.cnblogs.com/elvi/p/7735993.html)

############################openstack集群节点，3个controller  
**#openstack pike 高可用、均衡负载**

#---控制节点高可用、均衡负载---

[#0.集群环境准备](http://www.cnblogs.com/elvi/p/7736521.html)      [OpenStack源部署(推荐&可选)](http://www.cnblogs.com/elvi/p/7657770.html)  
[#1.集群高可用pacemaker+haproxy](http://www.cnblogs.com/elvi/p/7736570.html)  
[#2.Mariadb Galera Cluster集群](http://www.cnblogs.com/elvi/p/7736637.html)  
[#3.RabbitMQ Cluster集群](http://www.cnblogs.com/elvi/p/7736661.html)  
[#4.Keystone验证服务群集](http://www.cnblogs.com/elvi/p/7738055.html)  
[#5.Glance 镜像服务集群](http://www.cnblogs.com/elvi/p/7736673.html)  
[#6.Nova控制节点集群](http://www.cnblogs.com/elvi/p/7736691.html)  
[#7.Neutron控制节点集群](http://www.cnblogs.com/elvi/p/7736706.html)  
[#8.Dashboard集群](http://www.cnblogs.com/elvi/p/7736724.html)  
[#9.1 cinder存储节点安装配置](http://www.cnblogs.com/elvi/p/7736746.html)  
[#9.2 cinder控制节点集群](http://www.cnblogs.com/elvi/p/7736768.html)  
[#10. Nova计算节点安装配置](http://www.cnblogs.com/elvi/p/7738165.html)

[openstack高可用haproxy配置](http://www.cnblogs.com/elvi/p/7737297.html)

[快速增加controller节点](http://www.cnblogs.com/elvi/p/7775121.html)

可根据实际情况，把控制节点服务，拆分到不同物理节点

############################

**#------后续配置、使用------**  
[#使用linuxbridge+vlan网络模式](http://www.cnblogs.com/elvi/p/8184579.html)

[#使用linuxbridge+vxlan网络模式](http://www.cnblogs.com/elvi/p/7834787.html)  
[#使用openvswitch+vxlan网络模式](http://www.cnblogs.com/elvi/p/7834788.html)  
[#创建vxlan网络](http://www.cnblogs.com/elvi/p/7834866.html)  
[#创建虚机命令](http://www.cnblogs.com/elvi/p/7614499.html)

[#openstack pike与ceph集成](http://www.cnblogs.com/elvi/p/7897191.html)

[#openstack centos6 centos7 镜像制作](http://www.cnblogs.com/elvi/p/7922421.html)

[#openstack windows 2008镜像 制作](http://www.cnblogs.com/elvi/p/8001298.html)

[openstack域名配置](http://www.cnblogs.com/elvi/p/8005209.html)

[ #openstack故障处理汇总](http://www.cnblogs.com/elvi/p/7804507.html)

#其它部署后续补充

#官方参考  
[https://docs.openstack.org/pike/install/](https://docs.openstack.org/pike/install/)

#中文只作为参考，配置不同  
[https://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/index.html](https://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/index.html)
