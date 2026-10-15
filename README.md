# SecBan

SecBan is an open source intrusion-detection and prevention framework for Linux that enforces IP-level bans at the firewall (`nftables`). Inspired by fail2ban, SecBan uses a modular architecture separating detection, state management, enforcement, and alert dispatching across dedicated daemons.

> **Project Status / Warning:**
> SecBan is newly developed software.
> - It requires further production testing before critical deployments.
> - APIs, configurations, and internal protocols remain subject to breaking changes.
> - Only Linux with `nftables` packet filtering is supported at this time.

## Architecture

SecBan consists of three primary components communicating via UNIX domain sockets or HTTP:

```
[ Log Files / Journald ]
          │
          ▼
┌──────────────────────┐
│  secban-log-monitor  │
└──────────┬───────────┘
           │ HTTP / UNIX socket
           ▼
┌──────────────────────┐       exec       ┌──────────────────────┐
│     secban-agent     ├─────────────────►│ Custom Shell Scripts │
└──────────┬───────────┘                  └──────────────────────┘
           │                      IPC
           │ nftables             UNIX socket
           ▼                                  ▼
[ Linux Kernel Firewall ]                 ┌──────────────────────┐
                                          │    secban-mailer     │
                                          └──────────┬───────────┘
                                                     │ SMTP
                                                     ▼
                                            [ Email Notification ]
```

### Components

1. **`secban-agent`** (Core Daemon)
   - Exposes RESTful HTTP endpoints over UNIX domain socket or TCP.
   - Enforces IP blocks directly in the kernel using `nftables`.
   - Manages offender thresholds, tracking registries, duration timeouts, and CIDR/IP allowlists.
   - Audits ban history in an SQLite database.
   - Dispatches ban events to custom executables and IPC UNIX domain sockets.
   - Runs unprivileged under user `secban-agent` with capability `CAP_NET_ADMIN`.

2. **`secban-log-monitor`** (Log Detection Daemon)
   - Streams log output using external tools (`journalctl`, `tail -F`).
   - Extracts IPv4 and IPv6 addresses via configurable regular expressions.
   - Evaluates lines against pattern matchers and ignore rules.
   - Reports offending IPs to `secban-agent` via HTTP or UNIX domain socket.
   - Includes `secban-checkmatch` CLI tool to test patterns against recent logs without reporting.

3. **`secban-mailer`** (Alert Notification Daemon)
   - Listens on a dedicated UNIX domain socket for IPC notification payloads from `secban-agent`.
   - Sends alert emails via SMTP with auto-discovery authentication.
   - Configurable TLS policy (`notsl`, `opportunistic`, `mandatory`).
   - Formats messages with built-in or custom Go plain text templates.
   - Includes `secban-mailer-emailcheck` CLI tool to validate SMTP setup.

## Key Design Principles

- **Direct Application Integration:** Applications (web apps, PAM modules, API gateways) can report offenders or ban IPs directly via the `secban-agent` REST API without writing or parsing logs.
- **Kernel-Level Performance:** Bans use `nftables` sets for minimal overhead and wire-speed packet dropping.
- **Least Privilege:** Services run as dedicated unprivileged users (`secban-agent`, `secban-log-monitor`, `secban-mailer`) isolated with systemd security sandboxing.
- **Extensibility:** Actions support arbitrary executables and decoupled IPC socket subscribers.

## Repositories

- [secban-agent](https://gitlab.com/secban/secban-agent)
- [secban-log-monitor](https://gitlab.com/secban/secban-log-monitor)
- [secban-mailer](https://gitlab.com/secban/secban-mailer)
