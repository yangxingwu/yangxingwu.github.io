---
layout: post
title:  "Tuning Nginx"
date:   2017-011-29 17:25:00 +0800
categories: nginx
---

做`nginx`压测，发现qps总是上不去，于是尝试一了把参数调优；仅仅是流水账形式记录  

## 调整`nginx`返回内容

第一步想到的就是`nginx`不要返回文件，直接返回简单的字符串；于是配置

```bash
server {
	listen 80;
		location / {
		default_type text/html;
		return 200 'hello world';
	}
}
```

尝试了几把，发现总是不成功；后来发现是发型版自带`nginx`配置文件的坑:

```bash
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
```

上述两行配置，在其中也配置`server 80`；前面的配置被覆盖掉了；注释掉后解决问题  

## 网卡中断问题

运用`top`查看，发现系统`cpu 0`总是软中断占用`si`过高；怀疑是网卡没开始多队列以及没有`set_irq_affinity`导致的；  

运用`ethtool`查看：

```bash
~bash# ethtool -l eth0
Channel parameters for eth1:
Cannot get device channel parameters
: Operation not supported
```

发现其不支持多队列设置；遂切换到另外一台支持多队列的网卡；

单队列网卡可以通过`RPS`设置来达到中断均衡的目的，这里不详述；多队列网卡可以通过[set_irq_affinity.sh](https://github.com/xrl/pf_ring/blob/master/drivers/intel/ixgbe/ixgbe-3.1.15-FlowDirector-NoTNAPI/scripts/set_irq_affinity.sh)这个脚本来设置调整  

### set_irq_affinity

这个脚本做的工作其实很简单

1. 查询网卡中断号
2. 给相应的中断号配置给定的cpu

#### 查询网卡中断号

```bash
cat /proc/interrupts | grep eth
```

#### 给相应的中断号配置给定的cpu

```bash
echo ff > /proc/irq/45/smp_affinity
echo 0-7 > /proc/irq/45/smp_affinity_list
```

### 
