# vpns — VPN Namespace Manager

Bash script to run OpenVPN inside isolated Linux network namespaces with full lifecycle management, ADB forwarding, and optional OpenCode serve integration.

## How it works

Each VPN creates its own isolated network stack:

```
┌───────────────────────────────────────────────────┐
│ Host                                               │
│  ┌──────────────────────┐   ┌──────────────────┐  │
│  │ iptables MASQUERADE  │   │ socat             │  │
│  │ 10.200.1.0/24 → eth0 │   │ 127.0.0.1:5037   │  │
│  └──────────┬───────────┘   │ → 10.200.1.2:5037│  │
│             │               │ 127.0.0.1:4096   │  │
│    veth-host (10.200.1.1)   │ → 10.200.1.2:4096│  │
│             │               └──────────────────┘  │
└─────────────┼─────────────────────────────────────┘
              │ veth pair
┌─────────────┼─────────────────────────────────────┐
│ Namespace   │                                      │
│    veth-ns (10.200.1.2/24)                         │
│    default via 10.200.1.1                          │
│    DNS: 1.1.1.1                                    │
│    ┌────────────┐                                  │
│    │ OpenVPN    │ → tun0 → VPN provider            │
│    │ (daemon)   │                                  │
│    └────────────┘                                  │
│    ┌────────────────┐                              │
│    │ opencode serve │ (optional, port 4096)        │
│    └────────────────┘                              │
└───────────────────────────────────────────────────┘
```

1. **Network namespace** — `ip netns add` isolates the VPN's routes and interfaces
2. **veth pair** — connects the namespace to the host (`10.200.1.0/24`)
3. **NAT** — iptables MASQUERADE routes namespace traffic to the internet
4. **DNS** — per-namespace `/etc/netns/<name>/resolv.conf` (kernel feature)
5. **OpenVPN** — runs as daemon inside the namespace, creates `tun0`
6. **ADB forwarding** — two socat proxies bridge ADB (port 5037) between host and namespace
7. **OpenCode serve** (optional) — `opencode serve` runs inside the namespace, socat forwards port 4096 to host

## Requirements

| Tool | Required | For |
|---|---|---|
| `openvpn` | Yes | VPN client daemon |
| `ip` (iproute2) | Yes | Namespaces, veth, routes |
| `iptables` | Yes | NAT masquerade |
| `sudo` | Yes | Privilege escalation (NOPASSWD recommended) |
| `curl` | Yes | Detect public VPN IP via ifconfig.me |
| `socat` | No | ADB + OpenCode port forwarding |
| `fzf` | No | Interactive .ovpn/space picker |
| `fish` | No | Space selection UI |
| `runuser` | No | Drop privileges inside namespace |

## Installation

```bash
cp vpns ~/.local/bin/
chmod +x ~/.local/bin/vpns
mkdir -p ~/Downloads/VPNS   # place .ovpn files here
mkdir -p ~/.config/vpns      # spaces registry
touch ~/.config/vpns/spaces
```

## CLI commands

```bash
vpns                          # interactive: pick .ovpn → create/choose space → start
vpns start                    # same as above
vpns list / ls                # list .ovpn files in ~/Downloads/VPNS/
vpns restart / reboot         # pick a saved space → clean → restart
vpns stop                     # pick a saved space → clean → remove from registry
vpns status                   # show status of the last-used namespace
vpns shell                    # root shell inside the last namespace
vpns enter                    # interactive bash as your user inside the last namespace
vpns serve [port]             # start opencode serve + port forward in the last namespace
vpns attach [port]            # connect via opencode attach http://127.0.0.1:<port>
vpns help / -h / --help       # show usage
vpns <cmd> [args...]          # run an arbitrary command inside a selected namespace
```

### Interactive mode (default)

1. Shows saved spaces with status
2. fzf picker of .ovpn files:
   - Green `●` + IP → VPN is connected
   - Red `●` + offline → namespace exists, VPN not connected
   - Bare filename → never used before
3. fzf picker: create a new space or reuse an existing one
4. Namespace is created/cleaned and VPN starts

### serve + attach (OpenCode integration)

```bash
vpns serve                    # start opencode serve on 4096 inside namespace
opencode attach http://127.0.0.1:4096   # from another terminal
```

Or the shortcut:
```bash
vpns attach                   # calls opencode attach automatically
```

## Configuration files

| Path | Format | Purpose |
|---|---|---|
| `~/.config/vpns/spaces` | `name\|config\|ip\|ISO_timestamp` | Space registry (one per line) |
| `~/.config/vpns/last` | plain text: namespace name | Last-used namespace |
| `~/Downloads/VPNS/*.ovpn` | OpenVPN config | VPN connection files |
| `/etc/netns/<name>/resolv.conf` | `nameserver 1.1.1.1` | Per-namespace DNS |

### Space registry format

```
my_vpn|office.ovpn|203.0.113.45|2026-06-27T14:30:00
work|work.ovpn|offline|2026-06-26T10:00:00
```

Fields: `namespace_name | config_filename | vpn_ip_or_offline | ISO8601_timestamp`

## PID lifecycle

All PID files live in `/tmp/` and are cleaned up by `ns_cleanup`:

| File | Process |
|---|---|
| `/tmp/openvpn-<ns>.pid` | OpenVPN daemon |
| `/tmp/openvpn-<ns>.log` | OpenVPN log |
| `/tmp/socat-host-<ns>.pid` | ADB socat on host |
| `/tmp/socat-ns-<ns>.pid` | ADB socat in namespace |
| `/tmp/opencode-serve-<ns>.pid` | `opencode serve` |
| `/tmp/socat-opencode-<ns>.pid` | OpenCode port forwarder |

**Startup order** (`ns_start`):
1. `ns_cleanup` → kill old processes, delete old namespace/veth
2. Create namespace + veth pair + IPs + routes
3. Write DNS config
4. Enable IP forwarding + iptables NAT
5. Start ADB socat forwarders (if socat available)
6. Launch OpenVPN daemon with `--writepid`
7. Poll up to 20s for `tun0`
8. Detect VPN public IP via `ifconfig.me`
9. Save to spaces registry

**Teardown order** (`ns_cleanup`):
1. Kill OpenVPN
2. Kill ADB socats (host + namespace)
3. Kill OpenCode serve + its socat
4. Delete namespace
5. Delete veth interface

## Security

- All privileged operations escalate via `sudo` (NOPASSWD recommended)
- Processes inside the namespace drop privileges to the original user via `runuser -u $ORIG_USER`
- `detect_original_user()` captures the real user whether invoked directly or via `sudo`
- Namespace is deleted on cleanup — no leftover network state

## Troubleshooting

**"openvpn no instalado":** install OpenVPN.

**VPN doesn't connect (timeout on tun0):** check the log at `/tmp/openvpn-<ns>.log`. Common issues: wrong .ovpn path, missing auth credentials, network unreachable.

**No .ovpn files found:** place your OpenVPN configs in `~/Downloads/VPNS/`.

**"Namespace no existe":** start a VPN first with `vpns start` or `vpns serve`.

**Port already in use:** only one namespace can use port 4096/5037 at a time. Stop the other namespace first.

**fzf not showing:** fzf is optional — the script falls back to auto-selecting if only one option exists.

**Fish not available:** the `select_space` and `select_saved_space` functions use fish internally. Without fish, these menus fall back to bash (limited functionality).
