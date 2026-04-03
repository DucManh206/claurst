# Feature Flags Integration Examples

This document provides practical examples for using the GrowthBook feature flags system in Claude Code.

## 1. Voice Mode Feature Flag

### Example: Enable voice input when flag is on

File: `crates/cli/src/main.rs` (in `run_interactive` or command execution)

```rust
// Check if voice mode feature is enabled
if feature_flags.flag("voice_mode") {
    println!("Voice mode is enabled - use 'voice' command to record");
    // Initialize voice capture system
    voice_recorder::initialize();
} else {
    println!("Voice mode is disabled");
}
```

## 2. Experimental Features

### Example: Extended thinking

File: `crates/query/src/lib.rs` or within query execution

```rust
if feature_flags.flag("extended_thinking") {
    // Enable extended thinking for complex reasoning
    query_config.thinking_budget = Some(5000);
    println!("Extended thinking enabled for this query");
}
```

### Example: Experimental code analysis

File: `crates/tools/src/lib.rs` in file analysis tool

```rust
pub async fn analyze_code(&self, file_path: &str, ff: &FeatureFlagManager) -> ToolResult {
    let analysis = if ff.flag("advanced_code_analysis") {
        // Use new advanced analysis engine
        analyze_with_advanced_engine(file_path).await
    } else {
        // Use standard analysis
        analyze_standard(file_path).await
    };

    ToolResult::success(format!("Analysis: {}", analysis))
}
```

## 3. Beta Features Access

### Example: Gradual rollout

File: In tool execution context

```rust
use cc_core::FeatureFlagManager;

fn execute_command(&self, ff: &FeatureFlagManager) -> Result<String> {
    if ff.flag("beta_code_generation") {
        // New code generation with better formatting
        use_beta_code_gen()
    } else {
        // Stable code generation
        use_stable_code_gen()
    }
}
```

## 4. Performance Optimizations

### Example: Caching strategy

File: In cache management code

```rust
// Enable aggressive caching for specific workloads
if feature_flags.flag("enable_query_caching") {
    cache.set_retention_policy(Duration::from_secs(3600));
} else {
    // No caching
    cache.disable();
}
```

## 5. Telemetry and Diagnostics

### Example: Optional telemetry

File: In telemetry module

```rust
if feature_flags.flag("diagnostic_telemetry") {
    // Send detailed diagnostics
    send_detailed_metrics();
    trace!("Detailed diagnostics enabled");
} else {
    // Send only critical metrics
    send_critical_metrics_only();
}
```

## 6. UI/UX Features

### Example: New TUI features

File: `crates/tui/src/lib.rs` or `crates/tui/src/app.rs`

```rust
impl App {
    pub fn render_with_flags(&self, ff: &FeatureFlagManager) -> String {
        let mut ui = String::new();

        if ff.flag("new_sidebar_layout") {
            // Render new sidebar design
            ui.push_str(&self.render_new_sidebar());
        } else {
            // Use classic layout
            ui.push_str(&self.render_classic_sidebar());
        }

        if ff.flag("command_palette") {
            // Show command palette feature
            ui.push_str(&self.render_command_palette());
        }

        ui
    }
}
```

## 7. API Features

### Example: New MCP features

File: `crates/mcp/src/lib.rs`

```rust
pub async fn initialize_mcp(&self, ff: &FeatureFlagManager) -> Result<()> {
    if ff.flag("mcp_sse_support") {
        // Enable SSE (Server-Sent Events) for MCP
        self.enable_sse_transport().await?;
    }

    if ff.flag("mcp_resource_caching") {
        // Enable resource caching layer
        self.enable_resource_cache();
    }

    Ok(())
}
```

## 8. Database/Storage Features

### Example: New storage backend

File: In session storage code

```rust
pub async fn save_session(&self, session: &Session, ff: &FeatureFlagManager) -> Result<()> {
    if ff.flag("use_sqlite_backend") {
        // Use new SQLite backend
        self.save_to_sqlite(session).await
    } else {
        // Use JSONL format (current)
        self.save_to_jsonl(session).await
    }
}
```

## 9. Plugin System Features

### Example: Plugin capabilities

File: `crates/plugins/src/lib.rs`

```rust
pub fn get_available_plugin_features(&self, ff: &FeatureFlagManager) -> Vec<&str> {
    let mut features = vec!["basic"];

    if ff.flag("plugin_hooks") {
        features.push("hooks");
    }

    if ff.flag("plugin_middleware") {
        features.push("middleware");
    }

    features
}
```

## 10. Multi-Modal Features

### Example: Image generation

File: In tool execution

```rust
if feature_flags.flag("image_generation") {
    // Enable DALL-E or similar image generation
    enable_image_generation_tool();
} else {
    // Only text-based features
    println!("Image generation is not available");
}
```

## Integration Checklist

When adding a new feature behind a flag:

- [ ] Define the feature in GrowthBook dashboard
- [ ] Add documentation to FEATURE_FLAGS.md
- [ ] Check the flag at decision point: `feature_flags.flag("feature_name")`
- [ ] Provide sensible defaults (flags default to `false`)
- [ ] Test with both enabled and disabled states
- [ ] Log feature flag state in verbose mode
- [ ] Document the feature in the appropriate crate's README

## Testing Features with Flags

### Local Override (Development)

For testing, you can create a local cached flags file manually:

```json
{
  "flags": {
    "voice_mode": true,
    "extended_thinking": true,
    "new_sidebar_layout": true
  },
  "fetched_at": 9999999999
}
```

Save to: `~/.claude/feature_flags.json`

### Using GrowthBook API Directly

You can test the API response locally:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  https://api.growthbook.io/api/features
```

This helps verify your feature flags are correctly configured in GrowthBook.

## Performance Considerations

1. **Cache hits are instant** - In-memory HashMap lookup (O(1))
2. **Cache refresh is async** - Doesn't block startup
3. **API timeout is 10 seconds** - Won't hang application
4. **Graceful fallback** - Uses stale cache if API fails

## Monitoring

To monitor feature flag usage in production:

```rust
if feature_flags.flag("enable_query_caching") {
    debug!("Using advanced query caching");
    // Feature enabled - log for monitoring
}
```

Check logs to see which flags are being used and how often they're evaluated.
