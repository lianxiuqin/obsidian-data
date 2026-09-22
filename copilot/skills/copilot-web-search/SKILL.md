---
name: copilot-web-search
description: 使用 Copilot Plus 或已配置的自托管搜索服务商在网络上搜索最新信息。当用户要求联网搜索、在网上查找资料，或需要超出库范围的最新事实时使用。凡是涉及用户自己笔记的内容，优先读取库。
license: Copilot Plus 或自托管
metadata:
  copilot-enabled-agents: opencode, claude, codex
  copilot-builtin-version: "6"
---

# Copilot 网络搜索

通过 Copilot 进行网络搜索，并返回用户查询的结果。

## How to run

Find the absolute path to this SKILL.md file on disk, then run the script next
to it that matches the operating system. No extra runtime is needed — `sh`
(macOS/Linux) and `cmd`/PowerShell (Windows) are always present.

On macOS or Linux:

```bash
sh "/absolute/path/to/this/skill/directory/web-search.sh" "<your search query>"
```

On Windows, run the `.cmd` wrapper. In PowerShell you must prefix it with the
call operator `&` (PowerShell treats a quoted path on its own as a string and
won't run it); from cmd, run the quoted path without the `&`:

```powershell
& "/absolute/path/to/this/skill/directory/web-search.cmd" "<your search query>"
```

Both print the result to stdout.

## If Copilot Plus is not active

If the script exits saying Copilot Plus is unavailable, do NOT retry it. Do what
the message says: fall back to your own equivalent built-in capability to handle
the request when you have one (otherwise tell the user it's unavailable) — never
refuse or block the user. Only mention upgrading or renewing Copilot Plus when
the script's message explicitly invites it, and keep any such note short and
friendly.

