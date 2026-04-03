# MCP Resource Prompt Templates Implementation

## Overview

Added template rendering for MCP resource prompts with variable substitution, enabling dynamic prompt generation from resource properties.

## Files Created

### 1. `/src-rust/crates/core/src/mcp_templates.rs`
- **TemplateRenderer** struct with template rendering and variable substitution
- Supports `{{variable}}` syntax for simple fields
- Supports `{{nested.path}}` syntax for nested JSON objects
- Gracefully handles missing variables by leaving placeholders in place
- Comprehensive unit tests included in the module

Key functions:
- `TemplateRenderer::render(template: &str, context: &Value) -> String`
  - Parses template for `{{...}}` placeholders
  - Substitutes with values from JSON context
  - Returns rendered string with substitutions applied

- `get_nested_value(value: &Value, path: &str) -> Option<Value>`
  - Supports dot-notation for nested paths (e.g., `meta.author`)
  - Returns None for missing paths

- `value_to_string(value: &Value) -> String`
  - Converts JSON values to string representation
  - Handles strings, numbers, booleans, arrays, objects, null

## Files Modified

### 1. `/src-rust/crates/core/src/lib.rs`
- Added module declaration: `pub mod mcp_templates;`
- Placed in utilities section alongside other core modules

### 2. `/src-rust/crates/mcp/src/lib.rs`
- Added import: `use cc_core::mcp_templates::TemplateRenderer;`
- Extended `McpResource` struct with optional `annotations` field
- Updated `list_resources()` method to apply template rendering
- Template logic:
  1. Checks each resource for `annotations.prompt` field
  2. If found, builds context from resource properties (uri, name, description, mimeType)
  3. Renders the template using TemplateRenderer
  4. Replaces the resource description with rendered output

## Example Usage

### Resource with Prompt Template

```json
{
  "uri": "file:///config.db",
  "name": "Configuration Database",
  "description": null,
  "mimeType": "application/json",
  "annotations": {
    "prompt": "Use {{name}} located at {{uri}} to {{description}}"
  }
}
```

When `list_resources()` is called:
1. Detects `annotations.prompt` field
2. Builds context: `{ "uri": "file:///config.db", "name": "Configuration Database", "description": null, "mimeType": "application/json" }`
3. Renders: `"Use Configuration Database located at file:///config.db to null"`
4. Sets as resource description

### Nested Path Example

Template: `"Created by {{meta.author}} on {{meta.created_date}}"`

With context containing nested `meta` object:
- `{{meta.author}}` substitutes with the author value
- `{{meta.created_date}}` substitutes with the date value

## Testing

### Unit Tests
- Located in `mcp_templates.rs` module (tests module at end of file)
- 8 comprehensive tests covering:
  - Simple substitution
  - Nested paths
  - Missing variables
  - No variables
  - Number and boolean values
  - Array values
  - Empty context

### Integration Tests
- Located in `crates/core/tests/test_mcp_templates.rs`
- 6 tests demonstrating real-world usage:
  - `test_simple_substitution()` - Basic {{var}} replacement
  - `test_nested_path()` - {{nested.path}} replacement
  - `test_missing_variable()` - Handling of undefined variables
  - `test_resource_context()` - Simulated MCP resource context
  - `test_multiple_occurrences()` - Multiple instances of same variable
  - `test_numeric_and_bool_values()` - Type conversions

All tests pass successfully.

## Compilation

```bash
cd src-rust
cargo check -p cc-core    # Passes without errors
cargo check -p cc-mcp     # Passes without errors
cargo test --test test_mcp_templates  # All 6 tests pass
```

## Design Decisions

1. **Lazy Substitution**: Variables are substituted left-to-right, single pass. Prevents infinite loops with recursive templates.

2. **Missing Variable Handling**: Undefined variables are left as-is (`{{variable}}`). This is safer than erroring and allows partial templating.

3. **Nested Path Format**: Dot notation (`meta.author`) chosen for readability and compatibility with JSON path conventions.

4. **Context Building**: Only creates context from resource fields when processing resources, not from all server config.

5. **Optional Annotations**: The `annotations` field is optional on resources. Not all MCP servers will use this feature.

## Future Enhancements

1. **Conditional Logic**: Support `{{#if condition}}...{{/if}}` syntax
2. **Filters**: Support `{{variable | uppercase}}` syntax
3. **Default Values**: Support `{{variable | default:"N/A"}}` syntax
4. **Circular Reference Detection**: Detect and prevent infinite loops in nested templates
5. **Custom Functions**: Allow registering custom substitution functions

## Integration Points

The template rendering is integrated into the resource listing flow:

```
MCPClient::list_resources()
  → calls MCP server "resources/list"
  → receives ListResourcesResult with resources
  → for each resource:
      - check for annotations.prompt
      - if present, render template with resource context
      - replace description with rendered output
  → return modified resources
```

This integration is transparent to callers - they receive resources with descriptions already populated from templates if available.
