# Feature Flags Integration with GrowthBook

This document describes the feature flags system integrated with GrowthBook for server-driven feature control in Claude Code.

## Overview

The `FeatureFlagManager` in `crates/core/src/feature_flags.rs` provides:

1. **Remote flag fetching** - Pulls flags from GrowthBook API
2. **Local caching** - Stores flags in `~/.claude/feature_flags.json` with 1-hour TTL
3. **Graceful fallback** - Uses cached flags if API is unavailable
4. **Simple API** - Check flags with `flag(name: &str) -> bool`

## Setup

### Environment Configuration

Set the GrowthBook API key:

```bash
export GROWTHBOOK_API_KEY="your-api-key-here"
```

The API key is read from the `GROWTHBOOK_API_KEY` environment variable. If not set, the system will still work but will only use cached flags.

### GrowthBook Configuration

1. Create a GrowthBook account at https://www.growthbook.io/
2. Generate an API key from your GrowthBook project settings
3. Configure feature flags in GrowthBook's UI
4. The API endpoint is: `https://api.growthbook.io/api/features`

## Usage

### In the CLI

The `FeatureFlagManager` is initialized on startup in `crates/cli/src/main.rs`:

```rust
// Initialize feature flags from GrowthBook
let feature_flags = FeatureFlagManager::new();
if let Err(e) = feature_flags.fetch_flags_async().await {
    debug!("Failed to initialize feature flags: {}", e);
    // Non-fatal error: continue with defaults
}

// Pass to run modes
run_headless(..., feature_flags)
run_interactive(..., feature_flags)
```

### Checking Flags in Code

To check if a feature is enabled:

```rust
use cc_core::FeatureFlagManager;

// Check a flag
if feature_flags.flag("voice_mode") {
    // Enable voice capture
    start_voice_capture();
}

if feature_flags.flag("experimental_ai_reasoning") {
    // Enable extended thinking
    enable_extended_thinking();
}
```

### Example: Voice Mode Feature

```rust
// In command execution
if ctx.feature_flags.flag("voice_mode") {
    println!("Voice mode is enabled");
    // Initialize voice capture
} else {
    println!("Voice mode is disabled");
}
```

## Caching

### Cache Location
- **Path**: `~/.claude/feature_flags.json`
- **Format**: JSON with flags and fetch timestamp
- **TTL**: 1 hour

### Cache Behavior

1. **On startup**: Manager checks for cached flags
2. **If cache is fresh** (< 1 hour old): Uses cache immediately
3. **If cache is stale**: Fetches from GrowthBook API
4. **If API fails**: Falls back to stale cache if available
5. **If no cache**: Continues with all flags defaulting to false

## API Reference

### `FeatureFlagManager`

#### `new() -> Self`
Creates a new feature flag manager. The API key is read from `GROWTHBOOK_API_KEY` env var.

```rust
let manager = FeatureFlagManager::new();
```

#### `flag(name: &str) -> bool`
Checks if a feature flag is enabled. Returns `false` for unknown flags.

```rust
let enabled = manager.flag("feature_name");
```

#### `fetch_flags_async() -> Result<()>`
Fetches flags from the GrowthBook API (or uses cache if available). Should be called during initialization.

```rust
manager.fetch_flags_async().await?;
```

## GrowthBook API Response Format

The system expects the GrowthBook API to return features in this format:

```json
{
  "features": [
    {
      "id": "feature-id-1",
      "key": "voice_mode",
      "enabled": true,
      "variant": "treatment"
    },
    {
      "id": "feature-id-2",
      "key": "experimental_reasoning",
      "enabled": false
    }
  ]
}
```

## Error Handling

The system is designed to be resilient:

- **Missing API key**: Works with cached flags only
- **API timeout**: Uses cached flags as fallback
- **Invalid response**: Logs error and uses cached flags
- **Missing cache**: Continues with all flags disabled

This ensures Claude Code continues to function even if feature flag retrieval fails.

## Testing

### Running Tests

```bash
cd src-rust
cargo test -p cc-core feature_flags
```

### Test Cases

1. **Cache path generation** - Verifies correct path construction
2. **Default flag behavior** - Flags default to false when not found
3. **Cache validity** - Checks TTL logic correctly identifies fresh vs stale cache

## Integration Points

The feature flags are passed through the application at startup:

1. **Main function** (`crates/cli/src/main.rs`)
   - Initializes `FeatureFlagManager`
   - Calls `fetch_flags_async()`
   - Passes to both headless and interactive modes

2. **Headless mode** (`run_headless`)
   - Receives `feature_flags` parameter
   - Available for tool execution

3. **Interactive mode** (`run_interactive`)
   - Receives `feature_flags` parameter
   - Available for TUI and command execution

## Security Considerations

1. **API Key Storage**: The API key is read from environment variables only, never hardcoded
2. **HTTPS Only**: All API calls use HTTPS to GrowthBook's secure endpoint
3. **Cache Permissions**: Cache file is stored in user's home directory with standard permissions
4. **Timeout**: API requests timeout after 10 seconds to prevent hanging

## Future Enhancements

Possible future improvements:

1. **User attributes** - Support feature flag targeting based on user properties
2. **Real-time updates** - WebSocket support for instant flag changes
3. **Metrics tracking** - Log flag evaluation metrics
4. **Local overrides** - Environment variable overrides for development
5. **Feature flag rollouts** - Support gradual rollout percentages
