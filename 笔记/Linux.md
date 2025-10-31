2025-10-31 10:07
Status: #idea
Tags:

# 1 一些知识
## 1.1 LVS
**LVS 是Linux Virtual Server**的简写，意即Linux虚拟服务器，是一个虚拟的服务器集群系统。**它主要的作用是实现多服务器的负载均衡**。
它是我们国家的章文嵩博士的一个开源项目。在linux内存2.6中，它已经成为内核的一部分，在此之前的内核版本则需要重新编译内核。

### 1.1.1 LVS组成部分
1. Load Balancer：它负责将客户的请求按照一定的算法分发到下一层不同的服务器进行处理，自己本身不做具体业务的处理。该层由一台或者几台Director Server组成。
2. Server Array：该层负责具体业务。
3. Shared Storage：主要是提高上一层数据和为上一层保持数据一致。

### 1.1.2 负载均衡机制
**LVS 工作在网络层，通过控制IP来实现负载均衡**。相对于其它负载均衡的解决办法，比如DNS域名轮流解析、应用层负载的调度、客户端的调度等，它的效率是非常高的。
**IPVS**是其具体的实现模块。IPVS的主要作用：安装在Director Server上面，在Director Server虚拟一个对外访问的IP（[[计算机网络#1.1 VIP]]）。用户访问VIP，到达Director Server，Director Server根据一定的规则选择一个Real Server，处理完成后然后返回给客户端数据。
IPVS为此有三种选择机制：
1. **VS/NAT**(Virtual Server via Network Address Translation)：即网络地址翻转技术实现虚拟服务器。当请求来到时，Diretor server上处理的程序将数据报文中的目标地址（即虚拟IP地址）改成具体的某台Real Server,端口也改成Real Server的端口，然后把报文发给Real Server。
	Real Server处理完数据后，需要返回给Diretor Server，然后Diretor server将数据包中的源地址和源端口改成VIP的地址和端口，最后把数据发送出去。
	由此可以看出，用户的请求和返回都要经过Diretor Server，如果数据过多，Diretor Server肯定会不堪重负。
2. **VS/TUN**（Virtual Server via IP Tunneling）：即IP隧道技术实现虚拟服务器。
	它跟VS/NAT基本一样，但是Real server是直接返回数据给客户端，不需要经过Diretor server,这大大降低了Diretor server的压力。
3. **VS/DR**（Virtual Server via Direct Routing）：即用直接路由技术实现虚拟服务器。
	VS/DR通过改写请求报文的MAC地址，将请求发送到Real Server，而Real Server将响应直接返回给客户，免去了VS/TUN中的IP隧道开销。
	这种方式是三种负载调度机制中性能最高最好的，但是必须要求Director Server与Real Server都有一块网卡连在同一物理网段上。

### 1.1.3 负载调度算法
IPVS实现了八种调度方法：
1. **轮叫调度 (Round Robin, RR)** 请求按顺序分配到服务器，简单高效，但不考虑服务器负载。
2. **加权轮叫 (Weighted Round Robin, WRR)** 根据服务器性能设置权重，权重高的服务器分配更多请求。
3. **最少连接 (Least Connections, LC)** 动态分配请求到当前连接数最少的服务器，适合负载变化大的场景。如果集群系统的真实服务器具有相近的系统性能，采用"最小连接"调度算法可以较好地均衡负载。
4. **加权最少连接 (Weighted Least Connections, WLC)** 在最少连接的基础上引入权重，性能强的服务器处理更多连接。
5. **基于局部性的最少连接 (Locality-Based Least Connections, LBLC)** 针对目标 IP 地址的调度，优先将相同目标 IP 的请求分配到同一服务器，提高缓存命中率。目前主要用于Cache集群系统。
6. **带复制的基于局部性最少连接 (Locality-Based Least Connections with Replication, LBLCR)** 维护从一个目标 IP 地址到一组服务器的映射，动态调整服务器组以优化负载。
	LBLCR 算法先根据请求的目标 IP 地址找出该目标 IP 地址对应的服务器组。按 “ 最小连接 ” 原则从该服务器组中选出一台服务器,若服务器没有超载,将请求发送到该服务器；若服务器超载，则按 “ 最小连接 ” 原则从整个集群中选出一台服务器,将该服务器加入到服务器组中,将请求发送到该服务器。同时,当该服务器组有一段时间没有被修改,将最忙的服务器从服务器组中删除,以降低负载的程度。
7. **目标地址散列 (Destination Hashing, DH)** 使用目标 IP 地址的哈希值静态映射到服务器，适合固定目标的请求，保证请求的唯一出口。
8. **源地址散列 (Source Hashing, SH)** 它正好与目标地址散列调度算法相反，根据源 IP 地址的哈希值分配服务器。常用于防火墙集群，保证请求的唯一入口。
	在实际应用中,源地址散列调度和目标地址散列调度可以结合使用在防火墙集群中,它们可以保证整个系统的唯一出入口。

### 1.1.4 具体配置操作
首先我们这里有三台机子，IP分别是192.168.132.30（Diretor server）,192.168.132.64(Real server 1)，192.168.132.68(real server 2)。并且我们假设还有一个对外访问的虚拟IP是192.168.132.254（VIP）。
另外在Diretor server上面已经安装好了ipvsadm。 下面我们VS/DR介绍详细的配置过程。
```shell
// Diretor server上面的配置：
//首先在Director Server上绑定一个虚拟IP（也叫VIP），此IP用于对外提供服务：
Ifconfig eth0:0 192.168.132.254 broadcast 192.168.132.254 netmask 255.255.255.255 up
 
//给设备eth0:0指定一条路由
route add -host 192.168.132.254 dev eth0:0
 
//启用系统的包转发功能
echo "1">/proc/sys/net/ipv4/ip_forward
 
//清除ipvsadm以前的设置
ipvsadm -C

//添加一个新的虚拟IP记录192.168.132.254，其持续服务之间是120秒
ipvsadm -A -t 192.168.132.254:80 -s rr -p 120
 
//在新增的虚拟IP记录中新增两条real server记录，-g即为使用VS/DR模式
ipvsadm -a -t 192.168.132.254:80 -r 192.168.132.64:80 -g
 
ipvsadm -a -t 192.168.132.254:80 -r 192.168.132.68:80 -g
 
//启用LVS服务
ipvsadm
 



// 两台real server上的配置：

/*在回环设备上绑定了一个虚拟IP地址，并设定其子网掩码为255.255.255.255，与Director Server上的虚拟IP保持互通*/
ifconfig lo:0 192.168.132.254 broadcast 192.168.132.254 netmask 255.255.255.255 up

route add -host 192.168.132.254 dev lo:0

//禁用本机的ARP请求echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
```
之后在其他客户端机子上面，访问[http://192.168.132.254/](http://192.168.132.254/)，则可以看到结果了。

---
# 2 引用
LVS：https://www.cnblogs.com/Rivend/p/12071156.html