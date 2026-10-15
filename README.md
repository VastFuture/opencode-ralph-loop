# 🔁 opencode-ralph-loop

> Compatible with the OpenCode V2 plugin API while retaining the V1 `server()` entrypoint during migration.

**[English](README.md)** · **[简体中文](README_CN.md)**

A lightweight [OpenCode](https://opencode.ai) plugin that implements the [Ralph Loop](https://ghuntley.com/ralph/) pattern — a self-referential completion loop that keeps your AI agent working until a task is truly 100% done.

Also includes **ebuilder agent** support — switch to the ebuilder agent and auto-continuation activates automatically, no commands needed.

Inspired by the Ralph Loop and Sisyphus features in [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode), reimplemented as a standalone plugin with **zero dependencies**.

## ✅ OpenCode Version Support

| OpenCode version | Status | Notes |
|---|---|---|
| V2 `>= 2.0.0` | ✅ Primary | Uses the V2 `{ id, setup }` plugin definition; verified with OpenCode `2.0.29` using the recommended symlink installation below |
| V1 `>= 1.0.0` | ⚠️ Migration compatibility | Keeps the legacy `server()` entrypoint; new installations should use OpenCode V2 |

The installation and verification commands below target OpenCode V2. After installing or upgrading the plugin, restart the OpenCode background service rather than only reopening the TUI.

---

## 🔄 How It Works

### Ralph Loop (command-based)

```
User issues /ralph-loop or /ulw-loop command
  → Plugin writes loop state to .ralph-loop.state.json
  → Agent executes the task
  → session.idle event fires
  → Plugin checks Agent output for <promise>DONE</promise>
    → ✅ Found: Loop complete! Toast notification.
    → ❌ Not found: Inject continuation prompt → Agent continues
    → Repeat until completion or max iterations
```

### ebuilder Agent (automatic)

```
User switches to ebuilder agent in TUI
  → chat.message hook detects agent === "ebuilder"
  → Plugin writes loop state to .ebuilder.state.json
  → Agent executes the task
  → session.idle event fires
  → Plugin checks Agent output for <promise>DONE</promise>
    → ✅ Found: Loop complete!
    → ❌ Not found: Inject continuation prompt → Agent continues
  → User switches back to build/plan → ebuilder state auto-cleared → Loop stops
```

---

## ✨ Features

| Feature | Description |
|---------|-------------|
| 🔁 **`/ralph-loop`** | Standard completion loop (max 100 iterations) |
| 🚀 **`/ulw-loop`** | Ultrawork mode with maximum intensity (max 500 iterations) |
| 🛑 **`/cancel-ralph`** | Cancel an active loop (works for both modes) |
| 🤖 **ebuilder agent** | Auto-continuation when switched to ebuilder agent (max 500 iterations) |
| 🎯 **Completion promise detection** | Scans Agent output for `<promise>DONE</promise>` |
| 🔔 **Toast notifications** | Real-time iteration and completion updates |
| 🛡️ **In-flight protection** | Prevents duplicate idle event handling |
| ⚙️ **Custom parameters** | `--max-iterations=N` and `--completion-promise=TEXT` |

---

## 📦 Installation

### Option 1: Global source symlink (best practice)

This is the installation layout verified with OpenCode `2.0.29`. Keep the Git checkout as the source of truth and expose only `src/index.js` through OpenCode's global single-file plugin directory:

```bash
git clone https://github.com/VastFuture/opencode-ralph-loop.git \
  ~/.opencode/plugins/opencode-ralph-loop

cd ~/.opencode/plugins/opencode-ralph-loop
node tests/test-plugin.mjs

mkdir -p ~/.config/opencode/plugins
ln -sfn \
  ~/.opencode/plugins/opencode-ralph-loop/src/index.js \
  ~/.config/opencode/plugins/ralph-loop.js

opencode service restart
opencode api get /api/plugin
```

The API result must contain:

```text
id: opencode-ralph-loop
source.path: ~/.config/opencode/plugins/ralph-loop.js
state.status: active
```

Then confirm that `/ralph-loop`, `/ulw-loop`, and `/cancel-ralph` are available. The symlink keeps the installed plugin aligned with the checked-out source without copying files after every update.

> Do not add a root `index.js` under `~/.opencode/plugins/opencode-ralph-loop/` and do not also list this plugin in `opencode.json`. OpenCode scans plugin directories automatically; loading both the checkout and the symlink creates two instances and fails with `Duplicate plugin ID`.

Upgrade the recommended installation with:

```bash
cd ~/.opencode/plugins/opencode-ralph-loop
git pull --ff-only
node tests/test-plugin.mjs
opencode service restart
opencode api get /api/plugin
```

### Option 2: Direct file path

Reference the source file from the V2-native `plugins` field in `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugins": [
    "file:///path/to/ralph-loop/src/index.js"
  ]
}
```

Choose exactly one loading method. A local auto-discovered file, a configured path, and an npm package must not be enabled together.

### ebuilder agent configuration

To use the ebuilder agent for automatic continuation, add the agent configuration separately:

```json
{
  "agent": {
    "ebuilder": {
      "mode": "primary",
      "description": "Autonomous agent with auto-continuation.",
      "prompt": "You are ebuilder..."
    }
  }
}
```

## 🗂️ Project Structure

```
opencode-ralph-loop/
├── src/
│   └── index.js        # 插件入口
├── scripts/
│   └── cleanup.py      # 清理脚本（卸载前执行）
├── package.json
├── README.md
└── .gitignore
```

## 🧹 Uninstall

Before deleting the plugin, run the cleanup script to remove generated command entries:

```bash
# 1. Preview the cleanup
python scripts/cleanup.py --source ralph-loop --dry-run

# 2. Apply the cleanup
python scripts/cleanup.py --source ralph-loop

# 3. Remove the recommended global symlink
rm ~/.config/opencode/plugins/ralph-loop.js

# 4. Restart OpenCode
opencode service restart
```

If you used a configured-path installation instead, remove that source from `opencode.json`. Use `--dry-run` before cleanup when you want to preview the changes.

---

## 🎮 Usage

### 🤖 ebuilder Agent (recommended for long tasks)

1. Switch to the **ebuilder** agent in the TUI (Tab key or agent selector)
2. Type your task normally
3. The agent will work continuously without stopping
4. Switch back to **build** to stop auto-continuation

### 🔁 Ralph Loop Commands

```bash
# Start a standard loop
/ralph-loop "Refactor the auth module and ensure all tests pass"

# Start an ultrawork loop (maximum intensity)
/ulw-loop "Migrate all API clients to v2"

# With custom parameters
/ralph-loop "Build the dashboard" --max-iterations=50 --completion-promise=SHIPPED

# Cancel an active loop
/cancel-ralph
```

### 🏁 How the Agent Signals Completion

When the Agent has fully completed the task, it outputs:

```xml
<promise>DONE</promise>
```

The plugin detects this tag in the Agent's response and stops the loop. You can customize the promise text with `--completion-promise`.

---

## ⚙️ Configuration

No configuration file needed. Loop states are stored in `.ralph-loop.state.json` and `.ebuilder.state.json` in the project directory and automatically cleaned up when loops complete or are cancelled.

---

## 📊 Comparison with oh-my-opencode

| Feature | oh-my-opencode | opencode-ralph-loop |
|---------|---------------|-------------------|
| 📁 File count | ~25 TypeScript files | 1 JS file |
| 📦 Dependencies | Full OmO plugin | Zero dependencies |
| 👁️ Oracle verification | ✅ | ❌ |
| 🔄 Session reset strategy | ✅ | ❌ (continue only) |
| 🔧 Session recovery | ✅ | ❌ |
| 🛡️ Race condition mutex | ✅ (debounce) | ✅ (inFlight Set) |
| 🔁 `/ralph-loop` | ✅ | ✅ |
| 🚀 `/ulw-loop` | ✅ | ✅ |
| 🛑 `/cancel-ralph` | ✅ | ✅ |
| 🤖 ebuilder/Sisyphus agent | ✅ (Sisyphus) | ✅ (ebuilder) |
| ⏩ Auto-continuation on agent switch | ✅ | ✅ |
| 🔔 Toast notifications | ✅ | ✅ |
| ⚙️ Custom max iterations | ✅ | ✅ |
| 🎯 Custom completion promise | ✅ | ✅ |

---

## 📄 License

MIT
