# Notification Sounds for Claude Code

Add audio feedback to Claude Code — hear a sound when Claude starts thinking, when tools run, when replies finish, and more.

## How it works

The skill configures Claude Code hooks to randomly play audio files from organized folders. Hooks are shell-level commands — **zero token consumption, zero performance impact**.

## Install

1. Copy `notification-sounds/` into `~/.claude/skills/`
2. In Claude Code, type: `setup notification sounds`

## What you get

| Event | When | Example |
|-------|------|---------|
| Start thinking | You submit a prompt | "Thinking..." |
| Tool start | Claude starts writing/editing | "Writing..." |
| Tool end | Claude finishes a tool call | "Done!" |
| Reply end | Claude finishes responding | "Complete!" |
| Permission | Claude asks for permission | "Need approval" |

## Folder structure

```
~/Music/notification-sounds/
  think-start/       ← UserPromptSubmit
  edit-start/        ← PreToolUse
  edit-end/          ← PostToolUse
  reply-end/         ← Stop
  permission/        ← PermissionRequest
```

Drop `.wav` files into any folder — the hook randomly picks one each time.

## Concurrency

Default mode: **interrupt** — new sound kills the previous one, no overlapping playback. Can be changed to `queue`, `skip`, or `none` during setup.

## Commands

| Command | What it does |
|---------|-------------|
| `/notification-sounds` | Interactive setup |
| `--check` | Diagnose existing setup, verify hooks and folders |
| `--dry-run` | Preview changes without writing |
| `--remove` | Clean uninstall |
| `--dir <path>` | Custom audio folder location |
| `--formats mp3,wav` | Custom audio formats |

## Supported platforms

- **Windows** — PowerShell + Media.SoundPlayer (WAV, zero dependency)
- **macOS** — afplay (WAV/MP3/AAC, built-in)
- **Linux** — paplay → aplay → ffplay (auto-detect)
