<p align="center">
  <img src="docs/assets/dsh-connect-qoder-usage-card.png" width="640" alt="dsh-connect-qoder 设置页：国内版 / 国际版两个 tab，账号池、积分、一键签到与自动换号" />

  <img src="docs/assets/dsh-connect-qoder-settings-page.png" width="640" alt="设置 → Qoder 页：标题、三条核心用途、活动瓦片，以及账号池（每账号一行身份 + 一行原因）" />
</p>

<h1 align="center">dsh-connect-qoder</h1>

<p align="center"><b>把 Qoder（通义灵码国内版 / 国际版）的模型接入 DeepSeek Harness，并提供账号池、每日签到、积分读数与模型管理。</b></p>

<p align="center">
  <a href="README.en.md">English</a> ·
  <a href="#安装">安装</a> ·
  <a href="#工作原理">工作原理</a> ·
  <a href="#配置">配置</a> ·
  <a href="#账号池">账号池</a> ·
  <a href="CHANGELOG.md">更新日志</a> ·
  <a href="https://github.com/dingminhua/dsh-connect-qoder/issues">问题反馈</a>
</p>

<p align="center">
  <a href="https://www.npmjs.com/package/%40dmh2002%2Fdsh-connect-qoder"><img src="https://img.shields.io/npm/v/%40dmh2002%2Fdsh-connect-qoder?style=flat-square&label=npm&color=cb3837" alt="npm version"></a>
  <a href="https://www.npmjs.com/package/%40dmh2002%2Fdsh-connect-qoder"><img src="https://img.shields.io/npm/d18m/%40dmh2002%2Fdsh-connect-qoder?style=flat-square&label=downloads&color=cb3837" alt="npm downloads"></a>
  <a href="https://github.com/dingminhua/dsh-connect-qoder/actions/workflows/ci.yml"><img src="https://img.shields.io/github/actions/workflow/status/dingminhua/dsh-connect-qoder/ci.yml?branch=main&style=flat-square&label=tests" alt="test status"></a>
  <a href="LICENSE"><img src="https://img.shields.io/github/license/dingminhua/dsh-connect-qoder?style=flat-square" alt="MIT license"></a>
  <a href="https://github.com/dingminhua/dsh-connect-qoder/stargazers"><img src="https://img.shields.io/github/stars/dingminhua/dsh-connect-qoder?style=flat-square" alt="GitHub stars"></a>
</p>

一个独立的 [DeepSeek Harness (DSH)](https://github.com/deepseek-ai/deepseek-harness) bundle 插件。它把 Qoder 账号的模型接到 DSH 的模型选择器，同时提供模型管理与只读的用量概览。**国内版与国际版是两个并行的供应商（`qoder` / `qoder-global`），可同时使用**；插件的设置页以 tab 区分两者，方便统一管理。

> Qoder 官方 SDK 仅提供 agent 启动功能，官方也未提供 OpenAI 兼容端点，因此本插件实现所需的协议客户端，使用 PAT（Personal Access Token）直接连接 cosy 网关。配置 PAT 后即可使用，无需在 Qoder 桌面端登录。

## 功能特性

- **双供应商并行接入（国内版 + 国际版）**：国内版注册为 `qoder` provider（`gateway.qoder.com.cn`），国际版注册为 `qoder-global` provider（`api1.qoder.sh`）。

  已配置 PAT 的区域模型可在 DSH 模型选择器中使用；不同会话可分别选择国内版或国际版。配置 PAT 后执行“刷新模型”即可更新目录。provider 的配置入口位于“设置 → Qoder”。

  模型名包含上游积分倍率（如 `Qwen3.8-Flash · x0.00`）。

  <img src="docs/assets/dsh-connect-qoder-model-picker.png" width="360" alt="DSH 模型选择器：Qoder 与 Qoder Global 两个分组，模型名带积分倍率" />
- **独立设置页**：配置界面位于 DSH 设置面板的“设置 → Qoder”，与“账号与余额 / 通用设置 / 模型 / 插件”并列。页面依次显示标题、功能说明和配置正文。
- **双 tab 切换**：页面顶部为「国内版 / 国际版」两个 tab，各含独立的 PAT、模型管理与用量概览；每个 tab 的目录、勾选与状态完全隔离。
- **账号 / 模型 / 诊断分栏**：每个区域分别提供三个栏目。账号栏包含账号池、积分读数开关和凭据说明，操作即时生效；模型栏包含模型选择、上下文档位、思考强度和逐模型测试，草稿须点击“保存”。切换栏目保留已加载数据和未保存草稿。诊断栏显示区域、实际请求的上游域名、服务账号、凭据来源、积分读取时间、目录模型数、已启用数、账号池状态及经脱敏的最近上游错误，供问题排查使用。
- **区域独立启停**：每个区域提供启用开关，默认开启。停用时撤销该供应商的适配器路由和“设置 → 模型”目录条目；PAT、用量与模型选择保留，重新启用后恢复。已选择停用供应商的会话再次调用时会收到 `NO_ADAPTER`。
- **模型管理**：从上游刷新目录并逐项启用或禁用模型。模型行显示倍率（`(0.50x)`）、上下文/输出上限、思考档位、免费标记，以及“图片”“可关闭思考”、上下文档位和“测试”控件。

  模型选择保存后同步到DSH选择器。列表随页面滚动，完整展示目录。上游 `auto` 为动态路由；`Fast` / `Ultimate` / `Efficient` 为未明确标识具体模型的抽象条目，均从列表过滤。

  `Sonus`（`smodel`，x8）和 `Cantus`（`cmodel`，x4）是上游提供的产品模型，带 `is_new` 标记，在国际版显示（issue #37）；国内版按其自身目录展示。

  <img src="docs/assets/dsh-connect-qoder-model-management.png" width="640" alt="模型管理区：逐模型显示倍率、最大上下文/输出、思考强度档位，以及图片/可关闭思考/上下文档位与测试按钮" />
- **活动优惠徽标**：读取模型目录的 `promotion`（文案、时段、时区），在模型名旁显示活动标签。中文界面取 `badge.zh`，英文界面取 `badge.en`；文案保留上游原文，悬停显示说明、时段、时区和活动链接。

  有时段时按渲染时刻及上游时区计算，支持跨零点；缺少时段时使用上游 `active`。无活动字段时省略标签。模型选择器通过 `LlmModelInfo.badge`（DSH 0.2.2 起支持，形状 `{ text, tone }`）展示同一信息，各 provider 分别提供目录数据。

  标签上限为 16 个 grapheme；超长时省略整枚标签，以保持原文含义。无活动时字段为 `undefined`；传入 `null` 会导致 provider 目录被拒绝。

  其他 provider 的目录 `badge` 按DSH顺序渲染：`trigger` 位置在模型名前，菜单行在模型名后。依据见 [docs/PROMO_BADGE.md](docs/PROMO_BADGE.md) 与 WorkBuddy 的 `docs/MODEL-LABEL-SPEC.md`。

  <img src="docs/assets/dsh-connect-qoder-promo-badge.png" width="640" alt="模型管理区：Qwen3.8-Max 与 Qwen3.7-Plus 两行在倍率之后、模型名之外各有一枚蓝色胶囊，文案分别是上游自己写的「错峰 4 折」与「错峰4折」" />
- **自动刷新间隔（能力栏）**：全局配置 `pollMinutes` 控制设置页和输入框用量读数的刷新间隔，可选 1 / 5 / 10 / 30 / 60 分钟，默认 5 分钟。两个区域和两个界面共用此配置；同一路由的并发读取合并为一次上游请求。修改即时生效。
- **模型可用性测试**：每个模型行提供“测试”按钮，通过真实上游请求区分限流（429）、凭据无效、内容审核拒绝、额度耗尽和瞬时故障。探测由后台执行，凭据保留在 Host，结果独立于设置和已保存的模型选择。测试会消耗少量上游额度，模型列表提供逐项手动测试。
- **每日签到**：个人用户签到活动的历史规则为每日 100 Credits、有效期 30 天，实际奖励以活动数据为准。“一键签到所有账号”逐账号查询后领取；已领取的活动直接按成功处理，完成后刷新积分。国内版与国际版均已接入；国际版需要本机 Qoder 客户端的设备风控组件，缺失时显示相应错误。
- **每天自动签到（可选，默认关闭）**：`regions.<region>.pool.autoCheckin` 控制后台签到，仅作用于池成员。每个活动窗口先读活动状态并检查 `claimStatus`，`CLAIMED` 直接记录为已领取，其余符合条件的活动再发起领取。

  窗口优先依据活动 `endAt`，后备分界为 10:00（UTC+8）。失败按 30 秒 → 1 → 2 → 5 → 15 分钟退避，之后每 30 分钟重试至窗口结束。

  开关旁显示完成数、待重试数与最近失败原因。
- **活动展示（只读，可关闭）**：各区域在额度读数下显示上游活动，包括每日签到和 `VIEW_DETAILS` 推广活动。标题、描述按界面语言选择 `zh` / `en`，缺少对应语言时使用另一语言；详情链接取 `detailUrl`，缺失时使用活动详情页。

  网格采用 `auto-fill` 与 260px 最小列宽，描述最多两行，悬停显示全文，状态及详情链接底部对齐；历史测量中，612px 正文栏内三条活动约占 180px。

  “显示活动”对应 `regions.<region>.showCampaigns`，默认开启；关闭后省略活动区及其 `/campaigns` 请求。空列表显示“当前没有活动”，请求失败显示原因。

  2026-10-25 的用户截图确认旧实现遗漏 banner，后续版本已读取完整活动列表。
- **积分查询（只读）**：账号池逐账号显示剩余积分、积分到期估算及套餐到期时间。用量通过 Qoder 的 `sash/api/v2/me/usage` 查询，优先使用交换后的令牌；查询本身不消耗积分。3 天内到期的积分高亮；认证失败或上游无响应时显示原因。
  - **两个到期日是两件事**（issue #31）：`qoderUsage.expiresAt` 是**账号级套餐 / 14 天试用**到期（实测它等于 `userId` 前缀 + 14 天），因此单独显示为「套餐 {at} 到期」；积分包的临期高亮和账号池排序仍依据积分到期信息。
  - **积分到期估算**：领取签到奖励时记录 `claimAtMs`、数额和 `validity.days`（历史规则为 30 天），保存在 `$DSH_HOME/.qoder-pool.<region>.json` 的 `credits`。插件按 FIFO 记账，显示最早未过期批次的保守估算；上游未说明消耗顺序，因此估算可能提前。缺少领取时间的旧记录省略到期估算，界面标注“最早一批签到积分约 {at} 到期”。
- **安全 loopback shim**：每区域一个随机端口 + 进程内随机 secret，真实 Qoder PAT **不交给 pi-ai**：pi-ai 只与本地 shim 对话，shim 用 cosy 会话（jobToken）做上游请求。
- **多账号池（PAT）**：每区域一个本地 vault `$DSH_HOME/.qoder-accounts.<region>.json`（0600），把设置 PAT、环境变量与 vault 账号三源去重合并；卡片可新增（粘贴 `pt-…`，经 `JobToken` 交换校验并取显示名，也可自填备注名）/ 改名账号，也可显式选择某一账号服务（`regions.<region>.accountId`，留空自动按 settings → env → vault 选取）。

  **移出池**与**删除**是两件事：移出池 = 在池表格里取消勾选并保存（凭据留着，随时勾回）；删除 = 池表格行内的「删除」按钮，二次确认后抹掉 vault 里的 PAT 并自动退出池与区域选择（不可恢复；设置/环境变量里的 PAT 不归本插件管，不提供删除按钮）。
- **实测排序与失败接管**：“一键测试所有账号”对池成员发真实请求，默认选取本区域倍率为 0 的免费模型，结果保存至 `$DSH_HOME/.qoder-pool.<region>.json`。

  首发使用当前选中的账号；开启账号池后，可在首字节前按 `rankPool` 选择接管账号，优先级为可用性 → 临期积分 → 用户手动顺序 → 积分到期与金额 → 余额 → id。

  失败记录为 `live-request` 测量。关闭账号池时直接报告错误。业务码 110 / 113 / 118 等配额错误会在区域卡片显示“上游报额度耗尽”，包含时间、业务码和脱敏原文，与账号池开关无关。

  该通知补充了 0.7.x 移除“今日额度疑似已用尽”胶囊后的错误展示。HTTP 429 短时限流单独归为冷却状态。
- **账号转移（导入 / 导出）**：按区域把 vault 账号导出为传输文件，预览后按索引导入；跨区域域名（`qoder.com.cn` / `qoder.sh`）正向匹配，不匹配的记录计入 `skipped`。
- **思考强度与上下文预算**：档位取自上游 `thinking_config.enabled.efforts`，例如 Qwen 系 `low/medium/xhigh`、DeepSeek/GLM 系 `low/high/max`。

  选择保存后，请求按 Qoder 客户端格式双写顶层和 `parameters` 字段。上下文选项取自 `context_config`。模型选择器默认使用上游默认档（如 Qwen3.8-Max 的 `medium`）；用户保存的模型档位优先。
- **模型能力核验**：图片默认值取自按精确 model id 维护的核对表，依据为厂商文档和实机测试；未核实模型默认不勾选。上游 `is_vl` 表示平台声明，仅据此无法确认模型实际读图能力。2026-10-29 使用含数字 `7` 的图片测试两区域目录并集的 14 个模型，均识别成功，修正了早期按模型命名推断的纯文本结论。“可关闭思考”仅向上游显式声明且未列入已知拒绝集的模型提供。
- **缓存命中统计**：上游 `usage.prompt_tokens_details.cached_tokens` 透传至DSH，供 GUI 计算轮次命中比例。该值包含在 `prompt_tokens` 内，总数按子集关系计算；上游未提供时保留缺失状态。历史修复前该字段在归一化过程中丢失，界面恒显示 0%。
- **会话积分读数（位置可选）**：按区域显示 `QD CN` / `QD AI` 和当前计费账号的余额，例如 `QD CN · 92`。账号池接管后显示接管账号的余额，悬停说明归属；点击打开逐账号余额表，点击表格行可切换账号。

  国际版的 SOTA 专属积分包单列显示在“积分”列左侧，仅在使用绑定模型时优先抵扣，独立于普通积分。缺少专属包的账号显示“—”；全部账号均无专属包时省略整列及表头。

  每个区域通过“在对话框显示积分”开关控制，默认开启。位置可选输入框工具栏（`conversation.input.right`，模型选择器左侧，与权限切换器分隔）或输入框下方（`conversation.composer.dock`，追加至轮次、tok/s、token 和缓存命中统计之后）。共享工具栏空间不足时，可切换至输入框下方（参照 workbuddy issue #62）。

  余额表的“设置”按钮优先调用设置外壳的状态句柄，打开“设置 → Qoder”。旧DSH依次尝试侧边栏设置入口与导航行，最后使用插件市场本插件页。

  <img src="docs/assets/dsh-connect-qoder-composer-credits.png" width="360" alt="对话输入框的积分面板：QD CN 逐账号余额表、设置按钮与最近刷新时间" />
- **命令行诊断与现场验证**：`dsh-connect-qoder doctor|status|logout [--json] [--region cn|global]`：`doctor` 报 PAT 来源、cosy 指纹派生、pi-ai 版本及对应的数据格式、DSH 进程运行状态存活与 vault 账号计数；`status` 报凭据来源、上游可达性与当前剩余积分；`logout` 清除指定区域的 vault 账号与账号池测量副本。

  另有 `pnpm run probe:cosy`（不依赖DSH直连 cosy 请求过程验证）与 `pnpm run probe:vault`（只读 dump vault / 账号池测量，PAT 只显示掩码）。

## 工作原理

```text
DSH PiAiAdapter（每个 provider 一套）
  -> 安全 loopback shim（每区域一个随机端口 + 进程内随机 secret）
  -> QoderUpstreamClient（cosy 协议最小客户端）
  -> 国内版 https://gateway.qoder.com.cn/algo/api/v2/...
  -> 国际版 https://api1.qoder.sh/algo/api/v2/...
  -> Qoder SSE（OpenAI 形状增量）
  -> DSH 本地执行工具并回传结果
```

国内版与国际版分别初始化上游客户端、模型目录、回环 shim 和 adapter，并各自使用对应的网关、openapi 与目录路径，因此**两个区域的模型可同时出现在选择器中，供不同会话独立使用**。

凭据是 PAT（设置页 `regions.<region>.pat` 或环境变量，见[配置](#配置)），设置项优先。每次会话按需惰性交换 **jobToken**：PAT 只在这个交换请求里使用一次，之后的对外请求全部携带 cosy bearer（自定义 Base64 载荷 + MD5 签名）与「机器指纹」会话头。**PAT 从不进入 pi-ai，也从不外发到网关之外；本插件不读不写 Qoder 桌面端的本地登录文件。**

模型的 cosy 编码与签名细节（自定义字母表、`n/3` 重排、`cosy&<SECRET>&<date>` 签名、RSA/AES 会话封装）见 `src/cosy.ts` 的头部注释；错误分类（内容审核 / 限流 / 额度耗尽 / 瞬时故障）与 DSH 重试语义的对应见 `src/upstream.ts`。

## 安装

### 从发布包安装（推荐）

```bash
dsh plugin --profile desktop add @dmh2002/dsh-connect-qoder   # Desktop
dsh plugin --profile web add @dmh2002/dsh-connect-qoder       # Web
```

装完**重启该 profile 的 DSH 进程**（bundle 的 host 半边在启动时加载）。也可以在 DSH 的「插件市场 / Bundles」里搜 `dsh-connect-qoder` 一键安装。

> **`dsh` 命令报 `Cannot find module .../DSH NEXT.app/...`？** 那是DSH换代留下的旧 shim：DSH app 目录名已从 `DSH NEXT.app` 变为 `DeepSeek Harness.app`，而 `~/.local/bin/dsh` 仍指向被移除的旧路径。把 shim 改指新DSH自带的 CLI 入口即可：
>
> ```bash
> exec "/Applications/DeepSeek Harness.app/Contents/Resources/runtime/cli/bin/dsh" "$@"
> ```
>
> 新入口内部以 `ELECTRON_RUN_AS_NODE` 运行 `app.asar` 里的 `dsh/lib/bin.js`，profile 仍由 `--profile` 决定。

> npm 包名带 `@dmh2002/` 作用域。插件在 DSH 内的设置命名空间、`/plugins/dsh-connect-qoder/…` 路由与命令行程序名均沿用**不带作用域的配置标识** `dsh-connect-qoder`，所以 0.2.0 之前装过的用户升级后配置照旧生效，不用重填 PAT。设置页的寻址也不再依赖包名：它是 `settings.section`，键是**不带作用域**的 entry id。但**浏览器 bundle 的模块 id、模型选择器标签与「设置」按钮退回插件市场时用的包名**仍按带作用域的包名建，这几处由构建期从 `package.json` 注入，改包名不会漏下它们。

### 从源码构建（开发）

```bash
git clone https://github.com/dingminhua/dsh-connect-qoder.git
cd dsh-connect-qoder
pnpm install
pnpm run check        # typecheck + test + build 全部通过后再安装
```

以 `link:` 安装进当前 profile，修改并构建后重启 DSH 即生效（不会重装依赖树）：

```bash
dsh plugin --profile desktop add ~/DshProject/dsh-connect-qoder
```

### 验证安装

- 打开「设置」（侧边栏设置按钮，或 `Cmd/Ctrl + ,`）→ 左侧导航最下方的 **Qoder** → 应看到配置页（国内版 / 国际版两个 tab）。
- 在该 profile 目录里 `pnpm exec dsh-connect-qoder doctor` 应输出两个区域的 PAT 状态与指纹派生。
- 配置 PAT 后，「从 Qoder 刷新」应拉出该区域的真实模型清单；模型选择器出现 `Qoder` / `Qoder Global` 两个分组。

## 配置

### PAT（Personal Access Token）

在 Qoder 账户设置里创建 PAT（`pt-…` 开头）。两种提供方式，**设置页优先于环境变量**：

| 区域 | 发证入口 | 设置项 | 环境变量 |
| --- | --- | --- | --- |
| 国内版（qoder.com.cn） | 控制台 → 设置 → API 令牌 | 卡片「国内版」tab → 添加 PAT | `QODERCN_PERSONAL_ACCESS_TOKEN` |
| 国际版（qoder.sh） | 控制台 → Settings → Personal Access Tokens | 卡片「国际版」tab → 添加 PAT | `QODER_PERSONAL_ACCESS_TOKEN` |

- PAT 保存在本地配置、环境变量或 vault，并经 HTTPS 发送至 Qoder 令牌交换接口；用量查询的兼容回退路径也可能使用 PAT。
- 未配置 PAT 的区域不列出任何模型；`doctor` 命令会指出哪个区域缺 PAT。
- **逐步操作说明见 [`docs/PAT.md`](docs/PAT.md)**（含创建流程、验证机制、优先级与常见问题）。

### 区域启停

卡片每个 tab 右侧的勾选框控制该区域是否参与模型选择（opt-out：**显式取消勾选才停用**）。停用只摘除路由与目录条目，配置全部保留。

### 侧边栏面板（可选，默认关闭）

卡片「账号」栏里有一行「在侧边栏显示 Qoder」：勾上之后，DSH 侧边栏会多一个 Qoder 图标行，点开就是**这张卡片的整页版**（同样的账号池、模型目录、诊断栏，只是占满中央那一列），页头带一个「返回会话」。

- **默认关闭**。启用后通过DSH `sidebar.panellist` 增加 Qoder 导航项，点击可打开中央面板。原有设置页仍可访问相同功能。
- **点一下即生效，不用保存**。关掉时如果整页正开着，中央那一列会自动回到对话（DSH会丢掉 `main` key 已不存在的那次选中）。
- 保存为顶层配置 `sidebarPanel`，适用于整个插件。即使两个区域均停用，仍可打开面板添加 PAT。
- 判据是 `=== true`：缺席、`false`、以及被手工改成字符串 `"true"` 的垃圾值，一律读作关闭。

### 多账号池与凭据来源

除设置 PAT 与环境变量外，每区域还有一个本地 vault（`$DSH_HOME/.qoder-accounts.<region>.json`，权限 0600）保存多个 PAT 账号。三源按 **settings → env → vault** 的优先级去重合并（同一个 PAT 只在最高优先级的来源里列出一次）。

- 卡片可新增（粘贴 `pt-…`，经 `JobToken` 交换校验并取回显示名，也可自填备注名）/ 改名 vault 账号，也可显式选择某一账号服务（`regions.<region>.accountId`，留空自动）。删除 vault 账号在池表格的行内「删除」按钮（需二次确认）；取消勾选并保存只是把它移出轮换，PAT 仍留在本地。
- Host 使用选中的 PAT 交换令牌，并在用量查询兼容回退路径中使用 PAT。pi-ai 使用本地 shim 凭据；设置页仅接收已保存凭据的脱敏信息。
- `logout [--region cn|global]` 会清除指定区域的 vault 账号与账号池测量副本（`$DSH_HOME/.qoder-accounts.<region>.json` 与 `.qoder-pool.<region>.json`）。
- 勾选成员、一键签到、一键测试与自动换号的规则见[账号池](#账号池)。

## 账号池

**账号池**用于管理多个 Qoder 账号。勾选成员后，可执行“一键签到所有账号”或“一键测试所有账号”；开启“启用账号池”后，可恢复请求失败将触发自动接管。

**国内版与国际版各有独立账号池**，分别维护成员、目标模型、积分和签到状态。两个区域均支持每日签到；国际版请求需要设备风控头，插件从本机 Qoder 客户端的设备身份组件读取所需信息，读取失败时显示对应错误。

### 失败接管（开启账号池即生效）

- **首发账号**：使用当前选中的账号。
- **接管顺序**：开启账号池后，首字节前发生可恢复失败时，按排序依次尝试其他可用成员；全部候选失败后返回错误。
- **可接管的失败**：429 限流、额度耗尽、401/403 凭据被拒、5xx 网关与网络故障。
- **直接返回的失败**：请求无效（HTTP 400），或流式内容已经开始输出。
- **成员可用性**：限流冷却中的账号及凭据已被拒绝的账号被跳过；未测试成员默认可用。
- **账号选择保存**：接管仅修改运行时服务账号，保留用户保存的账号选择。
- **失败记录**：将失败及上游恢复时间写为账号测量，供后续接管判断使用。
- **接管提示**：回答开头显示引用块，例如 `> [dsh-connect-qoder] 「acct-demo」被限流，已改用池中另一个账号回答。`。提示说明触发接管的失败账号；连续接管时保留紧邻成功接管前的一次失败提示。

### 账号池开关的作用

账号池开关控制失败接管；批量操作按所选成员执行。

| | 账号池 **开启** | 账号池 **关闭** |
| --- | --- | --- |
| 首发账号 | 当前选中的账号 | 当前选中的账号 |
| 请求失败 | 首字节前按顺序选择可用成员接管 | 直接返回错误 |
| 一键签到 / 一键测试 | 可用 | 可用 |
| 目标测试模型 | 可修改 | 可修改 |

批量签到与测试作用于勾选成员。成员集合为空时拒绝执行，以确保操作范围与用户选择一致。

### 重试预算可以调（`regions.<region>.retry`）

DSH对可重试请求执行退避重试。插件声明等待阶梯，由DSH执行；配置缺省时采用DSH默认值。

Qoder 的 `RESOURCE_LIMIT`（业务码 `10605`）可能包含等待时长。插件在交给DSH的错误消息中保留该值，例如 `[upstream retry-after 5s]`。

重试参数按区域配置，前三项控制DSH退避预算，第四项控制插件等待上限：

```yaml
regions:
  cn:
    retry:
      maxRetries: 4        # 最多重试几次（0 = 跟随DSH默认）
      initialDelayMs: 5000 # 首次退避毫秒（0 = 跟随DSH默认）
      maxDelayMs: 30000    # 退避上限毫秒（0 = 跟随DSH默认）
      queueWaitMs: 60000   # 排队时插件自己等的上限（0 = 关掉，默认 60000）
```

上面的 `5000 → 30000` 兼顾了上游两种常见形态：瞬时限流（申报 5 秒）与 `queueType:p2` 排队（约 30 秒）。**前三个 0 表示跟随DSH默认**，所以留空与不写这个配置等价；第四个 `queueWaitMs` 的 0 则是**关掉**（见下节）。

> 此前仅支持DSH默认预算；直接修改 `node_modules` 内的构建产物会在依赖升级时被覆盖。

#### 插件排队等待（`queueWaitMs`，issue #29）

DSH按递增阶梯（5s → 10s → 20s → 30s，带抖动）执行退避，早期重试间隔可能短于上游申报的 30 秒。历史实测一轮包含 28 次重试、296 秒纯等待，其中 11 次在上游要求等待 30 秒后不足 5 秒即重发。

issue #24 复核确认，`pi-ai` 路由缺少 `providerRetryAfterMs` 通道，模块与底层 vendor 均未解析 `retry-after`。插件在转发层按上游申报时长等待，使该次等待保留DSH重试预算：

- 上游回 `isQueued: true` 并申报了等待秒数，且该值 **≤ `queueWaitMs`** ⇒ 插件等满这个时长再重发一次，这次失败**不交给DSH**；
- 申报值超过上限、或重发后仍排队 ⇒ 原样交回DSH，此时它至少还有剩余预算；
- 每个请求**只等一次**，等待可中断（用户点击停止时中断等待）；
- 配 `queueWaitMs: 0` 关掉，交由DSH执行退避。

#### 排队状态分类（issue #29）

同一个 `isQueued: true` 有两种形态，插件的措辞与账号池的处理不同：

| 上游字段 | 含义 | 插件怎么处理 |
| --- | --- | --- |
| `isQueued:true` + `serviceAvailable:true` | 服务可用，请求正在排队（`queueCount` 通常 > 0） | 记为 **排队中**，按上游等待时长冷却，可尝试其他账号 |
| `isQueued:true` + `serviceAvailable:false` | 上游整体服务不可用（实测 `queueCount: 0`） | 记为 **上游整体不可用**，换号提示明说「不是该账号的问题」 |

服务可用时，等待或更换账号可能恢复请求；整体不可用时，更换 PAT 无法解决服务故障。历史实测曾有 14 个模型同时不可用，因此插件分别显示排队与整体不可用状态。

### 排序规则（账号池失败接管顺序）

1. **可用优先**：被测量判定为不可用（限流未到恢复时间、凭据被拒等）的排到最后；
2. **有临期积分的优先**：优先使用 3 天内到期的积分（临期判定优先于下面第 3 条的用户手排顺序，见 `rankPool` 的注释）；
3. **用户手动顺序优先**：按 `orderAccountIds` 保存的拖拽顺序排列；
4. **积分到期最近 → 临期金额大 → 积分高**：只在有**新鲜读数**时参与（卡片轮询、批量测试或保存时读到的那份，10 分钟内有效）：请求路径仅使用已有读数，避免为排序增加上游请求；
5. 按账号 id 排定剩余顺序，保证结果可复现。

**套餐到期与积分到期分别处理**（issue #31）：套餐日期表示账号套餐或试用期的结束时间，独立展示；账号池的临期排序仅依据积分批次的到期时间。

### 两个批量动作

- **一键签到所有账号**：逐个账号领取当日的官方签到活动。**先查状态再领取**，已领过的直接标「已签到」而**不发写入请求**（已领取的活动跳过领取请求）；任何单个账号失败只标记该行，**不中断整批**。签到会改变余额，因此批次结束后作废该区域的积分缓存，卡片不会继续显示领取前的数字。
- **一键测试所有账号**：对每个池成员发起真实探测。默认自动选择本区域倍率为 0 的免费模型；多个候选中选上下文窗口最大者，也支持显式指定目标。缺少可用免费模型时，自动目标测试保持关闭。探测与模型行测试采用相同实现，可能消耗上游额度。

  **测试目标独立于已启用模型集合**（issue #33）。卡片指定的目标与自动选出的免费模型均可用于测试，即使对应模型未勾选为 DSH 可用模型。历史实现中，卡片读取完整目录而测试路由读取裁剪后的运行时目录，曾导致卡片显示 `qfmodel` 而接口返回空结果；现测试目标解析使用完整目录。

  目标无法解析时（目录缺少免费模型，或指定模型已下架），接口返回 `409` 及 `no-free-model` / `target-model-stale`，卡片显示“没有可测的目标模型”。

批量动作执行时，按钮显示“签到中… / 测试中…”。每个账号完成后立即更新对应行和活动日志，日志时间戳取该账号的完成时刻，其余界面保持可用。保存配置与删除账号期间显示忙碌遮罩。

设置页顶部显示国内版和国际版两个供应商卡片，各自提供边框、底色和启用开关，选中区域使用品牌色描边。点击名称切换 tab，点击开关启用对应 provider。

下方依次显示“在对话框显示积分”“显示位置”和“在侧边栏显示 Qoder”，随后为账号池与模型管理。三项设置整行可点击、标题下显示说明，修改后立即生效：前两项控制会话积分读数，第三项控制侧边栏入口及其页面。

自动签到默认关闭，手动签到通过账号池按钮执行。

> 1.24.0 起支持可选自动签到（issue #36）。开启后，后台按活动窗口为池成员查询并领取奖励；已领取状态直接记录。签到本身不消耗积分。

### 上游未提供恢复时间时的处理（issue #32）

恢复时间的来源决定显示方式：

| | 意思 | 谁决定的 |
| --- | --- | --- |
| `retryAtMs` 有值 | 「上游说这个时刻之后可以再用」 | 上游的字段、报文里的重置句，或按签到快照的当日窗口**推算**（`retrySource` 区分这几种） |
| `retrySource: 'unknown-cooldown'` | 上游与快照均缺少恢复时间，插件按 30 分钟安排重试 | 插件 |

当 `retryAtMs` 缺失时，仅凭该字段无法区分“没有重试计划”与“按本地间隔重试”。后备冷却分支以 `retrySource: 'unknown-cooldown'` 标记来源，卡片显示“上游未给出何时恢复 · 每 30 分钟自动重试一次”。该分支保留 `retryAtMs` 为空，使其继续专用于上游声明或根据窗口推算的恢复时刻。

后备冷却间隔保持 30 分钟。上游仅说明当日用量耗尽后可在次日继续，具体恢复时刻优先由签到活动窗口提供。

#### 签到窗口持久化与自动签到（issue #31 / #32 / #36）

`110`（当日用量限制）的恢复时刻来自签到活动行的 `endAt`。签到成功后，窗口时刻写入 `$DSH_HOME/.qoder-pool.<region>.json` 的 `windows` 字段。重启后优先读取进程内快照，其次读取磁盘；已过期的时刻予以忽略。缺少有效窗口时刻时，使用约 30 分钟的后备重试间隔。

自动签到通过 `regions.<region>.pool.autoCheckin` 开启，默认关闭。进度写入 `$DSH_HOME/.qoder-auto-checkin.<region>.json`，文件权限为 `0600`，按活动窗口记录。文件仅含账号 id、计数和失败原因。运行状态独立于 settings 保存，避免用户保存偏好草稿时覆盖后台签到进度。

Qoder 的每日窗口从当天 10:00 延续至次日 09:59（UTC+8）。用户本地零点可能位于同一活动窗口内，因此按日历日记录会导致提前重复检查，并延迟领取下一窗口奖励。例如，UTC+8 的 00:05 若被记为新一天已完成，10:00 的新窗口可能延后至次日 00:05 才被检查，相差 14 小时。

窗口标识优先使用仍在未来的活动 `endAt`；缺少该值时，使用“当前时刻减 2 小时后的 UTC 日期”，对应 10:00（UTC+8）的分界。两种标识分别以 `upstream:` 和 `boundary:` 为前缀，来源变化后重置进度并重新查询领取状态。`summaryForWindow` 仅汇报当前窗口的进度，避免在窗口切换后显示上一窗口的完成数。

## 命令行诊断

在**已装本插件的 profile 目录里**跑（推荐，无需全局安装）：

```bash
cd ~/.dsh/profiles/desktop
pnpm exec dsh-connect-qoder doctor    # PAT 来源 + cosy 指纹派生 + pi-ai 版本及对应的数据格式 + 心跳 + vault 计数（纯本地，不发请求）
pnpm exec dsh-connect-qoder status    # 每区域凭据来源 + 上游模型目录可达性 + 剩余积分（需 PAT）
pnpm exec dsh-connect-qoder logout --region cn   # 清除指定区域的 vault 账号与账号池测量副本
pnpm exec dsh-connect-qoder doctor --json        # 机器可读输出
```

`status` 会真实交换一次 jobToken 并拉取模型目录（随后只读查询一次用量），因此需要对应区域的环境变量或 vault PAT；`doctor` 完全离线。

> **两条实测过的注意事项**：
> 1. **CLI 入口使用作用域包路径**：0.3.0 起包名带作用域，对应软链位于 `node_modules/@dmh2002/dsh-connect-qoder`，旧路径会返回 `MODULE_NOT_FOUND`（缺的是**入口文件**，不是 peer）。
> 2. CLI 依赖DSH提供的 peer 包（`@deepseek-ai/dsh-atomic-write` / `dsh-home-paths`）。本机实测：pnpm 会随包把它们装进 profile 的依赖树，`pnpm exec` 与 npm 装的 tarball 均可正常执行。依赖能否自动安装**取决于 pnpm 版本与 `auto-install-peers` 配置**；运行时报 `Cannot find package '@deepseek-ai/dsh-atomic-write'`，表示 CLI 缺少所需依赖包；设置页与 provider **不受影响**（它们在DSH进程内，DSH自己解析这些名字）。同族 workbuddy 的 issue #47 属于同类问题，其修复方案（优先使用 DSH 提供的依赖包，缺少时使用插件自带实现）适用于本项目，尚未移植。

## 平台支持

| 平台 | 状态 |
| --- | --- |
| macOS | ✅ 已验证（开发与 CI） |
| Windows | ✅ CI 测试通过（上游请求走标准 fetch，无平台专属路径） |
| Linux | ✅ CI 测试通过 |

## 受支持的DSH范围

| DSH | peer 范围 |
| --- | --- |
| DSH（`@deepseek-ai/*`） | `>=0.1.7-rc.1 <0.2.0-0 \|\| >=0.2.0-rc.1 <0.3.0-0 \|\| >=0.2.1-rc.1 <0.3.0-0 \|\| >=0.2.2-rc.1 <0.3.0-0` |
| `@earendil-works/pi-ai` | `>=0.85.0 <0.88.0 \|\| >=1.0.0 <2.0.0` |
| Node.js | `^22.19.0 \|\| >=24.0.0` |

> 范围由 `tests/dsh-line.spec.ts` 强制。DSH 至今只发预发布，而 node-semver 只有当范围里*某个*比较符与该版本的 `major.minor.patch` 元组完全一致、且自身也带预发布标签时，才会放行预发布版本。所以每开一条新线就得加一条分支：只写 `>=0.1.7-rc.1 <0.3.0-0` 会**静默排除整条 `0.2.0-rc.*`**（`0.2.0-rc.2` 曾是 `latest`），同理 `>=0.2.0-rc.1` 这一支**不含 `0.2.1-rc.1`**（桌面端 0.2.1-rc.1 已发布）。上界仍须带 `-0`（`<0.3.0` 不挡任何东西）。
>
> **两套语义并存**（1.1.2 实测更正）：npm / pnpm 安装走 node-semver **标准**语义，新补丁号的 rc（`0.2.2-rc.1` 已于 2026-11-05 补齐分支，下一个待补的是 `0.2.3-rc.1`）仍需手工追加匹配分支，待补版本记录在 `UNGATED_PRERELEASES` 中。但**DSH的插件兼容性检查不是标准语义**：`dsh-app-boot` 用 `semver.satisfies(v, range, { includePrerelease: true })`，在这个语义下 `>=0.2.1-rc.1 <0.3.0-0` 一支就覆盖**整条 `0.2.x` 线**（`0.2.2-rc.1`、`0.2.9` 都放行），`0.3.0-rc.1` 仍被上界拦住。1.1.0/1.1.1 曾断言「没有任何静态范围能覆盖任意 `0.2.x` 预发布」，那句话只对 npm 成立，对DSH兼容性检查不成立，已更正。GA 版本（如 `0.2.1`）对既有分支本来就判 true。
>
> **pi-ai 单独列一行**是因为它换了主版本：0.2.1-rc.1 DSH 将 pi-ai 从 0.85 换到 1.x，而两代的上下文数据格式不同（1.x 在 Models 层归一化）。声明与检测必须一致，否则会出现 1.1.0 修复的工具字段丢失问题，详见 [`docs/DSH-0.2.1-RC1-FOLLOWUP.md`](docs/DSH-0.2.1-RC1-FOLLOWUP.md)。

### 上一轮DSH基线：0.2.2-rc.1（2026-11-05 实测）

DSH升到 `0.2.2-rc.1`（`app.asar` 内 `@deepseek-ai/dsh-*` 均为 `0.2.2-rc.1`，内置 pi-ai **`1.3.0`**、cordis `4.1.0`、schemastery `3.19.0`）。这一轮**需要适配**，与 0.2.1 线的「零改动」不同：

| 检查项 | 结论 |
| --- | --- |
| `ResolvedPiAiProviderProfile` | 新增必填 `maxRequestImages` / `imageOffloadCountQuantum`（单请求图片条数上限，DSH默认 50 / 10）。插件手工拼 profile，DSH的默认化不会跑 → 已显式补齐 |
| pi-ai 1.3.0 的 api 入参 | 改为品牌化的 `TranscriptContext`（只有 `normalizeContext()` 能产出）。只影响 legacy（0.85）回填分支，已在该分支边界加断言，并注明 1.3.0 DSH走不到这里 |
| `ctx.locale.register` | 签名加宽成 `Record<BuiltInLocaleId, …> & Record<LocaleId, …>`，实参需要 string 索引签名 → 已给 `QoderLocale` 加索引签名（双语仍合法，缺的语种使用英文） |
| peer 范围 | 补 `\|\| >=0.2.2-rc.1 <0.3.0-0` 一支；在此之前 npm 标准语义判 false、DSH兼容性检查一直判 true |
| modern 分支直接传递上下文 | **无回归**：真实 1.3.0 栈上提示词与工具都在（pi-ai 新版本测试环境 升到 1.3.0 后 4 条直通用例全部通过） |
| 中途 system 消息 | 0.2.2 起默认改写成 user 角色的 `<system-reminder>`（除非模型声明 `supportsMidConvoSystemMessages`）；Qoder 路由目前取默认 |
| 插件用到的槽位 / 事件 | `conversation.input.*`、`conversation.composer.dock`、`settings.section`、`plugins.*.config` 接口约定保持兼容；`webserver/index-inject` 的 `peer` 参数与 `WebResourceRoute` 都是新增，插件不涉及 |
| 回归 | `pnpm run typecheck`（host + client 两套）与 `pnpm test` 全部通过 |

> 0.2.1-rc.4 / rc.5 那一轮的逐条核查保留在 [`docs/DSH-0.2.1-RC1-FOLLOWUP.md`](docs/DSH-0.2.1-RC1-FOLLOWUP.md)；rc.1 那一轮的实测依据、复现脚本与仍待决策项见 [`docs/DSH-0.2.2-RC1-FOLLOWUP.md`](docs/DSH-0.2.2-RC1-FOLLOWUP.md)。

### 当前 DSH：0.2.2-rc.2（2026-11-12 实测）

DSH 升至 `0.2.2-rc.2` 后，插件运行时代码保持兼容，其中 pi-ai（`1.3.0`）、cordis（`4.1.0`）、
schemastery（`3.19.0`）版本均保持不变，`dsh-*` 的改动也保持接口兼容：

| 检查项 | 结论 |
| --- | --- |
| 失败分类器 | 🔴 DSH 将 `classifyPiAiError` 改成「先取报文开头的状态码」（新增常量 `LEADING_HTTP_STATUS`）→ DSH 分类器代码样本**须重新提取**，已改为一条命令 `pnpm run fixture:host-classifier` |
| 分类结果 | **行为不变**：插件的净化一直把独立出现的 401/403 改写成 `4xx` 并加 `qoder upstream <kind> (http …)` 前缀，新判据读到的是同一个结论（逐条对比见 followup 文档 §二） |
| `dsh-llm-pi-ai` | 新增可选的请求前缀摘要（`PrefixDigests`）与 `onThinkingOmitted` 观察点；插件不构造 replay、只用于 Anthropic 的 signed thinking → 未触及 |
| `HarnessError` / `AttachmentError` | `HarnessError` 搬进独立的 `@deepseek-ai/dsh-error`；插件从不使用错误基类，只 `import type { AttachmentStore }` |
| 槽位 | `dsh-client-ui-slots` 新增 `hoc` 种类、`conversation.input.model.options` 等新槽；插件用到的六个槽接口约定保持兼容（`settings.launcher` 的 `openSettings` 只多了可选形参，插件也不注册它） |
| peer 范围 | **不需要新分支**：`0.2.2-rc.2` 与 `0.2.2-rc.1` 同一元组，第四个版本范围分支已覆盖该版本。下一个要补的仍是 `0.2.3-rc.1` |
| 回归 | `pnpm run check` 全部通过（1472 例通过 / 12 跳过） |

> 依据、逐包 diff 与兼容性核对清单见 [`docs/DSH-0.2.2-RC2-FOLLOWUP.md`](docs/DSH-0.2.2-RC2-FOLLOWUP.md)。

## 安全说明

- PAT 由插件 Host 保管，经 HTTPS 用于 Qoder 令牌交换；用量查询保留 PAT 兼容回退。聊天请求使用 cosy 签名会话与机器指纹头，pi-ai 使用本地 shim 凭据。
- 本地 shim 用随机端口的 loopback 服务 + 进程内 secret 鉴权，并校验 Host / Origin / Content-Type，请求体有 64MB 上限。
- 本插件使用用户显式提供的 PAT；**不读取、不修改** Qoder 桌面端的本地登录/凭据文件。
- 诊断输出（`doctor` / `status` 与卡片探测）均脱敏，不打印任何令牌样式的字符串。

## 市场收录与展示

本插件声明了可安装的 `dsh.bundle` manifest（`dsh.bundle.patch: ./cordis.patch.yml`），因此可以用 `dsh plugin add` 安装。

社区市场通常从 [Awesome DSH Plugin](https://github.com/awesome-dsh-plugin/awesome-dsh-plugin) 注册表同步条目；仓库内的提交草稿位于 [`awesome-dsh-plugin-submission/dingminhua__dsh-connect-qoder.yml`](awesome-dsh-plugin-submission/dingminhua__dsh-connect-qoder.yml)。

**收录目录规范**（依据 [contributing.md](https://github.com/awesome-dsh-plugin/awesome-dsh-plugin/blob/main/contributing.md)）：

- 每个插件一个 YAML：`data/plugins/<owner>__<repo>.yml`，字段为 `url`（与仓库地址完全一致）、`name`（`owner/repo`）、`category`（有效值之一，本项目为 `model`）、`description.en`（必填，以句号结尾）/ `description.zh`（可选）；描述中含 `: ` 必须加引号。
- 仓库需声明 `dsh.bundle` manifest、被 `dsh-plugin` topic 标记、创建满 1 天且处于活跃维护；描述须准确、客观，并在投稿时逐项对照代码核查。
- 注册表的两个 README 由脚本生成，不得手工编辑；修改 YAML 后在其仓库执行 `npm ci` + `node scripts/generate-readme.mjs` 重新生成。
- 一个 PR 最多收录 3 条；只改自己的条目。

市场卡片中的截图与 GitHub README 徽章是两套机制：

- **市场截图**按当前约定由**插件自己仓库根目录的 `screenshots.json`** 声明（本项目声明了 [screenshots.json](screenshots.json)）；注册表的 `data/screenshots.json` 是旧约定下的回退，**不再添加新键**。
- **GitHub 徽章**由本 README 顶部的 Shields 图片链接生成；**市场图标/占位图**由具体市场的展示规则决定，不是 npm 字段，也不是 README 徽章。

## 已知限制

- 依赖 Qoder 的 cosy 网关与 `sash` 接口（**非官方开放 API**），上游变动后插件可能需要随之调整。协议行为参照 [Zhengyuuuui/qoder2api](https://github.com/Zhengyuuuui/qoder2api)，本项目独立实现，不复制其源码。
- **每日签到**：个人用户签到活动的历史规则为每日 100 Credits、有效期 30 天，实际奖励以活动数据为准。“一键签到所有账号”逐账号查询后领取；已领取的活动直接按成功处理，完成后刷新积分。国内版与国际版均已接入；国际版需要本机 Qoder 客户端的设备风控组件，缺失时显示相应错误。
- 图片输入与“可关闭思考”的默认值依据已核实清单；新模型可手动开启。保存图片选择后，仅勾选的模型启用图片能力；清空并保存后保持全部关闭。
- 模型可用性测试和批量测试均发送真实请求，可能消耗少量额度；批量操作范围为显式勾选的池成员。
- **独立 CLI 依赖DSH提供的 peer 包**（`@deepseek-ai/dsh-atomic-write` / `dsh-home-paths`）。

  本机实测：pnpm 会随包把它们装进 profile 的依赖树，`pnpm exec dsh-connect-qoder …` 与 npm 装的 tarball 均可正常执行；但这取决于 pnpm 版本与 `auto-install-peers` 配置，若报 `Cannot find package '@deepseek-ai/dsh-atomic-write'` 表示对应 peer 依赖缺失。

  provider 注册、设置页与对话**都不受影响**（它们在DSH进程内，DSH 进程解析这些依赖包）。同族 workbuddy 的修复方案（优先使用 DSH 提供的依赖包，缺少时使用插件自带实现）适用于本项目，尚未移植。

## 隐私与截图

**仓库与 npm 包里不出现任何真实账号数据。** 这一条是硬约束，规则与操作细则见 [docs/PRIVACY.md](docs/PRIVACY.md)：

- README 与 `docs/` 里的截图**须使用示例数据**：账号名是 `acct-demo`，邮箱是 `account-a@example.com`，余额是整数。**使用不透明白块完整遮挡，或直接替换为示例值**。模糊、马赛克和半透明色块仍会保留字符宽度与亮暗分布。
- 代码、注释、CHANGELOG 里记录「用户实拍」现象时，同样只保留现象、换掉身份。
- `tests/privacy.spec.ts` 在 CI 中独立检查已知邮箱、上游昵称、凭据指纹、本机绝对路径和凭据本体。新增或替换 `docs/assets/` 图片时，须人工检查实际图像内容。

> 早期版本截图曾包含真实数据，后续已替换为示例数据。泄露处理流程见 [docs/PRIVACY.md](docs/PRIVACY.md)。

## 免责声明

- 本项目**仅供个人学习和研究使用**，仅驱动使用者自己的 Qoder 账号在本机调用，请勿用于商业用途或超出个人合理使用的场景。
- 使用者需遵守 Qoder / 通义灵码的服务条款；因使用本项目产生的任何后果（包括但不限于账号被限制、额度被清空、服务中断），由使用者自行承担。
- 本项目作者不对任何因使用或滥用本项目产生的直接或间接损失负责。
- 本项目与阿里巴巴、通义灵码、Qoder、DeepSeek 均无关联，未获其授权或认可；文中出现的名称仅用于描述兼容关系，其商标权利归各自所有。

## 致谢

本项目建立在他人已公开的工作之上，详见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)：

- [**Zhengyuuuui/qoder2api**](https://github.com/Zhengyuuuui/qoder2api)：cosy 协议 wire behavior 的参照实现（编码、签名、会话、指纹、jobToken、SSE 形状、错误分类）。本项目按其行为**独立实现**为 TypeScript，不复制其源码；该仓库未附独立 LICENSE，其 README 声明源自 `wangtufly/QCCG`（GPL-3.0）。
- [**L0NE-6/Qoder-Daily**](https://github.com/L0NE-6/Qoder-Daily)（MIT）：每日签到的协议参照（`campaigns` / `claim` 形状、重复领取的处理规则、国际版设备风控思路）。本项目按其文档与本机实测行为独立实现，不复制其源码。
- [**dingminhua/dsh-connect-workbuddy**](https://github.com/dingminhua/dsh-connect-workbuddy)（MIT）：插件工程结构、卡片交互模型、loopback shim 模式与 npm 发布工程的基准。
- [**dingminhua/dsh-connect-trae**](https://github.com/dingminhua/dsh-connect-trae)（MIT）：经 workbuddy 转引的卡片外观与发布工程基准。
- [**franksong2702/dsh-codex-connect**](https://github.com/franksong2702/dsh-codex-connect)（Apache-2.0）：经 workbuddy 转引的 DSH 插件结构与 pi-ai provider 注册参照。
- Qoder 官方 SDK 的类型定义：仅用于 usage/头的形状核对，不随本项目分发。

## License

本项目采用 MIT 许可证，版权归 LaoDing 所有，详见 [LICENSE](LICENSE)。
