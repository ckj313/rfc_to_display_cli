---
name: rfc-to-huawei-cli
description: Use when mapping RFC fields, packet structures, state-machine objects, path attributes, TLVs, counters, or protocol databases to Huawei display commands across routing, MPLS, multicast, security, and neighbor protocols.
---

# 通过 RFC 字段定位华为设备 CLI

## Overview

这个 skill 用来回答一类问题：

> **RFC 里的字段 / 对象 / TLV / 报文 / 状态，在华为设备上应该看什么 `display` 命令？**

它不是“背命令表”，而是一个**通用推导流程**。

**核心链条固定为：**

```text
字段路径 / RFC章节
-> 标准协议对象
-> 厂商命名模式
-> 华为官方文档命令
-> 输出字段回验
```

只要按这条链走，就不只适用于 OSPF，还能扩展到：
- BGP path attributes
- IS-IS LSP / TLV
- MPLS / LDP / RSVP-TE
- PIM / IGMP / MLD
- BFD
- ARP / ND
- IPsec / IKE payload
- VRRP / LLDP / STP 等协议报文或状态对象

## When to Use

在下面这类问题里使用：
- “RFC 2328 的 Router LSA 对应华为哪个命令？”
- “`BGP-4.path_attributes.NEXT_HOP` 在华为设备上看哪里？”
- “`OSPFv2.body@LSUpdate.lsas[*]@OpaqueAreaLocalLSA.tlvs` 应该看什么？”
- “给你一个 RFC URL / 章节名，你能推导对应的华为查询命令吗？”
- “想把 RFC 字段名和设备输出字段名建立对应关系”

不要用于：
- 非华为设备
- 用户明确要的是配置命令，而不是查询命令
- 只需要二进制报文字节偏移解析，不关心设备 CLI

## 本地手册配置

本 skill 目录下固定放一个配置文件：`manuals-config.json`。

当前约定格式：

```json
{
  "local_chm_paths": [
    "/Users/ccc/Downloads/HUAWEI_usg.chm"
  ],
  "local_html_manual_dirs": []
}
```

使用规则：
- `local_chm_paths`：本地华为 CHM 手册绝对路径列表
- `local_html_manual_dirs`：CHM 解包后的 HTML 手册目录列表
- 查询时先读取这个配置，再决定去哪些本地资料源里搜索
- 如果配置文件存在且路径有效，本地资料源优先级高于线上 `support/hedex`

## 总流程

### 第 1 步：先判断这是“哪一层对象”

先看字段路径，不先看厂商命令。

例如：
- `OSPFv2.body@LSAcknowledge.lsa_headers`
- `OSPFv2.body@LSUpdate.lsas[*]@SummaryASBRLSA`
- `OSPFv2.body@LSUpdate.lsas[*]@OpaqueAreaLocalLSA.tlvs`
- `BGP-4.path_attributes.NEXT_HOP`

先拆成 4 个问题：
1. **协议**是什么
2. **报文类型 / 载体**是什么
3. **对象类型**是什么
4. **对象粒度**是什么

### 对象层级判断规则

| 路径特征 | 对象层级 | 解释 |
| --- | --- | --- |
| `...Acknowledge...headers` / `...packet.header...` | 报文头对象 | 关注报文本身或报文里携带的头部对象 |
| `...Update...objects[*]...` / `...lsas[*]@...` | 协议实体对象 | 关注真正的数据库对象/广告对象本体 |
| `...tlvs` / `...subtlvs` / `...attributes` | TLV / 属性层对象 | 关注对象内部结构，而不是对象名本身 |
| `...statistics` / `...counter` / `...packets_sent` | 统计层对象 | 关注收发统计，不是对象内容 |
| `...state` / `...fsm` / `...neighbor` / `...peer` | 邻居/状态层对象 | 关注状态机、能力、邻接或会话 |

### 重要原则

- `Acknowledge.lsa_headers` 通常意味着：**确认报文里的对象头**，不等于对象 body 本体
- `Update.objects[*]@Xxx` 通常意味着：**真正的对象内容**
- `.tlvs` / `.subtlvs` 通常意味着：**需要更细粒度的对象解析与输出回验**

**不要把“报文层对象”和“数据库/广告对象”混为一谈。**

---

### 第 2 步：先转成标准协议对象名

不要直接用内部字段路径去搜华为命令。
先把路径翻译成**标准协议术语**。

例如：
- `RouterLSA` -> `Router-LSA` -> OSPF Type 1
- `SummaryASBRLSA` -> `ASBR-summary-LSA` -> OSPF Type 4
- `OpaqueAreaLocalLSA` -> `Area-local Opaque LSA` -> OSPF Opaque Area scope
- `NEXT_HOP` -> BGP path attribute `NEXT_HOP`
- `AS_PATH` -> BGP path attribute `AS_PATH`
- `LSPEntry.TLVs` -> IS-IS LSP TLVs

### 标准化目标

这一阶段要拿到的是：
- 协议名
- 标准对象名
- 作用域 / type / subtype / scope
- 是否存在标准编号（如 Type 1 / Type 4 / TLV 22）

如果用户给的是 RFC URL / 章节名，先抽：
1. 协议名
2. 章节里的标准对象名
3. 对象类别（报文 / 数据库 / 属性 / TLV / 状态 / 统计）

**重点：先拿标准名字，再找厂商命令。**

---

### 第 3 步：推导华为 CLI 的命名模式

拿到标准对象名后，再推测华为会怎么命名。

华为命令通常不是照抄 RFC 全称，而是会收敛到较稳定的关键词。

### 常见命令族

| 对象层级 | 常见华为命令族 |
| --- | --- |
| 报文 / message / packet | `display <proto> peer/interface/statistics/error ...` |
| 数据库 / LSA / LSP / FEC / tunnel object | `display <proto> lsdb/database/...` |
| 路由 / path attribute / best path / next hop | `display <proto> routing-table ...` |
| 邻居 / peer / adjacency / session | `display <proto> peer/neighbor/session ...` |
| 统计 / counter / cumulative | `display <proto> statistics` / `display <proto> cumulative` |
| 配置态 / policy / timer | `display current-configuration` / `display this` |

### 厂商关键词映射示例

| 标准对象名 | 华为可能关键词 |
| --- | --- |
| Router-LSA | `router` |
| Network-LSA | `network` |
| Summary-LSA | `summary` |
| ASBR-summary-LSA | `asbr` |
| AS-External-LSA | `ase` |
| Opaque link-local | `opaque-link` |
| Opaque area-local | `opaque-area` |
| Opaque AS-wide | `opaque-as` |
| BGP NEXT_HOP | `routing-table` / `original-attributes` |
| IS-IS LSP | `lsdb` / `database` |
| LDP FEC | `mpls ldp` |

这一阶段的目标是构造：

```text
候选命令模式
+ 候选关键词
+ 候选输出特征
```

---

### 第 4 步：优先查官方资料源，不直接信推断

这一步是必须的。

**资料源优先级：**
1. 本地华为 CHM 手册 / 解包后的 HTML 手册
2. `support.huawei.com`
3. `info.support.huawei.com`
4. 同命令族模式类比推断

**优先搜索位置：**
- 本地华为 CHM 手册
- 本地 CHM 解包后的 HTML 目录
- `support.huawei.com`
- `info.support.huawei.com`

### 搜索策略

不要直接搜整条内部字段路径。

#### 如果有本地华为手册
先搜索本地 CHM 或解包 HTML：

```text
<协议名> + <标准对象名>
<协议名> + <候选关键词>
display + <候选关键词>
```

优先在本地手册里找：
- 命令格式页
- 命令参数页
- 输出示例页
- 诊断命令总表

#### 如果本地手册没有
再搜索线上官方站：

```text
<协议名> + <标准对象名> + Huawei
<协议名> + display + <标准对象名>
<协议名> + <候选关键词> + Huawei
```

例如：
- `display ospf lsdb router Huawei`
- `display ospf lsdb asbr Huawei`
- `display ospf lsdb opaque-area Huawei`
- `display bgp routing-table next hop Huawei`
- `Huawei display bgp original-attributes NEXT_HOP`

### 证据优先级

优先级从高到低：
1. **命令格式页**：参数列表里明确出现候选关键词
2. **命令输出示例**：输出里的 `Type` / `字段名` 能对上目标对象
3. **诊断命令总表**：能确认该命令族存在
4. **本地/线上相邻命令模式类比**：只能作为推断，不算最终确认

### 回答时必须标注证据状态

| 状态 | 说明 |
| --- | --- |
| 官方确认 | 已在本地华为手册或华为官方文档中看到命令格式或输出示例 |
| 高置信推断 | 未直接找到目标页，但同命令族、同对象模式充分一致 |
| 低置信推断 | 只能给候选命令族，不能给唯一命令 |

**搜不到官方页时，不要硬编唯一答案。**

---

### 第 5 步：用输出字段做回验

这一步非常关键。
命令名字像，不代表它就是目标命令。

必须检查输出里是否真的能看到：
- 目标对象类型
- 目标字段名或等价字段
- 目标 scope / subtype / TLV 特征

### 回验规则

| 候选命令 | 期望输出特征 |
| --- | --- |
| `display ospf lsdb router` | `Type : Router` |
| `display ospf lsdb asbr` | `Type : Sum-Asbr` |
| `display ospf lsdb opaque-area` | `Type : Opq-Area` / `Opaque Type` / `Opaque Id` |
| `display bgp routing-table` | `NextHop` / 路由表属性字段 |
| `... original-attributes` | 原始属性名，如 `Original nexthop` |

### 否定性回验

如果候选命令输出对不上目标对象，要明确排除。

例如：
- `display ospf lsdb asbr` 才是 Type 4 / `Sum-Asbr`
- `display ospf asbr-summary` 可能是汇总/统计视图，不一定是 LSDB 里的 Type 4 LSA

**名字像，不算命中；输出对上，才算命中。**

---

### 第 6 步：如果是报文头对象，不要直接等同于对象 body

这是专门的防误判规则。

例如：
- `LSAcknowledge.lsa_headers`

它表示的是：
- 确认报文里带着哪些对象头

它**不等于**：
- 该对象 body 本体的完整内容

所以这类问题要拆两条线回答：

#### 线 A：想看被确认的对象本体
- 去查数据库/对象命令
- 例如：`display ospf lsdb <type>`

#### 线 B：想看设备有没有发/收这种报文
- 去查统计/调试命令
- 例如：`display ospf statistics packet` / `display ospf cumulative` / `debugging ospf`

**报文头对象 ≠ 数据库对象本体。**

## 搜索与推导的固定决策树

```text
输入字段 / RFC章节
-> 识别协议
-> 识别对象层级
-> 标准化为协议对象名
-> 推导华为命名关键词
-> 先搜本地华为 CHM/HTML 手册
-> 再搜华为官方 support/hedex
-> 用命令格式页确认候选命令
-> 用输出字段回验对象是否匹配
-> 给出主命令 / 补充命令 / 证据状态 / 注意事项
```

## 通用输出格式

优先输出结构化结果，便于后续做程序化规则引擎。

```json
{
  "input_field": "<原始字段路径或RFC对象>",
  "protocol": "<协议名>",
  "packet_type": "<报文类型，可选>",
  "object_layer": "<packet_header|object_body|tlv|attribute|state|statistics|config>",
  "canonical_name": "<标准协议对象名>",
  "standard_type": "<Type/TLV编号/Scope，可选>",
  "huawei_cli": "<主命令>",
  "secondary_cli": ["<补充命令1>", "<补充命令2>"],
  "verify_output_contains": ["<期望输出特征1>", "<期望输出特征2>"],
  "evidence_status": "<官方确认|高置信推断|低置信推断>",
  "confidence": 0.0,
  "notes": ["<注意事项1>", "<注意事项2>"]
}
```

## 面向自然语言回答的输出模板

```text
问题对象: <RFC路径 / RFC章节 / 标准协议对象>
协议: <协议名>
报文类型: <可选>
分类: <报文头对象 | 协议实体对象 | TLV/属性层对象 | 邻居/状态层对象 | 统计层对象 | 配置层对象>
标准对象名: <标准协议术语>
标准编号/范围: <Type / Scope / TLV / Attribute，可选>
推荐命令: <主命令>
补充命令:
- <补充命令1>
- <补充命令2>
为什么是它: <1~3句>
字段对应:
- <RFC字段> -> <Huawei字段>
- <RFC字段> -> <Huawei字段>
输出回验:
- 期望看到 <字段/Type/Scope>
证据状态: <官方确认|高置信推断|低置信推断>
置信度: <高/中/低>
注意事项:
- <报文头 vs 对象本体>
- <平台差异>
- <前置条件>
```

## display-only 约束

- 最终答案只允许输出 `display` 开头的命令。
- 即使本地手册中存在 `debugging` 类命令，也只能把它当作背景证据，不作为最终答案输出。
- 报文层对象优先映射到 `display <proto> peer/interface/statistics/error ...` 这类可观察结果面。
- 如果字段本质上来自某个报文，但设备上没有直接的 `display packet ...` 视图，就回答最接近该字段语义的 `display` 观察面。

## 协议无关的主规则

### 规则 1：先分类对象层级

- 报文头对象
- 协议实体对象
- TLV / 属性层对象
- 邻居 / 状态层对象
- 统计层对象
- 配置层对象

### 规则 2：先转标准术语，再找 CLI

不要直接从内部字段名搜厂商命令。
先转成标准协议对象名。

### 规则 3：CLI 优先走统一命令族

- 报文 -> `display <proto> peer/interface/statistics/error ...`
- 数据库对象 -> `display <proto> lsdb/database/...`
- 属性/结果 -> `display <proto> routing-table/...`
- 邻居/状态 -> `display <proto> peer/neighbor/...`
- 统计 -> `display <proto> statistics/cumulative`

### 规则 4：必须做输出回验

命令只有在输出特征对上目标对象时，才算命中。

### 规则 5：遇到 scope / TLV / subtype 必须补范围判断

例如：
- Opaque link-local -> `opaque-link`
- Opaque area-local -> `opaque-area`
- Opaque AS-wide -> `opaque-as`
- BGP 属性 -> 先判断是原始属性、当前值还是递归结果

### 规则 6：优先官方确认，推断必须显式降级

- 能搜到华为官方页，就标 `官方确认`
- 搜不到，只能标 `高置信推断` 或 `低置信推断`
- 不能把推断写成已确认事实

## Worked Example 1：OSPF Ack 报文头（display-only 约束）

**输入**
`OSPFv2.body@LSAcknowledge.lsa_headers`

**推导**
- 协议：OSPFv2
- 报文类型：LSAcknowledge
- 对象层级：报文头对象
- 标准对象：被确认的 LSA 头

**推荐命令**
```text
display ospf statistics packet
```

**补充命令**
```text
display ospf cumulative
display ospf peer
```

**输出回验**
- Ack / LSAck 相关统计项
- 邻居或状态面里的相关结果字段

**注意事项**
- 在本 skill 的约束下，所有回答只允许输出 `display` 命令，不输出 `debugging`
- 这是报文头对象，不是 LSA body 本体
- 如果用户想看被确认的那条 LSA 本体，要继续映射到 `display ospf lsdb <type>`

## Worked Example 2：OSPF Router-LSA

**输入**
RFC 2328 -> Router LSA

**推导**
- 标准对象名：Router-LSA
- 标准编号：Type 1
- 厂商关键词：`router`

**推荐命令**
```text
display ospf [process-id] lsdb router [link-state-id] [originate-router [advertising-router-id] | self-originate]
```

**输出回验**
- `Type : Router`
- `Ls id`
- `Adv rtr`

**字段对应**
- `Link State ID` -> `Ls id`
- `Advertising Router` -> `Adv rtr`
- `LS Age` -> `Ls age`
- `Length` -> `Len`
- `LS Sequence Number` -> `Seq#`
- `LS Checksum` -> `CkSum` / `chksum`
- `Number of links` -> `Link Count`

## Worked Example 3：OSPF ASBR-summary-LSA

**输入**
`OSPFv2.body@LSUpdate.lsas[*]@SummaryASBRLSA`

**推导**
- 协议：OSPFv2
- 报文类型：LSUpdate
- 对象层级：协议实体对象
- 标准对象名：ASBR-summary-LSA
- 标准编号：Type 4
- 厂商关键词：`asbr`

**推荐命令**
```text
display ospf [process-id] lsdb asbr [link-state-id] [originate-router [advertising-router-id] | self-originate]
```

**输出回验**
- `Type : Sum-Asbr`

**关键注意事项**
- Type 3 = `summary`
- Type 4 = `asbr`
- 不要把 `display ospf lsdb asbr` 和其他名字相似的汇总视图混掉

## Worked Example 4：OSPF Opaque Area LSA TLV

**输入**
`OSPFv2.body@LSUpdate.lsas[*]@OpaqueAreaLocalLSA.tlvs`

**推导**
- 协议：OSPFv2
- 报文类型：LSUpdate
- 对象层级：TLV 层对象
- 标准对象名：Area-local Opaque LSA
- 标准范围：Opaque Area / Type 10 scope
- 厂商关键词：`opaque-area`

**推荐命令**
```text
display ospf [process-id] lsdb opaque-area [link-state-id]
```

**输出回验**
- `Type : Opq-Area`
- `Opaque Type`
- `Opaque Id`
- TLV 相关信息

**注意事项**
- 这里关心的是 TLV 级信息，不只是 LSA 类型名
- 如果设备输出只给基本头部，不给 TLV 明细，要显式说明显示粒度受平台限制

## Worked Example 5：BGP NEXT_HOP

**输入**
`BGP-4.path_attributes.NEXT_HOP`

**推导**
- 协议：BGP-4
- 对象层级：属性层对象
- 标准对象名：BGP path attribute `NEXT_HOP`

**推荐命令**
```text
display bgp routing-table <prefix>
```

**补充命令**
```text
display bgp routing-table peer <peer-ip> received-routes <prefix> original-attributes
```

**输出回验**
- `NextHop`
- `Original nexthop`
- `Relay IP Nexthop`

**注意事项**
- `Original nexthop` 是最接近 RFC 原始属性的值
- `NextHop` 是当前表项里的展示字段
- `Relay IP Nexthop` 是递归后的结果
- 不能把三者混为一谈

## 常见错误

1. **字段路径直接找命令**
   - 正确做法是：先转标准对象，再找 `display` CLI

2. **只看命令名字，不看输出回验**
   - 名字像不算命中，输出对上才算命中

3. **把报文头对象当成数据库对象本体**
   - `Acknowledge.headers` 不等于被确认对象的完整 body

4. **把原始协议值和设备最终结果混为一谈**
   - 尤其是 BGP 属性、递归下一跳、策略后结果

5. **遇到 TLV / scope / subtype 不补范围判断**
   - 这会导致选错命令族或选错子类型

6. **搜不到官方页还给唯一答案**
   - 必须显式降级为推断，不能伪装成官方确认

7. **输出了非 `display` 命令**
   - 本 skill 当前约束是：最终答案只允许给 `display` 命令

## References

- RFC 2328: https://www.rfc-editor.org/rfc/rfc2328.txt
- RFC 4271: https://www.rfc-editor.org/rfc/rfc4271
- RFC 5250: https://www.rfc-editor.org/rfc/rfc5250
- Local Huawei CHM manuals / extracted HTML manuals
- Huawei Support: https://support.huawei.com/
- Huawei HedEx: https://info.support.huawei.com/
