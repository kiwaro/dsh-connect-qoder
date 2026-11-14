# 活动优惠徽标：取证与判定

活动徽标的文案、时段与规则来自上游模型目录。插件在渲染时判断状态，并在模型名旁展示。

## 1. 数据来源

`/algo/api/v2/model/list?Encode=1` 的模型行包含 `promotion` 对象。插件在既有目录请求中读取，通过 `src/promo.ts` 解析并计算显示状态。

2026-10-21 16:2x 本机实测（有效凭据、只读 GET、不消耗积分）：

| 区域 | 主机 | 去重后模型数 | 带 `promotion` 的行 |
| --- | --- | --- | --- |
| 国内 | `https://gateway.qoder.com.cn` | 11 | 2（`qmodel_38max`、`qmodel`） |
| 国际（全部分组） | `https://api1.qoder.sh` | 22 | 5（`ultimate`、`quest-ultimate`、`experts-ultimate`、`qmodel_38max`、`qmodel`） |
| 国际（插件读的 `chat` 分组） | 同上 | 13 | 3（`ultimate`、`qmodel_38max`、`qmodel`） |

两个区域是**同一套 schema**，只有时区和文案不同：国内 `Asia/Shanghai`，「错峰 4 折」；
国际 `Asia/Singapore`，「错峰2折」/`Off-Peak 80% off`。国内文案存在格式差异：
`qmodel_38max` 是「错峰 4 折」（有空格），`qmodel` 是「错峰4折」（没空格）。插件保留上游原文。

## 2. 逐字抓包（国内 `qmodel_38max`）

```json
"promotion": {
  "active": false,
  "badge": { "en": "Off-Peak 60% off", "zh": "错峰 4 折" },
  "description": {
    "en": "Off-Peak 60% off (10 PM-8 AM UTC+8)",
    "zh": "错峰时段4折优惠（10 PM-8 AM UTC+8）"
  },
  "timezone": "Asia/Shanghai",
  "rule_id": "idle_time_model_credit_discount",
  "discount_factor": 0.4,
  "before_promotion_price_factor": 0.5,
  "link_url": {
    "zh": "https://docs.qoder.cn/product-overview/qwen-3-7-series-model-staggering-discount"
  },
  "window_start": "22:00",
  "window_end": "08:00"
}
```

国际区同一条规则换成了 `discount_factor: 0.2` / `before_promotion_price_factor: 2`，
`timezone: "Asia/Singapore"`，`link_url` 指向 `https://docs.qoder.com/events/offpeakrate`。

字段三组：**展示字段**（`badge`、`description`）、**规则**（`active`、`rule_id`、
`discount_factor`、`window_start/end`、`timezone`）、**跳转**（`link_url`）。

## 3. 判定规则（`src/promo.ts`）

1. **文案要求**：`badge.zh` 或 `badge.en` 至少有一项有效，缺少文案时省略规则。
2. **状态计算**：有 `window_start/end` 时，按渲染时刻判断是否处于窗口内；缺少窗口声明时使用上游 `active`。
3. **无窗口的规则**：`active: true` 时显示，`false` 时省略。此类规则缺少可计算的生效时刻。
4. **时区与有效时段**：使用上游 `timezone`，起点包含、终点不含。跨零点窗口按 `[22:00, 24:00) ∪ [0, 08:00)` 判断；终点接受 `24:00`，`start === end` 视为全天。时间越界、缺少一端或格式不符时省略规则。
5. **显示位置**：每个模型一枚徽标，位于模型名之外、倍率之后，与 free 标记及测试结果并列。窗口外采用灰显状态（§3.3）。
6. **文案与悬停**：按界面语言读取 `badge.zh/en`；悬停组合 `description`、时段、时区及 `link_url` 原文。折扣数字直接使用上游文案；`discount_factor` 等三个数值字段在解析时省略，存储对象保留展示和判定所需字段。

### 3.1 时段优先于目录快照状态

`active` 表示请求目录时的状态。目录在进程启动或手动刷新时读取，因此可能跨越多个活动时段：白天读取的 `active:false` 到夜间仍保留原值；夜间读取的 `active:true` 到白天也可能保留。

`window_start/end` 与 `timezone` 是上游声明的时段，可结合当前时钟重新计算。插件因此优先使用时段，缺少时段时回退至 `active`。

上游提前取消活动时，本地仍可能显示徽标至下一次目录刷新，限制见 §7。用户可执行“刷新模型”更新规则。

### 3.2 渲染时计算活动状态

`promotion` 随 `lastCatalog` 保存规则。若解析时就固定显示结果，23:00 读取的活动状态会在次日 09:00 继续保留。当前实现保存 `QoderPromotion`，在渲染时结合 `new Date()` 调用 `promoDisplayAt` / `promoLiveAt`。WorkBuddy 的 `selectPromoBadge` 与 Trae 的 `selectTraePromoBadge` 采用相同方式。

### 3.3 窗口外的灰显状态

2026-10-23 用户于 14:5x 打开卡片，`Qwen3.8-Max` 未显示活动徽标，而目录中已有 `qmodel_38max` 的 `22:00–08:00 Asia/Shanghai` 规则，`active:false`。旧实现只展示生效中的活动，无法向窗口外访问的用户说明后续优惠。

`promoDisplayAt` 返回以下三种状态：

| 状态 | 条件 | 展示 |
| --- | --- | --- |
| `live` | 当前在窗口内，或无窗口且 `active:true` | 实心品牌色徽标 |
| `pending` | 窗口有效，当前在窗口外 | 灰色文字、中性底色、虚线描边 |
| `none` | 无规则、窗口无效，或无窗口且 `active:false` | 省略徽标 |

`pending` 同时改变颜色、底色和描边形状，便于在色觉差异及高对比主题下区分状态。悬停先显示 `row.promoPending`（“当前未生效，22:00 起生效”），再显示上游说明、时段和链接。无效窗口无法给出可信的生效时间，因此保持 `none`。

## 4. 与另两个插件的能力对照

| 能力 | workbuddy PR #91 | Trae PR #44 | Qoder（本 PR） |
| --- | --- | --- | --- |
| 文案来源 | 上游 `badge.label` | 无文案，插件自造「闲时」 | 上游 `badge.zh/en`，原样 |
| 颜色 | 上游 `badge.color` | 插件自选三色 | 插件自选（上游没给） |
| 绝对生命周期 | `validFrom/validUntil` | 无 | 无（只有每日时段） |
| 每日时段 | `schedule.daily`（具名时区） | `off_peak.time_windows`（周几+分钟） | `window_start/end` + `timezone` |
| 多枚候选 / 优先级 | 有（`priority`、`activeOnly`） | 无（一对一） | 无（一对一） |
| 服务端「此刻生效」标志 | 无 | `current.discount_type` | `active` |
| 额外请求 / 换客户端身份 | 需要（`/v3/config` 按通道下发） | 不需要 | **不需要** |

在上述版本对照中，Qoder 的文案和规则来自既有目录请求，可直接按上游原文展示。

## 5. 实机复核

`scripts/probe-promo.mjs` 用有效凭据、cosy 签名（与客户端同一条路径）读一次目录，然后跑
**插件的解析与判定**（`parseUpstreamModel` → `parseQoderPromotion` → `promoLiveAt`）：

```
$ node --experimental-transform-types scripts/probe-promo.mjs --shifts
account=QODERCN_ACCOUNT_EXAMPLE region=cn uid=<uid 前 6 位>
GET /algo/api/v2/model/list?Encode=1 → HTTP 200 / 55477 bytes
chat rows: 11
parsed models: 11   carrying a rule: 2
- qmodel_38max  active=false  22:00–08:00 Asia/Shanghai  badge[zh]="错峰 4 折"
- qmodel        active=false  22:00–08:00 Asia/Shanghai  badge[zh]="错峰4折"

== badges now (2026-10-21T09:23:24.783Z) ==
(none)
+6h  (15:23Z = 23:23 Asia/Shanghai): qmodel_38max「错峰 4 折」, qmodel「错峰4折」
+12h (21:23Z = 05:23 次日):          qmodel_38max「错峰 4 折」, qmodel「错峰4折」
+24h (09:23Z = 17:23 次日):          (none)
+36h (21:23Z = 05:23 第三天):         qmodel_38max「错峰 4 折」, qmodel「错峰4折」
```

两条规则的时段、时区和中文文案均解析成功。同一份解析结果在 17:23（窗口外）显示零枚徽标，在 +6h（23:23）显示两枚，在 +24h（次日 17:23）再次显示零枚。

这组模拟验证了 §3.2 的渲染时刻判断。解析结果中的 `active` 始终为 `false`，保留取目录时的状态；存在窗口规则时，插件依据时段判断。`tests/promo.spec.ts` 使用固定时刻覆盖该行为。

### 5.1 窗口内实机复核（2026-10-21 22:2x）

上面的时钟平移是模拟。活动窗口开启后，使用真实时钟再次复核，结果如下：

1. **`active` 确实按当前时刻计算，在窗口内为 true。** 22:2x DSH 的 `/usage` 响应中，
   `qmodel_38max` 与 `qmodel` 的 `promotion.active` 都已是 `true`（16:2x 同一批规则是
   `false`）。所以 §3.1 的担忧不是「上游始终不给 true」，而是**这个 true 只在取目录那
   一刻有效**——白天取到的 `false` 不会因为到了夜里就自己变。
2. **真实时钟下显示两枚徽标。** 用DSH `/usage` 的真实返回传给卡片组件渲染
   （数据、组件、CSS 都是生产代码），`Qwen3.8-Max (0.20x)` 旁出现「错峰 4 折」、
   `Qwen3.7-Plus (0.10x)` 旁出现「错峰4折」，位置在名字之外、倍率之后。截图存为
   `docs/assets/dsh-connect-qoder-promo-badge.png`（README 双语引用、`screenshots.json`
   登记）。

> 截图使用窗口内的实机数据与生产组件渲染：`promotion` 取自运行中DSH的
> `/usage`，启用状态、倍率、上下文都来自同一份真实目录。

同时确认：`quest-ultimate` / `experts-ultimate` 只出现在 `quest` / `experts` 分组里，
而插件目录只读 `chat`，所以国际区实际能画的是 `chat` 分组里的三条：`ultimate`、
`qmodel_38max`、`qmodel`（同上命令换成 `QODER_ACCOUNT_EXAMPLE` 即可复核）。

## 6. 实现位置

| 位置 | 作用 |
| --- | --- |
| `src/promo.ts` `parseQoderPromotion` | 上游 snake_case → `QoderPromotion` |
| `src/promo.ts` `readPromotion` | 从文件读取活动数据，使用同一解析函数生成 `QoderPromotion` |
| `src/promo.ts` `clockMinute` / `parsePromoWindow` | `HH:MM` → 分钟数（终点接受 `24:00`） |
| `src/promo.ts` `promoWithinWindow` | 按上游时区判时段（跨零点、起点含终点不含） |
| `src/promo.ts` `promoLiveAt` | 渲染时刻的「当前时刻是否显示」 |
| `src/catalog.ts` `QoderModelInfo.promotion` | 目录条目带规则（`parseUpstreamModel` 里解析） |
| `src/index.ts` `modelConfig.promotion` | `lastCatalog` 持久化 schema（`z.any<QoderPromotion>()`） |
| `src/status-paths.ts` `QoderWebModel` / `QoderPersistedModel` | 传给卡片的模型行数据类型 |
| `src/web-status.ts` `webModelOf` | 构建卡片数据时，通过对象展开保留活动字段 |
| `src/client/promo-badge.ts` `hostLanguage` / `promoBadgeLabel` / `promoBadgeTooltip` | 选择中文或英文文案，生成悬停提示文本 |
| `src/client/QoderCard.tsx` `renderModels` | 在模型名旁渲染活动徽标 |
| `src/model-badge.ts` `modelBadgeOf` | 选择器胶囊的**数据侧**：此刻该挂哪一枚（`live` 才挂）、文案表、16 grapheme 上限 |
| `src/adapter.ts` `listModels` | 将徽标数据写入 `LlmModelInfo.badge`（展开写法，0.1.x 也能编译） |
| `src/client/QoderModelLabel.tsx` `createQoderModelLabel` | 选择器里的同一枚胶囊（issue #12；工厂按 fiber 捕获槽位表，非本插件 provider 委托回账本下一位） |
| `src/client/host-badge.tsx` `ForeignModelLabel` | 其他 provider 的行：名字 + 该模型自带的 `model.badge`（trigger 在前、菜单在后） |
| `src/client/index.tsx` `registerModelLabel` | `conversation.input.model.label` 注册：keyed DSH按 provider 分别注册；旧 single DSH使用 `priority: -101`，由优先级决定渲染项；0.2.2 DSH的默认渲染包含模型名与目录徽标 |
| `src/client/styles.ts` `.dsm-qoder-model-promo` / `.dsm-qoder-model-label-host-badge` | 本插件自己的胶囊 / 其他 provider 行上复现DSH `Tag` 的几何 |
| `scripts/probe-promo.mjs` | 实机只读探针（解析 + 判定 + 时钟平移） |

## 7. 已知限制与DSH兼容

- **活动链接**：`link_url` 作为悬停文本展示，卡片徽标采用提示形式。
- **颜色**：上游提供文案，徽标颜色由插件选用品牌蓝，与倍率标签保持一致。
- **提前取消活动**：有窗口时按本地时钟计算，上游取消或提前结束活动后，需刷新目录才能取得新规则。
- **账号资格**：窗口判断仅依据时段；若上游通过 `active` 表达账号资格，该信息在窗口分支中不会单独参与判断。无窗口活动直接使用 `active`。§5.1 实测账号在窗口内取得 `active:true`。
- **目录范围**：插件读取 `chat` 分组；`quest` / `experts` 专属模型及活动保留在对应上游分组。
- **卡片刷新间隔**：重新渲染时计算活动状态，自动更新受 `pollMinutes` 控制，默认 5 分钟；1.22.0 前固定为 60 秒。因此仅靠自动轮询时，窗口切换的显示延迟通常最多为一个刷新间隔，另受请求与页面调度影响。
- **标签组件数据缓存**：读取 `detail=fast` 返回的本地缓存数据，按区域共享缓存并采用 `pollMinutes` 刷新。共享分钟定时器重新读取时钟，计算窗口切换；规则变更仍须取得新目录。
- **目录徽标更新**：`LlmModelInfo.badge` 随模型目录刷新。通过该字段显示的徽标会保留至下一次目录更新；本插件标签组件另按当前时钟计算。每次发 `llm/models-updated` 会让DSH逐模型调用 `resolveModelInfo()`，当前实现因此未为提示性徽标增加逐分钟目录更新。文案上限为 16 个 grapheme，超长时省略整枚。
- **模型标签槽兼容**：keyed DSH按 provider 分别注册。旧 single DSH只渲染优先级最小的注册项；历史实测 WorkBuddy `-105`、Trae `-102`、Qoder `-101`，由 WorkBuddy 渲染。

  Qoder 被选中时，外部 provider 优先委托其他标签组件；委托失败时显示名字与目录 `model.badge`（`src/client/host-badge.tsx`）。

  互相委托可能形成递归，DSH错误处理会撤下抛错 entry；该行为属于旧槽机制的兼容限制。
- **委托翻译器**：须按 `entry.locale` 通过 `ctx.locale.bind(ns)` 为被委托组件绑定 `t`。issue #23 中，DSH `DeepSeekModelLabel`（`locale: deepseek-model-label-1017`）调用 `t("badge.label")`，缺少翻译器时抛出 `TypeError: t is not a function`，导致标签组件被撤下。

  绑定失败时改用本地后备标签。Qoder 自身 `settings.qoder` 字典仅用于本插件文案。
- **版本记录**：版本号与 CHANGELOG 按 `RELEASING.md` §4 在发布时更新。
