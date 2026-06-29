# DAY8：路由策略和策略路由-流量过滤（QoS-MQC）


## 一、filter-policy

### 关键特性

- **限制路由表条目进入**，但**无法限制LSA/LSP的发布**
- 即：邻居知道该路由存在，但本机路由表中没有
- **华为默认方向**：不指定时默认为 `export`（出方向）

```text
# 控制接收（本机不使用）
filter-policy route-policy Filter_Tag import

# 控制发布（邻居不学习）
filter-policy route-policy Filter_Tag export    # 默认
```


## 二、双点双向路由重分布

### 概念对比

| 类型         | 描述                                       | 风险                     |
| ------------ | ------------------------------------------ | ------------------------ |
| **单点双向** | 一台路由器连接两个路由域，双向重发布       | 无环路风险               |
| **双点双向** | 两台路由器都连接两个路由域，都做双向重发布 | **易产生环路和次优路径** |

### 次优路径问题（核心痛点）

```
OSPF外部路由优先级 = 150
IS-IS引入OSPF的外部路由优先级 = 15

问题：AR2通过IS-IS学到AR3引入的路由（优先级15）
      比通过OSPF直接学习（优先级150）更优
      → 选择了更远的次优路径
```

### 解决方案：打Tag + 过滤Tag

**原理**：在路由引入时打标记，在另一台边缘路由器引入时丢弃带标记的路由，防止环路和次优路径。

#### 配置示例

```text
# ===== 点1和点2均需配置 =====

# 1. 打Tag（引入时标记）
route-policy Set_Tag permit node 10
 apply tag 200

isis 1
 import-route ospf 1 route-policy Set_Tag

# 2. 过滤Tag（接收时丢弃）
route-policy Filter_Tag deny node 10
 if-match tag 200
route-policy Filter_Tag permit node 20

isis 1
 filter-policy route-policy Filter_Tag import   # 入方向过滤，本机不使用
```

**工作流程**：

```
OSPF路由 → 打Tag 200 → 引入IS-IS → 另一台收到 → 匹配Tag 200 → 丢弃 → 不进路由表
```


## 三、流量过滤-概念

### 什么是流量过滤
通过识别数据流的特征（源IP、目的IP、协议、端口等），决定**允许通过**、**拒绝通过**或**改变路径**。

### 两大流派

| 流派                         | 控制对象             | 核心工具                                             | 典型场景                         |
| ---------------------------- | -------------------- | ---------------------------------------------------- | -------------------------------- |
| **路由策略**（Route-Policy） | 路由条目（控制平面） | route-policy、filter-policy、tag                     | 路由引入、路由过滤、双点双向防环 |
| **策略路由**（PBR）          | 数据报文（转发平面） | traffic policy、traffic classifier、traffic behavior | 基于源地址选路、多出口负载分担   |


## 四、QoS-MQC

**MQC** = Modular QoS Command-Line Interface（模块化QoS命令行），是QoS的配置框架。

### 核心概念（策略路由PBR）

##### **策略路由（PBR）** = 基于策略的路由，**优先于路由表**进行路径选择。

| 对比项 | 传统路由       | 策略路由(PBR)                    |
| ------ | -------------- | -------------------------------- |
| 依据   | 目的IP查路由表 | 匹配条件（源IP、目的IP、协议等） |
| 优先级 | 普通           | **高于路由表**                   |
| 灵活性 | 低             | 高，可实现基于应用的选路         |

> **注意**：PBR指定下一跳后，仍需递归查路由表找到出接口因只指定下一跳，并未指定出接口。

### 策略路由PBR分类

根据生效范围不同，PBR分为两种：

| 类型                         | 生效对象                 | 生效位置                     | 典型场景                                 |
| :--------------------------- | :----------------------- | :--------------------------- | :--------------------------------------- |
| **接口PBR**（Interface PBR） | 通过该接口**转发的流量** | 接口的 inbound/outbound 方向 | 控制穿越设备的流量走向                   |
| **本地PBR**（Local PBR）     | **设备自身产生的流量**   | 全局（本地出方向）           | 控制本机发出的报文（如ping、SNMP响应等） |

#### 1️⃣ 接口PBR（Interface PBR）

**概念**：对**经过**设备的流量（即从一个接口进入、从另一个接口出去的流量）进行策略控制，决定其转发路径。

**生效位置**：接口的 **inbound**（入）或 **outbound**（出）方向。

**作用**：控制**穿越设备**的数据报文的走向。

**配置要点**：

- 调用 `traffic-policy` 时**必须指定方向**（inbound / outbound）
- 通常应用在**入方向（inbound）**，在流量进入设备时决定其下一跳

```
interface GigabitEthernet0/0/1
 traffic-policy p_redirect inbound   # 接口PBR，控制穿越流量，p_redirect是流策略
```



#### 2️⃣ 本地PBR（Local PBR）

**概念**：对**设备自身产生**的流量（即本机主动发出的数据包）进行策略控制，决定其选择哪个下一跳或出接口。

**生效位置**：全局生效，不绑定物理接口。

**作用**：控制**设备自身发出**的报文走向，不影响其他设备的流量。

**典型应用场景**：

- 本机ping检测走特定链路
- 本机SNMP Trap/Log报文强制走指定下一跳
- 本机路由协议报文（如OSPF Hello）的源地址/出接口控制

**配置方法**：
华为设备通过 **本地策略路由**（Local Policy-Based Routing）实现，在系统视图下配置：

```
# 1. 配置ACL（匹配本机产生的流量，如ping的ICMP报文）
acl number 3001
 rule 10 permit icmp source 10.0.0.1 0 destination 8.8.8.8 0

# 2. 创建策略路由（绑定ACL，指定下一跳）
policy-based-route PBR_Local permit node 10
 if-match acl 3001
 apply ip-address next-hop 192.168.1.254

# 3. 应用本地PBR（在系统视图下全局生效）
ip policy-based-route PBR_Local
```

**关键区别**：

- 本地PBR使用 `ip local policy-based-route p_name` 命令在**系统视图**下应用
- 接口PBR使用 `traffic-policy` 命令在**接口视图**下应用

#### 3️⃣ 接口PBR vs 本地PBR 对比总结

| 对比维度     | 接口PBR                                  | 本地PBR                              |
| :----------- | :--------------------------------------- | :----------------------------------- |
| **控制对象** | 穿越设备的流量                           | 设备自身发出的流量                   |
| **应用位置** | 接口视图                                 | 系统视图                             |
| **关键命令** | `traffic-policy p_name inbound/outbound` | `ip local policy-based-route p_name` |
| **方向选择** | 必须指定 inbound/outbound                | 无需指定方向（仅影响本地出方向）     |
| **典型场景** | 控制用户上网流量走哪条链路               | 控制本机ping/SNMP/日志走指定链路     |

> **补充一句话**：**接口PBR管"过路"流量，本地PBR管"自产"流量。**

### MQC三要素配置模型

#### 1️⃣ 流分类（Classifier）— "挑出来"

```text
traffic classifier c_192
 if-match source-ip 192.168.1.0 24
```

**常用匹配条件**：

```text
if-match source-ip x.x.x.x mask       # 源IP
if-match destination-ip x.x.x.x mask  # 目的IP
if-match protocol tcp                  # 协议
if-match tcp-port destination-port 80 # 端口
if-match vlan-id 100                  # VLAN
```

#### 2️⃣ 流行为（Behavior）— "怎么管"

```text
traffic behavior b_redirect
 redirect ip-nexthop 10.0.0.1
```

**常用动作**：

```text
permit                       # 允许
deny                         # 拒绝
car cir 1000                # 限速
redirect ip-nexthop x.x.x.x # 重定向下一跳
remark dscp 46              # 重标记优先级
```

#### 3️⃣ 流策略（Policy）— "绑起来"

```text
traffic policy p_redirect
 classifier c_192 behavior b_redirect
```

#### 4️⃣ 应用接口 — "贴上去"

```text
interface GigabitEthernet0/0/1
 traffic-policy p_redirect inbound
```


### 完整配置示例

**场景**：将来自 192.168.1.0/24 的流量重定向到 10.0.0.1

```text
# 1. 配置ACL匹配需要重定向的流量
acl number 3000
 rule 10 permit ip source 192.168.1.0 0.0.0.255

# 2. 创建流分类（匹配ACL）
traffic classifier c1
 if-match acl 3000

# 3. 创建流行为（指定重定向下一跳）
traffic behavior b1
 redirect ip-nexthop 10.0.0.1

# 4. 创建流策略（绑定分类和行为）
traffic policy p1
 classifier c1 behavior b1

# 5. 应用到接口入方向
interface GigabitEthernet0/0/1
 traffic-policy p1 inbound   # 接口PBR的标准调用命令
```


### 入方向 vs 出方向判断

| 方向               | 判断标准                     |
| ------------------ | ---------------------------- |
| **inbound（入）**  | 数据**进入**该接口时执行策略 |
| **outbound（出）** | 数据**离开**该接口时执行策略 |

> **口诀**：数据从该接口进来 → `import` / `inbound`；数据从该接口出去 → `export` / `outbound`


### 验证命令

```text
display traffic classifier user-defined   # 查看流分类
display traffic behavior user-defined     # 查看流行为
display traffic policy user-defined       # 查看流策略
display qos policy interface              # 查看接口执行情况
```


### 删除配置（顺序不可乱）

```text
interface GigabitEthernet0/0/1
 undo traffic-policy p_redirect inbound    # 1. 先从接口移除
undo traffic policy p_redirect             # 2. 删策略
undo traffic behavior b_redirect           # 3. 删行为
undo traffic classifier c_192              # 4. 最后删分类
```


## 五、常见踩坑点

| 坑点                     | 正确理解                             |
| ------------------------ | ------------------------------------ |
| 以为流分类=流策略        | 流分类只"挑"，流策略才是完整规则     |
| 只配分类和行为，忘配策略 | 不生效                               |
| 策略顺序搞反             | 按序号从小到大匹配，**序号小的优先** |
| 接口方向弄错             | inbound/outbound效果完全不同         |
| filter-policy不写方向    | 华为默认export，需明确意图           |


## 六、配置顺序总览

```
路由策略：打Tag → 引入 → 过滤Tag → 应用filter-policy
策略路由（MQC）：分类 → 行为 → 策略 → 贴接口（顺序不可乱！）
```


## 七、两大流派对比总结

| 对比维度       | 路由策略（Route-Policy）    | 策略路由（PBR）                           |
| -------------- | --------------------------- | ----------------------------------------- |
| **操作对象**   | 路由条目（LSA/LSP）         | 数据报文                                  |
| **生效平面**   | 控制平面                    | 转发平面                                  |
| **执行优先级** | 路由计算时                  | **高于路由表**                            |
| **核心工具**   | route-policy、filter-policy | MQC（traffic classifier/behavior/policy） |
| **典型用途**   | 路由引入过滤、双点双向防环  | 基于源地址选路、多出口负载                |
| **方向关键字** | import / export             | inbound / outbound                        |


> **一句话总结**：`filter-policy` 控制路由表学习，PBR（通过MQC实现）控制数据包转发路径；打Tag+过滤Tag 解决双点双向路由回馈，MQC三要素按 **分类→行为→策略→贴接口** 顺序配置。