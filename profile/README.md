# Weft

> *Without the weft, the warp is just parallel threads.*

**Weft** is an open, Go-native cloud platform — **multi-hypervisor, multi-tenant, multi-AZ**. One binary, server or client mode, like the HashiCorp tools. Runs on a laptop. Scales to a 3-DC cluster.

The name comes from weaving: the *weft* is the thread that crosses perpendicularly through the warp to form fabric — exactly what middleware does between heterogeneous hosts, networks, and storage backends.

The GitHub `weft` org was taken, so the project's home is **`openweft`** (the Go import prefix is `github.com/openweft/…`).

---

## Why Weft

- **One binary, two modes.** `weft agent` runs the control plane (single-host all-in-one by default; `--server`/`--client` split for multi-host). `weft <noun> <verb>` is the gRPC client (`weft microvm run`, `weft project create`, `weft share attach`, …).
- **Infrastructure as micro-VMs.** etcd, NATS, dex, zot, CoreDNS, CubeFS, plus the `weft` daemon itself — all run as micro-VMs on the same substrate as tenant workloads. **Every one is a [CNCF](https://www.cncf.io/projects/) project**; there is no separate control-plane box.
- **microVM = Docker-style.** OCI image → virtio-fs rootfs → shared kernel → `weft-microvm-init` execs the entrypoint as PID 1. ~180 ms cold boot.
- **Multi-hypervisor.** Apple Virtualization.framework on macOS, QEMU/KVM elsewhere. Drivers are external go-plugin binaries (`weft-driver-vz`, `weft-driver-qemu`, …) pulled by digest — `weft` itself stays pure-Go, `CGO_ENABLED=0`.
- **Go-native end-to-end.** Control plane, agents, in-guest init, FUSE clients, even the WireGuard mesh and BGP router — no Rust, no C dependencies in the control plane. cgo isolated in driver plugins where Apple-VZ needs it.
- **Multi-AZ, multi-rack placement.** HA intent declared in HCL (`placement { count=3, az="different", rack="different" }`); the scheduler honours `AZ ⊃ Rack ⊃ Host`.
- **Three primitives, deliberately separate.** A **flavor** is compute (vCPU/RAM). A **volume** is single-attach block (`weft-block`, replicated). A **share** is multi-attach POSIX (CubeFS, RWX) — `weft share attach` propagates a mount across a group of micro-VMs over the event bus.

---

## Get started

| | |
|---|---|
| 🌐 **Site & docs** | <https://openweft.github.io> (Hugo landing + MkDocs reference) |
| 🔧 **Main binary** | [`weft`](https://github.com/openweft/weft) — agent + client + control plane |
| 🌍 **Web dashboard** | [`weft-webui`](https://github.com/openweft/weft-webui) — Go + huma API + Svelte SPA |
| 🧩 **Terraform** | [`terraform-provider-weft`](https://github.com/openweft/terraform-provider-weft) |
| 📜 **License** | BSD-3-Clause (matches the sibling cloud-boot project) |

---

## Where the code lives

### Control plane
- [**weft**](https://github.com/openweft/weft) · unified daemon + CLI
- [weft-client](https://github.com/openweft/weft-client) · gRPC dial / token / event-stream helpers
- [weft-proto](https://github.com/openweft/weft-proto) · agent gRPC contract
- [weft-drivers](https://github.com/openweft/weft-drivers) · Hypervisor / Network / Volume / Image driver interfaces
- [weft-driver-plugin](https://github.com/openweft/weft-driver-plugin) · go-plugin protocol between `weft agent` and driver binaries
- [hclconfig](https://github.com/openweft/hclconfig) · shared HCL parser (`cluster.hcl`, …)

### Hypervisor drivers (go-plugin binaries, pulled OCI)
- [weft-driver-vz](https://github.com/openweft/weft-driver-vz) · Apple Virtualization.framework (darwin, cgo, entitled)
- [weft-driver-qemu](https://github.com/openweft/weft-driver-qemu) · QEMU/KVM (Linux) + QEMU/TCG (cross-platform dev)
- [weft-driver-vmd](https://github.com/openweft/weft-driver-vmd) · OpenBSD VMD

### microVM stack (Docker-style: OCI image → microVM)
- [weft-microvm](https://github.com/openweft/weft-microvm) · host-side runtime (OCI pull, virtio-fs/9p rootfs)
- [weft-microvm-init](https://github.com/openweft/weft-microvm-init) · PID 1 (pod mode supervises N containers; mono-container `pivot_root+exec`)
- [weft-microvm-agent](https://github.com/openweft/weft-microvm-agent) · in-VM agent — NATS subscribers apply dynamic config (WireGuard mesh, CubeFS share mounts)
- [weft-microvm-kernel](https://github.com/openweft/weft-microvm-kernel) · custom Linux kconfig (WireGuard + virtio-9p + virtio-fs + FUSE)

### Storage
- [weft-block](https://github.com/openweft/weft-block) · single-attach block volumes — Longhorn-engine fork, Weft-native control plane, **NBD** frontend (no iSCSI/tgt)
- [cloud-init](https://github.com/openweft/cloud-init) · pure-Go NoCloud cloud-init seed builder (classic VMs)

### Network
- [weft-network](https://github.com/openweft/weft-network) · network controller (Routers, LBs, DNS, scheduling rules)
- [weft-network-proto](https://github.com/openweft/weft-network-proto) · network controller gRPC contract
- [weft-proxy](https://github.com/openweft/weft-proxy) · embedded Caddy (L7) + caddy-l4 (L4) reverse proxy
- [weft-router](https://github.com/openweft/weft-router) · BGP egress router micro-VM (GoBGP: BGP-4 + EVPN + flowspec)

### Apps & UI
- [weft-webui](https://github.com/openweft/weft-webui) · Go + huma API + embedded Svelte/daisyUI SPA
- [weft-app-core](https://github.com/openweft/weft-app-core) · client-side supervisor (SSH/WireGuard transports, failover anti-flap)
- [weft-app-osx](https://github.com/openweft/weft-app-osx) · macOS tray (Go + webview + systray)
- [weft-app-windows](https://github.com/openweft/weft-app-windows) · Windows tray
- [weft-app-gtk](https://github.com/openweft/weft-app-gtk) · Linux GTK tray
- [weft-app-android](https://github.com/openweft/weft-app-android) · Android client
- [weft-app-ios](https://github.com/openweft/weft-app-ios) · iOS client

### CI runners (self-hosted, microVM-backed)
- [weft-runner-github](https://github.com/openweft/weft-runner-github) · GitHub Actions runner
- [weft-runner-gitlab](https://github.com/openweft/weft-runner-gitlab) · GitLab CI runner
- [weft-runner-forgejo](https://github.com/openweft/weft-runner-forgejo) · Forgejo Actions runner

### Tooling
- [terraform-provider-weft](https://github.com/openweft/terraform-provider-weft) · 100% terraform-plugin-framework
- [openweft.github.io](https://github.com/openweft/openweft.github.io) · landing site (Hugo) + reference docs (MkDocs Material)

---

<sub>Built on, and bet on, CNCF projects: etcd · NATS · CoreDNS · CubeFS · dex · zot.</sub>
