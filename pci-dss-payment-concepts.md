# PCI DSS 与现代支付概念速查

本文整理本轮讨论中的关键概念，适合放在 PCI DSS 培训材料、讲稿备注或内部支付安全知识库中。

## 1. PCI DSS 基础术语

### SAD

**SAD = Sensitive Authentication Data**，即敏感认证数据。

典型包括：

- CVV / CVC / CID
- PIN / PIN Block
- 磁条完整数据，例如 Track 1 / Track 2
- 芯片交易中的等效认证数据

核心规则：

> 授权完成后，SAD 不能存储。即使加密也不行。

特别注意：

- CVV 只能在授权过程中临时使用。
- CVV 不能进入日志、数据库、APM、BI、客服截图或测试环境。
- “误存”“临时存”“加密存”都不能作为合理理由。

### AOC 与 ROC

**ROC = Report on Compliance**，合规评估报告。

它是详细审计报告，通常由 QSA 现场评估后出具，逐条说明 PCI DSS 控制项是否满足、证据是什么、范围是什么。

**AOC = Attestation of Compliance**，合规证明或合规声明。

它是对外使用的摘要证明，说明某家公司已经完成 PCI DSS 验证，验证方式、覆盖范围和有效期是什么。

| 项目 | ROC | AOC |
|---|---|---|
| 性质 | 详细审计报告 | 合规证明摘要 |
| 内容 | 控制项、证据、范围、结论 | 验证方式、范围、有效期 |
| 敏感程度 | 高，通常不外发 | 较低，常用于对外证明 |
| 常见用途 | QSA、收单行、卡组织、内部审计 | 供应商尽调、客户准入、合作证明 |

一句话：

> ROC 是审计底稿级报告，AOC 是对外证明书。

## 2. 支付凭证与卡数据托管

### PCI DSS 保护哪些账户数据

PCI DSS 保护的是 **Account Data**，不只是 PAN。

Account Data 分两类：

```text
Account Data
├─ Cardholder Data, CHD
│  ├─ PAN
│  ├─ Cardholder Name
│  ├─ Expiration Date
│  └─ Service Code
│
└─ Sensitive Authentication Data, SAD
   ├─ Full Track Data
   ├─ CVV / CVC / CID
   └─ PIN / PIN Block
```

关键判断：

- PAN 是 PCI DSS 保护的核心对象。
- 持卡人姓名、有效期、服务码在和 PAN 一起出现时，属于 CHD。
- CVV、完整磁道、PIN / PIN Block 属于 SAD，授权后禁止落地。

一句话：

> PCI 保护的核心是 PAN 及其关联持卡人数据；CVV、磁道、PIN 这类 SAD 是授权后禁止落地的数据。

### 只显示持卡人姓名是否合规

只显示持卡人姓名，通常不构成 PCI DSS 里的核心卡数据暴露，也通常不会因为这一点单独进入 PCI Scope。

风险取决于它是否和 PAN 或可识别卡账户的信息组合出现：

| 界面显示内容 | PCI 风险 |
|---|---|
| 仅持卡人姓名 | 通常不构成 CHD |
| 姓名 + 卡品牌 | 风险较低 |
| 姓名 + 后四位 | 常见做法，可接受，但要控制访问 |
| 姓名 + 有效期 | 风险上升，但通常仍不等同完整 PAN |
| 姓名 + 完整 PAN | 明确是 CHD，进入 PCI Scope |
| 姓名 + CVV | 不应该出现 |
| 姓名 + 完整磁道 / PIN | 不应该出现 |

工程建议：

- 客服后台只显示必要字段。
- 卡片列表显示卡品牌和后四位。
- 不显示完整 PAN。
- 不显示 CVV。
- 截图、导出、BI 不要带完整卡数据。
- 做访问控制和操作审计。

一句话：

> 只显示持卡人姓名通常没问题；和完整 PAN 组合显示就进入 PCI DSS 保护范围。

### PAN、有效期与日志脱敏

日志里不能记录完整 PAN。确实需要排障时，前六后四是常见做法：

```text
411111******1111
```

或：

```text
bin=411111 last4=1111
```

但要注意：

- 不能记录完整 PAN。
- 不能记录 CVV。
- 不能记录完整磁道 / PIN / PIN Block。
- 前六后四要确保中间位数真的被遮蔽，而不是 UI 遮蔽但原始日志仍有完整值。
- 测试、APM、错误堆栈、网关 access log 也要同样脱敏。

有效期单独出现通常不是最高敏数据，但它的熵很低，基本可以枚举：

```text
月份：01-12
年份：未来几年
```

如果日志里已经有：

```text
BIN + last4 + cardholder name + user_id + merchant_id
```

再加上有效期，会让卡片画像更完整，也会降低攻击者试错成本。

推荐日志规范：

```text
card_brand=visa
bin=411111
last4=1111
exp_present=true
token_id=tok_abc123
```

默认不要记录：

```text
exp_month
exp_year
expiry
card_expiry
expirationDate
```

如果排障必须记录，最多记录：

```text
exp_year=2029
exp_month=**
```

一句话：

> 前六后四可以作为排障标识，但有效期默认不进日志；一旦和 PAN、token、姓名、用户 ID 组合，就应该按敏感支付元数据治理。

### Network Token 是否也要保护

Network Token 的目标是替代 PAN，降低 PAN 暴露面。它通常会绑定商户、设备、钱包或 token requestor，泄露后的影响范围比 PAN 小。

但它不是普通业务 ID。

如果攻击者同时拿到：

```text
Network Token
token expiry
cryptogram 或生成 cryptogram 的能力
商户上下文
相关支付参数
```

仍可能尝试交易或欺诈。

| 项目 | PAN | Network Token |
|---|---|---|
| 通用性 | 通常可跨商户、跨场景使用 | 通常绑定商户、设备、钱包或 token requestor |
| 泄露影响 | 影响大 | 影响范围更小 |
| 生命周期 | 卡过期、换卡会变 | 可做生命周期管理 |
| 风控 | 依赖传统卡风控 | 可结合 token domain、cryptogram、设备和商户绑定 |
| PCI 判断 | 明确 CHD | 需结合可逆性、映射关系和使用能力评估 |

注意：

- 如果 token 可以被系统映射回 PAN，token vault / 映射系统仍在 PCI Scope。
- 商户拿到的 Network Token 通常不是原始 PAN，但仍是支付凭证。
- 不要随便展示。
- 不要进普通日志。
- 不要导出到 BI。
- 不要放浏览器长期存储。
- token + cryptogram 要严格保护。

一句话：

> Network Token 降低 PAN 风险，不等于没有安全责任；它不是普通业务 ID，而是受控支付凭证。

### HSM

**HSM = Hardware Security Module**，硬件安全模块。

它是专门用于生成、保存和使用密钥的安全硬件。核心价值是：

> 密钥不以明文离开设备。

在支付场景中，HSM 常用于：

- 生成和保护主密钥
- PIN / PIN Block 加解密
- 交易签名或 MAC 计算
- 支付网关、Vault、收单系统中的高敏感密钥管理
- 密钥轮换、密钥分发和双人控制

一句话：

> HSM 是支付系统里的密钥保险柜和加密计算器。

### Card Vault

**Card Vault** 是专门保存和管理银行卡敏感数据的系统，可以理解为卡数据保险库。

它的核心作用：

> 把真实卡号 PAN 存在高安全环境里，对外只返回 token。

典型流程：

```text
用户输入卡信息
  ↓
Card Vault 接收并处理卡数据
  ↓
Vault 保存 PAN，不保存授权后的 CVV
  ↓
Vault 返回 token
  ↓
业务系统以后用 token 发起支付、绑卡、续费、退款
```

常见方案：

| 类型 | 说明 |
|---|---|
| PSP Vault | 支付服务商自带，成本低，但容易绑定单 PSP |
| 第三方 Vault | 如 VGS、Spreedly、Basis Theory，支持多 PSP 编排 |
| 自建 Vault | 控制权强，但 PCI、HSM、密钥、审计成本最高 |

一句话：

> Card Vault 是把真实卡号集中关进高安全区，让其他系统只拿 token 干活。

## 3. 前端支付页安全

### CSP

**CSP = Content Security Policy**，内容安全策略。

它是浏览器安全机制，用来限制页面可以加载哪些脚本、图片、样式、接口等资源。

示例：

```http
Content-Security-Policy: script-src 'self' https://checkout.psp.com https://cdn.example.com;
```

含义：

> 这个页面只能执行本站、指定 PSP 和指定 CDN 的脚本，其他来源脚本不准运行。

### SRI

**SRI = Subresource Integrity**，子资源完整性校验。

它用于确认外部 JS / CSS 文件有没有被篡改。

示例：

```html
<script
  src="https://cdn.example.com/sdk.js"
  integrity="sha384-abc123..."
  crossorigin="anonymous"></script>
```

浏览器会下载脚本后计算哈希。如果实际内容和 `integrity` 不匹配，就拒绝执行。

### CSP 与 SRI 区别

| 项目 | 解决的问题 |
|---|---|
| CSP | 限制哪些来源的脚本能运行 |
| SRI | 校验这个脚本内容有没有被改 |

支付页最佳实践：

> CSP 控来源，SRI 控完整性，脚本白名单控变更。

### Magecart 攻击

**Magecart 攻击** 是一种前端窃卡攻击。

它的做法是把恶意 JavaScript 注入到结账页或支付页里，在用户输入卡号、姓名、有效期、CVV 时，直接从浏览器页面窃取数据。

典型路径：

```text
第三方脚本 / CDN / npm 包 / 广告脚本被篡改
  ↓
恶意 JS 被加载到支付页
  ↓
监听输入框、表单提交或支付请求
  ↓
偷走 PAN、有效期、CVV、姓名、地址
  ↓
发送到攻击者控制的域名
```

危险点：

- 后端数据库可能完全没被入侵。
- HTTPS 挡不住，因为攻击发生在用户浏览器里。
- WAF 和后端日志不一定能发现。
- 一小段 JS 就可能长期窃取支付数据。

防御方式：

- CSP
- SRI
- 支付页脚本白名单
- 前端篡改检测
- 第三方脚本治理
- 减少 Tag Manager、广告、埋点、客服浮窗进入支付页

一句话：

> Magecart 是跑在用户浏览器里的前端窃卡器。

### 供应链攻击

供应链攻击是不直接攻击你的主系统，而是攻击你依赖的组件、服务或流程。

支付页常见入口：

- 第三方 JS SDK
- CDN 文件
- npm 包
- Tag Manager
- 客服浮窗
- 广告脚本
- 埋点 SDK
- A/B 测试平台
- 构建流水线

例子：

```html
<script src="https://cdn.some-vendor.com/widget.js"></script>
```

如果 vendor、CDN、npm 包或构建账号被入侵，攻击者改了 `widget.js`，你的支付页就会自动加载恶意代码。

## 4. Passkey、Click to Pay 与 Token

### Passkey 是什么

Passkey 属于 **FIDO2 / WebAuthn** 体系，是无密码认证技术。

它解决的问题是：

> 证明这个用户或这个设备是合法的。

Passkey 存的不是卡号，而是认证私钥。

```text
设备侧：passkey 私钥
服务端：公钥
认证时：设备用私钥签名挑战，服务端用公钥验证
```

Passkey 不是：

- 卡号
- CVV
- Network Token
- PSP Token
- Card Vault Token

### Click to Pay 是什么

Click to Pay 是卡组织推动的在线结账体验和标准体系，基于 EMVCo Secure Remote Commerce。

它解决的问题是：

> 让用户在不同商户中用一致的方式选择已登记卡片并完成在线结账。

典型结构：

```text
用户
  ↓ passkey / OTP / 设备认证
Click to Pay Profile
  ↓ 选卡，例如 Visa **** 1234
卡组织 / 发卡行 / Token Service Provider
  ↓ 提供 token 化支付凭证
商户 / PSP
  ↓ 发起授权
收单行 → 卡组织 → 发卡行
```

### 卡号 token 是在哪里做的

卡号令牌化通常不是商户页面本地完成的，而是由卡组织、发卡行或 Token Service Provider 这一侧完成。

可以理解为：

```text
用户绑卡 / 注册 Click to Pay
  ↓
卡组织 / 发卡行校验卡片
  ↓
TSP 生成或管理 Network Token
  ↓
Click to Pay 账户关联这张卡的 token 化凭证
  ↓
用户结账时选择 Click to Pay
  ↓
商户拿到 token / cryptogram / payment payload
```

Click to Pay 是结账入口和编排层，Tokenization 是底层支付网络能力。

### Token 存在哪里

通常不应该把可直接支付的长期 token 存在普通浏览器环境里。

| 数据 | 常见存放位置 |
|---|---|
| 真实卡号 PAN | 发卡行、卡组织/TSP、PSP Vault、第三方 Vault、自建 Card Vault |
| Network Token / EMV Payment Token | TSP、发卡行/卡组织网络、钱包/设备安全环境、PSP/Vault 侧 |
| Merchant Token / PSP Token | PSP Vault、第三方 Vault、商户自建 Vault |
| 浏览器端 | 临时会话数据、一次性 payment credential、短期 checkout session id |

浏览器端通常只短暂处理：

- masked card，例如 `Visa **** 1234`
- checkout session id
- 一次性 cryptogram
- 短时支付 payload

浏览器端不应该长期保存：

- PAN
- CVV
- 长期可复用 token

原因：

- XSS
- Magecart
- 恶意浏览器插件
- 被篡改的第三方 JS
- localStorage / sessionStorage 泄露
- 调试工具和日志误打

一句话：

> 浏览器负责用户交互、Passkey 认证和临时支付凭证中转；长期凭证由 Click to Pay / TSP / 发卡行 / PSP / Vault 托管。

## 5. KEK、DEK 与 KMS 最佳实践

### KEK 与 DEK

**DEK = Data Encryption Key**，数据加密密钥，用来加密真实数据，例如 PAN。

**KEK = Key Encryption Key**，密钥加密密钥，用来加密或包裹 DEK。

典型 envelope encryption 结构：

```text
PAN / 敏感数据
  ↓ 用 DEK 加密
DEK
  ↓ 用 KEK 包起来
KEK
  ↓ 由 KMS / HSM / Root Key 保护
```

### DEK 常见算法

DEK 通常是对称密钥。

推荐：

- AES-256-GCM
- AES-128-GCM
- AES-CBC + HMAC
- AES-CTR + HMAC
- 必须保持格式时，才考虑 FPE / FF1

不建议：

- AES-ECB
- 自研算法
- 单纯 AES-CBC 但没有 MAC
- 新系统继续使用 3DES
- 一个 DEK 长期加密所有 PAN

### KEK 是否必须非对称

不必须。

KEK 可以是对称密钥，也可以是非对称密钥，取决于系统设计。但在 AWS Secrets Manager 和常见 KMS envelope encryption 场景中，KEK 通常是 **KMS symmetric encryption key**。

AWS Secrets Manager 的典型模式：

```text
KMS symmetric key
  ↓
GenerateDataKey 生成 256-bit AES data key
  ↓
data key 加密 secret value
  ↓
encrypted data key 存在 metadata
```

结论：

> 在 AWS Secrets Manager 场景，KEK 不但不要求非对称，反而使用 symmetric KMS key。

非对称 KMS key 更常见于：

- 外部公钥加密
- 签名 / 验签
- 密钥协商
- 需要公开公钥给外部系统的场景

### 用 KMS 的推荐实践

一般最佳实践：

> 用 KMS 创建 KEK / Root Key，用 KMS 生成 DEK，再由应用侧用 DEK 加密数据。

流程：

```text
KMS 里创建 symmetric KMS key
  ↓
这个 KMS key 作为 KEK / Root Key
  ↓
应用调用 GenerateDataKey
  ↓
KMS 返回 plaintext_dek + encrypted_dek
  ↓
应用用 plaintext_dek 加密 PAN
  ↓
数据库保存 ciphertext + encrypted_dek + metadata
  ↓
应用清除 plaintext_dek
```

解密流程：

```text
数据库取 encrypted_dek + ciphertext
  ↓
应用调用 KMS Decrypt(encrypted_dek)
  ↓
拿到 plaintext_dek
  ↓
应用解密 ciphertext
  ↓
清除 plaintext_dek
```

数据库建议保存：

```text
ciphertext
encrypted_dek
key_id
key_version
iv / nonce
auth_tag
algorithm
encryption_context
```

权限建议：

- 支付写入服务可以 `GenerateDataKey`
- 支付授权服务可以 `Decrypt`
- BI、客服、测试环境不应该有 `kms:Decrypt`
- 使用 encryption context 绑定用途，例如：

```text
env=prod
purpose=pan-encryption
merchant_id=xxx
```

### KMS 里创建 KEK，还是创建 DEK

KMS 里长期创建和托管的是 KEK / Root Key。

DEK 通常不是长期手工创建在 KMS 里，而是通过 API 动态生成。

常用 API：

| API | 用途 |
|---|---|
| `CreateKey` | 创建 KMS key，作为 KEK / Root Key |
| `CreateAlias` | 给 key 起别名，例如 `alias/prod/pan-vault` |
| `GenerateDataKey` | 生成 plaintext DEK 和 encrypted DEK |
| `Decrypt` | 解开 encrypted DEK |
| `ReEncrypt` | 换 KEK 或重新包裹 DEK |
| `ScheduleKeyDeletion` | 删除 key，需极其谨慎 |

一句话：

> KMS 里建的是 KEK；DEK 用 `GenerateDataKey` 临时生成，明文 DEK 只在应用内存里短暂存在，密文 DEK 跟业务密文一起存。

## 6. 总体架构判断

推荐优先级：

```text
能不碰卡号，就不碰卡号
  ↓
能用 PSP Vault / Network Token / 第三方 Vault，就不要自建 Vault
  ↓
必须存 PAN 时，用 KMS + envelope encryption
  ↓
涉及 PIN / PIN Block / 收单核心密钥时，用支付 HSM
  ↓
CVV 授权后永远不存
```

关键边界：

- Passkey 负责认证用户身份。
- Click to Pay 负责在线结账体验和选卡编排。
- Network Token / EMV Payment Token 负责替代真实卡号参与授权。
- Card Vault 负责长期托管卡数据或 token。
- 浏览器只适合做临时交互和短期凭证中转。
- KMS 适合做 KEK / Root Key，不适合当高频 Card Vault。
- Secrets Manager 适合存 API key、数据库密码等 secret，不适合存大量 PAN。
