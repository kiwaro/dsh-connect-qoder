# dsh-connect-qoder 的 DSH 0.2.2-rc.2 升级调研

> 调研日期：2026-11-12
> DSH：DSH Desktop **`0.2.2-rc.2`**（`app.asar` 内 `@deepseek-ai/dsh-*` 全部为
> `0.2.2-rc.2`，asar sha256 `1535941c…39d`）
> 插件：`@dmh2002/dsh-connect-qoder` **1.25.0 → 1.25.1**（本轮**不改运行时代码**，只重提取分类器代码样本）
> 上一轮：[`docs/DSH-0.2.2-RC1-FOLLOWUP.md`](DSH-0.2.2-RC1-FOLLOWUP.md)（`0.2.1-rc.5` → `0.2.2-rc.1`，那一轮要改代码）

---

## 升级影响

**DSH 更新了失败分类器的判据，须重新提取测试使用的分类器代码样本；插件运行时代码保持兼容。**

与上一轮（rc.1，四处类型破坏）不同，`0.2.2-rc.1 → 0.2.2-rc.2` 在**插件使用的DSH 接口**上均为兼容扩展：`typecheck` 一次通过，`build` 一次通过。唯一失败项位于**本插件的 DSH 分类器代码样本**：DSH 为 `classifyPiAiError` 增加报文开头状态码匹配逻辑及依赖常量，而此前手工复制的代码样本仍使用旧实现。

| 层 | 结论 | 所需动作 |
|---|---|---|
| `classifyPiAiError` 判据改写（新增 `LEADING_HTTP_STATUS`） | 🔴 代码样本一致性检查失败（保存内容与已安装的 DSH 存在差异）；手工补时若漏了这个常量，`new Function` 会 `ReferenceError` | **重提取分类器代码样本**（P0，已做） |
| `dsh-llm-pi-ai`：replay 加请求前缀摘要 + `onThinkingOmitted` | 🟢 纯新增，插件不构造 replay、不传该观察点 | 无 |
| `dsh-llm`：`HarnessError` 迁到新的 `@deepseek-ai/dsh-error` | 🟢 `dsh-llm` 仍重新导出它；插件从不用错误基类 | 无（但固定了 overrides，见 §3.2） |
| `dsh-attachment`：`AttachmentError` 改继承 `HarnessError` | 🟢 `code` 与 `name` 不变；插件只 `import type` | 无 |
| `dsh-client-ui-slots`：新增 `hoc` 槽位种类 | 🟢 插件只用 `single` / `list` / `keyed` / `chain` | 无 |
| `dsh-client-ui-settings`：`openSettings(sectionId?)` | 🟢 只加了一个可选形参 | 无 |
| peer 范围覆盖 `0.2.2-rc.2` | 🟢 同一元组，**不需要新分支** | 不用改 |
| 开发依赖树 | 🟡 仍固定在 rc.1 | **升到 rc.2**（已做） |

---

## 一、实测环境

DSH `app.asar` 内 `node_modules` 的实际版本：

| 包 | 0.2.2-rc.1 | **0.2.2-rc.2（本机）** |
|---|---|---|
| `@deepseek-ai/dsh` 及全部 `dsh-*` | 0.2.2-rc.1 | **0.2.2-rc.2** |
| `@deepseek-ai/dsh-error` | 不存在（`HarnessError` 写在 `dsh-llm` 里） | **0.2.2-rc.2（新包，npm 上首个版本）** |
| `@earendil-works/pi-ai` | 1.3.0 | **1.3.0（不变）** |
| `@deepseek-ai/cordis` | 4.1.0 | **4.1.0（不变）** |
| `@deepseek-ai/schemastery` | 3.19.0 | **3.19.0（不变）** |

也就是说：这一轮**没有换 pi-ai、没有换 cordis、没有换 schemastery**，改动全部落在
`dsh-*` 自己的声明与实现里。复现方式：

```bash
rsync -a --exclude node_modules --exclude .git ./ /tmp/qoder022rc2/
cd /tmp/qoder022rc2
# overrides 改成 rc.2 线
rm -f pnpm-lock.yaml && pnpm install --no-frozen-lockfile
npx tsc -p tsconfig.json --noEmit          # 0
npx tsc -p tsconfig.client.json --noEmit   # 0
npx vitest run                             # 1 failed（代码样本一致性检查）→ 更新代码样本后 0
npx tsdown                                 # 通过
```

---

## 二、P0：DSH 改写了失败分类器，分类器代码样本必须重提取

### 症状

`pnpm test` 的唯一失败项为 `tests/shim.spec.ts` 里那条**自检**：

```
classifyPiAiError 第 38 个字符起与装机DSH不一致：
期望 "if (/\b(?:401|403)\b/.test(message)) return \"AUT"，
实为 "const statusSource = LEADING_HTTP_STATUS.exec(me"
```

DSH更新了判定逻辑：

```js
// 0.2.2-rc.2 · app.asar
const LEADING_HTTP_STATUS = /^[45]\d\d\b/;

function classifyPiAiError(message) {
	const statusSource = LEADING_HTTP_STATUS.exec(message)?.[0] ?? message;
	if (/\b(?:401|403)\b/.test(statusSource)) return "AUTH";
	if (isQuotaExceededError(message)) return QUOTA_EXCEEDED_CODE;
	if (/\b429\b/.test(statusSource) || /rate.?limit/i.test(message)) return "RATE_LIMIT";
	if (/\b413\b/.test(statusSource) || /failed to buffer …/i.test(message)) return "INVALID_REQUEST";
	if (/\b400\b/.test(statusSource) || /invalid.?request/i.test(message)) return "INVALID_REQUEST";
	if (/\b5\d\d\b/.test(statusSource)) return "SERVER";
	…
}
```

DSH的意图写在源码注释里（asar 里就是这段）：

> The body can quote another hop's status, such as a gateway 500 that embeds its
> upstream's 401, so when a message leads with a status the status rules read
> only that status; other wordings scan the whole message.

分类器优先匹配错误报文开头的 HTTP 状态码：匹配成功时，状态码规则仅使用该值；匹配失败时，仍检查完整报文。`isQuotaExceededError`、`rate.?limit` 与 `invalid.?request` 等文本规则继续检查完整报文，因此插件仍须对错误消息中的敏感状态码进行改写。

### 对本插件的行为影响：零

将插件实际错误消息传入新分类器逐条对比（旧 → 新）：

| 文案 | 旧 | 新 |
|---|---|---|
| `qoder upstream quota-exhausted (http 4xx): … HTTP 4xx {"code":"110"…}` | `QUOTA` | `QUOTA` |
| `qoder upstream rate-limited (http 4xx): …` | `RATE_LIMIT` | `RATE_LIMIT` |
| `qoder upstream transient (http 503): provider error` | `SERVER` | `SERVER` |
| `qoder upstream invalid-request (http 400): bad` | `INVALID_REQUEST` | `INVALID_REQUEST` |
| `401 TOKEN_EXPIRE`（`auth` 档原文照送） | `AUTH` | `AUTH` |
| `未配置 Qoder PAT` | `PI_AI_ERROR` | `PI_AI_ERROR` |

原因：`src/shim.ts` 的 `hostFailureMessage` 一直把独立出现的 401/403 改写成 `4xx`，并给非 auth 档加 `qoder upstream <kind> (http <4xx|5xx>)` 前缀，使新判据优先读取该前缀中的状态码，从而保持原有分类结果。

`4xx` 不匹配 `^[45]\d\d\b`（`x` 不是数字），因此 `statusSource === message` 全串扫描， 与旧行为一致。

### 修法：把「重提取」变成一条命令

手工更新代码样本时仅复制了两个函数，遗漏新增常量 `LEADING_HTTP_STATUS`。
执行 `new Function(...)` 时抛出 `ReferenceError: LEADING_HTTP_STATUS is not defined`，
测试在执行分类逻辑前即因依赖缺失而失败。

所以新增 `scripts/extract-host-classifier.mjs`：

- DSH 打包后的函数体可能包含顶格的 `}`，因此提取时按**签名**定位、按**花括号配平**截取；直接截取到下一个 `\n}` 可能提前结束；
- 三段一起提：`LEADING_HTTP_STATUS`（只存初始化部分）、`isQuotaExceededError`、
  `classifyPiAiError`；
- `--check` 只比对不写（给 CI / 非 macOS 用）；
- 记录 `occurrences`，并且 `classifyPiAiError` 出现两份时**停止提取并报错**，因为这意味着
  DSH内嵌了两代 `dsh-llm-pi-ai`，取哪一份就不再是无所谓的决定。

`package.json` 加了 `pnpm run fixture:host-classifier`，分类器代码样本里也写了 `_refresh` 字段。
同时修正 `tests/host-piai.ts` 文件头对提取脚本的引用，使其指向实际脚本；原注释所指的 `source` 字段未记录该脚本位置。

---

## 三、P1：五个变更包对插件的影响

逐包比对 rc.1 与 rc.2 的 tarball（`lib/types/**/*.d.ts` + `lib/index.js` + `lib/client.js`）。
插件声明了 13 个 `dsh-*` 依赖，其中 11 个在代码中使用 import；`dsh-client-ui-primitives` 与 `dsh-settings` 仅声明依赖，前者的用途见 §3.4。以下比较实际使用的 11 个包：

- **5 个有变化**：`dsh-llm`、`dsh-llm-pi-ai`、`dsh-attachment`、
  `dsh-client-ui-slots`、`dsh-client-ui-settings`（下面 3.1–3.4 逐条说）；
- **6 个的 `lib/types` 与 host 侧 `lib/index.js` 逐字节相同**：`dsh-atomic-write`、
  `dsh-client-locale`、`dsh-client-ui-renderer`、`dsh-home-paths`、
  `dsh-host-webserver`、`dsh-client-ui-settings-plugins`。

那 6 个里，`dsh-client-locale` / `dsh-client-ui-renderer` / `dsh-client-ui-settings-plugins`
的**浏览器半边**（`lib/client.js`）确实重新打包过（rc.2 的 tarball 开始带 `src/`，
产物也重压了），但插件对它们的用法是 `import type {}` 的**模块增强**（给 `SlotMap`
补声明），类型引用在编译后消除，而且它们提供的槽与语言行接口约定在两边一致。

### 3.1 `dsh-llm-pi-ai`：replay 增加请求前缀摘要

新增 `lib/types/prefix-binding.d.ts`：

```ts
export type PrefixDigests = Map<AssistantMessage, string>
export declare function bindThinkingToPrefix(
  context: PiContext, api: Api, prefixDigests: PrefixDigests,
  manualThinking: boolean, onOmitted: (count: number) => void,
): PrefixBoundContext
```

配套地，`toPiReplayState(…, prefixDigest?)`、`toPiAssistant(…, prefixDigests?)`、
`toPiContext(…, prefixDigests?)`、`toStreamChunks(…, prefixDigest?)` 各加了一个
**可选**尾参，`PiAiAdapterOptions` 新增可选观察点 `onThinkingOmitted`。

Anthropic 的 signed thinking 块与生成时的请求前缀绑定；压缩或裁剪工具结果会改变前缀，可能导致重放请求被拒绝。rc.2 为每条 assistant 响应记录前缀摘要，重放时比较摘要，并在前缀发生变化时移除对应的 signed thinking 块。

**对本插件的影响：零。** 全部是可选形参；插件从不自己调 `toPiContext` /
`toPiReplayState`（它交给 `PiAiAdapter`），也不传 `onThinkingOmitted`。
`bindThinkingToPrefix` 只在 `api === 'anthropic-messages'` 时才有意义，而 Qoder 路由
走的是 `openai-completions`。

### 3.2 `dsh-llm`：`HarnessError` 搬到 `@deepseek-ai/dsh-error`

`lib/types/error.d.ts` 里 `HarnessError` 的定义删了，改成

```ts
import { HarnessError } from '@deepseek-ai/dsh-error'
export { HarnessError }
```

即两个 import 路径指向**同一个类**。同时 `LlmError` 的构造签名收紧：
`code: string → code: BuiltinLlmFailureCode`，并新增 `LlmError.fromFailure()` 供
「外部提供方自定义 code」的场景使用。

**对本插件的影响：零。** 插件 `import` 的 `dsh-llm` 名字只有 `resolveRetryPolicy` / `ReasoningEffortId` / `LlmModelInfo` / `LlmResolvedModelInfo` / `PreparedAdapterCall`，从不构造 `LlmError`、也不 `instanceof HarnessError`。

插件失败分类读的是**上游 HTTP 状态与 Qoder 业务码**，不是 `LlmError.code` （这一点上一轮已经记过，本轮再次核查确认，因此 `LlmError` 的构造签名收紧 不影响本插件调用路径）。

> 但 `@deepseek-ai/dsh-error` 现在有 `dsh-llm` 与 `dsh-attachment` 两个**精确版本**的
> peer 声明。所以 `pnpm-workspace.yaml` 里显式固定其版本：将来某个包改成 `^0.2.2` 时，
> 解析出第二个副本会让 `instanceof HarnessError` 在两个类之间失效，而这正是
> DSH用来判断「附件失败」的机制。

### 3.3 `dsh-client-ui-slots`：新增 `hoc` 槽位种类

`SlotKind` 从 `'single' | 'list' | 'keyed' | 'chain'` 变成了再**多**一档 `'hoc'`，配套
`HocRenderOpts` / `HocShare` / `HocKeysOf` 与 `KindOptions` 的 hoc 分支。DSH用它做
「包裹层」：`conversation.hero.frame`（包住会话滚动体）就是第一个 hoc 槽。

**对本插件的影响：零。** 插件用到的槽位全是 `conversation.input.right`（list）、`conversation.composer.dock`（list）、 `settings.section`（list）、`settings.section.icon`（list）、 `sidebar.panellist`（list）、`conversation.input.model.label`（single/keyed 运行时探测）。

`model-label-seat.ts` 的 `declaredSlotKind()` 遇到不认识的 kind 会**返回 undefined** ，并采用单槽兼容方式，因此即使读到 `'hoc'` 也不会崩，只是退回保守形状。

### 3.4 其余「有变化但无关」

| 包 | 变化 | 为什么无关 |
|---|---|---|
| `dsh-client-ui-settings` | `SettingsLauncherOwnerProps.openSettings: () => void` → `(sectionId?: string) => void` | 只加可选形参；插件从不注册 `settings.launcher`，它跳设置页走的是 `sidebar.settings` 的 store 槽位（`openSection(id)`），rc.2 里那段槽位代码逐字未变 |
| `dsh-client-ui-primitives` | 删 `useAnchoredPosition` / 新增 `usePointerGrace`、`Modal.tone`、`Checkbox.modalAutofocus` | 插件**从不 import** 这个包（`src/client/popover.ts` 是本地 40 行实现，文件头写着为什么），这是刻意的，因此未受该导出删除影响 |
| `dsh-client-ui-conversation` | 新增 `conversation.hero.frame`（hoc）、`insertQuote(text, summary?)`、`ComposerBarProps.occurrence` | 插件不碰 hero、不调 `insertQuote`、不读 `occurrence` |
| `dsh-client-ui-model-selection` | 新增 `conversation.input.model.options`（list）槽 | 新增槽；标签订阅用的 `conversation.input.model.label` 声明**没变**（仍是 `single`） |
| `dsh-client-locale` | client 产物重压过，`register` 签名逐字相同；另有新的语言行组件与 `settings-store` | rc.1 已适配过的索引签名仍够用；插件只注册自己的命名空间，不碰语言行 |
| `dsh-client-ui-settings-plugins` | client 产物重压过；`settings.plugins` / `settings.plugins.tab` 两个槽在两边一致 | 插件自 1.18.0 起不再注册这两个槽（配置搬进 `settings.section`） |

`dsh-host-webserver` / `dsh-settings` / `dsh-atomic-write` / `dsh-home-paths` 的
`lib/types` 与 `lib/index.js` 逐字节相同（rc.2 只是多发了 `.d.ts.map` 与 `src/`），
不单独列行。

### 3.5 有一条**看起来要改、实际不用**的

`dsh-client-ui-settings-general` 的 rc.2 client bundle 里，侧边栏设置触发器**仍是**

```js
<button type="button" aria-haspopup="dialog" aria-expanded={open} …>
```

只是同一份文件里又多了一颗 `aria-haspopup="menu"` 的按钮（「发行通道」选择器）。 `src/client/settings-jump.ts` 的 `TRIGGER_SELECTOR` 是 `button[aria-haspopup="dialog"][aria-expanded]`，**不会**误命中新按钮； `sidebar.settings` 的 store 槽位（`create()` 固定在唯一活动实例上）在 rc.2 里逐字未变。

两条路径都仍然可用。

---

## 四、P2：peer 范围不需要新分支（这一条要写下来，因为直觉相反）

当前声明（`package.json` 的 `peerDependencies`）：

```
>=0.1.7-rc.1 <0.2.0-0 || >=0.2.0-rc.1 <0.3.0-0 || >=0.2.1-rc.1 <0.3.0-0 || >=0.2.2-rc.1 <0.3.0-0
```

`0.2.2-rc.2` 与 `0.2.2-rc.1` **同一元组**，第四支的比较符自带预发布标签，于是整个
`0.2.2` 元组均可通过版本范围检查。实测（直接运行 node-semver）：

| 版本 | 标准语义（npm/pnpm） | DSH兼容性检查（`includePrerelease`） |
|---|---|---|
| `0.2.2-rc.1` | true | true |
| **`0.2.2-rc.2`** | **true** | **true** |
| `0.2.3-rc.1` | false ❌ | true |

**新增补丁版本的预发布匹配分支时，以 `major.minor.patch` 元组为依据。** 上一轮为 rc.1 补分支，是因为
`0.2.2` 这个元组此前没有任何带预发布标签的比较符；现在有了，rc.2 / rc.3 / … 都通过版本范围检查。真正的下一个待办仍是 `0.2.3-rc.1`（`tests/dsh-line.spec.ts` 的
`UNGATED_PRERELEASES` 中列出）。

本轮在 `tests/dsh-line.spec.ts` 把 `0.2.2-rc.2` 加进 `ADMITTED_PRERELEASES` 与
「DSH兼容性检查覆盖整条 0.2.x」用例，通过断言验证现有范围已支持该版本。

---

## 五、P3：开发依赖树

`pnpm-workspace.yaml` 整体迁到 `0.2.2-rc.2`（`dsh-*` 全部、外加新验证的
`@deepseek-ai/dsh-error`）。`cordis 4.1.0` / `schemastery 3.19.0` / `pi-ai 1.3.0`
与 DSH 保持相同版本，因为此次 DSH 升级未更换这些依赖。

实测结果：

| 检查 | rc.1 树 | **rc.2 树** |
|---|---|---|
| `tsc -p tsconfig.json` | 0 | **0** |
| `tsc -p tsconfig.client.json` | 0 | **0** |
| `vitest run` | 1471 passed / 1 failed（分类器代码样本） | **1472 passed / 12 skipped / 0 failed** |
| `tsdown` | 通过 | **通过** |

（rc.1 树的 1 failed 就是本轮修掉的那条自检；跳过数两边一致，都是 `live-intl-session`
这类需要有效凭据的用例。）

---

## 六、明确排除的部分

- **`ResolvedPiAiProviderProfile` 的两个图片预算字段**：rc.1 补的
  `maxRequestImages: 50` / `imageOffloadCountQuantum: 10` 仍然必需，rc.2 的
  `resolveProfile` 里 `?? 50` / `?? Math.min(10, maxRequestImages)` 逐字未变。
- **中途 system 消息**：`conversationUpdates()` 的判据在 rc.2 里逐字未变，
  插件显式写下的 `supportsMidConvoSystemMessages: false` 仍然是「改写成
  `<system-reminder>`」那一侧，与上一轮的结论一致。
- **`toolUpdate`**：插件不声明它（于是每次请求都声明完整工具表），rc.2 对
  `addition-only` 的变更（`projectToolUpdates` 的注释变长了一截：重建时已激活的声明
  变成立即可用、未激活的保持延后）未通过插件的该路由处理。
- **窗口拖拽区域**：`v0.2.2-rc.1/window-drag-regions` 那条指南讲的是 macOS 上依赖
  「全局 no-drag 排除」的自定义浮层。本插件的积分浮层是 `position: fixed` 的 body portal，**向上**挂在输入框上方（`side: 'top'`），因此该浮层不会覆盖页头，所述 「被裁剪到视口上方的内容挡住页头拖动」的场景无法在该布局下复现；它也没有任何 `data-window-drag*`。

  因此本插件沿用现有区域声明。这类窗口拖拽行为须在 macOS Desktop 上实测；普通 Web 与 Windows 以及现有 CI 无法验证该平台差异。
- **`BundleInfo.update` 取代 `updatable`**：插件不调 `pluginManager.listBundles()`。
- **`AttachmentError` 错误码进 `turn/end`**：插件只 `import type { AttachmentStore }`，
  从不抛附件错误。
- **`runtime-context` 的提醒包装**：那是DSH写会话日志的形状；插件不读也不写
  `runtime-context` 记录。

---

## 七、实施记录（2026-11-12，1.25.1）

| 项 | 改动 | 位置 |
|---|---|---|
| P0 | 新增提取脚本（按签名定位 + 花括号配平 + 唯一性检查 + `--check`） | `scripts/extract-host-classifier.mjs` |
| P0 | 分类器代码样本重提取到 rc.2：`classifyPiAiError` 新版 + `isQuotaExceededError`（逐字未变）+ `LEADING_HTTP_STATUS`（新） | `tests/fixtures/host-piai-classifier.json` |
| P0 | 自检把新常量也纳入逐字比对；`new Function` 根据代码样本是否包含该常量添加相应声明 | `tests/host-piai.ts` |
| P0 | 补掉「提取脚本见 `source` 字段」这句悬空引用；分类器代码样本里加 `_refresh` 字段 | `tests/host-piai.ts`、`tests/fixtures/host-piai-classifier.json` |
| P2 | `0.2.2-rc.2` 加进 `ADMITTED_PRERELEASES` 与DSH兼容性检查那条用例（**不新增分支**，用断言验证这个判断） | `tests/dsh-line.spec.ts` |
| P3 | overrides 整体迁到 `0.2.2-rc.2`，并显式固定 `@deepseek-ai/dsh-error` | `pnpm-workspace.yaml` |
| 工具 | `pnpm run fixture:host-classifier` | `package.json` |
| 文档 | 本文 + README 双语「当前 DSH」小节 + CHANGELOG `1.25.1` | — |

本轮修改集中在测试、提取脚本、依赖配置和文档。DSH 接口扩展保持兼容；分类器更新后，
通过 `pnpm run fixture:host-classifier` 同步提取函数及其依赖常量，更新测试使用的代码样本。
