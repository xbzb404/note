# DAY15-MPLS VPN基础配置随笔



公网区域的RR因为没有VPN实例，必须配置 `undo policy vpn-target` 才能正常反射VPNv4路由。这是MPLS VPN中RR的标准配置。



```
ip vpn-instance VPNX
	route-distinguisher 345:2
		vpn-target 100:10 export
		vpn-target 100:20 import
		
ip vpn-instance VPNX
	route-distinguisher 345:1
		vpn-target 100:20 export
		vpn-target 100:10 import
		#也可以两边直接配置，两边出入都配置为100：10也可以，但正常情况不允许这样做，除非是不重要的部分
		#vpn-target 100:10 both 
```



怎么查看vpnx的路由表

```
dis ip routing-table vpn-instance VPNX
```

PE的配置中因是使用vpnv4建立邻居，要记得配置把默认ipv4的协议栈关闭掉

```
undo default ipv4-unicast
```

## 控制平面与转发平面

**控制平面**：跑协议、算路由、分标签——决定"怎么走"
**转发平面**：查表、压标签、转发报文——执行"直接走"

### 控制平面组件

| 组件              | 作用                                                     |
| :---------------- | :------------------------------------------------------- |
| IGP（OSPF/IS-IS） | 公网路由学习，建立IP路由表                               |
| LDP               | 为**公网路由**分发外层标签，建立公网LSP隧道              |
| MP-BGP            | 为**VPN私网路由**分配内层标签，携带RT属性，交换VPNv4路由 |
| VPN实例           | 独立路由表，执行RT过滤，隔离不同VPN客户                  |

### 转发平面组件

| 组件                   | 作用                                            |
| :--------------------- | :---------------------------------------------- |
| 外层标签（LDP分配）    | **公网隧道标签**，P设备凭此逐跳转发，不感知私网 |
| 内层标签（MP-BGP分配） | **VPN私网标签**，出口PE凭此确定目标VPN实例      |
| LFIB（标签转发表）     | P设备维护，基于外层标签快速转发                 |

> **LDP管公网隧道，MP-BGP管私网标识，各司其职。**

### MPLS VPN 流程

| 阶段     | 做什么                                                       |
| :------- | :----------------------------------------------------------- |
| 控制平面 | IGP建公网可达 → LDP为公网路由分标签 → MP-BGP为VPN私网路由分标签+RT → RT过滤导入VPN实例 |
| 转发平面 | 压两层标签（外层LDP+内层MP-BGP）→ P设备只看外层转发 → PE看内层确定VPN → 转发给CE |

**数据流**：`CE → 入口PE（压外层LDP标签 + 内层MP-BGP标签）→ P设备（外层标签交换）→ 出口PE（剥外层、看内层确定VPN）→ CE`

> **一句话**：LDP建公网隧道，MP-BGP区分私网VPN，控制平面建好，转发平面执行。

## OSPF与BGP互引时的路由信息传递（小特性）

**场景**：`Site1(OSPF) → CE1 → PE1(OSPF引入BGP) → 公网MP-BGP → PE2(BGP引入OSPF) → CE2 → Site2`

OSPF路由引入BGP时，原始路由信息（区域、路由类型）会丢失，对端PE将BGP路由引入OSPF时无法还原。BGP通过两种扩展团体属性解决：

- **Domain ID**：域标识符。PE1引入时添加，PE2引入OSPF时比较。相同则还原路由类型，不同则视为外部路由。
- **Route Type**：携带Area-ID + OSPF路由类型，供PE2还原LSA类型。

**Route Type取值**：1或2=区域内（Type-1/Type-2 LSA）；3=区域间；5=外部路由（Type-5 LSA，Area-ID必须为0.0.0.0）；7=NSSA路由（Type-7 LSA）。

### Domain ID默认规则

- 缺省Domain ID为0（NULL）。不同OSPF域若都使用NULL，无法区分，路由会被当作区域内路由。
- 任一OSPF域配置了非0的Domain ID后，NULL不再代表该域。

### PE2生成LSA的规则

- Domain ID相同 + Route Type 1/2/3 → 生成Type-3 LSA（区域间路由）
- Domain ID相同 + Route Type 5/7 → 生成Type-5/7 LSA（外部路由）
- Domain ID不同 → 生成Type-5/7 LSA（视为外部路由）

**完整流程**：`OSPF→BGP`时PE1添加Domain ID和Route Type，由BGP Update携带；`BGP→OSPF`时PE2比较Domain ID，据此决定还原为区域间路由还是外部路由，生成对应LSA发布给CE2。

> **一句话**：Domain ID决定“是否同一域”，Route Type决定“具体路由类型”。

**修改 Domain ID 配置（华为设备）**

```
system-view
ospf process-id vpn-instance vpnname
domain-id <domain-id>
```

- `domain-id`：整数（如 `1`）或点分十进制（如 `0.0.0.1`）
- 缺省为 NULL（0）
- `undo domain-id` 恢复默认

> 同一VPN的所有OSPF进程须配相同Domain ID，否则路由被当成外部路由（Type-5 LSA）。

### 路由表中的体现

**场景**：PE1上OSPF域Domain ID配置为`0.0.0.1`，PE2上也配置为`0.0.0.1`（相同）。

**PE2将BGP路由引入OSPF后，在CE2上查看路由表**：

```text
<CE2> display ip routing-table

Destination/Mask    Proto   Pre   Cost   NextHop         Interface
10.1.1.0/24         OSPF    10    2      192.168.1.1     Gig0/0/1   ← Type-3 LSA（区域间路由）
```



如果Domain ID不同（PE2未配置，保持NULL）：

```text
<CE2> display ip routing-table

Destination/Mask    Proto   Pre   Cost   NextHop         Interface
10.1.1.0/24         O_ASE   150   1      192.168.1.1     Gig0/0/1   ← Type-5 LSA（外部路由）
```



**区别**：

| 场景          | CE2路由表显示     | LSA类型 | 协议         | 优先级 |
| :------------ | :---------------- | :------ | :----------- | :----- |
| Domain ID相同 | OSPF（区域内/间） | Type-3  | OSPF内部路由 | 10     |
| Domain ID不同 | O_ASE（外部路由） | Type-5  | OSPF外部路由 | 150    |

> **路由表中直接看`Proto`列：显示`OSPF`表示内部路由（还原成功），显示`O_ASE`表示外部路由（未识别为同一域）。**