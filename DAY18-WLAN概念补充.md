# DAY18：WLAN概念补充

## 一、二层组网与三层组网

- **二层组网**：所有AP和AC在同一个二层网络环境下，AP通过广播即可发现AC。
- **三层组网**：AP和AC不在一个二层网络中，处于三层网络互联的环境下，AP需借助Option43或DNS等方式获取AC地址。


## 二、DHCP的配置方式

### 方式1：`dhcp select global`（全局地址池模式）

**判断逻辑**：设备拿接收请求的接口IP地址，去和所有全局地址池的`network`段做网段匹配。

```bash
# 接口配置
interface Vlanif10
 ip address 192.168.10.1 255.255.255.0
 dhcp select global

# 全局地址池
ip pool VLAN10
 network 192.168.10.0 mask 255.255.255.0
```

**匹配规则**：请求从Vlanif10进来，IP是192.168.10.1，设备遍历所有全局池，找到network 192.168.10.0/24能包含该IP，就用这个池。不需要显式绑定，设备自动对号入座。

### 方式2：`dhcp select interface`（基于接口模式）

**判断逻辑**：直接使用接口自身的IP地址网段作为地址池，不需要另外创建`ip pool`。

```bash
interface Vlanif10
 ip address 192.168.10.1 255.255.255.0
 dhcp select interface
 dhcp server dns-list 114.114.114.114
```

**效果**：请求从Vlanif10进来，分配192.168.10.2-192.168.10.254（排除接口IP自己），系统自动生成一个隐含池。

### 方式3：`dhcp select relay`（中继模式）

**判断逻辑**：不自己分配，直接把请求转发给指定的远端DHCP服务器。

```bash
interface Vlanif10
 ip address 192.168.10.1 255.255.255.0
 dhcp select relay
 dhcp relay server-ip 10.10.10.100
```

此时判断权在远端服务器，它会根据请求报文中的网关地址（giaddr字段）来选池。

![image-20260716202102090](DAY18-WLAN概念补充随笔.assets/image-20260716202102090.png)


## 三、Wi-Fi安全协议对比

| 安全协议                  | 链路认证          | 接入认证          | 数据加密           | 适用场景             | 安全级别 |
| ------------------------- | ----------------- | ----------------- | ------------------ | -------------------- | -------- |
| WEP（已淘汰）             | Open / Shared-Key | 不涉及            | 不加密 / WEP       | 无                   | 🔴 危险   |
| WPA/WPA2-802.1X（企业级） | Open              | 802.1X（EAP）     | TKIP / CCMP（AES） | 大型企业、政府、金融 | 🟢 高     |
| WPA/WPA2-PSK（个人级）    | Open              | PSK（预共享密钥） | TKIP / CCMP（AES） | 中小企业、家庭、SOHO | 🟡 中高   |


## 四、接口上使能OSPF

```
interface Vlanif10
 ospf enable 1 area 0
```


## 五、三层组网Option43配置

### 概念

1. Option43是DHCP协议中的标准选项字段，用于在分配IP地址时附带额外信息。
2. 在华为无线组网中，它专门用来携带AC的IP地址，让跨网段的AP能够获知AC的位置，从而建立CAPWAP隧道并上线。

### 作用

1. 解决三层组网中AP和AC不在同一网段时，AP无法通过广播方式自动发现AC的问题。
2. AP通过DHCP获取IP地址的同时，从Option43字段中解析出AC的IP地址，随后用单播方式发起连接请求。
3. 无需在AP上手工配置AC地址，实现AP的零配置上线。

### 使用方法（以华为设备为例）

华为设备支持三种子选项格式，区别在于AC地址的填写方式：

| 子选项       | 格式        | 配置示例                                      | 特点                         |
| ------------ | ----------- | --------------------------------------------- | ---------------------------- |
| sub-option 3 | ASCII字符串 | `option 43 sub-option 3 ascii 10.1.10.1`      | 最直观，直接写IP地址，最常用 |
| sub-option 2 | IP地址      | `option 43 sub-option 2 ip-address 10.1.10.1` | 比较直观                     |
| sub-option 1 | 十六进制    | `option 43 sub-option 1 hex C0A80A01`         | 需转换进制，复杂易错，较少用 |

推荐使用`sub-option 3 ascii`，直接写AC的IP地址，无需转换，最不易出错。

**单个AC配置**：
```
dhcp server option 43 sub-option 3 ascii 192.168.20.2
```

**多个AC配置**（逗号分隔，最多8个）：
```
dhcp server option 43 sub-option 3 ascii 192.168.100.1,192.168.100.2
```

**接口地址池中配置**（VLANIF视图）：
```
[L3-Vlanif100] dhcp server option 43 sub-option 3 ascii 10.1.10.1
```

**全局地址池中配置**：
```
[WAC-ip-pool-ap] option 43 sub-option 3 ascii 10.1.10.1
```

### 三层组网中的路由问题

当AC和AP被VLAN隔开时，三层交换机有各VLAN的IP和直连路由，但AC没有去往AP网段的路由。解决方法是在AC上添加一条默认路由指向三层交换机网关，这样所有AP回程流量都走交换机，由交换机的直连路由完成精确转发。


## 六、三层无线组网实验（AC跨VLAN管理AP）

### 地址规划

| 设备 | VLAN    | 接口地址       | 说明        |
| ---- | ------- | -------------- | ----------- |
| AC   | VLAN10  | 10.1.10.1/24   | AC管理地址  |
| LSW1 | VLAN10  | 10.1.10.254/24 | AC网关      |
| LSW1 | VLAN100 | 10.1.100.1/24  | AP1管理网关 |
| LSW1 | VLAN200 | 10.1.200.1/24  | AP2管理网关 |
| LSW1 | VLAN102 | 11.1.1.1/30    | 与AR互联    |
| AR1  | VLAN102 | 11.1.1.2/30    | 与LSW1互联  |

### LSW1（三层交换机）配置

```
vlan batch 10 100 200 102

# AP1接入端口（管理VLAN 100）
int g0/0/1
 port link-type trunk
 port trunk pvid vlan 100
 port trunk allow-pass vlan 100

# AP2接入端口（管理VLAN 200）
int g0/0/2
 port link-type trunk
 port trunk pvid vlan 200
 port trunk allow-pass vlan 200

# AC互联端口（放行VLAN10管理流量）
int g0/0/3
 port link-type trunk
 port trunk allow-pass vlan 10

# AR互联端口
int g0/0/4
 port link-type access
 port default vlan 102

# 各VLANIF接口配置
int Vlanif10
 ip add 10.1.10.254 24

int Vlanif100
 ip add 10.1.100.1 24
 dhcp select interface
 dhcp server option 43 sub-option 3 ascii 10.1.10.1

int Vlanif200
 ip add 10.1.200.1 24
 dhcp select interface
 dhcp server option 43 sub-option 3 ascii 10.1.10.1

int Vlanif102
 ip add 11.1.1.1 30

ip routing
ip route-static 0.0.0.0 0.0.0.0 11.1.1.2
```

### AC（无线控制器）配置

```
vlan batch 10
int g0/0/1
 port link-type trunk
 port trunk allow-pass vlan 10

int Vlanif10
 ip add 10.1.10.1 24

capwap source interface Vlanif10

# 关键：默认路由指向交换机，解决所有AP网段回程问题
ip route-static 0.0.0.0 0.0.0.0 10.1.10.254

wlan
 ap auth-mode no-auth

 ap-group name group1

 regulatory-domain-profile name group1
  country-code cn

 ssid-profile name group1
  ssid wlan-test

 security-profile name group1
  security wpa-wpa2 psk pass-phrase Huawei@123 aes

 vap-profile name group1
  forward-mode tunnel
  service-vlan vlan-id 101
  security-profile group1
  ssid-profile group1

 ap-group name group1
  regulatory-domain-profile group1
  vap-profile group1 wlan 1 radio 0
  vap-profile group1 wlan 1 radio 1
```

### AR1（出口路由器）配置

```
int g0/0/0
 ip add 11.1.1.2 30

int LoopBack0
 ip add 100.100.100.100 32

ip route-static 10.1.0.0 255.255.255.0 11.1.1.1
```

### AP上线配置

```
wlan
 ap-id 0
  ap-name AP1
  ap-group group1
 quit

 ap-id 1
  ap-name AP2
  ap-group group1
 quit
```

### 验证命令

```
display ap all                     # AP状态应为Normal
display ap unauthorized record     # 查看未认证AP连接记录
display vap ssid wlan-test         # 查看VAP是否生效
display ip routing-table           # AC上确认默认路由已配置
```

### 流量走向

AP通过Option43拿到AC地址（10.1.10.1）→ 发起CAPWAP请求 → AC查路由表，默认路由指向交换机（10.1.10.254）→ 交换机有各AP网段的直连路由 → 报文送达AP，双向通信建立。

### 关键点

| 关键点       | 说明                                       |
| ------------ | ------------------------------------------ |
| Option43     | 在AP管理VLAN的DHCP地址池中配置，指向AC的IP |
| AC默认路由   | 指向三层交换机网关，新增AP网段无需改AC配置 |
| CAPWAP源接口 | AC上必须指定，告诉AP用哪个IP建隧道         |
| 三层路由     | 交换机需开启`ip routing`                   |
| 安全模式     | 测试用no-auth，生产环境改sn-auth或mac-auth |

### 与二层组网的区别

| 对比项       | 二层组网             | 三层组网                       |
| ------------ | -------------------- | ------------------------------ |
| AC与AP位置   | 同一网段             | 不同网段，中间隔三层交换机     |
| AP发现AC方式 | 广播自动发现         | 需通过Option43指定AC地址       |
| AC回程路由   | 不需要（同网段直连） | 需要（默认路由指向三层交换机） |
| 配置复杂度   | 简单                 | 稍复杂，需规划路由和Option43   |
| 适用场景     | 小型园区             | 大型园区，AP分布在不同网段     |


## 七、真机桥接访问AC管理页面

在ENSP中可以添加Cloud设备，与真机网卡桥接，然后就可以在网页中访问AC的Web管理页面了。具体操作：

1. 添加Cloud设备到拓扑，绑定物理网卡（如有线或无线网卡）。
2. 在Cloud设备上配置端口映射，将AC的IP与本地网卡IP连通。
3. 确保PC网卡IP与AC管理地址在同一网段或路由可达。
4. 浏览器输入AC的VLANIF10接口IP（如`https://10.1.10.1`），即可登录AC的Web管理界面进行图形化配置。
