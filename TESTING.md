# BLE 性能测试标准指南

本指南定义 README 成绩表所使用的 GATT 与 L2CAP CoC 单向饱和吞吐测试。基准口径源自 CH32V208WBU6 在 Mac mini 上的历史正式测试，目的是让不同芯片的结果可复核、可比较。

## 1. 测试对象与方向

- 芯片端作为 BLE Peripheral，Mac mini 作为 Central，使用 macOS 系统 CoreBluetooth，不以手机、USB 蓝牙适配器或另一块开发板替代中心端成绩。
- **上行（up）**：Mac mini → 芯片；**下行（down）**：芯片 → Mac mini。
- 测量接收端验证通过的**单向应用载荷**。不回显后再驱动下一包，不把双向流量相加，不把协议头或控制包计入成绩。
- 探索性回显、延迟、长时可靠性、Coded PHY 和射频仪器测试可以另行记录，但不能作为本指南的饱和吞吐成绩。

## 2. 固定测试环境

1. 使用 Release 固件，记录完整芯片型号、实际 CPU 主频、Host/Controller 协议栈及版本、Host ↔ Controller 通信方式、工程 URL 和源码版本。
2. 对比测试尽量使用同一台 Mac mini、相同系统客户端、板卡摆放和射频环境。必须记录 Mac 型号、架构、macOS 版本和测试日期；历史 CH32 基线为 Mac14,12（arm64）、macOS 15.7.5。
3. 关闭逐包 DEBUG/INFO 日志和载荷打印，保留必要的 WARN/ERROR、连接、PHY 与统计日志。**日志通道必须非阻塞**：可以使用 UART DMA、已验证不会暂停 CPU 或阻塞协议栈任务的 RTT，或其他异步缓冲方式。WCH 当前的 RTT 访问存在暂停 CPU 的已知问题，因此 CH32 历史测试使用 USART1 DMA；这是平台特定限制，不能据此禁止其他 ARM 或芯片使用已验证的非阻塞 RTT。禁止阻塞轮询输出、同步等待调试器，以及测量期间会 halt CPU 的 GDB/probe 访问。记录所用日志方式和丢日志计数。
4. Mac 测试期间防止自动睡眠，避免其他程序同时控制蓝牙客户端或调试器。不得通过修改系统蓝牙设置来提高成绩。
5. GATT 数据库变更后处理 macOS 缓存，确保实际发现的是新服务；不能使用缓存中的旧句柄或旧特征完成测试。

## 3. 测试矩阵与能力适配

双通道、双 PHY 均支持时，完整测试矩阵为：

```text
2 种通道（GATT、L2CAP CoC）
× 2 种 PHY（LE 1M、LE 2M）
× 2 个方向（上行、下行）
× 2 次独立重复
= 16 项
```

**16 项是完整能力配置的覆盖上限，不是所有芯片的准入门栏。**正式测试前先确认芯片、协议栈和客户端共同支持的通道与 PHY，固定本次计划矩阵：

- 每种受支持的“通道 × PHY”均测试上行、下行，各独立重复 2 次，即 4 项。只支持 1M 时，两种通道合计 8 项；只有一种通道和一种 PHY 时，合计 4 项，仍可填写对应成绩表。
- 不支持的通道或 PHY 标记“不支持”并说明依据，不执行、不记 0，也不因此判定整组测试失败。声明支持但实测失败的案例仍记“未通过”，不得在失败后改标“不支持”。
- 不同芯片的能力可以不同，但同一次正式矩阵的包长、计时和计算口径必须固定，不能只保留较快配置或单次最好成绩。

每一实际执行项使用独立连接，连续发送/生产约 **15 秒**，随后停止生产并等待队列尾部排空。最终成绩使用同一项两次结果的算术平均值，不能把不同组别的结果合并求均值。

### GATT

- 上行使用 Write Without Response，按 CoreBluetooth 可写状态持续填充，并在 ready 回调后恢复；下行由芯片持续排入 Notification，不得使用 Indication 的逐包确认代替饱和发送。
- **244 B 是推荐值和 CH32 历史配置，不是强制要求。**根据实际 ATT MTU、DLE、协议栈缓冲和 CoreBluetooth 写入上限选择固定载荷；能力不足时使用更小值。为便于对比，能力充足时可沿用 244 B；选择更大值也可记录，但必须注明，不能宣称与 244 B 结果属于完全同等条件。
- 上行载荷不得超过实际 ATT MTU 减 3 B，也不得超过 `maximumWriteValueLength(for: .withoutResponse)`；下行 Notification 同样受实际 ATT MTU 限制。CoreBluetooth 不同写入类型的返回值可能不同，应使用本次实际写入类型核验。
- 每包前 4 字节为递增序号，其余为可复现 pattern；序号字段计入应用载荷。接收端检查序号、内容和最终总字节数。
- ATT MTU 按双方能力尽力协商，记录实际 ATT MTU 和本次载荷长度。CH32 的“244 B 载荷 + 3 B ATT 头 + 4 B L2CAP 头 = 251 B”只是历史配置示例，不是其他芯片的门槛；不得仅凭 preferred MTU 或 DLE 推算实际 ATT MTU。

### L2CAP CoC

- 使用 LE Credit Based Connection，PSM `0x0080`。CH32 历史配置为本端接收 MTU **2048 B**、MPS **247 B**、初始 **9 credits**、测试 SDU **512 B**；这些均为推荐目标，不是强制要求。
- 根据双方实际 MTU、MPS、credits 和缓冲能力选择固定测试 SDU，记录实际值。上行通过 `CBL2CAPChannel` 的 OutputStream 连续写入，遵守 Stream 可写事件和 credits 背压；下行由芯片持续发送同一选定长度的 SDU。
- Mac Stream 的读写粒度不等于空口 SDU，接收端按全局字节偏移验证 pattern，允许部分读写和重新分段。
- 分别记录本端 CoC MTU、Mac 对端 CoC MTU 和 MPS。CoC 不是对称的单一 MTU；不能拿本端配置值冒充 Mac 实际通告值。

### 链路参数

- 对双方声明支持的 1M、2M 尽力协商并分别测试，**不要求必须支持或协商到 2M**。PHY 在传输前后均须读取并确认 TX/RX 与本次目标一致，不能以请求成功或理论速率代替实际状态。
- DLE 按双方能力尽力扩展，251 B 是推荐目标而非门槛；传输结束时记录实际 DLE TX/RX 长度。
- 准备阶段可请求 7.5–15 ms 连接间隔，由中心端决定最终结果；15 ms 是 CH32 历史实测值而非通用门槛，必须记录传输时的实际连接间隔。
- 实际 MTU、包长、MPS、credits、PHY、DLE 或连接间隔与历史基线不同，不妨碍如实记录有效成绩；但必须注明差异，不作完全同等条件下的直接排名。

## 4. 饱和、计时与计算口径

- 上行在 Mac 侧遵循 `canSendWriteWithoutResponse`/ready callback 或 Stream 背压；必须观察到主机背压，证明发送队列确实受压。
- 下行由芯片持续补充发送队列，并记录生产受阻、缓冲余量及 More Data 等证据；必须观察到生产受阻和发送积压。背压只证明施加了对端无法立即消化的负载，不代表达到芯片理论极限。
- 上行从芯片处理 START 到最后一个有效数据字节，以芯片端高精度单调计时器计时；历史 CH32 使用 TIM3 1 MHz cputime，不能用低速 RTOS tick 估算。
- 下行由 Mac 单调时钟计量，从传输开始（若先到数据则取首个接收回调）到最后一个校验成功的接收回调。
- 两端计时都包含尾部排空，排除扫描、连接、服务发现、PHY/CI/DLE 设置及最终统计和断开。

每项吞吐按下式独立计算：

```text
payload_B_per_s = verified_bytes / transfer_seconds
payload_kB_per_s = verified_bytes / transfer_seconds / 1000
payload_kbps = verified_bytes × 8 / transfer_seconds / 1000
最终成绩 = 两次 payload_kB_per_s 的算术平均值
```

`verified_bytes` 必须是接收端实际收到且校验通过的字节数。Mac 已排队、芯片已提交或队列待发送的字节不能当作已交付吞吐。

## 5. 执行流程

### 通用流程

1. 构建并烧录目标固件，正常复位后独立运行，启动第 2 节所述的非阻塞日志采集。
2. 确认芯片、协议栈和客户端能力，记录不支持的通道或 PHY，并据此固定本次计划矩阵及各项包长。
3. 构建 Mac 客户端并运行其自测；扫描并精确选择目标，确认服务、特征和测试身份正确。
4. 运行本次计划矩阵，每项记录实际 ATT/CoC MTU 和包长，确认传输前后的 PHY，并核对传输时的 CI 和结束时的 DLE。
5. 任一实际执行项失败即停止，不得自动重试失败案例后只保留成功结果。修复问题后应重新运行本次计划矩阵，并生成新的证据目录。
6. 保存原始结果，计算两项重复的均值后填写 README；展示时统一保留两位小数，计算过程中不提前舍入。

### CH32 历史工程复现示例

以下命令在 [`ch32v208-nimble`](https://github.com/kaidegit/ch32v208-nimble) 根目录执行；同级 `../nimble` 应为 Apache NimBLE 源码，历史测试版本为 `a86045acb7635cabac0e53d7bdee44a99359a887`。本示例复现 CH32 的完整 16 项历史配置，不要求其他芯片照搬其 MTU、包长或 PHY 能力。

```sh
cmake -S . -B build -DCH32_SYSCLK_MHZ=96 \
  -DCH32_BLE_TEST_IDENTITY=ON -DCH32_BLE_PACKET_LOGS=OFF
scripts/build.sh
scripts/run.sh --verify
tools/macos-ble/ble.sh --self-test

# 独立终端采集 UART，串口名按实际情况替换；测量期间不使用 probe。
scripts/log.sh -p /dev/cu.usbmodemXXX -o build/perf-uart.log

# 必须显式传参；perf.py 自身默认的 30 秒 × 3 次不是本表历史基线。
python3 tools/macos-ble/perf.py --duration 15 --repeats 2
```

单项调试可以使用工程中的 `ble.sh probe --perf` 和 `ble.sh l2cap --perf`，但单项、短测或 `--echo-perf` 结果不能替代本次计划矩阵。

## 6. 有效成绩判定

正式成绩必须同时满足：

- 本次计划矩阵内所有实际执行项通过，客户端返回码和 summary 均为成功，`failures = 0`，`verified_bytes > 0`。完整能力配置为 16/16；能力较少时为对应的 N/N，不支持的通道或 PHY 不计入 N。
- 每种“通道 × PHY”至少有上行、下行各两次有效结果，才能填写该组最终均值；缺项或失败不能由另一次较快的重复代替。
- 每项有效传输时间达到约定时长（历史 runner 的判定为 `transfer_seconds >= duration - 0.25 s`），且完成尾部排空和最终字节数核对。
- 序号、pattern、字节总数均正确，无丢失、额外字节、停滞、意外断开或校验错误。
- 每项至少取得传输前、后两次匹配目标且成功的 PHY 状态；实际 MTU、包长、CI、DLE 按第 3 节记录，不要求所有芯片达到 CH32 的历史数值。
- 上下行均有饱和证据；所有原始失败、背压、资源和 PHY 计数均保留，不删除异常记录。
- 不把 PHY CRC 错误或 RX 超时直接换算为空口 PER，也不要求这些射频计数必须为零；应保留原值并在需要时单独说明。

证据目录至少保存 `report.json`、每项 JSONL/stderr、所选非阻塞通道的原始日志、固件 ELF 及 SHA256、客户端和 runner 源码快照及哈希、Git revision/diff、协议栈依赖版本、构建选项、能力/跳过项说明和 Mac 系统信息。保存的 ELF 本身不构成对板内固件的远程证明，烧录后仍需核对并独立复位运行。

## 7. 填写 README 成绩

- GATT 与 L2CAP CoC 分表填写；每个芯片实际完成测试的每种 PHY 单独一行。未支持或未通过某项不影响填写其他已经通过且证据完整的项目。
- 每行必须包含芯片名称、主频、Host 协议栈、Controller 协议栈、Host ↔ Controller 通信方式、与 Mac mini 实际协商的 MTU 和 PHY、工程地址、真实测试日期及最终上行/下行成绩。
- 实际 GATT 载荷、CoC SDU、MPS、credits、CI 和 DLE 也应写入来源记录或备注；这些参数与 CH32 历史值不同并不使成绩失效，但比较时必须说明条件差异。
- 协议栈或依赖填写可定位的版本；工程链接应尽量指向被测源码版本或附带 commit。主频、协议栈、通信方式、MTU/credits 或测试口径变化时另增记录，不覆盖旧历史。
- 最终成绩按“上行 / 下行（kB/s）”填写两次均值，原始单次结果和 kbps 可在来源记录中查阅。
- 历史资料缺少某字段时填“未记录”，不支持时填“不支持”，失败时填“未通过”；不得猜测、用配置值代替实测值或用 0 代替缺失成绩。
- 新记录必须链接可访问的来源或证据。不同 Mac、测试时长、包长或链路参数的结果应单独标注，不与条件不同的成绩混合求均值或直接排名。
