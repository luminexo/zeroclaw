# Issue #3569: Hot-Reload Config Without Daemon Restart

## Summary
Add `zeroclaw config reload` CLI command and `POST /admin/reload-config` endpoint to hot-reload `config.toml` into a running gateway without restarting the daemon, preserving conversation history.

## Current Architecture Analysis

### Existing Infrastructure
ZeroClaw already has partial config reload infrastructure in `src/channels/mod.rs`:

1. **`runtime_config_store()`** - Global state for tracking config file stamps
2. **`maybe_apply_runtime_config_update()`** - Detects and applies config changes to provider cache
3. **`load_runtime_defaults_from_config_file()`** - Loads config from disk, decrypts secrets, applies env overrides
4. **Admin endpoints** - `/admin/shutdown`, `/admin/paircode`, `/admin/paircode/new` already exist in `src/gateway/mod.rs`

### Gaps
- No explicit `/admin/reload-config` endpoint
- No `zeroclaw config reload` CLI subcommand
- Runtime config reload only triggers on channel message processing, not on-demand

## Proposed Implementation

### 1. Gateway Endpoint: `POST /admin/reload-config`

**Location**: `src/gateway/mod.rs`

```rust
/// POST /admin/reload-config — hot-reload config.toml (localhost only)
async fn handle_admin_reload_config(
    State(state): State<Arc<GatewayState>>,
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
) -> Result<Json<AdminReloadConfigResponse>, AdminError> {
    // Verify localhost origin
    if !is_localhost(&addr) {
        return Err(AdminError::Forbidden);
    }

    // Load config from disk
    let config_path = state.config.config_path.clone();
    let config = load_config_from_path(&config_path).await?;

    // Apply runtime updates (existing infrastructure)
    apply_config_update(&state, config).await?;

    Ok(Json(AdminReloadConfigResponse {
        success: true,
        message: "Config reloaded successfully".into(),
    }))
}
```

**Registration** (in `build_gateway_app`):
```rust
.route("/admin/reload-config", post(handle_admin_reload_config))
```

### 2. CLI Command: `zeroclaw config reload`

**Location**: `src/main.rs`

Add to `ConfigCommands` enum:
```rust
enum ConfigCommands {
    /// Dump the full configuration JSON Schema to stdout
    Schema,
    /// Hot-reload config.toml into running gateway
    Reload,
}
```

**Implementation**:
```rust
ConfigCommands::Reload => {
    let gateway = crate::gateway::connect_to_gateway().await?;
    let response = gateway
        .post("/admin/reload-config")
        .send()
        .await?
        .json::<AdminReloadConfigResponse>()
        .await?;
    
    if response.success {
        println!("✅ {}", response.message);
    } else {
        bail!("Failed to reload config: {}", response.message);
    }
}
```

### 3. Config Reload Logic

**Location**: `src/channels/mod.rs` (extend existing)

The current `maybe_apply_runtime_config_update()` handles:
- Provider updates (default provider, API key, URL)
- Provider warmup
- Cache invalidation

For full reload, also need:
- Channel config updates (Telegram, Discord, etc.)
- MCP server config updates
- Skills config updates
- Memory config updates

**Approach**: Extend `ChannelRuntimeContext` to hold a mutable config reference:
```rust
pub struct ChannelRuntimeContext {
    // ... existing fields ...
    pub config: Arc<RwLock<Config>>,
}
```

## Acceptance Criteria Mapping

| Criterion | Implementation |
|-----------|----------------|
| `zeroclaw config reload` succeeds when gateway is running | CLI command + endpoint |
| Config changes take effect without daemon restart | Reuse existing `apply_config_update` |
| Conversation history preserved | No state clear, just config swap |
| Non-localhost requests rejected (403) | `is_localhost()` check |

## Files Changed

1. `src/gateway/mod.rs` - Add `/admin/reload-config` endpoint
2. `src/main.rs` - Add `ConfigCommands::Reload` variant
3. `src/channels/mod.rs` - (optional) Extend config update scope

## Risk Assessment

- **Low risk**: Additive change, existing endpoints untouched
- **Rollback**: Remove route + CLI variant
- **Breaking change**: None

## Implementation Steps

1. [ ] Add `AdminReloadConfigResponse` struct
2. [ ] Implement `handle_admin_reload_config` handler
3. [ ] Register route in `build_gateway_app`
4. [ ] Add `ConfigCommands::Reload` to CLI
5. [ ] Add integration tests
6. [ ] Update documentation

## Open Questions

1. Should reload also update channel configs (requires channel restart)?
2. Should reload validate config before applying (fail fast)?
3. Should there be a diff preview before applying?

## References

- Issue: https://github.com/zeroclaw-labs/zeroclaw/issues/3569
- Existing reload: `src/channels/mod.rs:maybe_apply_runtime_config_update()`
- Admin endpoints: `src/gateway/mod.rs:handle_admin_shutdown()`