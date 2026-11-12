# ESP-Cockpit 公共 C API 补充规范（草案）

> **文档状态**：补充规范（草案）。
> **权威关系**：本文件补全[最小互操作契约（冻结基线）](./esp-cockpit-interoperability-baseline.md)第 7 节尚未展开的配置结构体、网络生命周期控制、物理安全窗口、设备证书轮换/恢复、可信时间、安全审计及维护锁 C API 声明。本文不得修改或覆盖冻结基线已定稿的注册描述符、回调签名与所有权规则；若存在冲突，以冻结基线为准。本规范保持与 C API 主版本 `ESP_COCKPIT_API_MAJOR = 1` 的源码兼容，不承诺跨 ESP-IDF 或工具链的预编译二进制 ABI。
> 
> 

---

## 1. 补充公共 C 头文件声明（`esp_cockpit.h`）

以下声明与[冻结互操作基线第 7 节](./esp-cockpit-interoperability-baseline.md%237-%E5%85%AC%E5%85%B1-c-%E6%B3%A8%E5%86%8C%E5%9B%9E%E8%B0%83-api)共同构成完整的 `esp_cockpit.h` 公共头文件：

```c
#pragma once

#include <stdbool.h>
#include <stddef.h>
#include <stdint.h>
#include "esp_err.h"

#ifdef __cplusplus
extern "C" {
#endif

/* ==========================================================================
 * 1. 网络生命周期与实例状态（衔接网络生命周期设计补充第 3 节）
 * ========================================================================== */

/**
 * @brief 实例网络管理模式（创建时显式二选一，生命周期内不可切换）
 */
typedef enum {
    ESP_COCKPIT_NETWORK_MODE_UNSPECIFIED = 0, /**< 默认未指定（将导致 new() 返回 ESP_ERR_INVALID_ARG） */
    ESP_COCKPIT_NETWORK_MODE_MANAGED_WIFI,    /**< 托管 Wi-Fi 模式（需编译期启用 Wi-Fi 配网能力） */
    ESP_COCKPIT_NETWORK_MODE_EXTERNAL,        /**< 外部网络模式（首版默认交付模式，由应用管理网络） */
} esp_cockpit_network_mode_t;

/**
 * @brief 组件生命周期控制器状态
 */
typedef enum {
    ESP_COCKPIT_STATE_CREATED = 0,          /**< 已创建，尚未启动 */
    ESP_COCKPIT_STATE_WAITING_FOR_NETWORK,  /**< 已启动，等待网络就绪 */
    ESP_COCKPIT_STATE_PROVISIONING,         /**< 托管模式 SoftAP 配网窗口或本地恢复窗口开放中 */
    ESP_COCKPIT_STATE_STARTING_WEB,         /**< 正在启动 HTTPS/WSS 监听器 */
    ESP_COCKPIT_STATE_READY,                /**< Web Portal 与 JSON-RPC 服务运行中 */
    ESP_COCKPIT_STATE_STOPPING,             /**< 正在停止服务并等待活动回调结束 */
    ESP_COCKPIT_STATE_STOPPED,              /**< 已停止，可再次调用 esp_cockpit_start() */
    ESP_COCKPIT_STATE_ERROR,                /**< 异步启动或运行期致命生命周期错误 */
} esp_cockpit_state_t;

/**
 * @brief 订阅刷新机制内部故障原因（与生命周期状态独立查询）
 */
typedef enum {
    ESP_COCKPIT_REFRESH_FAULT_NONE = 0,           /**< 无故障 */
    ESP_COCKPIT_REFRESH_FAULT_SIGNAL_FAILED,      /**< 内部工作线程唤醒/信号投递失败 */
    ESP_COCKPIT_REFRESH_FAULT_INTERNAL_INVARIANT, /**< 内部刷新状态机异常 */
} esp_cockpit_refresh_fault_reason_t;

typedef struct {
    bool faulted;                                /**< 是否处于刷新内部故障状态 */
    esp_cockpit_refresh_fault_reason_t reason;   /**< 稳定故障原因码 */
    int32_t internal_code;                       /**< 仅用于本地诊断的内部错误码，不得透传至前端 */
} esp_cockpit_refresh_fault_info_t;

/* ==========================================================================
 * 2. 资源上限配置（与冻结基线第 8 节 limits 字段 1:1 对应）
 * ========================================================================== */

/**
 * @brief 实例资源硬上限配置（所有字段必须为正整数，不得为 0）
 */
typedef struct {
    uint32_t max_http_connections;
    uint32_t max_websocket_connections;
    size_t   max_request_bytes;
    size_t   max_response_bytes;
    size_t   max_queued_parameter_bytes;
    size_t   max_id_bytes;
    size_t   max_string_bytes;
    size_t   max_error_message_bytes;
    size_t   max_value_version_bytes;
    uint32_t max_schema_page_items;
    uint32_t max_multi_path_items;
    uint32_t max_queued_requests;
    uint32_t max_queue_wait_ms;
    uint32_t max_subscriptions_per_connection;
    uint32_t max_subscriptions_per_instance;
    uint32_t max_subscription_paths;
    uint32_t max_updates_per_second_per_subscription;
    size_t   max_pending_notification_bytes_per_subscription;
    size_t   max_pending_notification_bytes_per_connection;
    size_t   max_pending_notification_bytes_per_instance;
    uint32_t max_operations;
    uint32_t operation_retention_ms;
    uint32_t max_operation_duration_ms;
    uint32_t max_concurrent_password_verifications;
    uint32_t max_queued_password_verifications;
    uint32_t max_login_attempts_per_minute;
    uint32_t login_failure_window_ms;
    uint32_t max_login_delay_ms;
    uint32_t max_clusters;
    uint32_t max_attributes_per_cluster;
    uint32_t max_commands_per_cluster;
    uint32_t max_application_methods;
    uint32_t max_capabilities;
} esp_cockpit_limits_config_t;

/* ==========================================================================
 * 3. 子模块配置与根配置结构 esp_cockpit_config_t
 * ========================================================================== */

/**
 * @brief 托管 Wi-Fi 模式配置（不含明文 STA 凭据）
 */
typedef struct {
    const char *softap_ssid_prefix;     /**< 配网 SoftAP SSID 前缀（后接设备唯一标识） */
    const char *softap_password;        /**< 设备唯一强随机 SoftAP 密码（不得为空或全局共享） */
    uint32_t portal_timeout_ms;         /**< 配网门户总超时时长 */
    uint32_t online_grace_window_ms;    /**< STA 进入 ONLINE 后 SoftAP 过渡关闭窗口时长 */
} esp_cockpit_managed_wifi_config_t;

/**
 * @brief 外部网络模式配置
 */
typedef struct {
    const char *bind_if_key;            /**< 可选：显式指定监听的 esp_netif key；NULL 表示按 bind_addr 绑定 */
    const char *bind_addr;              /**< 可选：显式指定监听 IPv4/IPv6 地址字符串；NULL 表示绑定指定接口全部地址 */
} esp_cockpit_external_network_config_t;

/**
 * @brief HTTPS / WSS 服务与传输保活配置
 */
typedef struct {
    uint16_t https_port;                /**< 管理面 HTTPS/WSS 监听端口，默认 443 */
    uint32_t ws_ping_interval_ms;       /**< WebSocket 服务端保活 Ping 周期（不续期管理员会话） */
    uint32_t ws_pong_timeout_ms;        /**< WebSocket Pong 响应超时（超时判定为半开死连接并回收） */
} esp_cockpit_http_config_t;

/**
 * @brief 设备身份与入站 TLS 初始配置
 * @note 运行期通过 CSR 导入更新后的证书由组件受保护存储接管，优先于初始配置。
 */
typedef struct {
    const char *device_hostname;        /**< 稳定设备主机名，如 "esp-<factory-id>.local"，须匹配证书 SAN */
    const uint8_t *initial_cert_pem;    /**< 出厂/初始设备证书链（PEM 格式，含 '\0' 结尾） */
    size_t initial_cert_pem_len;        /**< 证书字节长度（含结尾 '\0'） */
    const uint8_t *initial_key_pem;     /**< 可选：初始设备私钥（PEM 格式）；使用安全存储/硬件密钥时可为 NULL */
    size_t initial_key_pem_len;         /**< 私钥字节长度 */
    const char *storage_namespace;      /**< 组件存储设备轮换证书与 KDF 凭据的受保护 NVS 命名空间 */
} esp_cockpit_tls_config_t;

/**
 * @brief 前端静态资源加载与完整性校验配置
 */
typedef struct {
    const uint8_t *app_embedded_gz;     /**< 可选：应用编译期嵌入的单页 gzip 资源指针（优先级高于组件默认资源） */
    size_t app_embedded_gz_len;         /**< 应用嵌入资源字节长度 */
    const char *fs_bundle_path;         /**< 可选：Flash 文件系统运行时更新资源包路径（如 "/spiffs/webui.gz"） */
    const uint8_t *bundle_pubkey_pem;   /**< 可选：用于验证 Flash 资源包 manifest 签名的产品公钥 */
    size_t bundle_pubkey_pem_len;       /**< 公钥字节长度 */
} esp_cockpit_frontend_config_t;

/**
 * @brief ESP-Cockpit 实例主配置结构
 * @note esp_cockpit_new() 深拷贝本结构及其所有指针字段指向的数据。
 */
typedef struct {
    esp_cockpit_network_mode_t mode;
    union {
        esp_cockpit_managed_wifi_config_t managed_wifi;
        esp_cockpit_external_network_config_t external;
    } network;
    esp_cockpit_http_config_t http;
    esp_cockpit_tls_config_t tls;
    esp_cockpit_frontend_config_t frontend;
    esp_cockpit_limits_config_t limits;
} esp_cockpit_config_t;

/**
 * @brief 默认配置初始化宏（强制将 mode 设为 UNSPECIFIED，要求应用显式指定）
 */
#define ESP_COCKPIT_CONFIG_DEFAULT() { \
    .mode = ESP_COCKPIT_NETWORK_MODE_UNSPECIFIED, \
    .http = { \
        .https_port = 443, \
        .ws_ping_interval_ms = 15000, \
        .ws_pong_timeout_ms = 10000, \
    }, \
}

/* ==========================================================================
 * 4. 生命周期控制与高阶事件订阅 API
 * ========================================================================== */

/**
 * @brief 高阶组件事件标识（供应用驱动 LED、屏幕或安全策略）
 */
typedef enum {
    ESP_COCKPIT_EVENT_STATE_CHANGED = 0,        /**< 控制器生命周期状态变更 */
    ESP_COCKPIT_EVENT_INIT_WINDOW_OPENED,       /**< 10 分钟物理管理员初始化窗口已开启 */
    ESP_COCKPIT_EVENT_INIT_WINDOW_CLOSED,       /**< 管理员初始化窗口已关闭（完成初始化或超时） */
    ESP_COCKPIT_EVENT_RECOVERY_WINDOW_OPENED,   /**< 本地身份恢复受限窗口已开启 */
    ESP_COCKPIT_EVENT_RECOVERY_WINDOW_CLOSED,   /**< 本地身份恢复受限窗口已关闭 */
    ESP_COCKPIT_EVENT_REFRESH_FAULT,            /**< 订阅刷新机制发生内部故障 */
    ESP_COCKPIT_EVENT_ADMIN_INITIALIZED,        /**< 首个本地管理员凭据已创建 */
    ESP_COCKPIT_EVENT_FACTORY_RESET_COMPLETED,  /**< 本地用户数据与管理凭据已清除 */
} esp_cockpit_event_id_t;

typedef struct {
    esp_cockpit_event_id_t event_id;
    union {
        struct {
            esp_cockpit_state_t old_state;
            esp_cockpit_state_t new_state;
            esp_err_t last_error;
        } state_changed;
        struct {
            bool completed;                     /**< true 表示成功初始化；false 表示超时关闭 */
        } init_window_closed;
        esp_cockpit_refresh_fault_info_t refresh_fault;
    } data;
} esp_cockpit_event_data_t;

/**
 * @brief 高阶事件回调函数签名（不得在回调内阻塞或递归调用 stop/del）
 */
typedef void (*esp_cockpit_event_handler_t)(
    esp_cockpit_handle_t handle,
    const esp_cockpit_event_data_t *event,
    void *user_ctx);

esp_err_t esp_cockpit_register_event_handler(
    esp_cockpit_handle_t handle,
    esp_cockpit_event_handler_t handler,
    void *user_ctx);

esp_err_t esp_cockpit_unregister_event_handler(
    esp_cockpit_handle_t handle,
    esp_cockpit_event_handler_t handler);

/**
 * @brief 外部网络模式下报告网络可服务状态（幂等，异步投递至控制任务）
 */
esp_err_t esp_cockpit_set_external_network_ready(
    esp_cockpit_handle_t handle,
    bool ready);

/**
 * @brief 查询控制器当前生命周期状态
 */
esp_err_t esp_cockpit_get_state(
    esp_cockpit_handle_t handle,
    esp_cockpit_state_t *out_state);

/**
 * @brief 查询订阅刷新内部故障状态（在 stop() 后及重启失败后仍保持可查，直至下次成功 start()）
 */
esp_err_t esp_cockpit_get_refresh_fault(
    esp_cockpit_handle_t handle,
    esp_cockpit_refresh_fault_info_t *out_fault);

/* ==========================================================================
 * 5. 物理触发安全窗口与恢复出厂 API
 * ========================================================================== */

/**
 * @brief 物理按键触发开启 10 分钟管理员初始化窗口（不使用独立 claim code）
 * @return
 *  - ESP_OK: 窗口已开启（若已开启则返回 ESP_ERR_INVALID_STATE）
 *  - ESP_ERR_INVALID_STATE: 设备已完成管理员初始化或实例未运行
 */
esp_err_t esp_cockpit_open_init_window(esp_cockpit_handle_t handle);

/**
 * @brief 本地身份恢复通道配置（用于旧 HTTPS 私钥/证书失效时的受限文件交换）
 */
typedef struct {
    const char *recovery_ap_ssid;       /**< 恢复模式 SoftAP SSID */
    const char *recovery_ap_password;   /**< 设备独立高熵 SoftAP 密码（必填，不得为空） */
    uint32_t window_duration_ms;        /**< 受限恢复窗口时长上限 */
} esp_cockpit_recovery_window_config_t;

/**
 * @brief 物理触发开启受限本地身份恢复通道
 * @note 仅开放本地 CSR 导出与签发证书导入端点，不暴露常规 RPC 或收集远程恢复码。
 */
esp_err_t esp_cockpit_open_identity_recovery_window(
    esp_cockpit_handle_t handle,
    const esp_cockpit_recovery_window_config_t *config);

/**
 * @brief 执行组件级本地恢复出厂设置
 * @note 清除管理员凭据、活动会话、审计日志及托管 Wi-Fi 凭据；
 *       保留设备出厂/轮换证书、私钥及稳定主机名，不直接授予管理权限。
 */
esp_err_t esp_cockpit_factory_reset(esp_cockpit_handle_t handle);

/* ==========================================================================
 * 6. 设备证书轮换与可信时间状态 API
 * ========================================================================== */

typedef struct {
    char subject_cn[64];
    char san_dns[128];
    int64_t not_before_unix;            /**< 证书生效时间戳（UTC 秒） */
    int64_t not_after_unix;             /**< 证书过期时间戳（UTC 秒） */
    bool is_expired;                    /**< 在可信时间下是否已过期 */
    bool has_pending_csr;               /**< 是否存在尚未完成证书导入的待轮换新私钥 */
} esp_cockpit_cert_info_t;

/**
 * @brief 在设备本地生成新私钥与 PKCS#10 CSR（私钥永不导出）
 * @param[out] out_csr_pem    调用方提供的输出缓冲区（以 '\0' 结尾）
 * @param[in]  csr_capacity   缓冲区容量
 * @param[out] out_csr_len    实际写入的 CSR 字节数（不含结尾 '\0'）
 */
esp_err_t esp_cockpit_generate_csr(
    esp_cockpit_handle_t handle,
    char *out_csr_pem,
    size_t csr_capacity,
    size_t *out_csr_len);

/**
 * @brief 导入厂商签发的设备证书并执行原子切换
 * @note 校验证书链、SAN 主机名匹配、TLS server authentication 用途、有效期及与本地新私钥匹配；
 *       任何校验失败均保留当前有效证书不中断服务。
 */
esp_err_t esp_cockpit_import_certificate(
    esp_cockpit_handle_t handle,
    const uint8_t *cert_pem,
    size_t cert_pem_len);

/**
 * @brief 查询当前活动设备证书状态（供应用与前端触发到期告警）
 */
esp_err_t esp_cockpit_get_certificate_info(
    esp_cockpit_handle_t handle,
    esp_cockpit_cert_info_t *out_info);

/**
 * @brief 注入系统时钟可信度状态（用于证书有效期边界检查与安全审计标记）
 * @param[in] trusted_utc  true 表示系统时钟已通过可信时间源校准；false 表示时钟不可信
 */
esp_err_t esp_cockpit_set_time_trusted(
    esp_cockpit_handle_t handle,
    bool trusted_utc);

/* ==========================================================================
 * 7. 本地安全审计与设备级维护锁 API
 * ========================================================================== */

typedef enum {
    ESP_COCKPIT_AUDIT_INIT = 1,
    ESP_COCKPIT_AUDIT_LOGIN_SUCCESS,
    ESP_COCKPIT_AUDIT_LOGIN_FAILURE,
    ESP_COCKPIT_AUDIT_LOGOUT,
    ESP_COCKPIT_AUDIT_PASSWORD_CHANGED,
    ESP_COCKPIT_AUDIT_SESSION_REVOKED,
    ESP_COCKPIT_AUDIT_CERT_ROTATED,
    ESP_COCKPIT_AUDIT_RECOVERY_WINDOW,
    ESP_COCKPIT_AUDIT_FACTORY_RESET,
} esp_cockpit_audit_event_type_t;

/**
 * @brief 单条安全审计记录（不含凭据、Token 或客户端 IP）
 */
typedef struct {
    uint64_t record_seq;                        /**< 单调递增记录序号 */
    uint32_t boot_id;                           /**< 设备启动计数标识 */
    uint64_t uptime_ms;                         /**< 单调运行时间（毫秒） */
    int64_t  utc_timestamp;                     /**< UTC 时间戳（仅当 time_valid 为 true 时有效） */
    bool     time_valid;                        /**< 触发时设备时钟是否可信 */
    bool     authenticated_admin;               /**< true 归因于管理员；false 归因于未认证尝试 */
    esp_cockpit_audit_event_type_t event_type;
    uint32_t aggregated_count;                  /**< 高频重复事件聚合计数（默认 1） */
} esp_cockpit_audit_entry_t;

/**
 * @brief 读取本地有界安全审计日志
 * @param[in]  start_seq    起始记录序号（传入 0 表示从当前最旧保留记录开始）
 * @param[out] out_entries  调用方分配的记录数组
 * @param[in]  max_entries  数组容量
 * @param[out] out_count    实际读取条数
 * @param[out] out_dropped  因缓冲区满被覆盖的最旧记录累计丢弃数
 */
esp_err_t esp_cockpit_audit_read(
    esp_cockpit_handle_t handle,
    uint64_t start_seq,
    esp_cockpit_audit_entry_t *out_entries,
    size_t max_entries,
    size_t *out_count,
    uint64_t *out_dropped);

/**
 * @brief 获取设备级维护互斥锁（供应用在执行大规模 Flash 擦写或分区维护时与 OTA 互斥）
 * @return
 *  - ESP_OK: 成功获取锁
 *  - ESP_ERR_INVALID_STATE: 锁已被 OTA 或其他维护任务占用（立即失败，不排队、不抢占）
 */
esp_err_t esp_cockpit_maintenance_lock_acquire(esp_cockpit_handle_t handle);

/**
 * @brief 释放设备级维护互斥锁
 */
esp_err_t esp_cockpit_maintenance_lock_release(esp_cockpit_handle_t handle);

#ifdef __cplusplus
}
#endif

```

---

## 2. 内存所有权与执行上下文约定

| API / 数据结构 | 内存所有权规则 | 允许调用上下文 |
| --- | --- | --- |
| `esp_cockpit_config_t` 及所有子结构体指针 | `esp_cockpit_new()` 在返回前完成深拷贝；调用返回后应用可立即释放原始配置内存 | 仅任务上下文（不得在 ISR 中调用） |
| `esp_cockpit_app_descriptor_t` 及其下属描述符/字符串/`user_ctx` | 组件仅借用指针，不深拷贝；应用必须保证其在 `esp_cockpit_del()` 成功返回前保持有效且不可变 | 仅任务上下文 |
| `esp_cockpit_generate_csr()` 的 `out_csr_pem` | 调用方分配并拥有；组件在函数返回前写入以 `\0` 结尾的 PEM 数据 | 仅任务上下文 |
| `esp_cockpit_import_certificate()` 的 `cert_pem` | 调用方拥有；组件在返回前完成解析校验并写入受保护存储，返回后调用方即可释放 | 仅任务上下文 |
| `esp_cockpit_event_handler_t` 的 `event` 参数 | 组件拥有，仅在回调执行期间有效；应用不得保存 `event` 指针供异步使用 | 组件生命周期控制器任务上下文（回调内禁止阻塞或递归调用 `stop()`/`del()`） |

---

## 3. 关键行为与错误处理补充说明

1. **进程级单例与端口冲突回滚**：
* `esp_cockpit_new()` 原子校验进程级托管 Wi-Fi 实例配额（首版上限为 `1`），创建失败时自动回滚已分配副本与配额。


* 控制器进入 `ESP_COCKPIT_STATE_STARTING_WEB` 时原子申请进程级唯一活动 HTTP 服务配额；若端口已被应用自有服务占用或另一实例正在运行，状态转为 `ESP_COCKPIT_STATE_ERROR` 并派发 `ESP_COCKPIT_EVENT_STATE_CHANGED`，同时释放本次配额。




2. **物理初始化窗口与单客户端原子抢占**：
* 仅当管理员尚未初始化且实例处于运行态时，应用检测到物理按键后调用 `esp_cockpit_open_init_window()` 开启 `10` 分钟窗口。


* 首个合法完成 `esp_cockpit.auth.initialize` 的客户端使窗口原子关闭，组件立即派发 `ESP_COCKPIT_EVENT_INIT_WINDOW_CLOSED(completed = true)`；超时未初始化则关闭窗口并派发 `completed = false`。




3. **可信时间与证书有效期强制边界**：
* 系统默认启动时 `trusted_utc` 为 `false`；应用通过 SNTP、RTC 或其他可信时间源完成校准后调用 `esp_cockpit_set_time_trusted(handle, true)`。


* 当 `trusted_utc` 为 `true` 且当前时间超出证书 `[not_before_unix, not_after_unix]` 范围时，组件关闭常规管理端点，仅保留物理触发的 `esp_cockpit_open_identity_recovery_window()` 通道，绝不静默降级为明文 HTTP。




4. **本地身份恢复窗口隔离边界**：
* `esp_cockpit_open_identity_recovery_window()` 要求 `recovery_ap_password` 非空且满足强随机要求（禁止开放无密码 SoftAP）。窗口期内仅挂载受限的 CSR 导出与证书导入端点，不启动 JSON-RPC 分发器，超时或证书导入成功后自动关闭 SoftAP。




5. **审计日志节流与恢复出厂原子性**：
* `ESP_COCKPIT_AUDIT_LOGIN_FAILURE` 等高频事件在组件内部按时间窗口聚合计数（更新 `aggregated_count`），避免单次暴力破解触发高频 NVS 写入。


* `esp_cockpit_factory_reset()` 擦除全部历史审计记录与管理员凭据后，在新日志首条写入 `ESP_COCKPIT_AUDIT_FACTORY_RESET`，确保设备转手后不泄露前任操作历史。





---

## 4. 待实现与实测冻结项

* **默认资源阈值**：`ESP_COCKPIT_CONFIG_DEFAULT()` 中未包含 `limits` 的生产默认数值；该组数值须在 `ESP32-S31-Function-CoreBoard-1`（Wi-Fi 与 Ethernet 两种配置）完成 5+5 并发及 KDF 基准实测后填入并登记至[组件支持矩阵](./esp-cockpit-support-matrix.md)。


* **硬件密钥 Provider 扩展**：若目标硬件采用外置安全元件或芯片数字签名外设（DS Peripheral）保护设备私钥，可在后续次版本中以兼容扩展方式在 `esp_cockpit_tls_config_t` 增加签名回调指针，而不破坏现有结构体源码兼容性。
