---
layout:     post
title:      Failed to start firewall with iptables
subtitle:   用iptables开启防火墙报错
date:       2018-05-25
author:     haibarai
header-img: img/post-bg-firewall.jpeg
catalog: true
tags:
    - Linux
    - iptables
---
错误原因：因为centos7.0默认不是使用iptables方式管理，而是firewalld方式。CentOS6.0防火墙用iptables管理。
## 停止并屏蔽firewalld：

    systemctl stop firewalld
    systemctl mask firewalld

## 安装iptables-services：

    yum install iptables-services
	
## 设置开机启动：

	systemctl enable iptables
## 停止/启动/重启 防火墙：

    systemctl [stop|start|restart] iptables  
    or
    service iptables [stop|start|restart] 
##  保存防火墙配置
    service iptables save  
    or  
    /usr/libexec/iptables/iptables.init save  
