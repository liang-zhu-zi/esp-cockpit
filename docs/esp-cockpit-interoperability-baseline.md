# ESP-Cockpit 最小互操作契约（冻结基线）

> 状态：规范性冻结，适用于首版组件、WebUI 和示例客户端的并行实现。本文与 [JSON-RPC 协议契约](./esp-cockpit-json-rpc-contract.md) 共同构成线协议及公共 C API 的基线。`limits` 字段名、单位和语义已冻结；各生产配置的有限数值须经目标板测量后写入能力发现，不得省略、改名或用 `null` 代替。

## 1. 通用规则

- 线协议为 JSON-RPC 2.0，协议版本在 `protocolVersion` 对象中表示为非负整数 `major`、`minor`。首版版本为 `1.0`。所有客户端方法调用必须使用非空字符串 `id`；其 UTF-8 字节数不得超过 `limits.maxIdBytes`。不接受客户端 Notification、Batch、位置参数数组或 `id: null`。
- 所有方法只接受命名参数对象。未知参数、缺少必需参数或类型错误均以 JSON-RPC `-32602` 拒绝，不调用应用回调。除明确说明可省略的 `params` 外，`params: null` 不等同于空对象。
- JSON-RPC 成功响应使用标准信封，方法结果位于 `result`；RPC 失败位于 `error: {code, message}`。错误 `message` 是面向用户的安全提示；首版不发送 `error.data`。响应新增字段客户端必须忽略；必需字段缺失或类型错误视为契约错误。
- 字符串均为合法 UTF-8；普通协议字符串上限为 `limits.maxStringBytes` 字节，错误提示上限为 `limits.maxErrorMessageBytes` 字节。所有资源拒绝均明确报错，不得截断、丢弃字段或伪装成功。

## 2. 冻结的核心 RPC

能力发现的 `methods` 必须列出实例实际启用的下列核心方法；未启用的可选插件或应用方法不列出。

| 方法 | `params` | 成功 `result` |
|---|---|---|
| `esp_cockpit.auth.status` | 省略或 `{}` | `{initialized: boolean, initializationAvailable: boolean}` |
| `esp_cockpit.auth.initialize` | `{password: string}` | `{csrfToken: string}` |
| `esp_cockpit.auth.login` | `{password: string}` | `{csrfToken: string}` |
| `esp_cockpit.auth.logout` | `{}` | `{loggedOut: true}` |
| `esp_cockpit.auth.password.change` | `{currentPassword: string, newPassword: string}` | `{changed: true}`; 清除当前 Cookie |
| `esp_cockpit.capabilities.get` | 省略或 `{}` | 第 3 节定义的能力对象 |
| `esp_cockpit.schema.list` | `{clusterId: string, limit: integer, cursor?: string}` | 第 3 节定义的 Schema 页面 |
| `esp_cockpit.read` | `{items: [{path}]}` | `{items: [{path, status, value?, version?, error?}]}` |
| `esp_cockpit.write` | `{items: [{path, value, expectedVersion}]}` | `{items: [{path, status, version?, error?}]}` |
| `esp_cockpit.invoke` | `{clusterId: string, commandId: string, arguments: object}` | 同步命令：`{status: "completed", result: JSON value}`；长操作：`{status: "accepted", operationId: string}` |
| `esp_cockpit.subscribe` | `{items: [{path}]}` | `{subscriptionId: string, items: [{path, value, version, sequence}]}` |
| `esp_cockpit.unsubscribe` | `{subscriptionId: string}` | `{removed: boolean}` |
| `esp_cockpit.operation.get` | `{operationId: string}` | `{operationId, state, progress, result, error}` |

`path` 是 `{clusterId: string, attributeId: string}`；`clusterId` 使用完整 `endpoint/cluster` 标识。Read/Write/Subscribe 的 `items` 数量不得超过 `limits.maxMultiPathItems`；同一调用中不允许重复路径。Read/Write 结果与请求顺序一致，每项 `status` 为 `ok` 或 `error`。逐项失败使用 `{code, message}` 放在该项的 `error` 中；整体无法执行时使用顶层 JSON-RPC `error`。Read 成功项包含 `value` 和 `version`；Write 成功项包含新 `version`。只写属性的 Read 只返回版本及必要元数据，不返回值。

`version` 是不透明、非空字符串，表示当前运行期该路径值的版本；仅在值变化时推进，设备重启后失效。客户端只可原样传入 `expectedVersion`，不得解析或自行构造。Write 版本冲突仅使对应项失败，不影响其他项。

每个 Invoke 命令的 Schema 必须有 `retrySafe: boolean`。缺少或不理解该字段一律视为 `false`。仅当 `retrySafe` 为 `true` 时客户端可向用户提供显式重试；不得自动重放已开始但结果未知的 Write/Invoke。

Operation 状态固定为 `queued`、`running`、`succeeded`、`failed`。`operation_begin()` 创建 `queued` 项并启动 `maxOperationDurationMs` 计时；应用工作开始时首次调用 `operation_set_progress()` 将其转为 `running`。`progress` 固定为 `null` 或 `{completed: non-negative integer, total: positive integer}`；同一 Operation 的 `total` 固定且 `completed` 单调不减。未到终态时 `result`、`error` 均为 `null`，成功终态包含 `result`（无值时为 `null`）且 `error: null`，失败终态包含 `error: {code, message}` 且 `result: null`。达到最长运行期后组件将 Operation 标记为 `failed`，错误码为 `-32011`；组件不强制中断应用任务，应用须自行确保任务有界，并在过期后停止尝试更新该 Operation。首版不支持客户端取消；活动项不得驱逐，满载时只能回收最旧终态，否则拒绝新操作。

## 3. 能力发现、Schema 与通知

`esp_cockpit.capabilities.get` 的 `result` 必须且只需包含以下必需字段；未知新增字段可忽略：

```json
{
  "protocolVersion": {"major": 1, "minor": 0},
  "methods": [],
  "capabilities": [],
  "clusters": [{"id": "room1/temperature", "schemaVersion": 1}],
  "limits": {}
}
```

`methods`、`capabilities`、`clusters` 总是数组，`limits` 总是对象。列表按标识符 ASCII 字典序排序。Schema 分页的 `result` 为 `{clusterId, schemaVersion, items, nextCursor}`；`nextCursor` 为不透明字符串或 `null`。游标失效使用 `-32009`。Schema `items[].kind` 首版为 `attribute`、`command` 或 `method`；未知 kind 或未知语义规则须隔离该 Cluster。

Schema 条目字段固定如下：

- `attribute`：必需 `kind`, `id`, `type`, `access`, `nullable`；`type` 为 `boolean`、`integer`、`number`、`string` 或 `enum`。`access` 是非空数组，成员为 `read`、`write`。可选 `minimum`、`maximum`（数值且包含边界）、`maxBytes`（UTF-8 字节数）、`enumValues`（稳定字符串值数组）、`enumLabels`（与 `enumValues` 等长的显示标签数组）、`label`、`unit` 和 `default`。只有适用的约束字段可以出现；`enum` 必须有非空 `enumValues`，其他类型不得带枚举字段。
- `command` 与 `method`：必需 `kind`, `id`, `arguments`, `result`, `retrySafe`；`arguments` 始终为数组，每项必需 `id`, `type`, `required`, `nullable`，并可带与属性相同的类型约束及 `label`/`unit`/`default`。`result` 为同样的类型描述对象但不含 `id`、`required`、`access`、`default`；无业务返回值用 `{type: "null", nullable: true}` 表示，`null` 类型仅允许用于结果描述，可空业务值仍使用其实际类型。方法 `id` 是完整 RPC 方法名；命令 `id` 是 Cluster 内局部标识。`retrySafe` 为布尔值。

`integer` 取值限于有符号 `int32`；`number` 必须是有限数值；字符串长度按 UTF-8 字节计算。`default` 只作为 UI 建议，不得因省略参数或加载界面而自动写入设备。未知展示字段可忽略，未知类型、权限或约束必须隔离所属 Cluster。显示标签不参与值比较或 RPC 调用。

服务器通知不带 `id`：

- `esp_cockpit.subscription.report`：`params: {subscriptionId, updates: [{path, value, version, sequence}]}`。`sequence` 是从 `"1"` 开始递增的无符号十进制字符串，按每个订阅路径独立计数；客户端按数值顺序应用，忽略重复/较旧序号。合并更新可跳号，不要求重读。
- `esp_cockpit.subscription.terminated`：`params: {subscriptionId, reason}`；`reason` 仅为 `resource_limit` 或 `read_failed`。不得附带内部诊断信息。

订阅仅属于创建它的 WebSocket 连接；初始快照与后续更新不得有遗漏，成功响应必须先于该订阅的首条通知。Unsubscribe 对格式有效但不存在/已终止的订阅是成功 no-op，并以 `removed: false` 表示。

## 4. 稳定错误码

协议标准码依 JSON-RPC 2.0。组件保留码固定如下，禁止重用或改变既有含义：

| Code | 名称及使用场景 |
|---:|---|
| `-32000` | `internal_error`：组件内部错误 |
| `-32001` | `unauthenticated`：缺失/失效会话，或登录凭据无效 |
| `-32002` | `not_authorized`：已认证但不允许访问目标 |
| `-32003` | `version_conflict`：Write 的预期路径值版本已过期（逐项错误） |
| `-32004` | `not_found`：未知 Cluster、Operation 或资源 |
| `-32005` | `resource_limit`：请求/响应/schema/订阅/Operation 等超过公布上限 |
| `-32006` | `busy`：等待队列满、参数预算不足或排队超时 |
| `-32007` | `rate_limited`：RPC 分发后检测到的认证/操作速率限制；登录准入在分发前拒绝时使用 HTTP `429` |
| `-32008` | `initialization_unavailable`：设备已初始化、尚未初始化可登录或物理触发初始化窗口未开放/已关闭 |
| `-32009` | `stale_cursor`：游标对应的 Schema 修订已变化 |
| `-32010` | `subscription_unavailable`：订阅刷新机制故障，暂不能建立订阅 |
| `-32011` | `operation_timed_out`：Operation 超过公布的最长运行期 |

应用错误码使用 `1..INT32_MAX` 内由应用命名空间声明的固定正整数区间；各应用区间互不重叠。错误提示可选，但业务错误码不得映射为组件码；底层 ESP-IDF 错误码不得透传。未知错误码按通用失败处理。

## 5. HTTP / WebSocket 映射

| 情况 | HTTP 状态 / JSON-RPC 行为 |
|---|---|
| 已进入 RPC 分发（含解析错误、无效参数、未知方法及方法错误） | HTTP `200`，使用 JSON-RPC `result` 或 `error`；解析错误使用 `-32700` |
| 非法 HTTP 报文、缺少/错误 `Content-Type`、不支持的 HTTP 方法 | `400`、`415`、`405`，不分发 RPC |
| 请求体超过 `maxRequestBytes` | `413`，不分发 RPC |
| 普通管理 RPC 无有效管理员会话；初始化/登录/状态查询除外 | `401`，不分发 RPC |
| 浏览器 `Origin` 不匹配，或已认证状态变更请求缺少/错误 CSRF token | `403`，不分发 RPC；初始化和登录免 CSRF，但仍校验 Origin |
| 登录限速或认证资源准入失败 | `429`，不执行密码 KDF |
| Web 服务暂不可用或预分发资源不足 | `503`，不分发 RPC |
| 不存在的 HTTP 路径 | `404` |

HTTP JSON-RPC endpoint 固定为 `POST /rpc`，请求体为 UTF-8 `application/json`；WebSocket endpoint 固定为 `/rpc/ws`。HTTPS/WSS 是常规管理面的唯一传输。WebSocket Upgrade 的认证/Origin 失败分别返回 `401`/`403`。Upgrade 成功后所有逐消息结果只使用 JSON-RPC 信封。认证/Origin/CSRF 等预分发拒绝不返回 JSON-RPC 信封；已经分发的 JSON-RPC 错误不能改用非 `200` HTTP 状态码。

## 6. 认证与管理员初始化

- 设备不设用户名、访客或角色，仅有一个本地管理员；不提供共享默认密码。所有管理操作要求已认证管理员会话。
- 首次初始化仅能在设备物理触发后开放的 10 分钟窗口中经已验证 HTTPS 完成，不使用独立 claim code。窗口中常规管理/RPC 不可用，仅 `auth.status`、`auth.initialize`、`auth.login` 可用；初始化仅允许一个客户端原子成功，成功立即关闭窗口。超时关闭；再次开放必须再次物理触发。忘记密码只能经产品定义的物理恢复出厂/重新初始化流程处理，不提供仅凭 Cookie 的密码重置。
- `auth.initialize` 和 `auth.login` 的密码按同一规则校验：去除首尾 ASCII 空格后为 12–128 字节 ASCII，保留内部空格，拒绝控制字符和离线弱密码拒绝列表命中；超限不得截断。登录失败使用 `-32001`，受限使用 `-32007`。在执行高成本 KDF 前实施限速，并限制并发验证和排队；不得永久锁定管理员。
- 登录凭据校验失败在 RPC 已分发后使用 HTTP `200` 与 `-32001`；限速器在密码 KDF 前拒绝的请求不进入 RPC，使用 HTTP `429`。仅当速率限制是在 RPC 分发后判定时使用 `-32007`。
- 登录/初始化成功时服务端旋转会话 ID，设置 host-only Cookie `esp_cockpit_session`（`Path=/; Secure; HttpOnly; SameSite=Strict`，绝对存续上限 24 小时），并在 RPC `result.csrfToken` 返回状态变更请求使用的 CSRF token。所有浏览器 HTTP 请求均须精确校验 `Origin`；除初始化和登录外，状态变更请求必须同时携带该 Cookie、`X-CSRF-Token` 和匹配的 `Origin`。WebSocket 使用同一 Cookie 并校验 `Origin`。
- 会话空闲 30 分钟或绝对存续 24 小时失效；只有成功的用户发起认证操作续期。心跳、后台轮询和自动重连不续期。会话只保存在 RAM，设备重启后失效。Logout 撤销当前会话；改密成功后撤销所有会话（包括当前会话），要求重新登录。
- `auth.password.change` 必须重新验证当前密码；`auth.logout` 与改密均须 CSRF 校验。Logout 和改密成功均以 `Max-Age=0` 清除当前 `esp_cockpit_session` Cookie；改密或全局撤销时关闭所有关联 WebSocket。认证失败不得泄露管理员是否存在、密码校验阶段或内部原因。认证和密码相关响应使用 `Cache-Control: no-store`。

## 7. 公共 C 注册/回调 API

以下名称、字段、签名及所有权规则为 C API 主版本 `1` 的源代码契约；与 [网络生命周期 API](./esp-cockpit-network-lifecycle.md) 合用。实现应将其声明在 `esp_cockpit.h`，不承诺跨 ESP-IDF/工具链/构建配置的二进制 ABI。

```c
#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>
#include "esp_err.h"

#define ESP_COCKPIT_API_MAJOR 1
#define ESP_COCKPIT_ACCESS_READ  (1u << 0)
#define ESP_COCKPIT_ACCESS_WRITE (1u << 1)

typedef struct esp_cockpit *esp_cockpit_handle_t;

typedef enum {
    ESP_COCKPIT_VALUE_NULL = 0,
    ESP_COCKPIT_VALUE_BOOL = 1,
    ESP_COCKPIT_VALUE_INT32 = 2,
    ESP_COCKPIT_VALUE_DOUBLE = 3,
    ESP_COCKPIT_VALUE_STRING = 4,
} esp_cockpit_value_type_t;

typedef struct {
    esp_cockpit_value_type_t type;
    union {
        bool boolean;
        int32_t int32;
        double number;
        struct { const char *data; size_t length; } string;
    } as;
} esp_cockpit_value_t;

typedef struct {
    const char *data;
    size_t length;
} esp_cockpit_text_t;

typedef struct {
    const char *id;
    esp_cockpit_value_type_t type;
    uint32_t access;            /* attributes: ESP_COCKPIT_ACCESS_READ / _WRITE; arguments: 0 */
    bool required;              /* applies to arguments; attributes/results set true */
    bool nullable;
    bool has_default;           /* defaults are UI suggestions, never implicit writes */
    esp_cockpit_value_t default_value;
    bool has_minimum;
    bool has_maximum;
    int32_t int32_min, int32_max;
    double number_min, number_max;
    size_t max_bytes;           /* strings only; 0 means maxStringBytes */
    const char *const *enum_values;
    size_t enum_value_count;
    const char *const *enum_labels; /* optional; if present, count matches enum_value_count */
    size_t enum_label_count;
    const char *label;          /* optional display label */
    const char *unit;           /* optional numeric unit */
} esp_cockpit_parameter_descriptor_t;
typedef esp_cockpit_parameter_descriptor_t esp_cockpit_attribute_descriptor_t;
typedef struct esp_cockpit_operation *esp_cockpit_operation_handle_t;

typedef struct {
    const char *id;
    bool present;               /* always true for a declared required argument */
    esp_cockpit_value_t value;
} esp_cockpit_argument_t;

typedef enum {
    ESP_COCKPIT_CALLBACK_OK = 0,
    ESP_COCKPIT_CALLBACK_ACCEPTED = 1,
    ESP_COCKPIT_CALLBACK_CONFLICT = 2,
    ESP_COCKPIT_CALLBACK_RESOURCE_LIMIT = 3,
    ESP_COCKPIT_CALLBACK_ERROR = 4,
} esp_cockpit_callback_status_t;

typedef struct {
    int32_t code;               /* 0 or an application-owned positive error code */
    esp_cockpit_text_t message; /* optional UTF-8; absent is NULL + 0 */
} esp_cockpit_callback_error_t;

typedef struct {
    bool has_value;             /* false only for version-only Read */
    esp_cockpit_value_t value;  /* meaningful when has_value is true */
    char *string_buffer;        /* component-owned, prefilled for STRING output */
    size_t string_capacity;
    esp_cockpit_text_t version; /* required opaque runtime version token */
} esp_cockpit_read_result_t;

typedef esp_cockpit_callback_status_t (*esp_cockpit_read_fn)(
    const char *attribute_id, bool version_only,
    esp_cockpit_read_result_t *out, esp_cockpit_callback_error_t *error,
    void *user_ctx);
typedef esp_cockpit_callback_status_t (*esp_cockpit_write_fn)(
    const char *attribute_id, const esp_cockpit_value_t *value,
    esp_cockpit_text_t expected_version, esp_cockpit_text_t *out_version,
    esp_cockpit_callback_error_t *error, void *user_ctx);
typedef esp_cockpit_callback_status_t (*esp_cockpit_invoke_fn)(
    const esp_cockpit_argument_t *arguments, size_t argument_count,
    esp_cockpit_value_t *out_result,
    esp_cockpit_operation_handle_t *out_operation,
    esp_cockpit_callback_error_t *error, void *user_ctx);

typedef struct {
    const char *id;
    const esp_cockpit_parameter_descriptor_t *arguments;
    size_t argument_count;
    esp_cockpit_parameter_descriptor_t result;
    bool retry_safe;
    esp_cockpit_invoke_fn invoke;
    void *user_ctx;
} esp_cockpit_command_descriptor_t;

typedef struct {
    const char *id;
    uint32_t schema_version;
    const esp_cockpit_attribute_descriptor_t *attributes;
    size_t attribute_count;
    const esp_cockpit_command_descriptor_t *commands;
    size_t command_count;
    esp_cockpit_read_fn read;
    esp_cockpit_write_fn write;
    void *user_ctx;
} esp_cockpit_cluster_descriptor_t;

typedef struct {
    const char *method;
    const char *cluster_id;
    const esp_cockpit_parameter_descriptor_t *arguments;
    size_t argument_count;
    esp_cockpit_parameter_descriptor_t result;
    bool retry_safe;
    esp_cockpit_invoke_fn invoke;
    void *user_ctx;
} esp_cockpit_method_descriptor_t;

typedef struct esp_cockpit_app_descriptor {
    const char *namespace_id;
    const esp_cockpit_cluster_descriptor_t *clusters;
    size_t cluster_count;
    const esp_cockpit_method_descriptor_t *methods;
    size_t method_count;
    const char *const *capabilities;
    size_t capability_count;
    int32_t app_error_first, app_error_last;
} esp_cockpit_app_descriptor_t;

esp_err_t esp_cockpit_new(const esp_cockpit_config_t *config,
                          const esp_cockpit_app_descriptor_t *app,
                          esp_cockpit_handle_t *out_handle);
esp_err_t esp_cockpit_start(esp_cockpit_handle_t handle);
esp_err_t esp_cockpit_stop(esp_cockpit_handle_t handle);
esp_err_t esp_cockpit_del(esp_cockpit_handle_t handle);
esp_err_t esp_cockpit_attribute_changed(esp_cockpit_handle_t handle,
                                       const char *endpoint_id,
                                       size_t endpoint_id_len,
                                       const char *cluster_id,
                                       size_t cluster_id_len,
                                       const char *attribute_id,
                                       size_t attribute_id_len);
esp_err_t esp_cockpit_operation_begin(esp_cockpit_handle_t handle,
                                     esp_cockpit_operation_handle_t *out_operation);
esp_err_t esp_cockpit_operation_set_progress(
    esp_cockpit_operation_handle_t operation, uint32_t completed, uint32_t total);
esp_err_t esp_cockpit_operation_complete(
    esp_cockpit_operation_handle_t operation, const esp_cockpit_value_t *result);
esp_err_t esp_cockpit_operation_fail(
    esp_cockpit_operation_handle_t operation,
    const esp_cockpit_callback_error_t *error);
```

`esp_cockpit_parameter_descriptor_t` scalar constraints apply only to the matching type; enum values are stable strings, `double` must be finite, strings are UTF-8 byte bounded, and `NULL` is allowed only when `nullable` is true. A `STRING` value always has a non-NULL data pointer, including the empty string (length zero). An attribute's `type` is `STRING` with non-empty `enum_values` for Schema type `enum`; otherwise it maps directly to `boolean`, `integer`, `number`, or `string`. `enum_labels` is either `NULL, 0` or non-NULL with exactly `enum_value_count` entries. `esp_cockpit_attribute_descriptor_t` aliases this type; its `access` is exactly `ESP_COCKPIT_ACCESS_READ`, `ESP_COCKPIT_ACCESS_WRITE`, or their bitwise OR. For command/method arguments and result descriptors, `access` must be zero; arguments carry their required flag, and attributes/results set `required` true. `has_default` controls whether `default_value` is used as a UI suggestion; it never becomes an implicit write or argument. Non-string descriptors set `max_bytes` to zero; `has_minimum`/`has_maximum` indicate inclusive bounds and apply only to the matching numeric type. A result type of `NULL` means no business return value; otherwise it maps to the described Schema type. Each Cluster `id` is the complete `endpoint/cluster` identifier. `write` is required iff a Cluster has writable attributes; `read` is required iff it has readable or version-only-readable attributes. Each command/method requires an `invoke` callback and a result descriptor whose `id` is `"result"` and `has_default` is false. Command and method argument descriptors are named fields; callback arguments use the same names and descriptor order; duplicate names are invalid. An app with no application-defined errors sets `app_error_first` and `app_error_last` to zero; otherwise both are positive and inclusive, with the first not greater than the last.

All descriptor lists use pointer plus count: empty is exactly `NULL, 0`; non-empty requires a non-NULL pointer and positive count. The root namespace, identifier ownership, uniqueness, schema versions and callbacks are validated atomically by `start()`; any invalid entry rejects the whole start. Descriptor arrays, strings, enum arrays and callback `user_ctx` are borrowed, must be immutable and remain valid until `del()` succeeds. `new()` copies configuration data only. Registration cannot be changed after `start()`.

Callbacks are synchronous and serialized on one instance worker in successful-enqueue FIFO order; no two application callbacks execute concurrently. A multi-path request calls the relevant callback per path in request order. Read/Write input and Invoke arguments are owned by the component and borrowed only for the callback duration. Invoke result strings, version tokens and error messages are borrowed from the application and must remain valid until copied immediately after callback return and before the next callback. Version and error-message text use explicit UTF-8 byte lengths; absent message is exactly `NULL, 0`, and present text requires a non-NULL pointer. For Read string output, the component provides `string_buffer` and `string_capacity` in `out`; the callback writes into that buffer, points `value.as.string.data` to the buffer and sets `value.as.string.length`, which must not exceed the capacity. Invoke arguments include every declared parameter in descriptor order; `present: false` represents an omitted optional argument, while explicit JSON `null` is `present: true` with type `ESP_COCKPIT_VALUE_NULL`. Result descriptors use the fixed id `"result"`. A synchronous callback returns `OK` with `out_operation == NULL`; a long-running callback first calls `esp_cockpit_operation_begin()` and returns `ACCEPTED` with that handle in `out_operation`. The application may then update progress or complete/fail the operation from an application task. Operation API calls copy supplied output data before returning; handles are component-owned and valid until instance destruction or terminal-record expiry. `CONFLICT` is valid only for Write and maps to `-32003`; `RESOURCE_LIMIT` maps to `-32005`; `ERROR` may provide a positive code inside the declared app range and optional message, otherwise maps to `-32000`. Unknown ESP-IDF errors are never exposed as protocol codes.

`esp_cockpit_attribute_changed()` takes Endpoint, Cluster and Attribute identifiers and explicit UTF-8 byte lengths, is non-blocking, callable from application tasks after releasing application locks, and only schedules/coalesces a refresh; it does not call Read or perform network I/O inline. It returns `ESP_OK` when accepted/coalesced, `ESP_ERR_INVALID_ARG` for invalid/unregistered paths, `ESP_ERR_INVALID_STATE` when the instance is not running, and `ESP_FAIL` if the refresh mechanism has failed. `operation_begin()` is valid only during an Invoke callback; it returns `ESP_ERR_NO_MEM` if no Operation slot is available. Progress may be set on a queued/running operation, requires `total > 0`, `completed <= total`, and an unchanged total; complete/fail require a queued/running handle. Invalid/stale/terminal handles return `ESP_ERR_INVALID_STATE`; invalid values return `ESP_ERR_INVALID_ARG`. `stop()` waits for already-started callbacks; lifecycle APIs must not be called recursively from a callback. Existing lifecycle return conventions apply: invalid arguments return `ESP_ERR_INVALID_ARG`, invalid transitions return `ESP_ERR_INVALID_STATE`, unsupported build features return `ESP_ERR_NOT_SUPPORTED`, and allocation failure returns `ESP_ERR_NO_MEM`.

## 8. `limits` 字段名及单位

能力发现的 `limits` 必须包含下列字段，均为正整数，表示设备当前配置实际生效的硬上限；`0`、负数、缺项及无穷值均非法。字段语义和单位固定，数值按目标板/生产配置验收后填入。

| 字段 | 单位/范围 |
|---|---|
| `maxHttpConnections`, `maxWebSocketConnections` | 连接数 |
| `maxRequestBytes`, `maxResponseBytes`, `maxQueuedParameterBytes` | 字节 |
| `maxIdBytes`, `maxStringBytes`, `maxErrorMessageBytes`, `maxValueVersionBytes` | UTF-8 字节 |
| `maxSchemaPageItems`, `maxMultiPathItems`, `maxQueuedRequests` | 项/请求数 |
| `maxQueueWaitMs` | 毫秒；排队等待上限，不强制终止已开始的回调 |
| `maxSubscriptionsPerConnection`, `maxSubscriptionsPerInstance`, `maxSubscriptionPaths` | 订阅/路径数 |
| `maxUpdatesPerSecondPerSubscription` | 每订阅每秒更新数；超出时按路径合并为最新值 |
| `maxPendingNotificationBytesPerSubscription`, `maxPendingNotificationBytesPerConnection`, `maxPendingNotificationBytesPerInstance` | 字节 |
| `maxOperations` | 活动及终态 Operation 总数 |
| `operationRetentionMs`, `maxOperationDurationMs` | 毫秒；终态保留期、最长运行期 |
| `maxConcurrentPasswordVerifications`, `maxQueuedPasswordVerifications` | 验证并发数、排队数 |
| `maxLoginAttemptsPerMinute` | 次/分钟；设备端全局速率上限 |
| `loginFailureWindowMs`, `maxLoginDelayMs` | 毫秒；失败计数窗口和渐进延迟上限 |
| `maxClusters`, `maxAttributesPerCluster`, `maxCommandsPerCluster`, `maxApplicationMethods`, `maxCapabilities` | 注册项数 |

组件必须在对应资源边界统一执行限制。能力发现完整响应本身不得超出 `maxResponseBytes`；无法完整返回时使用 `-32005`，不得分页或截断。HTTP 请求体超限按第 5 节返回 `413`；已进入 RPC 分发的资源超限按 `-32005` 返回。

## 9. 冻结状态与剩余验收

最小互操作门槛已冻结：客户端与组件可以据此并行实现，不得把本文件中的正式名称或字段当作示例。后续破坏性变更须提升 JSON-RPC 主版本或 C API 主版本；兼容新增只能增加可选字段/能力，并遵守未知响应字段忽略规则。

尚待实测/细化的包括不同目标配置对应的 `limits` 实际数值、TLS/KDF 参数及性能目标板验收；网络生命周期故障查询/高阶事件的 C 细节继续由[网络生命周期文档](./esp-cockpit-network-lifecycle.md)跟踪，不属于本次 RPC/WebUI 互操作门槛。上述后续工作不得更改已冻结的资源字段名、单位、RPC 结构、稳定错误码或注册/回调 API 所有权语义。
