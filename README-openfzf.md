# openfzf — OpenCode Session Browser

fzf-based TUI to browse, filter, and launch OpenCode sessions with optional VPN namespace isolation.

## Features

- **Two-level session browser** — list deduplicated sessions (level 1), expand forks (level 2)
- **Agent filtering** — show only `build`/`plan` by default; `--all`, `--plan`, `--build`, `--explore`, `--general`
- **VPN namespace selector** — pick Local or an existing VPN namespace; create new VPNs inline
- **Recency ordering** — sessions opened last appear first (tiny SQLite cache)
- **Color-coded dates** — green <2d, yellow <7d, red >7d (based on `time_updated`)
- **Adaptive columns** — title/dir widths auto-adjust to terminal size
- **Kitty integration** — opens sessions in new tabs (Local) or new windows (namespace)
- **Persistent menu** — fzf stays open after launching a session; Esc in level 2 returns to level 1
- **`--no-vpn` flag** — skip the space selector entirely, always open locally

## Requirements

- Bash 4+
- [fzf](https://github.com/junegunn/fzf)
- [OpenCode](https://opencode.ai)
- SQLite3 (for session DB + recency cache)
- sudo with NOPASSWD (for namespace operations)
- kitty (optional, for terminal launching)
- iproute2, iptables, openvpn, socat (optional, for VPN creation)

## Installation

```bash
cp openfzf ~/.local/bin/
chmod +x ~/.local/bin/openfzf
```

## Usage

```bash
openfzf                         # default: show build/plan sessions
openfzf --all                   # show all agents
openfzf --plan                  # only plan sessions
openfzf --build                 # only build sessions
openfzf --explore               # only explore sessions
openfzf --general               # only general sessions
openfzf --no-vpn                # skip VPN space selector
openfzf --no-vpn --plan         # combine flags
```

### Key bindings

| Key | Action |
|---|---|
| Enter | Open selected session |
| Ctrl+E | Expand forks of selected session (show level 2) |
| Esc (level 1) | Exit |
| Esc (level 2) | Return to level 1 |
| Tab (New Session) | Create a new OpenCode session |

### VPN flow

1. After selecting a session, a space selector appears:
   - `Local` — open directly on the host
   - `● namespace` — namespace is running and VPN is connected
   - `○ namespace` — namespace exists but VPN is disconnected
   - `Crear New VPN` — create a new VPN namespace (select .ovpn, auto-name, start)
2. Inactive namespaces (dimmed) cannot be selected.
3. When a namespace is chosen, opencode runs inside it via:
   ```
   sudo ip netns exec <ns> runuser -u <user> -- opencode --session <id>
   ```
4. `fzf` stays open after launching; press Esc to navigate back.

## Architecture

```
openfzf
├── Level 1 (build_l1)
│   SQL: GROUP BY canonical_title, ordered by last_opened DESC
│   Shows: title, agent, date, directory, VPN tag, fork count
│   "+ New Session" always at top
│
├── Level 2 (build_l2 / gen_l2_entries)
│   SQL: WHERE title LIKE 'canonical%', ordered by last_opened DESC
│   Shows: title, agent, date+time, directory
│
├── Space selector (select_space)
│   fzf menu: Local / running namespaces / Crear New VPN
│   Extracts space name via cut -d'|' -f1
│
└── Session launcher (open_session / new_session)
    ├── Local → kitty @ launch (tab) — opencode --session <id>
    └── Namespace → kitty --hold (new window) — ip netns exec ... opencode --session <id>
```

### Recency cache

A small SQLite database at `~/.cache/openfzf/opened.db` tracks session open times:

```sql
CREATE TABLE opened (id TEXT PRIMARY KEY, last_opened INTEGER);
```

The main OpenCode DB (at `~/.local/share/opencode/opencode.db`) is ATTACHed and LEFT JOINed — no data is duplicated.

## Configuration

### Environment variables

| Variable | Default | Description |
|---|---|---|
| `OPENFZF_ALL` | empty | Show all sessions (skip dedup, flat list) |
| `OPENFZF_DEBUG` | empty | Print debug info to stderr |
| `OPENFZF_LOG` | empty | Write debug log to file |

### Config files

| Path | Purpose |
|---|---|
| `~/.local/share/opencode/opencode.db` | OpenCode session database |
| `~/.cache/openfzf/opened.db` | Recency cache |
| `~/.config/vpns/spaces` | VPN namespace registry (created by vpns) |
| `~/Downloads/VPNS/*.ovpn` | OpenVPN config files |

## Troubleshooting

**fzf doesn't open:** ensure `fzf` is installed and you're in a terminal (not piped).

**"Cannot open network namespace" with extra text:** the `select_space` parser was fixed to use `cut -d'|' -f1`. Make sure you have the latest version.

**VPN creation fails:** check that OpenVPN configs exist in `~/Downloads/VPNS/`, that `sudo` has NOPASSWD, and that `openvpn` is installed.

**Session doesn't appear in list:** the default filter only shows `build` and `plan` agents. Use `--all` to see everything.

**Port conflicts in namespace:** each namespace uses `10.200.1.0/24` — only one namespace with VPN can run at a time without manual reconfiguration.
