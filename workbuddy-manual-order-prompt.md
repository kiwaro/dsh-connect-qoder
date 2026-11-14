# 任务：给 WorkBuddy 账号池加「手动指定优先级」排序

## 修改目标

任务对象为 DSH 插件 `@dmh2002/dsh-connect-workbuddy`，账号池（account pool）当前**只能按积分策略自动排序**，用户无法干预。增加手动排序功能：用户能给池成员排先后，请求优先发给排在前面且可用的账号。

同源项目 `dsh-connect-qoder` 已经做过同一件事，并已发布。下面的方案就是那份实现，已按 WorkBuddy 的实际代码校准过。实施前须按目标仓库源码核对路径、行号及函数签名。

## 相关文件

- `src/account-pool.ts`：`rankPool`（约 409 行）与 `pickAccount`
- `src/web-status.ts`：`buildPoolView`（`rankPool` 调用在约 811 行）
- `src/index.ts`：`WorkBuddyPoolPreferences` 接口（约 357 行）、`poolConfig` schema（约 481 行）、三处 `rankPool` 调用（约 1495 / 1726 / 1751 行）
- `src/status-paths.ts`：`nextRegionPool`（约 357 行）
- `src/client/AccountPool.tsx`：表格渲染（`pool.accounts.map` 在约 1119 行）、`renderAccountRow`（约 1229 行）、账号名单元格（约 1328 行）
- `src/client/account-selection.ts`：`writePoolPreferences`（约 420 行起）

## 设计要求

### 1. 独立保存成员集合与手动顺序

新增 `orderAccountIds: string[]`。`memberAccountIds` 回答「谁在池里」，`orderAccountIds` 回答「先用谁」。合用一个数组会导致「取消勾选再勾回来」意外改变优先级，而配置里现有顺序只是勾选历史、不表达意图。

空数组 = 完全按积分策略（改动前的行为，**默认不变**）。

同步修改三处：

- `WorkBuddyPoolPreferences` 接口加 `orderAccountIds?: readonly string[]`
- `poolConfig` schema 加 `orderAccountIds: z.array(z.string()).default([])`
- `writePoolPreferences` 的参数类型与 slot 写入加上它

### 2. 排序层次：可用性 → 临期积分 → 用户顺序 → 金额 → 分数 → token 新鲜度 → id

`rankPool` 加第四个参数 `orderAccountIds?: readonly string[]`，在里面建位置查表（不在表里 = `Infinity`，即排到最后）：

```ts
const order = new Map<string, number>()
for (const [index, id] of (orderAccountIds ?? []).entries()) {
  if (!order.has(id)) order.set(id, index)
}
```

每行增加 `_order: order.get(member.account.id) ?? Number.POSITIVE_INFINITY`。

`sort` 比较器里，将其插入**临期判定之后、到期时刻之前**：

```ts
if (left._usable !== right._usable) return left._usable ? -1 : 1
if (left._atRisk !== right._atRisk) return left._atRisk ? -1 : 1   // 这一行在用户顺序之前
if (left._order !== right._order) return left._order - right._order  // 新增
if (left._atRisk && right._atRisk && left._expiring !== right._expiring) { ... }
```

### 3. 优先使用临期积分，其次采用用户顺序

`rankPool` 的注释里写了 issue #70 的来龙去脉（临期积分必须优先花，否则到期作废）。用户一次误排若能导致当天要作废的分没花掉，代价远大于排序的便利。所以 `_atRisk` 保持压过 `_order`。

**必须写一个测试验证它**，测试名建议：

```
expiring credits still outrank the manual order (issue #70 wins)
```

测试和注释须保留该排序依据，供后续维护核查。在 `rankPool` 的 doc comment 里也写明这一点。

### 4. 按 `orderIds` 即时渲染表格行序

`src/client/AccountPool.tsx` 现在直接 `...pool.accounts.map(account => renderAccountRow(...))`，行序就是DSH下发顺序。**这一处不改，点上下移按钮后行不会动，必须等保存 + DSH重算才生效**。

增加一个纯函数（放在 `AccountPool.tsx` 底部或同目录一个模块里）：

```ts
/**
 * 按 orderIds 重排账号行。
 *
 * 不在名单里的（非成员、或还没排过的）保持DSH给的相对顺序接在末尾：
 * 稳定排序保证同样输入永远得到同样行序，否则表格会在每次快照回传时无声抖动。
 */
function sortAccountsByOrder(
  accounts: readonly WorkBuddyWebPoolAccount[],
  orderIds: readonly string[],
): readonly WorkBuddyWebPoolAccount[] {
  if (orderIds.length === 0) return accounts
  const rank = new Map<string, number>()
  for (const [index, id] of orderIds.entries()) if (!rank.has(id)) rank.set(id, index)
  const fallback = orderIds.length
  return accounts
    .map((account, index) => ({ account, index, rank: rank.get(account.accountId) ?? fallback }))
    .sort((left, right) => left.rank - right.rank || left.index - right.index)
    .map(entry => entry.account)
}
```

渲染处改成 `...sortAccountsByOrder(pool.accounts, orderIds).map(...)`。

### 5. 客户端算出「当前生效顺序」

在 `saved` 里加 `orderAccountIds: pool?.orderAccountIds ?? []`，在 `active` 之后算：

```ts
const orderIds: string[] = (() => {
  const members = active.memberAccountIds.filter(id => listedIds.has(id))
  const explicit = active.orderAccountIds.filter(id => members.includes(id))
  const rest = members.filter(id => !explicit.includes(id))
  return [...explicit, ...rest]
})()
const orderIsManual = active.orderAccountIds.some(id => active.memberAccountIds.includes(id))
```

未排序成员按池表原顺序排列在已排序成员之后，使卡片显示与 `rankPool` 的实际选择顺序保持一致。

### 6. 区分自动排序与手动名次

在账号名单元格（`renderAccountRow` 里 `h('b', null, name)` 之后）插入：

- `checked === false`（非成员）→ 不渲染徽章，非成员不参与排名
- `orderIsManual` → 实心品牌色徽章 `第 N 位`，`title` 说明「用户已指定手动顺序」
- 否则 → 虚线灰框徽章 `自动排序`，`title` 说明「还没有手动排序：池会先花临期积分，再按余额多寡选号。可使用 ↑/↓ 指定顺序」

**展示要求**：如果对着一个由积分策略决定的顺序显示「第 1 位」「第 2 位」，用户会以为这是自己的选择，而它每次刷新都在变。容易造成误解。

### 7. ↑/↓ 按钮

放在账号名单元格里、名次徽章旁边。**不要**放进 `.dsm-workbuddy-pool-row-actions`：那一列放的是「测试 / 删除」，语义不同，而且既有测试按索引取那一列的按钮，插进去会破坏它们。

可见条件：`pool.enabled === true && checked && canEdit`。

顺序调整仅在账号池启用时参与账号选择，因此账号池关闭时隐藏对应按钮。

首尾按钮在无法移动时保持可见并禁用：隐藏会让按钮在行与行之间跳位，增加操作定位成本。

移动函数：

```ts
const moveMember = (accountId: string, delta: -1 | 1) => {
  const members = orderIds.filter(id => active.memberAccountIds.includes(id))
  const from = members.indexOf(accountId)
  const to = from + delta
  if (from < 0 || to < 0 || to >= members.length) return
  const next = members.slice()
  const [moved] = next.splice(from, 1)
  next.splice(to, 0, moved!)
  editDraft(prev => ({ ...prev, orderAccountIds: next }))
}
```

**第一次点 ↑/↓ 就把当前可见顺序完整写入草稿**，从此由用户决定。该次操作建立显式手动顺序。

### 8. 保存校验必须**逐位**比对

`writePoolPreferences` 现在的成员校验是**按集合**比（源码注释明确写了「the stored order is not meaningful」，该判断适用于成员资格校验）。但 `orderAccountIds` 是**顺序敏感的**：仅比较集合无法验证顺序写入；误判成功后会丢弃草稿，导致排序修改丢失。

```ts
const storedOrder = pool['orderAccountIds']
if (!Array.isArray(storedOrder) || storedOrder.length !== orderAccountIds.length) return false
return storedOrder.every((id, index) => id === orderAccountIds[index])
```

注意 `orderAccountIds` 缺省时要写 `[]` 而不是 `undefined`，因为要参与逐位比对。

### 9. DSH返回前过滤失效账号

`buildPoolView` 里透传 `orderAccountIds` 前，只保留**当前确实存在的账号 id**：

```ts
...(orderedIds.length === 0 ? {} : { orderAccountIds: orderedIds }),
```

配置里的 `orderAccountIds` 在删除账号后不会自动收缩（与 `memberAccountIds` 同一个残留配置问题），原样透传会让卡片把名次算在一个已经不存在的行上。

在 `web-status.ts` 中为 `pool.preferences` 的类型增加 `orderAccountIds?: readonly string[]`，对应 Qoder 的 `QoderPoolDeps` 修改。`WorkBuddyWebPool` 类型定义在 `status-paths.ts`（约 983 行 `memberAccountIds` 附近），也在那里加。

### 10. 传递到所有 `rankPool` 调用点

`src/index.ts` 有三处 `rankPool(...)`（约 1495 / 1726 / 1751 行），`src/web-status.ts` 一处（约 811 行）。**全部**要传 `prefs.orderAccountIds`，确保全部调用路径采用相同排序规则。

同时检查 `updateMembership` 之类重建 preferences 对象的地方：WorkBuddy 里如果有 `{ ...poolPreferencesOf(...), memberAccountIds }` 这种写法要注意，展开能保住新字段；如果是**手写字段列表**的，必须补上 `orderAccountIds`，否则保存成员会清掉排序。

## 测试要求

`tests/` 下已有 `account-pool.spec.ts` 与渲染测试，沿用现有的 `member(...)` 等 helper。

**`rankPool` 层**（每条都要有）：

1. 省略 order 时，按积分策略排序（回归测试）
2. 传 order → 优先于积分策略（即使余额最少，手动排在首位的账号仍优先）
3. 不在 order 里的成员排到所有显式排过的之后
4. 出池账号不能被排到可用账号前面
5. **临期积分仍压过手动顺序**（issue #70 那条）
6. order 里有不存在的 id → 忽略，真实成员不被挤下去
7. order 里重复 id → 首次出现的位置生效，排序结果稳定
8. order 里也有分数/余额都相等的两个号 → 结果稳定（不随输入数组顺序变化）

**渲染层**：

1. 没排过 → 只出现 `自动排序` 徽章，一个「第 N 位」都不能有
2. 保存过顺序 → 行序按 orderIds 排（与余额相反才说明生效），徽章显示 `第 1 位` / `第 2 位`
3. 首尾按钮各自禁用、按钮数量恒定为 2
4. 池关闭时不渲染移动按钮
5. 点 ↓ → 行当场对调，徽章从「自动排序」变成名次

## 验收标准

在目标仓库根目录执行：

```
npx tsc -p tsconfig.json && npx tsc -p tsconfig.client.json && npx vitest run && npx tsdown
```

四项全部通过，且**已有测试须保持有效**。如果为了新功能必须改既有测试，在回复里逐条说明为什么那是正当的，不要静默修改断言。

## 交付核对

1. 实际改了哪些文件（列路径）
2. 新增测试的名单
3. `rankPool` 最终的比较器顺序（贴代码）
4. 实施中发现的设计冲突及处理结果

## 测试定位要求

先执行一次 `npx vitest run` 建立基线，记下改动前有多少个测试通过。WorkBuddy 的渲染测试使用 `h()` 构建元素，部分断言通过索引定位 DOM 节点，因此插入新元素后须先核对失败断言的目标与节点位置，再调整受影响的定位方式。
