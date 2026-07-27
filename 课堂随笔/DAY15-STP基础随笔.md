# DAY15

stp是早期协议，新的都是使用RSTP，而MSTP可以兼容STP和RSTP

根桥：STP中作为STP树起点的设备

桥ID：早期的交换机被称为桥，或者网桥

​	每台运行STP协议的交换机都有一个唯一的桥ID

​	由16bit的桥优先级和48bit的MAC组成

​	并且16bit的桥优先级默认为32768，默认情况会比较MAC大小

STP接口cost值：

​	可以在交换机系统视图下使用stp pathcost-standard命令来修改cost

​	cost值非常重要，在STP对链接的计算中，cost用于计算交换机到根桥的开销值：RPC（从本机到根桥的所有cost之和）

接口ID：

​	接口

端口角色：

​	根端口：

​	指定端口：

​	阻塞端口：

STP的BPDU有：

 配置BPDU

TCNBPDU