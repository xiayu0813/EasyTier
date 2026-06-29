# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

EasyTier is a decentralized, peer-to-peer VPN written in Rust/Tokio. It creates a full-mesh network between devices using pluggable transport tunnels (TCP, UDP, WebSocket, WireGuard, QUIC, KCP, Fake TCP) with NAT traversal, subnet proxy, and OSPF-like intelligent routing. Licensed under LGPL-3.0.

## Build / Test / Lint Commands

### Core (default members: `easytier`, `easytier-web`)

```bash
# Build with default features
cargo build --release

# Build with all features
cargo build --features full

# Build a specific package
cargo build -p easytier --features full

# Run all tests (requires Linux; needs bridge-utils, br_netfilter, IPv6 on lo)
cargo test --no-default-features --features full

# Run tests with nextest (faster, used in CI)
cargo nextest archive --archive-file tests.tar.zst --package easytier --features full
cargo nextest run --archive-file tests.tar.zst

# Run a single test
cargo test --no-default-features --features full tests::three_node::three_node_test_bridge

# Run excluding three_node tests (CI partition 1)
cargo nextest run --archive-file tests.tar.zst -E 'not test(tests::three_node)' --test-threads 1 --no-fail-fast
```

### Linting

```bash
# Format check
cargo fmt --all -- --check

# Clippy (with all features)
cargo clippy --all-targets --features full --all -- -D warnings

# Feature combinations check
cargo hack check --package easytier --each-feature --exclude-features macos-ne --verbose

# Verify Cargo.lock is up to date
cargo metadata --format-version 1 --locked
```

### GUI / Mobile

```bash
# Build all frontends first
pnpm -r build

# GUI (Tauri)
cd easytier-gui && pnpm tauri build

# Android
cd easytier-gui && pnpm tauri android build
```

## Workspace Architecture

The workspace has 6 members (default: `easytier`, `easytier-web`):

| Crate | Purpose |
|---|---|
| `easytier/` | **Core**: library + 2 binaries (`easytier-core` daemon, `easytier-cli` CLI). Everything depends on this. |
| `easytier-web/` | AXUM web server with Vue 3 dashboard, SQLite/SeaORM config persistence. Depends on `easytier` lib. |
| `easytier-gui/src-tauri/` | Tauri 2 desktop app (Vue 3 frontend). Depends on `easytier` lib. |
| `easytier-contrib/easytier-ffi/` | C FFI bindings wrapping the dataplane. Depends on `easytier` lib. |
| `easytier-contrib/easytier-android-jni/` | JNI bindings atop `easytier-ffi` for Android. |
| `easytier-contrib/easytier-uptime/` | Standalone uptime monitoring web service with SQLite. Depends on `easytier` lib. |

Excluded from default build: `easytier-contrib/easytier-ohrs` (needs OpenHarmony SDK).

## Core Crate Internal Architecture (`easytier/src/`)

The core follows a layered design with a unified `Tunnel` trait for all transports:

1. **`tunnel/`** — Pluggable transport tunnels (TCP, UDP, WebSocket, WireGuard via boringtun, QUIC via quinn, KCP, Fake TCP with netfilter, ring, MPSC for testing). All implement `Tunnel`. Includes packet defs, stats, TLS helpers, compression (zstd).

2. **`gateway/`** — Protocol proxies bridging the virtual network to external protocols: SOCKS5 (fast_socks5 via smoltcp userspace TCP stack), ICMP proxy, IP fragment reassembly, KCP proxy, QUIC proxy, TCP/UDP proxies.

3. **`peers/` + `peer_center/`** — OSPF-like link-state routing (petgraph-based), peer state management, NAT traversal (UDP hole punching via `connector/`), encryption layer (`peers/encrypt/`: AES-GCM or WireGuard/Noise).

4. **`instance/`** — VPN node lifecycle, configuration, built-in DNS server (`instance/dns_server/`), OS-level DNS config (`instance/dns_server/system_config/`).

5. **`rpc_service/` + `proto/`** — Custom TARP RPC framework. Protobuf definitions in `proto/*.proto` are compiled by `build/main.rs` via `prost-build` + `prost-reflect-build`. RPC impls in `proto/rpc_impl/`.

6. **`vpn_portal/`** — WireGuard portal: exposes EasyTier network to standard WireGuard clients.

7. **`service_manager/`** — Cross-platform OS service management (install/uninstall/start/stop).

8. **`arch/`** — Platform abstraction layer. `common/ifcfg/` has Windows-specific network interface configuration.

9. **`tests/`** — Integration tests including multi-node setups with bridge-based networking (see `tests/three_node.rs`).

## Feature Flags

Key feature flags on the `easytier` crate (see `easytier/Cargo.toml` for full list):

- `default` = wireguard + websocket + smoltcp + tun + socks5 + kcp + quic + faketcp + magic-dns + zstd
- `full` = default + aes-gcm + openssl-crypto (used for testing)
- `ffi-dataplane` = socks5 only (stripped down for FFI embedding)
- `macos-ne` = macOS Network Extension compatibility (no deps, just cfg gates)
- `hotpath` / `hotpath-cpu` / `hotpath-alloc` = optional performance profiling via `hotpath` crate
- `tracing` = Tokio console tracing support
- `mimalloc` / `jemalloc` / `jemalloc-prof` = allocator selection

Use `--features full` for development and testing. Production builds use `--features default`.

## Key Technical Details

- **Rust edition 2024**, minimum Rust **1.95** (pinned in `rust-toolchain.toml`).
- **Release profile**: `panic = "abort"`, LTO enabled, single codegen unit, opt-level 3, stripped symbols.
- **Protobuf**: The build script (`easytier/build/main.rs`) generates code from 9 `.proto` files. Requires `protoc` >= 35.1. On CI, protoc is auto-downloaded on Windows. Do not edit generated code in `easytier/src/proto/*.rs` — edit the `.proto` files instead.
- **Allocator**: On Linux (non-MIPS), `jemalloc` is the default. On RISC-V, LoongArch, and ARM, `mimalloc` is used. macOS uses the system allocator.
- **Zero-copy**: The dataplane uses `zerocopy` + `bytes` for zero-copy throughout the packet processing pipeline.
- **Cross-compilation**: Uses `cargo-zigbuild` (Zig as linker) for most non-Windows/non-macOS targets. MIPS targets use `musl-cross` + `-Z build-std` + `RUSTC_BOOTSTRAP=1`. Windows links third-party DLLs from `easytier/third_party/`.
- **Integration tests** require Linux with `bridge-utils`, `br_netfilter` kernel module, IPv6 on loopback, and root privileges. CI partitions them into 3 groups for parallelism: non-three-node tests, three_node tests, and subnet_proxy_three_node tests.
- **Frontend**: pnpm workspace with Vue 3 + Vite + TypeScript + Tailwind. Build with `pnpm -r build` before building GUI or web packages that embed the frontend.
- **Conventional commits**: Use `feat:`, `fix:`, `docs:`, `test:`, `chore:` prefixes. PRs target the `develop` branch, not `main`.
- **Nix flake**: Provides dev shells (`nix develop` for core, `nix develop .#gui` for GUI, `nix develop .#full` for all deps including Android SDK).
