---
name: rfc-to-huawei-cli
description: Use when mapping RFC fields, packet formats, protocol states, LSA/path attributes, timers, or counters to Huawei display/debugging commands across routing, MPLS, multicast, and neighbor protocols.
---

# 通过 RFC 字段定位华为设备 CLI

## Overview

这个 skill 不是只回答 OSPF。它的目标是：**把 RFC 里的对象、字段、状态机、报文、属性、计数器，映射成华为设备上的查询命令。**

适用面向：
- 所有协议族：OSPF、BGP、IS-IS、RIP、LDP、RSVP-TE、BFD、PIM、IGMP、MLD、ARP、ND、IPv6 邻居发现等
- 所有 RFC 问法：
  - 直接给字段路径
  - 直接给 RFC URL / 章节名
  - 直接给协议对象名
  - 直接问“这个字段在设备上看什么命令”

**核心原则：先识别“观察对象”，再选命令族，不要一上来猜某条具体命令。**

## When to Use

在下面这类问题里使用：
- “RFC 2328 的 Router LSA 在华为设备上看哪个命令？”
- “RFC 4271 的 `NEXT_HOP` / `AS_PATH` / `LOCAL_PREF` 怎么在华为上查？”
- “`OSPFv2.body@LSAcknowledge.lsa_headers` / `BGP-4.path_attributes.NEXT_HOP` 这种字段路径，对应哪个 CLI？”
- “给你一个 RFC URL，你能推导华为设备上的 show / display / debugging 命令吗？”
- “想把 RFC 字段名和设备输出字段名建立对应关系”

不要用于：
- 非华为设备
- 需要精确原始报文字节偏移/二进制解码而不是 CLI 排障
- 用户明确要求的是配置命令而不是查询命令，但没有说明是否要查“运行态 / 收到值 / 宣告值 / 配置值”

## 核心方法

### 第一步：先识别“用户到底想看什么”

先把 RFC 问题归到下面一种观察对象：

| 观察对象 | 典型关键词 | 常见华为命令族 |
| --- | --- | --- |
| 报文/消息本身 | Hello / Update / Ack / PDU / Notification / TLV | `debugging <proto> packet ...` |
| 协议数据库/广告对象 | LSA / LSP / RIB 条目 / FEC / Label / Join/Prune | `display <proto> lsdb/database/...` |
| 路由/标签结果 | NEXT_HOP / metric / route / label / preference | `display <proto> routing-table ...` / `display ip routing-table` / `display mpls ...` |
| 邻居/会话/FSM | adjacency / peer / neighbor / state / capability | `display <proto> peer/neighbor/adjacency/session ...` |
| 统计/计数器 | cumulative / statistics / counter / packet count | `display <proto> cumulative/statistics ...` |
| 本地配置值 | configured / enabled / timer / policy | `display current-configuration | include ...` 或协议专用 `display this` |
| 收到的原始属性 | received-routes / original attribute / received LSA/LSP | 面向“收到值”的专用显示命令 |
| 发给邻居的值 | advertised-routes / sent / originated | 面向“宣告值”的专用显示命令 |

### 第二步：区分“当前结果”还是“原始协议值”

同一个 RFC 字段，在设备上可能对应多个观察面：

| 用户真实问题 | 应优先回答什么 |
| --- | --- |
| 这个字段当前生效成什么结果？ | 路由表 / 转发表 / 当前数据库 |
| 邻居发给我的原始值是什么？ | `received-routes` / `original-attributes` / 报文调试 |
| 我发给邻居的值是什么？ | `advertised-routes` / 发送侧数据库 / 报文调试 |
| 我本地怎么配置了它？ | `display current-configuration` |

**不要把“原始协议值”和“设备递归后的最终结果”混成一个答案。**

### 第三步：把 RFC 术语翻译成华为术语

华为 CLI 很少原样照抄 RFC 术语。回答时要主动做术语翻译：

| RFC 风格术语 | 华为常见术语 |
| --- | --- |
| packet / message | `packet` / `message` / `debugging` 输出 |
| database | `lsdb` / `database` |
| advertisement | `route` / `lsa` / `lsp` / `advertised-routes` |
| neighbor / peer | `peer` / `neighbor` / `adjacency` |
| path attribute | `attribute` / `routing-table` 明细字段 |
| next hop | `NextHop` / `Original nexthop` / `Relay IP Nexthop` |
| checksum / sequence / age | 可能显示为缩写：`CkSum` / `Seq#` / `Ls age` |

### 第四步：如果用户只给 RFC URL 或章节名

按这个顺序处理：

1. 抽取协议名
2. 抽取对象名
   - 例如：Router LSA、Summary ASBR LSA、NEXT_HOP、AS_PATH、LDP FEC、PIM Join/Prune
3. 判定对象属于哪一层
   - 报文层 / 数据库层 / 路由结果层 / 邻居层 / 统计层 / 配置层
4. 再选 CLI 家族
5. 最后给字段映射和注意事项

**重点：先像“例子 2”那样抓协议对象，再找命令，不要直接从 RFC 标题跳到具体命令。**

## 通用命令选择框架

### A. 报文层

如果对象是报文、PDU、packet body、message header：

```text
debugging <proto> packet ...
```

典型场景：
- OSPF Hello / DD / LSUpdate / LSAck
- BGP OPEN / UPDATE / NOTIFICATION
- IS-IS IIH / LSP / SNP
- LDP message / label mapping
- PIM Join/Prune / Hello

### B. 数据库/广告对象层

如果对象是协议数据库条目、LSA、LSP、FEC、标签映射、组播树条目：

```text
display <proto> lsdb/database/...
```

典型场景：
- OSPF LSA → `display ospf ... lsdb ...`
- IS-IS LSP → `display isis lsdb ...`
- MPLS LDP FEC / LSP → `display mpls ldp ...`
- PIM 路由/组播树 → `display pim routing-table`

### C. 路由/属性结果层

如果对象是 NEXT_HOP、MED、LOCAL_PREF、metric、best path、preference：

```text
display <proto> routing-table ...
```

必要时补：
- 当前生效值
- 邻居收到的原始属性
- 递归后的下一跳

### D. 邻居/会话层

如果对象是 capability、state、holdtime、keepalive、adjacency 状态：

```text
display <proto> peer/neighbor/adjacency/session ...
```

### E. 统计层

如果对象是收发计数、消息类型计数、累计统计：

```text
display <proto> cumulative
display <proto> statistics
```

### F. 配置层

如果对象问的是“设备是否启用了这个能力 / 这个 timer 配了多少”：

```text
display current-configuration | include <keyword>
```

## 常见协议的起始命令家族

> 这是“起始命令族”，不是完整命令大全。回答时要先判对象层次，再落到某个家族。

| 协议 | 常见观察对象 | 华为起始命令族 |
| --- | --- | --- |
| OSPF | LSA / 邻居 / 包 / 统计 | `display ospf ...` / `debugging ospf ...` |
| BGP | 路由属性 / peer / 收到路由 / 宣告路由 | `display bgp ...` |
| IS-IS | LSP / 邻接 / SPF / 包 | `display isis ...` / `debugging isis ...` |
| RIP | 路由 / 邻居 / 报文 | `display rip ...` / `debugging rip ...` |
| BFD | 会话状态 / 计数器 | `display bfd session ...` |
| LDP | peer / session / FEC / label | `display mpls ldp ...` |
| RSVP-TE / TE | tunnel / path / RSVP 状态 | `display mpls te ...` / `display rsvp-te ...` |
| PIM | neighbor / join-prune / routing-table | `display pim ...` |
| IGMP / MLD | group / interface / statistics | `display igmp ...` / `display mld ...` |
| ARP / ND | 邻居缓存 / 接口邻居 | `display arp` / `display nd` |

## 输出规则

当你无法 100% 确认某条唯一命令时，**不要硬编成唯一答案**。按下面格式回答：

```text
问题对象: <RFC 路径 / RFC 章节 / 协议对象>
协议: <OSPF/BGP/IS-IS/...>
分类: <报文层 | 数据库层 | 属性/结果层 | 邻居层 | 统计层 | 配置层>
推荐命令: <最可能的主命令>
补充命令:
- <用来看原始值的命令>
- <用来看结果值的命令>
- <用来看统计的命令>
为什么是它: <1~3 句>
字段对应:
- <RFC 字段> -> <Huawei 字段>
- <RFC 字段> -> <Huawei 字段>
置信度: <高 / 中 / 低>
注意事项:
- <平台差异>
- <原始值 vs 当前结果>
- <如果需要前置配置，如 keep-all-routes>
```

## Worked Example 1：OSPF Ack 报文

**问题对象**
`OSPFv2.body@LSAcknowledge.lsa_headers`

**分类**
- 报文层

**推荐命令**
```text
debugging ospf [process-id] packet ack [interface ...] [brief] [filter ...]
```

**补充命令**
```text
display ospf cumulative
```

**为什么是它**
- `LSAcknowledge` 是 OSPF 报文类型
- `lsa_headers` 是 Ack 报文里携带的 LSA Header 列表
- 所以优先走 **debug 报文**，不是 LSDB

**字段对应**
- `LS Type` -> `Type`
- `Link State ID` -> `Ls id` / `LinkState ID`
- `Advertising Router` -> `Adv rtr` / `AdvRouter`
- `LS Age` -> `Ls age` / `Age`
- `LS Sequence Number` -> `Seq#` / `Sequence`
- `LS Checksum` -> `CkSum` / `chksum`

**注意事项**
- 这是报文字段，不是数据库条目
- 如果用户只关心是否存在 Ack 收发，可退化成 `display ospf cumulative`

## Worked Example 2：RFC 2328 的 Router LSA

**问题对象**
RFC 2328 → Router LSA

**分类**
- 数据库层 / LSA 对象层

**推荐命令**
```text
display ospf [process-id] lsdb router [link-state-id] [originate-router [advertising-router-id] | self-originate]
```

**为什么是它**
- `Router LSA` 是明确的协议对象
- 所以像“例子 2”这种问法，应该先抽取对象名 **Router LSA**，再映射到 `lsdb router`

**字段对应**
- `Link State ID` -> `Ls id`
- `Advertising Router` -> `Adv rtr`
- `LS Age` -> `Ls age`
- `Length` -> `Len`
- `Options` -> `Options`
- `LS Sequence Number` -> `Seq#`
- `LS Checksum` -> `CkSum` / `chksum`
- `Number of links` -> `Link Count`
- `Link ID` -> `Link ID`
- `Link Data` -> `Data`
- `Link Type` -> `Link Type`
- `Metric` -> `Metric`

**你可以按这种风格生成解释**

```text
OSPF Process 1 with Router ID 1.1.1.1
                 Link State Database

  Type      : Router                 <-- [RFC 2328] LSA Type = Router-LSA
  Ls ID     : 1.1.1.1                <-- [RFC 2328] Link State ID
  Adv Rtr   : 1.1.1.1                <-- [RFC 2328] Advertising Router
  Ls Age    : 117                    <-- [RFC 2328] LS Age
  Len       : 48                     <-- [RFC 2328] Length
  Options   : E                      <-- [RFC 2328] Options
  Seq#      : 80000004               <-- [RFC 2328] LS Sequence Number
  CkSum     : 0xd32b                 <-- [RFC 2328] LS Checksum
  Flags     : 0x0 / B / V / E        <-- [RFC 2328] Router-LSA Flags（平台相关）

  Link Count: 2                      <-- [RFC 2328] Number of links
```

## Worked Example 3：OSPF Summary ASBR LSA

**问题对象**
`OSPFv2.body@LSUpdate.lsas[*]@SummaryASBRLSA`

**分类**
- 数据库层 / LSA 对象层

**推荐命令**
```text
display ospf [process-id] lsdb asbr [link-state-id] [originate-router [advertising-router-id] | self-originate]
```

**关键点**
- Type 3 = `summary`
- Type 4 = `asbr`

**字段对应**
- `SummaryASBRLSA` -> `Type: Sum-Asbr`
- `Link State ID` -> `Ls id`
- `Advertising Router` -> `Adv rtr`
- `LS Age` -> `Ls age`
- `Length` -> `Len`
- `LS Sequence Number` -> `Seq#`
- `LS Checksum` -> `CkSum` / `chksum`
- `Metric` -> `Tos 0 metric`

## Worked Example 4：BGP RFC 4271 的 NEXT_HOP

**问题对象**
`BGP-4.path_attributes.NEXT_HOP`

**分类**
- 属性/结果层

**推荐命令**
```text
display bgp routing-table <prefix>
```

**补充命令**
如果用户问的是“邻居发给我的原始 RFC 属性值”：

```text
display bgp routing-table peer <peer-ip> received-routes <prefix> original-attributes
```

**为什么是它**
- RFC 4271 的 `NEXT_HOP` 是 BGP UPDATE 路径属性
- 在设备上通常要分清：
  - `NextHop`：当前路由表展示列
  - `Original nexthop`：更接近收到的原始 RFC 属性
  - `Relay IP Nexthop`：递归解析后的结果

**字段对应**
- `NEXT_HOP` -> `Original nexthop`
- `NEXT_HOP` -> `NextHop`
- `NEXT_HOP` 的递归结果 -> `Relay IP Nexthop`

**注意事项**
- 不要把“收到的原始属性”和“递归后的最终转发结果”混成一个字段
- 某些视图需要前置条件，例如保留收到路由

## 常见错误

1. **直接从 RFC 名称跳到具体命令**
   - 应先判断对象层次：报文 / 数据库 / 属性 / 邻居 / 统计 / 配置

2. **把原始协议值和最终生效结果混为一谈**
   - 例如 BGP `NEXT_HOP` ≠ 递归后的下一跳结果

3. **把“收到值 / 宣告值 / 当前值 / 配置值”混为一谈**
   - 这四种观察面经常需要不同命令

4. **只会套 OSPF 的 `lsdb` 思路**
   - OSPF 是 worked example，不是唯一协议
   - BGP、IS-IS、LDP、PIM 都要先回到“对象层次”再选命令族

5. **把平台差异写成绝对化结论**
   - 回答时要标出：不同 VRP 平台、版本、产品线，字段名和可见深度可能不同

## References

- RFC 2328: https://www.rfc-editor.org/rfc/rfc2328.txt
- RFC 4271: https://www.rfc-editor.org/rfc/rfc4271
- Huawei `display ospf lsdb`: https://support.huawei.com/enterprise/en/doc/EDOC1100459463/9b099a20
- Huawei OSPF debugging commands: https://support.huawei.com/enterprise/en/doc/EDOC1100332318/915d8ce7/ospf-debugging-commands
- Huawei `display ospf cumulative`: https://support.huawei.com/enterprise/en/doc/EDOC1100064351/a23305cc/display-ospf-cumulative
- Huawei `display bgp routing-table`: https://support.huawei.com/enterprise/en/doc/EDOC1100325910/bdd63b27/display-bgp-routing-table
