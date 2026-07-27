# DAY12-BGP核心知识-防环配置-路由选路

## 📋 当前笔记速览

- **BGP邻居配置要点** —— IBGP邻居必配参数及顺序  
- **IBGP防环机制** —— 路由反射器（RR）与联邦（Confederation）的原理与配置  
- **BGP选路规则** —— 11条优选规则（含口诀“漂亮老男人”）及负载分担  
- **五大关键属性** —— 实际中最常调优的属性及入向/出向优化策略  
- **配置速查** —— 常用选路策略命令模板  
- **故障排查命令** —— 查看路由未优选原因的方法  

---

## 一、BGP邻居配置要点

- **IBGP邻居**通常需设置：
  - `connect-interface LoopBack0` —— 使用环回口建立连接，提高稳定性
  - `next-hop-local` —— 将下一跳修改为本机地址，确保IBGP邻居能正确寻址
- 上述参数**必须**在 `peer <地址> as-number <AS号>` 邻居建立命令**之后**配置。

---

## 二、IBGP路由全网互传与防环

### 1. 路由反射器（RR）

**适用场景**：替代IBGP全互联，减少会话数量。

**关键路径属性（可选非过渡）** ：

| 属性            | 作用                      | 防环范围    |
| --------------- | ------------------------- | ----------- |
| `Originator_ID` | 记录路由始发者的Router-ID | 整个BGP网络 |
| `Cluster_List`  | 记录经过的RR簇ID          | RR簇内部    |

**配置模板**：
```
bgp 200
    group test internal                     # 创建内部对等体组
    peer test reflect-client                # 启用RR功能（组成员成为客户端）
    peer test connect-interface LoopBack0
    peer test next-hop-local
    peer 2.2.2.2 group test                 # 添加客户端
    reflector cluster-id 10.0.0.1           # 多RR时设置相同簇ID（冗余备份）
    peer test timer keepalive 30 hold 90
    peer test route-update-interval 0       # 取消更新间隔，快速收敛
```
> 多RR若配置相同`cluster-id`可实现备份；不同簇则需不同ID防环。

---

### 2. 联邦（BGP Confederation）

**适用场景**：将一个AS分割为多个成员AS，降低IBGP连接数。

**核心机制**：内部防环依靠`AS_Path`，成员AS号用括号标识，离开联邦时剥离。

**特殊AS_Path类型**：

| 类型                 | 示例                          | 说明                       |
| -------------------- | ----------------------------- | -------------------------- |
| `AS_Sequence`        | `300 200 100`                 | 有序记录（默认）           |
| `AS_Set`             | `300 {200 100}`               | 无序集合（路由聚合时使用） |
| `AS_Confed_Sequence` | `(65001 65002) 300 200`       | 联邦内部有序记录           |
| `AS_Confed_Set`      | `(65001 65002) 300 {200 100}` | 联邦内部无序集合           |

**跨成员AS建邻注意**：
- 使用环回口跨设备建立EBGP邻居时，需修改`ebgp-max-hop`（默认TTL=1不够）。
- 同时需配置`connect-interface`指定源接口，并确保路由可达。

**配置要点**：
```
bgp 64512
    confederation id 300              # 设置联邦AS号
    confederation peer-as 64513       # 指定其他成员AS（边界路由器需配置）
    peer 5.5.5.5 as 64513
    peer 5.5.5.5 ebgp-max-hop 2
```

---

## 三、BGP路由选路

### 选路前提
路由必须**有效**（Valid）才参与优选。

### 11条优选规则（口诀：**漂亮老男人**）

（面试才会背这个，太多了，正常不记忆，不过有个口诀：(漂亮老男人P L LAO MEN)

P(Pre_V) L(Local_P) L（Local）A(AS_Path) O（Origin） M（MED） E（EBGP） N（Next_hop））

| 规则 | 属性                        | 优选条件                                         | 适用范围                 |
| ---- | --------------------------- | ------------------------------------------------ | ------------------------ |
| 1    | `Preferred_Value`           | 越大越优                                         | **华为私有，仅本地有效** |
| 2    | `Local_Preference`          | 越大越优                                         | 仅IBGP内传播             |
| 3    | 本地始发路由                | 手工汇总 > 自动汇总 > network > import           | 本地始发                 |
| 4    | `AS_Path`                   | **越短越优**                                     | 公认必遵                 |
| 5    | `Origin`                    | IGP > EGP > Incomplete                           | 公认必遵                 |
| 6    | `MED`                       | **越小越优**                                     | 可选非过渡，跨AS传递     |
| 7    | 路由类型                    | EBGP > IBGP                                      | —                        |
| 8    | `Next_Hop`的IGP度量         | 度量值越小越优                                   | 依赖底层IGP              |
| 9    | `Cluster_List`              | 越短越优                                         | RR场景                   |
| 10   | `Router-ID / Originator_ID` | 最小者优（有`Originator_ID`就不考虑`Router-ID`） | —                        |
| 11   | Peer IP地址                 | 最小IP优                                         | 最终PK                   |

> 规则9-11属于“强制选路”，仅在前8条无法决出时启用。

### BGP选路规则属性作用说明

| 规则 | 属性                      | 作用说明                                                     |
| :--- | :------------------------ | :----------------------------------------------------------- |
| 1    | Preferred_Value           | 华为私有的本地管理员指定策略，仅本设备生效。用于在本地强制指定某条路径为最优，优先级最高。 |
| 2    | Local_Preference          | 在AS内传播，用于统一控制整个AS的出口流量方向。值越大，该出口越被优先选择。 |
| 3    | 本地始发路由              | 本地注入（network/import）的路由优于从邻居学来的路由，主要目的是防止路由环路。 |
| 4    | AS_Path                   | 路径越长，代表经过的AS越多，成本和延迟通常也越高。同时用于EBGP防环（收到含自身AS号的路由则丢弃）。 |
| 5    | Origin                    | 标识路由来源的可信度：IGP > EGP > Incomplete。network注入的可信度最高，重分发（import）最低。 |
| 6    | MED                       | 向外部AS传递，用于影响外部AS对本AS的入口路径选择。仅在来自同一AS的路径之间进行比较。 |
| 7    | 路由类型                  | EBGP路由优于IBGP路由，因为EBGP信息来源更直接。               |
| 8    | Next_Hop的IGP度量         | BGP提供下一跳IP，实际到达该IP的物理路径由IGP计算。IGP度量越小，该BGP路径越优。 |
| 9    | Cluster_List              | RR场景下，记录经过的RR集群。列表越长优先级越低，用于RR防环。 |
| 10   | Router-ID / Originator_ID | 当前面规则均无法区分时，选择Router-ID最小的路径。纯技术保底机制。 |
| 11   | Peer IP地址               | 极端情况下的最终保底，选择邻居IP较小的路径。                 |

### 负载分担

- 默认**不开启**。
- 当规则1-8完全相同（且AS_Path也完全一致）时可配置。
- 命令：`maximum load-balancing [ibgp|ebgp] <数量>`

---

## 四、必须记住的5个关键属性

| 优先级 | 属性                 | 优选方向 | 修改方式                   |
| ------ | -------------------- | -------- | -------------------------- |
| 1      | **Local_Pref**       | 越大越优 | 入方向（import）           |
| 2      | **AS_Path**          | 越短越优 | 出方向添加AS（加长则降权） |
| 3      | **MED**              | 越小越优 | 出方向（export）           |
| 4      | **EBGP > IBGP**      | EBGP更优 | 由架构决定                 |
| 5      | **Next_Hop IGP度量** | 越小越优 | 优化底层IGP                |

### 优化方向速查

```
入向优化（让外部访问我优先走此路径）      出向优化（我访问外部优先走此出口）
        ↓                                    ↓
    改 MED（减小）                       改 Local_Pref（增大）
    改 AS_Path（加长则降权）             开启负载分担
```

---

## 五、配置速查

```bash
# 1. 入向：让邻居优先走我（MED改小）
route-policy SET_MED permit node 10
    apply cost 50
bgp 200
    peer 1.1.1.1 route-policy SET_MED export

# 2. 出向：优先走某出口（Local_Pref改大）
route-policy SET_PREF permit node 10
    apply local-preference 200
bgp 200
    peer 2.2.2.2 route-policy SET_PREF import

# 3. 负载分担（规则1-8完全相同）
bgp 200
    maximum load-balancing 2
```

---

## 六、故障排查命令

```bash
display bgp routing-table <prefix>           # 查看路由详情
display bgp routing-table <prefix> verbose   # 查看选路过程（找"Best: No"原因）
```
> 常见未优选原因：下一跳不可达、AS环路、被策略过滤。

---

**总结**：BGP选路核心在于掌握**属性优先级**和**修改方向**，RR与联邦是IBGP规模化部署的关键防环技术，合理使用策略（入向改MED/出向改Local_Pref）可实现流量精准调度。