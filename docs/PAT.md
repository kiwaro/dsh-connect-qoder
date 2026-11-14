# 获取并配置 PAT（Personal Access Token）

本页说明 PAT 的用途、申请方式、插件配置与接口验证记录。

## 一、PAT 定义与用途

PAT（Personal Access Token）是 Qoder 签发的个人访问令牌，以 `pt-` 开头。持有者可在令牌授权范围内调用 Qoder 接口。

- **创建与撤销**：用户在 Qoder 控制台创建令牌，并可随时撤销。
- **保密要求**：令牌须按账号凭据保管，禁止提交至代码仓库或公开聊天、issue。

插件使用 PAT 向 Qoder 交换会话令牌，后续模型请求由 cosy 会话认证：

```text
PAT -> Qoder jobToken 交换接口 -> jobToken -> cosy 签名会话
    -> sash 令牌交换接口 -> sash 接口令牌
```

PAT 由插件 Host 进程保管，并经 HTTPS 发送至 Qoder 的令牌交换接口；用量查询的兼容回退路径也可能使用 PAT。DSH 的模型客户端（pi-ai）连接本机回环端口，使用本地 shim 凭据。PAT 不写入会话记录。

## 二、申请 PAT

国内版和国际版分别签发令牌，使用时须匹配区域：

| 区域 | 服务站点 | 发证入口 |
| --- | --- | --- |
| 国内版（通义灵码） | `qoder.com.cn` | 登录 [Qoder CN 控制台](https://www.alibabacloud.com/help/zh/lingma/cloud-agents-cn/api-authentication) → **设置 → API 令牌** → 创建令牌 |
| 国际版 | `qoder.sh` | 登录 Qoder 控制台 → **Settings → Personal Access Tokens** → 创建（见 [Qoder 官方认证文档](https://docs.qoder.com/cloud-agents/api/conventions/authentication)） |

创建时：

1. 填写令牌名称，用于区分用途，例如“DSH 笔记本”。
2. 按控制台要求选择**权限范围 / 有效期**，确保授权覆盖所需接口；到期后重新签发。
3. 点击创建，复制并妥善保存完整令牌（形如 `pt-xxxxxxxx`）；控制台通常仅在创建时完整显示一次。

> 注意：这里说的是 Qoder **控制台网页版**里的入口，不是桌面客户端的设置页。

## 三、配置插件凭据

插件支持设置卡片和环境变量两种配置方式；同时配置时按账号去重合并。

### 方式 A：设置卡片（支持多账号）

```text
DSH → 插件 → dsh-connect-qoder → 设置卡片
     → 选对应的 tab（国内版 / 国际版）
     → 点「添加 PAT」
     → 输入 pt-…（可填写备注名，验证时读取账号名）
     → 提交
```

提交时，插件通过 Qoder 网关交换 jobToken，成功后保存凭据。验证失败会显示错误；可能原因包括令牌、区域、网络连接和上游服务状态。

每个区域拥有独立的本地凭据库（vault），支持添加、选择和删除多个账号。

### 方式 B：环境变量

适用于通过环境变量或密钥管理器提供凭据的场景：

```bash
# 国际版
export QODER_PERSONAL_ACCESS_TOKEN=pt-xxxxxxxx

# 国内版
export QODERCN_PERSONAL_ACCESS_TOKEN=pt-xxxxxxxx
```

可通过 shell 配置（如 `~/.zshrc`）或密钥管理工具（如 1Password / Bitwarden 的 shell 插件）注入环境变量，DSH 启动时读取。

### 优先级

```text
设置卡片的 PAT  >  环境变量 PAT  >  保险库（vault）账号
```

三种来源按账号去重合并。以下命令可查看凭据来源与状态：

```bash
dsh-connect-qoder status      # 各区域当前凭据来源与上游可达性
dsh-connect-qoder doctor      # PAT 是否就位、指纹派生、账号数（不联网）
```

## 四、常见问题

**提交后报「验证失败」**
令牌与区域不匹配（国内版的令牌不能用在国际版 tab），或复制时带上了空格/换行，或已过期。

**新令牌收到认证拒绝（实测记录）**
2026-10 使用同一个 CN 令牌的实测结果如下：

| 请求 | 结果 |
| --- | --- |
| `api.qoder.com.cn/api/v1/cloud/agents` | **200** —— 令牌有效、账号正常 |
| `openapi.qoder.com.cn/sash/api/v2/me/usage` | 401 `TOKEN_EXPIRE / token is not active` |
| `openapi.qoder.sh/sash/api/v2/me/usage` | 401 `TOKEN_EXPIRE` |
| `gateway.qoder.com.cn/algo/api/v2/model/list` | 403 `Signature invalid` |

接口认证要求如下：

1. `sash` 接口可能拒绝 PAT 直连并返回 `TOKEN_EXPIRE`；该结果仅说明当前请求未通过此端点认证。缺少浏览器 cookie 的 `qoder.com.cn/sash/...` 请求会返回 `UNAUTHORIZED / missing cookie header`。
2. `gateway.*/algo/*` 要求 COSY 签名，直接发送 `Bearer PAT` 会返回 `Signature invalid`。插件通过 PAT 交换 jobToken，再建立签名会话。

积分显示“—”且聊天正常时，应检查用量查询端点。当前积分查询优先使用交换所得令牌，历史实测返回 200；PAT 直连保留为兼容回退。需要单独验证 PAT 时，可参照上表检查 Cloud Agents 端点。

**模型列表为空**

确认对应区域已配置 PAT，并执行“从 Qoder 刷新”。插件仅展示具备凭据的区域模型。

**撤销 PAT 与清除本地账号**

在 Qoder 控制台撤销令牌；使用 `dsh-connect-qoder logout [--region cn|global]` 清除本地 vault 账号。

**桌面端登录凭据**

本插件采用用户提供的 PAT。WorkBuddy 的凭据读取方式依赖其桌面端文件结构；Qoder 的本地凭据存储和 CLI 登录文件具有不同格式及生成条件，本插件未接入这些来源。

## 五、Qoder 每日签到（实测协议，2026-10）

**参考实现**：https://github.com/L0NE-6/Qoder-Daily （青龙脚本，CN + Global）
**官方规则**：https://help.aliyun.com/zh/lingma/daily-check-in-100-credits-reward-program-terms

### 协议（CN 实测通过）

查询活动列表（**v1**，路径为 `campaigns`）：

    GET {openapi}/sash/api/v1/me/campaigns?forceRefresh=true
    headers:
      authorization:    Bearer {jobToken.securityOauthToken}   ← jt- 令牌，与用量同一套
      cosy-clienttype:  10
      accept:           application/json
      accept-language:  zh-CN
      user-agent:       Qoder

响应：

    {"campaigns":[
      {"campaignId":"01a12102-…","actionType":"CLAIM_BENEFIT","claimStatus":"CLAIMED",
       "benefit":{"kind":"CREDITS","amount":100,
                  "modelScope":{"modelSeries":{"key":"ALL_MODELS"}},
                  "validity":{"mode":"RELATIVE_DAYS","days":30}}},
      {"campaignId":"01a05bbf-…","actionType":"VIEW_DETAILS","claimStatus":"CLAIMED"}
    ]}

领取：

    POST {openapi}/sash/api/v1/me/campaigns/{campaignId}/claim
    （同 headers + content-type: application/json + origin）

已领取的判定：HTTP 409，或 body.result === "ALREADY_CLAIMED"，或 body.replayed === true。

### 关键事实

- 签到使用 **campaigns**（活动/权益）接口；早期按 checkin / signin / reward 推测的 13 条路径均返回 404。
- 状态机：`claimStatus` ∈ {CLAIMABLE, CLAIMED}；只有 `actionType === "CLAIM_BENEFIT"`
  且 `CLAIMABLE` 的才可领。
- 官方刷新时间：**每天 10:00 (UTC+8)** 开窗（阿里云文档写 0 点，桌面端截图显示 10:00
  ——以桌面端为准）。
- **国际版**还需要设备风控头 `Cosy-MachineToken` / `Cosy-MachineCode` /
  `Cosy-MachineType`；**CN 无此要求**。
  这三个字段由 Qoder 客户端自带的 UMID 组件在本机生成，插件按区域读取：

  | 区域 | 优先组件路径 |
  |---|---|
  | 国内版 | `/Applications/Qoder CN.app/Contents/Resources/umid/runtime-info` |
  | 国际版 | `/Applications/Qoder.app/Contents/Resources/umid/runtime-info` |

  同时安装两个区域的客户端时，优先读取对应区域组件；区域不匹配可能导致风控拒绝。
  组件缺失时显示“未找到 Qoder 设备风控组件”；可使用 `QODER_UMID_PATH` 显式指定路径。

  Windows 按以下顺序查找：

  1. `%LOCALAPPDATA%\Programs\Qoder[ CN]\resources\umid\runtime-info.exe`：**默认**安装位置；
  2. **卸载登记表**（`HKCU\…\Uninstall` → `HKLM\…\Uninstall` → `HKLM\…\WOW6432Node\…`）里
     `DisplayName` 为 `Qoder` / `Qoder CN` 的项，从 `UninstallString`（或 `DisplayIcon`）
     解析安装目录。自定义安装位置（历史实测为 `D:\…\Programs\Qoder`）通过登记表定位。

  `QODER_UMID_PATH` 的优先级在两层之上，路径失效时继续检查默认位置和登记表。
  读取结果按 50 分钟在进程内缓存（参考实现同量级），减少组件进程启动次数。
- 企业版账号不支持该活动（仅个人用户）。
