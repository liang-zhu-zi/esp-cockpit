# ESP-Cockpit 设备身份、恢复与审计线协议契约（补充规范草案）

> **文档状态**：补充规范（草案）。
> **权威关系**：本文定义设备证书轮换、受限本地身份恢复通道及本地安全审计日志的 HTTP 与 JSON-RPC 2.0 线协议契约，与配套 C API 声明保持一致。通用 JSON-RPC 信封、认证会话、`limits` 字段及既定错误码以[最小互操作契约（冻结基线）](https://www.google.com/search?q=./esp-cockpit-interoperability-baseline.md)为唯一权威来源；兼容性与扩展规则遵循[JSON-RPC 协议契约](https://www.google.com/search?q=./esp-cockpit-json-rpc-contract.md)。
> 
> 

---

## 1. 传输路由与兼容性规则

1. **双通道安全隔离**：
* **常规管理面（HTTPS/WSS）**：设备证书状态查询、在线发起本地 CSR 生成、导入轮换证书以及安全审计日志查询，均作为内建安全核心能力挂载在常规 HTTPS/WSS JSON-RPC 端点（`POST /rpc` 与 `/rpc/ws`）上。


* **本地身份恢复通道（受限隔离 SoftAP HTTP）**：仅当旧设备私钥损坏/丢失或证书已过期导致常规 HTTPS 不可用时，通过物理触发开启隔离 SoftAP 窗口，提供独立且极简的本地 HTTP 文件交换端点；窗口期内完全关闭常规 JSON-RPC 与管理接口。




2. **协议版本与能力声明**：
* 新增的 `esp_cockpit.cert.*` 与 `esp_cockpit.audit.*` 方法属于向后兼容的增量扩展，启用后使能声明的 JSON-RPC 协议次版本递增至 `protocolVersion: {"major": 1, "minor": 1}`（主版本保持 `1` 不变）。


* 实例启用对应能力时，必须在 `esp_cockpit.capabilities.get` 的 `methods` 数组中列出实际启用的完整方法名，并在 `capabilities` 数组中声明 `"esp_cockpit.cert_management"` 与 `"esp_cockpit.security_audit"`。




3. **64 位整数字段编码约定**：
* 冻结基线规定 JSON-RPC `integer` 类型取值限于有符号 `int32`。为避免 2038 年时间戳溢出及 64 位序列号/运行时间在浏览器端发生精度丢失，审计序号（`recordSeq`）、丢弃计数（`droppedCount`）、启动计数（`bootId`）、单调运行时间（`uptimeMs`）及 Unix 秒级时间戳（`notBeforeUnix`、`notAfterUnix`、`utcTimestamp`）在线协议中统一编码为**十进制整数字符串**（沿用冻结基线中 `sequence` 的编码约定）。




4. **PEM 字符串容量约束**：
* 证书链与 CSR 的 PEM 文本受 `limits.maxStringBytes`、`limits.maxRequestBytes` 和 `limits.maxResponseBytes` 共同约束。启用证书管理能力的产品配置必须将上述上限设为足以容纳完整 PEM 证书链的有限正整数（建议 `maxStringBytes >= 4096`、`maxRequestBytes >= 8192`）；超限请求在传输层返回 HTTP `413` 或在 RPC 层返回 `-32005`（`resource_limit`），不得截断。





---

## 2. 常规管理面：设备证书与可信时间 JSON-RPC 契约

本节所有方法均要求**已认证管理员会话**。经 HTTP `POST /rpc` 调用时，必须校验 `esp_cockpit_session` Cookie 与精确 `Origin`；其中具有状态变更语义的方法（`csr.generate`、`cert.import`）还必须携带有效的 `X-CSRF-Token` 请求头。所有响应均须附带 `Cache-Control: no-store`。

### 2.1 `esp_cockpit.cert.info.get`（查询活动证书与时钟状态）

对应 C 接口 `esp_cockpit_get_certificate_info()` 及时钟可信状态，供通用 WebUI 在登录后展示证书有效期及到期/不可信时钟告警。

* **请求 `params**`：省略或 `{}`（传入 `null`、数组或未知字段返回 `-32602`）。


* **成功 `result**`：
```json
{
  "subjectCn": "esp-9f8e7d6c5b4a.local",
  "sanDns": "esp-9f8e7d6c5b4a.local",
  "notBeforeUnix": "1767225600",
  "notAfterUnix": "1830297600",
  "timeTrusted": true,
  "isExpired": false,
  "expiryWarning": false,
  "hasPendingCsr": false
}

```


* **字段语义**：
* `timeTrusted`（boolean）：设备当前 UTC 时钟是否已通过可信时间源校准。当 `timeTrusted` 为 `false` 时，`isExpired` 与 `expiryWarning` 固定返回 `false`，前端须提示管理员当前设备时钟不可信、证书有效期校验处于待校准状态。


* `expiryWarning`（boolean）：仅当 `timeTrusted` 为 `true` 且当前时间距 `notAfterUnix` 小于产品配置的告警提前量时为 `true`。


* `hasPendingCsr`（boolean）：设备本地是否已生成尚未完成证书导入的新私钥。





### 2.2 `esp_cockpit.cert.csr.generate`（生成本地密钥对与 CSR）

对应 C 接口 `esp_cockpit_generate_csr()`。设备在本地安全存储中生成新私钥（私钥永不导出）并返回 PKCS#10 CSR PEM 文本，供管理员下载后提交厂商 HTTPS 签发门户。重复调用将覆盖尚未导入证书的旧待轮换私钥，不影响当前正在服务的活动证书。

* **请求 `params**`：省略或 `{}`。
* **成功 `result**`：
```json
{
  "csrPem": "-----BEGIN CERTIFICATE REQUEST-----\n...\n-----END CERTIFICATE REQUEST-----\n",
  "hasPendingCsr": true
}

```


* **错误语义**：
* 底层密钥生成或缓冲分配失败返回 `-32000`（`internal_error`），不泄露底层加密库错误码。





### 2.3 `esp_cockpit.cert.import`（导入签发证书并原子切换）

对应 C 接口 `esp_cockpit_import_certificate()`。

* **请求 `params**`：
```json
{
  "certPem": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----\n"
}

```


* `certPem`（string，必填）：合法 UTF-8 编码的 PEM 格式设备证书（可包含受控中间 CA 证书链），首尾允许空白，不得包含空字节。


* **校验与切换规则**：
1. 确认设备存在待轮换的本地私钥（否则返回 `-32013` `no_pending_csr`）。
2. 依次校验证书格式、目标根/中间 CA 信任链、SAN 主机名与设备固定 `device_hostname` 完全一致、扩展密钥用途（EKU）包含 `TLS Web Server Authentication`、有效期边界（当 `timeTrusted` 为 `true` 时校验 `notBefore <= now <= notAfter`），以及证书公钥与本地新私钥精确匹配。


3. 任一校验失败则丢弃输入并保留原活动证书，返回 `-32012`（`cert_validation_failed`）及安全用户提示。


4. 全部校验通过后，原子提交新证书与新私钥至受保护存储、清除待轮换标记、写入安全审计事件 `cert_rotated`。已建立的 HTTPS/WSS 连接与当前管理员会话 Cookie 在本次连接生命周期内保持有效，后续新建 TLS 握手立即使用新证书。




* **成功 `result**`：
```json
{
  "imported": true,
  "notBeforeUnix": "1798761600",
  "notAfterUnix": "1861920000"
}

```



---

## 3. 受限本地身份恢复通道 HTTP 端点契约

当旧设备私钥损坏/丢失或证书已过期（导致浏览器拒绝建立 HTTPS 连接）时，用户通过物理按键调用 `esp_cockpit_open_identity_recovery_window()` 开启受限恢复窗口。

### 3.1 安全隔离与暴露面约束

* **接入隔离**：仅在受设备独立高熵强密码保护的隔离 SoftAP 上提供服务，并在窗口超时或证书导入成功后立即关闭 SoftAP 与 HTTP 监听器。


* **功能白名单**：恢复窗口开放期间，**不挂载** `POST /rpc`、`/rpc/ws`、登录接口及任何业务端点（访问一律返回 `404 Not Found`）。


* **零远程凭据接触**：本通道绝不要求用户输入或提交一次性恢复码、厂商账户密码或管理员会话 Token（上述核验仅在用户使用另一台联网设备访问厂商 HTTPS 恢复门户时发生）。


* **安全响应头**：所有响应必须包含 `Cache-Control: no-store`、`X-Content-Type-Options: nosniff`、`X-Frame-Options: DENY` 及禁止内联/外部脚本的限制性 `Content-Security-Policy`。



### 3.2 恢复端点定义

| 方法与路径 | 请求要求 | 成功响应 | 职责与行为说明 |
| --- | --- | --- | --- |
| `GET /` | 无请求体 | `200 OK` (`text/html; charset=utf-8`) | 返回固件内置的本地身份恢复静态单页（仅含 CSR 下载与证书上传表单）|
| `GET /recovery/status` | 无请求体 | `200 OK` (`application/json`) | 返回 `{"active": true, "remainingSeconds": 540, "hasPendingCsr": true, "deviceHostname": "esp-9f8e7d6c5b4a.local"}` |
| `POST /recovery/csr` | 空请求体 `{}` (`application/json`)；校验 `Origin` 匹配 SoftAP 主机 | `200 OK` (`application/x-pem-file`) | 调用本地密钥/CSR 生成（若已有未消费的 `hasPendingCsr` 则幂等返回该 CSR，避免误点刷新导致刚下载的 CSR 失效），附带 `Content-Disposition: attachment; filename="device.csr"`  |
| `POST /recovery/cert` | `Content-Type: application/x-pem-file`（原始 PEM 字节流）或 `application/json` (`{"certPem": "..."}`)；大小受 `maxRequestBytes` 限制；校验 `Origin` | `200 OK` (`application/json`) `{"imported": true}` | 执行与 `esp_cockpit.cert.import` 完全一致的链、SAN、EKU、有效期及公钥匹配校验；成功后原子切换身份、记录 `cert_rotated` 审计，并在响应发送后立即关闭恢复窗口与 SoftAP|
| `POST /recovery/cancel` | 空请求体 `{}` (`application/json`)；校验 `Origin` | `200 OK` (`application/json`) `{"closed": true}` | 提前手动关闭恢复窗口并退出恢复模式 |

### 3.3 恢复端点 HTTP 错误映射

| HTTP 状态码 | 响应体 (`application/json`) | 触发条件 |
| --- | --- | --- |
| `400` | `{"error": "invalid_request", "message": "..."}` | 报文格式错误或缺少 `certPem` |
| `403` | `{"error": "forbidden_origin", "message": "..."}` | `Origin` 头缺失或不匹配恢复 SoftAP 地址 |
| `404` | `{"error": "not_found", "message": "..."}` | 访问白名单之外的任意路径（含 `/rpc`） |
| `405` | `{"error": "method_not_allowed", "message": "..."}` | 使用不支持的 HTTP 方法（如对变更端点使用 `GET`） |
| `413` | `{"error": "payload_too_large", "message": "..."}` | 上传证书超过 `maxRequestBytes`|
| `415` | `{"error": "unsupported_media_type", "message": "..."}` | `Content-Type` 不符合要求 |
| `422` | `{"error": "cert_validation_failed", "message": "..."}` 或 `no_pending_csr` | 证书链、SAN、用途、有效期或私钥不匹配（原身份保持不变） |
| `429` | `{"error": "rate_limited", "message": "..."}` | 密钥生成或证书校验频率超限 |

---

## 4. 本地安全审计日志 JSON-RPC 契约

对应 C 接口 `esp_cockpit_audit_read()`，供已认证管理员在常规管理面分页查看安全事件；前端可基于分页结果在浏览器本地组装并导出 JSON/CSV 文件，无需在组件端增设额外文件下载端点。

### 4.1 `esp_cockpit.audit.list`

* **访问控制**：仅限已认证管理员会话；经 HTTP `POST /rpc` 调用时校验 `Cookie` 与 `Origin`（只读查询不修改设备状态）。响应必须设置 `Cache-Control: no-store`。


* **请求 `params**`：
```json
{
  "startSeq": "101",
  "limit": 20
}

```


* `startSeq`（string，可选）：起始读取的记录序号（无符号十进制字符串）。省略或传入 `"0"` 表示从当前环形缓冲区中保留的最旧记录开始读取。
* `limit`（integer，必填）：本页最大返回条数，必须为正 `int32` 且不超过 `limits.maxSchemaPageItems`（否则返回 `-32602` 或 `-32005`）。




* **成功 `result**`：
```json
{
  "items": [
    {
      "recordSeq": "101",
      "bootId": "4",
      "uptimeMs": "182450",
      "utcTimestamp": null,
      "timeValid": false,
      "actor": "unauthenticated",
      "eventType": "login_failure",
      "count": 3
    },
    {
      "recordSeq": "102",
      "bootId": "4",
      "uptimeMs": "240110",
      "utcTimestamp": "1767226800",
      "timeValid": true,
      "actor": "admin",
      "eventType": "cert_rotated",
      "count": 1
    }
  ],
  "nextStartSeq": "103",
  "droppedCount": "15"
}

```


* **字段语义与隐私红线**：
* `items` 按 `recordSeq` 严格升序排列；当本次返回已抵达最新记录时，`nextStartSeq` 为 `null`。若请求的 `startSeq` 已被环形缓冲区覆盖淘汰，服务端自动从当前最旧保留记录开始返回，客户端可通过首条记录的 `recordSeq` 与 `droppedCount` 感知覆盖情况。


* `utcTimestamp`（string | null）：仅当记录产生时 `timeValid` 为 `true`，返回有符号 64 位 Unix 秒级时间戳字符串；`timeValid` 为 `false` 时固定为 `null`，严禁将未校准设备时钟作为有效 UTC 输出。


* `actor`（string）：枚举值固定为 `"admin"` 或 `"unauthenticated"`，严格遵守单管理员产品约束，不记录用户名或区分个人操作者。


* `eventType`（string）：枚举值固定为 `"init"`、`"login_success"`、`"login_failure"`、`"logout"`、`"password_changed"`、`"session_revoked"`、`"cert_rotated"`、`"recovery_window"`、`"factory_reset"`。


* `count`（integer）：高频重复事件（如短时间多次登录失败）在落盘前聚合的次数，未聚合事件固定为 `1`。


* **敏感字段脱敏**：审计记录结构及响应中严禁包含明文密码、恢复码、Session Cookie、CSRF Token、Wi-Fi 凭据或客户端 IP 地址。执行 `esp_cockpit_factory_reset()` 后，旧记录全部擦除且 `droppedCount` 归零，新日志首条记录为 `eventType: "factory_reset"`。





---

## 5. 新增组件保留错误码

在[冻结互操作基线第 4 节](https://www.google.com/search?q=./esp-cockpit-interoperability-baseline.md%234-%E7%A8%B3%E5%AE%9A%E9%94%99%E8%AF%AF%E7%A0%81)已分配的 `-32000..-32011` 基础上，为设备身份操作新增以下保留组件错误码（位于 `-32099..-32000` 保留区间内，不包含 `error.data`）：

| Code | 名称 | 使用场景与客户端处理建议 |
| --- | --- | --- |
| `-32012` | `cert_validation_failed` | 导入的证书链验证失败、SAN 主机名与设备不匹配、缺少 TLS Server Auth 用途、证书在可信时间下已过期/尚未生效，或公钥与本地待轮换私钥不匹配。设备保持原有效证书不变，客户端提示用户核对签发文件后重新上传。 |
| `-32013` | `no_pending_csr` | 调用 `esp_cockpit.cert.import` 时设备本地不存在尚未消费的待轮换私钥。客户端应先调用 `esp_cockpit.cert.csr.generate` 生成 CSR 并完成签发后再导入。 |

---

## 6. C API 与线协议映射及测试要点对照

| C API 函数 | 对应线协议接口 | 传输与鉴权边界 | 自动化验证要点（Unity / `pytest-embedded`） |
| --- | --- | --- | --- |
| `esp_cockpit_get_certificate_info()`<br><br>`esp_cockpit_set_time_trusted()` | `esp_cockpit.cert.info.get` | HTTPS/WSS JSON-RPC；需有效管理员会话 | 验证 `timeTrusted=false` 时不误报过期；校准时间后进入告警窗口时 `expiryWarning` 准确置 `true`。 |
| `esp_cockpit_generate_csr()` | `esp_cockpit.cert.csr.generate` | HTTPS/WSS JSON-RPC；需会话 + CSRF + Origin | 验证生成合法 PKCS#10 PEM、私钥不可通过任何接口导出、重复生成不破坏当前活动 TLS 连接。 |
| `esp_cockpit_import_certificate()` | `esp_cockpit.cert.import` | HTTPS/WSS JSON-RPC；需会话 + CSRF + Origin | 注入错误 CA 链、错误 SAN、ClientAuth EKU、过期证书及不匹配公钥，验证均返回 `-32012` 且旧证书持续可用；合法证书导入后新建 TLS 握手生效。 |
| `esp_cockpit_open_identity_recovery_window()` | `GET /`<br><br>`GET /recovery/status`<br><br>`POST /recovery/csr`<br><br>`POST /recovery/cert` | 仅限物理触发后的隔离 SoftAP；强随机 AP 密码 + 有界时窗 + 端点白名单 | 验证窗口内访问 `/rpc` 返回 `404`；刷新页面重复 `POST /recovery/csr` 幂等返回同一待轮换 CSR；导入成功或超时后自动关闭 SoftAP。 |
| `esp_cockpit_audit_read()` | `esp_cockpit.audit.list` | HTTPS/WSS JSON-RPC；需有效管理员会话 | 验证环形覆盖后 `droppedCount` 准确、不可信时钟下 `utcTimestamp` 为 `null`、连续登录失败触发聚合、恢复出厂后清空前任历史。 |
