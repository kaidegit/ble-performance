# BLE 性能测试成绩

记录不同芯片以 **Mac mini 为中心端**时的 GATT 与 L2CAP CoC 单向饱和吞吐成绩。统一测试方法、有效成绩条件及新增记录要求见[测试标准指南](TESTING.md)。

- **上行**：Mac mini → 芯片；**下行**：芯片 → Mac mini。
- 成绩单位为十进制 **kB/s**（1 kB/s = 1000 B/s），统计接收端实际收到且校验通过的应用载荷，不含 ATT、L2CAP、LL 头和协议控制包。
- 每格成绩为“**上行 / 下行**”，CH32 历史成绩取两次独立测量的算术平均值；本次 N32 补充测量取三次均值，另标 PHY 待核验。均保留两位小数，不取单次最好成绩、双向合计值或 PHY 标称速率。
- 当前 CH32 历史基线使用 Mac14,12（arm64）、macOS 15.7.5、Release 固件；所有案例实际连接间隔均为 15 ms，DLE TX/RX 均为 251 B。
- 表中的包长、MTU、PHY 和链路参数是 CH32 的历史实测配置，不是其他芯片的准入门槛。新芯片按实际能力尽力协商并记录；不支持的通道或 PHY 不记 0，也不影响其他有效成绩入库。

## GATT

CH32 历史测试使用 244 B 无应答写入（上行）和 244 B Notification（下行），不采用逐包回显或 Indication 确认。

| 芯片名称 | 主频 | Host 协议栈 | Controller 协议栈 | Host ↔ Controller 通信方式 | 与 Mac mini 实际协商 MTU | 实际 PHY 速率 | 工程地址 | 测试日期 | 最终成绩：上行 / 下行（kB/s） |
| --- | ---: | --- | --- | --- | --- | --- | --- | --- | ---: |
| CH32V208WBU6 | 96 MHz | Apache NimBLE Host（`a86045a`） | Apache NimBLE Controller/LL（`a86045a`）+ WCH 射频寄存器驱动 | 单芯片 native/native HCI，直接函数调用，共享 MSYS | ATT MTU 256 B | LE 1M，TX/RX 均为 1M | [ch32v208-nimble](https://github.com/kaidegit/ch32v208-nimble) | 2026-10-31 | **28.48 / 28.08** |
| CH32V208WBU6 | 96 MHz | Apache NimBLE Host（`a86045a`） | Apache NimBLE Controller/LL（`a86045a`）+ WCH 射频寄存器驱动 | 单芯片 native/native HCI，直接函数调用，共享 MSYS | ATT MTU 256 B | LE 2M，TX/RX 均为 2M | [ch32v208-nimble](https://github.com/kaidegit/ch32v208-nimble) | 2026-10-31 | **56.06 / 52.23** |
| N32WB452CEQ6（nimble 模式） | M4F 144 MHz | 板载 Apache NimBLE Host（源码固化于所链工程提交） | NZ8801 闭源 RivieraWaves Controller（Nations SDK 2.5.0；ROM/Patch 版本未独立核验） | M4F Host → 片内 SPI H4 → NZ8801 | ATT MTU 185 B；载荷 182 B | 默认 LE 1M；PHY 查询不支持，未独立核验 | [n32wb452-nimble@2beed5f](https://github.com/kaidegit/n32wb452-nimble/tree/2beed5f1878829e221b27a81bf24ac704f769032) | 2026-11-02 | **27.84 / 9.72**（PHY 待核验） |
| N32WB452CEQ6（bridge 模式） | M4F 144 MHz；Host 运行于 Mac mini | macOS blectl + Apache NimBLE Host（源码固化于所链工程提交） | NZ8801 闭源 RivieraWaves Controller（Nations SDK 2.5.0；ROM/Patch 版本未独立核验） | Mac Host → USB CDC H4 → M4F bridge → SPI H4 → NZ8801 | ATT MTU 185 B；载荷 182 B | 默认 LE 1M；PHY 查询不支持，未独立核验 | [n32wb452-nimble@2beed5f](https://github.com/kaidegit/n32wb452-nimble/tree/2beed5f1878829e221b27a81bf24ac704f769032) | 2026-11-02 | **28.63 / 9.81**（PHY 待核验） |

N32 两行为补充测量：数据完整性、持续负载和实际 DLE/CI 已验证，但尚未满足 [TESTING.md 第 6 节](TESTING.md#6-有效成绩判定)“传输前后 PHY 状态读取成功”的完整有效成绩条件，因此不作为 PHY 已核验的正式排名。每方向 **3×30 秒**、182 B 负载、实际 DLE TX/RX=251 B（2120 µs）、CI=15 ms；与 CH32 的 2×15 秒、244 B 负载分别标注，不能视为完全同等条件。详见下方 N32 来源记录。

## L2CAP CoC

CH32 历史测试使用 LE Credit Based Connection，PSM `0x0080`，测试发送 SDU 为 512 B，初始 9 credits。CoC 两个方向各有 MTU，因此同时列出芯片本端和 Mac 对端的实际值，不用单一“协商 MTU”代替。

| 芯片名称 | 主频 | Host 协议栈 | Controller 协议栈 | Host ↔ Controller 通信方式 | 与 Mac mini 实际协商 MTU | 实际 PHY 速率 | 工程地址 | 测试日期 | 最终成绩：上行 / 下行（kB/s） |
| --- | ---: | --- | --- | --- | --- | --- | --- | --- | ---: |
| CH32V208WBU6 | 96 MHz | Apache NimBLE Host（`a86045a`） | Apache NimBLE Controller/LL（`a86045a`）+ WCH 射频寄存器驱动 | 单芯片 native/native HCI，直接函数调用，共享 MSYS | 本端 2048 B；Mac 对端 1498 B；MPS 247 B | LE 1M，TX/RX 均为 1M | [ch32v208-nimble](https://github.com/kaidegit/ch32v208-nimble) | 2026-10-31 | **20.45 / 29.73** |
| CH32V208WBU6 | 96 MHz | Apache NimBLE Host（`a86045a`） | Apache NimBLE Controller/LL（`a86045a`）+ WCH 射频寄存器驱动 | 单芯片 native/native HCI，直接函数调用，共享 MSYS | 本端 2048 B；Mac 对端 1251 B；MPS 247 B | LE 2M，TX/RX 均为 2M | [ch32v208-nimble](https://github.com/kaidegit/ch32v208-nimble) | 2026-10-31 | **30.82 / 50.78** |

## CH32 历史成绩来源

- 原始记录：[macOS L2CAP / GATT 饱和吞吐实测（2026-10-31）](https://github.com/kaidegit/ch32v208-nimble/blob/main/docs/l2cap-gatt-perf-2026-10-31.md)。其中保留了每次测量值、kB/s 与 kbps、固件和客户端哈希及测试限制。
- 正式测试于 2026-10-31 17:41:34–17:46:54（Asia/Shanghai）执行，每项连续生产约 15 秒并等待队列排空，2 种通道 × 2 种 PHY × 2 个方向 × 2 次，共 **16/16 项通过**，累计校验 **8,970,200 B**。
- 表内均值已从原始 `report.json` 的 16 项结果重新计算核对。ATT MTU、CoC 本端/对端 MTU 和 MPS 来自同一证据目录的 UART 记录；Mac 的 CoC 对端 MTU 在 1M、2M 下确实不同，并非誊写错误。
- ATT MTU 已再次复核：调试 UART 中 WCH 发出 Exchange MTU Request `02 00 01`，Mac 返回 Exchange MTU Response `03 00 01`，均为小端 `0x0100 = 256`；正式证据 UART 的 21 次 MTU 事件也全部为 256。Mac 客户端在上行前检查 `.withoutResponse` 可写长度至少为 244 B，随后 244 B 数据帧均通过固件的定长、序号和 pattern 校验。因此 **185 不是这组 WCH 历史连接的 ATT MTU**；该结论也不排除其他连接或写入类型出现不同值。
- 这些成绩代表上述 Mac、固件、协议栈和连接参数下的应用层饱和吞吐，不代表芯片理论射频峰值。不同测试时长、连接参数或测量口径的结果不得直接混入本表求均值。


## N32 本次测量来源与限制

- 固定源码：[2beed5f](https://github.com/kaidegit/n32wb452-nimble/commit/2beed5f1878829e221b27a81bf24ac704f769032)；[详细报告](https://github.com/kaidegit/n32wb452-nimble/blob/2beed5f1878829e221b27a81bf24ac704f769032/docs/gatt-perf-2026-11-02.md)；[12 项机器可读结果、固件/客户端哈希](https://github.com/kaidegit/n32wb452-nimble/blob/2beed5f1878829e221b27a81bf24ac704f769032/docs/gatt-perf-2026-11-02-results.json)。NimBLE 源码、控制器下载数组及 blectl 运行时 JSON 均可在该提交定位。
- 相同 Mac mini、同一板卡分别运行两种模式；同一个 CoreBluetooth central、同一 GATT bulk 协议；Mac14,12（arm64），macOS 15.7.5（24G624）。每模式上行和下行各 3 次独立连接、每次 30 秒，交错执行。MCU 上行用 TIM2 1 MHz 硬件时钟，bridge 上行用桌面 CLOCK_MONOTONIC，下行用 Mac 单调时钟，均包含尾部排空。传输期间不访问 probe/RTT/GDB，关闭逐包 DEBUG/INFO 打印。
- 12/12 项接收端字节数、序号及 pattern 校验通过，累计校验 **6,895,798 B**；每项都有主机或生产者背压，DLE 实际251、CI实际15ms。均值直接从每项 `payload_bytes_per_second / 1000` 计算，没有先对 kbps 舍入。

| 模式 | 方向 | 第一次 kB/s | 第二次 kB/s | 第三次 kB/s | 三次均值 kB/s |
| --- | --- | ---: | ---: | ---: | ---: |
| nimble | 上行 | 27.84874 | 27.88698 | 27.79703 | **27.84** |
| nimble | 下行 | 9.22117 | 9.97443 | 9.97425 | **9.72** |
| bridge | 上行 | 28.49963 | 28.69347 | 28.70660 | **28.63** |
| bridge | 下行 | 9.44366 | 9.98197 | 10.01527 | **9.81** |

- NZ8801 的 LE PHY 读写命令返回 HCI `0x01 Unknown Command`。工具前后均记录该返回及默认 LE 1M 降级，**不是成功读取出的 PHY 实测值**；来源中的 `reference_standard_met=false` 如实保留。若将来允许此类结果正式入库，需先明确 PHY 的替代核验依据，不能把能力适配解释为略过 PHY 证据。
- 本次只测 GATT，没有 L2CAP CoC 成绩；未测通道不填0。控制器射频错误/重传计数不可获取，应用层零校验错误不代表空口无重传。两种模式下行均约9.7～9.8 kB/s，仍低于上行；现有证据不足以判定为控制器射频极限。
