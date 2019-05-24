---
layout:     post
title:      "OpenStack安装文档（单点部署） "
subtitle:   "基础安装"
date:       2019-05-07 15:45:00
author:     "Joker"
header-img: "img/post-think-try-write.jpg"
tags:
    - OpenStack
---




* 系统环境：CentOS 7.6

* 安装前需要修改 /etc/hosts文件，添加 192.168.20.21       localhost(localhost为当前机器名)

* 如果是虚拟机，建议 4核、6G内存、25G容量的硬盘空间。（**官方推荐配置：**Machine with at least**16GB RAM**, processors with hardware virtualization extensions, and**at least one network adapter**.）

* 固定IP，192.168.137.20（是静态IP且能连接外网，IP部署方法链接：[http://www.cnblogs.com/suhaha/p/8619102.html](http://www.cnblogs.com/suhaha/p/8619102.html)

* 关闭防火墙等一些网络服务（此处质疑的是NetworkManager，实际测试不需要关）

  sudo systemctl disable firewalld

  sudo systemctl stop firewalld

  sudo systemctl disable NetworkManager

  sudo systemctl stop NetworkManager

  sudo systemctl enable network

  sudo systemctl start network

### 开始安装

安装过程比较简单，总共就四条命令，按顺序一步步执行就可以了——不过执行时间较长，尤其第4步，差不多要两小时；第2步更新yum所需时间也挺长，半个小时左右。

* **安装Openstack仓库**

  命令：sudo yum install -y centos-release-openstack-**queens**

* **更新yum**

  命令：sudo yum update -y

* **安装packstack**

  命令： sudo yum install -y openstack-packstack

* **安装OpenStack**

  命令：sudo packstack --allinone

  # 

  #### 

```
  Welcome to Installer setup utility
  Installing:
  Clean Up [ DONE ]
  Setting up ssh keys [ DONE ]
  Discovering hosts' details [ DONE ]
  Adding pre install manifest entries [ DONE ]
  Preparing servers [ DONE ]
  Adding AMQP manifest entries [ DONE ]
  Adding MySQL manifest entries [ DONE ]
  Adding Keystone manifest entries [ DONE ]
  Adding Glance Keystone manifest entries [ DONE ]
  Adding Glance manifest entries [ DONE ]
  Adding Cinder Keystone manifest entries [ DONE ]
  Adding Cinder manifest entries [ DONE ]
  Checking if the Cinder server has a cinder-volumes vg[ DONE ]
  Adding Nova API manifest entries [ DONE ]
  Adding Nova Keystone manifest entries [ DONE ]
  Adding Nova Cert manifest entries [ DONE ]
  Adding Nova Conductor manifest entries [ DONE ]
  Creating ssh keys for Nova migration [ DONE ]
  Gathering ssh host keys for Nova migration [ DONE ]
  Adding Nova Compute manifest entries [ DONE ]
  Adding Nova Scheduler manifest entries [ DONE ]
  Adding Nova VNC Proxy manifest entries [ DONE ]
  Adding Openstack Network-related Nova manifest entries[ DONE ]
  Adding Nova Common manifest entries [ DONE ]
  Adding Neutron API manifest entries [ DONE ]
  Adding Neutron Keystone manifest entries [ DONE ]
  Adding Neutron L3 manifest entries [ DONE ]
  Adding Neutron L2 Agent manifest entries [ DONE ]
  Adding Neutron DHCP Agent manifest entries [ DONE ]
  Adding Neutron LBaaS Agent manifest entries [ DONE ]
  Adding Neutron Metering Agent manifest entries [ DONE ]
  Adding Neutron Metadata Agent manifest entries [ DONE ]
  Checking if NetworkManager is enabled and running [ DONE ]
  Adding OpenStack Client manifest entries [ DONE ]
  Adding Horizon manifest entries [ DONE ]
  Adding Swift Keystone manifest entries [ DONE ]
  Adding Swift builder manifest entries [ DONE ]
  Adding Swift proxy manifest entries [ DONE ]
  Adding Swift storage manifest entries [ DONE ]
  Adding Swift common manifest entries [ DONE ]
  Adding Provisioning Demo manifest entries [ DONE ]
  Adding MongoDB manifest entries [ DONE ]
  Adding Ceilometer manifest entries [ DONE ]
  Adding Ceilometer Keystone manifest entries [ DONE ]
  Adding Nagios server manifest entries [ DONE ]
  Adding Nagios host manifest entries [ DONE ]
  Adding post install manifest entries [ DONE ]
  Installing Dependencies [ DONE ]
  Copying Puppet modules and manifests [ DONE ]
  Applying 192.168.1.105_prescript.pp
  192.168.1.105_prescript.pp: [ DONE ] 
  Applying 192.168.1.105_amqp.pp
  Applying 192.168.1.105_mysql.pp
  192.168.1.105_amqp.pp: [ DONE ] 
  192.168.1.105_mysql.pp: [ DONE ] 
  Applying 192.168.1.105_keystone.pp
  Applying 192.168.1.105_glance.pp
  Applying 192.168.1.105_cinder.pp
  192.168.1.105_keystone.pp: [ DONE ] 
  192.168.1.105_glance.pp: [ DONE ] 
  192.168.1.105_cinder.pp: [ DONE ] 
  Applying 192.168.1.105_api_nova.pp
  192.168.1.105_api_nova.pp: [ DONE ] 
  Applying 192.168.1.105_nova.pp
  192.168.1.105_nova.pp: [ DONE ] 
  Applying 192.168.1.105_neutron.pp
  192.168.1.105_neutron.pp: [ DONE ] 
  Applying 192.168.1.105_neutron_fwaas.pp
  Applying 192.168.1.105_osclient.pp
  Applying 192.168.1.105_horizon.pp
  192.168.1.105_neutron_fwaas.pp: [ DONE ] 
  192.168.1.105_osclient.pp: [ DONE ] 
  192.168.1.105_horizon.pp: [ DONE ] 
  Applying 192.168.1.105_ring_swift.pp
  192.168.1.105_ring_swift.pp: [ DONE ] 
  Applying 192.168.1.105_swift.pp
  Applying 192.168.1.105_provision_demo.pp
  192.168.1.105_swift.pp: [ DONE ] 
  192.168.1.105_provision_demo.pp: [ DONE ] 
  Applying 192.168.1.105_mongodb.pp
  192.168.1.105_mongodb.pp: [ DONE ] 
  Applying 192.168.1.105_ceilometer.pp
  Applying 192.168.1.105_nagios.pp
  Applying 192.168.1.105_nagios_nrpe.pp
  192.168.1.105_ceilometer.pp: [ DONE ] 
  192.168.1.105_nagios.pp: [ DONE ] 
  192.168.1.105_nagios_nrpe.pp: [ DONE ] 
  Applying 192.168.1.105_postscript.pp
  192.168.1.105_postscript.pp: [ DONE ] 
  Applying Puppet manifests [ DONE ]
  Finalizing [ DONE ]
```

# 

### 安装成功输出信息

```
 **** Installation completed successfully ******



Additional information:

 * A new answerfile was created in: /root/packstack-answers-20180321-152621.txt

 * Time synchronization installation was skipped. Please note that unsynchronized time on server instances might be problem for some OpenStack components.

 * File /root/keystonerc_admin has been created on OpenStack client host 192.168.137.20. To use the command line tools you need to source the file.

 * To access the OpenStack Dashboard browse to http://192.168.137.20/dashboard .

Please, find your login credentials stored in the keystonerc_admin in your home directory.

 * The installation log file is available at: /var/tmp/packstack/20180321-152620-CBbu76/openstack-setup.log

 * The generated manifests are available at: /var/tmp/packstack/20180321-152620-CBbu76/manifests
```

### 访问DashBoard

* 访问地址：[http://192.168.20.21/dashboard/auth/login/](http://192.168.20.21/dashboard/auth/login/)

![Snip20190505_10](/Users/dyonline/Documents/中基凌云/截图/Snip20190505_10.png)

* /root/keystonerc_admin文件中有登录dashboard的用户名和密码。

**在本地浏览器输入dashboard地址进行登录验证**

登录成功：

![Snip20190505_11](/Users/dyonline/Documents/中基凌云/截图/Snip20190505_11.png)

### 可能遇见的的错误

1.ERROR:root:Failed to load plugin from file ssl_001.py

![](https://images2018.cnblogs.com/blog/1110742/201803/1110742-20180321223140068-1660516868.png)

解决方法：

说是有可能没有安装python-setuptools包。于是用yum来进行安装，如下图，安装完成之后再执行sudo packstack --allinone命令继续安装OpenStack。

![](https://images2018.cnblogs.com/blog/1110742/201803/1110742-20180321223212029-1583288659.png)

2.ERROR : Error appeared during Puppet run: 192.168.137.20_controller.pp

Error: Execution of '/usr/bin/yum -d 0 -e 0 -y install openstack-aodh-common' returned 1:**Error downloading packages**:

You will find full trace in log /var/tmp/packstack/20180321-130307-4Ja7pR/manifests/192.168.137.20_controller.pp.log

![](https://images2018.cnblogs.com/blog/1110742/201803/1110742-20180321223224813-824556564.png)

**解决办法：手动安装报错的包**

![](https://images2018.cnblogs.com/blog/1110742/201803/1110742-20180321223240533-1856949979.png)

3.ERROR : Error appeared during Puppet run: 192.168.137.20_controller.pp

Error: /Stage[main]/Swift::Keystone::Auth/Keystone::Resource::Service_identity[swift]/Keystone_user[swift]:**Could not evaluate**: Command: 'openstack ["user", "show", "--format", "shell", ["swift", "--domain", "default"]]' has been running for more than 40 seconds (tried 2, for a total of 170 seconds)

![](https://images2018.cnblogs.com/blog/1110742/201803/1110742-20180321223306378-453582288.png)

**解决办法：**

![](https://images2018.cnblogs.com/blog/1110742/201803/1110742-20180321223317669-275518069.png)

![](https://images2018.cnblogs.com/blog/1110742/201803/1110742-20180321223323275-1286752729.png)

上图是网上查看到的解决办法（原文地址：[https://www.redhat.com/archives/rdo-list/2016-July/msg00010.html](https://www.redhat.com/archives/rdo-list/2016-July/msg00010.html)），说是**需更新/etc/hosts文件，在其中加上fqdn**。

FQDN是完全合格域名/全程域名缩写，全称为Fully Qualified Domain Name，即是域名，访问时将由DNS进行解析，得到IP。使用命令**hostname -f**查看FQDN，我查到的是openstack-rdo，跟我的主机名相同，于是我在**/etc/hosts**文件中加上如下内容，然后再次运行

![](https://images2018.cnblogs.com/blog/1110742/201803/1110742-20180321223337274-221061282.png)
