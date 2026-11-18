[id:1ca7a222] [2026-09-30] DSH Desktop（Electron 44 宿主，process.execPath = "E:\dsh\DSH Desktop\DSH Desktop.exe"）里，插件若用 spawn(process.execPath, [script.mjs]) 派生 Node 子进程，必须显式给 env 加 ELECTRON_RUN_AS_NODE=1；否则子进程是第二次启动 GUI，被单实例锁（resources/app/lib/main.js:3735 app.quit）与 isDesktopBackgroundNodeRequest 守卫（main.js:171/3967）静默吞掉，stdout/stderr 全空——dsh-memory-evolve 的记忆同步（spawnWorker，lib/sync/index.js:74）就因此报“worker 无输出”，所有同步动作在桌面端均不可用。上游 dsh-web-app（lib/index.js:129）与 dsh-subprocess-local、dsh-ptc-runtime-node 都按此约定显式设置该变量。
§
[id:4796d1d8] [2026-10-08] 代码评审通用教训：当代码同时使用浮点字面量 + 多个 Q 常数 + 多重因子（×10 / 100 / dz / 2^24）时，公式极易出现"因子不闭合"bug。审查清单：1) 每个系数命名要和数值语义一致（K vs K_Z）；2) ΔT 单位（0.1℃ vs ℃）必须在公式里显式表达；3) 所有缩放因子（×10 / /100 / >> 24）的乘积必须是 1；4) 边界条件（idx=0, idx=max）单独算一次与中间区域对比。</content>
</invoke>
§
[id:0ecdb37d] [2026-10-08] 定点补偿公式的单位闭合检查：定标系数（k、k1 等）的量纲由**拟合时自变量的单位**决定。若固件产出自变量的单位与标定表不一致（如标定表 ΔT 用 ℃、固件用 0.1℃），公式里必须显式补 1/10，或把 1/10 折进 Q 系数——折进后必须同步提高 Q，否则系数只剩 1~2 位有效数字（Q24 折 1/10 得 -10，误差 3.8%；升到 Q28 得 -166，误差 0.13%）。三步判别：① 把系数代回标定表，RMSE 正常 ⇒ 系数与该表单位匹配；② 检查固件在同一物理条件下产出的自变量数值是否等于表里的值；③ 不等就补因子。
§
[id:b7dc2ddc] [2026-10-08] [summary:DSH Windows 沙箱下 pwsh 对原生命令做重定向/管道会导致子进程不启动（输出与退出码皆空）；绕过用 cmd /c "python x.py > out 2> err" 内层重定向。]
DSH Windows 沙箱下 pwsh 调用 Python/原生命令的坑：在 workspace-write 等受限模式里，只要用 PowerShell 层做输出捕获或重定向（> f、2> f、2>&1、| cmdlet），子进程根本不会启动——stdout 为空、$LASTEXITCODE 为空，偶发报 "程序 xxx 无法运行: Access is denied"(NativeCommandFailed)；裸跑则完全正常。与 Python 无关（node -e、where.exe 同样复现）。判别方法：让脚本自己写 marker 文件，带重定向时不生成即证明进程未启动。绕过写法：cmd /c "python x.py > out.txt 2> err.txt"（内层重定向），已验证 200 行 stdout+stderr+非零退出码完整保留。根因即已知的沙箱禁止命名管道/管道 stdio 边界，属预期行为而非缺陷。
§
[id:97ce2d5a] [2026-10-09] 嵌入式调试心法：当一段"理论上无效果"的代码疑似改变了运行结果（差 1 个 LSB），先怀疑量化/舍入边界与环境漂移，而不是理论错了——计算结果翻转点（如 dev 在 dT=258.7 处从 -19 跳到 -20），实测温度跨过该点 0.1℃ 即可复现差异。验证手段是打印中间变量（dt:）对账，而不是反复改代码试。另：用脚本仿真 C 整数除法时，负数必须用向零截断语义，Python 的 // 是 floor，结果不同。
§
[id:83a33d09] [2026-10-13] 本机 git 全局配置 http.proxy/https.proxy = http://127.0.0.1:7890（Clash）。Clash 未启动时所有 git HTTP 操作（含内网 192.168.1.8:3000 的 Gitea）都会失败，报错形如 "Failed to connect to <host> via 127.0.0.1"。内网地址应绕过代理：git config --global http.http://192.168.1.8:3000.proxy ""。
§
[id:5fd723c6] [2026-10-16] 镜头伺服 MCU 职责边界（plastic-lens-servo 系列通用）：镜头 MCU 只做"光圈 + 变焦/聚焦 + 日夜滤光片 + 温度曲线"，不碰曝光三要素中的快门/增益/亮度——0xF00D 自动增益、0xF02B 亮度设置、0x0093/0x0095/0x0097 快门命令在 Pelco 头文件里有定义但 sys_task.c 无 case，全部被忽略（归相机主控/ISP）。判别方法：grep 命令宏在 *.c 是否有引用，仅头文件定义=未实现。另注意 MANU_DANI(0xF00F)/EXTR_DANI(0xF019) 名字是增益控制，实际被本工程复用为日夜滤光片切换（SetIrStatus）。
§
[id:cee89552] [2026-10-21] MATLAB 的 webread 会走代理（MATLAB 预设项→Web 或系统代理），Clash 关闭但系统/MATLAB 代理仍指向 127.0.0.1:7890 时，对内网地址（如 192.168.1.161 相机）的请求报 "Failed to connect to 127.0.0.1 port 7890: Connection refused"。weboptions 没有 Proxy 参数；按请求绕过代理要用 matlab.net.http：HTTPOptions('UseProxy', false, 'ConnectTimeout', 5) + RequestMessage('get').send(uri, opts)。nm_client 走裸 TCP 不受影响，故同脚本里串口命令正常、仅 webread 挂。
§
[id:b263a21e] [2026-10-22] 记忆同步（dsh-memory-evolve）网络故障的两类错误判别 + 本机代理端口迁移：① git 报 `Failed to connect to <host> port 443 via 127.0.0.1 after ~2100 ms: Could not connect to server`（2 秒快速失败）= 代理端口没有进程监听（代理没启动，或换了代理软件换了端口）；② 报 `Failed to connect ... after 21000 ms` 或 `Recv failure: Connection was reset` = 代理进程在跑但节点不通（订阅节点失效/内核不支持该协议）。2026-10-22 本机从 Clash for Windows（混合端口 7890，内核不支持订阅里的 vless/reality，只解析出 server:1.1.1.1 port:65535 的占位节点）迁到 Clash Verge Rev（verge-mihomo，混合端口 **7897**，socks 7898，http 7899），git 全局 http.proxy/https.proxy 必须同步改成 http://127.0.0.1:7897。换代理软件后务必先确认 `Get-NetTCPConnection -State Listen -LocalPort <port>` 有监听，再改 git 配置。
§
[id:5238dadb] [2026-10-26] 曲线图（尤其对数坐标的 R-T 图）数字化取数的可靠流程：① 不要靠肉眼读，先定位坐标轴——用**十进制网格线/刻度文字的中心**做线性回归，而不是用图像边框（本例边框在 1e5 线上方 6px、右边框在 120℃ 线右侧 19px，直接用边框会让拟合的 B 值系统性偏 1.8%，而用网格线校准后 R25 与型号标称完全吻合、B 误差 <0.1%）；② 对数轴上每十倍程像素数 D 要由相邻十进制线间距求平均，取数 R=10^((y1-y)·5/(y1-y0))；③ 曲线提取靠逐列找彩色像素连续段（假设曲线不交叉，按上下顺序编号），比颜色聚类稳健；④ 有标称参数时一定要回头验算（R25、B），残差 1~2% 属正常读数误差，系统性偏移则说明轴校准错了。
§
[id:4429c597] [2026-10-27] DSH NEXT 0.2.1 的 Windows 沙箱后端 = @deepseek-ai/dsh-sandbox-windows-acl：每条受限命令先由 runner 建 WRITE_RESTRICTED 受限令牌（restricting SIDs = 登录 SID + Everyone + 工作区 SID + 会话临时 SID）并把完整性降到 Low，再 spawn 真实子进程；runner 自身失败会打印 "windows-acl-run: <detail>" 并以 127 退出。若命令返回 0xC0000142（STATUS_DLL_INIT_FAILED）且无输出，说明子进程在 DLL 初始化阶段就死了（不是命令错、不是权限拒绝）——本机实测成因：DSH 以管理员/High 完整性运行时，High→Low 令牌组合导致子进程 DLL 初始化失败；改用 danger-full-access 立即恢复。另：该后端在 Windows 上读不受限、只限写；console 隔离不可用（CREATE_NO_WINDOW/CREATE_NEW_CONSOLE 会引发同样错误）。
§
[id:295cca71] [2026-10-29] MATLAB 的 fit(x,y,'smoothingspline','SmoothingParam',1) 是"最大平滑"的最小二乘样条，**不插值数据点**，端点弯曲最大处的偏差可达 5%（本轮 -40℃ 处公式值 526 而脚本输出 552）。用 Python 复现这类脚本时只能对趋势/量级做校验，不能要求逐点相等；反过来，把这种拟合结果当"精确对照表"也会引入端点偏差。
§
[id:0b222bdf] [2026-10-29] NTC 只有 R25+B 两个参数时，单 B 公式 R=R25·exp(B·(1/T−1/T0)) 在两端必然偏陡（低温偏高 10~15%、高温偏低 5~10%），厂家图上的真实曲线是"变平"的。工程上不用换 B，而是给单 B 结果乘一个形状因子 k(T)=R真实/R单B，用三次多项式拟合 ln k（本例 33 个点、拟合误差 0.34%、等效 0.15℃），既保住物理模型又对齐厂家数据。判别方法：若换 B 值能让端点对上，说明模型选错；若只有加 k(T) 才能同时对上两端，则是真实器件的非线性。
§
[id:81efbfb3] [2026-10-29] DSH 插件（profile bundle）升级的正确姿势与两个坑：① 全局 dsh CLI（AppData\Roaming\npm\dsh.cmd，v0.1.5-rc.2）对 desktop profile 直接拒绝——报 "profile \"desktop\" is managed exclusively by the Electron application"，只能用应用自带 pnpm（C:\Users\Administrator\AppData\Local\Programs\DSH NEXT\resources\runtime\x64\primary-runtime\dependencies\pnpm\bin\pnpm.cjs，配 E:\node\node.exe 运行，v11.28.5）在 C:\Users\Administrator\.dsh\profiles\desktop 里执行 add。② 运行中的 DSH（patchReload: live）锁住 node_modules\<plugin> 目录本身：Rename-Item 该目录报"另一个进程正在使用此文件"，但目录内文件/子目录仍可增删改名。pnpm 安装最后一步是 rename 临时目录，因此必然失败（EPERM，偶尔 0xC0000005 访问冲突），结果是 package.json 与 pnpm-lock.yaml 已写成新版本、node_modules 还是旧版本，出现不一致。绕过办法：先在临时目录 pnpm add 解出新版本（注意 node_modules\<pkg> 是符号链接，Target 是相对路径，必须用 Join-Path 拼 node_modules 目录再 GetFullPath，不能对 Target 直接 Resolve-Path，否则会解析到当前工作目录），再 robocopy /MIR 把内容镜像进目标目录（只动内容、不动被锁的目录本身）。改完必须重启 DSH 才加载新版本，宿主侧模块不热更新。
§
[id:1df1188b] [2026-10-30] DSH profile 的 node_modules 里可能存在跨 profile 的 Junction：本机 C:\Users\Administrator\.dsh\profiles\desktop\node_modules\dsh-context 就是 Junction，Target = C:\Users\Administrator\.dsh\profiles\web\node_modules\dsh-context。判别方法：Get-Item 看 LinkType（Junction/SymbolicLink），LinkType 非空即链接；且 desktop 的 pnpm-lock.yaml 里查不到该包（0 条记录）而 web 的有——说明真实安装在 web profile。后果：升级该包必须在 web profile 里跑 pnpm add（desktop 的清单不用动），desktop 侧会因 Junction 自动跟随新版本，不要试图在 desktop 里 add（会与 Junction 冲突或生成第二份副本）。另注意 pnpm 的 minimumReleaseAge（默认 1440 分钟）会拦住刚发布的新版本，需加 --config.minimumReleaseAge=0；但注意该参数只在包确实很新时才需要，且加了它会跳过供应链延迟保护。
§
[id:7e0f00af] [2026-11-02] DSH NEXT 有三层不依赖 shell 的插件回退通道（当插件把 executor 搞坏、agent 跑不了命令时用）：
1) 安全模式：GUI 里有「进入安全模式…」动作，或带启动参数 `"--next-safe-mode"` 启动 C:\Users\Administrator\AppData\Local\Programs\DSH NEXT\DSH NEXT.exe。它使用独立的临时环境（文案：当前使用临时环境。退出安全模式并重启后返回原 Profile，临时数据不会保留），原 profile 原封不动，因此插件不会被加载，可以在其中修 package.json。退出用「退出安全模式并重启…」(normal-mode)。
2) 快速恢复/恢复模式：启动参数 `"--next-recovery"`，提供 切换 Profile / 管理数据 / 查看诊断。注意恢复模式下部分配置 API 会抛错（"安全模式和恢复模式下请使用恢复助手"），安全模式还强制固定桌面设置（SAFE_MODE_PREFERENCES）。
3) 插件停用：plugin-manager 的启停不是改 bundles 数组，而是写 package.json 里的 `dsh.desktopDeselectedBundles` 字符串数组（本机现有条目：@linxin666/dsh-client-ui-community-plugins、dsh-cost-meter、dsh-git-graph 等），把包名加进去即可停用而不卸载。
另：profile 的 `.plugin-manager\backup-<时间戳>\` 目录存 package.json + pnpm-lock.yaml 快照，是最终的还原源。
§
[id:e8018254] [2026-11-02] DSH profile 卸载插件后，包名可能残留在 `dsh.desktopDeselectedBundles` 里（本机 dsh-git-graph 就是：dependencies 与 node_modules 都没有，但停用列表里还留着 "dsh-git-graph"）。后果：重新安装该插件时它会被判为"已停用"而静默不加载，症状是"装了没效果"。因此重装任何插件前应先 grep 一下 desktopDeselectedBundles，需要启用就把该条删掉（注意 dsh-cost-meter 这类是刻意同时存在于 bundles 与 deselected 里的，属正常停用状态，不要误删）。另：给 profile 新增一个从未装过的包时，pnpm add 不会遇到那个著名的目录重命名 EPERM（rename 的目标目录不存在），所以不需要"临时目录安装 + robocopy /MIR 镜像"的绕过流程，那个流程只在替换已存在且被运行中 DSH 锁定的包时才需要。
§
[id:a0e304e6] [2026-11-02] 回差（backlash）建模时，基准点自身的接触状态必须与测量方向一致，否则"双向平均消回差"是错的。正确模型：电机位置 M 与滑块位置 C 分离，正向接触 C=M−b/2、反向接触 C=M+b/2。若基准点 M₀ 本身是在正向接触下确定的（例如单向逼近的找零），则正向复测得 s₁=δ（δ 为真实漂移，与 b 无关、与温度无关），反向复测得 s₂=δ−b，取平均得 δ−b/2 —— 平均反而引入常数偏差。所以"单向确定的基准 + 单向测量"才是自洽的，双向平均只在基准本身也由双向平均确定时才无偏。推论：① 任何时候都只用与基准同向的那次测量；② s₁−s₂ 是回差 b 的直接读数，降级为诊断量不进公式；③ 若无法确定基准的接触方向，就在基准条件下重测一次，读到的非零常数就是该偏置，作空白扣除。
§
[id:4125a44b] [2026-11-02] 开环步进定位系统里，"用传感器边沿做零点"会把坐标原点锚死在那个边沿上，因此**该边沿自身的漂移在重新归零后的坐标系里完全不可观测**——归零后边沿永远是同一个坐标值。推论：① 想测零点漂移 Δz，必须"只归零一次、之后冻结坐标"，绝不能在每个工况点重新归零；② 重新归零只适合测"相对零点的其它量"（如光学离焦、行程变化）；③ 冻结坐标法可一次同时分离两个量——扫传感器边沿得 Δz（纯机械，不需成像），扫功能目标点得 ΔX_sharp（如清晰点，需成像）；④ 归零算法自身有噪声（二分/衰减搜索的终止位置落在边沿 ±1 步），当它与被测 Δz 同量级时（常见于小步距机构），"每温度重新归零再比对"这条路信噪比≈1，不可用。
§
[id:39ee84f2] [2026-11-04] 烧曲线（YMODEM→W25Q64）断连/失败的根因模式：栈溢出。ymodem_handle_packet 栈上 received_data[1029]+file_name[50] + W25QXX_Erase_Write 栈上 W25QXX_BUFFER[4096] ≈ 5.5KB 栈需求；CBTx(20K RAM) 链接脚本下 _estack(0x20005000) 到 bss 顶(0x20003f98) 仅 4200B → 每个数据包必踩穿 .bss 顶部（_SEGGER_RTT@0x20003e04 → RTT 日志断连、__sf/errno → libc 状态损坏）→ 不回 ACK → 上位机报 Invalid up buffer 断开。判别特征：file_name/file_size 能打印（包头阶段栈浅）但无任何 Progress 日志（第一数据包就爆）。同代码在 RCTx(48K RAM) 链接脚本下没事——换小 RAM 芯片后爆。修复方向：包头回调按 file_size 预擦除 + 数据包改 Write_NoCheck 页写（去掉 4KB 栈缓冲）+ received_data 改指针。教训：栈上放大数组的调用链必须按目标芯片 RAM 预算核算，跨板移植（大 RAM→小 RAM）时优先查这类问题。
§
[id:6bdbf784] [2026-11-05] 嵌入式固件"运行时切换坐标单位系数"审查清单（本镜头伺服项目实战）：① 同单位字段必须成组换算（site/aim/min/max/back 一个不能漏）——尤其警惕 ISR 里的派生拷贝（本工程 TIM3 ISR 每拍 LensSiteUpload 执行 site=aim，只改 site 不改 aim 会在 33ms 内被静默盖回）；② 多字段换算序列必须关对应节拍中断，否则 ISR 读到撕裂状态可能触发限位误刹车；③ 单位换算一律从基准常数赋值，禁止原地累乘（复位函数被多次调用时累乘不幂等）；④ 换单位前确认所有读该坐标的下游（曲线表索引、协议上报）在新单位下不会被执行。
§
[id:f8594343] [2026-11-06] Reset2Zero 找零的理想收敛特性：最终停点与起步距离无关，只由起步侧奇偶决定——step 仅在检出传感器翻转时减半（8→4→2→1→0 恰好 4 次跨沿，方向严格交替），末次 step=1 跨沿固定落在"起步侧、沿后第一个整数步"。左起步停沿左邻步，右起步停沿右邻步，两侧差 1 步；只要每次复位起步侧一致（如 zoom 总停在 wide 侧），零点锚定在理想情况下零离散。±1 步噪声来自起步侧不定/回差补偿误差/阈值抖动，而非算法本身。
§
[id:595b8334] [2026-11-18] 配对的"换域"函数（sub/return 或 encode/decode）设计纪律：先问每个函数对"当前位置"做什么假设——重锚定（声明当前=新域零点）还是保位换算（site/aim 按域差缩放、原点不动）。两者函数体可能完全不同，也可能完全相同（若都是重锚定型，sub/return 其实是同一个函数，重复实现两个名字只会引入镜像错误如 ZERO>>num vs ZERO<<(3−num)）。写第二个之前先把第一个性质证明出来；常量永远从它的"家域"缩放进目标域（<<方向），不存在">>回家"的写法。
§
[id:ccd2a48a] [2026-11-18] 把有符号补偿量折进状态机计数时，设计纪律：①先用具体数字把四象限全走一遍（同向小/同向大/反向小/反向大），反例分支往往在这里暴露；②"负计数就提前完成"类短路几乎总是错的——先问短路点电机实际停在哪（状态入口≠你以为的位置），再考虑"换方向走 |target|"，同向延续通常零新增换向、比停下来更便宜；③分支处理后必须检查两侧的最终接触状态（逼近方向/换向次数）是否一致——不一致会在分支边界处产生回差量级的停点跳变，且边界温度完全可能落在工作范围内。
