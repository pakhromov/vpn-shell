# vpn-shell

Run a shell or a single command inside an isolated network namespace routed through WireGuard. All traffic from inside the namespace goes through the VPN while the rest of the system is unaffected.

## Demo

https://github.com/user-attachments/assets/997ea10c-0e7f-423a-988f-35eb2f943354

The shell on the right pane is never affected by anything happening in the left pane, showing that the VPN is fully isolated to the namespace.
On the left:
- `vpn-shell` opens a VPN shell - handshake completes, traffic inside routes through ProtonVPN (Amsterdam, NL), right pane stays on the real IP
- After `exit` the namespace is torn down and left pane returns to the real IP
- `vpn-shell curl...` (with a provided command) runs a single `curl` through the VPN and after it is done - the shell is back to the real IP

## How it works

`vpn-shell` creates a Linux network namespace, brings up a WireGuard interface inside it configured from a `.conf` file, and launches a shell (or command) as the original user inside that namespace. When the shell or command exits, the namespace and interface are torn down automatically.

Because it uses a network namespace rather than routing table tricks, the VPN is fully isolated: only the shell or command you explicitly launch inside it goes through the VPN. Everything else on the system continues using the normal network.

The WireGuard interface is configured in the host namespace first (so hostname endpoints resolve via host DNS), then moved into the isolated namespace. Routes are installed from the `AllowedIPs` field of the config, so both full-tunnel (`0.0.0.0/0`) and split-tunnel configurations are supported.

Before handing control to the user, the script sends a probe packet to trigger a WireGuard handshake and waits up to 15 seconds for it to complete. If no handshake is confirmed the script exits with an error rather than silently opening a shell with no VPN.

## Usage

```
vpn-shell [-c wg.conf] [command...]
```

Opens an interactive shell inside the VPN namespace. If a command is given it runs that command instead and exits when it finishes.

### Config selection

The config file is resolved in this order:

1. The `-c` flag: `vpn-shell -c ~/.config/wireguard/work.conf`
2. The `VPN_CONF` environment variable: `export VPN_CONF=~/.config/wireguard/work.conf`
3. Auto-detection: if exactly one `.conf` file exists in `~/.config/wireguard/` (respecting `$XDG_CONFIG_HOME`) it is used automatically

If multiple configs are found and none is specified, the script exits with an error listing the directory.

### Examples

```sh
# Interactive VPN shell using the auto-detected config
vpn-shell

# Interactive VPN shell with an explicit config
vpn-shell -c ~/.config/wireguard/mullvad.conf

# Run a single command through the VPN
vpn-shell curl -s https://ipinfo.io/json

# Run a single command with an explicit config
vpn-shell -c ~/.config/wireguard/work.conf ssh internal-host

# Use a fixed config via environment variable
export VPN_CONF=~/.config/wireguard/work.conf
vpn-shell curl -s https://ipinfo.io/json
```

### Dependencies

| Package | Purpose |
|---|---|
| `bash` | shell interpreter |
| `wireguard-tools` | `wg` for configuring the WireGuard interface |
| `iproute2` | `ip` for namespace and routing management |
| `iputils` | `ping` for triggering the initial handshake |
| `util-linux` | `runuser` for dropping privileges inside the namespace |
| `sudo` | privilege escalation for namespace setup |

## Requirements

- Linux kernel with WireGuard support (5.6+)
- A standard WireGuard `.conf` file compatible with `wg-quick`, as exported by ProtonVPN, Mullvad, or similar (single-peer configs only)
- `sudo` rights for the invoking user (needed to create and configure network namespaces)

## Supported config fields

| Field | Required | Notes |
|---|---|---|
| `PrivateKey` | yes | |
| `Address` | yes | Comma-separated, IPv4 and IPv6 |
| `DNS` | no | Comma-separated IPs; search domains are not supported |
| `PublicKey` | yes | |
| `Endpoint` | yes | `host:port`; hostnames are resolved via host DNS |
| `AllowedIPs` | yes | Comma-separated CIDRs; determines routes inside the namespace |
| `PresharedKey` | no | |
| `PersistentKeepalive` | no | Optional; forwarded to WireGuard if set |

Multi-peer configs are not supported and are rejected with an error.

## Environment

The shell launched inside the namespace starts with a clean environment containing only:

| Variable | Source |
|---|---|
| `HOME`, `USER`, `LOGNAME`, `SHELL` | Derived from `getent passwd` |
| `PATH` | The invoking user's original `$PATH` |
| `TERM` | Preserved from the calling session |
| `DISPLAY`, `WAYLAND_DISPLAY` | Preserved if set |
| `XDG_RUNTIME_DIR`, `DBUS_SESSION_BUS_ADDRESS` | Preserved if set |
| `XDG_CONFIG_HOME` | Preserved if set |
| `LANG`, `LC_ALL` | Preserved if set |

The shell's startup files (`.zshrc`, `.bashrc`, etc.) are sourced normally, so your prompt, aliases, and functions are available as usual.
