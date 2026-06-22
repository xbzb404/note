# BGP路由

概念

 BGP-EGP外部网关协议

 是IGP还是EGP由AS来决定，

AS一般由组织来分类

EGP以前是一个老路由协议，不过现在被取代了

AS号是一个区域范围的代称，只能指定一个AS号，因为一个AS号对应一个组织

IP 地址分公有和私有

OSPF不能在建邻居的同时不做对应链路的网络宣告，而BGP可以，建邻居是建邻居，发路由是发路由，使用起来非常灵活。



### 概述

BGP是一种实现自治系统之间的路由可达，并择优选路的**矢量性协议**。

BGP使用TCP作为传输层协议，



### 邻居建立

peer 1.1.1.1 as 100

分为 iBGP，eBGP



### 发布路由

​	network 192.168.1.0 24

​	import-route

​	上面两个发布路由方式，基本没有太大区别，只有选路时才有区别



​	什么时候路由是eBGP路由，什么是iBGP路由？

​	当路由两端是iBGP路由设备对等体时是iBGP，两端对等体是eBGP路由设备时是eBGP。





BGP的路径属性作用很多，可以防环、选路等等



现在常用的是mp-BGP



BGP只决定路由的控制平面

而路由最终的选路是转发平面，MPLS是转发平面的

MPLS的部分概念：

​	使用标签转发

​	根据标签转发表，识别标签就直接根据标签表转发，不需要修改报文头部大部分内容

​	需要转发需要标签表和路由，为什么需要路由，因为标签的标签分发协议是根据路由进行分发的

​	

协议栈

​	对应的协议栈传递对应的路由

​	ipv4

​	vpnv4

​	ipv6

​	evpn

​	I	PV4+字段 = VPNv4

​		IPV4

​	协议簇



### TCP 连接源地址





```
建立IBGP邻居
R1:
bgp 200
	peer 10.0.12.2 as 200

R2:
bgp 200
	peer 10.0.12.1 as 200
```

使用接口ip做邻居，当对应链路断开，直接断网

而使用回环口做邻居，会自动使用路由进行重新选路

所以常规情况都是使用IGP协议例如OSPF、ISIS协议把回环口打通后使用回环口建立IBGP邻居

而EBGP一般情况下使用直连进行建立邻居，不使用回环口

因为不需要另用协议，底层只有一条没有必要



underlay

​	回环口这种底层是underlay

​	ISIS OSPF 传递回环口路由

​	为了让BGP对等体关系建立

​	

overlay

​	BGP路由协议 传递 业务路由

​	使业务路由直接按能够互通



### BGP路由报文

#### Open报文

协商BGP对等体，建立对等关系。（协商并建立邻居）

**时刻**：BGP的TCP连接建立成功后

**关键字段**：

- Version：BGP版本号（目前为4）
- My AS：本地AS编号
- Hold Time：保持时间（默认180秒）
- BGP Identifier：本地Router ID
- Optional Parameters：可选参数（如认证、多协议扩展能力等）

#### Update报文

发送BGP路由更新

**时刻**：对等体关系建立后有路由需要发送时发送；或当路由发生变化（新增、撤销、属性改变）时向对等体发送增量更新

**关键字段**：

- Withdrawn Routes Length：撤销路由列表长度（0表示为更新路由）
- Withdrawn Routes：撤销路由列表
- Path Attributes：路径属性列表（AS_PATH、NEXT_HOP、LOCAL_PREF、MED等）（更新路由才有）
- NLRI（Network Layer Reachability Information）：可达路由前缀列表（更新路由才有）

#### Notification报文

当邻居建立出错，某一方放弃建立或断开，就会使用该报文报告错误信息，中止对等体关系，并且在该报文中说明原因。当收到该报文就会自动解除邻居关系。

**时刻**：检测到错误时发送（如报文格式错误、AS号不匹配、Hold Time超时等）；或设备主动断开对等体关系时发送（如管理员执行重置BGP命令、关闭接口等）

**关键字段**：

- Error Code：错误类型
  - 1 = Message Header Error（消息头错误）
  - 2 = OPEN Message Error（Open消息错误）
  - 3 = UPDATE Message Error（Update消息错误）
  - 4 = Hold Timer Expired（保持定时器超时）
  - 5 = FSM Error（状态机错误）
  - 6 = Cease（终止，含管理性关闭）
- Error Subcode：错误子码
  - Open消息错误常见子码：1=不支持的版本号、2=错误的AS号、3=错误的BGP ID、4=不支持的可选参数、5=认证失败、6=不支持的Hold Time
  - Update消息错误常见子码：1=非法属性列表、2=属性不可识别、3=属性缺失、4=属性标志错误、5=属性长度错误、6=无效的NEXT_HOP、8=错误的AS_PATH
  - Cease常见子码：1=最大前缀数超限、2=管理性关闭、3=对等体取消、4=管理性重置、5=连接拒绝
- Data：错误相关数据

#### Keepalive报文

标志对等体建立，维持BGP对等体关系（维护邻居）

**时刻**：对等体关系建立后周期性发送（默认间隔为Hold Time的1/3，通常60秒）；Open报文协商完成后立即发送首个Keepalive以确认邻居进入ESTABLISHED状态

**关键字段**：无（仅包含BGP报文头，无正文内容）

#### Route-refresh报文

用于在改变路由策略后请求对等体重新发送路由信息。只有支持路由刷新能力的BGP设备会发送和响应此报文。（唯一的用途就是"**请求**"对方重新发送一遍**全部**的路由信息。）

**时刻**：当路由策略发生变化时触发请求对等体重新通告路由；或管理员手动执行刷新命令时发送

**关键字段**：

- **AFI**（Address Family Identifier）：地址族标识
  - 1 = IPv4
  - 2 = IPv6
  - 25 = L2VPN
  - 16388 = VPNv4（通常用于MP-BGP）
- **SAFI**（Subsequent AFI）：子地址族标识
  - 1 = 单播（Unicast）
  - 2 = 组播（Multicast）
  - 4 = VPN（MPLS VPN）
  - 128 = VPN单播（VPNv4/VPNv6）
  - 129 = VPN组播

特殊点

当一条路由的路径属性值修改后，其不会周期性更新，有两种方法更新：

1.管理员在发送方使用命令触发告知邻居路由更新。（发送方使用Update报文）

2.管理员在接受方使用命令请求刷新路由。（请求方使用Route-refresh报文）







### **BGP状态机（6个状态）**

Idle —— 初始状态，准备建立TCP连接

Connect —— 等待TCP三次握手完成

Active —— TCP连接失败，主动重试（最常见的卡死点）

OpenSent —— TCP已通，已发Open报文，等待对方Open

OpenConfirm —— 双方Open都OK，等待对方Keepalive

Established —— 邻居UP，正式交换路由

------

**正常流转链路（带报文）**

Idle → （TCP三次握手成功） → Connect → （发送Open报文） → OpenSent → （收到对方Open，参数匹配，发送Keepalive） → OpenConfirm → （收到对方Keepalive） → Established

------

**异常流转链路（TCP建连失败）**

Connect → （TCP连接失败） → Active → （重试计时器超时，再次尝试TCP连接） → Connect

反复循环直到TCP成功，才进入正常链路

详细介绍

**1. Idle（空闲）**
这是初始状态。BGP拒绝所有进入的连接请求。

- **动作**：开始尝试建立TCP连接（目标端口179），并启动**ConnectRetry计时器**。
- **跳转**：TCP连接成功 -> 进入 **Connect**；连接失败 -> 进入 **Active**。

**2. Connect（连接）**
正在等待TCP连接完成。

- **动作**：如果ConnectRetry计时器超时，会重置计时器并再次尝试TCP连接。
- **跳转**：
  - TCP连接成功 -> 发送Open报文，进入 **OpenSent**。
  - TCP连接失败 -> 进入 **Active**（开始主动模式）。

**3. Active（活跃）**
这是BGP的“主动重试”状态。在此状态下，BGP会尝试与对端建立TCP连接。

- **动作**：不断尝试建立TCP连接。如果**ConnectRetry计时器超时**，会回到 **Connect** 状态重新发起连接。
- **常见问题**：如果设备长时间卡在 **Active** 状态，通常意味着TCP连接始终无法建立（例如IP不可达、端口被ACL过滤、对端未开启BGP）。

**4. OpenSent（已发送Open）**
TCP连接已建立，且本端已发送Open报文，等待对端回复Open报文。

- **动作**：收到对端的Open报文后，会检查其中的**版本号、AS号（自治系统号）、Hold Time（保持时间）**等参数是否兼容。
- **跳转**：
  - 参数兼容 -> 发送Keepalive报文，进入 **OpenConfirm**。
  - 参数不兼容 -> 发送Notification报文，退回 **Idle**。

**5. OpenConfirm（已确认Open）**
已收到对端兼容的Open报文，等待对端的Keepalive报文或Notification报文。

- **动作**：等待验证对方发送的Keepalive。
- **跳转**：
  - 收到Keepalive -> 进入 **Established**（邻居关系正式建立）。
  - 收到Notification 或 TCP拆链 -> 退回 **Idle**。

**6. Established（已建立）**
邻居关系成功建立，可以开始交换Update（路由更新）报文。

- **动作**：周期性发送Keepalive（默认60秒）维持连接。
- **异常**：如果Hold Time超时未收到Keepalive或Update，会认为邻居失效，退回 **Idle**。





iBGP建立抓包测试：

```
#R1：
bgp 100
	router-id 1.1.1.1
	peer 12.1.1.2 as-number 100 #as-number指定对方的as号
	
#R2:
bgp 100
	router-id 2.2.2.2
	peer 12.1.1.1 as-number 100


#请求路由刷新
refresh bgp all import
#模拟器会有bug，不一定会刷新，建议多发几条

#查看bgp路由
dis bgp routing-table
```

ipv4-family unicast 

如果没有ipv4的情况下，这个建议关闭，有，这个里面必须要有对应的ipv4地址的配置

undo default ipv4-unicast

不关，设置其他协议栈，例如：ipv6，vpnv4等都会自动额外创建ipv4的邻居

一般都是去bgp对应的协议栈视图里配置邻居，而不是在bgp全局视图下配置



实验注意：

水平分割：iBGP学习到的其他的iBGP的路由，不会把其路由传给其他iBGP路由

peer 1.1.1.1 