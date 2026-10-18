# Realtime Events & Wallet Enhancements Implementation Roadmap

This document provides implementation guidance for three critical OracleDesk Frontend enhancement issues focused on realtime event handling, wallet interaction reliability, and React Query cache reconciliation.

---

## Issue #38: Recover Missed Realtime Events After Disconnects Using RPC getEvents Cursors

### Problem
Socket.io connections drop during laptop sleep, network changes, or backend restarts. Events emitted during disconnection gaps are permanently lost, causing silent feed gaps. Users see incomplete trading activity with no indication of missing data.

### Solution Overview
Implement frontend-only event recovery by:
1. Querying Stellar RPC `getEvents` for missed contract events during disconnection gaps
2. Decoding events using contract specs the app already ships
3. Merging recovered events with socket events without duplicates
4. Surfacing "gap too old" notices when events exceed RPC retention window

### Key Components

**lib/stellar/events.ts** (NEW)
```typescript
export type ChainEventName = "Trade" | "TracePublished";

export interface DecodedChainEvent {
  id: string;
  ledger: number;
  ledgerClosedAt: string;
  txHash: string;
  contractId: string;
  name: ChainEventName;
  params: Record<string, unknown>;
}

export function decodeEvent(raw: rpc.Api.EventResponse): DecodedChainEvent | null;

export async function fetchEvents(opts: {
  startLedger?: number;
  cursor?: string;
  names: ChainEventName[];
  limit?: number;
  signal?: AbortSignal;
  server?: Pick<rpc.Server, "getEvents" | "getHealth">;
}): Promise<{ events: DecodedChainEvent[]; cursor: string; latestLedger: number }>;

export function planBackfill(
  hw: { ledger: number } | null,
  health: { oldestLedger: number; latestLedger: number }
): { kind: "none" } | { kind: "range"; startLedger: number } | { kind: "too-old"; missingFrom: number; missingTo: number; startLedger: number };
```

**Event Decoding**
- Use `marketCore().spec` and `reasoningRegistry().spec` for decoding
- Call `spec.parseEvent(raw.topic, raw.value)` (same as backend indexer)
- Return `null` for unknown contracts/events

**Normalization Functions**
- `toTradeExecuted(e)`: Converts chain events to socket payload format
- `toReasoningPublished(e)`: Converts trace events to socket payload format
- Compute `priceYesBps` from reserves using existing `lib/stellar/generated/fpmm.ts`

**Gap Recovery Flow**
1. Track high-water mark: `{ ledger, eventId }` per session
2. On reconnect or `visibilitychange` to visible (after >30s hidden):
   - Call `getHealth()` to get ledger bounds
   - Run `planBackfill()` to determine recovery strategy
   - Loop `fetchEvents()` with cursor pagination
   - Abort on unmount or new recovery start
3. Deliver recovered events flagged `origin: "rpc"` in ascending ledger order
4. Deduplicate using LRU cache (2,000 keys):
   - `TRADE_EXECUTED`: key by `eventId`
   - `REASONING_PUBLISHED`: key by `onChainTraceId`

**Configuration**
- `NEXT_PUBLIC_EVENTS_MAX_BACKFILL_PAGES` (default: 10)
- Add to `.env.example` with documentation

**Gap Notices**
- `useRealtimeGap()`: Returns gap info for UI rendering
- Display: "Some activity between ledgers N and M is older than the RPC keeps and can't be shown."
- Dismissible by user

### Acceptance Criteria
- [x] Decode `Trade` and `TracePublished` via binding specs (no hand-written XDR)
- [x] After simulated disconnect, trades appear in feeds after reconnect exactly once
- [x] Recovered events delivered in `(ledger, id)` order with `origin: "rpc"` flag
- [x] Visible notice for gaps older than `getHealth().oldestLedger`
- [x] Backfill stops at `EVENTS_MAX_BACKFILL_PAGES` with truncation report
- [x] Recovery cancels on unmount and newer recovery starts
- [x] `REASONING_PUBLISHED` duplicates suppressed by `onChainTraceId`
- [x] Documentation in `.env.example`, `README.md`, `docs/architecture.md`

---

## Issue #35: Add Cancellation and Timeouts to Wallet Signing Prompts

### Problem
Wallet promises from `kit.authModal()` and `kit.signTransaction()` can hang indefinitely when:
- User closes wallet popup with window close button (not "Reject")
- User switches away from WalletConnect prompt
- Hardware wallet unplugged mid-signature

UI gets stuck with perpetual "Connecting…", "Buying…", "Confirm the 0.50 USDC transfer…" states until page reload.

### Solution Overview
Wrap all wallet kit calls in abortable helper with configurable timeouts, add Cancel action to every waiting state, and ensure late signatures after cancellation are discarded.

### Key Components

**lib/stellar/wallet-call.ts** (NEW)
```typescript
export class WalletTimeoutError extends Error {
  constructor(public label: string, public ms: number);
}

export class WalletCancelledError extends Error {
  constructor(public label: string);
}

export interface WalletCallOptions {
  signal?: AbortSignal;
  timeoutMs: number;
  label: string;
}

export function withWalletDeadline<T>(
  start: () => Promise<T>,
  opts: WalletCallOptions
): Promise<T>;
```

**Implementation Details**
- Race wallet promise against timeout and abort signal
- Late signatures after cancel/timeout are dropped (never reach `sendTransaction`)
- Defense in depth: Split `tx.signAndSend()` into `tx.sign()` + check abort + `tx.send()`

**Timeout Configuration**
- `NEXT_PUBLIC_WALLET_TIMEOUT_MS`: 120000 (2 minutes) for extensions
- `NEXT_PUBLIC_REMOTE_WALLET_TIMEOUT_MS`: 600000 (10 minutes) for hardware/WalletConnect
- Detect wallet type via `kit.selectedModule.moduleType`

**Provider Integration (lib/stellar/WalletProvider.tsx)**
Wrap all kit calls:
- `kit.authModal()` → label "connect"
- `kit.getNetwork()` → label "network", 10s timeout
- Challenge signing in `signIn()` → label "login"
- Maintain `AbortController` ref for current operation
- Expose `cancelPending(): void` on context

**Error Mapping (lib/stellar/errors.ts)**
New `DescribedError.kind` values:
- `"wallet-timeout"`: "Your wallet didn't respond. Nothing was sent. Check the wallet window and try again."
- `"cancelled"`: "Cancelled. Nothing was sent."

**UI Components**
New `components/wallet/SigningPrompt.tsx`:
- "Waiting for your wallet…" message
- Countdown display when >50% timeout remaining
- Cancel button wired to `AbortController`

Usage locations:
- `components/layout/WalletDropdown.tsx` (connecting)
- `app/markets/quick-view/page.tsx` (buying)
- `app/copy-trade/page.tsx` (submitting)
- `app/premium/page.tsx` (payment)

**Premium Payment Special Case**
On timeout during premium payment, add warning:
> "If your wallet shows the payment as sent, it will appear in your wallet history; don't pay again."

### Acceptance Criteria
- [x] `withWalletDeadline` wraps all `authModal`, `getNetwork`, `signTransaction` calls
- [x] All waiting states have working Cancel button
- [x] No waiting state can stay pending past configured timeout
- [x] Late signatures after cancel/timeout never submitted (test proven)
- [x] `cancelled` and `wallet-timeout` returned ahead of network errors
- [x] Timeouts configurable via env vars, documented in `.env.example`
- [x] Cancelled outcomes shown neutrally (not as errors)
- [x] Premium payment timeout warns against double payment
- [x] `docs/manual-test.md` includes "Close wallet popup" test step

---

## Issue #37: Reconcile React Query Caches from Realtime Trade Events

### Problem
`TRADE_EXECUTED` socket events display in text feeds but don't update the numbers traders see. `MarketCard`, quick-view modal, and wallet dropdown show values from 30s-old cache while fresh trades scroll past, causing traders to buy against stale probabilities.

### Solution Overview
Wire realtime trade events into React Query cache with ledger-ordered patches marked as provisional, followed by confirming chain reads. Backend-relayed messages never override contract state.

### Key Components

**Ledger-Stamped Reads (lib/hooks/useStellar.ts)**
```typescript
export interface ChainStamp {
  ledger: number;
  provisional?: {
    source: "socket";
    eventId: string;
    ledger: number;
  };
}

export function simulatedWithLedger<T>(
  tx: AssembledTransaction<unknown>,
  contract?: ContractName
): { value: T; ledger: number };
```

Extend `OnChainMarketView`:
```typescript
interface OnChainMarketView {
  id: string;
  market: Market;
  yesBps: number;
  ledger: number;
  provisional?: ChainStamp["provisional"];
}
```

**Pure Reconciler (lib/realtime/reconcile.ts)** (NEW)
```typescript
export interface TradeHint {
  eventId: string;
  onChainMarketId: string;
  trader: string;
  priceYesBps: number;
  ledger: number;
}

export function patchMarketView(
  view: OnChainMarketView,
  hint: TradeHint
): OnChainMarketView | null;

export function applyTradeHint(
  qc: QueryClient,
  hint: TradeHint,
  opts: {
    connected: string | null;
    confirm: (marketId: string) => void;
  }
): void;
```

**Reconciliation Logic**
- `patchMarketView()`: Returns `null` if `hint.ledger <= view.ledger` or `view.provisional?.ledger >= hint.ledger`
- Otherwise returns copy with `yesBps = hint.priceYesBps` and `provisional` set
- Leaves `market` (reserves, `sets_minted`) untouched
- UI must not recompute probability from stale reserves when `provisional` is set

**Cache Updates**
- Update `stellarKeys.market(id)` via `qc.setQueryData`
- Update all `["stellar", "markets"]` lists via `qc.setQueriesData`
- If `BigInt(onChainMarketId) >= cached marketCount`, invalidate `stellarKeys.marketCount`

**Connected Trader Handling**
If `hint.trader === connected`:
- Invalidate `stellarKeys.position(id, connected)`
- Invalidate `stellarKeys.balance(connected)`
- Do not patch them (issue #33 owns provisional position/balance)

**Confirming Reads**
- `createConfirmScheduler(qc, delayMs = 1500)`: Per-market timer map
- First hint starts timer, subsequent hints absorbed within window
- On fire: invalidate and refetch that market
- Removes provisional flag when confirmed read arrives
- Coalesces bursts to prevent simulation storm

**Position & Balance Stamps**
- `usePosition` and `useUsdcBalance` store `{ value, ledger }` internally
- Use React Query `select` to expose unwrapped value
- Component call sites unchanged (still receive `Position` / `bigint`)

**UI Indicators**
- `SourceTag` component shows provisional state
- Tooltip: "Updated from realtime event (ledger N). Confirming…"
- Switches to "Confirmed from chain" when confirmation arrives

### Acceptance Criteria
- [x] All chain reads return ledger-stamped values
- [x] `OnChainMarketView` includes `ledger` and `provisional` metadata
- [x] Trade events patch `yesBps` only when event ledger > cached ledger
- [x] Patched values marked provisional with visible UI indicator
- [x] Confirming read scheduled per market with 1.5s coalescing window
- [x] Connected trader's position/balance invalidated (not patched)
- [x] Out-of-order and duplicate events ignored deterministically
- [x] Confirming read clears provisional flag
- [x] Pure reconciler module testable without socket dependency
- [x] Tests verify ledger ordering, deduplication, connected trader invalidation

---

## Implementation Priority

**Phase 1: Foundation (Week 1)**
1. Issue #35 (Wallet timeouts) - Critical UX blocker, foundational
2. Create `lib/stellar/wallet-call.ts`
3. Update error handling in `lib/stellar/errors.ts`

**Phase 2: Realtime Infrastructure (Week 2)**
1. Issue #38 (Event recovery) - Builds on #30 (RealtimeProvider)
2. Create `lib/stellar/events.ts`
3. Implement gap detection and backfill

**Phase 3: Cache Reconciliation (Week 3)**
1. Issue #37 (Query cache) - Depends on #38 for event types
2. Add ledger stamps to all chain reads
3. Create `lib/realtime/reconcile.ts`
4. Wire up provisional updates and confirmations

## Testing Strategy

### Unit Tests
- `lib/stellar/__tests__/wallet-signing.test.ts`: Timeout/cancel behavior, late signature discard
- `lib/stellar/__tests__/events.test.ts`: Event decoding, normalization, backfill planning
- `lib/realtime/__tests__/recovery.test.ts`: Deduplication, ordering, gap notices
- `lib/realtime/__tests__/reconcile.test.ts`: Ledger ordering, provisional marking, invalidation

### Integration Tests
- Fake RPC server for event fetching
- Fake socket emitter for realtime tests
- Mock QueryClient for cache reconciliation

### Manual Testing
- Simulate disconnect during trade (devtools offline mode)
- Close wallet popup mid-signature (Freighter, xBull)
- Switch away from WalletConnect prompt
- Verify duplicate suppression
- Verify provisional → confirmed transitions

### E2E Tests (Optional)
- `e2e/testnet.e2e.test.ts`: Fetch last 100 ledgers, verify all decode
- End-to-end wallet cancellation flow
- Realtime event delivery and cache update

---

## Configuration Reference

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `NEXT_PUBLIC_EVENTS_MAX_BACKFILL_PAGES` | `10` | Maximum pages to fetch during event recovery |
| `NEXT_PUBLIC_WALLET_TIMEOUT_MS` | `120000` | Timeout for extension wallets (2 min) |
| `NEXT_PUBLIC_REMOTE_WALLET_TIMEOUT_MS` | `600000` | Timeout for hardware/WalletConnect (10 min) |

Add to `.env.example` with inline comments.

---

## Monitoring & Observability

### Event Recovery Metrics
- Gap detection count
- Backfill page count per recovery
- Truncation events
- RPC retention violations

### Wallet Timeout Metrics
- Timeout count by wallet type
- Cancellation count by operation (connect, login, buy, transfer)
- Late signature arrival count (should be 0 in production)

### Cache Reconciliation Metrics
- Provisional update count per market
- Confirmation delay (event arrival → confirmed read)
- Out-of-order event count
- Duplicate suppression count

---

## Rollback Procedures

### Issue #35 Rollback
- Revert `lib/stellar/wallet-call.ts`
- Remove timeout wrappers from `WalletProvider.tsx`
- Remove Cancel buttons from UI
- UI returns to indefinite pending states (original behavior)

### Issue #38 Rollback
- Remove `lib/stellar/events.ts`
- Remove recovery logic from `RealtimeProvider.tsx`
- Feeds remain incomplete during disconnects (original behavior)

### Issue #37 Rollback
- Remove ledger stamps from chain reads
- Remove `lib/realtime/reconcile.ts`
- Remove provisional UI indicators
- Caches update only on refetch interval (original behavior)

---

## References

- [Stellar RPC getEvents](https://developers.stellar.org/docs/data/apis/rpc/api-reference/methods/getEvents)
- [Stellar RPC getHealth](https://developers.stellar.org/docs/data/apis/rpc/api-reference/methods/getHealth)
- [socket.io Client Reconnection](https://socket.io/docs/v4/client-socket-instance/)
- [AbortController MDN](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
- [Stellar Wallets Kit](https://github.com/Creit-Tech/Stellar-Wallets-Kit)
- [Soroban Contract Events](https://developers.stellar.org/docs/learn/encyclopedia/contract-development/events)

---

For detailed implementation of each issue, refer to the original issue descriptions in the repository.
