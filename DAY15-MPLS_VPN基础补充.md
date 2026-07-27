# DAY15：MPLS VPN 基础配置补充笔记

---

## **笔记速览** — 快速回顾核心知识点（详细内容见下文）

**RR反射VPNv4路由**：`undo policy vpn-target`（RR无VPN实例，需关闭RT检查）
**PE关闭IPv4协议栈**：`undo default ipv4-unicast`（避免与VPNv4冲突）
**查看VPN路由表**：`display ip routing-table vpn-instance VPNX`

**控制平面**：IGP+LDP（公网隧道）+ MP-BGP（私网标签+RT）+ VPN实例（隔离）
**转发平面**：压两层标签（外层LDP+内层MP-BGP）→ P设备交换外层 → PE剥外层看内层

**OSPF ↔ BGP互引关键**：Domain ID相同→还原内部路由（OSPF），不同→外部路由（O_ASE）
**修改Domain ID**：OSPF视图下 `domain-id <id>`（同一VPN需一致）

## 一、PE与RR的关键配置

### 1.1 RR（路由反射器）特殊配置

公网区域的RR因为没有VPN实例，必须配置：

```
undo policy vpn-target
```

> ⚠️ **原因**：RR不存储VPN实例路由表，默认会检查VPNv4路由的RT（Route Target）并进行过滤。配置`undo policy vpn-target`后，RR**反射VPNv4路由时不检查RT**，确保所有VPNv4路由都能被正常反射。

### 1.2 PE上的VPN实例配置

```
ip vpn-instance VPNX
    route-distinguisher 345:2          # RD：区分不同VPN实例的相同IP前缀
    vpn-target 100:10 export           # 出方向RT：路由发出时打上100:10
    vpn-target 100:20 import           # 入方向RT：只接收携带100:20的路由
```

**两边VPN实例的RT对应关系**：

```
PE1（VPNX）                          PE2（VPNX）
route-distinguisher 345:1            route-distinguisher 345:2
vpn-target 100:20 export    ←———→    vpn-target 100:20 import
vpn-target 100:10 import    ←———→    vpn-target 100:10 export
```

- **export**：路由发出时打上的"标签"（RT）
- **import**：只接收携带指定RT的路由

> **简化写法**（不推荐生产环境）：
> ```
> vpn-target 100:10 both    # 同时配置export和import为100:10
> ```
> ⚠️ **注意**：正常情况下应使用`export`/`import`分开配置实现路由的单向/双向隔离，`both`仅用于非重要场景或实验环境，生产环境不建议。

### 1.3 PE关闭默认IPv4协议栈

PE上因使用VPNv4地址族建立MP-BGP邻居，需要关闭默认IPv4协议栈：

```
undo default ipv4-unicast
```

> **原因**：BGP默认激活IPv4地址族，若不关闭，PE之间会尝试建立普通IPv4 BGP邻居，与VPNv4邻居冲突或产生多余会话。

### 1.4 查看VPN实例路由表

```
display ip routing-table vpn-instance VPNX
```

---

## 二、控制平面与转发平面

> **控制平面**：跑协议、算路由、分标签——决定"怎么走"
> **转发平面**：查表、压标签、转发报文——执行"直接走"

### 2.1 控制平面组件

| 组件                  | 作用                                                     |
| --------------------- | -------------------------------------------------------- |
| **IGP（OSPF/IS-IS）** | 公网路由学习，建立IP路由表                               |
| **LDP**               | 为**公网路由**分发外层标签，建立公网LSP隧道              |
| **MP-BGP**            | 为**VPN私网路由**分配内层标签，携带RT属性，交换VPNv4路由 |
| **VPN实例**           | 独立路由表，执行RT过滤，隔离不同VPN客户                  |

### 2.2 转发平面组件

| 组件                       | 作用                                            |
| -------------------------- | ----------------------------------------------- |
| **外层标签（LDP分配）**    | **公网隧道标签**，P设备凭此逐跳转发，不感知私网 |
| **内层标签（MP-BGP分配）** | **VPN私网标签**，出口PE凭此确定目标VPN实例      |
| **LFIB（标签转发表）**     | P设备维护，基于外层标签快速转发                 |

> **LDP管公网隧道，MP-BGP管私网标识，各司其职。**

### 2.3 MPLS VPN完整流程

**控制平面**：IGP建公网可达 → LDP为公网路由分标签 → MP-BGP为VPN私网路由分标签+RT → RT过滤导入VPN实例

**转发平面**：压两层标签（外层LDP + 内层MP-BGP）→ P设备只看外层转发 → 出口PE看内层确定VPN → 转发给CE

**数据流**：
```
CE → 入口PE（压外层LDP标签 + 内层MP-BGP标签）→ P设备（外层标签交换）→ 出口PE（剥外层、看内层确定VPN）→ CE
```

> **一句话总结**：LDP建公网隧道，MP-BGP区分私网VPN，控制平面建好，转发平面执行。

---

## 三、OSPF与BGP互引时的路由信息传递

**场景**：`Site1(OSPF) → CE1 → PE1(OSPF引入BGP) → 公网MP-BGP → PE2(BGP引入OSPF) → CE2 → Site2`

OSPF路由引入BGP时，原始路由信息（区域、路由类型）会丢失，对端PE将BGP路由引入OSPF时无法还原。BGP通过两种扩展团体属性解决：

- **Domain ID（域标识符）**：PE1引入时添加，PE2引入OSPF时比较。相同则还原路由类型，不同则视为外部路由。
- **Route Type**：携带Area-ID + OSPF路由类型，供PE2还原LSA类型。

**Route Type取值**：

| 值   | 含义       | LSA类型                        |
| ---- | ---------- | ------------------------------ |
| 1或2 | 区域内路由 | Type-1 / Type-2                |
| 3    | 区域间路由 | Type-3                         |
| 5    | 外部路由   | Type-5（Area-ID必须为0.0.0.0） |
| 7    | NSSA路由   | Type-7                         |

### 3.1 Domain ID默认规则

- 缺省Domain ID为**0（NULL）**。不同OSPF域若都使用NULL，无法区分，路由会被当作区域内路由。
- **任一OSPF域配置了非0的Domain ID后，NULL不再代表该域。**

### 3.2 PE2生成LSA的规则

| Domain ID比较结果 | Route Type | PE2生成的LSA                     |
| ----------------- | ---------- | -------------------------------- |
| 相同              | 1/2/3      | **Type-3 LSA**（区域间路由）     |
| 相同              | 5          | **Type-5 LSA**（外部路由）       |
| 相同              | 7          | **Type-7 LSA**（NSSA外部路由）   |
| **不同**          | 任意       | **Type-5/7 LSA**（视为外部路由） |

### 3.3 修改Domain ID配置（华为）

```
system-view
ospf process-id vpn-instance vpnname
domain-id <domain-id>
```

- `domain-id`：整数（如`1`）或点分十进制（如`0.0.0.1`）
- 缺省为 **NULL（0）**
- `undo domain-id` 恢复默认

> ⚠️ **注意**：同一VPN的所有OSPF进程须配置**相同**的Domain ID，否则路由会被当成外部路由（Type-5 LSA）。

### 3.4 路由表中的体现

**场景**：PE1和PE2的OSPF域Domain ID均配置为`0.0.0.1`（相同）。

**CE2上查看路由表**：
```text
<CE2> display ip routing-table

Destination/Mask    Proto   Pre   Cost   NextHop         Interface
10.1.1.0/24         OSPF    10    2      192.168.1.1     Gig0/0/1
```
→ 显示`OSPF`，表示生成Type-3 LSA（区域间路由），还原成功。

---

**场景**：PE2未配置Domain ID（保持NULL），与PE1的`0.0.0.1`不同。

**CE2上查看路由表**：
```text
<CE2> display ip routing-table

Destination/Mask    Proto   Pre   Cost   NextHop         Interface
10.1.1.0/24         O_ASE   150   1      192.168.1.1     Gig0/0/1
```
→ 显示`O_ASE`，表示生成Type-5 LSA（外部路由），视为外部路由。

---

**对比总结**：

| 场景          | CE2路由表显示 | LSA类型 | 协议类型     | 优先级 |
| ------------- | ------------- | ------- | ------------ | ------ |
| Domain ID相同 | **OSPF**      | Type-3  | OSPF内部路由 | 10     |
| Domain ID不同 | **O_ASE**     | Type-5  | OSPF外部路由 | 150    |

> **一句话**：Domain ID决定"是否同一域"，Route Type决定"具体路由类型"。路由表中直接看`Proto`列：`OSPF`表示还原成功，`O_ASE`表示被视为外部路由。

---

## 四、完整流程总结

```
MPLS VPN整体流程：
┌─────────────────────────────────────────────────────────────────────────┐
│ 控制平面：                                                             │
│ IGP建公网可达 → LDP分外层标签 → MP-BGP分内层标签+RT → RT过滤导入VPN实例 │
├─────────────────────────────────────────────────────────────────────────┤
│ 转发平面：                                                             │
│ CE→入口PE（压两层标签）→ P设备（外层标签交换）→ 出口PE（确定VPN）→ CE  │
├─────────────────────────────────────────────────────────────────────────┤
│ OSPF跨域互引：                                                         │
│ Domain ID相同 → 还原为OSPF内部路由；不同 → 视为外部路由                │
└─────────────────────────────────────────────────────────────────────────┘
```

**核心配置要点**：
1. RR需配置`undo policy vpn-target`
2. PE需配置`undo default ipv4-unicast`
3. VPN实例需正确配置RD和RT（export/import对应关系）
4. OSPF跨域需统一Domain ID