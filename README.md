# Cybermes MCP Server — Setup & Maintenance Guide

Setting up the [Zyrexnn/Cybermes](https://github.com/Zyrexnn/Cybermes) offensive-security
MCP server in **Claude Desktop** on Linux, so that all **207 skill playbooks** and the
knowledge base load correctly.

> ⚠️ **Authorization note:** Cybermes is an offensive / bug-bounty toolkit. Only point its
> tools at targets you are explicitly authorized to test.

---

## 1. How it works (the key insight)

The npm package (`cybermes-mcp`) and the downloaded Go binary **only contain the launcher +
server executable**. They do **not** bundle the skill playbooks or the knowledge base.

- The **209 skill directories** (`skills/`) and the **knowledge base** (`knowledge/` —
  PayloadsAllTheThings, HackTricks, Claude-BugHunter, strix-skills, hack-skills) live
  **only in the GitHub repo**.
- At runtime, the Go binary locates that content via the **`CYBERMES_DIR`** environment
  variable.
- If `CYBERMES_DIR` is not set (and no `skills/` folder is in the working directory), the
  server silently loads **0 skills** and returns an **empty knowledge base** — no error,
  just empty results.

So a working setup is **three things**: the binary, a local copy of the repo data, and a
config that points `CYBERMES_DIR` at that data.

---

## 2. Prerequisites

- **Node.js / npm** (provides `npx`) — required.
- **git** — to clone and update the skill/knowledge data.
- **Claude Desktop**.
- Optional: Go 1.22+ and Python 3.11+ (only for building from source / some utilities).

---

## 3. Setup steps

### Step 1 — Get the server binary

Either let `npx` manage it, or use the native binary directly.

**Option A — npx (self-updates the binary, simplest):**

```bash
npx -y cybermes-mcp install --global
```

This downloads the native binary to `~/.cybermes/bin/` and writes a Claude Desktop config
entry.

**Option B — direct binary (most reliable, no per-launch download):**

The binary gets cached at `~/.cybermes/bin/cybermes-mcp-v<version>-linux-amd64`. Point the
config straight at it (see Step 3). This avoids the `npx` re-download race on every launch.

### Step 2 — Clone the skill + knowledge data to a permanent location

```bash
git clone --depth 1 https://github.com/Zyrexnn/Cybermes.git ~/.cybermes/data
```

Verify:

```bash
ls ~/.cybermes/data/skills | wc -l      # ~209
ls ~/.cybermes/data/knowledge           # Claude-BugHunter, hacktricks, PayloadsAllTheThings, ...
```

> Do **not** keep this in a temp directory — it must persist across reboots.

### Step 3 — Configure Claude Desktop

Config file location:

- **Linux:** `~/.config/Claude/claude_desktop_config.json`
- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

Back it up first, then set the `cybermes` entry. **The critical part is the `env` block with
`CYBERMES_DIR`.**

**Direct-binary form (recommended)** — replace the version with whatever is in `~/.cybermes/bin/`:

```json
{
  "mcpServers": {
    "cybermes": {
      "command": "/home/<you>/.cybermes/bin/cybermes-mcp-v3.5.0-linux-amd64",
      "args": [],
      "env": {
        "CYBERMES_DIR": "/home/<you>/.cybermes/data"
      }
    }
  }
}
```

**npx form (if you prefer auto-updating binary):**

```json
{
  "mcpServers": {
    "cybermes": {
      "command": "npx",
      "args": ["-y", "cybermes-mcp"],
      "env": {
        "CYBERMES_DIR": "/home/<you>/.cybermes/data"
      }
    }
  }
}
```

### Step 4 — Restart Claude Desktop

**Fully quit** (not just close the window) and reopen so it picks up the new `env` block.

### Step 5 — Verify

Ask Claude: *"list cybermes skills"*. You should see **`Showing 200 of 207`** (the list caps
at 200; pass a `filter` to narrow). Knowledge search (`cybermes_search_knowledge`) should now
return real snippets instead of empty results.

Command-line smoke test (bypasses Claude Desktop):

```bash
printf '%s\n' \
'{"jsonrpc":"2.0","id":0,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
'{"jsonrpc":"2.0","method":"notifications/initialized"}' \
'{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"cybermes_list_skills","arguments":{"limit":5}}}' \
| CYBERMES_DIR=~/.cybermes/data ~/.cybermes/bin/cybermes-mcp-v3.5.0-linux-amd64 2>/dev/null \
| grep -o 'Showing [0-9]* of [0-9]*'
```

Expected: `Showing 5 of 207`.

---

## 4. Keeping skills updated

The skills/knowledge are just a git checkout, so update them with a pull:

```bash
git -C ~/.cybermes/data pull
```

Then restart Claude Desktop (the server reads the folder at startup).

**To update the server binary too:**

```bash
# npx form updates automatically on next launch. For the direct-binary form, re-run:
npx -y cybermes-mcp install --global
# then point the config at the new ~/.cybermes/bin/cybermes-mcp-v<newversion>-linux-amd64
```

**Optional — automate the data refresh** (weekly, via cron):

```bash
(crontab -l 2>/dev/null; echo "0 9 * * 1 git -C $HOME/.cybermes/data pull --quiet") | crontab -
```

---

## 5. Troubleshooting

| Symptom (in Claude Desktop MCP log) | Cause | Fix |
| --- | --- | --- |
| `spawn cybermes-mcp ENOENT` / "Executable not found in $PATH" | Config uses a bare `cybermes-mcp` command that isn't on `PATH` | Use the absolute binary path (Step 3) or the `npx` form |
| `spawn ETXTBSY` / `chmod ... .tmp` errors on launch | Harmless race — Claude Desktop spawns the server several times at once and they collide downloading/chmod-ing the `npx` binary | Ignore (it self-recovers), or switch to the direct-binary form to avoid downloads |
| `Total available skills: 0` / empty knowledge results | `CYBERMES_DIR` not set, so the binary can't find the repo data | Add the `env.CYBERMES_DIR` block (Step 3) and restart |
| Skill shows `>-` as its description | That skill's `SKILL.md` has no parseable description line | Cosmetic only — the full playbook content still loads when opened |

---

## Appendix — Installing Playwright Chromium (standard method)

> **Note:** This was **not** performed as part of the Cybermes setup above. These are the
> standard, documented commands for installing Playwright's Chromium browser — included here
> for reference. Run them yourself, or ask Claude to run and verify them.

Playwright ships its own browser binaries; you install Chromium separately from the npm
package.

**Install the Chromium browser only:**

```bash
npx playwright install chromium
```

**Install Chromium + required OS libraries (Debian/Ubuntu/Kali):**

```bash
# installs the browser
npx playwright install chromium
# installs system dependencies the browser needs (needs sudo)
sudo npx playwright install-deps chromium
```

Browsers are cached under `~/.cache/ms-playwright/` on Linux.

**Verify the install:**

```bash
npx playwright install --dry-run chromium   # shows the resolved install path/version
ls ~/.cache/ms-playwright/                   # should list a chromium-<build> directory
```

**Using the Playwright MCP server in Claude Desktop** (add alongside `cybermes`):

```json
{
  "mcpServers": {
    "cybermes": {
      "command": "/home/<user>/.local/bin/cybermes-mcp",
      "args": [],
      "env": {
        "CYBERMES_DIR": "/home/<user>/.cybermes/data"
      }
    },
    "playwright": {
      "command": "npx",
      "args": [
        "@playwright/mcp@latest",
        "--config",
        "/home/<user>/burp-playwright.json"
      ]
    }
  },
[...rest of config...]
}
```

**Finally, configure `/home/<user>/burp-playwright.json` (where `server` is your running burp instance):**

```json
{
  "browser": {
    "browserName": "chromium",
    "launchOptions": {
      "proxy": {
        "server": "http://127.0.0.1:8085"
      }
    },
    "contextOptions": {
      "ignoreHTTPSErrors": true
    }
  }
}
```

The Playwright MCP server will use the Chromium you installed above.
