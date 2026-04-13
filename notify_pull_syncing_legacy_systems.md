# Syncing Into a System You Don't Own

## The problem nobody writes about

There's a class of engineering problem that's extremely common and almost never written about well.

You have a modern system. It produces authoritative data. And somewhere downstream there's a legacy system — one you didn't build, can't rewrite, and have to keep in sync with. The legacy system has its own database schema, its own ORM, its own opinions about what a valid write looks like. It might be running on VMs with local filesystems. It might have 15 indexes on a single collection. It's probably load-sensitive in ways nobody fully understands anymore.

Your job: keep them consistent. Without breaking the legacy system. Without taking it over. Without a rewrite budget.

This is the story of how we approached that problem, why the obvious solutions failed, and what we ended up building — including the parts we haven't fully solved yet.

---

## The starting point: what was already there

The existing integration used inline message handlers. An event would arrive — athlete updated, team created, org changed — and the handler would immediately write to two places: a Mapping Service (which tracks ID relationships between the two systems) and the Legacy Database directly.

This created three compounding problems.

**Race conditions.** Handlers ran concurrently. If an athlete update and a team update arrived for the same org within milliseconds of each other, they'd both read stale state, compute independent writes, and apply them in arbitrary order. The last write won. Sometimes that was the right write. Sometimes it wasn't.

**Dual-write fragility.** Writing to two independent systems in sequence — with no transaction boundary between them — means any failure between the two writes leaves you in a half-committed state. Mapping created, legacy write failed: you have a ghost mapping pointing at a record that doesn't exist. Legacy write succeeded, mapping failed: you have a record the system can't find. Both states are silent. Neither throws a useful error.

**Legacy system sensitivity.** The legacy database was running near its IOPS ceiling. Uncontrolled write traffic from sync operations was a contributing factor to periodic slowdowns. A support investigation found the cluster was consistently saturating its provisioned IOPS on two of three nodes. With 15 indexes on the primary collection, every write was amplified.

And underneath all of this: the legacy system's ORM had strong opinions about what a valid document looked like. External writes that didn't conform to its internal schema requirements caused hydration failures in the legacy application. We were writing into a system that expected to be the only writer.

---

## Why the obvious solutions didn't work

Before landing on the final design, we went through several candidates.

**Payload-based sync** — use the event payload directly as the source of truth for the write.

Rejected. The message bus doesn't guarantee that a payload reflects the absolute latest state if multiple updates arrive in rapid succession. If three athlete updates fire in 200ms, you might process them out of order and write an older version of the athlete over a newer one. The only safe source of truth is the upstream API at the time of processing.

**Full-org sync on every event** — when anything changes in an org, re-sync the entire org.

Too expensive. An org can have hundreds of athletes and teams. A single athlete update triggering a full org fetch would saturate both the source API and the legacy write path. In a bulk upload scenario — say, a CSV of 50 athletes being imported — this would cause a write storm on an already load-sensitive system.

**Dedicated sync database** — introduce a third store as a transactional boundary, maintaining a snapshot of Hudl state plus Wimu mappings.

Rejected for cost and complexity. It would solve the dual-write problem cleanly — you'd have a single ACID boundary for the "did we process this?" question — but it adds a new system to operate, a new source of data to keep consistent, and a new failure mode. Given that the legacy system is on a sunset path, the investment wasn't justified.

**Redis state machine** — use Redis to coordinate ordering and deduplication.

Too stateful. Redis as a primary coordination mechanism introduces questions about what happens when Redis is unavailable, what the persistence guarantees are, and how you recover from partial state. The internal engineering guidance discourages treating Redis as a source of truth for exactly these reasons.

**Actor model (Orleans/Akka)** — one virtual actor per org, serializing all writes.

Technically elegant, operationally expensive. The virtual actor model gives you exactly the per-entity serialization semantics you want, but it's a significant runtime to adopt for a relatively low-volume sync pipeline. The operational complexity wasn't justified by the problem size.

---

## The pattern we landed on: Notify-Pull

The core insight is simple: **treat incoming events as signals, not as data.**

When an event arrives saying "athlete 123 was updated," don't trust the payload. Don't try to sync whatever the event contains. Instead, use the event as a trigger to go fetch the current state of athlete 123 from the authoritative source API — right now, at processing time.

This is sometimes called the Notify-Pull pattern. The notification tells you something changed. The pull gets you the truth.

It solves the stale payload problem entirely. It doesn't matter if three athlete events arrived out of order — when you process each one, you fetch the latest state and write it. The last fetch wins, and the last fetch is always fresh.

It also makes the system stateless in a useful way. The worker doesn't need to track "what version did I last write?" It just fetches current state and upserts. If it processes the same event twice, the second upsert is a no-op. Idempotency falls out naturally.

The tradeoff is API load. Every event triggers a fetch from the source API. For high-frequency event streams this can be a concern. We address it with scoped fetching — more on that below.

---

## The concurrency model: SQS FIFO as a per-org mutex

Even with Notify-Pull, you still have a concurrency problem. If two events for the same org arrive simultaneously and both trigger a fetch-and-write, you can still get race conditions at the legacy database layer.

The solution: serialize processing per org.

We use SQS FIFO queues with `MessageGroupId = OrgId`. This means all events for a given org are processed by at most one consumer at a time. Events for Org A and Org B can process in parallel. Events for Org A are processed strictly in sequence.

This eliminates the race condition at the root. You can't have two concurrent writes for the same org because the queue won't deliver the second message until the first is acknowledged.

It also gives us things the original Janus-based approach didn't have: dead letter queues for messages that exhaust retries, configurable visibility timeouts, and batch processing support.

We considered RabbitMQ with consistent hashing exchange as an alternative — it can provide equivalent per-entity serialization semantics and is the standard messaging infrastructure at our company. We chose SQS FIFO for this specific pipeline to reduce operational overhead: no exchange bindings to manage, no cluster scaling to configure. If the pipeline needs to integrate more tightly with the broader event bus in the future, RabbitMQ remains the fallback.

---

## The mental model shift: legacy DB as write target, not master

This one took some discussion.

The instinct when dealing with a legacy system is to treat its database as authoritative. It's been running for years. It has data. It knows things. You don't want to overwrite it with something wrong.

We inverted this. The legacy database is a **write target**. The source API is the master. Our job is to keep the write target consistent with the master, not to negotiate between them.

This changes how you think about conflicts. If the legacy database has a record that differs from what the source API says, the source API wins. Every time. The legacy record gets overwritten. There's no merge logic, no last-write-wins timestamp comparison, no version negotiation.

It also changes how you think about failures. If a write to the legacy database fails, the right response isn't to figure out what partial state was written and how to reconcile it. The right response is to retry the full fetch-and-upsert. The source API has the truth. Fetch it again and apply it.

The one exception: deletions. The pull model handles updates well but can't detect deletions on its own — if an entity is removed from Hudl, no "updated" event fires, and a fetch would just return 404. Deletions are handled separately: real-time via explicit `EntityDeleted` events, and as a safety net via a nightly reconciliation job that compares active IDs in Hudl against the legacy database and soft-deletes anything that no longer exists upstream.

---

## Scoped fetching: protecting the source API

Full-org sync on every event was too expensive. But pure single-entity sync misses referential integrity — you can't write an athlete without knowing their team exists, and you can't write a team without knowing their org exists.

We use what we're calling scoped fetching. When an event arrives for an entity, we fetch the minimal chain of ancestors needed to guarantee referential integrity.

For an athlete update: fetch the athlete, their parent team, and the parent org.  
For a team update: fetch the team and the parent org.  
For an org update: fetch the org only.

This is enough to do a safe root-to-leaf upsert (org → team → athlete) without pulling entities that haven't changed. It keeps API load proportional to the event, not to the org size.

The root-to-leaf ordering matters. Parents must exist before children. If you write an athlete before their team exists in the legacy database, you get a referential integrity violation. Writing in root-to-leaf order means each write can assume its parent is already in place.

---

## The Anti-Corruption Layer: writing into a system with opinions

The legacy system's ORM expects documents to conform to specific internal schema requirements. Fields it doesn't expect in certain positions, or missing fields it assumes exist, cause hydration failures in the live application.

We don't want our sync logic to need intimate knowledge of the legacy ORM internals. That's a coupling that would make every schema evolution in the legacy system a breaking change for us.

The Anti-Corruption Layer sits between our sync logic and the legacy database write. It takes our domain representation of an entity and transforms it into a form the legacy ORM will accept — handling field mapping, required default values, and format normalization. If the legacy schema changes, we update the ACL, not the sync logic.

This is a standard DDD pattern but it earns its complexity here. Without it, every legacy schema quirk leaks into our codebase.

---

## ID management: deterministic mapping without coordination

One of the less obvious problems in syncing between two systems is ID translation. Hudl entities have Hudl IDs. Legacy Wimu entities have their own IDs. The Mapping Service maintains the translation table.

For entities that already exist in both systems, the mapping is pre-populated. For new entities — things created in Hudl after the integration is live — we need to generate legacy IDs on the fly.

We generate these deterministically: `deterministicId(namespace, hudlId)` always produces the same legacy ID for the same input. If the same "new athlete" event is processed twice (due to a retry), both runs generate the same ID and attempt the same upsert. The second run is a no-op. No duplicate records, no coordination required between consumers.

The alternative — generating a random ID and storing it — requires a read-before-write to check if you've already mapped this entity. Under concurrent processing that's a race condition waiting to happen. Deterministic generation sidesteps the problem entirely.

---

## What we haven't solved: the burst problem

The design handles steady-state traffic well. It doesn't fully solve the burst case.

When a new organisation is onboarded via CSV upload — say, 50 athletes imported in one action — the source system fires 50+ `AthleteUpdated` events in rapid succession. With `MessageGroupId = OrgId`, these all land in the same FIFO queue group and are processed one at a time. That's correct for consistency but means the legacy database receives 50 sequential write operations from a single burst.

The burst itself isn't the problem — the legacy database can handle 50 writes. The problem is that each write involves a scoped fetch (hitting the source API), a mapping resolution, and an upsert with writes across multiple indexed fields. Under burst conditions, this compounds.

Options we considered:

**Debouncing** — collapse multiple events for the same entity within a time window into one. Works in principle but naive implementations (tumbling windows) drop all events except the first in the window. We need the *latest* state, which means we actually want the last event in the window, not the first. Getting this right without a stateful accumulator (which brings Redis back into the picture) is non-trivial.

**Batch processing** — SQS FIFO supports receiving up to 10 messages per poll. We could batch-receive, deduplicate by entity ID (keeping only the most recent event per entity), and process the deduplicated set. This reduces write count proportionally to duplicate rate. In a 50-athlete CSV upload, many of those events are for different athletes, so deduplication gains may be limited.

**Artificial throttling** — introduce a configurable delay between writes within a burst. Simple to implement, but interacts badly with SQS visibility timeout: if processing takes too long, the message becomes visible again and gets redelivered.

We're shipping the MVP without a clean solution to this and monitoring whether it's actually a problem in practice. The legacy database has been patched to increase IOPS provisioning. The burst scenario affects a single org at onboarding time, not steady-state traffic. If it becomes a problem, the deduplicated batch approach is the most promising path.

---

## Phased delivery: validate before you go real-time

We're shipping in two phases deliberately.

**Phase 1 (nightly):** a reconciliation job that runs the full fetch-and-upsert logic against all active orgs on a schedule. No real-time events yet. This lets us validate the scoped fetching, root-to-leaf upsert, ACL transformation, and deterministic ID generation in a controlled, low-concurrency environment before we open the event firehose.

If the nightly job produces correct results, we have high confidence that the same logic running in real-time will be correct. If it produces incorrect results, we find out during a scheduled batch window rather than during a live user session.

**Phase 2 (real-time):** connect the SQS FIFO consumer. The nightly job continues running as a safety net — it will catch anything the real-time path missed due to dropped events, consumer failures, or edge cases.

The temptation in projects like this is to skip the nightly phase and go straight to real-time. The argument is usually "we'll catch bugs faster with real traffic." That's true, but "faster" in this context means "in production, during user sessions, against a load-sensitive legacy database." The nightly phase is cheap insurance.

---

## The lessons

**Events are signals, not data.** If your source system is authoritative, don't try to sync event payloads. Use events to trigger fetches from the API. The payload is stale by definition the moment it leaves the source.

**Serialize per entity at the infrastructure level.** Application-level locks and CAS logic for concurrency control are fragile. Route all events for a given entity to a single consumer using your queue's native partitioning. Let the infrastructure enforce the ordering.

**Treat the legacy system as a write target.** If you have a canonical source of truth upstream, the downstream system's state is derived. Own that framing. It simplifies conflict resolution, retry logic, and failure handling considerably.

**Idempotency is load-bearing.** In an eventually consistent sync system, you will process the same event more than once. Deterministic IDs and unconditional upserts mean that's fine. Build idempotency in from the start, not as an afterthought.

**The burst problem is real and often underplanned.** Steady-state performance is easy to reason about. Burst behaviour — bulk imports, org onboarding, backfills — is where sync systems typically break. Have an answer, or at least an honest acknowledgement of the gap, before you go to production.

**Phase your delivery.** Validate the logic in a controlled batch environment before connecting it to real-time events. The nightly reconciliation job isn't just a safety net — it's a test harness.

---

## What's next

The immediate next step is instrumenting the nightly reconciliation job and measuring write latency, error rate, and IOPS impact against the legacy database under real load. That data will inform whether the burst problem is theoretical or actual, and whether the ACL transformations are producing clean writes.

If the nightly phase looks good, we connect the SQS consumer and monitor for the burst scenario. The deduplication-in-batch approach is the first thing we'd reach for if burst becomes a problem.

Longer term, the legacy system is on a sunset path. At some point the sync problem goes away entirely. The goal of this architecture isn't elegance — it's buying reliable consistency at low operational cost until we don't need it anymore.

---

*If you're working on a similar problem — syncing into a system you don't own, bridging legacy and modern infrastructure, or dealing with dual-write consistency — the patterns here are general enough to apply beyond this specific context. The Notify-Pull pattern in particular is underused relative to how often the problem appears.*
