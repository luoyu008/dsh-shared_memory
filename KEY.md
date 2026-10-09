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
