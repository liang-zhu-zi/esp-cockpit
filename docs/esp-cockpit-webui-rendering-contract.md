# ESP-Cockpit 通用 WebUI 交互与 Schema 渲染补充规范（草案）

> **文档状态**：补充规范（草案）。
> **权威关系**：本文规定通用 WebUI 的启动加载状态机、Schema 控件映射、多路径版本冲突草稿合并、长操作交互、本地化（i18n）及无障碍渲染细则。线协议字段、稳定错误码与 `limits` 约束以[最小互操作契约（冻结基线）](https://www.google.com/search?q=./esp-cockpit-interoperability-baseline.md)为唯一权威来源；客户端协议与隔离规则遵循[JSON-RPC 协议契约](https://www.google.com/search?q=./esp-cockpit-json-rpc-contract.md)。
> 
> 

---

## 1. 启动、能力发现与 Schema 加载状态机

通用 WebUI 必须严格按以下顺序执行初始化与数据加载，严禁在能力发现完成前根据假设或历史缓存渲染业务控制入口：

1. **会话初始化与协议版本校验**：
* 每次完成 `esp_cockpit.auth.login` 或 `esp_cockpit.auth.initialize` 建立新会话后，以及 WebSocket 断线重连后，前端**必须**立即调用 `esp_cockpit.capabilities.get` 重新获取当前实例能力清单，不得回退使用前一会话的缓存。


* 前端必须校验响应中 `protocolVersion`、`methods`、`capabilities`、`clusters` 和 `limits` 五个必填字段的存在性与类型（同时忽略未知新增扩展字段）。若校验失败或请求报错，整次初始化视为失败，界面显示错误详情与手动“重试”按钮，禁止发起任何业务 RPC。


* 若设备返回的 `protocolVersion.major` 与前端支持的协议主版本不兼容，WebUI 必须进入**只读诊断模式**：清晰展示设备与前端的版本/能力差异及升级指引，并全局禁用所有 `Write`、`Invoke` 及状态变更操作。




2. **动态功能入口裁剪**：
* 仅当 `capabilities.get` 返回的 `methods` 或 `capabilities` 中明确包含对应标识时，才渲染相应功能模块入口（如证书管理、安全审计或可选插件）；未编译或未启用的功能入口必须完全隐藏。




3. **按需分页加载与单次自动重试**：
* WebUI 不在首屏一次性拉取所有 Cluster 的 Schema，而是在用户导航或展开目标 Cluster 时，按 `esp_cockpit.schema.list`（`limit <= limits.maxSchemaPageItems`）逐页请求直至 `nextCursor` 为 `null`。


* **完整性门禁**：只有当某个 Cluster 的全部 Schema 分页均加载完毕，或当前会话内存在对应 `schemaVersion` 未变更的完整缓存时，才允许渲染该 Cluster 的交互控件；加载期间显示骨架屏或加载指示器，禁止渲染半成品表单。


* **版本漂移与游标失效重试**：若分页过程中发现响应的 `schemaVersion` 与 `capabilities.get` 或前序页面不一致，或服务端返回 `-32009`（`stale_cursor`），前端必须立即丢弃本轮已拉取的所有页面、重新调用 `esp_cockpit.capabilities.get`，并在该 Cluster 仍存在时从头重新分页（每次用户操作最多自动重试 **1** 次；若重试仍发生版本不一致或游标失效，则停止自动重试并展示错误提示与手动重试按钮）。




4. **Cluster 级错误与未知语义隔离**：
* 若某 Cluster 的 Schema 包含首版未知的 `kind`（除 `attribute`、`command`、`method` 外）、未知 `type`、未知 `access` 值或影响数据语义的未知校验规则，或者分页读取失败，WebUI 必须**仅隔离该受影响的 Cluster**（隐藏/禁用其交互控件并明确提示不兼容或加载失败原因），其他兼容 Cluster 仍保持正常可用。





---

## 2. Schema 类型与 UI 控件映射规范

首版通用 WebUI 仅渲染冻结基线定义的标量类型子集，不支持数组、嵌套对象或动态条件表单。

### 2.1 属性（`attribute`）与参数（`argument`）控件映射表

| Schema `type` | 附加约束条件 | 只读（`access: ["read"]`）渲染 | 可写（含 `"write"`）渲染控件 | 前端输入校验与格式化规则 |
| --- | --- | --- | --- | --- |
| `boolean` | 无 | 状态徽章（如 `On` / `Off` 或 `True` / `False`） | 开关组件（`<input type="checkbox" role="switch">`） | 提交严格 JSON 布尔值 `true` / `false`。 |
| `integer` | 存在 `minimum` 且 `maximum`，且跨度 `<= 1000` | 带 `unit` 后缀的整数文本 | 滑块（`<input type="range">`）与数字输入框（`<input type="number" step="1">`）联动 | 必须为有符号 `int32` 整数（`-2147483648..2147483647`），且满足 `[minimum, maximum]` 闭区间约束。 |
| `integer` | 无完整边界或跨度 `> 1000` | 带 `unit` 后缀的整数文本 | 数字输入框（`<input type="number" step="1">`） | 同上；禁止输入小数点或科学计数法，超范围在提交前拦截提示。 |
| `number` | 可选 `minimum`、`maximum` | 带 `unit` 后缀的数值文本 | 数字输入框（`<input type="number" step="any">`） | 必须为有限浮点数（拒绝 `NaN`、`Infinity`），且满足 `[minimum, maximum]` 闭区间约束。 |
| `enum` | 非空 `enumValues` 与可选等长 `enumLabels` | 对应索引的 `enumLabels[i]`（无标签时显示 `enumValues[i]`） | 下拉选择器（`<select>`，选项数 `<= 5` 时可选用单选组 `radiogroup`） | 选项显示文本使用 `enumLabels[i]`，线协议提交与值比较必须严格使用 `enumValues[i]`。 |
| `string` | `maxBytes`（缺省为 `limits.maxStringBytes`） | 纯文本展示（保留内部空白） | 单行文本输入框（`<input type="text">`；当 `maxBytes > 128` 时渲染 `<textarea>`） | 实时使用 `new TextEncoder().encode(val).length` 计算 **UTF-8 字节数**；超出 `min(maxBytes, limits.maxStringBytes)` 时阻止提交并提示字节超限，**严禁前端自动截断**。 |

### 2.2 特殊属性与语义规则

1. **可空值（`nullable: true`）处理**：
* 当 `nullable` 为 `true` 时，可写控件旁必须提供显式的“设为空值（Set to Null）”切换控件（如复选框或清除按钮）。


* 对于 `Command` / `Method` 的可选参数（`required: false`），前端必须区分“用户未勾选启用该可选参数（请求 `arguments` 对象中省略该键）”与“用户启用该参数并显式设为 `null`（请求 `arguments` 中包含 `"param": null`）”。




2. **默认值（`default`）仅作输入建议**：
* Schema 中的 `default` 字段**绝对不得**在页面加载、表单初始化或缺少当前值时自动触发 `esp_cockpit.write` 写入设备。


* 仅当某属性当前值为 `null`（且 `nullable: true`）或用户点击“填入建议值”按钮时，才将 `default` 填入本地草稿输入框，仍需用户显式点击“保存”后才下发设备。




3. **只写属性（`access: ["write"]`）渲染**：
* 只写属性不向 `esp_cockpit.subscribe` 注册订阅，也无法通过 `esp_cockpit.read` 获取当前值。


* 输入框默认渲染为空白占位符（如“输入新值以更新”），不回显任何旧值；在表单加载或提交前，前端需静默调用 `esp_cockpit.read` 获取该只写属性的当前 `version` 令牌以备并发检查。




4. **延迟持久化状态呈现**：
* 若某 Cluster 包含表达持久化状态的只读属性（如枚举值 `Applied`、`Pending`、`Persisted`、`Failed`），WebUI 必须在对应设置区旁以醒目状态徽章实时展示（`Failed` 状态需配以错误警告样式），提醒用户未落盘前掉电可能丢失更改。





---

## 3. 多路径表单编辑与版本冲突（`-32003`）草稿合并规范

为兼顾操作效率与并发安全性，同一 Cluster 内的可写属性采用“本地草稿追踪 + 批量多路径 `esp_cockpit.write` 提交”交互模型：

### 3.1 前端字段状态模型

WebUI 为每个可写属性维护以下独立状态字段：

```typescript
interface AttributeFieldState {
  path: { clusterId: string; attributeId: string };
  isWriteOnly: boolean;
  deviceValue: JsonScalar | undefined; // 只写属性恒为 undefined
  deviceVersion: string;               // 最近一次 Read 或 subscription.report 获得的 opaque version
  draftValue: JsonScalar | undefined;  // 用户在表单中编辑的本地草稿值
  isDirty: boolean;                    // 用户是否已修改且尚未成功提交
  conflictState: null | {
    remoteValue: JsonScalar | undefined; // 冲突后重新读取到的设备最新值（可读属性）
    remoteVersion: string;               // 设备最新 version
    errorMessage: string;
  };
}

```

### 3.2 提交与部分成功（Partial Success）处理

1. **分批切片**：用户点击“保存更改”时，收集所有 `isDirty === true` 的路径。若待提交路径数超过 `limits.maxMultiPathItems`，前端按请求顺序拆分为多个不超过 `limits.maxMultiPathItems` 的 `esp_cockpit.write` 请求依次发送（单个请求内不得包含重复路径）。


2. **逐项结果回写**：收到响应 `result.items` 后，按顺序逐项处理：


* **`status === "ok"`**：将 `deviceVersion` 更新为响应返回的新 `version`，将 `deviceValue` 更新为刚刚提交的 `draftValue`（非只写属性），置 `isDirty = false` 并清除 `conflictState`。


* **`status === "error"`（非版本冲突）**：保留 `draftValue` 与 `isDirty = true`，在对应控件下方内联展示 `error.message`。


* 界面顶部提示栏必须准确反映部分成功结果（例如：“已保存 3 项，1 项存在版本冲突未保存”），严禁将多路径写入表述为全有或全无。





### 3.3 版本冲突（`-32003` `version_conflict`）的交互与草稿合并流程

当某路径返回 `error.code === -32003` 时，表明另一会话或设备本地操作已修改该属性。前端**严禁**静默拉取新版本并自动覆盖用户修改，必须执行以下冲突解决流程：

1. **保留用户草稿**：完整保留用户输入在 `draftValue` 中的草稿内容，`isDirty` 保持为 `true`。


2. **同步最新设备状态**：
* **对于可读属性（`access` 含 `"read"`）**：若未通过 WebSocket 订阅收到更新，前端立即对该冲突路径发起一次 `esp_cockpit.read`，取得最新 `value` 与 `version` 存入 `conflictState`。控件旁展开**双栏对比面板**：


* 展示“设备当前值（`remoteValue`）”与“你的草稿值（`draftValue`）”；
* 提供两个显式操作按钮：**“采用设备值（放弃草稿）”**（将 `deviceValue`/`deviceVersion` 同步为远程值并清除 `isDirty`）和“保留草稿并准备覆盖”（将 `deviceVersion` 更新为 `remoteVersion`，保持 `draftValue` 与 `isDirty = true`）。




* **对于只写属性（`access: ["write"]`）**：前端明确提示用户：*“该属性为只写安全属性，无法展示设备当前值，但自您打开页面后该值已被其他操作修改。”* 提供两个显式操作按钮：**“取消修改”**（清空草稿）和“获取最新版本并确认覆盖”（调用 `esp_cockpit.read` 取得最新 `version` 更新至 `deviceVersion`，保持 `draftValue` 与 `isDirty = true`）。




3. **仅重提冲突路径**：用户确认冲突处理并再次点击“保存更改”时，前端**仅**将仍处于 `isDirty === true` 的路径组装进新的 `esp_cockpit.write` 请求，绝不重复提交前一次已返回 `"ok"` 的路径。


4. **后台订阅到达时的草稿保护**：若在用户编辑某字段（`isDirty === true`）期间，WebSocket 收到该路径的 `esp_cockpit.subscription.report`，前端**不得**用推送的新值直接覆盖输入框中的 `draftValue`，而应在控件旁显示非阻塞提示（“设备值已在后台更新为 X”），并允许用户选择刷新或继续提交（若直接提交则会触发 `-32003` 进入上述标准冲突流程）。

---

## 4. 实时订阅、长操作（Operation）与异常降级交互

1. **订阅更新与序号合并**：
* 仅对用户当前正在查看的 Cluster 中具备 `"read"` 权限的属性建立 WebSocket `esp_cockpit.subscribe`；切换离开页面时主动调用 `esp_cockpit.unsubscribe` 释放设备订阅槽位。


* 收到 `esp_cockpit.subscription.report` 时，按路径将 `sequence` 转为大整数（`BigInt`）与本地记录的最后序号比较：仅当 `newSeq > lastSeq` 时更新 `deviceValue`、`deviceVersion` 与 `lastSeq`；忽略重复或旧序号；发生跳号时直接应用新值，不触发额外 `esp_cockpit.read`。




2. **订阅中断与服务故障降级**：
* 当收到 `esp_cockpit.subscription.terminated`（`reason` 为 `resource_limit` 或 `read_failed`）或订阅返回 `-32010`（`subscription_unavailable`）时，WebUI 必须立即移除页面上的“实时同步中”绿色指示灯，切换为“实时更新不可用（当前展示为静态快照）”警告横幅，严禁将过期数据伪装为实时状态，且严禁在后台死循环自动重连订阅（提供手动“重新订阅”按钮）。




3. **命令执行（`Invoke`）与长操作（`Operation`）轮询**：
* 执行 `Command` 或应用 `Method` 前，若接口可能产生副作用，展示参数输入弹窗；若接口返回 `{status: "accepted", operationId}`，前端将 `operationId` 纳入非阻塞任务状态栏，以 **1000 ms** 固定间隔轮询 `esp_cockpit.operation.get`（该轮询属于后台状态查询，不重置会话空闲超时）。


* 任务处于 `queued` 时显示“排队等待执行”，转为 `running` 且 `progress != null` 时渲染确定性进度条（`completed / total`），`progress === null` 时渲染不确定性旋转指示器。


* 首版不渲染通用的“取消操作”按钮。若轮询返回 `-32004`（`not_found`，通常因设备重启导致 RAM 中的 Operation 丢失），前端提示“任务记录已随设备重启清除，请刷新设备当前状态”并引导重新读取属性快照。


* 当 `Write` 或 `Invoke` 因网络超时或断连导致结果未知时，前端保留用户草稿并提示刷新状态；**仅当** Schema 明确声明 `retrySafe === true` 时才渲染“重试”按钮，否则禁止提供一键重放。





---

## 5. 本地化（i18n）与文本安全渲染规范

1. **离线语言包与回退链**：
* 所有 UI 框架文案与语言字典必须随单页 gzip 前端资源包离线打包，严禁请求外部翻译 API 或在线语言文件。


* 英文（`en`）为强制内置基线语言。页面初始化时按浏览器 `navigator.languages` 顺序匹配固件实际打包的语言列表（支持 `zh-CN` -> `zh` 前缀降级匹配），无匹配项时无条件回退至 `en`，同时在页脚提供手动语言切换器。


* 产品发布说明须明确列出固件实际打包的语言集合。




2. **Schema 显示元数据（`label` / `unit` / `enumLabels`）解析优先级**：
* 针对从设备动态读取的 Schema 条目，前端按以下优先级解析最终展示文本：
1. **结构化字典键查找**：在当前激活的本地化字典中查找标准命名键（Cluster 标题：`schema.<clusterId>.title`；属性/命令标签：`schema.<clusterId>.<id>.label`；枚举标签：`schema.<clusterId>.<id>.enum.<enumValue>`）；
2. **直接键查找**：若 C 描述符中的 `label` 或 `enumLabels[i]` 字符串本身命中本地化字典键（如 `"i18n.switch.on_off"`），则使用对应翻译文本；
3. **Schema 原文回退**：若字典中无对应条目，则直接使用 Schema 返回的 `label` / `enumLabels[i]` 字符串；


4. **标识符兜底**：若 `label` 未提供，则展示 `id` 或 `enumValues[i]` 原文。




* 任何情况下，显示文本仅用于界面渲染，参与 RPC 调用、排序或条件判断时必须使用原始 `id` 与 `enumValues`。




3. **防注入纯文本渲染红线**：
* 来自设备 Schema 的 `label`、`unit`、`enumLabels` 以及错误响应的 `message` 属于不可信数据，前端**必须**全部按纯文本节点渲染（使用框架默认文本插值或 DOM `textContent`），**严禁**使用 `v-html`、`dangerouslySetInnerHTML` 或 `innerHTML` 注入 HTML 片段。





---

## 6. 安全存储、CSP 与无障碍（WCAG 2.2 AA）验收规范

1. **前端安全存储与 CSP 策略**：
* 会话认证完全依赖服务端下发的 `HttpOnly; Secure; SameSite=Strict` Cookie；登录/初始化返回的 `csrfToken` **仅允许保存在 JavaScript 运行时内存变量中**，严禁写入 `localStorage`、`sessionStorage`、非 HttpOnly Cookie 或 URL 查询参数（页面整页刷新后，前端可通过调用 `esp_cockpit.auth.status` 或重新认证机制恢复界面状态）。


* WebUI 代码构建必须兼容严格的 `Content-Security-Policy`（如 `default-src 'self'; script-src 'self'; style-src 'self'; connect-src 'self' wss:; object-src 'none'; base-uri 'none'`），禁止使用 `eval()`、`new Function()`、内联 `<script>` 标签及任何第三方 CDN 资源。


* 登录与修改密码表单必须正确标注 `autocomplete="current-password"` 与 `autocomplete="new-password"`，严禁禁用输入框的复制、粘贴或密码管理器自动填充功能。




2. **WCAG 2.2 AA 可访问性实现细则**：
* 所有动态生成的 Schema 控件必须与可见 `<label for="...">` 显式绑定；校验错误提示与帮助文本必须通过 `aria-describedby` 关联至对应输入控件，校验失败时动态设置 `aria-invalid="true"`。


* 所有按钮、开关、自定义下拉框及模态对话框必须支持纯键盘操作（`Tab` / `Shift+Tab` / `Enter` / `Space` / `Esc`），且保持清晰可见的焦点轮廓（`:focus-visible` 对比度 `>= 3:1`），正文文本颜色对比度满足 `>= 4.5:1`。


* 实时的表单保存结果、版本冲突警报与 Operation 状态变化必须挂载至 `aria-live="polite"` 或 `role="alert"` 区域，确保主流屏幕阅读器（NVDA / VoiceOver / TalkBack）无需手动刷新即可播报状态变更。





---

## 7. 自动化与手动验收检查表

| 验收维度 | 验收项与执行方式 | 通过门槛 |
| --- | --- | --- |
| **协议兼容性** | 模拟高主版本（`major: 2`）及高次版本（`major: 1, minor: 5`）设备响应 | 高主版本自动进入只读诊断模式且禁用写入；高次版本正常忽略未知新增字段并运行。 |
| **Schema 隔离** | 注入含未知 `kind: "event_stream"` 的 Cluster 与正常 Cluster 并存 | 仅异常 Cluster 显示不兼容隔离提示，正常 Cluster 读写不受影响。 |
| **字节边界校验** | 在 `maxBytes: 10` 的字符串控件中输入 4 个中文汉字（12 字节 UTF-8） | 前端拦截提交并提示超出 10 字节上限，不发生截断或发送无效 RPC。 |
| **冲突草稿合并** | 多路径修改 3 项属性，模拟第 2 项返回 `-32003`、第 1/3 项返回 `ok` | 第 1/3 项转为已保存；第 2 项保留用户草稿并展示远程最新值，确认覆盖后仅重提第 2 项。 |
| **安全与存储审计** | 浏览器 DevTools 检查 Storage、网络请求与 XSS 载荷（`label: "<img src=x onerror=alert(1)>"`） | `localStorage` 与 `sessionStorage` 为空；XSS 载荷以纯文本原样显示且不触发 CSP 违规。 |
| **WCAG 2.2 AA** | 全键盘遍历表单与冲突面板 + 屏幕阅读器（VoiceOver / NVDA）实测 | 无键盘焦点陷阱，所有控件标签、字节超限错误与冲突提示均可被准确朗读。 |
