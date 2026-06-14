# OpenClaw Visualizer — Implementation Spec

> Status: Ready to implement
> Scope: Phase 1 — Admin API key path (no OpenClaw source changes required)

## What We're Building

A CodexBar provider that shows OpenClaw's Anthropic API spend in the menubar
alongside Claude Code, using an Anthropic Admin API key to query org-level usage.

---

## Environment

- **Machine:** Samir's MacBook Pro (arm64, macOS Darwin 24.6.0)
- **Repo:** `~/code/CodexBar` — fork of `steipete/CodexBar` at `smrghsh/CodexBar`
- **Running instance:** Built from source, launched from `~/code/CodexBar/CodexBar.app`
- **Rebuild command:** `cd ~/code/CodexBar && ./Scripts/compile_and_run.sh`
- **OpenClaw state dir:** `~/.openclaw/`
- **OpenClaw API key location:** `~/.openclaw/openclaw.json` (field: `auth.profiles`)

---

## Phase 1 — Anthropic Admin API (ship this first)

### How it works

Reuse `ClaudeAdminAPIUsageFetcher` with the user's Anthropic Admin API key
(`sk-ant-admin-...`). This hits two org-level endpoints:

```
GET https://api.anthropic.com/v1/organizations/cost_report?group_by[]=description&bucket_width=1d&limit=31
GET https://api.anthropic.com/v1/organizations/usage_report/messages?group_by[]=model&bucket_width=1d&limit=31
```

Headers: `x-api-key: <admin-key>`, `anthropic-version: 2023-06-01`

This returns **all** Anthropic org spend — OpenClaw + Claude Code combined.
That's fine for Phase 1. Phase 2 (local JSONL) adds per-source attribution.

### What to display

- **Menubar:** Today's cost in USD (e.g. `🦞 $2.14`)
- **Expanded menu:**
  - Today / 7-day / 30-day cost
  - Per-model token breakdown (input / output / cache)
  - Top models by spend
- **No quota bar** — this is pay-as-you-go, cost only

### Files to create

```
Sources/CodexBarCore/Providers/OpenClaw/
  OpenClawProviderDescriptor.swift    — register provider, wire fetch plan
  OpenClawAdminAPIFetchStrategy.swift — thin wrapper around ClaudeAdminAPIUsageFetcher
  OpenClawUsageSnapshot.swift         — toUsageSnapshot() → cost-only UsageSnapshot
  OpenClawSettingsReader.swift        — read admin key from env or settings
```

### Enum additions

**`Providers.swift`** — add:
```swift
case openclaw
```

**`IconStyle`** — add:
```swift
case openclaw
```

Add icon asset `ProviderIcon-openclaw` (suggest: lobster 🦞 or claw mark).
Color: `ProviderColor(red: 0.85, green: 0.30, blue: 0.20)` — warm red.

### Credential wiring

Read admin key from (in priority order):
1. `OPENCLAW_ANTHROPIC_ADMIN_KEY` env var
2. CodexBar settings pane (user pastes it in)
3. `~/.openclaw/openclaw.json` — check `auth.profiles.anthropic` for any admin-scoped key

### Descriptor sketch

```swift
ProviderDescriptor(
    id: .openclaw,
    metadata: ProviderMetadata(
        displayName: "OpenClaw",
        toggleTitle: "Show OpenClaw API usage",
        defaultEnabled: false,
        isPrimaryProvider: false,
        dashboardURL: "https://console.anthropic.com/settings/usage",
        supportsCredits: false,
        supportsOpus: false),
    branding: ProviderBranding(
        iconStyle: .openclaw,
        iconResourceName: "ProviderIcon-openclaw",
        color: ProviderColor(red: 0.85, green: 0.30, blue: 0.20)),
    tokenCost: ProviderTokenCostConfig(
        supportsTokenCost: true,
        noDataMessage: {
            "Paste an Anthropic Admin API key in settings to see OpenClaw usage."
        }),
    fetchPlan: ProviderFetchPlan(
        sourceModes: [.auto, .api],
        pipeline: ProviderFetchPipeline(resolveStrategies: resolveStrategies)))
```

---

## Phase 2 — Local JSONL Log (adds per-source attribution)

Once Phase 1 ships, add a local log scanner so OpenClaw usage is isolated from
Claude Code spend on the same Anthropic account.

### Log contract

OpenClaw writes one line per API response to:
```
~/.openclaw/usage/anthropic-usage.jsonl
```

Line format:
```json
{
  "v": 1,
  "ts": "2026-06-14T09:31:02.184Z",
  "request_id": "req_011...",
  "model": "claude-sonnet-4-6",
  "usage": {
    "input_tokens": 1234,
    "output_tokens": 567,
    "cache_read_input_tokens": 8901,
    "cache_creation_input_tokens": 0,
    "cache_creation_input_tokens_1h": 0
  },
  "source": "openclaw",
  "session_id": "optional-conversation-id"
}
```

### Scanner

Add `CostUsageScanner+OpenClaw.swift` modeled on `CostUsageScanner+Claude.swift`:
- Resolves log path from `OPENCLAW_STATE_DIR` → `~/.openclaw`
- Incremental scan by byte offset (mtime/size cache — same as Claude scanner)
- Dedup by `request_id`
- Prices via existing `CostUsagePricing` + `ModelsDevCatalog` (Anthropic models, same catalog)

### Fetch strategy priority in Phase 2

```
auto mode: [OpenClawLocalFetchStrategy, OpenClawAdminAPIFetchStrategy]
api mode:  [OpenClawAdminAPIFetchStrategy]
```

---

## Open Questions

1. **Key type available:** Is the Anthropic key a standard `sk-ant-api...` or admin
   `sk-ant-admin...`? Admin is required for the cost/usage report endpoints.
2. **Shared account:** OpenClaw and Claude Code bill the same Anthropic org.
   Phase 1 shows combined spend — is that acceptable until Phase 2 ships?
3. **OpenClaw upstream:** Can the usage JSONL emitter be added to OpenClaw core?
   It's ~20 lines in the Anthropic client response handler.
4. **Icon:** Lobster emoji asset, or custom SVG? steipete uses high-quality
   provider icons — worth a custom asset vs. emoji fallback.
5. **Settings pane:** Does OpenClaw expose its Anthropic key in a keychain entry
   CodexBar could read, or does the user need to paste it manually?

---

## Testing

```bash
# Build and run after changes
cd ~/code/CodexBar && ./Scripts/compile_and_run.sh

# Verify admin API endpoints manually
curl -s "https://api.anthropic.com/v1/organizations/cost_report?group_by[]=description&bucket_width=1d&limit=7" \
  -H "x-api-key: $OPENCLAW_ANTHROPIC_ADMIN_KEY" \
  -H "anthropic-version: 2023-06-01" | python3 -m json.tool
```

---

## Related Files

- `docs/openclaw-usage-tracking-spec.md` — deep architectural spec (CodexBar internals)
- `Sources/CodexBarCore/Providers/Claude/ClaudeAdminAPIUsageFetcher.swift` — reuse this
- `Sources/CodexBarCore/Vendored/CostUsage/CostUsageScanner+Claude.swift` — Phase 2 model
- `Sources/CodexBarCore/Providers/ProviderFetchPlan.swift` — fetch strategy protocol
