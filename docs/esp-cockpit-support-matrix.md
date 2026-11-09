# ESP-Cockpit 组件支持矩阵

## 首版范围与当前状态

首版候选范围仅包含 ESP-IDF v6.1、`esp32s31`、ESP32-S31-Function-CoreBoard-1 上的 Wi-Fi 和板载 Ethernet 两种配置。两种网络接口独立验收；目标范围已确认不代表构建、功能、安全或性能已通过，也不构成生产支持声明。

开发板指南列出的硬件规格：ESP32-S31-WROOM-3 模组，16 MB Flash、16 MB PSRAM、板载 Wi-Fi 6 和带 RJ45 的千兆 Ethernet 接口；板载 Ethernet PHY 通过 RGMII 连接 S31。指南称该板没有历史硬件版本。具体板卡 PCB revision、芯片 revision、PSRAM 构建设置/运行状态及 Ethernet PHY 型号仍需在实测记录中采集。见[ESP32-S31-Function-CoreBoard-1 官方指南](https://docs.espressif.com/projects/esp-dev-kits/zh_CN/latest/esp32s31/esp32-s31-function-coreboard-1/user_guide.html)。

当前没有该目标板的项目构建日志或实测数据。因此延迟门槛、内存余量门槛、资源 `limits` 数值、TLS/KDF 生产参数及验收结果均不得猜填；完成对应测试并归档证据前，两条配置都标记为“候选 / 未验收”。

## 逐配置记录

| 接入配置 | SoC / 板卡 / revision | ESP-IDF / PSRAM | 网络接口 / 驱动 | TLS / KDF | 延迟、内存与资源上限 | 构建、功能、安全、性能结果 | 证据 |
|---|---|---|---|---|---|---|---|
| Wi-Fi | ESP32-S31 (`esp32s31`) / ESP32-S31-Function-CoreBoard-1 / ESP32-S31-WROOM-3；板卡、芯片 revision 待上板记录 | 目标版本 v6.1，精确补丁版本及构建结果待记录；物理 PSRAM 16 MB，构建配置和运行时可用量待确认 | 板载 Wi-Fi 6；生产配置由应用管理。ESP-IDF Wi-Fi 集成与监听接口配置待构建验证 | TLS 后端、安全构建选项、KDF 算法及参数均待目标板比较和冻结 | p95/p99 门槛与观测值：未测；内部 RAM/PSRAM 峰值和最低余量：未测；冻结基线第 8 节全部资源 `limits` 的有限数值：未测 | 构建：未运行；核心功能：未测；TLS/认证/KDF 安全：未测；5+5 性能及资源边界：未测；当前无验收结论 | 硬件规格见官方指南；项目构建、测试报告和日志待归档 |
| Ethernet | ESP32-S31 (`esp32s31`) / ESP32-S31-Function-CoreBoard-1 / ESP32-S31-WROOM-3；板卡、芯片 revision 待上板记录 | 目标版本 v6.1，精确补丁版本及构建结果待记录；物理 PSRAM 16 MB，构建配置和运行时可用量待确认 | 板载 RGMII Ethernet PHY、RJ45（指南标称 10/100/1000 Mbps）；PHY 型号、ESP-IDF Ethernet MAC/PHY 驱动配置待确认 | TLS 后端、安全构建选项、KDF 算法及参数均待目标板比较和冻结 | p95/p99 门槛与观测值：未测；内部 RAM/PSRAM 峰值和最低余量：未测；冻结基线第 8 节全部资源 `limits` 的有限数值：未测 | 构建：未运行；核心功能：未测；TLS/认证/KDF 安全：未测；5+5 性能及资源边界：未测；当前无验收结论 | 硬件规格见官方指南；项目构建、测试报告和日志待归档 |

## 每配置验收记录要求

每条配置通过测试后，补齐并链接不可变报告，至少记录：

- 实际 SoC target/revision、模组和板卡标识/revision、ESP-IDF 精确版本、组件/固件版本及 commit。
- PSRAM 容量、Kconfig/构建选项、初始化结果和运行时可用量；Wi-Fi 或 Ethernet 的驱动、接口及监听配置。
- TLS 后端、证书与安全构建选项，以及候选 KDF 对比结果；冻结最终 KDF 算法、盐/参数格式及资源参数。
- 在既定 5+5 负载下的请求成功/拒绝数、HTTP 与通知延迟分布、连接断开/重连、崩溃/断言和持续内存增长结果。
- 空闲、TLS 握手、5 个 WebSocket 持续收发及 KDF 验证时的内部 RAM/PSRAM 峰值、最低余量和测试前后资源变化。
- 对每条配置单独测定并登记冻结基线第 8 节的全部 `limits` 字段；字段名和单位不得更改。记录其拒绝/断连行为，并验证超限不会崩溃或静默丢弃。
- 依据该配置实测基线确定并记录 p95/p99 延迟门槛和最低空闲内存余量；在门槛确定并通过回归前，不得标记性能验收通过。

仅当某条接入配置完成 IDF v6.1 构建、核心功能、安全、资源上限和正式 5+5 性能验收，且证据齐备后，才可单独将该配置升级为已验证支持。另一接入配置、同 target 其他板卡或不同生产构建均不得由此推定支持。
