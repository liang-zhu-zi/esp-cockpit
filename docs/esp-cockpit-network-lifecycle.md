# ESP-Cockpit 网络生命周期设计补充

> 状态：架构讨论稿。本文描述推荐的职责划分、状态模型和事件流转，不是已实现行为或最终公共 API。具体 ESP-IDF v6.1 API、目标芯片差异和资源预算须在实现及目标板验证阶段确认。

## 1. 设计目标

网络生命周期设计应让 ESP-Cockpit 可用于以下不同项目，而不接管应用不愿交出的资源：

- 启用 ESP-Cockpit 的 Wi-Fi 配网插件时，明确进入托管 Wi-Fi 模式：插件管理 STA SSID/Password、Wi-Fi 接口及连接生命周期，并按需提供 SoftAP 配网门户。
- 应用或其他组件已经管理 Wi-Fi 或 Ethernet 时，采用外部网络模式，不启用 ESP-Cockpit 内置的 Wi-Fi 配网和网络管理插件；ESP-Cockpit 只使用现有网络运行 Web Portal 和 JSON-RPC 服务。
- ESP-Cockpit 生命周期控制器可观察托管 Wi-Fi 或外部网络的就绪状态，并在满足服务条件时启动 Web 服务；观察事件不等于拥有或控制相应网络资源。
- 以太网接口和参数由应用或外部网络管理器负责，不作为内置 Wi-Fi Manager 的子功能。

核心原则是**单一资源所有者、显式生命周期、异步状态机和有界事件传递**。组件不得因提供 Web Portal 而假定自己独占系统网络或全局 ESP-IDF 基础设施。

Wi-Fi 配网和网络配置是可选插件能力。每个 ESP-Cockpit 实例必须且只能选择托管 Wi-Fi 或外部网络模式之一。Kconfig 决定相关实现是否编译进固件，不替实例选择模式；选择外部网络的实例不得启用或调用组件内置 Wi-Fi 配网和网络管理能力。配置插件只可修改其拥有或获授权管理的接口。

## 2. 职责边界与资源所有权

| 能力/资源 | 应用或平台集成责任 | ESP-Cockpit 责任 |
|---|---|---|
| NVS Flash、默认事件循环、`esp_netif` 全局初始化 | 确定初始化顺序及全局资源所有者；通常由应用启动代码或平台引导层初始化 | 声明所需前置条件，不盲目重复初始化；若提供托管模式，必须明确依赖、错误处理和所有权契约 |
| GPIO 按键、LED、屏幕及板级动作 | 识别按键长按、驱动指示器，并调用组件公开的抽象操作 | 提供不依赖 GPIO 的门户启停控制和高阶状态事件 |
| Wi-Fi 驱动、Wi-Fi netif | 托管模式下满足全局初始化前置条件；外部模式下由应用/网络组件初始化和管理 | 启用 Wi-Fi 配网插件即托管 Wi-Fi；组件创建/管理约定范围内的驱动、接口及连接状态机，不接管外部模式资源 |
| 托管 Wi-Fi 实例配额 | 遵循每进程单实例约束 | 原子保留并强制执行进程级配额；重复创建返回 `ESP_ERR_INVALID_STATE`，销毁原实例后才释放配额 |
| Wi-Fi 凭据配置 | 托管模式下提供产品初始 AP 凭据及密码策略；外部模式下由应用/网络管理器管理凭据 | 托管模式下的配网插件负责 STA SSID/Password 配置、验证和提交 |
| DHCP/静态 IP 配置 | 外部模式下由应用/网络组件配置其拥有的 Wi-Fi 或 Ethernet 接口 | 插件只配置其拥有或被明确授权管理的接口；不修改外部接口参数 |
| 原生以太网 MAC 与 RMII/MII PHY | 应用/网络组件负责板级接线、PHY 型号配置及相应 ESP-IDF 集成 | 支持清单内带原生 Ethernet MAC 的目标；通过适配器读取状态或注册网络服务，不默认调用 Wi-Fi 专属 API |
| 外部以太网控制器（例如 SPI Ethernet） | 应用可自行管理其驱动和板级集成 | 当前不属于 ESP-Cockpit 支持范围；若应用自行接入，需由项目另行验证 |
| 外部 Wi-Fi 协处理器 | 由应用负责驱动和板级集成 | 不属于原生 Wi-Fi 支持清单，需单独评估 |
| HTTP(S) 监听器、WebSocket、JSON-RPC | 提供监听地址、端口、证书等产品配置，并避免与应用自有服务冲突；外部模式下按约定提供网络就绪状态 | 管理唯一活动的 ESP-Cockpit Web 服务、连接、认证、RPC 分发和静态资源；拒绝本组件内并发监听冲突，且不得因此启动、停止或重配外部网络接口 |
| 默认事件循环及事件处理器 | 应用拥有并按契约创建/销毁共享默认事件循环；集成时避免重复注册冲突的应用级处理器 | 仅注册、注销本组件拥有的 handler；停止或销毁时不得删除共享事件循环或其他模块的 handler |
| Captive Portal DNS 与受限 HTTP 门户 | 提供 SoftAP 场景所需配置及安全策略 | 仅在托管配网门户启用期间启停对应 DNS/HTTP 服务，并实施范围限制 |
| 业务参数、传感器、执行器及具体协议 | 注册状态、参数和回调，执行实际业务操作 | 提供注册、校验、分发、状态通知等通用机制 |
| 参数持久化 | 应用或显式启用的持久化模块决定哪些参数持久化及其语义 | 不应默认对每次参数写入同步执行 NVS 写入；网络凭据和业务参数的存储职责应分别说明 |

### 2.1 全局初始化约定

`nvs_flash_init()`、`esp_netif_init()`、默认 `esp_event` loop 等全局设施具有跨组件的初始化顺序和共享所有权要求。推荐由应用启动层负责创建，并在启动 ESP-Cockpit 前满足其依赖。

这不等于禁止所有组件参与初始化：如果产品选择由托管 Wi-Fi 模块创建某一资源，应通过明确模式、前置状态检查和所有权规则执行，不能无条件重复创建全局资源，也不能在 `stop()` 时销毁应用或其他组件仍在使用的资源。默认事件循环已存在、网络接口已由外部创建等情况都应有定义明确的接入方式。

### 2.2 进程级实例与共享资源约束

- 首版每个进程最多允许创建一个托管 Wi-Fi 模式实例。创建时保留该进程级配额；重复创建应返回明确的状态错误，只有原实例销毁后才释放配额。`stop()` 不释放配额，因为实例仍然存在。
- HTTP/WebSocket 服务共享监听资源。首版最多允许一个 ESP-Cockpit 实例处于活动服务状态；其他实例达到服务就绪条件时不得覆盖或停止已有服务，必须报告明确的资源冲突错误。停止活动实例后，其他已创建实例可重新尝试启动服务。
- 对应用或其他组件创建的 HTTP 服务，ESP-Cockpit 无法仅凭自身实例表保证端口不冲突。监听地址/端口由应用配置并负责与外部服务协调；启动时应传播 HTTP Server 的绑定错误，不得伪报成功。
- 实例配额及活动服务资源必须通过进程级同步机制原子保留，避免并发 `new()`/`start()` 绕过单实例限制；创建或启动失败时回滚本次预留。
- 默认 `esp_event` loop 等全局设施仍由应用持有。组件 handler 必须按实例独立记录其事件 base、event id 和注册上下文并仅注销自身；不得使用会移除其他模块 handler 的宽泛注销方式，也不得重复创建或销毁全局资源。

### 2.3 网络管理模式

- **托管 Wi-Fi 模式**：实例创建时显式选择。ESP-Cockpit 管理 Wi-Fi 驱动（按全局初始化契约）、STA/AP netif、SSID/Password 配置、连接和重连状态机；网络配置插件仅操作该模式下由组件拥有的 Wi-Fi 接口。选择此模式但固件未编译 Wi-Fi 配网能力时，创建实例应返回 `ESP_ERR_NOT_SUPPORTED`。
- **外部网络模式**：实例创建时显式选择，适用于应用/其他网络管理组件已管理 Wi-Fi、Ethernet 或其他网络的场景。ESP-Cockpit 不创建、切换、停止或重配外部网络资源；该实例不得启用或调用组件内置 Wi-Fi 配网和网络管理能力。
- 两种模式在同一实例中严格互斥，创建时确定，运行期间不可切换。需要更换模式时，应停止并销毁原实例、完成资源交接，再以另一模式创建新实例。
- 组件可以订阅 ESP-IDF 网络事件或接收应用提供的网络就绪通知来协调 Web 服务启停；事件监听不代表资源所有权。外部网络管理器无需把每个底层事件逐一转发给组件，只需按约定提供“可服务/不可服务”状态或使用标准事件适配器。
- 以太网及混合 Wi-Fi/Ethernet 路由管理由应用/外部网络管理器负责；本设计不要求 ESP-Cockpit 内置网络管理器同时管理 Ethernet。

### 2.4 Kconfig 与编译能力

- Kconfig 只控制能力是否编译进固件，不隐式选择某个实例的网络模式。网络模式由 `esp_cockpit_new()` 的配置显式选择，且每个实例必须且只能选择一个模式。
- `ESP_COCKPIT_WIFI_PROVISIONING` 可控制内置 Wi-Fi 配网实现是否编译。启用该选项表示固件具备托管 Wi-Fi 能力，不代表所有实例都采用托管模式；外部网络实例不使用该能力。
- 同一固件可以同时编译 Wi-Fi 配网能力并由实例选择托管或外部网络模式；外部模式不使用配网插件，不要求固件必须裁掉该能力。确定只采用外部网络的产品可关闭配网能力以减小固件。
- 构建时应通过 Kconfig 依赖/冲突约束表达真实组件依赖，并在运行时由 `new()` 校验所选模式与已编译能力是否匹配。未编译的可选能力不得静默降级；相关 API 返回 `ESP_ERR_NOT_SUPPORTED`。
- TLS、NVS 持久化、SSE 等可选实现可分别由 Kconfig 控制；核心 HTTP/WebSocket/JSON-RPC 能力不应仅为开关数量而拆分选项。精确选项名和依赖关系在组件依赖及实现设计阶段确定。
- Kconfig 默认值不得包含产品凭据、Token、证书私钥等秘密。

网络模式与 Wi-Fi 配网编译能力的组合行为如下：

| 实例模式 | Wi-Fi 配网能力已编译 | `new()` 结果 | 运行行为 |
|---|---:|---|---|
| `MANAGED_WIFI` | 是 | 成功（资源配额等其他校验通过时） | 组件拥有并管理该实例的 Wi-Fi 配网生命周期 |
| `MANAGED_WIFI` | 否 | `ESP_ERR_NOT_SUPPORTED` | 不创建实例，不启动网络资源 |
| `EXTERNAL` | 是 | 成功 | 使用外部网络；不启动或调用内置 Wi-Fi 配网能力 |
| `EXTERNAL` | 否 | 成功 | 使用外部网络；行为与已编译时相同 |

该表只表示网络模式与 Wi-Fi 配网能力的关系；单实例配额、参数有效性及其他编译选项仍须分别校验。

## 3. 公共 API 与生命周期契约

公共 API 采用“应用启动一个生命周期控制器，由组件协调网络状态与 Web 服务”的模型。以下函数名和结构仅用于说明契约，不构成最终 ABI：

```c
typedef enum {
    ESP_COCKPIT_NETWORK_MODE_UNSPECIFIED = 0,
    ESP_COCKPIT_NETWORK_MODE_MANAGED_WIFI,
    ESP_COCKPIT_NETWORK_MODE_EXTERNAL,
} esp_cockpit_network_mode_t;

typedef enum {
    ESP_COCKPIT_STATE_CREATED,
    ESP_COCKPIT_STATE_WAITING_FOR_NETWORK,
    ESP_COCKPIT_STATE_PROVISIONING,
    ESP_COCKPIT_STATE_STARTING_WEB,
    ESP_COCKPIT_STATE_READY,
    ESP_COCKPIT_STATE_STOPPING,
    ESP_COCKPIT_STATE_STOPPED,
    ESP_COCKPIT_STATE_ERROR,
} esp_cockpit_state_t;

typedef void (*esp_cockpit_event_handler_t)(
    esp_cockpit_handle_t handle,
    const esp_cockpit_event_t *event,
    void *user_ctx);

esp_err_t esp_cockpit_new(
    const esp_cockpit_config_t *config,
    esp_cockpit_handle_t *out_handle);
esp_err_t esp_cockpit_register_param(
    esp_cockpit_handle_t handle,
    const esp_cockpit_param_t *param);
esp_err_t esp_cockpit_register_rpc_method(
    esp_cockpit_handle_t handle,
    const esp_cockpit_rpc_method_t *method);
esp_err_t esp_cockpit_register_event_handler(
    esp_cockpit_handle_t handle,
    esp_cockpit_event_handler_t handler,
    void *user_ctx);
esp_err_t esp_cockpit_start(esp_cockpit_handle_t handle);
esp_err_t esp_cockpit_stop(esp_cockpit_handle_t handle);
esp_err_t esp_cockpit_set_external_network_ready(
    esp_cockpit_handle_t handle,
    bool ready);
esp_err_t esp_cockpit_get_state(
    esp_cockpit_handle_t handle,
    esp_cockpit_state_t *out_state);
esp_err_t esp_cockpit_del(esp_cockpit_handle_t handle);
```

### 3.1 创建与注册

- `new()` 创建实例、复制/校验配置并建立内部状态；不得启动 Wi-Fi 驱动、配网门户或 Web Server。配置必须显式选择且只能选择一个网络模式；默认初始化的模式为 `ESP_COCKPIT_NETWORK_MODE_UNSPECIFIED`，未选择或选择无效模式时返回 `ESP_ERR_INVALID_ARG`；托管 Wi-Fi 能力未编译时选择该模式应返回 `ESP_ERR_NOT_SUPPORTED`。
- 首版托管 Wi-Fi 实例配额在 `new()` 时原子检查并保留，第二个托管实例返回 `ESP_ERR_INVALID_STATE`；配额仅在 `del()` 成功后释放。允许创建多个外部网络实例，但因 HTTP 服务资源限制，同时只能有一个实例成功进入活动服务状态。
- 配置结构按职责分组，网络模式与对应网络配置使用互斥联合体表达，避免同时传入托管与外部网络配置。以下仅示意，不构成最终 ABI：

  ```c
  typedef struct {
      esp_cockpit_network_mode_t mode;
      union {
          esp_cockpit_managed_wifi_config_t managed_wifi;
          esp_cockpit_external_network_config_t external;
      } network;
      esp_cockpit_http_config_t http;
  } esp_cockpit_config_t;
  ```

- 提供 `ESP_COCKPIT_CONFIG_DEFAULT()` 初始化宏，并令 `mode` 默认为 `ESP_COCKPIT_NETWORK_MODE_UNSPECIFIED`；应用必须显式设置模式。`new()` 校验模式、互斥字段、配置参数与编译能力，并明确配置字符串等数据是复制还是由调用方保证生命周期。配置缺失或组合无效返回 `ESP_ERR_INVALID_ARG`，选择未编译能力返回 `ESP_ERR_NOT_SUPPORTED`，不得静默猜测资源所有者或降级。
- 托管 Wi-Fi 配置首版只包含启动配网门户所必需的产品选项；不将 STA 凭据作为普通配置字段或 Kconfig 默认值。网络地址、DNS、hostname 等由拥有接口的网络管理方负责。外部网络配置首版只描述 ready/not-ready 的提供方式，优先使用应用主动通知；暂不引入多网口选择、优先级或自动事件适配器。
- 认证、证书、静态资源、RPC 限额等采用独立子配置，避免主配置结构无边界膨胀。
- 应用应在 `start()` 前注册参数、RPC 方法、事件处理器及扩展模块。首版不要求支持运行中修改注册表，以避免引入注册表并发和一致性复杂度。
- NVS、`esp_netif`、默认 `esp_event` loop 等全局设施仍须由应用启动层按契约准备。托管模式负责组件拥有的 Wi-Fi 资源，但不因此拥有或销毁这些共享全局设施。

### 3.2 启动与网络就绪

- `start()` 启动生命周期控制器并返回同步启动结果；返回成功表示控制器已运行，不代表 Web 页面此刻一定可访问。
- Web 服务仅在所选网络模式达到服务就绪条件时启动，并在启动监听器前原子申请进程内活动服务资源。若另一实例已持有该资源，或监听端口与应用/其他组件服务冲突，服务启动失败并通过状态及高阶事件报告错误；必须回滚本次申请，不得伪报 `READY`。
- 外部网络模式下，网络尚未就绪时控制器仍可启动并等待，不启动 HTTP/WebSocket 监听器。`start()` 成功仅表示生命周期控制器启动成功；网络状态变化及其触发的服务启动/停止由控制任务异步处理。
- 托管 Wi-Fi 模式下，组件启动 Wi-Fi Manager 并消费底层 Wi-Fi 事件；配网 SoftAP 可提供服务时即可按策略启动 Web Portal，不要求等待 STA 获取 IP。
- 外部网络模式首版由应用主动报告 ready/not-ready；应用无需等待就绪后再次调用 Web Server 启动 API，也无需逐一转发底层驱动事件。组件不自行推断多接口路由或选择哪个网口作为可服务网络。
- Web Server 生命周期由组件控制器根据网络就绪状态协调。状态可通过 `get_state()` 和高阶组件事件观察；状态事件应能表达等待网络、配网可用、服务就绪及异步故障。
- `esp_cockpit_set_external_network_ready(handle, ready)` 只更新外部网络的“可服务”状态，不代表组件拥有或重新配置该接口。初始状态为 not-ready；应用可在 `start()` 前或之后报告状态，重复报告相同状态应幂等处理。
- `ready=true` 表示外部网络管理器确认当前网络可用于提供 Portal 服务，不绑定某个固定底层事件；监听接口选择、多网口优先级及默认路由不由该布尔通知接口隐式决定。通知调用只投递状态变化，不在调用线程直接启停 Web Server。
- 外部模式从 not-ready 转为 ready 时，生命周期控制器异步启动 Web 服务；启动成功后状态进入 `READY`。若活动服务资源被其他实例占用或监听器启动失败，状态和高阶事件报告错误，不得将 ready 通知伪装成服务已就绪。
- 外部模式从 ready 转为 not-ready 时，组件停止接受新请求、停止 HTTP 服务并关闭现有 HTTP/WebSocket 客户端连接，释放本实例的活动服务资源，状态回到 `WAITING_FOR_NETWORK`。组件保留实例及注册表，但不保留失效连接。
- 外部网络再次 ready 时，控制器重新启动服务；客户端需重新连接、认证、获取状态快照并重新订阅。网络状态通知和服务启动/停止均由控制任务串行处理，避免并发通知导致资源竞争。
- 上述规则是首版外部网络模式的确定行为；后续自动 ESP-IDF 事件适配器若增加，也必须映射到同一 ready/not-ready 状态机，不改变资源所有权语义。

### 3.3 停止与销毁

- `stop()` 停止生命周期控制器、Web Server、配网门户、组件任务/定时器及组件注册的事件处理器，并释放本实例持有的活动 HTTP 服务配额及组件拥有的运行时网络资源。托管模式下停止组件创建/管理的 Wi-Fi 接口；外部网络模式下绝不停止或修改外部 Wi-Fi、Ethernet、DHCP、netif 或共享全局设施。停止不会释放仍存活托管实例的创建配额。
- 停止应拒绝新请求并结束或关闭现有客户端连接；资源清理完成后进入停止态。`del()` 只允许在停止后释放实例和注册表。
- `start()` 在已运行或停止流程中再次调用返回 `ESP_ERR_INVALID_STATE`；对已停止实例再次调用 `stop()` 可幂等成功。停止期间其他不兼容操作返回明确状态错误；部分启动失败返回原始错误并回滚本次已创建资源。
- `stop()` 完成后进入 `STOPPED`；如允许再次 `start()`，必须重新建立组件拥有的运行时资源并按当前网络状态恢复服务。最终 API 应明确实例是否可重启及注册表在 stop/start 间是否保留。
- 插件未启用时调用相应可选功能 API，应返回 `ESP_ERR_NOT_SUPPORTED`，不得静默成功。
- 事件 handler、timer、任务和 WebSocket 会话在销毁实例前必须安全摘除，迟到事件不得访问已释放的实例。回调保持短小，耗时操作由受控任务处理。

外部网络就绪状态控制的是 HTTP/WebSocket 服务可用性，不是组件实例本身的生命周期。网络暂时不可用时实例保持运行并等待恢复；应用显式调用 `stop()` 才停止控制器和释放实例运行时资源。

建议的应用侧调用顺序如下：

```text
应用启动层准备全局设施
  → esp_cockpit_new(config: managed Wi-Fi 或 external)
  → 注册参数、RPC 方法和组件事件处理器
  → esp_cockpit_start()
  → 通过状态查询/高阶事件观察 WAITING_FOR_NETWORK、PROVISIONING、READY 或 ERROR
  → esp_cockpit_stop()
  → esp_cockpit_del()
```

该设计的目标是让应用只需选择网络模式并启动组件，不必自行拼装 Wi-Fi/Ethernet 底层事件链，也不必在网络就绪时再调用第二个 Web 服务启动入口。

## 4. Wi-Fi 配网状态模型

以下逻辑状态用于讨论跨平台行为，不等同于 ESP-IDF `wifi_mode_t` 或最终 API：

| 逻辑状态 | 含义 | 典型行为 |
|---|---|---|
| `DISABLED` | Wi-Fi Manager 未启用或被停止 | 不拥有 Wi-Fi 资源 |
| `IDLE` | Wi-Fi Manager 已运行，但当前不提供 SoftAP 配网门户 | 等待连接策略、应用命令或物理触发 |
| `PORTAL_STARTING` | 正在准备 SoftAP、DHCP/DNS 和门户服务 | 启动结果由异步事件确认 |
| `PROVISIONING` | SoftAP 门户可用，等待凭据或用户操作 | 可提供受限页面、扫描能力（若平台支持）和状态查询 |
| `CONNECTING` | 已接受配网尝试，STA 正在关联/认证/获取 IP | 请求应快速返回“已接受/连接中”；后续结果异步上报 |
| `ONLINE` | 网络管理层确认所需接口就绪 | 托管策略可按产品配置关闭 SoftAP；外部模式由应用状态决定 |
| `RETRY_WAIT` | 正常业务连接暂时断开，等待退避重连 | 仅对托管 STA 重连策略适用 |
| `FAILED` | 当前配网尝试失败，可查询失败类别并重试 | 不清除原有有效配置，除非用户明确请求 |
| `PORTAL_STOPPING` | 正在停止配网服务或切换模式 | 完成资源清理后进入目标状态 |

必须区分“存在已保存凭据”和“网络已连接/可用”。有 SSID 配置不保证 AP 可达、认证成功、DHCP 成功或应用服务可访问。

### 4.1 AP、STA 与 AP+STA

- 是否使用纯 AP、STA 或 AP+STA，由 Wi-Fi Manager 状态和目标平台能力决定；不能将逻辑状态与某个底层 `wifi_mode_t` 永久一一绑定。
- SoftAP 与 STA 共用射频时，关联路由器可能导致信道变化、客户端暂时断连或门户访问中断。配网 UI 必须能够在客户端重连后恢复进度并查询最后结果。
- Wi-Fi 扫描能力应按 ESP-IDF v6.1、具体 target、当前模式及驱动约束验证。不得承诺任意模式均可扫描，也不得假设扫描不会影响 SoftAP 客户端。
- 发现目标网络或提交凭据后的连接操作是异步的。RPC 应先完成请求受理并记录尝试状态，再由网络管理任务执行连接；不能依赖固定的 500 ms 延迟保证响应已送达。
- STA 获得 IP 后是否关闭 SoftAP，以及关闭前的等待/确认策略，应由安全要求、配网 UX 和客户端重连能力共同决定。固定等待 3 秒不是可靠的传输保证；若关闭 AP 导致连接断开，结果必须能在后续会话中查询。
- 配网失败原因应使用稳定的组件级错误类别，并可附带经过筛选的底层原因码。不要把密码或其他凭据写入事件、日志或客户端可见错误。

### 4.2 自动与手动触发策略

可选触发策略包括：

- 首次启动或无有效配置时自动开放受保护的 SoftAP 门户，并设置明确超时。
- 已配置设备断网时按退避策略重连；是否自动开放 SoftAP 应可配置。工业设备可默认不因短暂断网长期开放配网 AP。
- 应用检测到实体按键、维护授权等条件后，调用与 GPIO 无关的门户启停接口。
- 已认证客户端可请求重配网、清除凭据或重启网络；这些操作需检查权限，并定义确认、断连、持久化和恢复行为。

AP 密码必须为设备独立或随机生成的非空密码。密码的生成、交付、轮换及设备身份绑定属于产品安全设计。超时关闭门户不等于清除已保存的 STA 凭据。

## 5. 网络配置与凭据存储

### 5.1 Wi-Fi 凭据

- 明确凭据由 ESP-IDF Wi-Fi 驱动存储还是由应用/独立配置模块存储；避免两套存储成为相互矛盾的事实来源。
- 若使用驱动的 Flash 存储，应核实对应 ESP-IDF v6.1 配置、NVS 分区及加密设置。不能仅因凭据存放在 NVS 就宣称其“天然加密”；保护依赖产品实际启用并正确配置的 NVS Encryption、Flash Encryption 等安全机制。
- “已配置”判断应检查有效配置和策略状态，而不仅是 SSID 字段非空；还应处理凭据损坏、旧版本迁移及外部配置管理器的情况。
- 清除凭据应只影响本组件拥有的目标配置。恢复默认 Wi-Fi 配置等较宽泛操作可能影响其他 Wi-Fi 设置，不应未经确认作为通用清除实现。
- 凭据绝不能通过普通状态查询、诊断事件或日志回传给前端。

### 5.2 IP、DNS 与多网络接口

- 静态 IPv4、DHCP、DNS、hostname 和 mDNS 是网络配置适配器的职责，不属于通用 Web Portal 核心逻辑。
- 应在网络接口就绪时应用 IP 配置，并处理接口重启、DHCP 状态、默认路由和配置持久化；IPv6 需单独定义地址和 DNS 行为。
- Wi-Fi STA、SoftAP 和 Ethernet 可能有不同 netif 与 DNS/DHCP 角色。不得把单个 `sta_netif` 假设扩展成所有目标网络架构。
- 外部网络模式下，ESP-Cockpit 不应直接停止 DHCP client、修改 IP 或 DNS；应通过外部适配器或应用注册的配置方法完成。

## 6. 事件流转与线程/任务约定

### 6.1 内部事件

Wi-Fi Manager（若启用）可订阅相关 ESP-IDF 网络事件来推进内部状态机，例如驱动启动/停止、STA 关联断开、IP 地址变化及 SoftAP 客户端加入/离开。具体事件及 payload 应以 ESP-IDF v6.1 和目标驱动实际定义为准。

事件回调应保持短小：解析必要字段、更新受锁保护的状态或投递到有界队列，然后尽快返回。连接尝试、NVS 写入、WebSocket 广播、DNS/HTTP 服务启停等可能阻塞或耗时的动作，应在受控任务上下文执行。队列满、重复事件、停止过程中的迟到事件都必须有定义的处理方式。

组件必须记录其创建/注册的 handler、task、timer 和网络资源，并在停止时按逆序释放。不得注销或销毁其他模块所有的共享事件循环和资源。

### 6.2 面向应用的高阶事件

组件可提供高阶事件供应用驱动 LED、屏幕、云连接或安全动作。建议定义事件类别而非过早固定最终枚举名：

- Portal 启动、停止或故障。
- 配网门户开启、关闭、超时。
- 网络连接状态变化（接口标识、在线状态、地址变化）。
- 配网尝试开始、成功、失败（稳定错误类别）。
- OTA 开始、完成或失败（如 OTA 模块启用）。

事件 payload 应包含足以判断状态的非敏感字段，不得包含密码、Token 或私钥。事件分发上下文和生命周期应写入 API 契约；如果事件回调可能在共享事件任务执行，调用方不得在回调内进行长时间阻塞操作。

托管模式下，底层 Wi-Fi 事件由组件消费并映射为稳定的组件级事件。外部模式下，组件可订阅标准 ESP-IDF 网络事件或使用应用提供的网络就绪适配器；应用无需重复处理每个底层事件。无论哪种方式，组件只观察外部网络，不取得其资源所有权，也不应要求应用为同一事件链重复注册/转发。

### 6.3 状态读取与通知

- JSON-RPC 查询可经 HTTP 或 WebSocket 返回当前快照；状态变化默认以 WebSocket JSON-RPC 通知发送。若产品启用 SSE，可作为单向只读事件流替代传输，不应默认与 WebSocket 重复推送同一数据集；消息格式和状态版本需后续定义。
- 通知不是唯一事实来源。客户端断线后应重新连接、认证、获取状态快照并重新订阅。
- 对频繁变化的状态可合并更新；发送队列必须有界，慢客户端可触发合并、丢弃非关键更新或断开连接。
- Wi-Fi 事件流和 Web 客户端状态流应解耦，不能让一个慢客户端阻塞 ESP-IDF 事件处理或设备控制任务。

## 7. 建议的集成流程（示意）

```text
应用启动层
  ├─ 按产品所有权初始化 NVS / esp_netif / 默认事件循环
  ├─ 外部模式：初始化应用拥有的 Wi-Fi/Ethernet 网络管理器
  ├─ 创建 ESP-Cockpit 实例，选择托管 Wi-Fi 或外部网络模式
  ├─ 注册业务参数、JSON-RPC 方法及应用事件处理
  └─ 启动 ESP-Cockpit 生命周期控制器（而非手动等待 IP 后启动 Web Server）

ESP-Cockpit
  ├─ 托管 Wi-Fi：启动 Wi-Fi Manager 并消费 Wi-Fi 事件
  ├─ 外部网络：观察标准网络事件或应用提供的就绪状态，不接管接口
  └─ 生命周期控制器：托管模式按配网/接口策略，外部模式按 ready 状态自动启动、停止和恢复 Web 服务

应用维护输入
  └─ 物理按键或维护策略 → 托管 Wi-Fi 模式下请求开启配网门户

Wi-Fi Manager
  ├─ 状态变更 → 内部状态机 → 高阶应用事件
  └─ 状态变更 → WebSocket JSON-RPC 通知/后续状态查询
```

上图是职责流转示意，不承诺固定的启动函数、C API 名称、任务数量或执行顺序。最终接口应支持应用在启动前完成注册，并清楚地区分组件创建、服务启动、配网门户启停和资源销毁。

## 8. 需要进一步决定和验证

### 插件模式与应用集成契约

- 首版进程级托管 Wi-Fi 实例配额、活动 HTTP 服务实例约束及与应用自有监听器的端口协调原则已明确；实现阶段仍需验证 ESP-IDF HTTP Server 的冲突错误传播和资源回滚。
- 每个实例严格选择托管 Wi-Fi 或外部网络之一；Kconfig 控制能力是否编译而不替实例选模式。仍需确定 Wi-Fi 配网等选项的最终名称、依赖/冲突关系及默认配置。
- NVS、`esp_netif`、默认事件循环的前置条件和应用启动契约。
- 外部网络就绪通知 API 的最终签名，以及如需自动事件适配器时指定接口的配置方式。
- 最终公共 C 类型/函数签名、配置字符串所有权规则、插件禁用时错误码，以及停止后同一实例是否可重新启动。
- 启停 API 并发调用的精确错误码及部分启动失败的资源回滚细节。

### 芯片与网络接口兼容性

- 托管 Wi-Fi 的 STA/AP netif 创建、错误恢复及不同 ESP-IDF v6.1 target 的兼容性。
- 外部模式下 Wi-Fi、Ethernet、多网口同时存在时，哪些标准事件代表 Web 服务就绪、监听地址选择和默认路由由谁决定。

### 配网状态机与用户体验

- 首次启动、断网回退、配网超时、物理维护触发和认证后重配网的状态策略。
- APSTA 下扫描、信道变化、SoftAP 客户端断开和 STA 获取 IP 后门户关闭的具体行为。
- STA 获得 IP 后是否关闭 SoftAP，以及关闭前的等待/确认策略，应由安全要求、配网 UX 和客户端重连能力共同决定。

### Portal 安全与凭据生命周期

- DNS/Captive Portal 的开放范围、HTTP 生命周期及 SoftAP 密码的生产配置方式。
- Wi-Fi 凭据所有者、NVS/Flash 加密配置和清除/迁移语义。

### Web 服务与组件生命周期

- 托管 Wi-Fi 模式下 STA 断线、SoftAP 可用性变化与 Web 服务生命周期之间的细化策略；外部网络首版的 ready/not-ready 启停与恢复契约已明确。
- 组件级事件 payload、分发上下文、错误类别及停止过程中的迟到事件处理。
- 托管 Wi-Fi 模式下，`stop()` 停止组件拥有的 Wi-Fi 运行时资源；外部网络始终不得被组件擅自销毁。
