<p align="center"><img src="https://raw.githubusercontent.com/openweft/brand/main/social/openweft.png" alt="openweft" width="720"></p>

# Weft

> *Without the weft, the warp is just parallel threads.*

**Weft** is an open, Go-native cloud platform — **microVM-first, multi-hypervisor, multi-tenant, multi-AZ**. One binary, server or client mode, like the HashiCorp tools. Runs on a laptop. Scales to a 3-DC cluster.

The platform stack is **pure-Go end-to-end** : control plane, agents, in-guest init, FUSE clients, even the WireGuard mesh and the BGP router. cgo only lives in the Apple-VZ driver plugin, isolated from the core. Greenfield code is **BSD 3-Clause** ; forks-and-adapt (e.g. `weft-block` from `longhorn-engine`) keep their upstream license.

The GitHub `weft` org was taken, so the project's home is **`openweft`** (the Go import prefix is `github.com/openweft/…`).

---

## Why Weft

- **One binary, two modes.** `weft agent` runs the control plane (single-host all-in-one by default; `--server`/`--client` split for multi-host). `weft <noun> <verb>` is the gRPC client (`weft microvm run`, `weft project create`, `weft share attach`, …).
- **microVM-first.** `weft microvm` is the default execution path — OCI image → virtio-fs / virtio-9p rootfs → shared kernel → `weft-microvm-init` execs the entrypoint as PID 1. All new platform features land here first. `weft instance` (classic VM) remains as an escape hatch for Windows / BSD guests, network appliances (VyOS, OPNsense, …), and workloads that need their own kernel.
- **Four hypervisor backends.** Apple Virtualization framework on macOS, QEMU/KVM on Linux, OpenBSD `vmd`, and Huawei FusionCompute (`dcs`). Drivers are external go-plugin binaries (`weft-driver-vz`, `weft-driver-qemu`, `weft-driver-vmd`, `weft-driver-dcs`) pulled by digest — `weft` itself stays pure-Go, `CGO_ENABLED=0` everywhere including darwin.
- **Embedded etcd state plane.** Cluster catalogues and dynamic config live in etcd ; `weft agent` embeds `go.etcd.io/etcd/server/v3/embed` for dev / single-host, talks to a 3-DC external cluster in HA. No JSON-on-disk parallel fallback path.
- **Embedded Caddy reverse proxy.** L7 (HTTP, auto-HTTPS via ACME) and L4 (TCP/UDP through the [caddy-l4](https://github.com/mholt/caddy-l4) plugin) live as a supervised Caddy subprocess inside `weft-agent` on every host. No separate proxy microVM, no Envoy.
- **WireGuard mesh overlay.** Multi-tenant isolation is cryptographic by construction — one mesh per `Network`, distinct keys per tenant, encrypted on the wire. No VXLAN, no L2 broadcast plane ; the `Network` + per-port registry covers IP allocation, gateway, and DNS.
- **Multi-AZ, multi-rack placement.** HA intent declared in HCL (`placement { count = 3, az = "different", rack = "different" }`) ; the scheduler honours `AZ ⊃ Rack ⊃ Host`.
- **Three primitives, deliberately separate.** A **flavor** is compute (vCPU / RAM / GPU). A **volume** is single-attach block (`weft-block`, Longhorn-engine fork — replicated, snapshots, backups). A **share** is multi-attach POSIX (CubeFS, RWX) — `weft share attach` propagates a mount across a group of micro-VMs over the event bus.

---

## Get started

| | |
|---|---|
| Site & docs | <https://openweft.github.io> (Hugo landing + MkDocs reference under `/docs/latest/`) |
| Main binary | [`weft`](https://github.com/openweft/weft) — agent + client + control plane |
| Web dashboard | [`weft-webui`](https://github.com/openweft/weft-webui) — Go + huma API + Svelte SPA |
| Terraform | [`terraform-provider-weft`](https://github.com/openweft/terraform-provider-weft) — 100% plugin-framework |
| License | BSD-3-Clause for greenfield code ; forks (e.g. `weft-block`) keep the upstream license |

The canonical bring-up is `weft up --apply` against a `cluster.hcl` declaring 1 or 3 hosts ; the planner reconciles convergent state over SSH (install + lifecycle). `weft infra bootstrap` then deploys the infrastructure microVMs (etcd, NATS, dex, zot, CoreDNS, Longhorn, CubeFS, `weft-network`, `weft-webui`, observability stack) in dependency order. See [Bring up a cluster](https://openweft.github.io/docs/latest/operations/bring-up/).

---

## What works today

- **`weft-network`** — full daemon, 16/16 RPCs across SchedulingRules + DNS Zones / Records + Routers + LoadBalancers, with both in-memory and etcd backends, `/metrics` on `:9100`, hardened systemd unit + Dockerfile.
- **`weft-block`** — single-attach replicated block volumes ; fork-and-adapt of `longhorn-engine` with a Weft-native control plane and an NBD frontend (the original iSCSI / tgt path is dropped). Pure-Go, builds linux/arm64 with `CGO=0`.
- **`weft-webui`** — token-bucket rate limiter, OIDC token refresh, JSONL audit log, Dockerfile + systemd unit + multi-arch release workflow.
- **Multi-arch infra images** — `weft-{etcd,dex,nats,zot,coredns}` rebuilt from source for **amd64 + arm64 + riscv64 + loong64**.
- **`weft-loom-server`** v0.2 — collaborative editor (CodeMirror 6 + Yjs + relay y-websocket, Overleaf-generic), with per-language compile images (`weft-loom-texlive` v0.1.0 published, `-golang` / `-cpp` / `-python` / `-rust` / `-node` siblings).
- **HA stateless replicas** — `weft-ha-forgejo` and `weft-ha-irods` v0.2.0-rc with real Controller + `EtcdStore` + workflow-published images ; `weft-ha-postgresql` (etcd DCS + VMFencer + reconcile).
- **Live 3-DC bring-up** — `weft up --apply` validated end-to-end against the dc1-r1-h1 / dc2 / dc3 lab cluster.

Experimental or in-progress : multi-driver agents on a single host (HCL + scheduler ready ; agent still launches a single driver subprocess) ; AI log triage (`weft-doctor` v0.1, Ollama-only, passive) ; routine remote agents.

---

## Respawn V0.1.2 (2026-06-08)

`SchedulingRule.RespawnPolicy` (proto v0.10.0, weft v0.3.0) declares a respawn block on the rule (HCL + CLI : `--respawn-enabled --respawn-grace-period --respawn-max-restarts --respawn-window --respawn-backoff exponential --respawn-initial-delay` plus a liveness probe via `--liveness-http-path/-port` or `--liveness-tcp-port`). The agent embeds a `health` package (HTTP + TCP probes), a pure-state `respawn` Reconciler (one goroutine per VM), and an `agentrespawn` bus subscriber + 2 s status poller for drivers that don't emit `vm.state_changed`. Selector grammar V0.1 is `vm.name=<exact>` ; validated live on dc1 with three consecutive cycles and observed exponential backoff. V0.1.2 (this commit run on `weft`) adds systemd `Type=notify` + `WatchdogSec` integration and `etcdcoord` — host-liveness lease, prefix watcher, and per-key leader election riding the existing etcd cluster — laying the foundation for cross-host failover in the next iteration.

---

## Where the code lives

### Control plane
- [**weft**](https://github.com/openweft/weft) · unified daemon + CLI (control plane, scheduler, embedded etcd, embedded Caddy, respawn, floating-IP NAT, security-group nftables)
- [weft-client](https://github.com/openweft/weft-client) · gRPC dial / token / event-stream helpers
- [weft-proto](https://github.com/openweft/weft-proto) · agent gRPC contract (v0.10.0 — HealthProbe + RespawnPolicy on SchedulingRule)
- [weft-network-proto](https://github.com/openweft/weft-network-proto) · networking control-plane gRPC contract
- [weft-drivers](https://github.com/openweft/weft-drivers) · Hypervisor / Network / Volume / Image driver interfaces
- [weft-driver-plugin](https://github.com/openweft/weft-driver-plugin) · go-plugin protocol between `weft-agent` and driver binaries
- [weft-hcl](https://github.com/openweft/weft-hcl) · shared HCL parser (`cluster.hcl`, infra plans, …) ; Go package `wefthcl`
- [weft-cidata](https://github.com/openweft/weft-cidata) · pure-Go NoCloud cloud-init seed builder for classic VMs — renamed from `cloud-init`
- [weft-firstboot](https://github.com/openweft/weft-firstboot) · cloud-init-lite v0.2 first-boot agent for classic VMs (linux + openbsd + freebsd + netbsd, amd64 + arm64)

### Hypervisor drivers (go-plugin binaries, pulled OCI)
- [weft-driver-vz](https://github.com/openweft/weft-driver-vz) · Apple Virtualization framework (darwin, cgo, entitled)
- [weft-driver-qemu](https://github.com/openweft/weft-driver-qemu) · QEMU/KVM (Linux) + QEMU/TCG (cross-platform dev)
- [weft-driver-vmd](https://github.com/openweft/weft-driver-vmd) · OpenBSD `vmd(8)` / `vmctl(8)`
- [weft-driver-dcs](https://github.com/openweft/weft-driver-dcs) · Huawei FusionCompute (VRM, REST upstream, gRPC downstream)
- [weft-drivers](https://github.com/openweft/weft-drivers) · driver interface module shared by the four backends

### microVM stack
- [weft-microvm](https://github.com/openweft/weft-microvm) · host-side runtime (OCI pull, virtio-fs / 9p rootfs, image cache)
- [weft-microvm-init](https://github.com/openweft/weft-microvm-init) · PID 1 (pod mode supervises N containers via embedded `crun` ; mono-container `pivot_root + exec`)
- [weft-microvm-agent](https://github.com/openweft/weft-microvm-agent) · in-VM agent — NATS subscribers apply dynamic config (WireGuard mesh, CubeFS / SFTP mounts, nftables firewall)
- [weft-microvm-kernel](https://github.com/openweft/weft-microvm-kernel) · custom Linux kconfig (WireGuard + virtio-9p + virtio-fs + FUSE + 9P), published as an OCI artifact
- [weft-microvm-backup](https://github.com/openweft/weft-microvm-backup) · in-VM backup agent on top of [kloset](https://github.com/PlakarKorp/kloset) (plakar storage engine)

### Storage
- [weft-block](https://github.com/openweft/weft-block) · single-attach block volumes — `longhorn-engine` fork (Apache 2.0 upstream), Weft-native control plane, NBD frontend, pure-Go `qcow2`. Linux/arm64 `CGO=0`.

### Network
- [weft-network](https://github.com/openweft/weft-network) · network controller (Routers, LBs, DNS zones, scheduling rules) — 16/16 RPCs, etcd-backed, multi-arch release
- [weft-proxy](https://github.com/openweft/weft-proxy) · legacy standalone Caddy binary ; superseded on most setups by the in-agent Caddy
- [weft-router](https://github.com/openweft/weft-router) · per-tenant BGP egress micro-VM (GoBGP: BGP-4 + EVPN + flowspec, programs the kernel FIB via netlink)

### Apps & UI
- [weft-webui](https://github.com/openweft/weft-webui) · Go + huma API + embedded Svelte / daisyUI SPA
- [weft-app-core](https://github.com/openweft/weft-app-core) · client-side supervisor (WireGuard / SSH transports, anti-flap multi-DC failover)
- [weft-app-osx](https://github.com/openweft/weft-app-osx) · macOS tray app (Go + webview + systray)
- [weft-app-windows](https://github.com/openweft/weft-app-windows) · Windows tray app
- [weft-app-gtk](https://github.com/openweft/weft-app-gtk) · Linux GTK tray app
- [weft-app-android](https://github.com/openweft/weft-app-android) · Android client (Kotlin WebView)
- [weft-app-ios](https://github.com/openweft/weft-app-ios) · iOS client (Swift WKWebView)

### Collaborative editing (weft-loom)
- [weft-loom-server](https://github.com/openweft/weft-loom-server) · multi-user editor + sandboxed compile (relay-only y-websocket, CodeMirror 6 + Yjs)
- [weft-loom-texlive](https://github.com/openweft/weft-loom-texlive) · TeX Live full + latexmk + biber compile image
- [weft-loom-golang](https://github.com/openweft/weft-loom-golang), [weft-loom-cpp](https://github.com/openweft/weft-loom-cpp), [weft-loom-python](https://github.com/openweft/weft-loom-python), [weft-loom-rust](https://github.com/openweft/weft-loom-rust), [weft-loom-node](https://github.com/openweft/weft-loom-node) · per-language compile images
- [weft-loom-app-osx](https://github.com/openweft/weft-loom-app-osx), [weft-loom-app-linux](https://github.com/openweft/weft-loom-app-linux), [weft-loom-app-windows](https://github.com/openweft/weft-loom-app-windows) · loom desktop shells (fork of `weft-app-*` with loom branding)

### HA platform plugins
- [weft-ha-postgresql](https://github.com/openweft/weft-ha-postgresql) · etcd DCS + VMFencer (StopVM gRPC) + reconcile state machine
- [weft-ha-forgejo](https://github.com/openweft/weft-ha-forgejo) · stateless-replica HA agent (v0.2.0-rc)
- [weft-ha-irods](https://github.com/openweft/weft-ha-irods) · stateless-replica HA agent (v0.2.0-rc)

### Diagnostics
- [weft-doctor](https://github.com/openweft/weft-doctor) · AI log triage v0.1.1 — Ollama-only, NATS sub → buffer dedup + burst → classify → NATS-only diagnosis (passive)
- [weft-slognats](https://github.com/openweft/weft-slognats) · `slog.Handler` fan-out to stderr + NATS — pre-requisite of `weft-doctor`

### Infra images (4-arch : amd64 + arm64 + riscv64 + loong64)
- [weft-etcd](https://github.com/openweft/weft-etcd) · [weft-dex](https://github.com/openweft/weft-dex) · [weft-nats](https://github.com/openweft/weft-nats) · [weft-zot](https://github.com/openweft/weft-zot) · [weft-coredns](https://github.com/openweft/weft-coredns) — built from source in-house, multi-stage distroless, tag-gated workflow

### CI runners (self-hosted, microVM-backed)
- [weft-runner-github](https://github.com/openweft/weft-runner-github) · [weft-runner-gitlab](https://github.com/openweft/weft-runner-gitlab) · [weft-runner-forgejo](https://github.com/openweft/weft-runner-forgejo)

### Tooling
- [terraform-provider-weft](https://github.com/openweft/terraform-provider-weft) · 100% terraform-plugin-framework
- [openweft.github.io](https://github.com/openweft/openweft.github.io) · landing site (Hugo) + reference docs (MkDocs Material under `/docs/latest/`)

---

<sub>Built on, and bet on, CNCF projects: etcd · NATS · CoreDNS · CubeFS · Longhorn · dex · zot · OpenTelemetry · VictoriaMetrics · Perses.</sub>
