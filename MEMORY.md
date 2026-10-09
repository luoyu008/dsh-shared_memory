[id:1ca7a222] [2026-09-30] DSH Desktop（Electron 44 宿主，process.execPath = "E:\dsh\DSH Desktop\DSH Desktop.exe"）里，插件若用 spawn(process.execPath, [script.mjs]) 派生 Node 子进程，必须显式给 env 加 ELECTRON_RUN_AS_NODE=1；否则子进程是第二次启动 GUI，被单实例锁（resources/app/lib/main.js:3735 app.quit）与 isDesktopBackgroundNodeRequest 守卫（main.js:171/3967）静默吞掉，stdout/stderr 全空——dsh-memory-evolve 的记忆同步（spawnWorker，lib/sync/index.js:74）就因此报“worker 无输出”，所有同步动作在桌面端均不可用。上游 dsh-web-app（lib/index.js:129）与 dsh-subprocess-local、dsh-ptc-runtime-node 都按此约定显式设置该变量。
§
[id:4796d1d8] [2026-10-08] 代码评审通用教训：当代码同时使用浮点字面量 + 多个 Q 常数 + 多重因子（×10 / 100 / dz / 2^24）时，公式极易出现"因子不闭合"bug。审查清单：1) 每个系数命名要和数值语义一致（K vs K_Z）；2) ΔT 单位（0.1℃ vs ℃）必须在公式里显式表达；3) 所有缩放因子（×10 / /100 / >> 24）的乘积必须是 1；4) 边界条件（idx=0, idx=max）单独算一次与中间区域对比。</content>
</invoke>
§
[id:0ecdb37d] [2026-10-08] 定点补偿公式的单位闭合检查：定标系数（k、k1 等）的量纲由**拟合时自变量的单位**决定。若固件产出自变量的单位与标定表不一致（如标定表 ΔT 用 ℃、固件用 0.1℃），公式里必须显式补 1/10，或把 1/10 折进 Q 系数——折进后必须同步提高 Q，否则系数只剩 1~2 位有效数字（Q24 折 1/10 得 -10，误差 3.8%；升到 Q28 得 -166，误差 0.13%）。三步判别：① 把系数代回标定表，RMSE 正常 ⇒ 系数与该表单位匹配；② 检查固件在同一物理条件下产出的自变量数值是否等于表里的值；③ 不等就补因子。
§
[id:b7dc2ddc] [2026-10-08] [summary:DSH Windows 沙箱下 pwsh 对原生命令做重定向/管道会导致子进程不启动（输出与退出码皆空）；绕过用 cmd /c "python x.py > out 2> err" 内层重定向。]
DSH Windows 沙箱下 pwsh 调用 Python/原生命令的坑：在 workspace-write 等受限模式里，只要用 PowerShell 层做输出捕获或重定向（> f、2> f、2>&1、| cmdlet），子进程根本不会启动——stdout 为空、$LASTEXITCODE 为空，偶发报 "程序 xxx 无法运行: Access is denied"(NativeCommandFailed)；裸跑则完全正常。与 Python 无关（node -e、where.exe 同样复现）。判别方法：让脚本自己写 marker 文件，带重定向时不生成即证明进程未启动。绕过写法：cmd /c "python x.py > out.txt 2> err.txt"（内层重定向），已验证 200 行 stdout+stderr+非零退出码完整保留。根因即已知的沙箱禁止命名管道/管道 stdio 边界，属预期行为而非缺陷。
