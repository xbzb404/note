# OSPF详解

OSPF基础概念：区域、邻居、状态机、报文、泛洪

OSPF核心：LSA、IPv4 OSPFv2

​	Type1、2、3、4、5、7

​	描述网段/掩码（路由）

​	描述网络节点（绘制拓扑）

​	1.邻居

​	2.LSA 放到LSDB

​	3.每台路由器使用SPF计算SPT树

​		当前网络有哪些节点 区域

​		路由信息

​		Cost

​		节点--> ASBR

​	4.路由-->OSPF 路由表

​	5。OSPF路由表 --> IP routing

路由器 OSPF 计算SPT树的

​	

OSPF的非骨干区域的路由不需要得到太多明细路由，只需要几条指向区域内的默认路由即可规避掉，可以降低非骨干网络的边缘路由器。



OSPF中Router-ID是用来

​	标识唯一OSPF识别

​	选举DR/BDR(Router-ID大的路由器默认情况下优先作为DR)

Router-ID推荐先手动配置

```
ospf 1 router-id 1.1.1.1
```

不配置自动选择

​	1.会按Loopback接口IP地址最大的

​	2.没有LoopBack，选择物理接口（MAC）中地址最大的



配错了或想修改

```
<Huawei> system-view
[Huawei] ospf 1 router-id 2.2.2.2      # 在系统视图下执行
[Huawei] quit                           # 退出到用户视图
<Huawei> reset ospf 1 process           # 在用户视图下执行
```

> （写法类似IP但注意不是IP地址！）



OSPF 分

​	1.骨干区域

​	2.非骨干区域

​	3.特殊区域

​		Stub（末梢区域）

​		NSSA（非完全末梢区域）	



网络上传输的基本都是报文

​	OSPF有五种类型的报文

​	1.Hello （发现和维护邻居关系）（单播）

​	2.（DD/DbD）Database Description （交互链路状态数据库宅摘要）

​	3.Link State Request （请求特定的链路状态信息）

​	4.Link State Update （发送详细的链路状态信息）（组播，单播）

​	5.Link State Ack （确认LSA）

​	

这些报文是如何交互的？：

​	1.先通过Hello报文发现对方，然后建立邻居或邻接关系

​	2.建立邻接关系后，通过DD报文交换一下双方的链路状态数据库的摘要

​	3.发现有缺少的数据，向对方发出LSR报文请求缺少的链路状态信息

​	4.收到LSR报文请求后，通过发送LSU报文把详细的链路状态信息发出

​	5.收到后使用LSAck报文确认收到（使用LSAck和LSA区分）



OSPF报文头部各参数作用如下：

- **Version**：OSPF版本号，IPv4为2，用于版本匹配。
- **Type**：标识报文类型（1=Hello，2=DD，3=LSR，4=LSU，5=LSAck）。
- **Packet Length**：整个OSPF报文长度，用于接收端解析。
- **Router ID**：发送者的唯一标识，用于识别邻居身份。
- **Area ID**：所属区域，只有同区域才能建立邻居关系。
- **Checksum**：校验和，检测报文传输是否出错。
- **Auth Type**：认证类型（0=无，1=明文，2=MD5）。
- **Authentication**：认证数据，存储密码或MD5摘要。



 OSPF  网络中有4中网络类型：

​	1.广播类型

​	作用：自动发现邻居并选举DR/BDR，适用于以太网等支持广播的多路访问网络。

​	2.NBMA 类型（NonBroadcast Multi-Access）

​	作用：需手动指定邻居，不自动发现，适用于帧中继等不支持广播的多路访问网络（可以一对多但不能同时一对多的网络）。

​	3.点到多点 P2MP

​	将NBMA网络模拟为多条点到点链路，不需选举DR/BDR。

​	3.点到点 P2P （只有两台设备，没有其他）

​	无需选举DR/BDR，直接建立邻居，适用于PPP等串行链路。

​	特殊：只有P2P是以组播形式发送Hello报文（224.0.0.5）

多路访问网络：广播类型、NBMA 类型

​	

​	组播（1对组）：

​		A组，B组，C组，D组

​		224.0.0.5是OSPF所有路由的组播地址（OSPF组）

​		只要一台路由器的接口在OSPF中工作，该接口就会在其所在的广播域监听224.0.0.5这个地址

​	广播（1对全）：在一个广播域，指向全F（255）的，所有人都要查看。

​	单播（1对1）：发送的报文只对一个目标，其他的收到自动忽略。



### OSPF邻居与邻接关系

- **邻居关系**：通过Hello报文发现对方，参数一致即可建立，是互相信任的初始状态。
- **邻接关系**：邻居关系基础上，成功交换DD报文和LSA，完成LSDB同步后的最终状态。

------

### 邻居状态机（共8种）

> 从Down到2-way就已经建立了邻居状态，DR/BDR选举和邻居关系同步完成

1. **Down**：未检测到邻居，初始状态。
2. **Attempt**：仅NBMA网络，未收到Hello回复，仍轮询发送。
3. **Init**：收到Hello报文。
4. **2-way**：收到的Hello报文中包含自己的Router-ID。若无需形成邻接（如DRother之间），则停留在此状态；否则进入Exstart。
5. **Exstart**：协商主从关系，确定DD序列号。
6. **Exchange**：开始交换DD报文。
7. **Loading**：DD交换完成，请求并同步LSA。
8. **Full**：LSDB完全同步，邻接关系建立完成（原文未列出但逻辑完整补充）。

Down--> Attempt (仅NBMA，大多数情况直接跳过)--> Init--> 2-way--> (若无需建立邻接，如DRother，则停留2-way，结束)--> Exstart--> Exchange--> Loading--> Full

正常：1-3-4-5-6-7-8

DR：1-3-4

帧中继：1-2-3-4-5-6-7-8



提问：如何加速OSPF的的收敛速度？

​	1.减少Hello报文的间隔时间

​	2.端口修改为p2p网络



提问：如果OSPF路由之间链路未出现问题，但无法发出hello包，最短多久可以发现问题？

4s，因为hello包间隔最短为1s，Dead interval 为其4倍



OSPF默认探测机制使用Hello报文，但hello报文是秒级别，为了更好的网络质量，无感切换网络，会使用bfd（毫秒级的报文）



在建立邻居关系是，必须检查hello包中的参数：

​	比较重要的 Hello/Dead interval

​	Hello包正常10秒一次

​	Dead interval 为Hello的4倍  40秒



OSPF网络中指定DR/BDR可以避免全互联的邻接关系，减少LSA泛洪，提高网络效率



DR（指定路由器）：只有一个，负责和其他DROther建立邻接关系，代表该网络所有设备发送LSA，减少重复泛洪。

BDR（备份指定路由器）：只有一个，和DR一样，但不向其泛洪LSA。

DROthers（非DR/BDR路由器）：非DR/BDR的OSPF路由器



广播网络中：

​	1类LSA包含（本机接口IP，DR的接口IP）

​	2类LSA包含（Router-ID，掩码mask）

​	1+2就可以确定本机ip、网段、DR



DR和BDR的选举基于优先级和Router ID ：

​	1.先比较优先级（Priority） 高的为DR，次高的BDR （默认1，范围0-255）

​	2.优先级相同

​	3.如果priority设置为0，视为放弃选举DR/BDR

​	

DR也被称为伪节点：

​	1.简化拓扑计算：在SPF计算中，其他路由器只需要计算到伪节点的路径路径，而非所有邻居。

​	2.DR生成网络LSA（Type-2 LSA）。描述所有路由器，避免不必要的泛洪。



在ExStart状态下，本路由要和邻居路由互传空DD报文，

​	作用为：

​		1.确认主从关系，避免DD序列号冲突。

​		选举规则：比较双方的Router-ID，值更大的一方为Master

​		主：控制DD序列号分配。

​		从：响应Master的序列号。

​	比较接口MTU（可选，华为不开）



Exchange状态：

​	从会先行向主发送带有摘要的DD报文，序列号seq为主在ExStart状态下基于的seq。



Loading状态：

​	主从关系解除，开始正式互相检查LSDB信息是否缺失某些LSA，缺少就开始发送LSR用于请求LSA，对方回应LSU，本机收到后发送LSAck确认收到，直到同步完成后进入FULL状态。



FULL：完全同步完成，建立起完全邻接关系。



实验

```
拓扑
R1 --- R2
R1: 
	loopback0 1.1.1.1 32
	g0/0/0 12.1.1.1 30
R2:
	loopback0 2.2.2.2 32
	g0/0/0 12.1.1.2 30

	
R1:
interface LoopBack0
 ip address 1.1.1.1 32
interface GigabitEthernet0/0/0
 ip address 12.1.1.1 30
ospf 1 router-id 1.1.1.1 
 area 0.0.0.0 
  network 1.1.1.1 0.0.0.0 
  network 12.1.1.0 0.0.0.3

R2:
interface LoopBack0
 ip address 2.2.2.2 32
interface GigabitEthernet0/0/0
 ip address 12.1.1.2 30
ospf 1 router-id 2.2.2.2
 area 0.0.0.0 
  network 2.2.2.2 0.0.0.0 
  network 12.1.1.0 0.0.0.3


```



![image-20260527215855136](DAY02-OSPF详解随笔.assets/image-20260527215855136.png)