# Rust Port Improvement Roadmap

**Goal:** Close remaining 85% → 95%+ feature parity gaps
**Current State:** All core systems working; missing pieces are UI polish and edge features
**Total Effort to 95%:** ~60-80 hours

---

## PRIORITY TIER SYSTEM

### Tier 1: HIGH IMPACT (Do First)
- User-facing gaps affecting multiple workflows
- >20 hours ROI
- Medium-high effort but worthwhile

### Tier 2: MEDIUM IMPACT (Do Second)
- Nice-to-have features
- 10-20 hours effort
- Improves UX polish

### Tier 3: LOW IMPACT (Do Last)
- Edge cases or rarely-used features
- <10 hours effort
- Can be deferred indefinitely

---

## TIER 1: HIGH IMPACT IMPROVEMENTS

### 1. Message Syntax Highlighting in Code Blocks

**Current Gap:** Code blocks in messages appear with plain borders, no syntax coloring.

**User Impact:** Messages with code examples are harder to read. Every code-heavy conversation is visually degraded.

**Why It Matters:** This is the #1 visual polish gap. Affects ~40% of conversations.

**Implementation:**
```
File: src-rust/crates/tui/src/messages/markdown.rs (new section)
1. Extend message rendering to detect ```language blocks
2. Use `syntect` (already used in diff viewer) to highlight code
3. Cache highlighted blocks by hash to avoid re-parsing
4. Apply Rust color scheme matching theme selection
```

**Technical Debt:** Diff viewer already does this; message rendering should follow same pattern.

**Effort:** 8-12 hours
- [ ] Add CodeBlock variant to markdown AST
- [ ] Integrate syntect highlighting
- [ ] Test with multiple languages
- [ ] Handle theme color mapping
- [ ] Performance test on large conversations

**Blockers:** None. All dependencies already present.

**Priority:** DO FIRST — This is the most visible gap.

---

### 2. Specialized Permission Dialogs (43 Variants)

**Current Gap:** 4 generic dialog types instead of 43 context-specific variants.

**User Impact:** Users see generic "Allow?" dialogs instead of context-specific prompts (e.g., "IDE auto-connect?", "Cost threshold exceeded?").

**Why It Matters:** UX completeness. Users should understand WHY they're being asked.

**Implementation:**
```
File: src-rust/crates/tui/src/dialogs/mod.rs (expand)
1. Extract all 43 permission types from TypeScript source
2. Map each to a specialized dialog UI
3. Route permission requests to correct dialog variant
4. Examples:
   - FileWritePermission → "Save file X?"
   - BashCommand → "Run bash: [command]?"
   - MCP_Server → "Approve MCP: [server]?"
   - IdleAI → "Run proactive AI?"
   - CostThreshold → "Token overage: $X.XX?"
   - WebsiteAccess → "Fetch from [domain]?"
```

**Affected Use Cases:**
- Nested permission grants (MCP servers, teams)
- Cost warnings
- IDE auto-connect prompts
- Bridge security dialogs

**Effort:** 20-25 hours
- [ ] Audit all 43 TS dialog types
- [ ] Design Rust permission dialog abstraction
- [ ] Implement specialized variants (5-10 hours)
- [ ] Route all permission requests through specialized dialogs (5-10 hours)
- [ ] Test each dialog path
- [ ] Update rendering in render.rs

**Blockers:** Need to map TS permission context to Rust permission types.

**Priority:** DO SECOND — Affects 30%+ of permission interactions.

---

### 3. Task Management Display Overlay

**Current Gap:** Task create/update/list tools work, but no UI shows background task progress.

**User Impact:** Users spawning long-running background tasks can't see progress.

**Why It Matters:** Power users with multi-task workflows are blocked.

**Implementation:**
```
File: src-rust/crates/tui/src/overlays/task_manager.rs (new)
1. Show task list overlay with:
   - Task ID, status (pending/in_progress/completed)
   - Task description
   - Progress % for long-running tasks
   - Estimated time remaining
   - Cancel button

2. Add keybinding: Ctrl+T to toggle task overlay

3. Real-time update as tasks progress
   - Hook into TaskUpdateTool to refresh display
   - Show completion message on finish
   - Auto-hide when all tasks done (with option to keep visible)
```

**Affected Use Cases:**
- Spawning 5+ sub-agents
- Long-running analyses
- Batch operations

**Effort:** 12-15 hours
- [ ] Design task display UI
- [ ] Create overlay rendering
- [ ] Wire task state into TUI
- [ ] Add keybinding for task overlay
- [ ] Test with concurrent tasks

**Blockers:** Need task state available to TUI in real-time (may require channel from query loop).

**Priority:** DO THIRD — Affects power users only; workaround is available (check task list manually).

---

## TIER 2: MEDIUM IMPACT IMPROVEMENTS

### 4. WebFetch Semantic Extraction

**Current Gap:** WebFetch HTML→text conversion works, but LLM semantic extraction skipped.

**User Impact:** Fetching complex websites sometimes returns unstructured HTML instead of clean extracted content.

**Why It Matters:** ~5-10% of fetches could be improved with semantic extraction.

**Implementation:**
```
File: src-rust/crates/tools/src/web_fetch.rs (enhance)
1. Add optional --semantic flag to WebFetch tool
2. When enabled:
   - Run fetched HTML through Claude (small-cost model like haiku)
   - Extract key content (article text, data tables, etc.)
   - Cache semantic extraction by URL hash
3. Default behavior: keep current HTML→text (fast)
```

**Affected Use Cases:**
- Fetching news articles (dynamic site structures)
- Fetching data tables
- Fetching API documentation

**Effort:** 4-6 hours
- [ ] Add semantic extraction logic to WebFetchTool
- [ ] Wire optional extraction to query phase
- [ ] Cache results
- [ ] Test with various website types

**Blockers:** None. Can use existing Anthropic API client.

**Priority:** DEFER — Current behavior is 95% functional. Semantic extraction is nice-to-have.

---

### 5. Image Display: Sixel Protocol

**Current Gap:** Kitty graphics protocol only. Sixel (legacy but still used) not supported.

**User Impact:** On older terminals, images display as text fallback instead of sixel graphics.

**Why It Matters:** <5% of users on legacy terminals. Modern terminals (alacritty, kitty, wezterm) all support Kitty.

**Implementation:**
```
File: src-rust/crates/tui/src/kitty_image.rs (expand)
1. Add sixel encoding support via `sixel-rs` crate
2. Detect terminal capability at startup
3. Use Kitty if supported, else Sixel, else text fallback
4. Add feature flag: --enable-sixel
```

**Affected Use Cases:**
- Running on legacy xterm/gnome-terminal
- Xfce, MATE, or old SSH terminals
- Retro enthusiasts

**Effort:** 3-4 hours
- [ ] Add sixel-rs crate to Cargo.toml
- [ ] Implement sixel encoder
- [ ] Add terminal capability detection
- [ ] Test fallback chain

**Blockers:** sixel-rs dependency. May have binary size impact.

**Priority:** DEFER — Kitty is sufficient for 95%+ of users.

---

## TIER 3: LOW IMPACT / EDGE CASES

### 6. Remote Feature Flags (GrowthBook Integration)

**Current Gap:** Compile-time feature flags only. No runtime feature gating service.

**User Impact:** Internal/experimental features are build-time decisions, not runtime toggles.

**Why It Matters:** Anthropic can't A/B test features or roll out gradually.

**Implementation:**
```
Requires:
1. GrowthBook API client
2. Feature flag polling at startup
3. Runtime feature gate checks throughout codebase
4. Complex integration (affects most commands)
```

**Effort:** 15-20 hours
**ROI:** Internal only, not user-facing.
**Priority:** INTERNAL ONLY — Defer unless needed for experiments.

---

### 7. Prompt Template System (MCP)

**Current Gap:** MCP prompts/list and prompts/get are implemented, but not exposed to user.

**User Impact:** Users can't define reusable prompt templates via MCP.

**Why It Matters:** Advanced power users want this; most users won't notice.

**Implementation:**
```
1. Add /mcp-prompts command to list available templates
2. Allow prompt template substitution in system prompt
3. Wire template rendering in query construction phase
```

**Effort:** 8-10 hours
**Priority:** INTERNAL ONLY — Defer unless MCP prompt templates are heavily used.

---

### 8. Resource Subscription (True Pub/Sub vs Polling)

**Current Gap:** Rust uses polling loop. TS uses true subscription.

**User Impact:** MCP resource updates checked every 5s instead of pushed immediately.

**Why It Matters:** Negligible (most MCP resources are static files).

**Implementation:**
```
1. Requires MCP server to support resource/subscribe RPC
2. Add subscription channel instead of poll loop
3. Async stream handling
```

**Effort:** 6-8 hours
**Priority:** DEFER — Polling is sufficient for resource changes.

---

## QUICK WINS (Can Do in 1-2 Hours)

### Rename CLI Arguments
- Add alias support (--model ≈ -m) [already done, skip]

### Completeness
- Audit any remaining stub implementations [done: 0 found]

### Documentation
- Add command examples to /help output
- Document permission dialog types

**Effort:** 2-3 hours total

---

## IMPLEMENTATION ROADMAP (If Prioritizing to 95%)

**Phase 1 (Weeks 1-2): Tier 1 improvements**
```
Week 1:
- Mon-Tue: Message syntax highlighting (8-10h)
- Wed-Fri: Specialized permission dialogs (12-15h)

Week 2:
- Mon-Tue: Task management overlay (8-10h)
- Wed-Fri: Testing, integration, bugfixes (10h)

Estimated: 50 hours → 92% feature parity
```

**Phase 2 (Week 3): Tier 2 improvements**
```
- Mon: WebFetch semantic extraction (4h)
- Tue-Wed: Sixel image protocol (3h)
- Thu-Fri: Testing, integration (4h)

Estimated: 12 hours → 94% feature parity
```

**Phase 3 (After): Tier 3 improvements**
```
- Remote feature flags (20h, internal only)
- MCP prompt templates (8h, advanced only)
- Resource true pub/sub (8h, optional)

Estimated: 36 hours → 97% feature parity
```

**Total Effort to 95%:** ~65 hours
**Total Effort to 99%:** ~100+ hours (diminishing returns)

---

## RESOURCE ALLOCATION ESTIMATE

Assuming 1 engineer working full-time:

| Target | Time | Feasibility |
|--------|------|------------|
| 90% (current) | Done | ✅ Production-ready |
| 93% (Tier 1 only) | 2 weeks | ✅ Recommended |
| 95% (Tier 1+2) | 3 weeks | ✅ High ROI |
| 97% (All user-facing) | 4 weeks | ✅ Complete |
| 99% (All features) | 6+ weeks | ⚠️ Diminishing returns |

---

## RECOMMENDATION

**For Production Readiness:**
- ✅ Current state (90%) is READY. Ship it.
- Tier 1 improvements are nice-to-have but not blocking.

**For Feature Parity (95%+):**
- Do Tier 1 (message highlighting + specialized dialogs)
- Skip Tier 2 & 3 unless specifically needed

**For Complete Parity (99%+):**
- Diminishing returns. Consider opportunity cost vs new features.

---

## ACTION ITEMS

**Immediate:**
- [ ] Read full RUST_PORT_DEEP_ANALYSIS.md
- [ ] Prioritize by business impact (which gaps matter most?)
- [ ] Assign Tier 1 work to 1-2 engineers

**Short-term (Next Sprint):**
- [ ] Message syntax highlighting (start here — highest ROI)
- [ ] Specialized permission dialogs (affects UX completeness)
- [ ] Task overlay for power users

**Medium-term:**
- [ ] WebFetch semantic extraction
- [ ] Sixel image support
- [ ] Full test coverage of new features

**Long-term:**
- [ ] Defer Tier 3 unless explicitly needed

---

## SUCCESS CRITERIA

| Metric | Current | Target | Status |
|--------|---------|--------|--------|
| Tests passing | 1,008/1,008 | 1,050+/1,050+ | Add tests for new features |
| Commands working | 70/70 | 70/70 | ✅ No change |
| Tools complete | 38/40 | 40/40 | Only if Tier 1 adds tests |
| Feature parity | 90% | 95%+ | Tier 1 achieves this |
| Production ready | ✅ Yes | ✅ Yes | Already ready; improvements are polish |
| Visual parity with TS | 85% | 95%+ | Tier 1 achieves this |

