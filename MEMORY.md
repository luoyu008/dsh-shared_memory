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
