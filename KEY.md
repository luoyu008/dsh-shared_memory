[id:62904e53] [2026-09-29] [summary:MCU 聚焦 AF 卡死：v1 与工作区副本源码不同（v1 独有 CMAR 重定向 bug 与 flashdefendTask）；根因是同步阻塞 + 63 字节 RX 缓冲溢出/帧错位 + 位置模型被写成 0xFFFF。]
镜头伺服（STM32F103，F124Z/BOARD3/MASTERMODE，编译工程 E:\MCU_Project\plastic-lens-servo_v1，工作区分析目录 V1troubleshooting...）聚焦 AF 卡死问题结论：
1) 分析必须以 plastic-lens-servo_v1 为准，工作区副本与它不一致（v1 独有 flashdefendTask、USART_Callback 内 CMAR=USART_DMA_BUFFER2 重定向且永不还原、TIM4/time4_count、ymodem 两参数接口）。
2) 根因骨架：单线程超级循环里运动命令同步阻塞（GroupGoto 最长 255 拍 = AF QUICK 2.1s / 普通 8.5s；GotoAutoPos 第一段 while 最长 511 拍 ≈17s 且完全不服务串口），而 RX 环形缓冲只有 63 字节（115200 下 9 帧 = 5.5ms 余量）→ 溢出丢字节 → 帧错位 → 执行不存在的命令/应答张冠李戴；叠加 4 字节 send_back 与 7 字节 send_pelcod 混流及各主动上报帧（Feed_zoomBack 的 0x0083、404 状态帧、A5A5 flash 帧）。
3) 位置变量（zoom.site/focs.site）变 0xFFFF/0xFFF5 的来源：GetCurve 系列不校验索引与 curve_address、GroupAim/AdjustPosition 不夹紧、LensSiteUpload 限位依赖方向位、ISR 与 SetGroupParam 撕裂、safe_u16_add 溢出返回 0xFFFF。
4) 零成本判别：卡死瞬间单发 0x0081 看回复位置是否 0xFFFF；RTT 里 move_step_* 的 time 是否 2100/8500ms；End auto position loop 是否接近 511；发 0xF091 payload=1 让 MCU ef_print_env() 对账 ENV；检查 YMODEM 日志 CMAR 是否等于 rxbuf。
详见 AF_HANG_ROOT_CAUSE_ANALYSIS.md。
