# IsletDesk

**A supervised Windows desktop for any AI agent.**

IsletDesk gives an AI agent a real Windows desktop of its own — a child session with its own
screen, pointer and keyboard focus — while you keep working on yours. A small status island
at the top of the screen reports what the agent is doing, without you having to watch it.

[中文说明](README.zh-CN.md)

---

## Why

Agents that use a computer need a screen, a pointer and a foreground window. Sharing yours is
painful: the pointer jumps away mid-sentence, windows steal focus, and you cannot tell what
the agent changed.

IsletDesk separates the two. The agent drives a Windows **child session** (the seat) through
[Anode](https://github.com/skulitom/Anode); your desktop keeps its own pointer, keyboard focus
and windows. Nothing the agent clicks or types reaches your session.

## Features

- **Dynamic Island status surface** — a top-centred capsule that expands by itself when an
  agent starts an action and collapses when it goes idle. Shows agent name, action, state,
  elapsed time and source, without ever taking focus from your window.
- **Real input isolation** — verified: 40 characters typed and a click inside the seat leave the
  user's cursor position and foreground window unchanged.
- **MCP first** — the agent talks to `IsletDesk.Bridge.exe`, an MCP stdio server that exposes
  41 desktop tools (click, type, screenshot, windows, processes, inspect, jobs, gamepad …).
- **One-click peek and take-over** — click the island to open the seat viewer, click again to
  hide it and return focus to where you were.
- **Safe by default** — the bridge is a machine-global named pipe guarded by the server's
  kernel ACL; status events carry no arguments, results or secrets; MCP calls expire after
  10 minutes; a desktop lease serialises multiple agents instead of letting them fight over
  one pointer.
- **Offline-friendly installer** — downloads a pinned Anode build (v0.11.4) and verifies its
  SHA-256; the Codex config is written as a single managed, reversible block.

## Requirements

- Windows 10 or 11, **Pro / Enterprise / Education**, x64. Home edition cannot host the
  child session IsletDesk relies on.
- Administrator rights **once**, for installation: enabling the Remote Desktop listener and
  child sessions is a system-level change.
- The installer downloads Anode from its GitHub releases (pinned version + SHA-256).

> IsletDesk is **not a security sandbox**. The seat signs in as you and shares your files,
> account and network. Treat every MCP client you connect as something you would let use your
> own desktop, because that is exactly what it gets.

## Install

1. Download `IsletDesk-Setup-vX.Y.Z-win-x64.zip` from
   [Releases](https://github.com/DaoFaZiran2010/IsletDesk/releases) and extract it. The archive is
   flat — three executables at the root, nothing else:

   | File | Role |
   |---|---|
   | `IsletDesk.Setup.exe` | installer (`install` / `check` / `uninstall`) |
   | `IsletDesk.exe` | status island + tray app |
   | `IsletDesk.Bridge.exe` | MCP stdio server that agents connect to |

2. Recommended — verify the download before running it:

   ```bat
   certutil -hashfile IsletDesk-Setup-vX.Y.Z-win-x64.zip SHA256
   ```

   Compare the output with the matching line in `SHA256SUMS.txt` from the same release.

3. Run, from a normal (non-elevated) terminal:

   ```bat
   IsletDesk.Setup.exe install --accept-rdp-change
   ```

   `--accept-rdp-change` is required: the installer enables the Remote Desktop host and
   child sessions, and records the previous values so `uninstall` can restore them.

4. Verify the result at any time:

   ```bat
   IsletDesk.Setup.exe check
   ```

5. `IsletDesk.Setup.exe uninstall` removes the runtime, the Codex managed block and the
   autostart entry, and restores the previous RDP / child-session settings.

The installer places the app in `%LOCALAPPDATA%\Programs\IsletDesk` and adds a per-user
autostart entry. Start it from the Start Menu, the desktop shortcut, or the tray icon.

> **Upgrading from AgentOrb (v0.1.10 and earlier).** Those releases are published under the old
> name: the archive is `AgentOrb-Setup-v0.1.10-win-x64.zip` and the executables inside are
> `AgentOrb.Setup.exe`, `AgentOrb.exe` and `AgentOrb.Bridge.exe`. The rename is breaking — the old
> autostart entry and the old `[mcp_servers.agentorb]` Codex block are **not** migrated. Run
> `AgentOrb.Setup.exe uninstall` first, then install IsletDesk. Never run both side by side: they
> would compete for the same seat.

## Use it with Codex

The installer appends a managed block to `%USERPROFILE%\.codex\config.toml`, leaving any model
provider configuration untouched:

```toml
# >>> IsletDesk managed block >>>
[mcp_servers.isletdesk]
command = "C:\\Users\\<you>\\AppData\\Local\\Programs\\IsletDesk\\IsletDesk.Bridge.exe"
# <<< IsletDesk managed block <<<
```

Restart the Codex client, then just ask for desktop work:

> Open Notepad in the agent desktop, type "hello from the seat", and take a screenshot.

What you will see:

1. Codex asks permission to call the tool;
2. the island expands and reports the action, agent name and elapsed time;
3. the seat (not your desktop) receives every click and keystroke.

Any other MCP-capable agent can be wired the same way — point it at `IsletDesk.Bridge.exe`.
Agents that cannot speak MCP can drive the seat through the Anode CLI; IsletDesk's island keeps
reporting through the same pipe.

## The island

| Action | Result |
|---|---|
| Do nothing | The island reports agent actions, then collapses |
| Click | Opens the seat viewer (you get control) |
| Click again | Hides the viewer and restores your previous foreground window |
| Right-click | Start desktop / view desktop / emergency stop / exit |

The island window itself is `WS_EX_NOACTIVATE | WS_EX_TOOLWINDOW`: it never steals focus.
The viewer deliberately *does* take focus when opened, because that is how you take over.

## Architecture

```
┌──────────────────┐     MCP over stdio      ┌─────────────────────┐
│  Agent (Codex,   │ ◄─────────────────────► │ IsletDesk.Bridge.exe │
│  Claude, …)      │                         │  (MCP server)       │
└──────────────────┘                         └──────────┬──────────┘
                                                       │ named pipe
                                                       │ \\.\pipe\isletdesk-status-v1
                                            ┌──────────▼──────────┐
                                            │ IsletDesk.Desktop    │
                                            │  status island UI   │
                                            └──────────┬──────────┘
                                                       │ CLI
                                            ┌──────────▼──────────┐
                                            │ Anode (seat)        │
                                            │ child session + RDP │
                                            └─────────────────────┘
```

| Project | Responsibility |
|---|---|
| `IsletDesk.Core` | Status contracts, JSON format, tool-name mapping |
| `IsletDesk.Anode` | Anode CLI wrapper (status / show / hide / start / stop) |
| `IsletDesk.Desktop` | WPF island, tray, viewer toggle, named-pipe status server |
| `IsletDesk.Bridge` | MCP stdio server, pipe client, call tracking and redaction |
| `IsletDesk.Setup` | Installer: RDP/child-session setup, Anode provisioning, Codex config |
| `tests/IsletDesk.Tests` | 24 dependency-free tests covering pipes, MCP and installer logic |

## Build from source

Requires the .NET 8 SDK on Windows.

```bat
dotnet build IsletDesk.sln -c Release
dotnet run --project tests\IsletDesk.Tests\IsletDesk.Tests.csproj -c Release
```

The test project has no external test framework: it is a plain console runner, so the build
has zero NuGet dependencies.

## Status

Early but working: **v0.2.0** is installed and verified on Windows 11 — input isolation, the status
island, the 41 MCP tools and the seat lease were all exercised against real MCP clients.
Not yet done: code signing (SmartScreen will warn), auto-update, crash recovery, multiple
concurrent seats.

Release packaging is automated: pushing a `v*` tag runs the build, the 24 tests, and the upload of
three assets (`IsletDesk-v*-win-x64.zip`, `IsletDesk-Setup-v*-win-x64.zip`, `SHA256SUMS.txt`). The
step-by-step checklist is in [docs/release-checklist.md](docs/release-checklist.md); the
open-source readiness list is in [docs/opensource-readiness.md](docs/opensource-readiness.md).

## License

MIT — see [LICENSE](LICENSE). Chosen to match Anode (the runtime dependency), keep adoption
friction low, and avoid patent-clause complexity for a desktop automation tool of this size.
Third-party notices: [THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md).
