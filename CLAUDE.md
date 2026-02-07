# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Bifrost is a Philips Hue Bridge emulator written in Rust that enables control of Zigbee2MQTT smart lights via Hue-compatible apps. It provides both V1 (legacy) and V2 (modern) Hue APIs.

## Build Commands

```bash
# Build
cargo build
cargo build --release

# Run tests (unit tests are spread across crates)
cargo test
cargo test --workspace              # Test all workspace crates
cargo test -p hue                   # Test specific crate
cargo test colorspace               # Run tests matching name

# Clippy (project uses strict lints - all clippy groups enabled as warnings)
cargo clippy --workspace

# Run the application (requires config.yaml in current directory)
cargo run
```

## Architecture

### Workspace Crates

- **bifrost** (src/) - Main application: HTTP/HTTPS servers, routing, state management
- **hue** (crates/hue/) - Philips Hue data models for V1/V2 APIs, color space conversions, ZCL encodings
- **z2m** (crates/z2m/) - Zigbee2MQTT message types and WebSocket protocol
- **zcl** (crates/zcl/) - Zigbee Cluster Library frame serialization
- **svc** (crates/svc/) - Custom service manager (systemd-inspired) for managing application services
- **bifrost-api** (crates/bifrost-api/) - Shared configuration and backend request types

### Core Components

**AppState** (`src/server/appstate.rs`) - Central application state containing config, resources, service manager, and version updater.

**Resources** (`src/resource.rs`) - State database managing all Hue resources (lights, groups, scenes, entertainment zones). Persists to YAML (`state.yaml`). Provides broadcast channels for state updates and backend events.

**Service Manager** (`crates/svc/`) - Manages running services via RPC. Services include: HTTP, HTTPS, mDNS, SSDP, Entertainment (DTLS), and Z2M backends. Supports service templates for dynamic Z2M instances.

**Z2M Backend** (`src/backend/z2m/`) - WebSocket connection to Zigbee2MQTT servers. Handles device discovery, state sync, and entertainment streaming. Uses `ServiceTemplate` pattern for multiple Z2M server instances.

**Routes** (`src/routes/`) - Axum HTTP handlers:
- `/api` - V1 legacy API
- `/clip/v2/resource` - V2 modern API
- `/eventstream` - Server-sent events
- `/bifrost` - Bifrost-specific endpoints

### Key Patterns

- **Event-driven updates**: Resources emit events via `tokio::sync::broadcast` channels for real-time state propagation
- **State versioning**: YAML state files include version for migrations (see `src/model/state.rs`)
- **Certificate from MAC**: Bridge identity derived from network interface MAC address
- **Throttled writes**: Config writer debounces rapid state changes before persisting
