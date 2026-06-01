# DAY4：VLink 、特殊区域实验

### 一、VLink

#### 1. VLink实验一：area 0被分割，使用虚连接将其连接

<img src="DAY4-VLink-特殊区域实验.assets/image-20260601214824120.png" alt="image-20260601214824120" style="zoom:200%;" />

AR1

```
interface GigabitEthernet0/0/0
 ip address 12.1.1.1 255.255.255.252 
interface LoopBack0
 ip address 1.1.1.1 255.255.255.255 
#
ospf 1 router-id 1.1.1.1 
 area 0.0.0.0 
  network 1.1.1.1 0.0.0.0 
  network 12.1.1.0 0.0.0.3 
```

AR2

```

interface GigabitEthernet0/0/0
 ip address 12.1.1.2 255.255.255.252 
interface GigabitEthernet0/0/1
 ip address 23.1.1.1 255.255.255.252 
interface LoopBack0
 ip address 2.2.2.2 255.255.255.255 
#
ospf 1 router-id 2.2.2.2 
 area 0.0.0.0 
  network 2.2.2.2 0.0.0.0 
  network 12.1.1.0 0.0.0.3 
 area 0.0.0.1 
  network 23.1.1.0 0.0.0.3 
  vlink-peer 3.3.3.3
```

AR3

```

interface GigabitEthernet0/0/0
 ip address 23.1.1.2 255.255.255.252 
interface GigabitEthernet0/0/1
 ip address 34.1.1.1 255.255.255.252 
interface LoopBack0
 ip address 3.3.3.3 255.255.255.255 
#
ospf 1 router-id 3.3.3.3 
 area 0.0.0.0 
  network 3.3.3.3 0.0.0.0 
  network 34.1.1.0 0.0.0.3 
 area 0.0.0.1 
  network 23.1.1.0 0.0.0.3 
  vlink-peer 2.2.2.2
```

AR4

```
interface GigabitEthernet0/0/0
 ip address 34.1.1.2 255.255.255.252 
interface LoopBack0
 ip address 4.4.4.4 255.255.255.255 
#
ospf 1 router-id 4.4.4.4 
 area 0.0.0.0 
  network 4.4.4.4 0.0.0.0 
  network 34.1.1.0 0.0.0.3 
```

结果

可以看到AR2-AR3互建邻居为FULL

![image-20260601215751989](DAY4-VLink-特殊区域实验.assets/image-20260601215751989.png)

![image-20260601215828369](DAY4-VLink-特殊区域实验.assets/image-20260601215828369.png)



#### 2.VLink实验二：区域2、3都需要Vlink到area0

![image-20260601192850517](DAY4-VLink-特殊区域实验.assets/image-20260601192850517.png)

![image-20260601223208801](DAY4-VLink-特殊区域实验.assets/image-20260601223208801.png)

AR1

```
interface GigabitEthernet0/0/0
 ip address 12.1.1.1 255.255.255.252 
interface LoopBack0
 ip address 1.1.1.1 255.255.255.255 
#
ospf 1 router-id 1.1.1.1 
 area 0.0.0.0 
  network 1.1.1.1 0.0.0.0 
  network 12.1.1.0 0.0.0.3 

```

AR2

```
interface GigabitEthernet0/0/0
 ip address 12.1.1.2 255.255.255.252 
interface GigabitEthernet0/0/1
 ip address 23.1.1.1 255.255.255.252 
interface LoopBack0
 ip address 2.2.2.2 255.255.255.255 
#
ospf 1 router-id 2.2.2.2 
 area 0.0.0.0 
  network 2.2.2.2 0.0.0.0 
  network 12.1.1.0 0.0.0.3 
 area 0.0.0.1 
  network 23.1.1.0 0.0.0.3 
  vlink-peer 3.3.3.3
```

AR3

```
interface GigabitEthernet0/0/0
 ip address 12.1.1.2 255.255.255.252 
interface GigabitEthernet0/0/1
 ip address 23.1.1.1 255.255.255.252 
interface LoopBack0
 ip address 2.2.2.2 255.255.255.255 
#
ospf 1 router-id 2.2.2.2 
 area 0.0.0.0 
  network 2.2.2.2 0.0.0.0 
  network 12.1.1.0 0.0.0.3 
 area 0.0.0.1 
  network 23.1.1.0 0.0.0.3 
  vlink-peer 3.3.3.3
```

AR4

```
interface GigabitEthernet0/0/0
 ip address 34.1.1.2 255.255.255.252 
interface GigabitEthernet0/0/1
 ip address 45.1.1.1 255.255.255.252 
interface LoopBack0
 ip address 4.4.4.4 255.255.255.255 
#
ospf 1 router-id 4.4.4.4 
 area 0.0.0.2 
  network 4.4.4.4 0.0.0.0 
  network 34.1.1.0 0.0.0.3 
  vlink-peer 3.3.3.3
 area 0.0.0.3 
  network 45.1.1.0 0.0.0.3 
```

AR5

```
interface GigabitEthernet0/0/0
 ip address 34.1.1.2 255.255.255.252 
interface GigabitEthernet0/0/1
 ip address 45.1.1.1 255.255.255.252 
interface LoopBack0
 ip address 4.4.4.4 255.255.255.255 
#
ospf 1 router-id 4.4.4.4 
 area 0.0.0.2 
  network 4.4.4.4 0.0.0.0 
  network 34.1.1.0 0.0.0.3 
  vlink-peer 3.3.3.3
 area 0.0.0.3 
  network 45.1.1.0 0.0.0.3 
```

结果

可以看到AR2-AR3，AR3-AR4互建邻居为FULL

![image-20260601222832842](DAY4-VLink-特殊区域实验.assets/image-20260601222832842.png)

![image-20260601222909492](DAY4-VLink-特殊区域实验.assets/image-20260601222909492.png)

![image-20260601222931948](DAY4-VLink-特殊区域实验.assets/image-20260601222931948.png)



### 二、特殊区域

#### 1.实验一：STUB

![image-20250816230050874](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902134912216.png)



AR1

```
interface GigabitEthernet0/0/0
 ip address 12.1.1.1 255.255.255.252 
interface LoopBack0
 ip address 1.1.1.1 255.255.255.255 
#
ospf 1 router-id 1.1.1.1 
 area 0.0.0.1 
  network 1.1.1.1 0.0.0.0 
  network 12.1.1.0 0.0.0.3 
```

AR2

```
interface GigabitEthernet0/0/0
 ip address 12.1.1.2 255.255.255.252 
interface GigabitEthernet0/0/1
 ip address 23.1.1.1 255.255.255.252 
interface LoopBack0
 ip address 2.2.2.2 255.255.255.255 
#
ospf 1 router-id 2.2.2.2 
 area 0.0.0.0 
  network 2.2.2.2 0.0.0.0 
  network 23.1.1.0 0.0.0.3 
 area 0.0.0.1 
  network 12.1.1.0 0.0.0.3 
```

AR3

```
interface GigabitEthernet0/0/0
 ip address 23.1.1.2 255.255.255.252 
interface GigabitEthernet0/0/1
 ip address 34.1.1.1 255.255.255.252 
interface LoopBack0
 ip address 3.3.3.3 255.255.255.255 
#
ospf 1 router-id 3.3.3.3 
 area 0.0.0.0 
  network 3.3.3.3 0.0.0.0 
  network 23.1.1.0 0.0.0.3 
 area 0.0.0.2 
  network 34.1.1.0 0.0.0.3 
  stub 
```

AR4

```
interface GigabitEthernet0/0/0
 ip address 34.1.1.2 255.255.255.252 
interface LoopBack0
 ip address 4.4.4.4 255.255.255.255 
#
ospf 1 router-id 4.4.4.4 
 area 0.0.0.2 
  network 4.4.4.4 0.0.0.0 
  network 34.1.1.0 0.0.0.3 
  stub 
```



可以看到有默认路由和路由明细

![image-20260601232016427](DAY4-VLink-特殊区域实验.assets/image-20260601232016427.png)

![image-20260601232500288](DAY4-VLink-特殊区域实验.assets/image-20260601232500288.png)

#### 2.实验二：Totally Stub

![image-20250816230050874](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902134912332.png)

AR1 - AR2 - AR3 - AR4 大部分和之前相同

AR3

```
ospf 1 router-id 3.3.3.3 
 area 0.0.0.0 
  network 3.3.3.3 0.0.0.0 
  network 23.1.1.0 0.0.0.3 
 area 0.0.0.2 
  network 34.1.1.0 0.0.0.3 
  stub no-summary
```

AR4

```
ospf 1 router-id 4.4.4.4 
 area 0.0.0.2 
  network 4.4.4.4 0.0.0.0 
  network 34.1.1.0 0.0.0.3 
  stub no-summary
```

可以看到有默认路由，路由明细没有了

![image-20260601233839241](DAY4-VLink-特殊区域实验.assets/image-20260601233839241.png)

![image-20260601233921736](DAY4-VLink-特殊区域实验.assets/image-20260601233921736.png)

#### 3.实验三：NSSA

![image-20250816225136103](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902134912393.png)

AR1 - AR2 - AR3 - AR4 大部分和之前相同

AR3

```
ospf 1 router-id 3.3.3.3 
 area 0.0.0.0 
  network 3.3.3.3 0.0.0.0 
  network 23.1.1.0 0.0.0.3 
 area 0.0.0.2 
  network 34.1.1.0 0.0.0.3 
  nssa
```

AR4

```
ospf 1 router-id 4.4.4.4 
 area 0.0.0.2 
  network 4.4.4.4 0.0.0.0 
  network 34.1.1.0 0.0.0.3 
  nssa
#
interface GigabitEthernet0/0/1
  ip address 45.1.1.1 255.255.255.252 
#
rip 1
 undo summary
 version 2
 network 45.0.0.0
```

AR5

```
interface GigabitEthernet0/0/0
 ip address 45.1.1.2 255.255.255.252 
#
interface LoopBack0
 ip address 5.5.5.5 255.255.255.255 
#
rip 1
 undo summary
 version 2
 network 5.0.0.0
 network 45.0.0.0
```

结果：可以看到所有OSPF路由可以看到出现了两个AS外部路由，并且Area 2内出现了LSA3的默认路由和路由明细、和NSSA类型的路由

AR3（ABR）在Area2（NSSA域）内发送LSA7类型的默认路由指向3.3.3.3，用来过滤外部路由，同时将LSA3的明细也在Area2内传播。AR4把外部路由引入为LSA7类路由，指向自己，交给AR3转换为LSA 5 向area 0传播。

AR2（ABR）会将area0中的LSA5在Area1中传播，并同时生成对应的LSA4，指向AR3（ABR）

![image-20260602002241220](DAY4-VLink-特殊区域实验.assets/image-20260602002241220.png)

![image-20260602002323348](DAY4-VLink-特殊区域实验.assets/image-20260602002323348.png)

![image-20260602000725090](DAY4-VLink-特殊区域实验.assets/image-20260602000725090.png)

![image-20260602000817654](DAY4-VLink-特殊区域实验.assets/image-20260602000817654.png)

![image-20260602002218438](DAY4-VLink-特殊区域实验.assets/image-20260602002218438.png)

#### 4.实验四：Totally NSSA

![image-20250816225136103](https://qyt-gujun.oss-cn-shanghai.aliyuncs.com/20250902134912393.png)

配置修改为nssa no-summary



结果可以看到Sum-Net 的LSA3（路由明细）没有了，其他的都相同

![image-20260602001655949](DAY4-VLink-特殊区域实验.assets/image-20260602001655949.png)

![image-20260602001730948](DAY4-VLink-特殊区域实验.assets/image-20260602001730948.png)