# Lab: Display IPv4 Addresses and the Routing Table with `ip`

**Series:** linux-ops-mastery — RHCSA Networking
**Subjects covered:** `iproute2` replacement for legacy `ifconfig`, `ip addr show`, interface states and flags, secondary addresses, `ip route show`, default routes, `ip -brief`, scope, `proto kernel` vs `proto dhcp` vs `proto static`, reading `metric`
**Career arcs covered:** RHCSA (verify NM changes quickly), RHCE (facts modules echo the same fields), SRE ("why is traffic leaving the wrong NIC?"), DevOps (container host multi-homing), AI/MLOps (separate storage vs training plane IPs)
**Prerequisite:** Lab 31 (static IPv4) or any working DHCP VM — you need at least one configured interface
**Time Estimate:** 30 to 40 minutes
**Difficulty arc:** Task 1 foundation · 2–3 `ip addr` deep read · 4–5 `ip route` semantics · 6 exam-realistic capstone

---

## Objective

Retire the habit of guessing network state from scattered tools. By the end of this lab you can **enumerate interfaces**, read **IPv4/CIDR assignments** including secondary aliases, and interpret the **main routing table** well enough to answer "which path will this packet take?" without opening a GUI.

The capstone is the RHCSA-realistic prompt: *"Show all IPv4 addresses attached to `eth0`, summarize all interfaces in one line each, and print the full IPv4 routing table highlighting the default route."*

> **Lab safety note:** This lab is **read-only** — no interfaces are renumbered here. If you experimentally add addresses in optional notes, remove them afterward to avoid confusing later labs.

---

## Concept: Addresses Live on Interfaces; Routes Live in the FIB

Linux separates **interface configuration** (addresses, link state, MTU) from **forwarding decisions** (the Forwarding Information Base — what `ip route` prints for the main table). NetworkManager (or manual scripts) pushes addresses and routes; `ip` is the **truth viewer**.

```
   ┌────────────────────────────────────────────────────────────┐
   │  NIC `eth0`                                                │
   │    ├── LOWER_UP link state                                 │
   │    ├── inet 192.168.50.10/24  (primary)                   │
   │    └── inet 192.168.50.11/24  (secondary alias)            │
   └────────────────────────────────────────────────────────────┘
                           │
                           ▼
   ┌────────────────────────────────────────────────────────────┐
   │  Main routing table (IPv4)                                 │
   │    default via 192.168.50.1 dev eth0 metric 100            │
   │    192.168.50.0/24 dev eth0 proto kernel scope link src … │
   └────────────────────────────────────────────────────────────┘
```

> **Why this matters:** EX200 wants **verification**, not vibes. Graders expect `ip -4 addr` and `ip -4 route` because those outputs are unambiguous and scriptable.

---

## 📜 Why `ip` Replaced `ifconfig` — The Story

The `ifconfig` tool predates many modern kernel networking features. As Linux gained **multiple routing tables**, **policy routing**, **IPv6**, **VLANs**, **bridges**, and **network namespaces**, maintaining parallel ioctl-based tools became brittle.

Alexey Kuznetsov and the `iproute2` suite (`ip`, `tc`, `ss`, `bridge`) unified configuration around **netlink** — a flexible binary protocol between user space and the kernel. RHEL 7 deprecated `ifconfig` documentation; RHEL 8 and 9 ship `ip` as the **supported** inspection path.

NetworkManager on RHEL still **implements** changes, but operators **verify** with `ip` because it prints exactly what the kernel will use when a packet is forwarded.

> **The point of the story:** `ip` is not "new syntax for old ideas" — it is the CLI aligned with how the kernel actually stores routes and addresses.

---

## 👪 The `ip` Object Family — Who Lives There

### Top-level objects you will touch today

| Object | Command | What you learn |
|---|---|---|
| **link** | `ip link show` | MAC, state, MTU, master (bridge bond) |
| **address** | `ip addr show` | IPv4/IPv6 attached to interfaces |
| **route** | `ip route show` | Main table paths and gateways |
| **rule** | `ip rule show` | Policy routing (RHCE depth) |

### Common `ip` flags

| Flag | Effect |
|---|---|
| `-4` | Restrict to IPv4 objects |
| `-6` | Restrict to IPv6 objects |
| `-brief` | One-line summary per interface |
| `dev NAME` | Scope command to a single NIC |

> **The point of the family tree:** If you know **link / address / route**, you can answer 90% of day-two networking questions on RHEL.

---

## 🔬 The Anatomy of `ip -4 addr show eth0` — In One Diagram

```
$ ip -4 addr show eth0
  │   │   │    │
  │   │   │    └─ Interface name filter
  │   │   └─ Address object (layer-3 view includes IPv4 lines)
  │   └─ IPv4-only filter
  └─ iproute2 multiplexer command

2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    altname enp0s3
    inet 192.168.50.10/24 brd 192.168.50.255 scope global noprefixroute eth0
       │      │               │                         │                 │
       │      │               │                         │                 └─ Device binding
       │      │               │                         └─ Scope: where this address is considered "local"
       │      │               └─ Broadcast address derived from prefix
       │      └─ IPv4 + CIDR prefix length (bits)
       └─ Kernel keyword marking IPv4 line
```

> **Reading rule:** `UP,LOWER_UP` means admin-up **and** carrier detected — the link is truly live.

---

## 📚 `ip addr` / `ip route` Reference Table

| Task | Command | Notes |
|---|---|---|
| All IPv4 addresses | `ip -4 addr` | Verbose per interface |
| One interface | `ip -4 addr show dev eth0` | Replace `eth0` with yours |
| Brief table | `ip -br addr` | Ops favorite |
| Main IPv4 routes | `ip -4 route` | Same as `ip -4 route show` |
| Default only | `ip -4 route show default` | Quick gateway check |
| Match destination | `ip route get 8.8.8.8 from 192.168.50.10` | Which route wins (advanced) |

> **Rule one of route reading:** The **most specific prefix** wins; the `default` route is least specific and is chosen only when nothing else matches.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Rapid verification after `nmcli` edits — saves minutes under exam clock pressure. |
| **RHCE candidate** | Jinja templates often render `ansible_default_ipv4` facts sourced from the same kernel view. |
| **SRE / Platform** | Multi-homed hosts: `ip route` explains asymmetric flows when metrics collide. |
| **DevOps** | Kubernetes nodes: quickly see if Calico CIDRs landed on `tunl0` / `cali*` interfaces. |
| **AI / MLOps** | Verify RDMA `mlx` NIC `UP` and correct `/24` on storage VLAN before launching multi-node jobs. |

---

## 🔧 The 6 Tasks

> Six read-only phases that build **interface literacy** and **route literacy**.

---

### Task 1 — Baseline: links, brief addresses, and device inventory

**Purpose:** Orient yourself to interface names on this VM and capture a one-line-per-interface summary.

```bash
ip link show
ip -br link
ip -br addr
```

**Human-Readable Breakdown:** Show full link-layer details, then brief link states, then brief L3 addresses — a progression from physical names to assigned IPs.

**Reading it left to right:** `ip link` focuses on MAC/MTU/state. `ip -br addr` merges IPv4/IPv6 per interface in a dense ops-friendly view.

**The story:** RHEL 9 may show **predictable interface names** like `enp0s3` — always read before typing `eth0` in exam answers.

**Expected output:**

```text
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:ab:cd:ef brd ff:ff:ff:ff:ff:ff
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
eth0             UP             52:54:00:ab:cd:ef <BROADCAST,MULTICAST,UP,LOWER_UP>
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             192.168.122.45/24 fe80::5054:ff:feab:cdef/64
```

**Switches**

| Token | Meaning |
|---|---|
| `ip link show` | Layer-2 oriented interface listing |
| `-br` | Brief tabular output |
| `ip -br addr` | Addresses paired with brief link rows |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Interface `DOWN` | `nmcli dev connect IFACE` or fix link |
| Unexpected `DOWN` on `eth0` | Check VM cable / virt-manager NIC attachment |
| Many `cali*` interfaces | Kubernetes present — focus on primary `eth0` |

---

### Task 2 — Deep read: `ip -4 addr show` for the primary NIC

**Purpose:** Parse flags, broadcast, scope, and `noprefixroute` semantics for your main Ethernet NIC.

```bash
DEV=eth0
ip -4 addr show dev "$DEV"
```

**Human-Readable Breakdown:** Filter IPv4 output to a single device and read the `inet` line components slowly.

**Reading it left to right:** `brd` is broadcast IPv4. `scope global` means not restricted to same-host. `noprefixroute` indicates NM may manage connected routes explicitly.

**The story:** This output is what you paste (mentally) into incident tickets — it beats paraphrasing.

**Expected output:**

```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    altname enp0s3
    inet 192.168.122.45/24 brd 192.168.122.255 scope global noprefixroute eth0
```

**Switches**

| Token | Meaning |
|---|---|
| `dev eth0` | Limit to one interface |
| `inet …/24` | Address + prefix |
| `brd` | Subnet broadcast |
| `scope global` | Usable beyond host-only |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| No `inet` line | No IPv4 on device — check NM profile |
| Multiple `inet` lines | Primary + secondary aliases — read order |
| `scope link` only | Address not usable beyond subnet |

---

### Task 3 — Routing table 101: show all IPv4 routes and the default

**Purpose:** Read the **main** IPv4 FIB entries and isolate the default route.

```bash
ip -4 route show
ip -4 route show default
```

**Human-Readable Breakdown:** Dump all IPv4 routes, then filter to `default` — usually one line showing gateway and dev.

**Reading it left to right:** Connected routes appear as `proto kernel scope link src YOUR_IP`. Default appears as `default via GATEWAY dev IFACE`.

**The story:** If `ping 8.8.8.8` fails, these two commands tell you whether you **lack a default** or have one pointing nowhere.

**Expected output:**

```text
default via 192.168.122.1 dev eth0 proto dhcp src 192.168.122.45 metric 100
192.168.122.0/24 dev eth0 proto kernel scope link src 192.168.122.45 metric 100
default via 192.168.122.1 dev eth0 proto dhcp src 192.168.122.45 metric 100
```

**Switches**

| Token | Meaning |
|---|---|
| `default via` | Default route (0.0.0.0/0) |
| `proto dhcp` | Learned from DHCP transaction |
| `proto static` | From static NM profile |
| `metric` | Lower is preferred among same-prefix routes |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| No default line | Add gateway in NM profile or fix DHCP |
| Two defaults | Compare metrics — lower wins |
| Unexpected `via` | Wrong profile active — `nmcli dev status` |

---

### Task 4 — Correlation exercise: compare NM intent to kernel reality

**Purpose:** Practice the **NM says / kernel shows** comparison loop central to troubleshooting.

```bash
CON=$(nmcli -t -f DEVICE,NAME dev status | awk -F: '$1=="eth0"{print $2; exit}')
nmcli -f IP4.ADDRESS,IP4.GATEWAY,IP4.DNS,IP4.ROUTE ip4 con show "$CON"
echo "---- kernel ----"
ip -4 addr show dev eth0
ip -4 route show dev eth0
```

**Human-Readable Breakdown:** Resolve the active NM connection for `eth0`, print NM's summarized IPv4 fields, then print kernel addresses and routes scoped to the same device.

**Reading it left to right:** NM fields are **higher-level**; `ip` is **authoritative** after successful activation.

**The story:** When they disagree, either activation failed or a lower tool injected transient routes — rare on stock RHEL if you stay in NM.

**Expected output:**

```text
IP4.ADDRESS[1]:         192.168.122.45/24
IP4.GATEWAY:            192.168.122.1
IP4.DNS[1]:             192.168.122.1
IP4.ROUTE[1]:           dst = 0.0.0.0/0, nh = 192.168.122.1, mt = 100
---- kernel ----
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 192.168.122.45/24 brd 192.168.122.255 scope global dynamic noprefixroute eth0
default via 192.168.122.1 dev eth0 proto dhcp src 192.168.122.45 metric 100
192.168.122.0/24 dev eth0 proto kernel scope link src 192.168.122.45 metric 100
```

**Switches**

| Token | Meaning |
|---|---|
| `nmcli … ip4 con show` | NM-side IPv4 summary |
| `ip … show dev eth0` | Kernel-side view for same NIC |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `awk` prints nothing | Device not named `eth0` — adjust `DEV=` |
| NM shows static, kernel shows DHCP | Profile not activated — `nmcli con up` |

---

### Task 5 — Edge case literacy: secondary addresses and `ip -br -4`

**Purpose:** Recognize **secondary** IPv4 and practice family-limited brief output.

```bash
sudo nmcli con mod "$(nmcli -t -f DEVICE,NAME dev status | awk -F: '$1=="eth0"{print $2; exit}")" +ipv4.addresses 192.168.122.210/24
sudo nmcli con up "$(nmcli -t -f DEVICE,NAME dev status | awk -F: '$1=="eth0"{print $2; exit}")"
ip -4 addr show dev eth0
ip -br -4 addr
```

**Human-Readable Breakdown:** Append a harmless secondary address on the lab LAN, re-activate, show verbose and brief IPv4-only views.

**Reading it left to right:** The `secondary` keyword appears in verbose `ip` output for aliases. `-br -4` suppresses IPv6 noise for quick scans.

**The story:** Service IPs, VIPs, and temporary diagnostics all show up as extra `inet` lines — learn to count them.

**Expected output:**

```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 192.168.122.45/24 brd 192.168.122.255 scope global dynamic noprefixroute eth0
    inet 192.168.122.210/24 brd 192.168.122.255 scope global secondary noprefixroute eth0
lo               UNKNOWN        127.0.0.1/8
eth0             UP             192.168.122.45/24 192.168.122.210/24
```

**Switches**

| Token | Meaning |
|---|---|
| `+ipv4.addresses` | Append alias without removing primary |
| `ip -br -4 addr` | Brief IPv4-only address listing |
| `secondary` | Kernel annotation for non-primary address |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Duplicate address detected | Pick unused `.210` in your subnet |
| Secondary missing after reboot | Wrong profile — verify autoconnect |

---

### Task 6 — Capstone: documentation-style interface and route report

**Task statement:** *"For interface `eth0`, print detailed IPv4 addresses, a brief IPv4 summary for all interfaces, the full IPv4 routing table, and the default route line alone. Remove any lab secondary address added in Task 5 afterward."*

**Purpose:** Simulate an exam-style **evidence bundle** of `ip` output with safe teardown.

```bash
sudo -i
DEV=eth0
CON=$(nmcli -t -f DEVICE,NAME dev status | awk -F: -v d="$DEV" '$1==d{print $2; exit}')
echo "=== ip -4 addr show $DEV ==="
ip -4 addr show dev "$DEV"
echo "=== ip -br -4 addr ==="
ip -br -4 addr
echo "=== ip -4 route ==="
ip -4 route
echo "=== default only ==="
ip -4 route show default
```

**Human-Readable Breakdown:** Parameterize the device, print four labeled sections examiners love, using `-br -4` for the dense table.

**Layer stack you documented:**

```text
`ip -4 addr` → kernel IFA addresses per interface
`ip -4 route` → kernel FIB entries for IPv4 forwarding decisions
```

**The story:** This is the **two-minute verification** after any NM change — read both layers before you declare victory.

**Expected output:**

```text
=== ip -4 addr show eth0 ===
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 192.168.122.45/24 brd 192.168.122.255 scope global dynamic noprefixroute eth0
    inet 192.168.122.210/24 brd 192.168.122.255 scope global secondary noprefixroute eth0
=== ip -br -4 addr ===
lo               UNKNOWN        127.0.0.1/8
eth0             UP             192.168.122.45/24 192.168.122.210/24
=== ip -4 route ===
default via 192.168.122.1 dev eth0 proto dhcp src 192.168.122.45 metric 100
192.168.122.0/24 dev eth0 proto kernel scope link src 192.168.122.45 metric 100
=== default only ===
default via 192.168.122.1 dev eth0 proto dhcp src 192.168.122.45 metric 100
```

**Cleanup**

```bash
nmcli con mod "$CON" -ipv4.addresses 192.168.122.210/24
nmcli con up "$CON"
ip -4 addr show dev "$DEV"
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `-ipv4.addresses` syntax error | Use exact `/24` string as added |
| Primary address flapped | You removed wrong entry — re-open `nmcli con show` |

---

## 🔍 `ip` Reading Decision Guide

```
What do I need to know?
  │
  ├── "Is the NIC physically up?"
  │       └── ✅ `ip link show DEV` — look for LOWER_UP
  │
  ├── "What IPv4 is configured?"
  │       └── ✅ `ip -4 addr show dev DEV`
  │
  ├── "Give me the one-line ops view"
  │       └── ✅ `ip -br -4 addr`
  │
  ├── "Where will packets to 0.0.0.0/0 go?"
  │       └── ✅ `ip -4 route show default`
  │
  └── "Why two defaults?"
          └── ✅ Compare `metric` and `proto` — lower metric wins
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Inventory links and brief addresses
- [ ] 02 Parse `ip -4 addr show` for primary NIC
- [ ] 03 Read full IPv4 routes + default route
- [ ] 04 Correlate `nmcli` IPv4 summary with `ip` output
- [ ] 05 Add secondary address; observe `ip -br -4 addr`
- [ ] 06 Capstone report + remove secondary alias

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Assuming interface is `eth0` | Commands target wrong NIC | Use `ip link` first |
| Ignoring `metric` | Traffic exits unexpected NIC | Lower metric preferred |
| Confusing `brd` with gateway | Misconfigured ACL rules | `via` is gateway; `brd` is broadcast |
| Reading only NM, never `ip` | Stale truth | Always `ip` after changes |
| Forgetting `-4` | IPv6 noise in brief output | Add `-4` |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Train yourself to type `ip -br -4 addr` and `ip -4 route` from muscle memory — faster than clicking any GUI.

**RHCE candidate**
- Explain how you would template interface names with Ansible facts instead of hardcoding `eth0`.

**SRE / Platform interview**
- Discuss **ecmp** routes — when two equal-cost defaults appear, hashing decides — beyond RHCSA but adjacent.

**DevOps**
- Container hosts: correlate `docker0` / `br-` bridges in `ip link` with CNI plugins.

**AI / MLOps**
- Document both **RoCEv2** IPoIB addresses and **management** `eth0` in postmortems — different `ip` sections.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 31 — Configure a Static IP Address | Changes you verify with `ip` |
| Lab 32 — Check Network Connectivity | Uses routes learned from `ip route` |
| Lab 34 — Inspecting Listening Sockets | Complements L3 view with L4 listeners |
| Lab 35 — Text-Based Network Config `nmtui` | GUI path to addresses seen here |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
