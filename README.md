<![CDATA[# ⚡ CORTEX RAM GUARD

**Sovereign macOS memory pressure daemon.**

A lightweight, zero-dependency watchdog that prevents macOS swap death by enforcing graduated memory pressure response, per-app RSS budgets, and predictive swap exhaustion alerts.

Built for power users running heavy IDE + browser + agent workloads on memory-constrained machines.

---

## The Problem

macOS handles memory pressure by swapping aggressively to disk. On machines with 16-18GB RAM running Electron IDEs, multiple browser instances, and background agents, swap can silently balloon to 30GB+, grinding the system to a halt.

By the time you notice, `WindowServer` is at 30fps and every click takes 2 seconds.

**RAM Guard fixes this by acting before you notice.**

## How It Works

```
┌─────────────────────────────────────────────────┐
│              CORTEX RAM GUARD v3.0              │
├──────────┬──────────┬──────────┬────────────────┤
│ 🟢 GREEN │ 🟡 YELLOW│ 🟠 ORANGE│ 🔴 RED         │
│ Monitor  │ Cull     │ Kill+    │ Emergency      │
│ only     │ excess   │ Purge    │ Purge          │
│ 120s     │ 60s      │ 30s      │ 15s            │
├──────────┴──────────┴──────────┴────────────────┤
│  Thresholds scale to physical RAM automatically │
│  18GB → Y:9GB  O:14.4GB  R:21.6GB              │
│  32GB → Y:16GB O:25.6GB  R:38.4GB              │
└─────────────────────────────────────────────────┘
```

### Graduated Response

| Level | Trigger | Actions |
|-------|---------|---------|
| 🟢 **GREEN** | Swap < threshold | Monitor only. Restore Spotlight if throttled. |
| 🟡 **YELLOW** | Swap > 0.5× RAM | Cull excess processes. Kill stale renderers. Enforce RSS budgets. |
| 🟠 **ORANGE** | Swap > 0.8× RAM | Kill hot CPU processes. Purge disk cache. macOS notification. |
| 🔴 **RED** | Swap > 1.2× RAM | Emergency purge. Halve all RSS budgets. Throttle Spotlight. |

### Key Features

- **Per-app RSS budgets** — Set max memory per app group (Node, Python, Renderer, Comet). Largest processes killed first when budget exceeded.
- **Predictive alerts** — Linear extrapolation warns you N minutes before RED.
- **Memory leak detection** — Flags monotonically increasing swap across sample window.
- **Process ancestry logging** — Every kill logs `PID(cmd)←PPID(parent)` chain for forensics.
- **Self-health monitoring** — Daemon auto-restarts if its own RSS exceeds limit.
- **Editable whitelist** — Never kills processes matching patterns in `whitelist.conf`.
- **Hot-reload config** — Changes to `config.conf` picked up without restart.
- **Batch notifications** — Groups alerts to avoid notification spam.
- **Process snapshots** — Periodic TSV dumps of top-30 processes for trend analysis.
- **Log rotation** — Auto-compresses old logs, deletes after 14 days.

## Installation

```bash
# Clone
git clone https://github.com/borjamoskv/cortex-ram-guard.git
cd cortex-ram-guard

# Install (creates launchd agent + starts daemon)
./cortex-ram-guard.sh install
```

That's it. The daemon survives reboots and auto-restarts on crash.

### Manual install

```bash
# Copy files
mkdir -p ~/.cortex/ram-guard
cp cortex-ram-guard.sh config.conf whitelist.conf ~/.cortex/ram-guard/
chmod +x ~/.cortex/ram-guard/cortex-ram-guard.sh

# Install daemon
~/.cortex/ram-guard/cortex-ram-guard.sh install
```

### Optional: CLI alias

```bash
ln -sf ~/.cortex/ram-guard/cortex-ram-guard.sh ~/bin/ramguard
```

## Usage

```bash
ramguard status      # Current memory state + per-app RSS
ramguard purge       # Manual immediate purge
ramguard logs        # Tail today's log
ramguard stats       # 7-day kill/purge history
ramguard snapshot    # Forensic process dump
ramguard install     # Install/restart daemon
ramguard uninstall   # Stop and unload daemon
```

### Example output

```
═══════════════════════════════════════════════
  CORTEX RAM GUARD v3.0
═══════════════════════════════════════════════
  RAM:       18GB  │  Free: 75MB
  Swap:      3570MB  │  🟢 GREEN
  Thresholds: Y=9000 O=14400 R=21600
───────────────────────────────────────────────
  Comet            3 procs    150MB RSS  (budget 1500MB)
  Node            42 procs    890MB RSS  (budget 2000MB)
  Renderer         8 procs    620MB RSS  (budget 1800MB)
  Python           2 procs    129MB RSS  (budget 1200MB)
───────────────────────────────────────────────
  Total procs: 500
  Today: kills=27 purges=2 freed=13400MB
═══════════════════════════════════════════════
```

## Configuration

Edit `~/.cortex/ram-guard/config.conf`:

```bash
# Pressure thresholds (multiplier × RAM_GB in MB)
YELLOW_MULT=500     # YELLOW at RAM×500 MB
ORANGE_MULT=800     # ORANGE at RAM×800 MB
RED_MULT=1200       # RED at RAM×1200 MB

# Per-app RSS budgets (MB) — 0 to disable
BUDGET_COMET_MB=1500
BUDGET_NODE_MB=2000
BUDGET_RENDERER_MB=1800
BUDGET_PYTHON_MB=1200

# CPU thresholds for killing hot processes
CPU_KILL_PYTHON=60
CPU_KILL_NODE=70

# Process count limits
MAX_NODE_PROCS=80
MAX_RENDERER_PROCS=12

# Predictive alert: warn when projected time to RED < N minutes
PREDICT_ALERT_MIN=15
```

Changes are picked up automatically (no restart needed).

## Whitelist

Edit `~/.cortex/ram-guard/whitelist.conf` to protect processes from being killed:

```
# Main app — never kill
MyApp.app/Contents/MacOS/MyApp

# System daemons
WindowServer
coreaudiod
```

## Files

```
~/.cortex/ram-guard/
├── cortex-ram-guard.sh    # Main daemon + CLI
├── config.conf            # Tunable configuration
├── whitelist.conf         # Protected processes
├── guard.pid              # Daemon PID
├── stats-YYYY-MM-DD.json  # Daily statistics
├── logs/
│   ├── ram-guard-YYYY-MM-DD.log
│   └── stdout.log
└── snapshots/
    └── snap-HHMM.tsv      # Process snapshots
```

## Requirements

- macOS (tested on Sonoma/Sequoia, Apple Silicon)
- zsh (default macOS shell)
- No dependencies. No brew packages. No Python runtime (except for stats display).

## Uninstall

```bash
ramguard uninstall
rm -rf ~/.cortex/ram-guard
rm ~/Library/LaunchAgents/com.cortex.ram-guard.plist
```

## License

MIT

---

*Built by [borjamoskv](https://github.com/borjamoskv) — because 18GB should be enough for anybody.*
]]>
