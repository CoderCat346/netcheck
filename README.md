# netcheck

A field checklist for a network you just met.

You land on a new Wi-Fi, a hotel LAN, a friend's router, a client's office. `netcheck`
walks you down the same list every time — what am I attached to, who else is here, does
the path work, what is exposed, is any of it careless — and leaves a Markdown report
behind.

It is deliberately a checklist, not a scanner: one step at a time, plain language,
readable output, and no pretending when the machine it runs on cannot see the network.

```
  netcheck 1.0 — field checklist for a network you just met
  scans are active traffic: only use this where you are authorised
```

## What it does

Eight steps, each one usable on its own:

| # | Step | What you get |
|---|------|--------------|
| 1 | **Where am I?** | Host, user, addresses, routes, gateway, DNS resolvers, subnets, local listening sockets |
| 2 | **Link and Wi-Fi** | Wireless interface, SSID, signal strength with a verdict, channel, nearby APs, wired carrier/speed |
| 3 | **Who is on this network?** | Passive neighbour table, then an active sweep (`nmap -sn`, ping, or TCP connect fallback) with IP/hostname/MAC/vendor table |
| 4 | **Is the path healthy?** | Gateway reachability, DNS resolution, internet egress, MTU/PMTU, captive-portal detection |
| 5 | **What is exposed?** | Port scan with version detection, per-port risk verdicts, aggregated results for ranges |
| 6 | **Is it secure?** | TLS protocol/certificate/expiry checks, HTTP headers and redirects, old server banners, SSH auth methods, open DNS resolver, gateway admin UI |
| 7 | **SSH toolbox** | Hosts from `~/.ssh/config`, local keys and agent, host-key fingerprints, auth probe, then connect or `ssh-copy-id` |
| 8 | **Write the report** | Markdown report with findings table and a next-steps checklist, plus the raw command log |

Findings are collected across steps and de-duplicated, so step 6's summary and the final
report both reflect everything seen so far.

## Requirements

- **Bash 4.2+** (associative arrays, `mapfile`, `[[ ]]` regex).
- **Linux.** It reads `/sys/class/net`, `ip`, `/etc/resolv.conf`. It works on macOS only in
  the loosest sense; don't.
- Optional tools, all discovered at runtime:

  | Tool | Package | Used for |
  |------|---------|----------|
  | `nmap` | `nmap` | host discovery, port/version scanning |
  | `dig` | `bind` | DNS checks, open-resolver test |
  | `traceroute` | `traceroute` | path inspection |
  | `iw` | `iw` | Wi-Fi link, signal, channel |
  | `nmcli` | `networkmanager` | nearby AP list |
  | `ethtool` | `ethtool` | wired link speed/duplex |
  | `ss` | `iproute2` | local listening sockets |
  | `ssh`, `ssh-keygen`, `ssh-copy-id` | `openssh` | step 7 |
  | `curl`, `openssl` | `curl`, `openssl` | TLS and HTTP checks |

  Missing tools are reported at startup with a ready-to-paste `pacman -S` line; nothing
  hard-fails because of them.

## Install

```sh
git clone https://github.com/CoderCat346/netcheck.git
install -m 755 netcheck/netcheck ~/.local/bin/netcheck
```

Make sure `~/.local/bin` is on your `PATH`. There are no other dependencies to install
and no config file to write.

## Usage

```sh
netcheck                      # interactive checklist (menu)
netcheck --all                # run steps 1–7 without prompts
netcheck --quick              # steps 1–4 only: where am I / link / hosts / path
netcheck --step 5             # run a single step
netcheck --step 5 --target 10.0.0.1
netcheck --step 3 --subnet 192.168.1.0/24
netcheck --step 5 --full      # all 65535 TCP ports (slow)
netcheck --step 5 --ports 1-1024
netcheck --yes --all          # accept every confirmation (still asks for targets)
netcheck --out ~/notes/trip   # report base directory
```

| Flag | Meaning |
|------|---------|
| `-a`, `--all` | Run steps 1–7 in order |
| `-q`, `--quick` | Run steps 1–4 only |
| `-s`, `--step N` | Run one step (1–8) |
| `-t`, `--target HOST` | Default target for steps 5, 6, 7 |
| `--subnet CIDR` | Default subnet for step 3 |
| `--ports SPEC` | `nmap -p` value for step 5 (e.g. `1-1024`, `22,80,443`) |
| `--full` | Scan all 65535 TCP ports in step 5 |
| `-y`, `--yes` | Assume yes for confirmations |
| `--no-color` | Plain output (also honours `NO_COLOR`) |
| `--out DIR` | Report base directory (default `~/netcheck-reports`) |
| `-V`, `--version` | Print version |
| `-h`, `--help` | Usage |

The interactive menu shows live state at the top — host, subnet, gateway, hosts found,
findings so far — so you can leave it open while you work:

```
────────────────────────────────────────────────────────────────────
  host:laptop   net:192.168.1.24/24   gw:192.168.1.1   hosts:7   findings:3
  1) Where am I?              host, addresses, routes, DNS, listening ports
  2) Link and Wi-Fi           SSID, signal, channel, link speed
  3) Who is on this network?  neighbours + active sweep
  4) Is the path healthy?     gateway, DNS, internet, MTU, captive portal
  5) What is exposed?         port + version scan
  6) Is it secure?            TLS, HTTP, SSH, risky services
  7) SSH toolbox              fingerprints, auth probe, connect
  8) Write the report
  a) Run the whole checklist
  q) Quit
```

## Risk classification

Ports are graded by what their presence usually means, not by CVEs:

| Severity | Ports | Reason |
|----------|-------|--------|
| **HIGH** | 21, 23, 69, 512–514 | Cleartext protocols: credentials and data travel in the clear |
| **HIGH** | 139, 445 | SMB: classic ransomware and credential-theft target |
| **HIGH** | 3389 | RDP: brute-force and RCE target |
| **HIGH** | 5900, 5901 | VNC: frequently unauthenticated |
| **HIGH** | 2375, 2376 | Docker API: full host takeover if unauthenticated |
| **MEDIUM** | 3306, 5432, 1433, 1521, 27017, 6379, 9200, 11211 | Database/cache reachable: check bind address and auth |
| **MEDIUM** | 80, 8000, 8080, 8888 | HTTP without TLS: traffic can be read or modified |
| **MEDIUM** | 161 | SNMP: default community strings are common |
| **MEDIUM** | 111 | rpcbind: enumerate with `rpcinfo -p` |
| **INFO** | 22, 443, 8443, 53, anything else | Expected services; checked in step 6 |

Additional findings come from behaviour rather than ports:

- **HIGH** — obsolete TLS version offered, expired certificate, DNS resolution failing,
  HTTPS egress failing.
- **MEDIUM** — SSH accepting passwords, certificate expiring within days, old web-server
  banner, open DNS resolver, no default route, captive portal suspected.
- **INFO** — self-signed certificate (normal on routers), gateway admin UI reachable.

## Reports

Every run writes to a timestamped directory:

```
~/netcheck-reports/20261111-142530-4821/
├── report.md            # the deliverable
├── raw.log              # every command and its output
├── body.md              # step sections before the findings table
├── discovery.gnmap      # host discovery results
├── scan-<target>.nmap   # per-target scan output
└── ports-<target>.tsv   # parsed port/service/version
```

`report.md` looks like this:

```markdown
# Network field report

- **When:** 2026-11-11 14:25:31 CET
- **Host:** laptop (you)
- **System:** Linux 6.12.1 aarch64
- **Local subnet:** 192.168.1.24/24
- **Gateway:** 192.168.1.1

## 1. Host and interfaces
...

## Findings

| Severity | Finding | Why it matters |
|---|---|---|
| HIGH | 192.168.1.1: 23 telnet | cleartext protocol: credentials and data travel in the clear |
| MEDIUM | 192.168.1.1: 80 http | HTTP without TLS: traffic can be read or modified |

## Next steps

- [ ] Confirm every open port above is expected, owned and patched.
- [ ] Replace cleartext services (telnet, ftp, http) with TLS variants or turn it off.
- [ ] Keep management interfaces (SSH, router UI, databases) behind a VPN or ACL.
- [ ] Check the gateway firmware and change any default admin password.
- [ ] Re-run netcheck after changes and diff the two reports.
```

The report is also written on exit — Ctrl-C, `q`, or a step that bails early — so a
half-finished run still leaves you something. The directory is created with `umask 077`
(mode 700): reports contain network topology and service versions, so they are private
by default.

## Safety

- **Only scan networks you own or have permission to test.** Port scanning and service
  detection are active traffic; on a network you don't control that is at best rude and at
  worst illegal.
- Confirmations guard the noisy parts: subnet sweeps, scans of ranges larger than 16
  addresses, and full 65535-port scans all ask first. `--yes` skips them; targets are
  still asked for.
- Step 6 uses a deliberately harmless SSH probe: `PreferredAuthentications=none` against
  user `netcheck`, so no credentials are tried and no account is touched. It learns the
  accepted auth methods from the rejection message.
- TLS checks read certificates and try protocol versions; they do not exploit anything.
- Nothing is uploaded. There is no telemetry, no config file, no daemon.

## Limitations

`netcheck` tries hard to tell you when it cannot see, rather than inventing results:

- **Raw sockets.** Host discovery and nmap need raw-socket access. In a container or
  proot sandbox without it, nmap reports no routes, remote ICMP fails, and every TCP
  connect looks open. The script probes a TEST-NET address (`192.0.2.1:9`) to detect
  exactly this, and then refuses to run the scan rather than printing a fake port list.
- **Fallbacks are labelled.** If nmap is unusable, discovery falls back to parallel
  `ping`, then to a TCP connect sweep of common ports. The fallback port scan covers 26
  common ports with `/dev/tcp` and no version detection — it is labelled as such in the
  output and report.
- **No IPv6 scanning.** Steps 1 and 4 report IPv6 addresses and routes, but sweeps and
  scans are IPv4 only.
- **UDP is not scanned.** Everything on the port side is TCP.
- **Version detection is `--version-light`.** Fast and good enough to spot a service;
  not a vulnerability scanner. No CVE lookup is performed.
- **One host at a time for step 6.** Checks run against a single target; for a range,
  scan first, then point step 6 at the interesting host.
- **Wi-Fi details need `iw`/`nmcli`.** In a sandbox with no wireless interface exposed to
  Linux, step 2 says so and moves on.

## Exit behaviour

The report is written in an `EXIT` trap. Interrupting is safe; it costs you the step in
progress, not the report. Raw output is appended as it happens, so even a killed run
leaves `raw.log` usable.

## License

MIT. See [LICENSE](LICENSE).
