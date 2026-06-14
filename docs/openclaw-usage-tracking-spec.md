# Feature Spec: OpenClaw Usage Tracking

> Status: Proposed
> Author: CodexBar maintainers
> Target: New `UsageProvider` + fetch strategy for OpenClaw
> Audience: Developers implementing the provider; reviewers

## 1. Problem Statement

[OpenClaw](https://openclaw.ai) is a Node.js-based AI assistant that calls
`api.anthropic.com` **directly** using its own Anthropic API key. It does **not**
run through the Claude Code CLI and does **not** write to Claude Code's local
logs (`~/.config/claude/projects/`, `~/.claude/projects/`, or anything under
`~/.claude/`).

As a result, CodexBar has **zero visibility** into OpenClaw's API spend today.
Users who run both Claude Code and OpenClaw see only the Claude Code half of
their Anthropic bill in the menubar. The two streams share the same underlying
Anthropic billing account, but CodexBar can only surface the part that flows
through Claude Code's JSONL logs.

**Goal:** Let users see their OpenClaw API spend (cost + token usage) alongside
their Claude Code spend in the same CodexBar UI, as a first-class provider.

### Non-goals

- Tracking OpenClaw's non-AI activity (WhatsApp automation, cron, gateway logs).
  We only care about Anthropic API consumption attributable to OpenClaw.
- Real-time per-request streaming. Daily/periodic aggregation matches every
  other provider and the granularity of Anthropic's reporting APIs.
- Rate-limit/quota windows. OpenClaw uses pay-as-you-go API keys, so there is no
  session/weekly quota window to display (unlike Claude Code subscriptions).

## 2. How CodexBar Currently Tracks Claude Usage

CodexBar is a Swift Package Manager project. The relevant layers:

### 2.1 Provider abstraction

Every provider is a `ProviderDescriptor`
(`Sources/CodexBarCore/Providers/ProviderDescriptor.swift`) registered in a
static registry via the `@ProviderDescriptorRegistration` /
`@ProviderDescriptorDefinition` macros. A descriptor bundles:

- `id: UsageProvider` — an enum case in
  `Sources/CodexBarCore/Providers/Providers.swift` (currently ~50 cases:
  `codex`, `claude`, `cursor`, …).
- `metadata: ProviderMetadata` — display name, labels, toggle title, dashboard
  URLs, `defaultEnabled`, etc.
- `branding: ProviderBranding` — icon resource + color.
- `tokenCost: ProviderTokenCostConfig` — whether token-cost tracking is
  supported and the "no data" message.
- `fetchPlan: ProviderFetchPlan` — the set of allowed `ProviderSourceMode`s
  (`auto`/`api`/`web`/`cli`/`oauth`) plus a `ProviderFetchPipeline`.
- `cli: ProviderCLIConfig` — CLI name + version detector.

### 2.2 Fetch strategies

Data acquisition is a pipeline of `ProviderFetchStrategy` values
(`Sources/CodexBarCore/Providers/ProviderFetchPlan.swift:166`):

```swift
public protocol ProviderFetchStrategy: Sendable {
    var id: String { get }
    var kind: ProviderFetchKind { get }          // .cli/.web/.oauth/.apiToken/.localProbe/.webDashboard
    func isAvailable(_ context: ProviderFetchContext) async -> Bool
    func fetch(_ context: ProviderFetchContext) async throws -> ProviderFetchResult
    func shouldFallback(on error: Error, context: ProviderFetchContext) -> Bool
}
```

`ProviderFetchPipeline.fetch(...)` (`ProviderFetchPlan.swift:198`) walks the
resolved strategies in order: for each, it checks `isAvailable`, calls `fetch`,
and on error consults `shouldFallback` to decide whether to try the next one.
The first success wins; the result is a `ProviderFetchResult` carrying a
`UsageSnapshot` (and optional `CreditsSnapshot` / dashboard).

Claude resolves up to four strategies
(`Sources/CodexBarCore/Providers/Claude/ClaudeProviderDescriptor.swift:46`):

1. **Admin API** (`ClaudeAdminAPIFetchStrategy`, `kind = .apiToken`) — used when
   `sourceMode == .api` or an Anthropic Admin API key is present.
2. **OAuth** (`ClaudeOAuthFetchStrategy`) — keychain credentials.
3. **Web** (`ClaudeWebFetchStrategy`) — browser cookie scraping of claude.ai.
4. **CLI** (`ClaudeCLIFetchStrategy`) — invokes the `claude` binary.

### 2.3 The two data shapes

CodexBar tracks two distinct things:

1. **Quota/rate usage** (`UsageSnapshot` → `RateWindow`s). For subscriptions:
   session %, weekly %, Opus %, reset times. This is the menubar's primary
   display for Claude Code subscriptions.
2. **Token cost** (`ProviderCostSnapshot` / `CostUsageTokenSnapshot`). Per-day,
   per-model input/output/cache tokens and a USD cost. This is what we care
   about for a pay-as-you-go API tool like OpenClaw.

There are **two independent sources** for token cost:

**(a) Local JSONL scanning** —
`Sources/CodexBarCore/Vendored/CostUsage/CostUsageScanner+Claude.swift`.
`defaultClaudeProjectsRoots` (line 27) resolves roots from `CLAUDE_CONFIG_DIR`
or falls back to `~/.config/claude/projects` and `~/.claude/projects`.
`parseClaudeFileCancellable` (line 74) streams each `.jsonl` line, keeps only
`"type":"assistant"` lines containing `"usage"`, and extracts
`input_tokens`, `cache_read_input_tokens`, `cache_creation_input_tokens`,
`cache_creation_input_tokens_1h`, and `output_tokens`, plus `timestamp`,
`model`, `sessionId`, `messageId`, `requestId`. Rows are deduped by
`messageId:requestId`, aggregated by day+model, priced via `CostUsagePricing` /
`ModelsDevCatalog`, and cached by file mtime/size for incremental rescans.

**(b) Anthropic Admin/Usage API** —
`Sources/CodexBarCore/Providers/Claude/ClaudeAdminAPIUsageFetcher.swift`.
This is the most directly reusable prior art for OpenClaw. It hits two
organization-level endpoints with `x-api-key` + `anthropic-version: 2023-06-01`:

- `GET https://api.anthropic.com/v1/organizations/cost_report`
  (`group_by[]=description`) — daily USD cost buckets. `amount` is a decimal
  string in **lowest USD units** (cents); the fetcher divides by 100
  (`usdFromAnthropicLowestUnitAmount`, line 211).
- `GET https://api.anthropic.com/v1/organizations/usage_report/messages`
  (`group_by[]=model`) — daily token buckets:
  `uncached_input_tokens`, `cache_creation.{ephemeral_5m,ephemeral_1h}_input_tokens`,
  `cache_read_input_tokens`, `output_tokens`, `model`.

Both are queried with `bucket_width=1d`, `limit=31`, over a trailing ~31-day
window (`dailyRange`, line 226), then merged per day into a
`ClaudeAdminAPIUsageSnapshot` (`daily: [DailyBucket]`, `updatedAt`). That
snapshot exposes `last7Days` / `last30Days` / `latestDay` summaries and
`topModels`, and converts to a `UsageSnapshot` via `toUsageSnapshot()` for the
fetch result. HTTP goes through the shared `ProviderHTTPClient` transport (so it
is mockable in tests via `ProviderHTTPTransport`).

### 2.4 Flow to the menubar

`ProviderDescriptor.fetch()` → `ProviderFetchPipeline` → `UsageSnapshot`, stored
in `UsageStore` (`Sources/CodexBarCore/.../UsageStore.swift`) keyed by
`UsageProvider`. The status item, menu, and widget observe the store and render
per-provider rows, cost, and token breakdowns.

### 2.5 What adding a provider entails (the established pattern)

1. Add a `UsageProvider` enum case.
2. Add an `IconStyle` case + icon asset (`ProviderIcon-<id>`).
3. Create a `<Name>ProviderDescriptor` enum with the two registration macros and
   a `makeDescriptor()`.
4. Implement one or more `ProviderFetchStrategy` structs.
5. Implement a provider-specific snapshot model with a `toUsageSnapshot()` (and,
   for cost, populate a `ProviderCostSnapshot`).
6. Wire any settings/credentials (env var, keychain, or settings pane).
7. Add a `docs/<name>.md` page and tests.

## 3. Proposed Solution Options

The core challenge is **attribution**: OpenClaw and Claude Code can bill to the
same Anthropic account, so we must be able to isolate OpenClaw's slice. Three
options, with trade-offs.

### Option A — Poll the Anthropic Usage & Cost API (org-level)

Reuse `ClaudeAdminAPIUsageFetcher` against the same `cost_report` /
`usage_report/messages` endpoints, authenticated with the user's OpenClaw
Anthropic key.

- **Pros:** Maximal code reuse; authoritative cost numbers straight from
  Anthropic; no dependency on OpenClaw writing anything; works retroactively.
- **Cons / blocker:** These endpoints require an **Admin API key**
  (`sk-ant-admin...`) and report **org-wide** usage. If OpenClaw uses a plain
  workspace/standard key, it can't call them. Even with an admin key, the
  numbers are **not attributable to OpenClaw specifically** unless OpenClaw's
  traffic is isolated to its own **Anthropic Workspace** or **API key**, and we
  filter by it. The Usage API does support `group_by` and filtering by
  `api_key_id` / `workspace_id`; if OpenClaw runs under a dedicated workspace or
  key, we can scope the query to it. Otherwise Option A double-counts Claude
  Code's API usage.

### Option B — Read a local usage log that OpenClaw writes

Define a small append-only JSONL contract. OpenClaw (which we control / can file
an upstream request against) writes one line per Anthropic API response into a
known path under its state dir. CodexBar scans it exactly like it scans Claude
Code JSONL.

- **Pros:** Perfectly attributed (only OpenClaw's calls are logged); no API key
  handling in CodexBar; offline; mirrors the proven `CostUsageScanner+Claude`
  incremental-scan design; works with any Anthropic key type.
- **Cons:** Requires an OpenClaw change to emit the log; only captures usage from
  the moment logging is enabled forward; pricing is computed client-side by
  CodexBar (via `CostUsagePricing`/`ModelsDevCatalog`) rather than billed by
  Anthropic, so it's an estimate, not the invoice.

OpenClaw already keeps a state directory at `~/.openclaw/` (with
`~/.openclaw-<profile>` / `~/.openclaw-dev` variants, overridable via
`OPENCLAW_STATE_DIR`) and already writes JSONL audit logs there
(e.g. `~/.openclaw/logs/config-audit.jsonl`). A usage log fits this convention
naturally.

### Option C — Hybrid: local log for attribution, API for true cost

Scan the local log (Option B) for token counts and per-call attribution, and
optionally cross-check / reprice against the Usage API (Option A) when an admin
key scoped to OpenClaw's workspace is available.

- **Pros:** Best of both — accurate attribution + authoritative cost when
  possible.
- **Cons:** Most code; two sources to reconcile; only worth it once Option B
  exists.

### Recommendation

**Ship Option B first** (local JSONL log) as the default, with the fetch
strategy structured so **Option A can be added as a fallback/secondary strategy
later** without reworking the descriptor. This matches CodexBar's existing
multi-strategy pattern (`auto` resolves a prioritized list) and avoids the
attribution blocker that makes Option A unusable for the common case of a single
shared Anthropic account.

## 4. Recommended Approach — Implementation Details

### 4.1 The log contract (OpenClaw side)

OpenClaw appends one JSON object per line to:

```
$OPENCLAW_STATE_DIR/usage/anthropic-usage.jsonl
# default: ~/.openclaw/usage/anthropic-usage.jsonl
# dev profile: ~/.openclaw-dev/usage/anthropic-usage.jsonl
```

Each line is written when an Anthropic API response is received, using fields
copied directly from the response's `usage` block (so CodexBar needs no
guesswork):

```json
{
  "v": 1,
  "ts": "2026-06-14T09:31:02.184Z",
  "request_id": "req_011...",
  "model": "claude-opus-4-8",
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

Contract rules:

- **Append-only, never rewritten** — lets CodexBar scan incrementally by byte
  offset (mtime/size cache), exactly like the Claude scanner.
- **One object per line, newline-terminated**; partial trailing lines tolerated.
- `v` is a schema version for forward compatibility.
- `request_id` is the dedup key (CodexBar dedupes on it).
- `usage` field names mirror Anthropic's response so pricing is unambiguous.
- Token-count provenance is the API response, not estimates.
- If OpenClaw cannot emit this upstream in time, a thin wrapper/middleware around
  its Anthropic client (an `afterResponse` hook) can write the line — document
  this in `docs/openclaw.md`.

### 4.2 CodexBar side

**(1) Enum + branding.**
- `Sources/CodexBarCore/Providers/Providers.swift`: add `case openclaw` to
  `UsageProvider` and `case openclaw` to `IconStyle`.
- Add icon asset `ProviderIcon-openclaw` and a color in `ProviderBranding`.

**(2) Local scanner.**
Add `Sources/CodexBarCore/Vendored/CostUsage/CostUsageScanner+OpenClaw.swift`
modeled on `CostUsageScanner+Claude.swift`:

- `defaultOpenClawUsageRoots(options:)` — resolve from `OPENCLAW_STATE_DIR` (and
  honor `OPENCLAW_PROFILE`/`--profile` → `~/.openclaw-<name>`), else
  `~/.openclaw`; append `usage/anthropic-usage.jsonl`. Allow an override for
  tests.
- `parseOpenClawFileCancellable(...)` — reuse `CostUsageJsonl.scan` with the same
  `maxLineBytes`/offset/cancellation machinery. Parse each line, dedupe by
  `request_id`, aggregate by day+model into the same packed token array the
  Claude path uses, and price via `CostUsagePricing.normalizeClaudeModel` +
  `ModelsDevCatalog` (OpenClaw uses Anthropic models, so the existing pricing
  catalog applies directly).
- Produce a `CostUsageTokenSnapshot` / `CostUsageDailyReport` identical in shape
  to Claude's, so all downstream UI works unchanged.

**(3) Fetch strategy.**
Add `Sources/CodexBarCore/Providers/OpenClaw/OpenClawLocalFetchStrategy.swift`:

```swift
struct OpenClawLocalFetchStrategy: ProviderFetchStrategy {
    let id = "openclaw.local"
    let kind: ProviderFetchKind = .localProbe

    func isAvailable(_ ctx: ProviderFetchContext) async -> Bool {
        OpenClawUsageLocator.usageLogExists(environment: ctx.env)
    }

    func fetch(_ ctx: ProviderFetchContext) async throws -> ProviderFetchResult {
        let snapshot = try OpenClawCostUsageFetcher.loadTokenSnapshot(
            historyDays: ctx.costUsageHistoryDays,
            environment: ctx.env)
        return makeResult(usage: snapshot.toUsageSnapshot(), sourceLabel: "local-log")
    }

    func shouldFallback(on _: Error, context ctx: ProviderFetchContext) -> Bool {
        // Once Option A exists, fall back to admin-API in auto mode.
        ctx.sourceMode == .auto
    }
}
```

`kind = .localProbe` matches the "read a local file" intent (vs `.apiToken`).

**(4) Descriptor.**
Add `Sources/CodexBarCore/Providers/OpenClaw/OpenClawProviderDescriptor.swift`:

```swift
@ProviderDescriptorRegistration
@ProviderDescriptorDefinition
public enum OpenClawProviderDescriptor {
    static func makeDescriptor() -> ProviderDescriptor {
        ProviderDescriptor(
            id: .openclaw,
            metadata: ProviderMetadata(
                id: .openclaw,
                displayName: "OpenClaw",
                sessionLabel: "Today",
                weeklyLabel: "30 days",
                supportsOpus: false,
                supportsCredits: false,
                toggleTitle: "Show OpenClaw API usage",
                cliName: "openclaw",
                defaultEnabled: false,
                isPrimaryProvider: false,
                dashboardURL: "https://console.anthropic.com/settings/usage",
                changelogURL: "https://github.com/openclaw/openclaw/releases",
                statusPageURL: "https://status.claude.com/"),
            branding: ProviderBranding(
                iconStyle: .openclaw,
                iconResourceName: "ProviderIcon-openclaw",
                color: ProviderColor(red: 0.85, green: 0.30, blue: 0.20)),
            tokenCost: ProviderTokenCostConfig(
                supportsTokenCost: true,
                noDataMessage: {
                    "No OpenClaw usage found at ~/.openclaw/usage/anthropic-usage.jsonl."
                }),
            fetchPlan: ProviderFetchPlan(
                sourceModes: [.auto, .api],
                pipeline: ProviderFetchPipeline(resolveStrategies: resolveStrategies)),
            cli: ProviderCLIConfig(name: "openclaw", versionDetector: { _ in nil }))
    }

    private static func resolveStrategies(
        context: ProviderFetchContext) async -> [any ProviderFetchStrategy]
    {
        // Phase 1: local log only.
        // Phase 2: if sourceMode == .api && admin key present → [OpenClawAdminAPIFetchStrategy()]
        //          else [OpenClawLocalFetchStrategy(), OpenClawAdminAPIFetchStrategy()]
        [OpenClawLocalFetchStrategy()]
    }
}
```

**(5) Snapshot model.** Since OpenClaw has no rate windows, `toUsageSnapshot()`
returns a `UsageSnapshot` with `primary/secondary/tertiary == nil` and a
populated `providerCost` (`ProviderCostSnapshot`) — i.e. it is a **cost-only**
provider. Confirm the UI renders cost-only providers gracefully (Claude in
`.api` mode is already cost-only, so this path exists).

### 4.3 Optional Phase 2 — Anthropic Usage API fallback

Add `OpenClawAdminAPIFetchStrategy` (`kind = .apiToken`) that reuses
`ClaudeAdminAPIUsageFetcher` but:
- reads the key from `OPENCLAW_ANTHROPIC_API_KEY` (or the OpenClaw credentials
  store under `~/.openclaw/credentials/`) rather than `ANTHROPIC_API_KEY`;
- when the key is an **admin** key and OpenClaw runs under a dedicated workspace,
  adds `group_by[]=workspace_id` / a workspace filter so the result is scoped to
  OpenClaw and does not double-count Claude Code.

Generalize `ClaudeAdminAPIUsageFetcher` to accept the key + optional
workspace/api-key filter as parameters so it can be shared verbatim.

### 4.4 Testing

- Unit: feed fixture `anthropic-usage.jsonl` files to the scanner; assert daily
  + per-model aggregation, dedup by `request_id`, incremental offset rescans,
  and pricing.
- Strategy: `isAvailable` true/false on presence of the log; `fetch` maps to a
  cost-only `UsageSnapshot`.
- Phase 2: mock `ProviderHTTPTransport` with canned cost/usage JSON (mirror the
  existing Claude admin-API tests).
- Linux test target: keep the scanner free of AppKit so it runs in `TestsLinux`.

## 5. UI / UX Considerations

- **Menubar row.** OpenClaw appears as its own toggleable provider row (off by
  default, like Claude). Because it's cost-only, the row shows **today's cost**
  and **30-day cost** with a token breakdown on expand — not a quota bar. Reuse
  the cost layout Claude already uses in `.api` mode.
- **Icon.** Distinct `ProviderIcon-openclaw` (lobster 🦞 motif fits OpenClaw's
  branding) so it isn't confused with the Claude row, even though both bill
  Anthropic.
- **Combined Anthropic view.** Optional nicety: a tooltip or subtitle noting
  "OpenClaw + Claude Code both bill your Anthropic account" to set expectations.
  Do **not** auto-sum them into one number — keep providers independent, which is
  the established model.
- **Empty state.** When no log exists, surface the `noDataMessage` with the exact
  path and a one-line "enable usage logging in OpenClaw" hint linking to
  `docs/openclaw.md`.
- **Settings pane.** A provider settings entry to (a) override the log path and
  (b) in Phase 2, paste/locate the OpenClaw Anthropic key for the API fallback.
- **Cost-estimate disclaimer.** Since Phase-1 cost is computed from token counts
  via `CostUsagePricing`, label it "estimated" where the UI distinguishes billed
  vs. estimated cost, to avoid implying it's the Anthropic invoice.
- **Refresh cadence.** Driven by the existing background refresh timer; local
  scan is cheap (incremental, offset-cached), so no special handling needed.

## 6. Open Questions

1. **Attribution model.** Will OpenClaw users typically use a dedicated Anthropic
   workspace/API key, or share one with Claude Code? This determines whether the
   Phase-2 API path is viable or whether the local log is the only correct
   source.
2. **Upstream willingness.** Can OpenClaw add the usage-log emitter upstream, and
   on what timeline? If not, do we ship a documented client wrapper/middleware
   instead?
3. **Log location & profiles.** Confirm the canonical path and how to enumerate
   profiles (`~/.openclaw-<name>`) — should CodexBar scan all profiles or just
   the default/`OPENCLAW_STATE_DIR`?
4. **Schema authority.** Should OpenClaw log the *raw* Anthropic `usage` block
   verbatim (future-proof against new token fields), and should it also log the
   billed cost if Anthropic ever returns it inline?
5. **Credential storage (Phase 2).** Where does OpenClaw keep its Anthropic key
   (`~/.openclaw/credentials/`, env, keychain)? Does CodexBar read it or require
   the user to paste it into settings?
6. **Model naming.** Does OpenClaw ever use non-Anthropic models or proxied model
   names that `CostUsagePricing.normalizeClaudeModel` wouldn't recognize? If so,
   the pricing catalog may need OpenClaw-specific aliases.
7. **Log rotation.** If OpenClaw rotates/truncates the usage log, the offset
   cache must detect shrink and rescan from 0 — confirm the Claude scanner's
   shrink-detection covers this (it keys on size+mtime).
8. **Naming collision.** The `openclaw` CLI in some environments is unrelated
   tooling. Confirm `cliName`/version detection won't misfire; consider not
   shipping a version detector (return `nil`) in Phase 1.
