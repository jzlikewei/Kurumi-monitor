# 本机大盘

> 把 macOS 与 Windows 的系统指标搬进 A 股行情终端的界面语言。
> CPU、GPU、内存、磁盘、网络、进程——每一项系统指标都是一只「个股」，
> 实时 K 线、主力动向、逐笔成交、自选行情一应俱全。

这是一个跑在 MiniMax Code 里的 Mini App：Node 服务负责采集与聚合，
页面负责渲染，两者通过 SSE 推送每一条采样。

## 平台支持

| 平台 | 采集方式 | 指标数 | 验证状态 |
| --- | --- | --- | --- |
| macOS | `vm_stat` / `ioreg` / `netstat` / `ps` | 41 | 真机实测 |
| Windows | 常驻 PowerShell + 性能计数器 | 41 | 见下方说明 |

Windows 侧的采集跑在一个常驻 PowerShell 进程里：脚本启动时预建全部
`PerformanceCounter` 对象，之后每轮只读 `RawValue` / `NextValue`。
`Get-Counter` 每次枚举要 1 到 4 秒，预建对象读一次在微秒级——每秒拉起
六次 `powershell.exe` 会让采集进程自己变成 CPU 大户，把被测数据污染掉。

**Windows 验证范围**：进程采集链路（`Get-Process`）在 macOS 上用 PowerShell 7.6.9
实跑验证过，走的是 .NET 跨平台实现；内存、GPU、磁盘、网络这几项依赖 Windows 专有的
`PerformanceCounter` 与 `Get-CimInstance`，用真实输出格式的样本做了端到端测试
（29 项断言覆盖分段换算、CPU 差分、单位转换与生命周期），尚未在 Windows 真机上跑过。

## 界面

```
┌──────────────────────────────────────────────────────────────────────┐
│ 品牌 · 实时采样 · 采样周期 · 暂停 · 配色 · 主题 · 设置 · 时钟          │
├──────────────────────────────────────────────────────────────────────┤
│ 指数条（8 格，含迷你 K 线）                                           │
├───────────────┬──────────────────────────────┬───────────────────────┤
│ 我的自选       │ 品种头（今开/最高/最低/均价） │ 主力动向（Top 10）    │
│ 大盘行情       │ K 线周期切换                 │ 名次徽标              │
│ 按板块分组     │ 主图：K 线 + MA5/10/20       ├───────────────────────┤
│ 41 项 + 逐核   │ 副图：内存构成三色堆叠        │ 逐笔成交（事件流水）  │
├───────────────┴──────────────────────────────┴───────────────────────┤
│ 个股列表 · 进程（可搜索、可加自选、可清仓，每行一条迷你 K 线）        │
├──────────────────────────────────────────────────────────────────────┤
│ 采样/进程/历史/占用/用户 + 跑马灯快讯                                 │
└──────────────────────────────────────────────────────────────────────┘
```

红涨绿跌遵循 A 股习惯，标题栏「绿涨」按钮可切换为国际配色。
涨跌始终带 `+` / `−` 符号，不依赖颜色传达方向。

## 安装

需要 MiniMax Code 桌面端。

**手动安装**

1. 克隆或下载本仓库。
2. 把 `miniapps/kurumi-monitor/` 整个目录（包含隐藏的 `.minimax-plugin/`）
   复制到 `~/.minimax/plugins/kurumi-monitor/`。
3. 重启 MiniMax Code，确认插件已识别并启用。
4. 在对话里说「打开本机大盘」，或从插件列表里启动。

**交给 Agent 安装**

把目录或 ZIP 直接发进对话，让 Agent 按成品包流程发布。

## 数据访问范围

这个 Mini App 以你的身份在本机运行，请先确认以下行为：

- **读取全机进程列表**：PID、进程名、属主、CPU 占用、常驻内存、线程数、启动时刻。
- **读取系统指标**：CPU（含逐核）、GPU 占用与显存、内存分段、磁盘读写、网络流量。
- **可以终止进程**：进程表的「一键清仓」会向选中进程发送 `SIGTERM`，
  勾选强制清仓则发送 `SIGKILL`。按钮只对当前用户有权发信号的进程可用，
  本程序自身与宿主进程永远在保护名单里。操作前有二次确认。
- **落盘位置**：采样与事件写入系统临时目录（macOS 为 `$TMPDIR/kurumi-monitor/`，
  Windows 为 `%TEMP%\kurumi-monitor/`），由系统自行清理；
  偏好设置写入 `~/.minimax/v2/plugin-data/liveboards/kurumi-monitor/settings.json`。
- **不联网**：所有采集与渲染都在本机完成，没有任何外部请求。

## 致谢

界面语言与交互规格的灵感来自 [PerfTicker](https://github.com/QiuSui1145/PerfTicker)，
作者 **QiuSui**（QiuSui_Studio），一个把 Windows 任务管理器做成炒股软件行情终端的
C# 单文件程序，采用 GPL-3.0 协议。

本项目是独立实现：技术栈为 JavaScript / Node，运行在 MiniMax Code 的 Mini App 宿主里，
两边没有共享任何源码。沿用 GPL-3.0 是出于对原作者的尊重。

本项目补上了 PerfTicker 未覆盖的 macOS 采集链路，并在 Windows 侧用常驻 PowerShell
重做了同一套指标。

## 开发

包结构：

```
miniapps/kurumi-monitor/
├─ .minimax-plugin/plugin.json    插件清单
├─ package.json                   mcode 扩展声明
├─ icon.png
└─ miniapp/
   ├─ miniapp.json                Mini App 声明
   ├─ client/                     页面：index.html / app.css / app.js / chart.js
   └─ node/
      ├─ server.mjs               HTTP 路由、SSE 推送、采样调度、清仓校验
      ├─ metrics.mjs              指标目录与格式化
      ├─ candles.mjs              K 线聚合与内存分桶
      ├─ store.mjs                JSONL 按小时分片落盘
      └─ probes/                  采集层
         ├─ index.mjs             平台分发 + 跨平台 CPU 差分
         ├─ util.mjs              execFile 与数字解析
         ├─ darwin.mjs            macOS 实现
         ├─ win32.mjs             Windows 实现的 Node 侧
         └─ collector.ps1         Windows 常驻采集脚本
```

采集层按平台分发，`server.mjs` 只消费统一形状的返回值。拿不到的指标返回 `null`，
页面渲染成「—」。

所有外部命令走 `execFile` 加参数数组，不拼接 shell。

## 测试环境

| 项目 | 版本 |
| --- | --- |
| MiniMax Code | 3.1.10-preview.5019 |
| macOS | 26.5.2（Darwin 25.5.0，arm64） |
| Node.js | 22.22.2 |
| PowerShell | 7.6.9（Windows 分支验证用） |

## 许可证

[GPL-3.0](LICENSE) © 2026 李白

