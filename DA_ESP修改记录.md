ARM / ESP SunSpec DA 协调接口 V1
日期：2026-09-03
状态：ARM侧实现及本机测试已完成；ESP侧Type分配、解析和实际RS485行为待对齐。

一、分工和本次改动

Model 1 DA仍位于寄存器40068(0x9C84)，L=66，Pad=40069，布局不变。
合法DA为1..247，支持FC06和FC10；FC10改DA时必须单独写一个寄存器。
ARM校验写入并建立待处理事务。ESP是RS485地址筛选和掉电保存的唯一所有者。
ARM只在ESP成功确认后更新DA映像及自身站址过滤；普通RTU请求不会覆盖DA。
DSP不参与DA改址。ARM不重复保存DA到Flash，启动时由ESP明确同步实际地址。

必须双端配套：ESP未完成SYNC前，ARM仍允许默认地址1的普通读写，
但合法DA写请求返回异常06(Busy)，不再返回虚假的改址成功。
地址0只是广播目的地址，不是允许写入DA的值。非法值仍返回异常03；广播不回RTU。

二、承载方式（需要ESP确认）

1. 保留TLV Type=0x1060：Value为完整原始RTU帧(含Modbus CRC)。
   空闲仍为Length=2、Value=00 00。不能在RTU Value前后插入新字段。
2. 新增TLV Type=0x1061（暂定，ARM当前属性表无冲突，仍须核对ESP及平台分配）。
   双向Value固定12字节。该消息仅在ARM/ESP内部使用，不转发到RS485/云端控制口。
3. ESP写控制：沿用AA55协议FRAME_TYPE_WRITE=0x02，一帧只带一个0x1061 TLV。
   TLV为10 61 00 0C + 下文12字节Value，外层头/流水/CRC沿用现有协议。
4. ESP读状态：FRAME_TYPE_READ=0x01，读请求属性部分为10 61 00 01。
   注意读请求的0001表示属性个数，不是12字节Value长度。
   ARM写控制不另发即时写回包；ESP用随后属性查询确认状态。
5. 0x1061追加在上报表末尾，原属性顺序不变。推荐显式轮询此Type，
   或调整全量查询个数；本配置164项，上报最大125寄存器RTU读应答时共1360字节，
   小于现有1536字节发送缓冲区（已用实际协议/上报表做本机验证）。
6. Session和Sequence按大端传输，禁止依赖C结构体内存布局。
   12字节Value在现有TLV编解码中整体透传，不走短整数字节翻转分支。

三、ARM -> ESP 状态Value（12字节）

偏移    长度  含义
0       1     Version，固定01
1       1     State：00=UNSYNCED；01=READY；02=PENDING
2..3    2     Session，会话号，由ESP在同步时指定，非0
4..5    2     Sequence，ARM每接受一笔DA写递增，跳过0；同步后为0
6       1     OldDA，本次改址前的地址
7       1     NewDA，本次请求的新地址
8       1     RequestDA，原RTU请求首字节；00表示广播
9       1     Function，06或10；同步后的初始值为00
10      1     ActiveDA，ARM已经确认生效的地址
11      1     Result：00=无结果/等待；01=已应用；02=应用失败

READY后的OldDA/NewDA/Sequence/Result保留最近一次事务，用于核对ACK是否被ARM接收。
PENDING期间ActiveDA及寄存器40068仍为旧地址，直到收到匹配的APPLIED确认。
重复读取不消费状态。等待确认期间不接收新的0x1060事务，防止覆盖待处理应答。

四、ESP -> ARM 控制Value（12字节）

偏移0为Version=01；偏移1改为Command：01=SYNC，02=APPLIED，03=FAILED。
偏移2..9与上述字段相同；偏移10为ESP实际生效地址；偏移11必须为00。

SYNC（启动、重连或不确定状态恢复）：
  Session=ESP新生成的非0会话；Sequence=0；OldDA=NewDA=ActiveDA=实际地址；
  RequestDA=0；Function=0；Result=0。
  ESP应先暂停RTU转发、清空旧内部队列、确定NVS和当前筛选地址，再发送SYNC。
  ARM收到新会话后清除旧待处理事务和RTU缓存，并更新DA及地址筛选。
  同会话SYNC重传无副作用，不会取消正在进行的改址，也不能用它改地址。
  查询看到READY、匹配Session、Sequence=0和正确ActiveDA后再开放RTU转发。

APPLIED：
  回传PENDING中的Session/Sequence/OldDA/NewDA/RequestDA/Function；
  偏移10填NewDA，偏移11填0。ARM匹配后更新DA，READY，Result=01。

FAILED：
  回传相同事务字段；偏移10填OldDA，偏移11填0。
  仅在ESP确定没有切换、且旧地址仍有效时发送；ARM保留旧DA，READY，Result=02。

错误版本、长度、字段、过期会话/流水及重复ACK均不改变状态。
重复ACK也不会清掉后来普通RTU请求的缓存应答。
错误控制命令不单独产生错误TLV：ESP通过状态未改变判断未被接受。

五、单播改址时序（例：1 -> 25）

1. 主站向地址1写40068=25。ESP保留完整原请求，通过0x1060转发ARM。
2. ARM校验后建立PENDING事件，准备旧地址1的正常FC06/FC10应答。
   ESP应先读取0x1061，或一次读回0x1060/0x1061并完整解析后再决定如何发送485。
3. ESP校验会话/流水/原请求匹配，保存新DA到NVS并确认成功。
   这时继续使用旧地址，暂缓转发新RTU请求。
4. ESP将ARM提供的正常应答按旧地址发到RS485，等待UART发送完成(TX complete)，
   然后切换自身RS485地址筛选到NewDA。
5. ESP发送APPLIED，轮询直到ARM为READY、同会话/流水、ActiveDA=25、Result=01。
   确认前收到的新地址请求应暂存，确认后再转给ARM，不能提前转发后直接丢弃。
6. 后续地址25可读DA=25；旧地址1不再响应。

DA事务的0x1060正常应答与0x1061事件都会保留到匹配确认，以允许轮询丢包重读。
ESP必须按(Session, Sequence)去重：重复收到候选应答时不能重复发485、重复保存或切址。
普通非DA请求的0x1060应答仍只在一次查询中返回。

若NVS保存失败：不要把候选正常应答发到RS485。ESP在旧地址回复原功能码的
异常04(Server Device Failure，重新计算RTU CRC)，发送FAILED并确认ARM回到READY。
若已经切到新地址，仅ACK丢失，继续重发同一APPLIED，不能再发FAILED或自行回旧地址。

六、广播改址时序

ESP必须将合法地址0的FC06/FC10完整帧转发ARM，不能按"非本站地址"提前丢弃。
ARM校验成功后仍产生PENDING，RequestDA=0，但0x1060保持00 00，不产生RTU应答。
ESP保存NVS、切换筛选地址、发送APPLIED、确认READY；整个过程RS485始终不回复。
失败时保持旧地址并发内部FAILED；即使错误也不能向广播回复Modbus异常帧。
同值写入（例如1->1）仍有事件和确认，ESP可避免不必要的NVS写操作。

七、重试、重启和并发约定

建议沿用200ms轮询；ACK发出后1s仍未确认则重发相同ACK。
建议连续3次无确认后进入通讯故障/重连处理（时限与重试次数需ESP侧确认）。
ARM不自动超时回滚：ACK丢失不能证明ESP尚未切址。PENDING会一直保留到ACK或新会话SYNC。
ESP重连使用新的非0Session并清空旧队列；会话号不得和仍可能在途的旧消息重用。
Sequence达到65535后建议在READY时换新Session同步，避免长期运行流水重用。
ARM重启后状态为UNSYNCED；ESP从其持久化且实际启用的DA重新SYNC。
一次只允许一笔RTU请求在途；DA的0x1061确认命令不受PENDING下0x1060暂停影响。
广播应在单台设备或明确隔离的测试总线上执行，避免多台一起改成相同地址。
FC10含DA和其他寄存器的混合写请求整体拒绝(03)，必须将DA单独写1个寄存器。

八、可以直接对照的Value示例（仅内部12字节Value，不含AA55外层和CRC）

ESP同步当前DA=1，Session=0x1234：
  01 01 12 34 00 00 01 01 00 00 01 00
ARM收到地址1的FC06写DA=25，Sequence=1，返回PENDING：
  01 02 12 34 00 01 01 19 01 06 01 00
ESP保存、发完旧地址应答并切址后回APPLIED：
  01 02 12 34 00 01 01 19 01 06 19 00
ARM返回READY，ActiveDA=25，Result=APPLIED：
  01 01 12 34 00 01 01 19 01 06 19 01
下一笔广播FC06从25改到2，Sequence=2，ARM返回PENDING：
  01 02 12 34 00 02 19 02 00 06 19 00
对应APPLIED：
  01 02 12 34 00 02 19 02 00 06 02 00

九、需要ESP同事确认的事项

1. 0x1061 Type是否可用，V1的12字节定义和0x01/0x02外层读写帧是否接受。
2. 能否在完成SYNC后才开放RTU；新上报属性是否加入轮询。
3. 能否以NVS为唯一地址持久化来源，支持1..247并在重启后同步ARM。
4. 单播旧地址应答TX完成后切址、广播不回包、待确认期间暂存下一帧的时序。
5. 以Session/Sequence去重、重复轮询应答不重复发送、ACK丢失重试/重连的处理。
6. NVS失败时抑制正常应答并回异常04；失败确认和已经切址后的处理区别。
7. FC06和FC10单寄存器写40068均支持；广播帧原始目的地址0不能被改成本站地址。

十、验证边界

Tests/da/run_tests.ps1编译并运行生产Modbus/CRC及ESP协议、收发、上报队列源码。
测试替换了MCU类型、寄存器数据源和DSP/OTA等硬件调用；它不代表实机ESP/NVS/RS485通过。
覆盖：合法/非法DA、单播和广播06/10、旧新地址筛选、确认去重、失败/重启恢复、
流水回绕、普通模型写入、事件及DA应答重读、完整164项上报容量。
实际仍需验证：RTU-4/RTU-5、新旧地址读写、掉电保持、NVS失败与内部通讯丢包。
本次不处理RTU-3残帧问题和LabTest端口未打开问题，也未修改PICS表格。

参考：Modbus Application Protocol V1.1b3，异常码章节：
https://www.modbus.org/file/secure/modbusprotocolspecification.pdf
以上0x1061、状态机、重试时限为本项目接口约定，不是SunSpec新增寄存器或标准TLV。
