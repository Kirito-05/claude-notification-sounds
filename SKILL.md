---
name: notification-sounds
description: >
  Set up notification sounds for Claude Code hooks. Use this skill when the user
  wants to add audio feedback to Claude Code events (tool use start/end, response
  complete, permission requests, thinking start), configure custom sound folders,
  check/diagnose existing hook sound setups, or remove notification sounds.
  Triggers on: "提示音", "notification sound", "hook sound", "audio feedback",
  "beep", "sound effect", "播放音频", "音效", "提示音效".
---

# Notification Sounds Setup

Set up, check, and manage audio notification hooks for Claude Code.

## How this works

The skill configures Claude Code hooks to randomly play WAV files from organized
folders whenever specific events fire. The hooks are shell-level commands — they
run outside Claude's context, consume **zero tokens**, and have **no performance
impact** on response generation.

## Workflow

Ask the user what they want to do:

1. **Setup** (default) — create folders and install hooks into `settings.json`
2. **Check** (`--check`) — diagnose existing setup, verify hooks and folders
3. **Dry run** (`--dry-run`) — preview what would change without writing
4. **Remove** (`--remove`) — clean uninstall

If unclear, ask: "Setup, check, dry-run, or remove?"

---

## Step 1: Ask scope — global or project?

**This is the first question. It is independent of where audio files live.**

| Scope | Settings file | Hook fires when |
|-------|--------------|-----------------|
| Global | `~/.claude/settings.json` | Any project, any Claude Code session |
| Project | `<project>/.claude/settings.json` | Only when working in this specific project |

Ask: "Should notification sounds work everywhere (global) or only in this project?"

### Global scope

| OS | Path |
|----|------|
| Windows | `$env:USERPROFILE\.claude\settings.json` |
| macOS/Linux | `~/.claude/settings.json` |

### Project scope

Use `<project>/.claude/settings.json` where `<project>` is the current working
directory (or ask the user which project).

---

## Step 2: Detect platform and set audio player

Detect the OS and use the corresponding player command template:

### Windows (PowerShell)
```powershell
$d='<FOLDER_PATH>';if(Test-Path $d){$f=ls "$d\*.wav" -ea 0|Get-Random;if($f){(New-Object Media.SoundPlayer $f.FullName).PlaySync()}}
```

**Why `PlaySync()` not `Play()`:** `Play()` loads audio on a background thread and
returns immediately. The PowerShell process exits before playback starts, killing
the sound. `PlaySync()` blocks until playback completes — but since the hook has
`"async": true`, Claude is not blocked. This is a hard-won lesson from real usage.

### macOS
```bash
d='<FOLDER_PATH>'; if [ -d "$d" ]; then f=$(find "$d" -name '*.wav' -maxdepth 1 | sort -R | head -1); [ -n "$f" ] && afplay "$f"; fi
```

### Linux
```bash
d='<FOLDER_PATH>'; if [ -d "$d" ]; then f=$(find "$d" -name '*.wav' -maxdepth 1 | sort -R | head -1); [ -n "$f" ] && (command -v paplay >/dev/null && paplay "$f" || command -v aplay >/dev/null && aplay "$f" || command -v ffplay >/dev/null && ffplay -nodisp -autoexit "$f"); fi
```

Tries `paplay` → `aplay` → `ffplay`, silently skips if none available.

### Custom audio formats

Default format is `*.wav`. If the user specifies `--formats mp3,wav,ogg`:

- **macOS/Linux**: `afplay`/`paplay` support most formats natively. Change the
  `find -name` pattern to use `-name '*.mp3' -o -name '*.wav' -o -name '*.ogg'`.
- **Windows**: `Media.SoundPlayer` only supports WAV. Fall back to
  `Add-Type -AssemblyName PresentationCore` + `MediaPlayer` for general formats,
  or recommend WAV-only for zero-dependency setup.

---

## Step 3: Choose concurrency mode

When Claude fires hooks rapidly (e.g. submit → tool use → tool use in quick
succession), multiple audio processes may overlap. Ask the user how to handle it:

| Mode | Behavior | Best for |
|------|----------|----------|
| **interrupt** (default) | Kill previous playback, start new one immediately | Instant feedback, no stacking |
| `queue` | Wait for previous to finish, then play next (max 5s wait) | Don't want to miss any sound |
| `skip` | If something is already playing, discard the new one | Dense operations, avoid pile-up |
| `none` | No coordination — processes may overlap | Minimalist, zero extra logic |

**Default: `interrupt`.** The user can change it later with `--check` or a partial
re-setup without recreating folders.

### interrupt implementation

Uses a PID lock file. Each hook process kills the previous one before playing.

**Windows:**
```powershell
$l='<ROOT>\.playlock';$o=try{gc $l -ea 0}catch{$null};if($o-and(Get-Process -Id $o -ea 0)){Stop-Process -Id $o -Force -ea 0};$d='<FOLDER>';if(Test-Path $d){$f=ls "$d\*.wav" -ea 0|Get-Random;if($f){$PID|Out-File $l;(New-Object Media.SoundPlayer $f.FullName).PlaySync();if((gc $l -ea 0)-eq$PID){Remove-Item $l -ea 0}}}
```

**macOS:**
```bash
l='<ROOT>/.playlock'; [ -f "$l" ] && kill $(cat "$l") 2>/dev/null; d='<FOLDER>'; if [ -d "$d" ]; then f=$(find "$d" -name '*.wav' -maxdepth 1 | sort -R | head -1); [ -n "$f" ] && { echo $$ > "$l"; afplay "$f"; [ "$(cat "$l" 2>/dev/null)" = "$$" ] && rm -f "$l"; }; fi
```

**Linux:**
```bash
l='<ROOT>/.playlock'; [ -f "$l" ] && kill $(cat "$l") 2>/dev/null; d='<FOLDER>'; if [ -d "$d" ]; then f=$(find "$d" -name '*.wav' -maxdepth 1 | sort -R | head -1); [ -n "$f" ] && { echo $$ > "$l"; (command -v paplay >/dev/null && paplay "$f" || command -v aplay >/dev/null && aplay "$f" || command -v ffplay >/dev/null && ffplay -nodisp -autoexit "$f"); [ "$(cat "$l" 2>/dev/null)" = "$$" ] && rm -f "$l"; }; fi
```

The PID lock file lives at `<ROOT>/.playlock` — one file for ALL hooks in this
setup. Each process writes its own PID, and before playing, kills whatever PID
is currently in the file. After playback, only cleans up if its own PID is still
the one in the file (not overwritten by a newer process).

### queue implementation

Uses a named Mutex with a timeout. Subsequent hooks wait up to 5 seconds.

**Windows:**
```powershell
$m=New-Object Threading.Mutex(0,'Global\ClaudeNotify');if($m.WaitOne(5000)){try{$d='<FOLDER>';if(Test-Path $d){$f=ls "$d\*.wav" -ea 0|Get-Random;if($f){(New-Object Media.SoundPlayer $f.FullName).PlaySync()}}}}finally{$m.ReleaseMutex();$m.Dispose()}
```

**macOS:**
```bash
l='<ROOT>/.playlock'; for i in $(seq 1 50); do if ! [ -f "$l" ] || ! kill -0 $(cat "$l") 2>/dev/null; then break; fi; sleep 0.1; done; d='<FOLDER>'; if [ -d "$d" ]; then f=$(find "$d" -name '*.wav' -maxdepth 1 | sort -R | head -1); [ -n "$f" ] && { echo $$ > "$l"; afplay "$f"; rm -f "$l"; }; fi
```

**Linux:**
```bash
l='<ROOT>/.playlock'; for i in $(seq 1 50); do if ! [ -f "$l" ] || ! kill -0 $(cat "$l") 2>/dev/null; then break; fi; sleep 0.1; done; d='<FOLDER>'; if [ -d "$d" ]; then f=$(find "$d" -name '*.wav' -maxdepth 1 | sort -R | head -1); [ -n "$f" ] && { echo $$ > "$l"; (command -v paplay >/dev/null && paplay "$f" || command -v aplay >/dev/null && aplay "$f" || command -v ffplay >/dev/null && ffplay -nodisp -autoexit "$f"); rm -f "$l"; }; fi
```

### skip implementation

Uses a Mutex with zero wait — if locked, silently skip.

**Windows:**
```powershell
$m=New-Object Threading.Mutex(0,'Global\ClaudeNotify');if($m.WaitOne(0)){try{$d='<FOLDER>';if(Test-Path $d){$f=ls "$d\*.wav" -ea 0|Get-Random;if($f){(New-Object Media.SoundPlayer $f.FullName).PlaySync()}}}}finally{$m.ReleaseMutex();$m.Dispose()}
```

**macOS:**
```bash
l='<ROOT>/.playlock'; if ! [ -f "$l" ] || ! kill -0 $(cat "$l") 2>/dev/null; then d='<FOLDER>'; if [ -d "$d" ]; then f=$(find "$d" -name '*.wav' -maxdepth 1 | sort -R | head -1); [ -n "$f" ] && { echo $$ > "$l"; afplay "$f"; rm -f "$l"; }; fi; fi
```

**Linux:**
```bash
l='<ROOT>/.playlock'; if ! [ -f "$l" ] || ! kill -0 $(cat "$l") 2>/dev/null; then d='<FOLDER>'; if [ -d "$d" ]; then f=$(find "$d" -name '*.wav' -maxdepth 1 | sort -R | head -1); [ -n "$f" ] && { echo $$ > "$l"; (command -v paplay >/dev/null && paplay "$f" || command -v aplay >/dev/null && aplay "$f" || command -v ffplay >/dev/null && ffplay -nodisp -autoexit "$f"); rm -f "$l"; }; fi; fi
```

### none mode

No concurrency control. Uses the bare platform command from Step 2 as-is.

`<ROOT>` in all templates above refers to the audio root directory chosen in Step
4b. `<FOLDER>` is the per-event sub-folder (`<ROOT>/think-start`, etc.).

---

## Step 4: Hook event mapping and folder structure

Default audio folder root: `~/Music/notification-sounds/`

| Hook Event | Folder | Trigger | Needs `matcher` |
|------------|--------|---------|-----------------|
| `UserPromptSubmit` | `think-start/` | User submits prompt, thinking begins | No |
| `PreToolUse` | `edit-start/` | Before each tool executes | Yes (`"*"` = all tools) |
| `PostToolUse` | `edit-end/` | After each tool completes | Yes (`"*"` = all tools) |
| `Stop` | `reply-end/` | Claude finishes responding | No |
| `PermissionRequest` | `permission/` | Permission dialog appears | No |

**Critical rule:** Only `PreToolUse` and `PostToolUse` get `"matcher": "*"`.
Adding `matcher` to other events (like `UserPromptSubmit`) **prevents the hook
from firing**. This was discovered through debugging a real non-triggering hook.

---

## Step 5: Setup — create folders and install hooks

### 5a: Confirm concurrency mode and events

The concurrency mode was chosen in Step 3. Confirm it with the user. Also ask
which events to enable (default: all 5).

### 5b: Ask for audio folder location

**This is a separate question from scope (Step 1).** The folder path tells the
hook where to look for audio files — it does NOT imply which settings.json to use.

Ask the user where to create the folder structure:

1. **Default** (recommended): `~/Music/notification-sounds/`
2. **Custom absolute path**: Any path, e.g. `D:\audio\claude-sounds\`
3. **Custom relative path**: e.g. `./sounds/` or `./.claude/audio/` (resolved to
   absolute immediately — ask for the project root if unclear)

**Always resolve to an absolute path before writing hook commands.** Why:
relative paths in hooks are fragile — they resolve against the process working
directory, which varies per session.

Example: user chooses project scope + default path → hooks go in
`<project>/.claude/settings.json`, audio folders at `~/Music/notification-sounds/`.
Hooks reference absolute paths like `C:\Users\...\Music\notification-sounds\edit-start\`.

### 5c: Create folder structure

Create the root directory and sub-folders for each enabled event. If folders
already exist, skip — don't overwrite any files.

### 5d: Read existing settings.json

If `settings.json` exists, read it. If it doesn't, start with `{}`.
**Always create a backup** before modifying: copy to `settings.json.bak` in the
same directory.

### 5e: Build hook configuration

Use the command template from **Step 3** matching the chosen concurrency mode
and platform. Replace `<ROOT>` with the absolute audio root path, `<FOLDER>` with
the per-event sub-folder path.

For each enabled event, build the hook object:

**Events WITHOUT matcher** (`UserPromptSubmit`, `Stop`, `PermissionRequest`):
```json
{
  "hooks": [
    {
      "type": "command",
      "command": "<mode-specific command with resolved paths>",
      "async": true
    }
  ]
}
```

**Events WITH matcher** (`PreToolUse`, `PostToolUse`):
```json
{
  "matcher": "*",
  "hooks": [
    {
      "type": "command",
      "command": "<mode-specific command with resolved paths>",
      "async": true
    }
  ]
}
```

### 5f: Merge into settings.json

Merge the new hooks into the existing `settings.json`:

- If `"hooks"` key doesn't exist, create it
- If an event already has hooks, **append** the audio hook to the existing array
  (don't overwrite other hooks the user may have configured)
- If the exact same command already exists, skip (idempotent)
- Write the merged result back to `settings.json`

### 5g: Report to user

After writing, report:
- Which events were configured
- Folder paths created
- Concurrency mode used
- Remind: **"If you added new hook event types (not just modified existing commands), reload Claude Code (Reload Window) for them to take effect."**

---

## Step 6: Check (`--check`) — diagnostic mode

This is a read-only operation. Do NOT modify any files.

### 6a: Check settings.json structure

- Read `settings.json`, verify hooks section exists
- For each expected event, verify it's present
- Verify `matcher` is only present on `PreToolUse`/`PostToolUse`
- Verify `"async": true` is set on all audio hooks
- Verify commands use `PlaySync()` not `Play()` (Windows)
- Detect concurrency mode from command signature and report it

### 6b: Check folders

For each configured folder path, verify:
- Folder exists
- Contains at least one audio file (matching configured format)
- Count and report the number of audio files

### 6c: Check command executability

Run the audio playback command directly (not through the hook) with a test file
from each folder. This verifies:
- The audio player is available
- The file path is valid
- No syntax errors in the command

Report pass/fail for each.

### 6d: Check hook event names (live)

**Search the web** for the latest Claude Code hooks documentation. Compare the
configured event names against current valid events. If the docs list events not
yet configured, suggest them to the user. If any configured event is no longer
valid, flag it.

### 6e: Summary report

Present results in a table including concurrency mode:

```
  think-start/      ✓ config  ✓ folder  ✓ 2 files  ✓ cmd  mode:interrupt
  edit-start/       ✓ config  ✓ folder  ✓ 1 file   ✓ cmd  mode:interrupt
  edit-end/         ✓ config  ✓ folder  ✗ empty    (skip) mode:interrupt
  reply-end/        ✓ config  ✓ folder  ✗ empty    (skip) mode:interrupt
  permission/       ✓ config  ✓ folder  ✗ empty    (skip) mode:interrupt

  Events: 5/5 valid | Mode: interrupt | Audio in 2/5 folders
```

If issues found, offer to fix them.

---

## Step 7: Dry run (`--dry-run`)

Same as setup but write nothing. Show exactly what would change:

- Which folders would be created
- What would be added to `settings.json` (show the JSON diff)
- Concurrency mode that will be used
- Whether existing hooks would be modified or new ones added
- Whether a reload would be needed

---

## Step 8: Remove (`--remove`)

### 8a: Remove hooks from settings.json

Read `settings.json`, remove only the audio notification hooks (those whose
command matches the notification-sounds pattern with PID lock or Mutex). Leave
other hooks untouched.

### 8b: Offer to remove audio folders

Ask the user if they also want to delete the audio folders and their contents.
Default: don't delete (they might want to keep their audio files).

### 8c: Report

Show what was removed from `settings.json` and confirm folder status.

---

## Safety guarantees

These are baked into every generated hook command:

- **`"async": true`** on every hook — Claude never waits for audio playback
- **`Test-Path` guard** (or `[ -d ]` on Unix) — if the folder doesn't exist
  (e.g., deleted by user), the command exits silently without error
- **Empty folder guard** — `Get-Random` on empty list returns `$null`, which
  the `if ($f)` check handles, skipping playback silently
- **No polling, no daemons, no persistent processes** — each hook fires once
  per event, plays one sound, then the process exits
- **Zero token consumption** — hooks are shell commands, not LLM prompts

---

## Post-setup: adding more sounds

Remind the user: to add more audio files, simply drop `.wav` files into the
corresponding folder. The hook randomly picks one on each trigger. No
configuration changes needed — the skill's job is done after setup.
