[id:62904e53] [2026-09-29] [summary:MCU 聚焦 AF 卡死：v1 与工作区副本源码不同（v1 独有 CMAR 重定向 bug 与 flashdefendTask）；根因是同步阻塞 + 63 字节 RX 缓冲溢出/帧错位 + 位置模型被写成 0xFFFF。]
镜头伺服（STM32F103，F124Z/BOARD3/MASTERMODE，编译工程 E:\MCU_Project\plastic-lens-servo_v1，工作区分析目录 V1troubleshooting...）聚焦 AF 卡死问题结论：
1) 分析必须以 plastic-lens-servo_v1 为准，工作区副本与它不一致（v1 独有 flashdefendTask、USART_Callback 内 CMAR=USART_DMA_BUFFER2 重定向且永不还原、TIM4/time4_count、ymodem 两参数接口）。
2) 根因骨架：单线程超级循环里运动命令同步阻塞（GroupGoto 最长 255 拍 = AF QUICK 2.1s / 普通 8.5s；GotoAutoPos 第一段 while 最长 511 拍 ≈17s 且完全不服务串口），而 RX 环形缓冲只有 63 字节（115200 下 9 帧 = 5.5ms 余量）→ 溢出丢字节 → 帧错位 → 执行不存在的命令/应答张冠李戴；叠加 4 字节 send_back 与 7 字节 send_pelcod 混流及各主动上报帧（Feed_zoomBack 的 0x0083、404 状态帧、A5A5 flash 帧）。
3) 位置变量（zoom.site/focs.site）变 0xFFFF/0xFFF5 的来源：GetCurve 系列不校验索引与 curve_address、GroupAim/AdjustPosition 不夹紧、LensSiteUpload 限位依赖方向位、ISR 与 SetGroupParam 撕裂、safe_u16_add 溢出返回 0xFFFF。
4) 零成本判别：卡死瞬间单发 0x0081 看回复位置是否 0xFFFF；RTT 里 move_step_* 的 time 是否 2100/8500ms；End auto position loop 是否接近 511；发 0xF091 payload=1 让 MCU ef_print_env() 对账 ENV；检查 YMODEM 日志 CMAR 是否等于 rxbuf。
详见 AF_HANG_ROOT_CAUSE_ANALYSIS.md。
§
[id:0c10ac49] [2026-09-30] [summary:F124Z focus 温度补偿落点：f124z.c 新增 GetFocsCurve() 收口三处曲线读取，dev 用定点 Q20、ΔT 在 temperTask(1Hz) 缓存；temperTask 当前被注释、curve_addr_se]
[branch:develop] F124Z focus 温度补偿（数值补偿路线）落点约定（plastic-lens-servo-v2-bak-pitch-improve）：
1) 补偿只加在 HARDWARE/src/f124z.c 新增的曲线取值收口函数 GetFocsCurve(index) 里（raw 曲线 → +dev → 按 [MIN_FOCS,MAX_FOCS] 限幅），替换三处读取：GetCurveAim(f124z.c:61-63)、GetCurvePos(70-72)、GetTrack(138-140)；focspos 与 focsaim 必须同源，否则变焦结束会跳一下。
2) dev = df/dz*ΔT*k*(zoom-z0) - ΔT*k1*(focus-f0)，focus 用原始曲线值 raw（不要用补偿后的值）；df/dz 用 ±5 点差分/10，与标定表的 a 口径一致。
3) 定点实现：ΔT 与系数在 ntc.c temperature_offset()（由 temperTask 约 1Hz 调用）里算成 Q20 系数 focs_temp_kz/focs_temp_kf 存全局，读取路径只做整数乘法+移位，不碰 ADC。
4) 现有障碍：sysTick 里 temperTask() 被注释（sys_task.c:567）→ temperature_offset() 从未执行；temperature_offset() 里的 curve_addr_set() 是按温度平移 flash 地址的老方案，必须废弃（norflash.c:117），curve_read_focs 里 `data += (curve_offset>>1)` 只改形参属死代码；SetScope（f124z.c:89-97）的 focs.max 需要把 dev 计入，否则 LensSiteUpload 会在 site>=max 时刹车导致不到位（GroupAim 不夹紧）。
5) 待确认：ΔT 的基准温度（450=45.0℃ / 250=25.0℃ / 0）与符号——必须复现标定表里 dT 那一列，符号错了会把偏差反向放大一倍；在曲线标定温度下 ΔT 应≈0。
§
[id:c45044cb] [2026-10-08] [summary:F124Z focus 温度补偿修订（替代 2026-09-30 那条）：1) 系数用 PC 端预计算 Q24 整数 #define，无浮点字面量；2) temperature_offset 只缓存 int16 ΔT，dev 在 GetFo]
[branch:develop] F124Z focus 温度补偿方案 A 修订（取代 2026-09-30 那条的第 3 条'定点实现'部分）：
1) K 和 K1 不应写成浮点字面量 #define FOCS_TEMP_K (-6.19241E-05)。STM32F103 无 FPU，禁止浮点字面量。应改为 PC 端预计算好的 Q24 整数：#define FOCS_TEMP_KZ_Q24 (-104)  // k/dz × 2^24, #define FOCS_TEMP_KF_Q24 (-5239) // k1 × 2^24, 提供 Python coefficient_generator.py 给用户维护。
2) 不应在 temperature_offset(temperTask 1Hz) 里算 kz/kf 全局 int32 系数。正确做法：temperature_offset 只缓存 int16 focs_temp_dT（0.1℃单位），dev 在 GetFocsDev 内用 dev = (int32_t)(((int64_t)M × (int32_t)focs_temp_dT) >> 24) 计算，其中 M = df×(zoom-Z0)×K_Z_Q24 - (raw-F0)×K_F_Q24 是 int32 中间量，用 int64 防溢出。
3) 整套方案无浮点计算；libm 仅供 usart.c 等老代码使用，温度补偿路径纯定点。
4) 其他不变：GetFocsCurve 收口三处曲线读取；SetScope limt += dev；temperTask 取消注释；curve_addr_set 废弃。
§
[id:41fb128e] [2026-10-08] [summary:F124Z 温度补偿方案A最终版：标定表 dT 是℃，固件 0.1℃，必须补 1/10（或折进 Q28: KZ=-166/KF=-8383）。df 不除法。FOCS_TEMP_REF 待定标。]
[branch:develop] F124Z focus 温度补偿方案 A 最终版（plastic-lens-servo-v2-bak-pitch-improve, branch develop）——修正 2026-10-08 那条漏掉的 1/10：
1) 单位闭合：标定表 dT 列单位是 **℃**，固件 dT（ntc.c temperature_offset，0.1℃ 单位）差 10 倍，公式里必须显式补 1/10，否则 dev 大 10 倍（RMSE 134 vs 浮点真值 5.16）。
2) 可用写法：KZ_Q24=-104 = (k/dz)×2^24（**不含** 1/10），KF_Q24=-5239 = k1×2^24；M = df×(zoom-Z0)×KZ_Q24 - (raw-F0)×KF_Q24；dev = (int32_t)(((int64_t)M * dT) >> 24) / 10。df = raw_hi - raw_lo（±5 点），**不要再除 (idx_hi-idx_lo)**——可用 zoom 区间（index≥7）内跨度恒为 10，索引 0..6 不在可用范围。
3) 更优写法：把 1/10 折进系数并提高 Q——KZ_Q28=-166, KF_Q28=-8383, QSHIFT=28，dev=(int64)M*dT>>28，省一次软件整数除法（Cortex-M3 无硬件除法），精度更好（RMSE 5.20 vs 5.32）。折进后必须升 Q：Q24 折只剩 -10（3.8% 误差，RMSE 5.35），Q26 得 -42，Q28 得 -166（0.13%）。
4) 中间量：|M| ≤ 2.2e7 可留 int32；M*dT 可达 1.2e10 **必须 int64**。
5) 未决：FOCS_TEMP_REF 三处不一致（f124z.c:52 值 300/注释 25.0℃、ntc.c:6 值 300、老代码 ntc.c:90 与拟合报告注释都是 450 = 45.0℃）。基准温度必须等于标定曲线采集时的温度，判别标准：该温度下 dev 必须 ≈ 0。
§
[id:67c0ad49] [2026-10-08] [summary:Q24 口径（KZ=-104/KF=-5239 + 尾部/10，ΔT 用 0.1℃）与 Q28 口径（KZ=-166/KF=-8383，无/10，ΔT 用 0.1℃）的换算与配套判别：KZ28=KZ24×1.6、ΔT_Q28=10×ΔT_Q]
[branch:develop] F124Z 温度补偿两种定点口径的 ΔT 单位必须配套（v2-bak-pitch-improve, develop）：
1) Q24 口径（推荐给 Excel 手算）：KZ_Q24=-104 = k/dz×2^24，KF_Q24=-5239 = k1×2^24，公式 dev = (M×dT>>24)/10，dT 用 0.1℃；与之对应的表格公式 =ΔT(℃)×inner/2^24。
2) Q28 口径（推荐给固件，省一次软件除法）：KZ_Q28=-166 = k/10/dz×2^28，KF_Q28=-8383 = k1/10×2^28，dev = M×dT>>28，**无尾部 /10**，dT 用 0.1℃；与之对应的表格公式必须写 =ΔT(℃)×10×inner/2^28。
3) 换算关系：2^28/2^24=16，再乘折入的 1/10 ⇒ KZ28 = KZ24×1.6（实为 1.5967）、KF28 = KF24×1.6（1.6002）；因此 ΔT_Q28 = 10×ΔT_Q24。混用两个口径而不改 ΔT，结果必差 10 倍（RMSE 9.29 vs 2.97，看起来像"系数算错"）。
4) 判别法：拿到一对方程系数，先看尾部有没有 /10（或有没有折进 Q），再看 ΔT 是 ℃ 还是 0.1℃——两者只需一个成立即可，同时成立或同时不成立就错 10 倍。
§
[id:091fc4b1] [2026-10-08] [summary:F124Z 温度补偿审阅清单：GetFocsDev 变量遮蔽（分支内重复声明致 idx_lo/idx_hi 未初始化=UB）；边界条件把相对 index 与绝对 zoom 比较（应 index<(MIN_ZOOM-ZOOM_HEAD)+5）]
[branch:develop] F124Z 温度补偿代码审阅必查清单（v2-bak-pitch-improve, develop）——两条会直接毁掉功能的缺陷模式：
1)【变量遮蔽】f124z.c GetFocsDev 外层 `uint16_t idx_lo, idx_hi;` 未初始化，三个 if 分支里全写成 `uint16_t idx_lo = index;`（带类型=内层新声明，只活到该 if 块结束），于是块外的 `df = GetFocsRaw(idx_hi) - GetFocsRaw(idx_lo)` 读的是未初始化值（UB）。教训：分支里给"已声明变量"赋值时**不能再写类型**；改完务必用 -Wall 确认无 maybe-uninitialized。
2)【相对量 vs 绝对量】`index = zoom - ZOOM_HEAD` 是相对量（可用区间 7~2929），但边界条件写成 `index < MIN_ZOOM+5`(=894) / `index > MAX_ZOOM-5`(=3806) 用的是绝对量 → 上界分支永不触发、下界分支错误覆盖 zoom 889~1775，且取 [index, index+10] 而非 [index-5, index+5]，df 相位偏移（zoom900 处 90 vs 表里 95）。正确应写成 `index < (MIN_ZOOM-ZOOM_HEAD)+5` / `index > (MAX_ZOOM-ZOOM_HEAD)-5`。
3) 实测：可用区间内居中 ±5 需要下标 [2, 2934]，而 len(focs_array)=2935（最大合法 2934）→ 全程安全，三分支**完全不必要**。居中差分 RMSE 2.769 优于当前分支 2.872。
4) 关于 Q28 系数：KZ_Q28=-1662=round(k/dz*2^28)、KF_Q28=-83831=round(k1*2^28)，1/10 **未**折入，因此行93 的 /10 是必需的（配 dT 单位 0.1℃），两者自洽、不是 10 倍错误；但宏注释 "/* (k/dz) Q24 */" 是错的，应为 Q28。若折入 1/10 则应改成 KZ=-166/KF=-8383 并**删掉**行93 的 /10。（注意与"Q28 折叠版 -166/-8383"区分：-1662/-83831 是未折版。）
§
[id:14d42c18] [2026-10-08] 嵌入式代码审码清单：区分「绝对坐标宏」与「相对索引宏」两类量纲。ZOOM_HEAD/ZOOM_TAIL 等单位是物理 z 坐标（绝对），ZOOM_OFST = TAIL-HEAD = 数组长度-1 是索引上限（相对）。凡是对 index = coord - HEAD 做 clamp/比较，必须与同域的相对量比较；用绝对量去比相对 index（如 index < MIN_ZOOM+5）会静默错：上界永不触发、下界误覆盖。审码时先确认每个宏是绝对还是相对，再确认比较双方是否同域。
§
[id:476c8947] [2026-10-08] [summary:F124Z：GetFocsRaw 的 index clamp 必须用 ZOOM_OFST(相对数组上界=len-1=2934)，不能用绝对 zoom 宏；且 2934 恰好保证 GetFocsDev ±5 差分在长焦端点可读（2929+5=]
[branch:develop] F124Z 温度补偿（plastic-lens-servo-v2-bak-pitch-improve, develop）：GetFocsRaw 的 index clamp 只能且必须用 ZOOM_OFST。量纲原则：index=zoom-ZOOM_HEAD 是相对量，ZOOM_HEAD(882)/ZOOM_TAIL(3816)/MAX_ZOOM(3811)/MIN_ZOOM(889) 全是绝对 zoom 坐标，不能与 index 直接比。用 `index > MAX_ZOOM`(3811) 永不程，`index < MIN_ZOOM`(889) 会把 index 0..888（近焦半程）夹到 889 造成大面积错。且 ZOOM_OFST=2934（= ZOOM_TAIL-ZOOM_HEAD = len(focs_array)-1）比 MAX_ZOOM-HEAD=2929 更正确的深层原因：GetFocsDev 的 ±5 差分在最大 zoom 3811（index 2929）处需读 GetFocsRaw(2934)，若把上界收到 2929 会破坏长焦端点斜率。可用的「usable 范围」限制应发生在调用方、对绝对 zoom 做双向夹 [MIN_ZOOM, MAX_ZOOM]，再 index=zoom-HEAD；GetFocsRaw 内部只保留数组下标上限 ZOOM_OFST。
§
[id:a9df0a48] [2026-10-09] 定点系数取整误差的速算公式：err = |round(v) − v| / |v|（v = 系数 × 2^Q），最坏界 err ≤ 0.5/|K|（K = 取整后的整数）。推论：系数越小越需要大 Q——因为 |K| ≈ |系数|×2^Q，相对误差 ≈ 1/(2·|系数|·2^Q)。嵌入式定点选型时先算最小系数的 0.5/|K|，超过可接受精度就升 Q 直到中间量逼近溢出上限。
§
[id:c65bd1ba] [2026-10-10] ef_fix_version2 工程扇区头修复评审最终共识：① set/get_status 在 EF_WRITE_GRAN=1 下位编码自洽；② ef_rewrite_sectorhdr 校验上界必须与 FULL 判定解耦（ptr 每步自增 used_site 会导致恒 FULL）；③ sector_statistics_cb 的 using_sector 补偿仅在"搬迁且目标<当前地址"时执行；④ env_iterator 采用有条件重扫（仅向后搬迁时从头重扫+重试上限），原地修复/向前搬迁保留就地重读；⑤ UNUSED 扇区不做迁移目标（用户论证：UNUSED=头从未写成功，不可靠）。总结文档：EF_SECTOR_HEADER_FIX_REVIEW.md。
§
[id:3e90dd82] [2026-10-15] [branch:master] CFLAG_BOARD 决定启动文件+链接脚本组合：BOARD3=startup_stm32f103xb.s + STM32F103CBTx_FLASH.ld（CBT6，128K Flash/20K RAM）；BOARD2 等其他值=startup_stm32f103xe.s + STM32F103RCTx_FLASH.ld（RC，256K/48K）。板子实为 CBT6 时若配成 BOARD2，_estack 被定为 0x2000C000，超出 20KB 物理 RAM，复位后第一次压栈即总线 fault 升级为 HardFault——表现为点调试不进 main、直接停在 HardFault_Handler 的 while(1)，调试控制台典型特征：xPSR=0x01000000（Thread 模式）、msp=0x2000c000 未被压栈。反向也成立（大板配 BOARD3 会浪费资源但不一定立刻崩）。修复：config.mk 改对 CFLAG_BOARD 后必须 make clean 全量重建（启动文件和 .ld 都换）。判别口诀：HardFault 发生在 main 之前 + MSP 恰等于链接脚本 _estack 且物理 RAM 没这么大 ⇒ 先查 CFLAG_BOARD 与 launch.json device 是否一致。
§
[id:0cfbd5e7] [2026-10-15] [branch:master] SetAperture 档位语义（t5215b.c，其他镜头文件结构相同但数值各自独立）：SetAperture(status<16) → GroupGoto(&iris, ZERO + iris_status[status])。iris_status[16] 单调递减表，下标 0=210 最开、15=0 最关；iris.ccwcw=OPEN 且 ms419xx.c:685 中 cwccw==ccwcw 时 site 增加 ⇒ 坐标增大=光圈开。故 SetAperture(0)=全开（tgt=1233≈MAX_IRIS=1240，留 7 步余量），SetAperture(15)=全关（tgt=ZERO=1023≈MIN_IRIS=1013）。互证：sys_task.c PELCOD_STD_IRIS_OPEN(payload>0)→SetAperture(0)、IRIS_CLOSE→15；ZT_SET_IRIS 直设 0-15。判别口诀：先找该型号 .c 里的 iris_status 表（递减→0 开 15 关；若递增则相反），再看 ccwcw 宏与 ms419xx.c:685 的 site±psum 方向约定。
§
[id:c1eaf2da] [2026-11-02] [summary:t5215 v2 归零(找零点)算法：传感器电平定方向 + 每次过零 step 减半的衰减搜索，TIM3 30Hz 驱动，超时 511 拍静默失败]
[branch:master] t5215 v2 归零（Reset2Zero，HARDWARE/src/t5215b.c:497-593）算法要点：
1) 四电机（zoom/focs/ircs/iris）并发，初速 step=8（每拍步数）。
2) 方向直接由零点光电传感器电平决定：zoom.dir = !sens，focs/ircs/iris.dir = sens（GPIO_Mode_IPD，PB3=focs/M1、PB4=zoom/M2、PB5=iris/M3、PB6=ircs/M4）。
3) 每拍重新读传感器，与上一拍比较，**发生变化（过零点边沿）才 step>>=1**（8→4→2→1→0），即每次穿越零点摆幅减半的衰减搜索；step=0 时 GroupStop(brake=true)。
4) 节拍由 TIM3 更新中断（sys_task.c:66 TIM3_Int_Init(3000, CYCLE=800) → 72MHz/800/3000 = 30Hz）置 bLensStatus 驱动；每拍 SetGroupParam 下 psum=step<<msk（zoom msk=3，其余 2）、intct=33333/psum（step 减半 ⇒ intct 加倍 ⇒ 变慢）；每拍 GroupZero 把 aim/site 强制归 ZERO 放弃绝对坐标。
5) 结束后统一 GroupZero(ZERO=0x3FF) + FocusScopeUpdate(ZERO) + StopAll(false)，即把当前机械位置标定为坐标 0x3FF。
6) 风险：COUNTEREXIT 超时 511 拍≈17s 后直接 break，不报错；若传感器不翻转（脱线/极性反/机构卡死）电机会单向连跑 17s 撞限位，且事后坐标仍被标成 0x3FF（静默零点错误）。该函数全程不服务串口（对比 GotoAutoPos 内有 GetsysTask），开机 len_reset 连调两次。
§
[id:c1a46981] [2026-11-05] [summary:温补运行时切换 msk 方案成立但须四要素：site/aim 显式换算、min/max 量程同步或旁路、back 同步换算、切回余数挂 sub 记账；裸切换会被 psum>>msk 截断吃掉补偿量]
[branch:master] 温补细分方案选型结论（2026-11-05 评审）：用户提出“运行时切换 msk”——粗 msk 正常复位 → 切 msk=1 → 按拟合公式补 zoom/focs 细步 → 切回。评审结论：可行，且是三方案（全局降 msk / msk 不动+sub 定点 / 运行时切 msk）中最省的一个，因为 msk 只是 Len_TypeDef 里的软件换算系数，仅被 SetGroupParam（psum=num<<msk）与 LensAimUpload（psum>>msk）两处使用，拍间原子切换即可，芯片与现有运动代码零改动。
落地四要素（缺一不可）：
1) 切换时 site/aim 显式 ×2^Δmsk（细=粗×2^Δ），且必须在两个 VD 拍之间做，不能运动中途切；
2) min/max/FocusScope 等量程同步 ×2^Δ 或细相位临时旁路 LensSiteUpload 钳位——否则细单位下 site 立刻越界被误刹车；
3) len->back 同步 ×2^Δ：机械回差不变、单位变了，zoom 3→1 时不换算则换向补偿脉冲少走 6 PSUM；
4) 切回粗 msk 时余数（fine & (2^Δ−1)，最大 0.75 粗步）必须挂 sub 字段、混进下一次 SetGroupParam 的 psum 并参与记账——裸 psum>>msk 截断会让补偿量整体丢失，等于白补。
配套约定：温度测试、拟合公式、生产补偿必须使用同一套细单位（msk=1 时 1 细步=2 PSUM=1/4 步距角；要 1/8 步距角量子则细相位用 msk=0）；补偿完成前不得启动自动变焦/聚焦跟踪等读坐标的任务。前置门槛不变：先在 msk=1 验证“发 N 细步机构真实走 N/4 粗步”（翻转点扫描或 ADC 读出）。
§
[id:ecde7495] [2026-11-05] [summary:F124Z 切坐标单位须成组处理：4 处 min/max 旧常量覆盖、曲线索引越界、AutoFocStart 旧域判断、协议域转换、back 累乘不幂等]
[branch:develop] F124Z 运行时切换坐标单位（msk 变更）时必须成组处理的 5 类同域点（v2-bak-pitch-improve，分析定位 len_reset 切细分故障得出）：①min/max 覆盖点共 4 处会用未移位常量写回——sys_task.c PELCOD_FUJ_FOC_GOTO(:251-252) 与 PELCOD_FUJ_ZOM_GOTO(:241-242)、f124z.c SetScope(:186/:196) 与 AutoInit(:457-458)，任何一处命中都会把限位拉回旧域，GroupGoto 随即把新域目标夹停（实例：设 4095 被夹到 MAX_FOCS=0xBCF=3023）；②曲线访问链 index=zoom.site−ZOOM_HEAD：zoom 换域后索引=7306 超出 ZOOM_OFST(2934)，GetCurveAim(:161)/GetTrack(:233 等) 直下标无 clamp 属数组越界读（UB），GetFocsRaw 虽有 clamp 但取到曲线末端错值；曲线表/flash 曲线数值本身是旧域标定，出参需 ×2/×4；③AutoFocStart(:692) 的 foc>=MIN_FOCS&&foc<=MAX_FOCS 旧域判断会静默丢弃新域目标（focsaim 不更新导致走旧目标）；④协议域：0x0081/0x0083 回读 site 直发、FOC_GOTO/ZOM_GOTO payload 直用、预置位存取——全部需要边界转换（入 ÷2/÷4，出 ×2/×4），否则上位机看到的零点=4094/8188 而非 2047；⑤back 原地 <<= 不幂等：len_reset 挂在 sysInit(:106) 与 PELCOD_ZT_TO_ZERO(:348) 两处会被多次调用，zoom.back 每次 ×4、focs.back ×2，且 back 是 u8（≥64 时 <<2 截断）——必须改为从 GetBacklash 常量重赋值。判别技巧：异常值先对宏（3023=0xBCF=MAX_FOCS、4094=0x7FF<<1、8188=0x7FF<<2），"读数恰好等于某个限位/常量"基本就是域混用被夹停。
§
[id:70225b0a] [2026-11-06] [summary:curve.bin 记录格式（4 字节/记录，8 记录/zoom 单位 → index<<5 跨步、+2 取 step）与 zom_fit_t 必须从 ZOOM_HEAD 起、文件大小 =(ZOOM_OFST<<5)+4 的对齐判据；cur]
[branch:develop] curve.bin（FILE_CURVE flash 曲线）格式约定与移位判据（plastic-lens-servo 系列）：
1) 记录 = 4 字节：offset+0/+1 = focs(uint16 LE)，+2/+3 = step(uint16 LE，高字节=正向 s1/8、低字节=反向 s2/8)；MATLAB 写 fitfocs 时按 [focs, step] 交错 fwrite 'uint16'。
2) 跨步 = 4 字节 × 每 zoom 单位采样数。当前脚本 zom_fit_t = zoom(1):0.125:zoom(end) ⇒ 8 采样/单位 ⇒ 32 字节/单位，固件必须 (index<<5)；取 step 用 (index<<5)+2。若改回 1 采样/单位才用 <<2（此时 trajectory 的 dx 要传 zstep 而非 zstep*8，且删除 floor(path/8)）。
3) 判据：文件大小必须 = 4×numel(zom_fit_t)，且 == (ZOOM_OFST<<5)+4（f124z：93892 字节）。matlab calib.m 的 linerfit 用 id=cx-x(1)+1，证明源曲线 1 采样/zoom 单位。
4) 对齐前提：zom_fit_t(1) 必须等于 ZOOM_HEAD（f124z=0x372=882），否则整表位移 (zoom(1)-ZOOM_HEAD)×32 字节，读出的曲线整体挪位。
5) curve_addr_set() 里的 (curve_offset<<2) 是旧“按温度平移地址”方案的记录数→字节换算（4 字节/记录），与跨步无关，且该函数已废弃（死代码）；curve_read_focs 里的 data += (curve_offset>>1) 同样无效。
6) 该 <<5 格式由 commit 04e21af“完成温度控制模块”引入，与 FILE_CURVE/FILE_CURVE_BASE 方案同时出现，历史上不存在 1 采样/单位的 norflash 读取实现。
§
[id:4e462ccd] [2026-11-06] [summary:t5215 v2 工程坐标步→芯片 psum 微步的换算：psum = step<<msk，zoom msk=3（×8）、focs/ircs/iris msk=2（×4）]
[branch:master] t5215 v2 步距量纲约定（ms419xx.c:513-532 SetGroupParam、:685 LensAimUpload 反向换算）：
1) site/aim 的 1 个工程坐标单位 → psum += 2^msk，msk 来自 t5215b.h 的 *_STEP_Pos：ZOOM_STEP_Pos=3（×8）、FOCS_STEP_Pos=2（×4）、IRCS_STEP_Pos=2（×4）、IRIS_STEP_Pos=2（×4）。故"走一步=8 微步"只适用于 zoom，focs 是 4。
2) 每拍（TIM3 30Hz）最多走 onestep=ZOOM_STEP=FOCS_STEP=26 个工程步 ⇒ zoom psum 208、focs 104；归零 Reset2Zero 的 step=8⇒zoom psum 64，末级 step=1⇒8。
3) 换向时 num += back（MOTORx_BACKLASH=1）再一起 <<msk，lash 记账后由 LensAimUpload 的 -lash/+lash 扣回，不回改换算。
4) psum 寄存器 12 bit（Third_Party/ms419xx/ms41968.h:251）；intct = 33333/psum（16 bit）。工程里 micro=0、phmod=0（ms419xx.c:165 等）——psum 一个单位对应多少物理步（细分）取决于 MS41968 手册对 micro 的定义，代码里无法判定。
§
[id:a214eb56] [2026-11-06] [summary:MS41968：1 PSUM = 1/8 步距角（256 细分），zoom 1 工程步 = 8 PSUM = 1 步距角、focs = 4 PSUM；芯片可下 1 PSUM，sub-step 余量 8×，但受回差/零点重复性限制，不能替代温]
[branch:master] MS41968 PSUM 量纲与亚步进（sub-step）可用余量（依据 E:\downloade\MS41968\MS41968.pdf + MS419XX系列马达驱动原理与教程.pdf，2026-11-04 实读）：
1) 细分定义：MICRO[1:0] 00/01=256、10=128、11=64（手册 p.44 表），工程里 micro=0（ms419xx.c:165/199/233 等）⇒ **256 细分**。手册 p.46 PSUM 表：总微步 = 8n（256）/4n（128）/2n（64）；教程 p.16 明确“1 个相电流周期 = 4 个步距角，故按步距角算是 64 细分”。合得：**1 PSUM 单位 = 8 微步 = 1/8 个步距角**。旁证：教程 p.15 “监测脉冲每走 1/8 个步距角输出 1 个脉冲”；教程 p.10 例 “PSUM=0x10 → 走 0x10/8 = 2 个步距角”。
2) 工程换算：zoom msk=3 ⇒ 1 工程坐标步 = 8 PSUM = **1 个整步距角**；focs/ircs/iris msk=2 ⇒ 1 工程步 = 4 PSUM = 1/2 步距角。固件最细只下到 8/4 PSUM（SetGroupParam 里 (num<<msk)，num≥1），**芯片本身可接受 PSUM=1** ⇒ 闲置 8×/4× 分辨率。
3) 时序约束（手册 p.43/48）：INTCT×768 = OSCIN/转动频率；INTCT×PSUM×24 = OSCIN/VD频率（OSCIN=27MHz）。工程用 INTCT=33333/PSUM 恰好填满 1 个 VD（33.75Hz）：PSUM=208 → INTCT=160 → 29.6ms。PSUM 超出 VD 周期的部分“会被取消”（p.42），安全。PSUM=1 时 INTCT=33333 也正好填满 1 个 VD。
4) 结论（工程判断）：温度零点漂移用 sub-step 补收益极低——实测漂移 0.8 步/℃、45℃ 差 13 步 = 104 PSUM（zoom 尺度），比 1 PSUM 大两个数量级；回差 1 工程步 = 8 PSUM ≈ 20µm（GP1S196 过渡区 0.3mm≈15 步）会吞掉任何 ≤8 PSUM 的修正，故 sub-step 必须单向逼近。分辨率≠精度：先做 ADC 中点找零（重复性）+ 温度补偿表（系统性偏移）。若确要用 sub-step，走“末端补脉冲（主移动后补 1 个 VD 的 psum=0..7）”而非全轴降 msk；注意 LensAimUpload 的 psum>>msk 会把 sub-step 截断，需显式记账。
