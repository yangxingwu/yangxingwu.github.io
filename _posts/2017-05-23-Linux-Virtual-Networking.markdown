---
layout: post
title:  "Linux Virtual Networking"
date:   2017-05-23 16:52:49 +0800
categories: container
---

参见[CSDN 博文](http://blog.csdn.net/dog250/article/details/45788279)  

## VETH  

> 每一个VETH网卡都是一对儿以太网卡，除了`xmit`接口与常规的以太网卡驱动不同之外，其它的几乎就是一块标准的以太网卡。VETH网卡既然是一对儿两个，那么我们把一块称作另一块的peer，标准上也是这么讲的。其`xmit`的实现就是：将数据发送到其peer，触发其peer的`RX`。那么问题来了，这些数据如何发送到VETH网卡对儿之外呢？自问必有自答，自答如下：  
> 1. 如果确实需要将数据发到外部，通过将一块VETH网卡和一块普通ETHx网卡进行bridge，通过bridge逻辑将数据forward到ETHx，进而发出;
> 2. 难道非要把数据包发往外部吗？类似loopback那样的，不就是自发自收吗？使用VETH可以很方面并且隐秘地将数据包从一个net namespace发送到同一台机器的另一个net namespace，并且不被嗅探到。

![VETH](http://img.blog.csdn.net/20150517134821346?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG9nMjUw/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  


## Bridge  

`STP`: spanning tree protocol; 参见[wiki](https://en.wikipedia.org/wiki/Spanning_Tree_Protocol); 主要用来检测bridge中是否存在环。  

关于bridge的文档：https://wiki.linuxfoundation.org/networking/bridge  

## MacVlan

Refer to <https://hicu.be/bridge-vs-macvlan>  

![MacVlan](https://hicu.be/wp-content/uploads/2016/03/linux-macvlan-1.png)  

Macvlan allows you to configure sub-interfaces (also termed slave devices) of a parent, physical Ethernet interface (also termed upper device), each with its own unique MAC address, and consequently its own IP address. Applications, VMs and containers can then bind to a specific sub-interface to connect directly to the physical network, **using their own MAC and IP address**.  

Macvlan is a near-ideal solution to natively connect VMs and containers to a physical network, but it has its shortcomings:

1. The switch the host is connected to may have a policy that **limits the number of different MAC addresses on a physical port**. Although you should really work with your network administrator to change the policy, there are times when this might not be possible (or you just need to set up a quick PoC).  
2. Many NICs have a limit on the number of MAC addresses they support in hardware. Exceeding the limit may affect the performance.  
3. IEEE 802.11 doesn’t like multiple MAC addresses on a single client. It is likely macvlan sub-interfaces will be blocked by your wireless interface driver, AP or both. There are somehow complex ways around that limitation, but why not stick to a simple solution?

### MacVlan modes

Each sub-interface can be in one of 4 modes that affect possible traffic flows.  

#### Private

> Sub-interfaces on the same parent interface cannot communicate with each other. All frames from sub-interfaces are forwarded out through the parent interface. Even if physical switch reflects the frame sourced from one sub-interface and destined to another sub-interface, frame gets dropped.

#### Passthrough

> Allows a single VM to be connected directly to the physical interface. The advantage of this mode is that VM is then able to change MAC address and other interface parameters.  

`passthrough`，1. 只允许单个sub-interface；2. 这个模式的优势在于，可以更改这个单个sub-interface的一些参数  

#### Bridge

> Macvlan connects all sub-interfaces on a parent interface with a simple bridge. **Frames from one interface to another one get delivered directly and are not sent out**. Broadcast frames get flooded to all other bridge ports and to the external interface, but when they come back from a VEP switch, they are discarded. Since all macvlan sub-interface  MAC addresses are known, **macvlan bridge mode does not require MAC learning and does not need STP**. Bridge mode provides fastest communication between the VMs, but has a “flaw” you should be aware of – if parent interface state goes down, so do all macvlan sub-interfaces. VMs will not be able to communicate with each other when physical interfaces gets disconnected.

#### VEPA(Virtual Ethernet Port Aggregation)

> All frames from sub-interfaces are forwarded out through the parent interface. VEPA mode requires an IEEE 802.1Qbg aka Virtual Ethernet Port Aggregator physical switch. VEPA capable switch returns all frames where both source and destination are local to the macvlan interface. Consequently macvlan subinterfaces on the same parent interface are capable to communicate with each other through a physical switch. Broadcast frames coming in through the parent interface get flooded to all macvlan interfaces in VEPA mode.  VEPA mode is useful when you are enforcing policies on physical switch and you want all VM-to-VM traffic to traverse the physical switch.

`VEPA`相对于`bridge`模式的区别是，`VEPA`会把数据包转向交换机，然后由交换机送回来  

![VEPA](https://hicu.be/wp-content/uploads/2016/03/linux-macvlan-802.1qbg-vepa-mode.png)  

### MacVlan VS Bridge

> The macvlan is a trivial bridge that doesn’t need to do learning as it knows every mac address it can receive, so it doesn’t need to implement learning or stp. Which makes it simple stupid and and fast.  

#### Use Bridge:

When you need to connect VMs or containers on the same host. For complex topologies with multiple bridges and hybrid environments (hosts in the same Layer2 domain both on the same host and outside the host).
You need to apply advanced flood control, FDB manipulation, etc.

以下示例为：veth以及bridge组成的网络

```bash
#! /bin/bash

# add netns
ip netns add netns0
ip netns add netns1

# add veth
ip link add ns0eth0 type veth peer name ns0eth1
ip link add ns1eth0 type veth peer name ns1eth1

# add ns0eth0 to netns0, ns1eth0 to netns1
ip link set dev ns0eth0 netns netns0
ip link set dev ns1eth0 netns netns1

# set ns0eth0 up and add addr to it
ip netns exec netns0 ip link set dev ns0eth0 up
ip netns exec netns0 ip addr add 10.110.26.51/20 dev ns0eth0

# set ns1eth0 up and add addr to it
ip netns exec netns1 ip link set dev ns1eth0 up
ip netns exec netns1 ip addr add 10.110.26.52/20 dev ns1eth0

# add bridge 0
ip link add br0 type bridge
ip link set dev br0 up

# add addr to br0
ip addr add 10.110.26.53/20 dev br0

# add veth1 to br0
ip link set dev ns0eth1 up
ip link set dev ns0eth1 master br0
ip link set dev ns1eth1 up
ip link set dev ns1eth1 master br0
```

#### Use Macvlan:  

When you only need to provide egress connection to the physical network to your VMs or containers.
Because it uses less host CPU and provides slightly better throughput.

```bash
#! /bin/bash

# add sub interfaces
ip link add mac1 link eth0 type macvlan mode bridge
ip link add mac2 link eth0 type macvlan mode bridge

# add net namespace
ip netns add netns1
ip netns add netns2

# add sub interface to netns
ip link set mac1 netns netns1
ip link set mac2 netns netns2

# add address and set link up
ip netns exec netns1 ip addr add 10.110.26.51/20 dev mac1
ip netns exec netns1 ip link set mac1 up

ip netns exec netns2 ip addr add 10.110.26.52/20 dev mac2
ip netns exec netns2 ip link set mac2 up
```

需要注意的是，经过上面的配置后，两个sub interface之间可以ping通，但是自己ping自己居然不通。。。。


```bash
 ⚡ root@bash ~  ip netns exec netns2 ping 10.110.26.51
PING 10.110.26.51 (10.110.26.51) 56(84) bytes of data.
64 bytes from 10.110.26.51: icmp_seq=1 ttl=64 time=0.043 ms
64 bytes from 10.110.26.51: icmp_seq=2 ttl=64 time=0.037 ms
64 bytes from 10.110.26.51: icmp_seq=3 ttl=64 time=0.037 ms
^C
--- 10.110.26.51 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.037/0.039/0.043/0.003 ms

 ⚡ root@bash ~  ip netns exec netns1 ping 10.110.26.52
PING 10.110.26.52 (10.110.26.52) 56(84) bytes of data.
64 bytes from 10.110.26.52: icmp_seq=1 ttl=64 time=0.044 ms
64 bytes from 10.110.26.52: icmp_seq=2 ttl=64 time=0.033 ms
^C
--- 10.110.26.52 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.033/0.038/0.044/0.008 ms

 ⚡ root@bash ~  ip netns exec netns1 ping 10.110.26.51
PING 10.110.26.51 (10.110.26.51) 56(84) bytes of data.
^C
--- 10.110.26.51 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 1999ms


 ✘ ⚡ root@bash ~ ip netns exec netns2 ping 10.110.26.52
PING 10.110.26.52 (10.110.26.52) 56(84) bytes of data.
^C
--- 10.110.26.52 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 999ms
```

### 关于Mac地址

参见[wiki](https://en.wikipedia.org/wiki/MAC_address)；有趣的是，MAC地址其实是分类的，有`universal`和`local`之分：

> **Universal vs. local**  
> 
> Addresses can either be universally administered addresses or locally administered addresses. A universally administered address is uniquely assigned to a device by its manufacturer. The first three octets (in transmission order) identify the organization that issued the identifier and are known as the Organizationally Unique Identifier (OUI).[4] The remainder of the address (three octets for MAC-48 and EUI-48 or five for EUI-64) are assigned by that organization in nearly any manner they please, subject to the constraint of uniqueness. A locally administered address is assigned to a device by a network administrator, overriding the burned-in address.
> 
> Universally administered and locally administered addresses are distinguished by setting the **second-least-significant bit** of the first octet of the address. This bit is also referred to as the U/L bit, short for Universal/Local, which identifies how the address is administered. **If the bit is 0, the address is universally administered. If it is 1, the address is locally administered**. In the example address 06-00-00-00-00-00 the first octet is 06 (hex), the binary form of which is 00000110, where the second-least-significant bit is 1. Therefore, it is a locally administered address.[7] Consequently, this bit is 0 in all OUIs.

## IPVlan

Refer to [macvlan-vs-ipvlan](https://hicu.be/macvlan-vs-ipvlan)  

Ipvlan is very similar to macvlan, with an important difference. Ipvlan does not assign unique MAC addresses to created sub-interfaces. All sub-interfaces share parent’s interface MAC address, but use distinct IP addresses.  

![ipvlan](https://hicu.be/wp-content/uploads/2016/03/linux-ipvlan.png)  

Because all VMs or containers on a single parent interface use the same MAC address, ipvlan also has some shortcomings:

1. Shared MAC address can affect DHCP operations. If your VMs or containers use DHCP to acquire network settings, make sure they use unique ClientID in the DHCP request and ensure your DHCP server assigns IP addresses based on ClientID, not client’s MAC address.
2. Autoconfigured EUI-64 IPv6 addresses are based on MAC address. All VMs or containers sharing the same parent interface will auto-generate the same IPv6 address. Ensure that your VMs or containers use static IPv6 addresses or IPv6 privacy addresses and disable SLAAC.

### modes

#### Ipvlan L2

Parent interface acts as a switch between the sub-interfaces and the parent interface. All VMs or containers connected to the same parent Ipvlan interface and in the same subnet can communicate with each other directly through the parent interface. Traffic destined to other subnets is sent out through the parent interface to default gateway (a physical router). Ipvlan in L2 mode distributes broadcasts/multicasts to all sub-interfaces.  

#### Ipvlan L3

![l3-mode](https://hicu.be/wp-content/uploads/2016/03/linux-ipvlan-l3-mode-1.png)  

Ipvlan L3 mode routes the packets between all sub-interfaces, thus providing full Layer 3 connectivity. **Each sub-interface has to be configured with a different subnet**, i.e. you cannot configure 10.10.40.0/24 on both interfaces.

Broadcasts are limited to a Layer 2 domain, so they cannot pass from one sub-interface to another. **Ipvlan L3 mode does not support multicast**.

Ipvlan L3 mode does not support routing protocols, so it cannot notify the physical network router of the subnets it connects to. **You need to configure static routes on the physical router pointing to the Host’s physical interface for all subnets on the sub-interfaces**.

Ipvlan L3 mode behaves like a router – it forwards the IP packets between different subnets, however it does not reduce the TTL value of the passing packets. Thus, you will not see the Ipvlan “router” in the path when doing traceroute.  

**Ipvlan L3 can be used in conjunction with [VM or Container ran BGP](https://github.com/osrg/gobgp), used as a service advertisement protocol to advertise service availability into the network**. This advanced scenario exceeds the purpose of this post.  

### macvlan VS ipvlan

Macvlan and ipvlan cannot be used on the same parent interface at the same time.

**Use Ipvlan when**:

1. Parent interface is wireless.
2. Your parent interface performance drops because you have exceeded the number of different MAC addresses. For production, you should consider swapping your NIC for a better one and use macvlans.
3. Physical switch limits the number of MAC addresses allowed on a port (Port Security).  For production, you should solve this policy issue with your network administrator and use macvlans.
4. You run an advanced network scenario, such as advertising the service you run in the VM or container with the BGP daemon running in the same VM or container.

**Use Macvlan**:

1. In every other scenario.

### kernel doc

<https://www.kernel.org/doc/Documentation/networking/ipvlan.txt>  

```bash
# (a) Create two network namespaces - ns0, ns1
ip netns add ns0
ip netns add ns1

# (b) Create two ipvlan slaves on eth0 (master device)
ip link add link eth0 ipvl0 type ipvlan mode l2
ip link add link eth0 ipvl1 type ipvlan mode l2

# (c) Assign slaves to the respective network namespaces
ip link set dev ipvl0 netns ns0
ip link set dev ipvl1 netns ns1

# (d) Now switch to the namespace (ns0 or ns1) to configure the slave devices

# For ns0
    ip netns exec ns0 bash
	ip link set dev ipvl0 up
	ip link set dev lo up
	ip -4 addr add 127.0.0.1 dev lo
	ip -4 addr add $IPADDR dev ipvl0
	ip -4 route add default via $ROUTER dev ipvl0

# For ns1
	ip netns exec ns1 bash
	ip link set dev ipvl1 up
	ip link set dev lo up
	ip -4 addr add 127.0.0.1 dev lo
	ip -4 addr add $IPADDR dev ipvl1
	ip -4 route add default via $ROUTER dev ipvl1
```
