# Sluice SQM

> Zero-config smart queue management for OpenWrt — a standalone replacement for sqm-scripts.

[繁體中文](README.zh-TW.md)

Sluice keeps your connection responsive under load. Gaming, video calls and
browsing stay smooth even while someone else is downloading, uploading or
running BitTorrent.

Most SQM setups ask you to measure your bandwidth, pick an overhead value,
guess a link-layer type and choose between qdiscs and scripts. Sluice works
these out by itself:

- **Detects your link**: reads the WAN protocol (PPPoE, DHCP, VLAN…) and sets the overhead for you, erring on the high side
- **Calibrates bandwidth**: a ~3-minute test on first run. You can also skip it and let Sluice learn under real load
- **Adapts in real time**: watches latency to several reflectors and only throttles the direction that is actually congested
- **Skips shaping when it isn't needed**: if your line barely bloats under load, it stays at full speed. If only one direction bloats, Sluice shapes only that one
- **Knows your hardware**: probes what the running kernel can actually do (CAKE, cake_mq, IFB, ctinfo, HTB) and falls back gracefully
- **Protects the CPU**: when the router can't keep up, it first tunes (packet steering, cake_mq, a lower ceiling), then sheds features step by step, from the cheapest to lose to the most expensive
- **Prioritizes by what traffic is**: four tiers (calls → games → streaming → downloads) based on domains, device type and connection behaviour. P2P is detected even when it is encrypted
- **Never breaks your internet**: all per-packet work stays in the kernel. If the daemon stops, a watchdog removes shaping instead of leaving a broken queue

> Sluice SQM is an independent implementation, not a fork of sqm-scripts.
> Don't run it alongside sqm-scripts, qosify or nft-qos. Sluice will warn you if it finds them.

## Four tabs, three levels of control

| Tab | For | What you see |
|---|---|---|
| Overview | Everyone | Latency grade, current rates, what was detected and why |
| Simple | Everyone | One switch, a trade-off slider (latency ↔ bandwidth) |
| Advanced | Enthusiasts | Per-device priority, bandwidth bounds, adaptation sensitivity |
| Expert | Network nerds | Raw CAKE parameters, an HTB + fq_codel fallback, experimental L4S |

## How it works

```
        ┌────────────── sluice daemon (Rust): decisions ───────────────┐
        │ link detection  calibration  autorate  CPU guard  device/P2P │
        └──────┬───────────────────┬─────────────────────┬─────────────┘
       rate (netlink)     nft sets (devices, domains, P2P)    conntrack events, counters
               ▼                   ▼                     ▲
        ┌──────────────── Linux kernel: every packet ──────────────────┐
        │ CAKE (egress + IFB ingress)  nftables  conntrack  act_ctinfo │
        └──────────────────────────────────────────────────────────────┘
```

Rust only decides and adjusts. The kernel does all per-packet work. See
[docs/design.md](docs/design.md) for the full design.

## Status

🚧 **Design finalized, implementation starting.** No installable package yet.

| | |
|---|---|
| Target | OpenWrt 25.12+ (apk packages) |
| Architectures | x86_64, aarch64, armv7 (musl) |
| Not supported | MIPS (e.g. MT7621). CAKE tops out around 150–400 Mbps there |
| Reference hardware | Intel N100 on 500/500 Mbps FTTH (PPPoE) |

## Documentation

| File | Contents |
|---|---|
| [docs/design.md](docs/design.md) | Finalized design (in Traditional Chinese) |
| [docs/porting-lessons.md](docs/porting-lessons.md) | Lessons from porting kuroshio-dns to Padavan that shape this project |
| [docs/mockups/luci-ui.html](docs/mockups/luci-ui.html) | LuCI UI mockup (open in a browser) |
| [CLAUDE.md](CLAUDE.md) | Development handoff and conventions |

## Contributing

Contributions use the [DCO](https://developercertificate.org/). Sign off your
commits with `git commit -s`. No CLA.

## License

[GPL-2.0-only](LICENSE), for both the `sluice` daemon and `luci-app-sluice`.
