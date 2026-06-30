# DAY11：BGP路由基础（路径属性·汇总·复位·防环）

------

## 当前笔记速览

| 章节             | 核心内容                 | 关键命令/概念                                                |
| :--------------- | :----------------------- | :----------------------------------------------------------- |
| **一、路径属性** | 8种BGP属性详解           | AS_Path（选路/防环）、Origin（路由来源）、Next_Hop（路由下一跳）、Local_Preference（越大越优，仅IBGP）、MED（越小越优，EBGP）、Community（打标/过滤）、Atomic_Aggregate（警告）、Aggregator（溯源） |
| **二、路由汇总** | 自动汇总 vs 手动汇总     | `summary automatic`（仅import-route，自然主类汇总）；`aggregate` + `as-set`（保留AS_Path）、`suppress-policy`（部分抑制）、`origin-policy`（共生共灭）、`attribute-policy`（改属性） |
| **三、BGP复位**  | 策略修改后触发重新通告   | 硬复位 `reset`（TCP重建，慎用）；软复位 `refresh`（TCP保持，推荐） |
| **四、IBGP防环** | 路由反射器（RR）防环机制 | "非非不传"；`Originator_ID`（跨簇防环）；`Cluster_List`（簇内防环） |

------

## 一、BGP常见路径属性

### 1. AS_Path（公认必遵）

是一个有序列表，用于标记经过的AS区域号。

**作用**：防环 + BGP选路（路径越短越优）。

**示例**：`AS_Path: 200 100` → 100是始发AS号，200是最近经过的AS。

**详见DAY10笔记。**

### 2. Origin（公认必遵）

标记路由来源，有三种：

| 标记 | 类型       | 产生方式                             |
| :--- | :--------- | :----------------------------------- |
| `i`  | IGP        | network命令注入 或 aggregate聚合生成 |
| `e`  | EGP        | 通过EGP协议引入（现已几乎不用）      |
| `?`  | incomplete | import-route引入的直连/静态/IGP路由  |

### 3. Next_Hop（公认必遵）

指定到达目标网络的下一跳地址。

- 向**EBGP**对等体发布路由时：修改下一跳为**本端与对端建立邻居的接口地址**。
- 向**IBGP**对等体发布路由时：修改下一跳为**本端与对端建立邻居的接口地址**。
- 从EBGP学习到路由后向IBGP发布时：**不修改** Next_Hop 值。

### 4. Local_Preference（公认自愿）

用于AS内选路，告知路由器哪条路径是离开AS的首选路径。

- **值越大越优先**，缺省值为**100**。
- **仅在IBGP对等体之间传递**，不会发给EBGP对等体。
- 可在边界路由器的 **import** 方向使用路由策略修改该值。

### 5. MED（可选非过渡）

用于EBGP之间，当存在多条指向同一网段的路由且其他条件相同时，MED决定选路。

- **值越小越优先**（类似Cost值）。
- 缺省情况下，**仅比较来自同一相邻AS**的路由的MED值。可通过 `compare-different-as-med` 命令强制比较不同AS的路由。
- MED默认值：继承IBGP内部的IGP Cost（如OSPF的Cost）；直连/静态路由为0；从其他EBGP学来的路由**无MED**（默认不可传递）。

### 6. Community（可选过渡）

用于精细化路由过滤，可给路由打标记，无需匹配网络前缀即可执行策略。

**属性长度**：32bit

**两种格式**：

- 十进制整数
- `AA:NN`（AA为AS号，NN为自定义编号）

**传递前提**：必须在邻居上配置 `peer <邻居地址> advertise-community`，否则缺省不传播该属性。

**默认团体属性**：

| 名称                    | 值                      | 作用                                 |
| :---------------------- | :---------------------- | :----------------------------------- |
| **Internet**            | 0 (0x00000000)          | 缺省属性，可向任何BGP对等体发布      |
| **No_Advertise**        | 4294967042 (0xFFFFFFFE) | **不向任何**BGP对等体发布            |
| **No_Export**           | 4294967041 (0xFFFFFFFD) | **不向AS外**发布                     |
| **No_Export_Subconfed** | 4294967043 (0xFFFFFFFB) | 不向AS外发布，也不向AS内其他子AS发布 |

**操作示例**：

（1）单个路由打标签（不常用）

```bash
route-policy comm1 permit node 10 
 apply community 100:2

bgp 100
 network 172.16.1.0 24 router-policy comm1
```



（2）通过import方向批量打标签（常用）

```bash
ip ip-prefix comm1 permit 172.16.1.0 24

route-policy comm1 permit node 10 
 if-match ip-prefix comm1
 apply community 100:2

bgp 100
 import-route direct route-policy comm1
```



（3）使用community-filter过滤标签

```bash
# 仅放行带100:2标签的路由，其他拒绝
ip community-filter FromR1 permit 100:2

route-policy FromR1 deny node 10
 if-match community-filter FromR1
route-policy FromR1 permit node 20

bgp 200
 peer 12.1.1.1 route-policy FromR1 import
```



（4）追加标签（不覆盖原有）

```bash
ip community-filter Com1 permit 100:1

route-policy com1 permit node 10
 if-match community-filter Com1
 apply community no-export additive   # 必须加additive，否则覆盖原标签

bgp 200
 peer 12.1.1.1 route-policy ToR3 import
```



（5）直接使用filter-policy过滤（不用route-policy）

```bash
ip ip-prefix FromR1-Prefix deny 192.168.1.0 24
ip ip-prefix FromR1-Prefix permit 0.0.0.0 0 less-equal 32

bgp 200
 peer 12.1.1.1 filter-policy ip-prefix FromR1-Prefix import
```



### 7. Atomic_Aggregate（公认自愿）

用于**出错检查**。

- 是一个**警告标识**，告知"该路由是被聚合的路由，AS_Path路径可能不完整"。
- 不携带具体信息，只是一个标志位，要么有（表示存在路由故障隐患），要么没有。

常见故障场景：聚合路由导致AS_Path缺失，例如聚合前为 `100 200`，聚合后变为 `300 {200 100}`，因无序列表导致不知始发路由，200可能学习回去造成环路。

### 8. Aggregator（可选过渡）

用于**出错检查与溯源**。

- 记录了**谁执行了这次路由聚合操作**。
- 包含两个字段：
  1. 执行聚合的路由器的**AS号**
  2. 执行聚合的路由器的**BGP Router ID**

通常与 `Atomic_Aggregate` 一同出现，便于管理员溯源定位问题。

------

## 二、路由汇总

### 1. 自动汇总（summary automatic）

bash

```
bgp 100
 ipv4-family unicast
  summary automatic
```



**作用**：将 import-route 引入的外部路由（直连/静态/IGP）按**自然主类网段**自动汇总，并自动抑制被汇总的明细路由（标记为 `s>`）。

**关键特性**：

- 仅对 **import-route** 引入的路由生效，对 network 宣告无效。
- 汇总成**自然主类网段**，无法自定义。例如：`10.1.1.0/24` 和 `10.2.1.0/24` → `10.0.0.0/8`；`172.16.1.0/24` 和 `172.17.1.0/24` → `172.16.0.0/16`
- 被汇总的明细路由下一跳指向 `127.0.0.1`。
- 仅支持IPv4单播地址族，IPv6不支持。
- 手动聚合优先级高于自动聚合。

### 2. 手动汇总（aggregate）

（1）基本汇总（不抑制明细）

```bash
bgp 100
 aggregate 172.16.0.0 16
```

**说明**：发布汇总路由，同时明细路由仍继续对外发布，两者共存。

（2）抑制所有明细路由

```bash
bgp 100
 aggregate 172.16.0.0 16 detail-suppressed
```

**说明**：发布汇总路由，所有被汇总的明细路由均被抑制（标记为 `s>`），不再对外发布。

（3）抑制所有明细 + 保留原始AS_Path

```bash
bgp 100
 aggregate 172.16.0.0 16 detail-suppressed as-set
```

**说明**：发布汇总路由，抑制所有明细，同时通过 `as-set` 保留明细路由的原始AS_Path信息，防止因AS_Path丢失引发环路。

（4）仅抑制指定明细路由

```bash
ip ip-prefix yiZhi permit 172.16.1.0 24
route-policy yiZhi permit node 10
 if-match ip-prefix yiZhi

bgp 100
 aggregate 172.16.0.0 16 as-set detail-suppressed suppress-policy yiZhi
```

**说明**：发布汇总路由，保留原始AS_Path，仅抑制由 `suppress-policy` 匹配到的明细路由（如 `172.16.1.0/24`），其他明细仍正常发布。

（5）汇总路由与指定明细联动抑制（origin-policy）

```bash
ip ip-prefix test1 permit 172.16.3.0 24
route-policy test1 permit node 10
 if-match ip-prefix test1

bgp 65001
 aggregate 172.16.0.0 255.255.0.0 origin-policy test1
```

**说明**：汇总路由的发布与 `origin-policy` 匹配的明细路由**联动**——若该明细路由消失，汇总路由也随之撤消。

（6）修改汇总路由的路径属性（attribute-policy）

```bash
route-policy test2 permit node 10
 apply cost 200

bgp 100
 aggregate 172.16.0.0 16 detail-suppressed attribute-policy test2
```

**说明**：发布汇总路由（抑制明细），并通过 `attribute-policy` 修改汇总路由的属性，例如将MED修改为200。

------

## 三、BGP复位

BGP不是周期性更新协议，修改路径属性等策略后不会自动触发更新，需手动复位。

### 1. 硬复位（Hard Reset）

**特点**：拆除并重建TCP会话，导致邻居中断、路由震荡，生产环境慎用。

**命令汇总**：

| 命令                               | 作用范围                                    |
| :--------------------------------- | :------------------------------------------ |
| `reset bgp all`                    | 复位所有BGP连接                             |
| `reset bgp internal`               | 复位所有IBGP连接                            |
| `reset bgp external`               | 复位所有EBGP连接                            |
| `reset bgp as-number`              | 复位与指定AS的BGP连接                       |
| `reset bgp peer-address`           | 复位与指定对等体的BGP连接（默认单播地址族） |
| `reset bgp peer-address multicast` | 复位与指定对等体的IPv4组播地址族连接        |

### 2. 软复位（Soft Reset）

**特点**：不中断TCP会话，平滑重新通告路由，对业务影响小，推荐使用。

**命令汇总**：

| 命令                                               | 作用范围                   | 方向      |
| :------------------------------------------------- | :------------------------- | :-------- |
| `refresh bgp all import`                           | 所有BGP连接                | 入方向    |
| `refresh bgp all export`                           | 所有BGP连接                | 出方向    |
| `refresh bgp internal import/export`               | 所有IBGP连接               | 入/出方向 |
| `refresh bgp external import/export`               | 所有EBGP连接               | 入/出方向 |
| `refresh bgp peer-address multicast import/export` | 指定对等体的IPv4组播地址族 | 入/出方向 |

**方向选择**：

| 方向       | 适用场景                                                     |
| :--------- | :----------------------------------------------------------- |
| **import** | 修改了入方向策略（route-policy import / filter-policy import）后，需重新处理从邻居收到的路由 |
| **export** | 修改了出方向策略（route-policy export / filter-policy export）后，需重新处理发送给邻居的路由 |

------

## 四、IBGP防环

IBGP水平分割要求从IBGP邻居学到的路由不发给其他IBGP邻居，为避免全互联的性能开销，采用**路由反射器（RR）**或**联邦**解决。

### 路由反射器（RR）

- RR和其Client组成**路由反射簇（Cluster）**。
- 关键规则：**非非不传**——非Client之间不传递路由。
- 防环依赖两个属性（均为**可选非过渡**类型）：

**（1）Originator_ID（始发者ID）**

- **作用**：防止路由在RR之间反射时出现跨簇环路。
- **原理**：RR反射路由时，在路由上记录始发者的**Router-ID**。若某路由器收到IBGP路由后发现Originator_ID等于自己的Router-ID，说明"这条路由是我产生的，转了一圈又回来了"，于是**忽略该路由**。
- **通俗理解**：快递包裹上的发件人标签，转了一圈回到发件人手里，发件人拒收。

**（2）Cluster_List（簇列表）**

- **作用**：防止同一RR簇内出现路由环路。
- **原理**：每个RR簇有**Cluster-ID**（默认等于RR的Router-ID）。RR反射路由时将自己的Cluster-ID加入路由的Cluster_List。若RR收到路由后发现Cluster_List已有自己的Cluster-ID，说明"这是我之前反射出去的，转了一圈又回来了"，于是**忽略该路由**。
- **通俗理解**：打卡记录，每个经过的簇盖个章，看到有自己的章就拒收。

### 总结

| 属性              | 作用范围                 | 通俗理解                         |
| :---------------- | :----------------------- | :------------------------------- |
| **Originator_ID** | 防止**跨RR簇**的环路     | 发件人标签，看到是自己发的就丢弃 |
| **Cluster_List**  | 防止**同一RR簇内**的环路 | 打卡记录，看到有自己的章就丢弃   |