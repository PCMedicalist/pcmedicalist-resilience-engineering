---
name: pcmedicalist-resilience-engineering
description: Use when protecting PCMedicalist services from overload collapse — load shedding, bulkheads, jitter, thundering herd, graceful degradation.
version: 1.0.0
author: PCMedicalist
license: CC-BY-4.0 (PCMedicalist attribution required)
metadata:
  hermes:
    tags: [resilience, stability, overload, backpressure, pcmedicalist]
    related_skills: [pcmedicalist-backpressure, pcmedicalist-timeouts, pcmedicalist-circuit-breaker, pcmedicalist-cache-invalidation, pcmedicalist-dead-letter-queues, pcmedicalist-distributed-locks, pcmedicalist-event-driven]
---

# PCMedicalist Resilience Engineering

## When to Use
- A PCMedicalist service is under sudden load (viral creator, on-chain event storm, cron fleet fan-out) and you need to survive without cascading collapse.
- Designing retry/backoff so you don't amplify a partial outage into a full one.
- A cache expires and every request simultaneously hits the DB (thundering herd).
- You need to keep the critical path (tips, auth) up while shedding non-critical work (analytics, notifications).
- An upstream dependency is degraded and you must degrade gracefully, not hard-fail.

## Portfolio Reality
baseLINE (Node/Express + Postgres + Redis, PM2, single VPS) plus the 0x-agents fleet (Telegram bots) and RetroByte share one host. A CPU-bound RetroByte SD pipeline already caused cron timeouts (Sep 2026). The 0x-agents cron fleet fires ~30 posts/day and has hit Buffer contention. There is **no load shedding, no bulkheads, and no formal degradation policy** today — this skill defines them before the next overload event. Related primitives exist: `pcmedicalist-backpressure`, `pcmedicalist-timeouts`, `pcmedicalist-circuit-breaker`.

## Topics Covered
1. Load Shedding
2. Bulkheads
3. Jitter
4. Thundering Herd
5. Graceful Degradation

Each topic: **Problem → Pattern → Validate → Fix**.

---

## 1. Load Shedding
**Problem:** When incoming rate exceeds capacity, accepting everything means everything is slow/fails. Shedding low-priority load protects the critical path.

**Pattern:**
```ts
import { RateLimiterMemory } from 'rate-limiter-flexible';
const critical = new RateLimiterMemory({ points: 100, duration: 1 });   // tips/auth
const background = new RateLimiterMemory({ points: 20, duration: 1 });  // analytics

app.post('/api/tips', async (req, res) => {
  try { await critical.consume(req.ip); }
  catch { return res.status(503).json({ error: 'overloaded', retryAfter: 1 }); }
  // ... handle tip
});
app.post('/api/analytics/event', async (req, res) => {
  try { await background.consume(req.ip); }
  catch { return res.status(202).json({ error: 'dropped', note: 'shed_load' }); } // 202 = ok to lose
  // ... enqueue
});
```
Shed based on: priority class, queue depth, latency budget, or a load score. Always shed the *least important* first. Return `503` (retryable) for user-facing critical, `202`/silent-drop for background telemetry.

**Validate:** Inject 10× normal traffic. Critical-path p99 stays within SLO; background events are dropped (count the `dropped` metric rises) but tips still succeed. No OOM/crash.

**Fix:** Add priority-tiered limiters. Wire a load signal (CPU, queue len) to the shed decision. Pre-classify every endpoint as critical / background.

---

## 2. Bulkheads
**Problem:** One slow dependency consumes all threads/connections, taking down unrelated functionality (a stuck Redis blocks tip writes that never touch Redis).

**Pattern:** Isolate resources per dependency — separate connection pools, separate worker queues, separate thread limits.
```ts
// separate pools: DB pool vs Bankr HTTP pool vs Redis pool — never share a single global pool
const bankrPool = new Agent({ maxSockets: 10, keepAlive: true });   // bounded, isolated
const dbPool = new Pool({ max: 20 });                                // Postgres own pool
// queue isolation: tip processing vs social-posting on different queues/workers
```
In Kubernetes, use separate `Deployment`s / `ResourceQuota`s per component so one can't starve another (see `pcmedicalist-infrastructure-ops` topic 193 resource limits).

**Validate:** Kill/stall the Bankr upstream. Tip-write latency (uses DB pool) is unaffected; only Bankr-dependent features degrade. No global thread exhaustion.

**Fix:** Audit shared pools. Split the monolithic pool into per-dependency bounded pools. Put independent features in separate processes/queues so a failure is contained.

---

## 3. Jitter
**Problem:** Synchronized retries/executions (N cron jobs firing at :00, N clients retrying simultaneously) create periodic overload spikes — the retry storm (see `pcmedicalist-backpressure`).

**Pattern:** Add randomized jitter to every retry backoff and every scheduled beat.
```ts
function backoff(attempt: number): number {
  const base = 100;                         // ms
  const cap = 10000;
  const exp = Math.min(cap, base * 2 ** attempt);
  return exp / 2 + Math.random() * (exp / 2);   // full jitter (e.g. AWS style)
}
// for cron: instead of exact :00, fire at :00 + random(0..59s)
const delay = Math.floor(Math.random() * 60000);
setTimeout(runJob, delay);
```
Use **full jitter** (random in [0, exp]) for retries to avoid synchronization; use **decorrelated jitter** when you need progressive backing-off without clumping.

**Validate:** Under failure, retry timestamps form a spread distribution, not aligned spikes. A fleet of 10 jobs doesn't all hit the dependency on the same second (measured via dependency request-time histogram).

**Fix:** Replace fixed `sleep(1000)` retries with jittered exponential backoff. Add jitter to all cron schedules that can coincide. Never let N clients share a hardcoded retry interval.

---

## 4. Thundering Herd
**Problem:** A massively-shared resource (one cache key, one DB row, one leader lock) expires or invalidates → all waiters stampede the origin at once. Classic: cache key for a viral creator's profile expires → 10k requests all hit Postgres.

**Pattern (request coalescing / single-flight):**
```ts
// Node single-flight: only ONE in-flight computation per key
const inflight = new Map<string, Promise<any>>();
async function getOrCompute(key: string, ttl: number, compute: () => Promise<any>) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);
  if (inflight.has(key)) return inflight.get(key)!;     // join the existing computation
  const p = (async () => {
    const val = await compute();
    await redis.set(key, JSON.stringify(val), 'EX', ttl);
    return val;
  })();
  inflight.set(key, p);
  try { return await p; } finally { inflight.delete(key); }
}
```
Plus: **proactive refresh** (recompute at `ttl - jitter` before expiry, never after), **request coalescing** at the proxy (nginx `proxy_cache_lock`), and **jittered TTLs** so keys don't all expire together (topic 3).

**Validate:** Expire a hot key under 10k RPS. Origin (DB) sees ~1 request, not 10k. `inflight` dedupes correctly; no stampede spike in DB CPU.

**Fix:** Add single-flight/coalescing to every "compute-if-absent" path. Set **staggered TTLs** (`ttl + random(0..ttl*0.2)`) so a fleet of keys doesn't expire in lockstep. Enable `proxy_cache_lock` at the edge.

---

## 5. Graceful Degradation
**Problem:** An upstream (Bankr, Privy, on-chain RPC) is slow/dead. Hard-failing every request makes a partial outage a total one. Degrade: serve reduced functionality, keep the core alive.

**Pattern:**
```ts
async function getCreatorProfile(id: string) {
  try {
    const live = await bankr.getBalance(id, { timeoutMs: 800 });
    return { ...profile, balance: live, degraded: false };
  } catch {
    // degrade: return cached/last-known balance, flag it
    const cached = await redis.get(`bal:${id}`);
    return { ...profile, balance: cached ?? null, degraded: true, note: 'balance_stale' };
  }
}
```
Define per-feature degraded modes **in advance** (not ad-hoc during incidents): tips keep working with cached balances; notifications queue and flush later (outbox); real-time leaderboard shows last-known. Pair with circuit breakers (`pcmedicalist-circuit-breaker`) so a dead dep fails fast instead of hanging.

**Validate:** Take down Bankr. Tip flow still completes (uses cached balance + outbox for the live check later). Response carries `degraded:true` so the UI can show "balance may be stale". No 500s cascade.

**Fix:** Enumerate features → define degraded mode for each. Wrap every external call with timeout + fallback. Add a circuit breaker. Document the degradation matrix in the runbook (see `pcmedicalist-incident-response`).

---

## Validation Checklist
- [ ] Priority-tiered load shedding in place; critical path protected (1)
- [ ] Per-dependency isolated pools/queues (bulkheads) (2)
- [ ] All retries use full/decorrelated jitter; cron schedules jittered (3)
- [ ] Hot shared resources use single-flight + staggered TTLs (4)
- [ ] Every external dependency has a defined degraded mode + breaker (5)

## Fix Recipes
- **"Everything slow under load"** → topic 1: add tiered shedding; topic 2: isolate pools.
- **"Retries cause spikes"** → topic 3: jittered backoff; `pcmedicalist-backpressure` for retry storms.
- **"DB spikes when cache clears"** → topic 4: single-flight + staggered TTL.
- **"One dead API kills the app"** → topic 5 + `pcmedicalist-circuit-breaker`: timeouts, fallback, breaker.

## Cross-References
- `pcmedicalist-backpressure` — retry storms, queue bounds
- `pcmedicalist-timeouts` — per-call deadlines feeding degradation
- `pcmedicalist-circuit-breaker` — fast-fail for degraded deps
- `pcmedicalist-cache-invalidation` — TTL strategy feeding thundering herd
- `pcmedicalist-dead-letter-queues` — shed/failed work lands here, not lost
- `pcmedicalist-event-driven` — async decoupling for degradation
- `pcmedicalist-infrastructure-ops` — resource limits (193), health checks (141)
- `pcmedicalist-incident-response` — degradation runbook
