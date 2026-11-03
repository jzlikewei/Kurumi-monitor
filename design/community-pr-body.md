## 本机大盘 · Local Market Board

A system monitor for MiniMax Code that renders the machine as a Chinese A-share trading
terminal. CPU, GPU, temperature, memory, disk, network and every running process become
listed stocks with live candlesticks, moving averages, an index strip, a watchlist, a
leaderboard and a trade tape.

Package: `plugins/jzlikewei/kurumi-monitor/` · plugin ID `kurumi-monitor` · v1.1.0 · GPL-3.0

## Tested environment

| Item | Version |
| --- | --- |
| MiniMax Code | 3.1.10-preview.5019 |
| macOS | 26.5.2 (Darwin 25.5.0, arm64) |
| Node.js | 22.22.2 |
| PowerShell | 7.6.9, used to exercise the Windows branch |

The package declares `surface.window`, so it needs MiniMax Code 3.1.2 or later. This is
named under `## Tested environment` in the README.

## What was verified

On macOS, installed and opened in MiniMax Code:

- All 44 codes render with live values (34 fixed metrics plus per-core CPU), and the
  温度 board lists its three codes with empty values, as documented.
- One collection round across the six macOS probes takes 44–58 ms.
- A 20-entry watchlist survives a restart of the runtime.
- Selecting a process that later exits shows `PID xxx 已退出` in the quote header.
- Console reports no error; no failed network request.
- `npm run check` reports `plugins/jzlikewei/kurumi-monitor: OK` (0 errors, 0 warnings).

The Windows branch was exercised with a stub `powershell.exe` feeding recorded frames:
29 assertions covering memory segments, GPU utilisation and VRAM, GPU temperature and
power draw, ACPI thermal zone, disk, network and the process table. The PowerShell
collector itself was run on macOS under PowerShell 7.6.9 to confirm the child-process
link and the newline-delimited JSON framing.

**Not yet verified:** the Windows branch has not run on physical Windows hardware. The
README states this under "Platform support".

## Data & access

No `agentAccess`, `toolAccess` or `deviceAccess`. No browser permission. No outbound
network request, no remote font, no CDN, no telemetry.

- **Spawns** (all via `execFile` with an argument array): `vm_stat`, `ioreg`,
  `netstat`, `ps`, `sysctl` on macOS; one long-lived `powershell.exe` running the
  bundled `collector.ps1`, plus `nvidia-smi` when present, on Windows.
- **Reads**: the process table and system counters returned by those commands, plus its
  own `settings.json`.
- **Writes**: samples and events under the system temporary directory
  (`$TMPDIR/kurumi-monitor/` or `%TEMP%\kurumi-monitor\`), sharded as
  `ticks/YYYYMMDD-HH.jsonl` plus `events.jsonl`; and `settings.json` in its own state
  directory. Measured at about 88 KB per minute, roughly 253 MB over 48 hours.
- **Terminates processes**: 一键清仓 sends `SIGTERM`, or `SIGKILL` with 强制清仓 ticked,
  after a confirmation dialog. The button is enabled only for processes the current user
  can signal, checked with `process.kill(pid, 0)`. The app's own process and the host
  process are permanently protected.
- **Agent turns, conversations, tools, microphone**: none.

## Credits

The interface language and interaction spec were inspired by
[PerfTicker](https://github.com/QiuSui1145/PerfTicker) by QiuSui (QiuSui_Studio), a
single-file C# program under GPL-3.0. This package is an independent implementation in
JavaScript and Node with no shared source; GPL-3.0 is kept as a tribute. The README
carries the full attribution.
