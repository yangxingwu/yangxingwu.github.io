## 前端接入  

### 介绍  

1. Google front end
    1. <http://blog.huihoo.com/?p=176>
    2. <http://www.zdnet.com/article/google-the-nsa-and-the-need-for-locking-down-datacenter-traffic/>
    3. [Google cloud http(s) load balancing](https://cloud.google.com/compute/docs/load-balancing/http/)
2. [CloudFlare](www.cloudflare.com)
3. SPDY
4. GSLB, Global Server LoadBalancing
5. Facebook, [Building A Billion User Load Balancer](http://velocityconf.com/velocity2013/public/schedule/detail/28410)
6. [mod_security](http://www.modsecurity.org/)
7. [代码的艺术](http://www.newsmth.net/nForum/#!article/SoftEng/92859)

### 着力点

1. WAF, Web Application Firewall
2. Load balancing
3. Web server performance Tuning
4. Https offloading
5. DDOS
6. CDN

#### CDN动态内容加速

一般来说，CDN用于静态内容缓存及加速，那么对于动态内容，CDN能做什么呢？至少能做以下两个事情：

1. 动态路由技术：选择一个最优的回源路由，rtt最低，保证内容快速响应给用户
2. TCP优化技术：节点内通过tcp协议栈优化，减少握手时间，重用连接等

#### 什么是前段接入

什么是前端接入，或者说，前端接入架构指的是什么？web server frontend infrastructure，大概包含以下几个东西：

1. load balancing - 比如常用开源的lvs以及nginx
2. web accelerator - 比如CDN，cache等
3. DDOS
4. WAF

这部分知识内容可参考[tempesta - a open source application delivery](http://tempesta-tech.com/index.html)  

#### 关于安全DDOS

https://www.server110.com/ddos/201701/13194.html  

### 知识面

1. Operating system
    1. CSAPP
    2. 汇编语言
    3. linux insides
2. http cache

关于内核，凡设计到性能调优等相关的东西，必定要用到操作系统及内核的东西；我的目标不是成为一个内核开发者，而是要让内核知识为我所用，做到真正熟悉相关概念，需要的时候能去读及改代码。  

## 云计算平台

什么是云计算平台？我认为现在大概就分为两类，云主机管理以及app engine；所谓的云主机管理，就是基于opensack实现的虚拟机管理；所谓app engine，就是基于docker/kubernets实现的应用管理；作为一个网络方向的研发人员，切入点在哪？

### 云主机管理

我们说云主机管理，主要是讲云主机组网；具体就是，虚拟机怎么和外界交互，怎么保证虚拟机网络间的独立性，虚拟机和宿主机怎么交互？怎么管理和调整云计算网络？SDN究竟体现在哪？  

需要补充的知识点是：

1. 什么是kvm
2. 什么是qemu
3. 什么是SR-IOV

### app engine平台

所谓的app engine平台，其实就是如何管理docker虚拟机，如何做到和业务持续集成(CI)；现有方案是基于kubernets做集成，和网络相关的点有：

1. CNI，Container Network Interface
2. 容器网络性能优化

## 总结

随着技术进一步的发展，互联网公司会一步步分化；也就是说，互联网公司会越来越专注于业务层面的开发，也很可能只保留两个技术相关的部门，即前端和后端；而把所有的中间件相关的业务进行外包，或者说购买云计算公司的服务；所谓的基础架构平台，其实就是中间件服务，这个部门在以后的互联网公司中的重要性将大大降低。  

### 如何进一步的学习

1. 操作系统层面
    1. CSAPP
    2. linux insides

2. 应用层面
    1. WAF
    2. DDOS
    3. kubernets

### 元旦前目标

学习万CSAPP以及linux insides部分内容