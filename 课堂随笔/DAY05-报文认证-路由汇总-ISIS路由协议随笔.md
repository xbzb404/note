# DAY5-报文认证-路由汇总-ISIS路由协议随笔

### 一、报文认证

​	OSPF支持两种报文认证，同时存在时，优先使用接口认证

​		区域认证方式：一个OSPF区域内认证模式和口令必须一致。（Area 一致）

​		接口认证方式：相邻的路由器直连接口的认证你是和口令必须一致。（接口直连一致）

​	认证方式：

​		md5

​		simple

​		keychain



​	如何配置接口认证？

```
int g 0/0/0 #进入接口
	ospf authentication-mode simple cipher 123456 
#	ospf 指定协议
#	authentication-mode 标识后面是认证类型
#   simple 不进行传输加密的密码
#   cipher 配置时密文保存口令 
#   123456 

```

​	如何配置区域认证？

```
ospf 1
	area 0
		authentication-mode md5 1 cipher 123123
#       authentication-mode md5  ①  cipher  ②
                          ↓         ↓
                       Key-ID=1    密钥密码123123
md5：认证类型
① 数字 1：Key-id 密钥编号（1~255，邻居必须同 ID）
② 123123：认证密钥
cipher：配置时密文保存口令
```

接口认证 > 区域认证



### 二、路由汇总

​	ospf 的路由汇总

​	将精细或明细路由汇聚之后被称为汇总路由或聚合路由。

​	ABR路由汇总：对区域之间的路由进行路由汇总。

​	ASBR路由汇总：对外部路由进行路由汇总。



如果设置回环口在配置为24为掩码后，在ospf中查看还是为32

是因为loop的ospf 网络类型默认为p2p，其发送的掩码为32

可以修改

```
int loop 0
	 ospf network-type broadcast
```



这个汇总需要自己手动配置

abr 区域间路由汇总

```
ospf 1 router-id 2.2.2.2 
 area 0.0.0.0 
  abr-summary 192.168.0.0 255.255.0.0
```



ASBR 外部路由汇总

```
ospf 1 router-id 3.3.3.3 
 asbr-summary 172.16.0.0 255.255.0.0
 import-route direct
```



### 三、静默接口

​	避免一些边缘设备的不必要的开销，比如边缘接入PC的三层交换机的接口，或者直连PC的接口

​	路由会正常宣告网段，但不会接受hello报文和发送报文

​	如何配置

```
ospf 1 #进入ospf视图
```





### ospf 总结

链路状态协议的核心思想：每个路由器维护整个网络的拓扑数据库（LSDB），通过计算最短路径树（SPT）来决定最佳路由。

SPT的建立



特殊区域Stub、NSSA



特殊节点 两域之间的 ABR（域边缘路由），和域外设备连接的 ASBR（自治系统边缘路由）



DR BDR 了解即可，用于减少OSPF内的广播网络内的冗余链路