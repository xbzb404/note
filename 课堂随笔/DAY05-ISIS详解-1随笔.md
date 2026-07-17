# DAY5-ISIS详解随笔

ISIS（中间系统到中间系统）协议，是一种链路状态路由协议

ISIS原本是为了OSI七层模型创建的

使用NSAP地址

![image-20260603210228143](DAY05-ISIS详解-1随笔.assets/image-20260603210228143.png)

NSAP地址内，IDP和DSP都是可变长的，导致长度从8-20byte

AFI: 

​	49 最常用的代码 代表

​	47

​	39

​	00



IDI

HODSP

SID

SEL





NET网络实体名称

**全称：网络实体标题，NSAP 特例，SEL 固定 00**

格式：**区域 + System-ID (6 字节) + 00**

```
49.0001.aaaa.bbbb.cccc.00
#49.0001 区域号 长度可变
#aaaa.bbbb.cccc 设备ID 类似OSPF中的RID,可以是0000.0000.0001
#00 固定字段
```

- 49：私有 AFI
- System-ID：6 字节，全网唯一
- 末尾 00：判定为 NET

要点：一台设备多 NET，**System-ID 必须一致**。

ISIS如何配置NET地址

```
isis 1
	network-entity 49.0001.0000.0000.00001.00
```





ISIS的基本概念

1. 区域划分

​	和OSPF一样，要设置区域进行划分设备，避免广播域过大	

​	区域划分按照设备划分，并且按Level划分

​	Level 只有三种，设备也只有三种身份，Level 2，Level 1-2 ，Level 1

​	华为设备默认isis设备身份为Level 1-2

​	Level 2 为骨干路由

​	Level 1-2  连接level 1和2的设备，类似ABR

​	Level 1 为常规路由器

​	Level 1-2 会在所有接口上发送Level 1 和 Level 2的报文，就可以连接Level 1设备和其做邻居，和Level 2 也可以做邻居

​	一定要规划好设备的身份，不然按照华为默认配置会和相邻的设备建立两次邻居（Level 1 和Level 2）


​	非骨干区域内设备必须在同一区域

​	非骨干区域的域间路由只给默认路由，类似OSPF中的Totally NSSA

​	

​	

2.度量值

​	默认值：

​		默认cost为10，和接口带宽毫无关系

​		可以自行修改cost值，默认的cost宽度非常小，只有1-63，但可以通过命令设置为wide来修改其宽度

参数如下：

  compatible         Set cost style to compatible

  narrow             Set cost style to narrow
  narrow-compatible  Set cost style to narrow-compatible
  wide               Set cost style to wide
  wide-compatible    Set cost style to wide-compatible

```
[AR1-GigabitEthernet0/0/0]isis cost ?
  INTEGER<1-63>  Cost value
[AR1-GigabitEthernet0/0/0]isis 1
[AR1-isis-1]cost-style wide
[AR1-isis-1]int g0/0/0
[AR1-GigabitEthernet0/0/0]isis cost ?
  INTEGER<1-16777214>  Cost value
  maximum              The value is 16777215. This link will not be considered  
                       during the normal SPF computation.
```



3. 如何配置基础isis

```
isis 1 
	network-entity 49.0001.0000.0000.00001.00 # 配置NET地址，标识区域和设备身份
	cost-style wide # 
int g0/0/0
	ip add  12.1.1.1 30
	isis enable #在接口内应用ISIS
int loop 0
	ip add 1.1.1.1
	isis enable #在回环口上应用ISIS，非常重要，回环口在ISIS是一个很重要的存在
```





```
display isis lsdb verb
#查看isis的lsdb
#如果看到
49.0001.0000.0000.0001.01-00 
#重点就是后面这个01
#这个不是我们配置的，显示非0就是DR
```

