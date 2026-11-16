# IsletDesk 发布检查清单

适用对象：维护者。目标版本线：0.x（早期版本，接口仍可能变）。

这份清单的存在原因：v0.2.0 改名完成后，代码和 README 都已经是 0.2.0，但**没有打 tag**，
于是 Releases 页面还挂着 v0.1.10 的 `AgentOrb-*.zip`，README 里写的
`IsletDesk-Setup-vX.Y.Z-win-x64.zip` 根本没有对应的下载物。发布是"代码 + 文档 + tag + 产物 + 仓库元信息"
五件事必须同时成立，缺一项就会出现这种错位。

---

## 0. 命名不变量

打 tag 之前先确认下表两侧一致。`release.yml` 里已经内置了 tag 与版本号的强校验，
但产物命名没有自动校验，只能人工对。

| 项目 | 固定值 |
|---|---|
| tag | `v<Version>`，必须与 `Directory.Build.props` 的 `<Version>` 完全相同 |
| Release 标题 | `IsletDesk v<Version>` |
| 运行时包 | `IsletDesk-v<Version>-win-x64.zip` |
| 安装包 | `IsletDesk-Setup-v<Version>-win-x64.zip` |
| 校验文件 | `SHA256SUMS.txt`（两行，格式 `<hash>  <文件名>`） |
| 安装包内可执行文件 | `IsletDesk.Setup.exe`、`IsletDesk.exe`、`IsletDesk.Bridge.exe`（平铺在 zip 根目录） |

历史命名（仅用于解释老 Release，不要在新版本复用）：v0.1.10 及更早为
`AgentOrb-*-v0.1.10-win-x64.zip` + `AgentOrb.Setup.exe` / `AgentOrb.exe` / `AgentOrb.Bridge.exe`。

---

## 1. 版本号与变更日志

- [ ] `Directory.Build.props`：`Version`、`FileVersion`、`AssemblyVersion`、`InformationalVersion`
      四处同步（前三者带 `.0`，如 `0.2.0` / `0.2.0.0`）。
- [ ] `CHANGELOG.md`：把 `[Unreleased]` 里的条目落到新版本小节，并写上日期：
      `## [0.2.0] - 2026-11-15`。
- [ ] 破坏性变更必须在 CHANGELOG 里写明"升级前需先卸载旧版"这一类动作。
- [ ] `README.md` / `README.zh-CN.md` 的安装章节与"当前状态"里的版本号已更新。

## 2. 本地预检

Windows 上需要 .NET 8 SDK。本机若没有全局 SDK，用仓库自带的 `.tools\dotnet\dotnet.exe`
替换下面的 `dotnet`。

```bat
dotnet build IsletDesk.sln -c Release
dotnet run --project tests\IsletDesk.Tests\IsletDesk.Tests.csproj -c Release
```

- [ ] 构建 **0 错误 0 警告**（警告也算不通过，别让 CI 替你发现）。
- [ ] 测试 **24/24 通过**（`IsletDesk.Tests` 是零依赖控制台运行器，不是 xunit）。
- [ ] 手工冒烟（仅当本次改动触及运行时/安装器/灵动岛时）：
      `IsletDesk.Setup.exe check` → 启动 → 灵动岛紧凑态 → MCP 客户端握手列出 41 个工具 →
      调一次 `seat_processes` → 单击岛体打开查看器、再点隐藏并确认前台窗口恢复。
- [ ] `git status` 干净：`artifacts/ bin/ obj/ .tools/` 被 `.gitignore` 排除，不应出现在提交里。

## 3. 打 tag 并推送

```bat
git push origin main
git tag -a v0.2.0 -m "IsletDesk v0.2.0"
git push origin v0.2.0
```

- [ ] **先推 main，再推 tag**。tag 指向的 commit 必须已经在 main 上，否则 Release 的
      "Source code" 与 main 对不上。
- [ ] 用 annotated tag（`-a`），保留发布说明与打 tag 的人。
- [ ] 一个版本只打一次 tag。真要重发，先 `git tag -d` + `git push --delete`，
      确认没有用户下载过旧产物再重打；已发布的 Release 优先发补丁版本而不是覆盖。

## 4. 跟踪工作流

`release.yml` 在 `push tags: v*` 时触发，依次执行：tag/版本校验 → Release 构建 + 24 测试 →
三份单文件 self-contained 发布 → 打包 + SHA256 → 创建 Release。

页面：`https://github.com/DaoFaZiran2010/IsletDesk/actions/workflows/release.yml`

- [ ] `Verify tag matches project version` 通过（失败说明版本号没 bump 或 tag 拼错）。
- [ ] 整个 job 绿。任一步红，先看日志再决定是修 tag 还是发补丁版本。

## 5. 验收发布产物

```bat
curl -sSL https://api.github.com/repos/DaoFaZiran2010/IsletDesk/releases/latest
```

- [ ] 附件**三个齐全**：`IsletDesk-v<ver>-win-x64.zip`、`IsletDesk-Setup-v<ver>-win-x64.zip`、
      `SHA256SUMS.txt`。安装包约 120 MB、运行时包约 96 MB 属正常量级（单文件自解压 + 压缩）。
- [ ] `prerelease: false` 且页面标记为 **Latest**（`prerelease: true` 不会进 latest，
      用户点进仓库看到的还是旧版本）。
- [ ] 下载 `SHA256SUMS.txt`，抽一个包里核对：
      `certutil -hashfile IsletDesk-Setup-v<ver>-win-x64.zip SHA256`，与文本里对应行一致。
- [ ] **核对包内文件名**（这一步就是本次错位的根源，别省）：

      ```bat
      unzip -l IsletDesk-Setup-v<ver>-win-x64.zip
      ```

      期望恰好三行：`IsletDesk.Bridge.exe`、`IsletDesk.Setup.exe`、`IsletDesk.exe`，
      都在根目录、无子目录。出现 `AgentOrb.*` 或出现 `anode\` 目录都说明打包脚本被动过，
      必须停下查 `release.yml` 的 `Package and hash` 步骤。
- [ ] Release 正文里的安装指引与 README 的安装章节一致（zip 名、可执行文件名、`--accept-rdp-change`）。

## 6. 发布后同步

- [ ] 仓库 **About**：description 一行说清"给 AI Agent 一个受监督的独立 Windows 桌面"，
      topics 覆盖检索词（`ai-agent`、`computer-use`、`mcp`、`windows`、`desktop-automation`、
      `remote-desktop`、`dotnet`、`wpf` 等）。
      **注意**：`GITHUB_TOKEN` 和 GitHub App 连接器都改不了仓库设置（会 403），
      需要用账号本人操作网页 Settings 或带 `administration: write` 的 PAT 调
      `PATCH /repos/{owner}/{repo}`。
- [ ] 旧 Release 保持原样（不要删 v0.1.10，它是唯一能解释"AgentOrb 是什么"的证据）。
- [ ] 若本次是改名/破坏性变更：README 顶部或安装章节保留"从旧版升级需先卸载"的提醒。
- [ ] `docs/opensource-readiness.md` 的状态行按需更新（发布流程已从手工改为 CI 时）。

## 7. 升级与回滚

- [ ] 安装器只处理**当前用户名**下的 `%LOCALAPPDATA%\Programs\IsletDesk`、自启动项和
      `[mcp_servers.isletdesk]` 受管理块；卸载会还原 RDP/子会话注册表值。
- [ ] 破坏性改名（如 AgentOrb → IsletDesk）**不迁移**旧自启动项与旧 Codex 块，
      必须在 Release 正文与 README 里写明 `旧版 Setup.exe uninstall` 前置步骤。
- [ ] 回滚方式是装回上一个 tag 的包，不是覆盖文件；`uninstall` → 装旧版。

## 8. 已知缺口（发布时如实告知，不要藏）

- 二进制**未签名**，SmartScreen 会拦截，README 与 Release 正文都要写。
- 无自动更新；用户需手动下载新包并 `uninstall` → `install`。
- 无崩溃恢复；seat 内的异常退出需要右键菜单"紧急停止"或 `anode stop`。
- 单 seat 租约：多 Agent 串行共用，不能并发抢占。
