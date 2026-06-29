# DAY4: OSPF-VLink-特殊区域详解

## 1. VLink（虚链路）

### 作用
解决跨设备连接区域的问题

### 使用场景
- 连接被隔开的 Area 0：`Area 0 — Area 1(VLink) — Area 0`
- 连接被隔开的子区域 Area 2：`Area 0 — Area 1(VLink) — Area 2`

### 关键概念
- **Transit Area**：承载VLink的区域（如上面的Area 1）
- 限制：Transit Area **不能是** Stub 或 NSSA
- 传路由：使用 **LSA 3**
- 本质：类似VPN机制，**不可靠，仅作临时方案**，长期绝对不能使用

### 配置（华为）

```
ospf 1
 area 1                        # 进入传输区域
  vlink-peer 2.2.2.2           # 指向对端Router ID
```

> 两端都要配，互相指向对方Router ID

### 查看

```
display ospf vlink
```

> 看 **State** 是否为 **Full**（Full = 连接成功）

---

## 2. 外部LSA

### LSA 5（AS External LSA）
由 **ASBR** 产生，宣告外部网络（直连外部网络的路由器）

- LS Type：5
- Link State ID：外部网络的网络地址（目的地）
- Advertising Router：ASBR的Router ID
- Network Mask：外部路由的掩码
- metric：ASBR到外部路由的开销

### LSA 4（ASBR-Summary LSA）
由 **ABR** 产生，告诉其他区域“ASBR在哪里”（区域间路由，告知如何到达ASBR）

- LS Type：4
- Link State ID：ASBR的Router ID（找谁）
- Advertising Router：ABR的Router ID（谁在指路）
- Network Mask：仅保留，无意义
- metric：ABR到ASBR的开销

> LSA 4 和 LSA 5 相辅相成，**有5才会有4**

### 外部路由引入命令

```
ospf 1
 import-route static          # 引入静态路由
 import-route direct          # 引入直连路由
```

带参数示例：

```
ospf 1
 import-route static cost 10 type 1
```

- `cost 10`：设置开销
- `type`：外部路由类型（1或2，缺省为2）

**Type-2（缺省）**：只计算ASBR到外部路由的开销，不累加内部路径。内部结构简单、不需要选路时使用。开销和自身路由没有可比性，内部所有路径等价，无法选路。

**Type-1**：总开销 = 内部开销（到ASBR）+ 外部开销（ASBR到目标）。内网拓扑复杂、需要选路时使用。

> Type-1 优先级永远高于 Type-2

### 发布默认路由

```
ospf 1
 default-route-advertise always
```

### 查看结果

```
display ospf lsdb ase
display ospf routing
```

### 优先级
- 域间路由：10
- 域外路由：150

---

## 3. 特殊区域

### 核心作用
隔离 4、5 类 LSA

### 区域分类

**传输区域（Transit Area）**：承载本区域流量 + 其他区域穿越流量

**末梢区域（Stub Area）**：不接收外部路由

**Totally Stub**：比Stub更彻底，连区域间明细路由都没有

**NSSA**：可以引入外部路由（LSA 7），同时保留选路能力

**Totally NSSA**：NSSA + 完全末梢（无LSA 3明细）

### 关键区别

Totally Stub 没有区域间明细路由，只有一条默认路由，无法在多ABR场景下选路；Stub 有区域间明细路由，可以在多ABR场景下选路。

NSSA 和 Totally NSSA 的区别同上：NSSA 有 LSA 3 明细（适合多ABR场景），Totally NSSA 无 LSA 3 明细（适合单ABR或无需选路）。

### 配置命令

```
ospf 1
 area 1
  stub					  # Stub区域
  stub no-summary		  # Totally Stub区域
  nssa                    # NSSA区域
  nssa no-summary         # Totally NSSA区域
```

### 注意事项
从普通模式切换到特殊区域时，会导致路由重建、邻居重置，出现网络震荡（断网、卡顿）。

**Stub区域限制**：
- 不能是骨干区域
- 所有区域内路由都必须配置为Stub
- 不能引入也不能接收AS外部路由
- 虚连接不能穿越Stub区域

**NSSA区域特点**：
- 可以引入外部路由
- 引入的路由使用 LSA 7（不能传出本区域）
- ABR 会把 LSA 7 转换为 LSA 5 发给骨干区域（Area 0）

---

## 4. 选路顺序

### 优先级（从高到低）
1. 区域内路由
2. 区域间路由
3. 外部路由 Type-1（E1）
4. 外部路由 Type-2（E2）
5. NSSA 路由 Type-1（N1）
6. NSSA 路由 Type-2（N2）

### 选路规则
多条同类型路由比较 cost，cost 小者优先。同类型且 cost 相同则进行负载均衡。

同类型、cost 相同，且通过多个 ABR 收到某外部路由时，优先选择区域号大的那个。例如某外部路由同时从 Area 0 和 Area 2 传入，会优先选择 Area 2。

---

## 5. 默认路由发布详解

### default-route-advertise 的作用
利用 OSPF 动态特性，把默认静态路由变为 ASBR 的外部路由（LSA 5）

### 配置

```
ip route-static 0.0.0.0 0 12.1.1.1    # 指向外网的静态默认路由

ospf 1
 default-route-advertise               # 下发默认路由（外部路由形式）
 default-route-advertise always        # 强制下发，无论本机是否存在默认路由
```

### 故障场景
当外网断开时，静态默认路由失效。如果使用 `default-route-advertise`（不带always），OSPF网络能自动感知并删除该路由。但如果配置了 `always` 或在其他路由器上配了静态默认路由，即使外网已断，路由条目仍然存在，造成路由黑洞。

### 最佳实践
应该使用 OSPF 下发默认路由（`default-route-advertise`，不带 `always`），这样路由加入 OSPF 网络后支持动态选路，外网故障时能自动撤销，便于管理。避免 `always` 或纯静态方式导致的永久残留问题。
