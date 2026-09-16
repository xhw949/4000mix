**改动方案**

**波兰 SunSpec/Modbus 认证改动方案**

原方案成文日期：2026-07-30；2026-09-10 补写 0731 及 0909 的版本改动记录。

本文保留当前方案、代码状态、已验证指令和待办事项。

历史讨论和已经关闭的分支不再单独展开。

DSP 名称、地址、单位以已确认的 DSP 通信表为准。

**0\. 版本改动补记**

**0.1 2026-07-31：SunSpec/Modbus 主链路落地（补记）**

- 记录范围为 aodili_syn2.1.6-polan_add40000_26.7.31 相对

aodili_syn2.1.6 的 ARM 源码。下列"已实现"均为源码或留存构建日志结论，

不等同于 ESP、DSP 或整机认证已验证。

- 新增 MB_SUNSPEC_BASE=40000u 的连续 SunSpec 地图：

SunS → 1 → 701 → 702 → 703 → 704 → 714 → 715 → End。地图使用大端

backing store，初始化时回填 model ID/L；范围为十进制

40000～40418（0x9C40～0x9DE2）。

- 新增 MbSlave_HandleRequest() 从机引擎，支持 0x03、0x04、

0x06、0x10，并执行 CRC、地址/数量、RW 权限、枚举值、32 位字段完整写入

和跨 model 访问校验；错误时构造标准 Modbus 异常应答。

- ARM 工程新增 0x1060 RTU 隧道：value 承载完整 Modbus RTU 帧（含 CRC）；

应答在下一次周期读中取走一次，空闲值为 L=2、V=00 00。旧的逐项并网

参数/控制 TLV 队列从该 ARM 路径移除。

- 新增 Model 1/701/714 读映像刷新，以及 Model 702/703/704/715 写命中后的

ARM→DSP/ARM 门控分发。Model 703 ES=0 在 PMS 末端强制关机；

Model 715 LocRemCtl=LOCAL 停止周期控制帧。最终 DSP/整机动作仍需台架验证。

- 新增 flash_param_poland_t A/B Flash 参数区和上电恢复路径；站地址的 ARM

Flash 字段仅为兼容保留，0731 设计中 DA 的持久化真源仍属于 ESP。

- ESP_SEND_MAX_LEN 和 USART2_TXBUFF_SIZE 由 1284 扩至 1536，供包含最大

256 字节 RTU value 的上行帧使用。留存的 build_poland_0x4000.log 显示

3600Pro 为 0 Error(s), 1 Warning(s)；这不是本次构建或硬件验证。

**0.2 2026-09-09：Model 702 四个 PF 点声明与固件对齐**

- 0909 第一次报告因测试电脑的 PortNotOpenError 未取得有效寄存器应答，

不能作为固件失败证据。重新插拔后生成的

2026-09-09_12-45-36-293_zendure_0909.xlsx 为本节使用的有效测试记录：

Model 703 通过，Model 702 仍有 4 个点读回 0，低于工具要求的

800～1000，因此为唯一失败项。

- 失败点为 Model 702 的 WOvrExtPF、WUndExtPF、PFOvrExt、PFUndExt，

对应 model 偏移 28、30、42、43，绝对寄存器

40253、40255、40267、40268。问题是 PICS 当时仍将其声明为

supported，而产品没有对应实现；这不是已确认的 DSP 参数故障。

- Ac_Protocol/ac_protocol.c 将 Ac_MbRwM702 改为

{{26,2},{29,1},{31,11},{44,1}}，从可写图中排除这四个偏移，但保留 Model 702

的 52 寄存器长度和固定寄存器布局。

- Modul_HAL/Modbus/modul_modbus.c 在初始化 Model 702 映像时将上述 4 个

uint16 槽位写为 0xFFFF（SunSpec 未实现值）。因此 0x06 单写命中这些

只读槽位会返回非法地址；0x10 多写沿用现有的只读点静默忽略策略。

- 配套 PICS 修正版只修改工作表 702 的 D52、D54、D66、D67：

supported → unimplemented，文件为

```
outputs/01a08448-7526-7c53-a33e-faaf7b814a11/
```

SunSpec_Information_Model_Reference_20211209_0909_fixed.xlsx。原 PICS 未覆盖。

- 上述源代码与 PICS 均已作静态核对；未重新编译、未烧录，修正版 PICS 也尚未

随新 ARM 固件复测。RTU-3 虽在报告中标记 Pass，但日志存在测试工具

computeCRC 异常，不能据此认定残缺报文测试已经完整执行。

**1\. 当前结论**

- ESP、ARM、DSP 的 Modbus 认证主链路已经打通：

485主站 → ESP → TLV 0x1060 → ARM → CAN → DSP。

- DSP 测量数据的回传链路已经打通：

DSP → CAN → ARM → TLV 0x1060 → ESP → 485主站。

- ARM 的 Modbus 从机、寄存器地图、CRC、读写和异常处理，

已通过 485 实机联调。

- SunSpec基址采用十进制40000，线上地址为 0x9C40。
- NPrt=1 时，当前地图最后地址为 0x9DE2。
- ESP 透传内存池调整为 300B 后，最大标准 RTU 读应答也已验证。
- 原功能链已通过实机联调；新基址仍需ESP同步后完整回归。
- 当前剩余工作主要是 ESP 改址时序复测、DSP 实际功能验证，

新基址回归以及认证工具的最终扫描。

**2\. 已确定的通信架构**

**2.1 ESP 与 ARM**

- ESP 识别认证 Modbus 请求，并校验站地址、功能码和 CRC。
- ESP 不解析 SunSpec 的 model 或 Name。
- 一条完整 Modbus RTU 请求放入一个 TLV 的 value。
- TLV Type 已确认为 0x1060。
- 请求和应答复用同一个 TLV Type。
- 有应答时，value 包含完整 RTU 帧，包括站地址、功能码、

数据和 CRC。

- ARM 负责完整的 Modbus 从机解析和应答生成。
- ARM 将应答缓存到 0x1060 上报项，供 ESP 周期读回。
- 应答被 ESP 取走并发送一次后，ARM 恢复空闲占位值。
- 无待回应答时，0x1060 使用 L=2、V=00 00。
- 有效 RTU 应答最短为5字节，使用 L>=5 判断待回应答。
- ESP 必须识别并忽略空闲占位值，不能将 00 00 发到485。

**2.2 ARM 的 Modbus 从机**

- SunSpec 数据按一个连续的平铺寄存器地图访问。
- ARM 不按 model 分发请求，按地址范围定位数据。
- 支持功能码：0x03、0x04、0x06、0x10。
- 支持跨 model 读取和跨 model 写入。
- 32 位字段要求一次写入完整的两个寄存器。
- 写入时按字段权限判断，不能写只读字段。
- RTU CRC 使用标准 Modbus 顺序：低字节在前、高字节在后。
- MB_TUNNEL_FRAME_HAS_CRC 保持为 1。

**2.3 地址、发现流程和容量**

- 规范中的地址 40000 是十进制，不是十六进制 0x4000。
- MB_SUNSPEC_BASE = 40000u，线上地址字段为 0x9C40。
- SunSpec使用完整16位零基地址，不采用传统4xxxx编号换算。
- 发现工具依次读取十进制地址0、40000和50000，

查找两个寄存器组成的 SunS 标志。

- 本机在十进制40000放置标志，即 0x5375、0x6E53。
- 找到标志后，依次读取每个model的ID和L。
- 下一个model地址等于"L寄存器下一地址加L"。
- 扫描遇到 ID=0xFFFF、L=0 时结束。
- 当前地图范围为十进制40000～40418，

即 0x9C40～0x9DE2。

- ESP必须把 0x9C40 附近的认证请求透传给ARM。
- 旧的 0x2000 和十六进制 0x4000 均不再使用。
- 当前 Model 714 的 NPrt=1，仍需产品最终确认。
- 标准 Modbus 单次最大读取数量为 125 个寄存器。
- 最大读取应答为 255 字节，符合 RTU ADU 上限。
- ESP 透传内存池为 300B，已完成 255 字节应答测试。
- ESP 全量读查询上限为 175 或 176，ARM 实际返回 163 项。
- 不补充不存在的空 TLV；判断标准是应答中存在 0x1060。

**2.4 ESP 编译开关**

ESP 侧需要确认以下配置：

```
RS485_PROTO_MODBUS
MODBUS_SLAVE_ENABLE
MODBUS_PASSTHRU_ENABLE
MODBUS_CERT_PASSTHRU
```

ARM 侧无法从代码判断 ESP 的编译配置，

以 ESP 的 sdkconfig 或运行日志为准。

**3\. 已完成的代码改动**

**3.1 SunSpec 寄存器地图**

- 地图内联在 Ac_Protocol/ac_protocol.h/.c。
- 使用 Ac_MbModelMap 描述 model 的起始地址和长度。
- 使用 Ac_u8MbRegIsWritable 判断寄存器是否可写。
- 采用大端 SunSpec 映像作为 backing store。
- 已接入 model：

1、701、702、703、704、714、715。

- 只读映像由 Ac_vidPolandRdModelRefresh() 刷新。
- Model 1 的 Mn、Md、Opt、Vr、SN 已初始化。
- Model 701 的状态和主要交流测量已接入。
- Model 714 的直流功率和直流电压已接入。

**3.2 Modbus 从机引擎**

- 文件：Modul_HAL/Modbus/modul_modbus.c。
- MbSlave_HandleRequest() 已支持四种功能码。
- 支持 701 多帧、跨 model 访问和最大读取限制。
- 写入命中字段后，通过回调下发 ARM→DSP 数据。
- 只读空洞、非法枚举、部分 32 位写入均被拒绝。
- CRC 统一由 MbSlave_AppendCrc() 处理。
- devPolandModbusTunnelSetIdle() 统一设置

L=2、V=00 00。

- 上电初始化、无应答回退及应答消费后均调用该函数。

**3.3 ARM→DSP 写入**

- 直接参数使用 DSP 功能码 0x0C。
- 无功和 PF 使用周期控制帧功能码 0x04。
- 周期控制帧使用 StartAddr=6、数量 12，

连续覆盖 DSP reg6～reg17。

- 部分写字段使用 Ac_u8PolandFieldWritten，

只下发实际写入过的字段。

- ESP 认证优先门控已修复，避免被 PMS 周期覆盖。

**3.4 Flash 和启动恢复**

- 已实现波兰参数的写入、校验、Flash 保存和恢复。
- 需要持久化的字段在上电后重新进入 DSP 下发链路。
- ESDlyTms=60 已完成掉电重启恢复验证。

**3.5 测试代码和工程**

- 已删除 ARM 内部模拟注入和 Watch 自测入口。
- 正式联调入口为真实 485→ESP→ARM 链路。
- Keil 3600Pro 目标已构建通过：0 错误，1 个原有警告。
- ESP_SEND_MAX_LEN 和 USART2_TXBUFF_SIZE 已扩大到 1536。

**4\. ARM 与 DSP 映射**

**4.1 直接参数：DSP 功能码 0x0C**

- Model 702 VNom，model 偏移 38：

DSP u16RatedVolt，StartAddr=1，当前按 1V 换算。

- Model 702 WMax，model 偏移 26：

DSP u16RatedPower，StartAddr=3。

- Model 703 ESDlyTms，model 偏移 9～10：

DSP u16ConnectToNetTime，StartAddr=4，单位秒。

- Model 703 ESVHi，model 偏移 3：

DSP u16NetReConnectMaxVolt，StartAddr=10。

- Model 703 ESVLo，model 偏移 4：

DSP u16NetReConnectMinVolt，StartAddr=11。

- Model 703 ESHzHi，model 偏移 5～6：

DSP u16NetReConnectMaxFreq，StartAddr=12。

- Model 703 ESHzLo，model 偏移 7～8：

DSP u16NetReConnectMinFreq，StartAddr=13。

- Model 704 WRmp，model 偏移 49：

DSP u16NorPowGrad，StartAddr=14，ARM乘 60 换算。

- Model 704 AntiIslEna，model 偏移 52：

DSP u16IslandDetectionEnable，StartAddr=18。

- Model 704 VarSetEna：

使能时写 DSP StartAddr=17，值 0 表示固定无功。

- Model 704 PFWInjEna：

使能时写 DSP StartAddr=17，值 1 表示固定 PF。

**4.2 周期控制参数：DSP 功能码 0x04**

- Model 704 VarSet 对应 DSP reg6：

s16ReActivepower，单位 1Var/LSB。

- Model 704 PFWInj.PF 对应 DSP reg7：

s16ActivepowerFactor，单位 0.001/LSB。

- PFWInj.Ext 用于生成 PF 正负号。
- s16V_SF、s16PF_SF 等是 SunSpec 比例因子，

不是独立 DSP 寄存器。

- VarSetMod 当前只接受枚举 4"直接使用 Var"。
- VarSetPri 当前没有 DSP 对应项。
- 固定 PF 和固定无功不能同时使能。

**4.3 ARM 门控或最小实现**

- Model 703 ES 使用 Ac_u8PolandESPermit 做 ARM 门控，

当前不单独映射 DSP 参数。

- Model 715 LocRemCtl 使用 ARM 周期控制门控。
- Model 715 AlarmReset 写 1 后自清零，不触发 DSP 重启。
- Model 715 ControllerHb 当前只记录。
- Model 715 OpCtl 的 STOP/START 可映射到系统启动命令，

但可能被 PMS 或 App 覆盖。

- ENTER_STANDBY=2、EXIT_STANDBY=3 暂不测试。

**4.4 当前没有完整 DSP 目标的字段**

- Model 703：ESRndTms、ESRmpTms。
- Model 704：WMaxLimPct、VarRmp、WSet。
- Model 704：吸收 PF、回退 PF 和各回退时间。
- Model 715：待机进入/退出的最终含义。

这些字段即使 Modbus 写入成功，也不能仅凭 DSP 数值不变

判定 ARM 或 ESP 链路故障。

**5\. 485联调指令与结果**

**5.1 基础读取**

当前条件：站地址 1、115200、8N1、十六进制发送，

发送帧已经包含 CRC，串口工具不要重复追加 CRC。

本节帧已按新基址 0x9C40 重新计算CRC，

需要在ESP同步透传范围后重新执行。

**0x1060 空闲占位值**

ESP 全量查询且当前没有待回 RTU 帧时，ARM 应答中应包含：

```
Plain Text
10 60 00 02 00 00
```

含义为 Type=0x1060、L=2、V=00 00。

ESP 应正常解析整帧并忽略该占位值，不得发送到485。

该项需使用新 ARM 固件与 ESP 继续联调验证。

**读取 SunS 标识**

发送：

```
Plain Text
01 03 9C 40 00 01 AB 8E
```

应答：

```
Plain Text
01 03 02 53 75 45 53
```

其中寄存器值 0x5375 为 SunS 的前两个字符。

**使用0x03执行标准发现**

发送：

```
Plain Text
01 03 9C 40 00 02 EB 8F
```

应答：

```
Plain Text
01 03 04 53 75 6E 53 96 F0
```

**使用0x04读取完整SunS标识**

发送：

```
Plain Text
01 04 9C 40 00 02 5E 4F
```

应答：

```
Plain Text
01 04 04 53 75 6E 53 97 47
```

**最大合法读取数量**

发送：

```
Plain Text
01 03 9C 40 00 7D AA 6F
```

判定：应答总长 255 字节，开头为：

```
Plain Text
01 03 FA 53 75 6E 53
```

中间动态测量值可以变化，但长度和 CRC 必须正确。

**读取地图后半段**

发送：

```
Plain Text
01 03 9D 68 00 7B AB 99
```

判定：应答总长 251 字节，开头为：

```
Plain Text
01 03 F6 02 C0 00 41
```

数据区应包含：

```
Plain Text
model714头：02 CA 00 2B
model715头：02 CB 00 07
结束标志：FF FF 00 00
```

**读取地图最后一个合法地址**

发送：

```
Plain Text
01 03 9D E2 00 01 0B 90
```

应答：

```
Plain Text
01 03 02 00 00 B8 44
```

**5.2 Model 702 和 703 写入**

**读取额定功率和 WMax**

```
Plain Text
发送  01 03 9D 23 00 01 5A 6C
应答  01 03 02 0F A0 BD CC
```

```
Plain Text
发送  01 03 9D 3B 00 01 DA 6B
应答  01 03 02 0D AC BC A9
```

含义分别为 WMaxRtg=4000W、WMax=3500W。

**WMax 写入**

发送并预期原样回显：

```
Plain Text
01 06 9D 3B 0D AC D3 46
```

DSP 预期：功能码 0x0C、StartAddr=3、数据 3500。

**ESDlyTms 写入 60 秒**

读取：

```
Plain Text
01 03 9D 5E 00 02 8A 75
```

写入：

```
Plain Text
01 10 9D 5E 00 02 04 00 00 00 3C 83 98
```

写入应答：

```
Plain Text
01 10 9D 5E 00 02 0F B6
```

读回和掉电重启后应答：

```
Plain Text
01 03 04 00 00 00 3C FA 22
```

DSP 预期：功能码 0x0C、StartAddr=4、数据 60。

**ESRndTms 最小写入**

发送：

```
Plain Text
01 10 9D 60 00 02 04 00 00 00 00 01 11
```

应答：

```
Plain Text
01 10 9D 60 00 02 6E 7A
```

该字段当前不下发 DSP，也不保存 Flash。

**5.3 Model 704 参数和控制**

**读取 PF 参数区域**

发送：

```
Plain Text
01 03 9D 99 00 0C BA 4C
```

当前 PF_SF=-3。完成 PF 参数写入后，

PFWInjPF=0x03E8、PFWInjExt=0x0001。

**写入 PFWInjPF=1000**

发送并预期回显：

```
Plain Text
01 06 9D A3 03 E8 56 FA
```

读回：

```
Plain Text
01 03 9D A3 00 01 5B 84
应答  01 03 02 03 E8 B8 FA
```

Keil 预期：

```
Plain Text
Ac_s16PolandPFactorW = 1000
g_flash_poland.u32PFWInjPF = 1000
ValidMask bit10 = 1
```

**写入 PFWInjExt=1**

发送并预期回显：

```
Plain Text
01 06 9D A4 00 01 26 45
```

读回应为：

```
Plain Text
01 03 02 00 01 79 84
```

Keil 预期：g_flash_poland.u32PFWInjExt=1，

ValidMask bit11=1。

以上 PF 参数测试要求 PFWInjEna=0，

不会切换 DSP 控制模式。

**固定 PF 控制**

先关闭固定无功：

```
Plain Text
01 06 9D 8B 00 00 D6 4C
```

写入 PF=0.950：

```
Plain Text
01 06 9D A3 03 B6 D7 02
```

设置励磁方向并使能固定 PF：

```
Plain Text
01 06 9D A4 00 01 26 45
01 06 9D 6A 00 01 47 BA
```

DSP 预期：地址 17 为 1，控制 reg7 为 950，

Keil Ac_s16PolandPFactorW=950。

实际运行时可用下面的 485 读帧检查闭环结果：

```
Plain Text
01 03 9C 92 00 02 4B B6
```

两个寄存器依次为实际 Var 和 PF。

**固定无功控制**

先关闭固定 PF：

```
Plain Text
01 06 9D 6A 00 00 86 7A
```

设置 VarSetMod=4：

```
Plain Text
01 06 9D 8C 00 04 66 4E
```

写入 VarSet=+100Var：

```
Plain Text
01 10 9D 8E 00 02 04 00 00 00 64 8F 3E
```

使能固定无功：

```
Plain Text
01 06 9D 8B 00 01 17 8C
```

DSP 预期：地址 17 为 0，控制 reg6 为 +100，

Keil Ac_s16PolandVarSetW=100。

测试结束后先禁用，再将 VarSet 恢复为 0。

**5.4 待执行的 DSP 直接参数**

以下项目均通过 485 写入，再在 Keil 或 DSP 侧确认。

同一类 0x0C 参数已经用 WMax、ESDlyTms 做过代表性验证，

不需要在 DSP 上位机逐项寻找同名变量。

**比例因子前置检查**

发送：

```
Plain Text
01 03 9D 66 00 02 0B B8
```

当前预期两个比例因子均为 0：

```
Plain Text
01 03 04 00 00 00 00 FA 33
```

若比例因子变化，下面的 DSP 原始值按实际 SF 重算。

**Model 702 VNom**

写入 230V：

```
Plain Text
01 06 9D 47 00 E6 97 F9
```

DSP 预期：StartAddr=1，原始值 230，

对应 DSP 项"额定电压"。

**Model 703 重连电压和频率**

写入 ESVHi=110%：

```
Plain Text
01 06 9D 58 00 6E A6 59
```

DSP 预期：StartAddr=10，原始值 1100。

写入 ESVLo=90%：

```
Plain Text
01 06 9D 59 00 5A F6 4E
```

DSP 预期：StartAddr=11，原始值 900。

写入 ESHzHi=51Hz：

```
Plain Text
01 10 9D 5A 00 02 04 00 00 00 33 C2 6F
```

应答：

```
Plain Text
01 10 9D 5A 00 02 4E 77
```

DSP 预期：StartAddr=12，原始值 5100。

写入 ESHzLo=49Hz：

```
Plain Text
01 10 9D 5C 00 02 04 00 00 00 31 C3 84
```

应答：

```
Plain Text
01 10 9D 5C 00 02 AE 76
```

DSP 预期：StartAddr=13，原始值 4900。

**Model 704 WRmp**

写入 1%Max/s：

```
Plain Text
01 06 9D 99 00 01 B7 89
```

DSP 预期：StartAddr=14，原始值 60，

即 DSP 单位 60%Pn/min。

恢复为 0：

```
Plain Text
01 06 9D 99 00 00 76 49
```

实际功率可用下面的 485 读帧计算爬坡速率：

```
Plain Text
01 03 9C 90 00 01 AA 77
```

**Model 704 AntiIslEna**

仅在安全台架条件下写 1：

```
Plain Text
01 06 9D 9C 00 01 A7 88
```

DSP 预期：StartAddr=18，原始值 1。

恢复为 0：

```
Plain Text
01 06 9D 9C 00 00 66 48
```

**5.5 DSP 运行数据回读**

这些项目以 485 应答为主要判据，

只有数值异常时才用 Keil 对照 ARM 源变量。

**Model 701 状态**

发送：

```
Plain Text
01 03 9C 89 00 03 FA 71
```

读取 St/InvSt/ConnSt，应答数据为动态值。

**Model 701 主要交流测量**

发送：

```
Plain Text
01 03 9C 90 00 09 AB B1
```

依次读取 W、VA、Var、PF、A、LLV、LNV、Hz。

主要 ARM 来源：

- W：Ac_tstrRdReg.Info.s16ParallelActivePower。
- VA：Ac_tstrRdReg.Info.u16ParallelApparentPower。
- Var：Ac_tstrRdReg.Info.u16ParallelReActivepower。
- PF：Ac_tstrRdReg.Info.u16ParallelPowerFactor，除以 1000。
- A：Ac_tstrRdReg.Info.u16ParallelCur，除以 100。
- LNV：Ac_tstrRdReg.Info.u16GridVolt，除以 10。
- Hz：Ac_tstrRdReg.Info.u16GridFreq，除以 100。

**Model 714 直流总功率**

发送：

```
Plain Text
01 03 9D B1 00 01 FB 81
```

应答值应等于 Sys_globaldata.u16PVPowertotal，单位 1W。

**Model 714 端口数据**

发送：

```
Plain Text
01 03 9D C9 00 03 FA 59
```

依次读取 DCA、DCV、DCW。

当前 DCA 尚未接入，预期为 0；

DCV 来源为 Ac_tstrRdReg.Info.u16DCLinkVolt；

DCW 来源为 Sys_globaldata.u16PVPowertotal。

**5.6 ARM 门控和设备控制**

**Model 703 ES 门控**

禁止进入服务：

```
Plain Text
01 06 9D 57 00 00 17 B6
```

Keil 预期：Ac_u8PolandESPermit=0，

PMS 周期强制下发关机状态。

恢复许可：

```
Plain Text
01 06 9D 57 00 01 D6 76
```

写 1 只解除门控，不主动启动设备。

**Model 715 LocRemCtl**

进入 LOCAL：

```
Plain Text
01 06 9D DA 00 01 46 5D
```

Keil Ac_u8Poland715LocRemCtl=1，

ARM 停止发送周期控制帧。

恢复 REMOTE：

```
Plain Text
01 06 9D DA 00 00 87 9D
```

Keil 值恢复 0，周期控制帧恢复发送。

**Model 715 OpCtl**

STOP 和 START：

```
Plain Text
01 06 9D E0 00 00 A7 90
01 06 9D E0 00 01 66 50
```

Keil 可观察 Sys_Set_Info.u8SysStartUpCmd。

该值可能被 PMS/App 覆盖，必须确认最终设备状态。

ENTER_STANDBY=2、EXIT_STANDBY=3 暂不测试。

**6\. 异常、边界和保护测试**

**6.1 只读字段写保护**

发送：

```
Plain Text
01 06 9C 40 12 34 AB 39
```

预期：

```
Plain Text
01 86 02 C3 A1
```

异常码 0x02 为非法数据地址，SunS 标识保持不变。

**6.2 地图越界**

发送：

```
Plain Text
01 03 9D E3 00 01 5A 50
```

预期：

```
Plain Text
01 83 02 C0 F1
```

**6.3 读取数量超过 125**

发送：

```
Plain Text
01 03 9C 40 00 7E EA 6E
```

预期：

```
Plain Text
01 83 03 01 31
```

**6.4 CRC 错误**

发送错误 CRC：

```
Plain Text
01 03 9C 40 00 01 AB 8F
```

预期：485 端保持静默，不增加有效接收字节数。

**6.5 32 位字段只写一半**

发送：

```
Plain Text
01 10 9D 5E 00 01 02 12 34 EA 90
```

预期：

```
Plain Text
01 90 03 0C 01
```

ESDlyTms 不应被修改。

**6.6 非法枚举值**

发送：

```
Plain Text
01 06 9D 57 00 02 96 77
```

预期：

```
Plain Text
01 86 03 02 61
```

ES 值、Flash 和门控状态均保持原值。

**6.7 数量和 byteCount 不一致**

发送：

```
Plain Text
01 10 9D 60 00 02 02 12 34 EE 0A
```

预期：

```
Plain Text
01 90 03 0C 01
```

ESRndTms 保持原值。

**7\. 站地址、基址和旧帧说明**

**7.1 ESP 正式改址流程**

当前 ESP 站地址为 1 时，使用旧地址完成以下步骤。

暂存新地址 2：

```
Plain Text
01 06 1F 00 00 02 0F DF
```

预期旧地址 1 原样回显，实机此步骤已通过。

提交生效：

```
Plain Text
01 06 1F 04 00 01 0E 1F
```

协议预期旧地址 1 原样回显。

当前 ESP 固件曾出现提交帧无回显，需 ESP 修复后复测。

切换后用新地址 2 读回：

```
Plain Text
02 03 1F 00 00 01 83 ED
```

预期：

```
Plain Text
02 03 02 00 02 7D 85
```

**7.2 Model 1 的 DA**

- model1.DA 只应作为当前 ESP 站地址的镜像。
- 直接写 0x9C84 虽可收到 ARM 回显，

但不会完成 ESP 的提交和持久化。

- 旧测试帧仅用于复现历史问题：

```
Plain Text
01 06 9C 84 00 02 66 72
```

- ARM 不维护第二份站地址过滤逻辑。
- ESP 改址后，ARM按新站地址收到请求即可继续应答。
- 源码中的 DA 写权限需要恢复为只读，待最终确认。

**7.3 旧基址**

以下旧帧不再用于当前联调：

```
Plain Text
01 03 20 00 00 01 8F CA
01 03 40 00 00 01 91 CA
```

第一帧使用旧的 0x2000，第二帧误把十进制40000

写成了十六进制 0x4000。

当前ARM使用十进制40000，即线上地址 0x9C40。

**8\. Keil 观察原则**

- 0x9D5E 等是 Modbus 逻辑地址，

不是 MCU 物理 RAM 地址，不能直接填入 Memory 窗口。

- 写入 ESDlyTms=60 后，观察真实 RAM 地址的 4 字节：

```
Plain Text
00 00 00 3C
```

- 结构体成员显示 0x3C000000，

是大端 SunSpec 映像在小端 CPU 上的正常显示。

- Flash 观察：

```
Plain Text
g_flash_poland.u32ESDlyTms = 60
ValidMask bit8 = 1
```

- 直接参数至少用 WMax、ESDlyTms 各验证一次 DSP 接收。
- 周期控制至少验证一次固定 PF和一次固定无功。
- 只读数据优先以 485 应答为判据，

不需要在 DSP 上位机逐项寻找同名变量。

- 只有数值异常时，才使用 Keil 对照 ARM 源变量。
- 爬坡、孤岛、启停等功能必须确认最终运行效果。

**9\. 当前状态和后续计划**

**9.1 旧基址阶段已通过**

- ESP↔ARM 的 0x1060 请求、周期读回和应答回传。
- 0x1060 空闲值已改为 L=2、V=00 00。
- 0x03/0x04/0x06/0x10 正常读写。
- SunS、model 头、结束标志、地图边界和大容量读取。
- CRC、只读、越界、数量限制和非法写入保护。
- WMax=3500 下发 DSP 地址 3。
- ESDlyTms=60 下发 DSP 地址 4并掉电恢复。
- PF 参数保存、读回和固定 PF 数据链路。
- Model 701/714 主要测量数据回读。

以上结果证明功能链正确，但地址已整体平移。

ESP更新后必须使用第5节新帧重新回归。

本次基址调整已完成：

- ARM宏改为十进制 40000u，地图平移到 0x9C40～0x9DE2。
- 第5节地址和CRC已全部重新计算。
- Keil目标 3600Pro 编译为0错误、1个既有未使用函数警告。

**9.2 待处理**

- 等 ESP 修复 0x1F04 提交帧无回显问题后复测改址。
- 确认 ESP 忽略 0x1060 的 L=2、V=00 00，

且不会把占位值发送到485。

- ESP将认证透传起始地址从错误的 0x4000

改为十进制40000，即 0x9C40。

- 当前 Doc/esp.txt 仍记录旧路由范围，

ESP协议和固件需要同步更新。

- 使用第5节新地址帧完成全部读写及异常回归。
- 将 model 1 DA 写权限恢复为只读并重新编译验证。
- 最终确认 Model 714 的 NPrt。
- 确认 Model 715 待机命令 2/3 的正式语义。
- 验证 ESVHi、ESVLo、ESHzHi、ESHzLo、WRmp、

AntiIslEna 的 DSP 参数和实际效果。

- 完成固定 PF、固定无功和 model 701/714 回读闭环测试。
- 对未接入 DSP 的字段补充产品决定或保持明确的未实现状态。

**9.3 最终通过判据**

读链路：

1. 485 收到格式、长度和 CRC 正确的 Modbus 应答。
2. 工程值符合设备状态和字段定标。
3. 仅在数值异常时，对照 Keil 源变量。

写链路：

1. 485 收到正确的 Modbus 应答。
2. ARM下发的 DSP 地址、数据或控制值正确。
3. 每种 DSP 发送机制至少有一个代表项验证接收。
4. 爬坡、孤岛、启停等功能确认最终运行效果。
5. 需要持久化的字段掉电后仍能恢复并重新下发。
6. 测试结束后恢复安全控制模式和参数。

**9.4 认证前检查**

- ESP 将十进制40000，即 0x9C40，路由到认证透传。
- ESP 启用 Modbus 认证透传相关配置。
- ESP 与 ARM 使用同一个 TLV Type 0x1060。
- 认证工具关闭自动追加 CRC，配置正确站地址和基址。
- 确认 NPrt=1 及最终 Model 714 地图长度。
- 确认 Model 1 DA 的最终读写属性。
- 使用认证工具执行完整读、写、异常和边界扫描。
