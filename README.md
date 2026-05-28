# dev-env-guide

Teach Claude to find SDKs via env vars instead of blindly scanning the filesystem. Auto-configures PATH on install, cleans up stale entries on uninstall.

## Why

AI agents default to brute-force filesystem search when locating tools — slow, noisy, and often wrong. They rarely configure PATH after installing a tool, and never clean up after uninstalling. This skill gives Claude a smarter workflow:

- **Find tools fast** — check environment variables first, then PATH, filesystem last
- **Auto-configure on install** — add to PATH and set SDK env vars so the tool stays discoverable
- **Clean up on uninstall** — remove stale PATH entries and env vars
- **Cross-platform** — PowerShell on Windows, Bash on macOS/Linux

## Install

```bash
# In Claude Code
/plugin install dev-env-guide@local
```

Or copy the `SKILL.md` into your project's `.claude/skills/` directory.

## Supported SDKs

The skill knows the canonical environment variables for 20+ development tools including Java, Python, Node.js, Go, Rust, .NET, Android SDK, Flutter, CUDA, CMake, Qt, and more.

## License

MIT
