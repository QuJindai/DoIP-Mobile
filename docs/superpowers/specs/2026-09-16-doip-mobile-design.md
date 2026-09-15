# DoIP-Mobile V1 设计规范

日期：2026-09-16
状态：设计冻结，等待实现计划

## 1. 产品目标

DoIP-Mobile 是一个开源 Android 原生诊断应用。手机通过 USB-C 以太网适配器/一体式 OBD-DoIP Type-C 线直接连接车辆，不依赖 CAN、ELM327、蓝牙、Wi-Fi 或专有 VCI。

V1 的唯一目标是可靠跑通以下闭环：

1. Android 识别 USB-C Ethernet 网络；
2. 将 DoIP Socket 明确绑定到该 Ethernet `Network`；
3. UDP/13400 完成 Vehicle Identification；
4. 建立 TCP/13400 连接；
5. 完成 Routing Activation；
6. 发送 UDS `22 F1 90`；
7. 正确解析并显示 VIN；
8. 全过程保留可导出的原始 TX/RX 日志。

首个验收版本默认只读。清 DTC、ECU Reset、SecurityAccess、RoutineControl、刷写等改变车辆状态的功能不进入 V1 首个验收里程碑。

## 2. 非目标

V1 不实现：

- CAN / CAN FD；
- ISO-TP；
- ELM327；
- 蓝牙或 Wi-Fi VCI；
- Python 运行时；
- JNI/C++ DoIP 栈；
- OEM 在线账户、云诊断、SFD/证书体系；
- ECU 刷写；
- 商城、维修知识库、AI 故障结论等与链路验证无关的功能。

## 3. 技术路线

### 3.1 Android 技术栈

- Kotlin；
- Jetpack Compose + Material 3；
- Gradle Kotlin DSL；
- `minSdk = 29`；
- `targetSdk = 36`；
- Kotlin Coroutines / Flow；
- Android `ConnectivityManager` / `Network` API；
- Java/Kotlin 原生 `DatagramSocket`、`Socket`/`SocketChannel`；
- 不要求 Root，不修改 Android 内核。

### 3.2 网络原则

DoIP 不能依赖系统默认网络路由，因为手机同时可能存在蜂窝数据和 Wi-Fi。应用必须：

1. 枚举 `TRANSPORT_ETHERNET` 网络；
2. 显示接口状态、IPv4、本地 Link Properties；
3. 用户选择或自动选择 Ethernet 网络；
4. 使用 `Network.bindSocket(...)` 将 UDP/TCP Socket 显式绑定到车辆 Ethernet 网络；
5. 网络消失时立即关闭当前 DoIP Session 并回到未连接状态。

V1 不修改全局默认网络。

## 4. 分层架构

```text
Compose UI
    |
ViewModel / Session State
    |
DiagnosticFacade
    |---------------------|
    |                     |
UdsClient             DoipClient
                          |
              DoipCodec / Message Models
                          |
              AndroidEthernetTransport
                          |
                 Android Network API
                          |
                 USB-C Ethernet
                          |
                    Vehicle DoIP
```

### 4.1 `network` 模块

职责：

- 发现 Ethernet Network；
- 提供 IPv4 / LinkProperties；
- 对 UDP/TCP Socket 执行 network binding；
- 暴露网络丢失事件；
- 不包含任何 DoIP 协议语义。

核心接口：

```kotlin
interface VehicleNetworkProvider {
    val ethernetNetworks: StateFlow<List<VehicleNetwork>>
    suspend fun bind(socket: DatagramSocket, networkId: String)
    suspend fun bind(socket: Socket, networkId: String)
}
```

### 4.2 `doip` 模块

职责：

- ISO 13400 DoIP Header 编解码；
- UDP Vehicle Identification；
- Vehicle Announcement / Identification Response 解析；
- TCP 连接；
- Routing Activation；
- Alive Check；
- Diagnostic Message；
- Diagnostic ACK/NACK；
- 超时、连接断开和协议错误分类。

首批实现并测试的 Payload Type：

- `0x0001` Vehicle Identification Request；
- `0x0004` Vehicle Announcement / Identification Response；
- `0x0005` Routing Activation Request；
- `0x0006` Routing Activation Response；
- `0x0007` Alive Check Request；
- `0x0008` Alive Check Response；
- `0x8001` Diagnostic Message；
- `0x8002` Diagnostic Message Positive Acknowledgement；
- `0x8003` Diagnostic Message Negative Acknowledgement。

协议版本、inverse version、payload length 必须严格校验；未知 payload type 保留原始数据并记录，不导致应用崩溃。

### 4.3 `uds` 模块

V1 只实现最小只读集合：

- `0x22 ReadDataByIdentifier`；
- `0x19 ReadDTCInformation`；
- `0x3E TesterPresent`；
- 通用 Positive Response / Negative Response (`0x7F`) 解析。

首个真实车辆验收只强制 `22 F1 90` VIN。

V1 预留但不在首个验收开放：

- `0x10 DiagnosticSessionControl`；
- `0x11 ECUReset`；
- `0x14 ClearDiagnosticInformation`；
- `0x27 SecurityAccess`；
- `0x31 RoutineControl`；
- `0x34-0x37` Download/Transfer/Exit。

### 4.4 `app` 模块

只保留高价值界面：

1. **Connection**：Ethernet 状态、接口/IP、DoIP Discovery；
2. **Vehicle**：VIN、EID、GID、DoIP logical address、IP；
3. **ECU**：手动/配置化 logical address，后续扩展扫描；
4. **UDS**：VIN、DID、DTC 只读操作；
5. **Raw Console**：输入 UDS hex、显示 DoIP 封装后的 TX/RX；
6. **Logs**：时间、方向、协议层、十六进制报文、错误。

V1 不做复杂仪表盘或维修业务 UI。

## 5. 会话状态机

```text
NO_ETHERNET
   -> ETHERNET_READY
   -> DISCOVERING
   -> VEHICLE_FOUND
   -> TCP_CONNECTING
   -> ROUTING_ACTIVATING
   -> DIAGNOSTIC_READY
   -> UDS_REQUEST_ACTIVE
```

任一阶段出现 Ethernet 丢失、TCP 断开、协议错误或超时：

- 结束当前请求；
- 关闭 socket；
- 写入结构化错误日志；
- 根据 Ethernet 是否仍存在回到 `ETHERNET_READY` 或 `NO_ETHERNET`；
- 不伪造成功状态。

## 6. 配置

V1 提供 Advanced Settings：

- Tester logical address：默认 `0x0E00`，可修改；
- Target logical address：不硬编码，来自车辆响应、用户输入或车型配置；
- DoIP port：默认 `13400`；
- UDP Discovery timeout：默认 2 s；
- TCP connect timeout：默认 3 s；
- UDS P2 timeout：默认 2 s；
- Routing Activation Type：默认值可配置。

所有 OEM 差异项必须可配置，不在协议核心中写死。

## 7. 日志与可观测性

每条记录包含：

- monotonic timestamp + wall clock；
- network/interface；
- UDP/TCP；
- source/destination IP:port；
- DoIP payload type；
- source/target logical address；
- UDS SID；
- TX/RX 原始 hex；
- 解析结果或错误。

日志默认只保存在设备本地。导出时生成纯文本/JSON；VIN 等车辆标识在用户主动导出时保留，不自动上传。

## 8. 安全边界

首个验收版本采用 read-only 产品策略：

- 不提供 Clear DTC 按钮；
- 不提供 ECU Reset；
- 不执行 SecurityAccess；
- 不执行 RoutineControl；
- 不执行刷写；
- Raw Console 默认只允许白名单 SID：`22`、`19`、`3E`；
- 任何未来写操作必须增加明确确认、车辆状态提示、审计日志和独立测试。

## 9. 测试策略

### 9.1 单元测试

必须覆盖：

- DoIP Header serialize/parse；
- protocol version / inverse version 校验；
- payload length 边界；
- Identification response；
- Routing Activation response；
- Diagnostic Message / ACK / NACK；
- UDS `62 F1 90` VIN；
- UDS Negative Response；
- malformed / truncated frame；
- TCP 分片和多帧粘包解析。

### 9.2 JVM 集成测试

使用 Apache-2.0 的 `doip-sim-ecu-dsl` 作为测试侧参考/可选测试依赖，模拟：

- 一个 DoIP Gateway；
- Vehicle Announcement；
- Routing Activation；
- logical address；
- `22 F190` 返回固定 VIN；
- 超时；
- NACK；
- TCP 中断。

核心协议层不得依赖 Android Framework，以保证大部分测试可以在 JVM CI 中执行。

### 9.3 Android 仪器测试

验证：

- Ethernet 网络枚举；
- socket 绑定 API；
- Ethernet 丢失状态迁移；
- Compose 状态与后台会话一致。

### 9.4 实车验收

第一阶段只要求证明：

```text
USB-C Ethernet detected
 -> UDP Vehicle Identification response
 -> TCP 13400 connected
 -> Routing Activation accepted
 -> UDS 22 F1 90
 -> valid VIN displayed
 -> complete TX/RX log exported
```

任何一步失败都必须显示真实失败位置和原始证据，不得以模拟数据替代。

## 10. 开源复用原则

- DoIP 协议行为参考 `jacobschaer/python-doipclient`（MIT），但 Android 核心使用 Kotlin clean-room 实现；
- 并发/状态机可参考 `Gitlittlerubbish/YA-DoIP`（MIT）；
- 测试模拟参考/可选依赖 `doip-sim-ecu/doip-sim-ecu-dsl`（Apache-2.0）；
- `MEET-Mecanicos-Especialistas-En-Todo` 当前许可证未核验，仅作架构观察，不复制其源码；
- 本项目采用 Apache License 2.0，便于商业使用、修改与专利条款治理。

## 11. V1 完成定义

V1 只有在以下条件全部满足时才称为完成：

- `./gradlew test` 全通过；
- Android debug APK 可构建；
- 无 Root；
- 无 CAN/ISO-TP 依赖；
- 无 Python/JNI 运行时；
- 模拟 ECU 闭环自动测试通过；
- S24U 能识别 USB-C Ethernet；
- 实车完成 DoIP Discovery + Routing Activation + VIN 读取；
- 日志能证明每一个协议步骤；
- 无假数据兜底。
