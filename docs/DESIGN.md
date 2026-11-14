# 设计方案：dsh-connect-qoder

> **目标**：把 Qoder（通义灵码国际 / 国内）接入 DeepSeek Harness（DSH）。
> **表现形式**：与插件家族一致（`dsm-*` 卡片外观），基准实现为
> [`dsh-connect-workbuddy`](https://github.com/dingminhua/dsh-connect-workbuddy)。
> **内核**：Qoder 未提供本插件可直接使用的 OpenAI 兼容端点，Qoder 官方 SDK 用于启动 agent，
> 因此本项目直接实现 cosy 网关协议的最小客户端。

---

## 一、插件职责与DSH 接口

本插件管理 Qoder provider、凭据、账号池和用量。配置、磁盘文件、运行状态与事件均由本插件独立维护。跨连接器共享凭据需要额外的存储与日志约定；跨插件联动还会增加故障归因和版本兼容的维护成本，因此当前实现采用独立运行方式。

1.10.0 引入的跨连接器余额对比已在 1.12.0 移除（issue #14）。当前会话使用 Qoder 时显示相应计费读数；比较其他连接器余额可切换模型查看。

共享输入区槽位由DSH管理。`creditPosition` 提供输入框工具栏和输入框下方两种位置，供用户处理布局冲突。

以下能力通过DSH公开接口实现：

- 读取 `modelSelection`，判断当前会话是否使用 Qoder 模型。
- 参考同族项目的源码设计、卡片样式和缺陷修复经验，运行时各自维护状态。
- 在旧 single 模型标签槽上，通过DSH注册表委托其他标签组件渲染。
- 读取模型目录中的 `model.badge`，按DSH格式展示该模型的徽标。

### 模型选择器标签注册与委托

`conversation.input.model.label` 的注册实现见 `src/client/QoderModelLabel.tsx` 与 `src/client/index.tsx`。keyed DSH按 provider id 分别注册 Qoder 标签；旧 single DSH按 priority 升序选择一个注册项，Qoder 使用 `-101`。

在旧 single 槽中，外部 provider 的模型行优先委托给注册表中的其他组件，传递 `{ provider, model, placement, selected }`，并按该组件的 locale 命名空间绑定 `t`。组件须作为子元素渲染，以保持独立的 React hooks 序列。

缺少其他注册项、locale 服务或有效翻译器时，使用 `ForeignModelLabel` 显示名字及目录 `model.badge`：`trigger` 位置徽标在前，`menu` / `option` 位置在后。该分支采用本地数据渲染并处理无效数据。相关测试位于 `tests/model-label.spec.tsx`；同族约定见 WorkBuddy 的 `docs/MODEL-LABEL-SPEC.md` §4。

### 促销徽标作为目录数据

`src/model-badge.ts` 计算本插件的促销徽标，`adapter.listModels()` 将其写入 `LlmModelInfo.badge`。DSH 0.2.2 起支持该字段；目录徽标由各 provider 独立提供，可与标签组件的附加内容同时展示。

---

## 二、与 WorkBuddy 的实现差异

| 层 | 来源 | 取什么 |
|---|---|---|
| 插件外壳 / 外观 | `dsh-connect-workbuddy` | 卡片结构、`dsm-*` CSS、`settingsScope` 配置回写、路由形态、发布工程、CI |
| 连接内核 | 自行实现（`src/cosy.ts` / `src/upstream.ts`） | Qoder 的 cosy 协议：自定义 Base64、MD5 签名、RSA/AES 会话、指纹派生、jobToken 交换、SSE 增量解析 |
| 历史缺陷修复依据 | `dsh-connect-workbuddy` | 「平台标记 ≠ 模型原生能力」「关闭思考选项须经请求验证」等历史缺陷（见第六节） |

主要实现差异：

- **凭据**：workbuddy 读桌面端本地加密凭据（`auth.ts` / `at-rest.ts` / `oauth.ts`）；
  本插件用用户自备的 PAT（`credentials.ts`）+ 本地 vault（`accounts.ts`）。
- **签到**：两边都有每日签到，但端点不同——WorkBuddy 有专用 checkin 端点，Qoder 的签到通过 `sash/api/v1/me/campaigns`（活动/权益）+ `…/{campaignId}/claim` 里， 名字没有「签到」二字，照 checkin / signin / reward 猜路径一律 404（早期据此 误判为「没有签到」，已修正）。

  国际版的活动只下发给带设备风控头的请求， 设备风控头由 `device-risk.ts` 读本机 Qoder 客户端的 UMID 组件产出。
- **工具调用恢复**：workbuddy 需要 `dsml-recovery.ts` / `markup-diagnosis.ts`，
  因为其上游**不**发原生 tool_calls；Qoder 上游**原生**发 `delta.tool_calls`
  （见 `src/upstream.ts` 的解析与 `src/shim.ts` 的直写），故不需要这两个模块。

---

## 三、模块结构

```text
DSH PiAiAdapter（qoder / qoder-global 各一套）
  -> 安全 loopback shim（每区域一个随机端口 + 进程内随机 secret）
  -> QoderUpstreamClient（cosy 协议最小客户端）
  -> https://gateway.qoder.com.cn/algo/api/v2/...   （国内）
  -> https://api1.qoder.sh/algo/api/v2/...          （国际）
  -> Qoder SSE（OpenAI 形状增量）
```

| 模块 | 职责 |
|---|---|
| `status-paths.ts` | Host 与浏览器共用的路由常量、DTO、区域及动作辅助函数（无 Node.js 依赖） |
| `cosy.ts` | 协议层：自定义 Base64、`n/3` 重排、签名、RSA/AES 会话、指纹 |
| `upstream.ts` | `exchangeJobToken` / `fetchModels` / `chatStream` / `fetchUsage` / `fetchCampaignFeed`；错误分类 |
| `credentials.ts` | PAT 校验、`qoderAccountId`（sha256 前 16 位）、掩码 |
| `accounts.ts` | 本地 vault（`$DSH_HOME/.qoder-accounts.<region>.json`，0600），三源去重合并 |
| `account-pool.ts` / `-store.ts` / `-run.ts` | 账号池排序（`rankPool`）、将账号测试结果保存到文件、批量测试 |
| `off-thinking.ts` | 「可关闭思考」的校正表与种子（见第六节） |
| `native-modality.ts` | 模型原生多模态表与图片默认值（见第六节） |
| `shim.ts` | loopback 服务：入站校验、SSE 翻译、首字节前故障转移 |
| `adapter.ts` | pi-ai 初始化：按版本选择上下文处理方式、思考档位映射、模型描述符 |
| `web-status.ts` | 浏览器路由：usage / account-credits / campaigns / accounts / pool / transfer / probe / `__save` |
| `index.ts` | 插件入口：初始化国内区与国际区服务、定义配置 schema、注册目录与 provider |
| `bin.ts` | CLI：`doctor` / `status` / `logout` |

---

## 四、凭据与账号池

三源按 **settings → env → vault** 优先级去重合并（同一个 PAT 只在最高优先级
来源里列出一次，id 由 PAT 本身派生，故重复项必然精确相同）：

| 来源 | 位置 | 说明 |
|---|---|---|
| settings | `regions.<region>.pat` | 卡片填写，优先 |
| env | `QODERCN_PERSONAL_ACCESS_TOKEN` / `QODER_PERSONAL_ACCESS_TOKEN` | 无 UI 场景 |
| vault | `$DSH_HOME/.qoder-accounts.<region>.json` | 多账号池，0600 |

账号池批量测试会发送真实探测请求，结果保存至 `$DSH_HOME/.qoder-pool.<region>.json`。首发使用当前选中账号；账号池开启后，在首字节前发生可恢复失败时，按 `rankPool` 选择接管成员，顺序为可用性 → 临期积分 → 用户手动顺序 → 积分到期与金额 → 余额 → id。流开始输出后直接返回后续错误。

积分到期由签到批次的 `validity.days` 推算（`credit-expiry.ts`），按保守上界参与排序。套餐到期来自 `qoderUsage.expiresAt`（`planExpiresAtMs`），单独展示。签到批次与窗口时间分别保存在池文件的 `credits` / `windows` 字段，重启后可恢复（issue #31 / #32）。

国内版与国际版支持每日签到。批量动作包括 `checkin` / `test`，另有逐账号 `remove`。签到先查询领取状态，再领取可用奖励；`regions.<region>.pool.autoCheckin` 控制可选自动签到，默认关闭。

---

## 五、推理与上下文

- **推理**：从 `thinking_config.enabled.efforts` 读取逐模型声明的档位，`qoderThinkingLevelMap` 将DSH档位映射为上游同名档位，未声明的档位映射为 `null`。模型支持关闭思考时额外提供 `off`；没有可映射档位时返回 `undefined`，避免DSH拒绝空的推理档位列表。请求通过顶层 `reasoningEffort` 和 `parameters.reasoning_effort` 传递所选档位。
- **上下文档位**：从上游 `context_config` 解析 `contextTiers`。`applyContextBudgets` 仅对已声明档位应用所选预算，其余使用原生值。

---

## 六、模型能力修复依据

以下修复按精确 model id 维护能力核对表，有文档或实测依据时设置默认值，其余保留为 `unknown`。

### 6.1 图片能力核验（`native-modality.ts`）

上游 `supportsImages`（Qoder 的 `is_vl`）是平台的图片输入声明，历史目录中几乎每个模型均为 `true`。插件通过 `NATIVE_MODALITY_BY_MODEL_ID` 与 `imageDefaultFor` 维护逐模型依据；确认 `multimodal` 时预勾选，`supportsImages:false` 仍会关闭默认值，表外模型为 `unknown`。

旧表曾按厂商命名将 `kimi-k2.7-code`、DeepSeek-V4-Pro 和 GLM-5.2 判为纯文本。2026-10-29 使用包含数字 `7` 的 PNG 测试两区域目录并集的 14 个模型，全部识别成功，因此修正旧结论。

登记依据采用厂商文档及实机测试，主键使用内部短 id，保留厂商名为别名。上游曾将 `Qwen3.7-Max` 更名为 `Qwen3.8-Max`，使旧表四个 `qwen3.x` 键失配；内部短 id 可在显示名变化时保持匹配。

同次测试中，`mmodel` 失败源于 `parseStreamDelta` 无法处理 `body: "null"` 空帧而抛出 TypeError。登记模型能力前须检查请求和解析过程，以区分插件缺陷与模型能力限制。

### 6.2 关闭思考的校正依据（`off-thinking.ts`）

workbuddy 实测：声明可关闭的模型里有一部分对 `reasoning_effort:"off"` 回 400，
且错误体未标明参数。由于该档位写进运行时描述符，失败会在**此后每一次请求**上
重现，影响该模型的全部请求。

解法：`REFUSES_OFF_MODEL_IDS` + `offDefaultFor` + `defaultOffModelIds`（首读种子）
+ `withEffectiveOff`（将用户选择应用至运行时目录）。**差异**：本项目暂无实测拒绝
样本，因此拒绝集保持为空——默认行为等同 0.1.0（不倒退），取得拒绝样本后补入对应模型 id。代价不对称是这里的取舍依据：提供一个必然失败的档位，代价是
用户该模型的每一次请求；藏起一个本可用的档位，代价只是一个选项。

### 6.3 图片勾选必须独立持久（`imageModelKeys`）

workbuddy 用独立的 `imageModelIds`；本项目早期将图片选择**保存在** `lastCatalog`
快照，目录一被重写就可能连带改变某个模型的图片输入设置。现改为独立集合
（`regions.<region>.imageModelKeys`），运行时只从它派生图像能力。

---

## 七、安全

- **PAT 保管与传输**：Host 保管 PAT，并经 HTTPS 用于令牌交换及用量查询的兼容回退；pi-ai 连接本地 shim，聊天请求使用 cosy 签名会话。
- **shim 入站校验**：先查 Host / Origin（仅接受 `127.0.0.1`）/ Content-Type，
  再做 `timingSafeEqual` bearer 比对；body 有大小上限。
- **脱敏**：上游文本在写入文件或返回调用方前，均由 `redactUpstreamText` 脱敏
  （`key=secret` 保留键名抹值；裸 `pt-` token 整段替换）。
- **vault 权限**：`0600`（目录 `0700`），写入走原子替换 + 文件锁。
- **不读桌面端登录文件**：与 workbuddy 不同，本插件不接触、不修改 Qoder 桌面端
  的任何本地凭据。

---

## 八、源头与致谢

- 外观与工程骨架：[`dsh-connect-workbuddy`](https://github.com/dingminhua/dsh-connect-workbuddy)（MIT）—— 各源文件头部均带标注。
- cosy 协议行为参照：[`qoder2api`](https://github.com/Zhengyuuuui/qoder2api) / `QCCG`（其声明为 GPL-3.0）。
  本项目**未复制其源码**，按其文档与运行行为独立实现为 TypeScript。
  合规细节见 [`THIRD_PARTY_NOTICES.md`](../THIRD_PARTY_NOTICES.md)。
- 商标：Qoder 名称是厂商商标，此处仅用于标识本插件所连接的服务；插件图标沿用 LaoDing 插件家族通用的 LD 品牌标（与 dsh-connect-trae / dsh-connect-workbuddy 字节一致），不使用厂商商标图形。
