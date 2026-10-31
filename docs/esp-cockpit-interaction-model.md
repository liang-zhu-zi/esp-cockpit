# ESP-Cockpit 交互模型与 JSON-RPC 映射

> 状态：设计讨论稿。本文评估借鉴 Matter Interaction Model（IM）语义，不声明 ESP-Cockpit 实现 Matter 协议或具备 Matter 互操作性。

## 1. 结论与建议

**ESP-Cockpit 可以采用 Matter Interaction Model 的四类交互语义作为应用层操作模型，并通过 JSON-RPC 2.0 over HTTP/WebSocket 承载；订阅报告使用 WebSocket 通知。不应把它描述为“JSON-RPC 采用 Matter 协议”，也不应把 Matter IM 直接等同于 JSON-RPC。**

建议将交互抽象为：

- **Read**：读取一个或多个状态/属性。
- **Write**：设置一个或多个状态/属性。
- **Invoke**：执行一个有明确输入和结果的操作/命令。
- **Subscribe**：建立有生命周期的状态订阅，并接收后续报告。

JSON-RPC 负责方法调用、请求/响应关联和通知封装；Endpoint、Cluster、Attribute、Command 等路径则作为应用数据模型中的目标标识。是否采用 Matter 风格的命名、类型系统及状态码，仍需单独决定。

这是一种**受 Matter 启发的应用交互模型**。除非实现并验证 Matter 所要求的完整数据模型、交互语义、安全、发现、传输和一致性要求，否则不能声称“兼容 Matter”或“Matter 网关”。

本项目当前将传输、消息协议和安全层分开决策：WebSocket 作为双向实时传输，JSON-RPC 2.0 作为消息协议，TLS/WSS 的安全要求由威胁模型另行确定。当前不以 CBOR-RPC 替代 JSON-RPC 2.0，也不采用自定义应用层加密协议。CBOR 编码的 RPC 需要独立定义消息与互操作语义；只有目标板实测证明 JSON 是关键资源瓶颈后，才重新评估二进制协议。应用层加密若作为额外端到端需求提出，必须先解决密钥建立与认证、重放/降级防护、密钥轮换及浏览器前端可信交付，并经过独立安全评审，不能被视为 TLS/WSS 的直接替代品。

## 2. 分层与寻址建议

建议把目标路径作为参数，而不是将每一种属性/命令都编码进 JSON-RPC `method` 名称：

```json
{
  "jsonrpc": "2.0",
  "id": 31,
  "method": "im.read",
  "params": {
    "paths": [
      {
        "endpoint": 1,
        "cluster": "OnOff",
        "attribute": "onOff"
      }
    ]
  }
}
```

建议的数据模型概念：

- **Endpoint**：设备中可独立寻址的功能实例；系统根端点及其编号规则应在 ESP-Cockpit 自己的数据模型中定义。
- **Cluster**：一组相关属性、命令和事件的能力集合；可使用稳定名称、数字 ID 或带版本的注册标识。
- **Attribute**：可读取/订阅、在授权和策略允许时可写的状态值。
- **Command**：通过 `Invoke` 执行的操作，具有独立参数和结果类型。
- **Path**：完整目标（例如 endpoint + cluster + attribute/command）。需定义通配符、批处理和访问控制规则。

不要因 Matter Endpoint 0 的语义或某个既有产品的通道编号，普遍执行 `endpoint = id + 1`。Endpoint 是数据模型标识，不是可安全推导的数组下标；任何旧接口兼容映射都必须是针对明确旧协议的显式适配器，并逐项验证身份、范围和语义。

本组件目前没有已确认的 Shelly 兼容需求。`Switch.Set`、`Cover.Calibrate` 等旧方法别名不应默认加入核心设计。若未来确有兼容目标，可在独立兼容层定义静态映射、冲突策略、版本和测试，不应让核心 IM 模型依赖某厂商旧 API。

## 3. 四类交互与 JSON-RPC 映射

具体 JSON-RPC 方法名和参数尚待接口设计确定。Read、Write、Invoke 可根据交互需要通过 HTTP 或 WebSocket 承载；Subscribe 使用 WebSocket 建立及推送。下表中的 `im.*` 仅为示意名称。

| 交互 | JSON-RPC 请求/响应建议 | 需定义的语义 |
|---|---|---|
| Read | 带 `id` 的 `im.read` 请求（HTTP 或 WebSocket），响应在 `result` 中返回每个 Path 的值或逐项状态 | 单项/批量读取、读取权限、未知路径、快照一致性、最大结果尺寸 |
| Write | 带 `id` 的 `im.write` 请求（HTTP 或 WebSocket），响应在 `result` 中返回写入结果 | 逐项授权和范围校验、是否原子、部分成功、应用回调失败和持久化状态 |
| Invoke | 带 `id` 的 `im.invoke` 请求（HTTP 或 WebSocket），响应在 `result` 中返回命令结果 | 命令参数、幂等性、超时、重复请求、防重放及长时间操作的任务句柄 |
| Subscribe | 带 `id` 的订阅请求，立即返回订阅句柄/初始状态；后续报告以无 `id` JSON-RPC Notification 推送 | 订阅范围、报告格式、取消、过期、重连、序列/版本、背压与权限变化 |

### 3.1 Subscribe 生命周期

JSON-RPC 2.0 的请求/响应关联是一问一答；这与长生命周期订阅并不冲突。订阅应拆分为：

1. 客户端以带 `id` 的 JSON-RPC 请求创建订阅。
2. 设备返回一次普通 JSON-RPC 响应，包含订阅 ID、当前初始值或快照版本，以及协商后的订阅范围/参数。
3. 后续状态报告由设备通过不带 `id` 的 JSON-RPC Notification 发送；通知可包含订阅 ID、路径、值、版本/序列号和时间信息。
4. 客户端显式取消订阅，或订阅在连接关闭、会话过期、权限撤销、租期到期时终止。

HTTP JSON-RPC 请求可以执行一次性读写/调用，也可以返回初始快照，但不能仅凭普通 HTTP 请求建立设备向该请求持续推送的通道。若产品启用 SSE，可另行定义只读订阅传输；默认实时通知路径仍使用 WebSocket。

客户端应能在断线后重新认证、重新订阅并读取快照。不要默认假设订阅跨 WebSocket 断线仍有效，也不要仅靠通知保证状态不丢失。必须定义初始快照与并发变化之间如何避免遗漏（例如基于版本号/序列号的快照后增量规则）。

订阅数、每订阅路径数、报告频率、每客户端待发送字节数和空闲租期必须受限。慢客户端应触发合并/丢弃非关键报告或断开连接，而不能阻塞状态生产者和设备控制任务。

**不要把任意 `callback_url` 作为通用 Subscribe 参数。**设备向用户提供的 URL 主动发起 HTTP 请求会引入 SSRF、凭据泄露、网络访问控制、TLS 信任、重试和资源耗尽等额外风险。若产品确有 Webhook 需求，应作为独立功能设计注册/验证、允许目标范围、认证、TLS、限速、重试和禁用策略；不应仅因存在 Subscribe 就自动推导出 Webhook。

### 3.2 路径和方法命名

采用通用 `im.read` / `im.write` / `im.invoke` / `im.subscribe` 方法有利于统一分发及注册扩展，但会使方法本身不再表达具体业务能力。需配套能力发现、Schema、Cluster/Command 注册、访问控制和版本管理。

也可保留业务语义清晰的 RPC 方法（如 `wifi.configure`、`device.reboot`），同时让状态读取/订阅采用统一的数据模型。两种风格可以共存，但应明确边界，避免同一操作既能通过通用 IM 路径又能通过业务 RPC 执行而出现不一致的权限、校验或持久化语义。

不建议默认实现“任何非 `im.*` 方法都查别名表并转换”的双模解析。JSON-RPC 已经有方法路由语义；开放式别名可能导致方法冲突、隐式访问控制差异、兼容负担和调试困难。只有为明确的外部兼容协议提供适配时，才考虑显式、可禁用的别名映射。

## 4. 结果、错误与批处理

JSON-RPC 响应必须在 `result` 和 `error` 之间二选一。建议区分：

- **JSON-RPC 层错误**：无效 JSON-RPC 封装、未知 RPC 方法、无效参数结构、认证/授权拒绝或无法执行整个请求时，使用 JSON-RPC `error`。
- **交互层逐项结果**：请求结构合法且已按协议处理的多路径操作，可在 `result` 中返回各 Path 的成功/失败状态。部分失败并不要求把整个 JSON-RPC 请求转换为 `error`。

例如，批量 Write 可以返回：

```json
{
  "jsonrpc": "2.0",
  "id": 32,
  "result": {
    "results": [
      {
        "path": {
          "endpoint": 1,
          "cluster": "OnOff",
          "attribute": "onOff"
        },
        "status": "success"
      },
      {
        "path": {
          "endpoint": 1,
          "cluster": "LevelControl",
          "attribute": "currentLevel"
        },
        "status": "constraint_error"
      }
    ]
  }
}
```

但不能笼统假设“任何单项业务失败都必须成为 JSON-RPC error”。单路径拒绝也可以采用成功的 RPC 调用并在 `result` 中返回交互状态；选择何种规则应保持一致，并明确它表示“RPC 调用成功处理”而不是“所有目标操作均成功”。

如果提供批量写入，还须明确：

- 批量操作是否原子；默认不应让客户端误以为有全有或全无保证。
- 校验失败是否在任何写入前整体拒绝，还是允许逐项处理。
- 应用回调失败时，如何报告已执行与未执行项。
- 批处理大小、响应大小和执行超时上限。

## 5. 属性写入与命令调用边界

`Write` 与 `Invoke` 应按**目标数据模型的语义**区分，而不应通过任意字段存在与否隐式改写操作类型：

- 写入期望的持久状态（例如设定开关目标值）通常建模为属性写入。
- 执行一次性动作、触发带参流程或启动定时行为通常建模为命令调用。
- 若设备 API 把属性设置和瞬态参数放在同一请求（例如设置开关同时要求定时关闭），应定义明确的业务命令/组合操作，或定义经过 Schema 描述的显式映射；不得静默地把 `Write` 改成另一个命令。
- 具体是否映射到 Matter On/Off Cluster 的标准命令（例如某个 timed-on 命令），需核对 Matter 版本、方向、约束和具体命令语义。JSON-RPC 参数名 `toggle_after` 本身不足以证明等价映射。

这可避免“相同请求因可选字段而突然变成不同权限、不同副作用或不同幂等性”的隐式行为。

## 6. Matter 兼容性及编码边界

- 四类交互借鉴 Matter IM 不能单独带来 Matter 互操作性；Matter 还定义/依赖数据模型、访问控制、安全会话、错误状态、订阅行为、发现、传输与一致性要求。
- Matter 原生消息编码使用 **Matter TLV**，不能将 Matter 编码概括为 CBOR。ESP-Cockpit 当前选择的 JSON-RPC over WebSocket 是独立的 JSON 表达和传输设计。
- 若未来需要对接 Matter Fabric/设备，建议通过独立 Matter 适配层把 Matter 操作映射到应用能力，不要把 Matter 协议栈和安全模型塞入通用 JSON-RPC 核心。
- Cluster/Attribute/Command 数字 ID、名称、数据类型和状态码是否复用 Matter 定义，须逐项确认许可、版本和语义；不得只借用名字或 ID 就宣称兼容。

## 7. 推荐决策

当前推荐采用**Matter-inspired、非 Matter 兼容**的数据交互抽象：

1. 保持 JSON-RPC 2.0 作为消息协议，Read/Write/Invoke 可通过 HTTP 或 WebSocket 承载；Subscribe 默认使用 WebSocket。
2. 评估统一的 Read / Write / Invoke / Subscribe 操作，目标路径通过 params 表示。
3. 订阅使用一次响应返回 handle，后续以 JSON-RPC Notification 报告；提供取消、限额、版本/序列、重连快照和背压。
4. 业务控制仍允许注册具名 RPC；核心设计不做通用 legacy alias 自动转换。
5. 明确逐路径状态码与 JSON-RPC `error` 的分工，并定义批量写入的原子性。
6. 不自动支持 Webhook callback URL；如有需求另行做安全设计。
7. 核心数据模型先使用组件自己的 Endpoint/Cluster/Attribute/Command 注册表；是否复用 Matter 标识及是否开发 Matter 互通适配器作为后续决策。

## 8. 待决事项

### 交互模型与 RPC 边界

- 是否以四种交互作为默认状态 API，还是作为可选通用模块，同时保留具名业务 RPC。
- 具体 JSON-RPC 方法名、参数结构，以及 Read/Write/Invoke/Subscribe 在 HTTP 与 WebSocket 上的覆盖范围。
- 具名业务 RPC 与通用交互路径之间的职责边界，避免重复操作导致权限、校验或持久化语义不一致。

### 数据模型与访问控制

- Endpoint、Cluster、Attribute、Command 的标识格式、注册生命周期、版本和访问控制。
- `Read`/`Write` 的批量上限、原子性以及逐项结果状态码。

### 订阅与长时操作语义

- `Subscribe` 的租期、取消、初始快照/增量一致性、通知序列和断线恢复。
- 长时间 `Invoke` 的超时、幂等性及任务句柄。

### Matter 兼容与外部回调

- Matter 标识复用、Matter 适配器和真正 Matter 互操作是否属于项目范围。
- Webhook 是否为产品需求；如是，独立定义安全和运维策略。
