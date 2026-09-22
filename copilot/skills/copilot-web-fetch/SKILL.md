---
name: copilot-web-fetch
description: 使用 Copilot Plus 抓取并以干净的 Markdown 读取特定网页（URL）的完整内容。当用户分享链接，或要求你打开、阅读、总结某个页面时使用 — 不用于开放式网络搜索。需要有效的 Copilot Plus 许可证；没有时请改用你自己的抓取工具。
license: Copilot Plus
metadata:
  copilot-enabled-agents: opencode, claude, codex
  copilot-builtin-version: "6"
---

# Copilot 网页抓取

通过 Copilot Plus 抓取网页内容为 Markdown。

## How to run

Find the absolute path to this SKILL.md file on disk, then run the script next
to it that matches the operating system. No extra runtime is needed — `sh`
(macOS/Linux) and `cmd`/PowerShell (Windows) are always present.

On macOS or Linux:

```bash
sh "/absolute/path/to/this/skill/directory/web-fetch.sh" "<url-to-fetch>"
```

On Windows, run the `.cmd` wrapper. In PowerShell you must prefix it with the
call operator `&` (PowerShell treats a quoted path on its own as a string and
won't run it); from cmd, run the quoted path without the `&`:

```powershell
& "/absolute/path/to/this/skill/directory/web-fetch.cmd" "<url-to-fetch>"
```

Both print the result to stdout.

## If Copilot Plus is not active

If the script exits saying Copilot Plus is unavailable, do NOT retry it. Do what
the message says: fall back to your own equivalent built-in capability to handle
the request when you have one (otherwise tell the user it's unavailable) — never
refuse or block the user. Only mention upgrading or renewing Copilot Plus when
the script's message explicitly invites it, and keep any such note short and
friendly.

## Self-Host mode

Self-Host search providers do not provide a common full-page fetch contract. If
the script reports that Self-Host mode is active, never use an agent-native web
fetch tool. Use `copilot-web-search` when search results can answer the request;
otherwise tell the user that fetching the page is unavailable.

