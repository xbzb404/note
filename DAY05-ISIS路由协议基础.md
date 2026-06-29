# ISIS 路由协议基础



### 一、协议概述

- ISIS（中间系统到中间系统）：一种链路状态路由协议。
- 最初为 OSI 七层模型设计，使用 NSAP 地址。
- 现广泛用于运营商骨干网。

> ISIS 是链路状态协议，基于 OSI 模型，使用 NSAP 地址。

------

### 二、地址结构

#### 1. NSAP 地址（注意地址的数为16进制数）

- NSAP（网络服务接入点）：ISIS 使用的网络层地址。
- 长度可变：8 ~ 20 字节。
- 结构分为两大部分：
  - IDP（初始域部分）：标识地址所属的管理域。
  - DSP（域特定部分）：域内进一步细分。

> NSAP 地址长度 8~20 字节，分 IDP（管理域）和 DSP（域内细分）。

**各字段解释：**

- AFI（授权和格式标识符）：标识地址格式和管理机构，常见值：
  - 49：私有，最常用。
  - 47：国际代码指示符。
  - 39：ISO 数据国家代码。
  - 00：保留。
- IDI（初始域标识符）：标识管理域。
- HODSP（高层域特定部分）：进一步细分域结构。
- SID（系统标识符）：标识域内唯一设备。
- SEL（选择符）：指示上层协议，在 NET 中固定为 00，表示设备自身。

> AFI 决定地址类型（49=私有），SID 唯一标识设备，SEL 在 NET 中必须为 00。

**完整 NSAP 地址举例：**

text

```
47.0001.aaaa.bbbb.cccc.01 #16进制
```



逐段拆解：

- 47：AFI，表示国际代码指示符。
- 0001：IDI，标识管理域编号。
- aaaa：HODSP，高层域特定部分，进一步细分域。
- bbbb.cccc： SID，系统标识符，域内唯一标识设备。
- 01：SEL，选择符，此处为 01 表示某个上层协议实体，而非设备自身。

> SEL 不为 00 时，地址表示上层服务接入点，而非设备本身。

示例二：

text

```
49.0002.1111.2222.3333.00
```



逐段拆解：

- 49：AFI，私有，最常用。
- 0002：IDI，私有管理域编号。
- 1111：HODSP，高层域特定部分。
- 2222.3333：SID，系统标识符。
- 00：SEL，固定为 00，表示该地址是 NET（设备自身）。

> AFI=49 私有、SEL=00 设备自身，两个条件同时满足即为标准的 NET 地址。

#### 2. NET（网络实体标题）

- 全称：网络实体标题（Network Entity Title）。
- 定义：NSAP 的特例，SEL 固定为 00，用于标识设备。
- 格式：区域号 + System-ID（6 字节） + 00

> NET 是 SEL=00 的 NSAP 地址，格式：区域号 + 6 字节 System-ID + 00。

**示例：**

text

```
49.0001.aaaa.bbbb.cccc.00
```



- 49.0001：区域号，长度可变。
- aaaa.bbbb.cccc：System-ID，6 字节，全网唯一。
- 00：固定 SEL 值，判定该地址为 NET。

> System-ID 必须全网唯一，一台设备配多个 NET 时 System-ID 必须相同。

> 要点：一台设备可配置多个 NET，但 System-ID 必须一致。

#### 3. 配置 NET

text

```
isis 1
  network-entity 49.0001.0000.0000.0001.00
```



- isis 1：进入 ISIS 进程视图，进程号为 1。
- network-entity：配置网络实体标题。
- 49.0001.0000.0000.0001.00：完整 NET 地址。

> 通过 network-entity 命令一次性指定区域号、System-ID 并确认设备身份。

------

### 三、基本概念

#### 1. 区域划分

- ISIS 按设备划分区域，基于 Level 区分角色。
- 三种路由器角色：

Level-1 (L1)：非骨干常规路由器，只能与 L1 和 L1/2 建立邻居，类似 OSPF 普通区域路由器。

Level-2 (L2)：骨干路由器，只能与 L2 和 L1/2 建立邻居，类似 OSPF 骨干区域路由器。

Level-1-2 (L1/2)：连接 L1 和 L2，类似 OSPF 的 ABR。

> ISIS 区域边界在设备上（而非接口），路由器按 Level 分三种角色。

- 华为设备默认角色为 Level-1-2。
- L1/2 会在所有接口同时发送 L1 和 L2 的 Hello 报文，因此可以和 L1、L2 分别建立邻居。
- 注意：未规划角色时，默认 L1/2 会与同一台邻居建立两次邻居关系（L1 和 L2 各一次），造成资源浪费。

> 华为默认 L1/2，会与同一邻居建立两次邻居关系（L1+L2），需按网络规划调整角色。

- 非骨干区域规则：
  - 区域内设备必须在同一区域。
  - 域间路由只下发默认路由，类似 OSPF 的 Totally NSSA（完全末梢区域）。

> L1 区域域间只有默认路由，无明细，与 OSPF Totally NSSA 类似。

#### 2. 度量值（Cost）

- ISIS 默认 Cost = 10，与接口带宽无关。
- 两种度量风格：

> 默认 Cost=10，与带宽无关；默认 narrow 范围 1-63。

**窄度量（narrow）：**

- Cost 范围：1 - 63。
- 范围小，不适用于大网络。

**宽度量（wide）：**

- Cost 范围：1 - 16777214。
- 最大值 16777215 表示链路不参与 SPF 计算。

> wide 风格 Cost 可达 16777214，值为 16777215 的链路不参与 SPF。

**切换宽度量风格：**

text

```
isis 1
  cost-style wide
```



> 切换 wide 风格需在 ISIS 进程下全局配置，整网需统一。

**cost-style 参数解释：**

- compatible：兼容模式，同时发送和接收 narrow 与 wide。
- narrow：窄度量风格，范围 1-63。
- narrow-compatible：以 narrow 为主，兼容 wide。
- wide：宽度量风格，范围 1-16777214。
- wide-compatible：以 wide 为主，兼容 narrow。

> 过渡时期可用 compatible 风格保证互操作，最终整网统一为 wide。

**接口下设置 Cost：**

text

```
interface GigabitEthernet0/0/0
  isis cost 50
```



> Cost 在接口视图下手动设置，用于流量路径控制。

------

### 四、基础配置

**完整配置示例：**

text

```
isis 1
  network-entity 49.0001.0000.0000.0001.00
  cost-style wide
```



- isis 1：创建 ISIS 进程，进程号 1。
- network-entity：指定 NET 地址，同时定义了区域号和 System-ID。
- cost-style wide：启用宽度量，扩大 Cost 范围。

> 全局配置三要素：创建进程、配 NET、设 wide 风格。

text

```
interface GigabitEthernet0/0/0
  ip address 12.1.1.1 255.255.255.252
  isis enable
```



- interface GigabitEthernet0/0/0：进入物理接口。
- isis enable：在该接口上启用 ISIS，开始发送 Hello 报文。

> 接口启用 isis enable 后才发送 Hello 报文，建立邻居。

text

```
interface LoopBack0
  ip address 1.1.1.1 255.255.255.255
  isis enable
```



- Loopback 接口启用 ISIS 非常重要：
  - 其 IP 会作为路由器标识发布。
  - 通常与 System-ID 绑定，提高网络稳定性。

> Loopback 必须启用 ISIS，其 IP 作为设备标识发布，稳定性关键。

------

### 五、LSDB 查看

**查看命令：**

text

```
display isis lsdb verbose
```



- verbose：显示 LSP 详细信息。

> verbose 参数可查看 LSP 完整内容，定位 DIS 和分片信息。

**LSP ID 示例：**

text

```
49.0001.0000.0000.0001.01-00
```



- 49.0001.0000.0000.0001：产生该 LSP 的设备的 System-ID。
- 01：伪节点标识。非 00 表示该 LSP 由 DIS（指定中间系统）产生，类似 OSPF 的 DR。
- 00：分片号，表示第一个分片，用于承载超出单个 LSP 上限的信息。

> 末尾伪节点 ID 非 00 = DIS 产生；最后两位为分片号，00 为第一片。

------

### 六、OSPF 与 ISIS 概念对照

| 概念           | OSPF            | ISIS            |
| :------------- | :-------------- | :-------------- |
| 区域边界路由器 | ABR             | L1/2            |
| 骨干路由器     | Backbone Router | L2              |
| 指定路由器     | DR              | DIS             |
| 完全末梢区域   | Totally NSSA    | L1 区域默认行为 |
| 链路状态数据库 | LSDB            | LSDB            |

> OSPF ABR ↔ ISIS L1/2；DR ↔ DIS；L1 区域天然等同于 Totally NSSA。