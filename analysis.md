# Cowrie Honeypot Forensic Analysis
**Date of Activity:** 2026-03-23 00:07:44 – 17:12:36 UTC
**Analyst Level:** Tier 3 SOC
**Log Source:** cowrie.txt

---

## SUMMARY

- **60 unique attacker IPs** active over a **17-hour window** on 2026-03-23 (00:07–17:12 UTC)
- **76 successful root logins** from ~12 IPs; remainder failed with credential spraying
- **28 confirmed SSH authorized_keys injection attempts** — a coordinated, automated persistence campaign
- Shared attacker-controlled SSH public key deployed across all persistence sessions: comment label `mdrfckr`
- Password `3245gs5662d34` used as a **secondary credential** by multiple IPs post-initial compromise — suggests shared tooling or campaign coordination
- `chattr -ia .ssh` used to defeat immutable file attributes before key injection — indicates experienced adversaries
- One IP (`185.156.73.233`) attempted **TCP port forwarding to Google IPs (142.251.x.x:443)** — possible C2 or exfil proxy test
- Two distinct attacker clusters: **bulk credential scanners** (automated, high volume) and **targeted persistence operators** (post-auth, scripted)

---

## ATTACKERS

### IP: 2.57.122.208
| Field | Detail |
|---|---|
| Sessions | 37+ (`da9b497ad23e`, `ef42d1e71939`, `f245ccc23c78`, `4dd60517329b`, `38ff63334ff2`, ...) |
| Auth Attempts | High volume, multiple usernames |
| Success | Not observed |
| Commands | Not observed |
| Behavior | **Automated credential scanner** — SSH-2.0-Go client, high session count, no post-auth activity |

---

### IP: 146.70.172.97
| Field | Detail |
|---|---|
| Sessions | 7+ (`15531b670a13`, `0be4da981ad1`, `5f8d39eeb136`, `b6df9b560f96`, `8421db58674f`, `1384d142ef11`, `29391e5ff10d`) |
| Auth Attempts | `root/password`, `root/` (empty), `root/P@ssw0rd1!` |
| Success | Yes — `root/password`, `root/P@ssw0rd1!` |
| Commands | `echo $SHELL`, `history`, `ls`, `ls -la`, `cd`, `cd ~`, `cd root/`, `cd ../` |
| Behavior | **Recon / light enumeration** — manual-feeling navigation cadence; no persistence observed |

---

### IP: 103.13.207.34
| Field | Detail |
|---|---|
| Sessions | Multiple (`a21d9b6c5111`, `16b4849ce440`, `b2982e1daf10`, `86ba854189c7`, `c75d9a07323f`, `d354ef441be5`) |
| Auth Attempts | `root/automa`, `root/3245gs5662d34` |
| Success | Yes — both credentials |
| Commands | `chattr -ia .ssh`, `lockr -ia .ssh`, SSH key echo → `authorized_keys`, `chmod -R go= ~/.ssh` |
| Behavior | **Automated persistence operator** — scripted SSH key injection using shared campaign key (`mdrfckr`) |

---

### IP: 156.253.5.146
| Field | Detail |
|---|---|
| Sessions | 12+ (`f756436014c6`, `b342bb60037d`, `18835f9831cf`, `fcba1953fe8d`, `f9cb57ed8c4d`, `abb0beee143b`) |
| Auth Attempts | `root/12332145`, `root/3245gs5662d34` |
| Success | Yes — both |
| Commands | SSH key injection sequence (identical to 103.13.207.34) |
| Behavior | **Automated persistence operator** — same toolchain, same campaign key |

---

### IP: 92.207.4.157
| Field | Detail |
|---|---|
| Sessions | Multiple |
| Auth Attempts | `root/openmediavault`, `root/3245gs5662d34`, `root/Admin@2022`, `root/radware` |
| Success | Yes — multiple |
| Commands | SSH key injection sequence |
| Behavior | **Automated persistence operator** — broad credential list suggests earlier recon or credential list sharing |

---

### IP: 185.156.73.233
| Field | Detail |
|---|---|
| Sessions | Multiple |
| Auth Attempts | `root/Q1w2e3r4`, `root/P@55w0rd!` |
| Success | Yes |
| Commands | TCP port forwarding → `142.251.142.206:443`, `142.251.39.142:443` (Google infrastructure), `google.com:443` — all refused |
| Behavior | **Proxy/exfil probing** — unique behavior; testing network egress through honeypot |

---

### IP: 168.232.136.13
| Field | Detail |
|---|---|
| Sessions | Multiple |
| Auth Attempts | `root/automa`, `root/3245gs5662d34` |
| Success | Yes |
| Commands | SSH key injection sequence |
| Behavior | **Automated persistence operator** — same campaign as 103.13.207.34 |

---

### Coordinated Persistence Campaign IPs
The following IPs executed the same SSH key injection script with the same public key (`mdrfckr`):

`93.93.202.165` · `95.90.13.168` · `115.190.166.75` · `49.247.37.22` · `36.71.189.255` · `159.223.37.230` · `156.227.237.176` · `129.222.203.222` · `36.255.3.203` · `14.103.183.21` · `201.186.40.250` · `151.71.35.213` · `124.193.81.23` · `130.12.180.51` · `171.25.158.70` · `61.50.114.230` · `23.234.119.24` · `213.74.255.86` · `202.165.16.9`

| Field | Detail |
|---|---|
| Auth | `root/Huawei12`, `root/3245gs5662d34`, `root/@WSX3edc4rfv`, `root/Aaa123123`, `root/pf123456` |
| Behavior | **Automated persistence operator** — coordinated campaign, shared key |

---

## TIMELINE

| Time (UTC) | Event |
|---|---|
| 2026-03-23 00:07:44 | First connection — `2.57.122.208` begins scanning |
| 2026-03-23 00:xx | Credential spray wave begins (Go-based scanner, multiple IPs) |
| 2026-03-23 ~morning | First successful logins — `146.70.172.97` (`root/password`) |
| 2026-03-23 ~morning | `103.13.207.34` logs in as `root/automa` → SSH key injection #1 |
| 2026-03-23 ongoing | 28 IPs execute identical SSH key persistence sequence |
| 2026-03-23 ongoing | `185.156.73.233` attempts TCP port-forward to Google IPs (all refused) |
| 2026-03-23 ~midday | Secondary credential wave: `root/3245gs5662d34` used by 10+ IPs |
| 2026-03-23 17:12:36 | Last connection observed — `119.205.179.217` |

---

## COMMANDS

### Recon
```bash
echo $SHELL
history
ls
ls -la
cd / cd ~ / cd root/ / cd ../
```

### Persistence / Backdoor
```bash
cd ~; chattr -ia .ssh; lockr -ia .ssh
rm -rf .ssh && mkdir .ssh
echo "ssh-rsa AAAA...mdrfckr" >> .ssh/authorized_keys
chmod -R go= ~/.ssh
```

### Lateral Movement / Proxy
```
direct-tcpip → 142.251.142.206:443
direct-tcpip → 142.251.39.142:443
direct-tcpip → google.com:443
```

### Download / Execution
- No `wget` / `curl` / `tftp` observed
- File written: `~/.ssh/authorized_keys` via shell echo redirect (28 instances)
- SHA-256 of redirected content: `a8460f446be540410004b1a8db4083773fa46f7fe76fa84219c93daa1669f8f2`

---

## IOCs

### Attacker IPs (High Confidence)
```
103.13.207.34       156.253.5.146       168.232.136.13
92.207.4.157        93.93.202.165       95.90.13.168
115.190.166.75      49.247.37.22        36.71.189.255
159.223.37.230      185.156.73.233      146.70.172.97
2.57.122.208
```

### Usernames
`root`, `sol`, `ubuntu`, `admin`, `ftpuser`, `nova`, `otrs`, `kanban`, `omm`, `lcx`

### Passwords (Successful)
| Password | Notes |
|---|---|
| `password` | Common default |
| `` (empty) | No password set |
| `P@ssw0rd1!` | Common leet-speak variant |
| `12332145` | Numeric |
| `3245gs5662d34` | **Campaign pivot password — used by 10+ IPs** |
| `automa` | Likely default/service credential |
| `Q1w2e3r4` | Keyboard walk |
| `P@55w0rd!` | Leet-speak variant |
| `openmediavault` | Appliance default |
| `Admin@2022` | Year-based admin password |
| `radware` | Vendor default |
| `Huawei12` | Vendor default |
| `@WSX3edc4rfv` | Keyboard walk |
| `pf123456` | Numeric suffix |
| `Aaa123123` | Repeated pattern |

### Attacker SSH Backdoor Key
```
ssh-rsa AAAAB3NzaC1yc2EAAAABJQAAAQEArDp4cun2lhr4KUhBGE7VvAcwdli2a8dbnrTOrbMz1+5O73fc
BOx8NVbUT0bUanUV9tJ2/9p7+vD0EpZ3Tz/+0kX34uAx1RV/75GVOmNx+9EuWOnvNoaJe0QXxziIg9eLBH
pgLMuakb5+BgTFB+rKJAw9u9FSTDengvS8hX1kNFS4Mjux0hJOK8rvcEmPecjdySYMb66nylAKGwCEE6WEQ
Hmd1mUPgHwGQ0hWCwsQk13yCGPK5w6hYp5zYkFnvlC8hGmd4Ww+u97k6pfTGTUbJk14ujvcD9iUKQTTWY
YjIIu5PmUux5bsZ0R4WFwdIe6+i6rBLAsPKgAySVKPRK+oRw== mdrfckr
```
**Key comment / attacker tag:** `mdrfckr`

### SSH Client Fingerprints
| Fingerprint | Client |
|---|---|
| `16443846184eafde36765c9bab2f4397` | SSH-2.0-Go (scanner) |
| `03a80b21afa810682a776a7d42e5e6fb` | SSH-2.0-libssh_0.11.1 |
| `4c6bc5bc5953c531bde125c3af0ecc84` | SSH-2.0-libssh2_1.11.1 |
| `1f2f2f9b0a736952f743e5490c64c98a` | SSH-2.0-libssh_0.10.5 |

### File Hash
| Hash | Type | Description |
|---|---|---|
| `a8460f446be540410004b1a8db4083773fa46f7fe76fa84219c93daa1669f8f2` | SHA-256 | Redirected authorized_keys content |

---

## ASSESSMENT

> **Risk Level: CRITICAL**

This log captures an active, coordinated multi-stage SSH compromise campaign. The attacker cluster achieved **76 successful root logins** and deployed a persistent SSH backdoor (`mdrfckr` key) on **28 occasions** using a scripted, repeatable sequence that bypasses immutable file attributes via `chattr`.

The shared password `3245gs5662d34` and identical public key across 20+ geographically distributed IPs strongly indicates a **botnet or coordinated campaign tool** — not independent actors.

The port-forwarding attempts by `185.156.73.233` toward Google-routed IPs suggest a follow-on capability test for proxy pivoting or traffic laundering.

**Any real host running SSH with these credentials or this key should be considered fully compromised and requires immediate rebuild and forensic investigation.**
