# Low-Level Design (LLD) :)

## 1. What I have seen in LLD interviews in last 2 months ?

You are given a small system — a parking lot, a wallet, a rate limiter — and expected to produce **classes, interfaces, relationships, and working core flows** in 45-120 minutes.

### 1.1 Good vs Excellent - Isse generally interviewer khush ho jata h

| Dimension       | Good expectation                                                | Excellent                                                                                                       |
| --------------- | --------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Requirements    | Clarifies scope, lists functional requirements                  | Also identifies non-functional constraints (concurrency, extensibility, failure modes) unprompted               |
| Entity modeling | Correct classes, sensible relationships                         | Justifies composition vs inheritance choices, models for change ("payment methods will grow, so Strategy here") |
| Patterns        | Applies 2–3 patterns correctly when the fit is obvious          | Knows when **not** to apply a pattern; avoids over-engineering; names trade-offs                                |
| SOLID           | Can define and mostly follow                                    | Design visibly embodies it — small interfaces, dependency inversion at boundaries, open for extension           |
| Concurrency     | Handles it when asked ("what if two users book the same seat?") | Raises it proactively; knows lock granularity, optimistic vs pessimistic, thread-safe collections, idempotency  |
| Code quality    | Compiles, works, reasonably named                               | Also testable (interfaces + DI), exception strategy, immutability where sensible, no God classes                |
| Trade-offs      | States one option                                               | Presents 2 options with costs, picks one, defends it, and can reverse under new constraints                     |
| Driving         | Answers interviewer's questions                                 | Drives the session: proposes scope cuts, timeboxes, states assumptions, checkpoints with interviewer            |

## 2. Should Have Idea of OOPs

I have seen Vehicle, Car, Engine questions a lot <br>
Vehicle => {Car, Bus, Truck, ...} <br>
Car => {Audi, Mercedes, BMW, ...} <br>
Engine => {4 Cylinder Engine, 6 Cylinder Engine, 12 Cylinder Engine} <br>

### 2.1 OOP fundamentals

- **Four pillars** — sbse important **Encapsulation** and **Polymophism**
- **Composition over inheritance** — Yaad krke jana: combinatorial explosion of subclasses with an example like `Vehicle`/`Engine` or notifications.
- **Coupling and cohesion** — jitna jyada `separation of concern rakh sko`, example "I'm keeping `PricingService` decoupled from `ParkingLot` so pricing rules can change independently."
- **Abstract class vs interface** — rat ke jana for c++/java whatever you are using
- **Immutability** — value objects (`Money`, `Location`, `TimeSlot`) should be immutable; `ENUMs` pr bhi kafi focus rhta h.
- **Association vs Aggregation vs Composition** — you'll draw these in class diagrams; know the lifetime semantics.

### 2.2 SOLID Principles

Principle yaad mat krna, example krna

| Principle                 | The interview "move"                                                | Example callout in a session                                              |
| ------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| **S**ingle Responsibility | Split God classes; one reason to change per class                   | "Splitting `Invoice` from `InvoicePrinter` and `InvoiceRepository`"       |
| **O**pen/Closed           | New behavior via new classes, not edited switch statements          | Adding `UPIPayment` without touching `PaymentProcessor`                   |
| **L**iskov Substitution   | Subtypes honor the contract; avoid `Square extends Rectangle` traps | `ReadOnlyList` throwing on `add()` violates LSP                           |
| **I**nterface Segregation | Small role-based interfaces                                         | `Printable`, `Scannable` instead of fat `Machine`                         |
| **D**ependency Inversion  | Depend on abstractions; inject dependencies via constructor         | `NotificationService(sender: MessageSender)` — enables testing & swapping |

### 2.3 Design patterns

You do **not** need all 23 GoF patterns. Roughly 14 cover ~95% of LLD interviews. Study each as: _problem it solves → structure → one LLD question where it appears → when it's overkill._

**Tier A — appear constantly:**

1. **Strategy** — interchangeable algorithms. Pricing strategies, payment methods, parking-spot allocation, split strategies in Splitwise, eviction policies in a cache.
2. **Factory Method / Abstract Factory** — object creation without exposing instantiation logic. Vehicle factory, notification factory, parser factory.
3. **Singleton** — single shared instance (config, connection pool, logger). Know thread-safe variants: eager, enum singleton, initialization-on-demand holder. Also know _why it's criticized_ (hidden global state, hard to test).
4. **Observer** — event notification. Stock price alerts, order status updates, pub-sub, cache invalidation listeners.
5. **Builder** — complex object construction. `Order - (Zomato)`, `HttpRequest`, immutable config objects; telescoping-constructor problem.
6. **Decorator** — layering behavior. Pizza toppings, coffee add-ons, stream wrappers, middleware, retry/logging wrappers around a client.
7. **State** — behavior varies by state. Vending machine, elevator, order lifecycle, ATM, traffic signal, document workflow.
8. **Command** — encapsulate requests. Undo/redo in text editor, task scheduler jobs, remote control, transactional operations.

**Tier B — know well (frequent in follow-ups):**

9. **Chain of Responsibility** — request passes through handlers. ATM cash dispensing (denominations), logging levels, approval workflows, middleware/filters, rule engines.
10. **Adapter** — bridging incompatible interfaces. Wrapping third-party payment gateways (very relevant to fintech), legacy APIs.
11. **Facade** — simplified entry point over subsystems. `BookingFacade` orchestrating inventory + payment + notification.
12. **Template Method** — algorithm skeleton with overridable steps. Report generation, data pipeline stages, game turn loops.
13. **Proxy** — access control / lazy loading / caching in front of a real object. Rate-limiting proxy, cached repository.
14. **Iterator** — custom traversal (occasionally asked directly: "design an iterator over a nested list").

**Tier C — recognize on sight:** Composite (file system, org hierarchy, UI trees), Flyweight (chess pieces, text glyphs), Mediator (chat room, air traffic control), Memento (undo snapshots), Prototype (object cloning), Bridge (abstraction × implementation matrices like shapes × renderers).

### 2.4 Concurrency for LLD (optional, but I recommend :)

- **C++/Java memory model basics:** `stack` vs `heap` discussion
- **Locks:** `Lock/Mutex`, `ReadWriteLock`, lock granularity (lock per parking floor vs whole lot), lock ordering to avoid deadlock.
- **Concurrent collections:** `ConcurrentHashMap` (incl. `compute`/`putIfAbsent` for atomic check-then-act), `CopyOnWriteArrayList`, `BlockingQueue` family.
- **Atomics & CAS:** `AtomicInteger`, `AtomicReference`, when CAS beats locking.
- **Coordination:** `CountDownLatch`, `Semaphore` (bounded resources — parking spots, connection pool!), `CyclicBarrier`, `Condition`/`wait-notify`.
- **Executors:** thread pool sizing intuition, `ScheduledExecutorService` (task schedulers, TTL cleanup).
- **Classic patterns to be able to code from scratch:** producer–consumer with `BlockingQueue` _and_ with `wait/notify`; bounded blocking queue; thread-safe Singleton; thread-safe LRU cache; token-bucket rate limiter; read-write lock usage.
- **Transactional thinking:** Very very important ^\_^ optimistic locking (version numbers) vs pessimistic; idempotency keys for payments; the "check-then-act" race (seat booking, wallet debit) and how to close it.

### 2.5 API, schema, and error design

- Designing method signatures: what to return (`Optional`, result objects, exceptions?), what to accept (IDs vs objects), pagination on list methods.
- Exception strategy: checked vs unchecked, domain exceptions (`InsufficientBalanceException`), fail-fast validation.
- Enums vs class hierarchies vs configuration for variant behavior.
- Basic persistence mapping: which entities become tables, where you'd put a unique constraint (interviewers at product companies often ask "how would this look in the DB?").
- ID generation, timestamps, soft deletes.

### 2.6 UML-lite - not very important

You need just enough to draw fast and unambiguously:

- Class boxes: name / key fields / key methods.
- Arrows: solid arrow = association, hollow triangle = inheritance ("is-a"), dashed hollow triangle = interface implementation, filled diamond = composition ("owns"), hollow diamond = aggregation ("has").
- Multiplicities: `1`, `*`, `0..1` on association ends.
- A simple sequence diagram for one core flow (e.g., "book ticket").

## 3. Discussion Format - Which I follow - Time is approximated by Claude :)

A repeatable timeline. Practice until it's muscle memory.

| Time      | Phase                        | What you do                                                                                         | What you say out loud                                                                                                                 |
| --------- | ---------------------------- | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| 0–7 min   | **Requirements**             | Ask 5–8 clarifying questions; write functional requirements; explicitly cut scope                   | "I'll support multi-floor and multiple vehicle types, and keep payments as a pluggable interface but stub one implementation — fair?" |
| 7–12 min  | **Entities & relationships** | List core nouns → classes; identify verbs → methods/services; call out cardinalities                | "A `Floor` _composes_ `ParkingSpot`s; a `Ticket` _associates_ a `Vehicle` with a `Spot`"                                              |
| 12–25 min | **Class design**             | Interfaces first, then classes; enums for fixed sets; mark where patterns apply and why             | "Spot allocation is a Strategy so we can swap nearest-first for random under load"                                                    |
| 25–40 min | **Core flows + code**        | Write real code for 1–2 critical flows (entry→ticket, exit→payment); method signatures for the rest | Narrate invariants: "spot assignment must be atomic, so I'll use `compute` on a `ConcurrentHashMap`"                                  |
| 40–50 min | **Hardening**                | Concurrency, edge cases, error handling, extensibility follow-ups                                   | Raise one yourself before being asked — strong senior signal                                                                          |
| 50–60 min | **Extensions & wrap**        | Handle the interviewer's curveball ("add EV charging spots"); summarize design decisions            | "Because allocation was already a Strategy, EV support is a new spot type + new strategy — no existing class changes"                 |

**Universal opening questions to have ready:** Who are the actors? Read-heavy or write-heavy? Single machine or distributed (LLD usually = single process)? Do we need persistence or in-memory OK? Concurrency expected? What's explicitly out of scope?

---

## 4. The Question Bank

Organized by category and priority. For each: what it _really_ tests, the patterns involved, and the follow-ups interviewers use to separate SDE-2 from Senior.

### 4.1 Tier 1

| #   | Problem                               | Core skills tested                       | Patterns                                                                                                      | follow-ups                                                                                                                                                                     |
| --- | ------------------------------------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | **Parking Lot**                       | Entity modeling, enums, allocation logic | Strategy (allocation, pricing), Factory (vehicles), Singleton (lot)                                           | Concurrency on spot assignment; multiple entry gates; EV/handicap spots; dynamic pricing; nearest-spot efficiency (min-heap per type)                                          |
| 2   | **Elevator System**                   | State machines, scheduling algorithms    | State, Strategy (scheduling: SCAN/LOOK), Observer (buttons), Command                                          | Multiple elevators dispatch; direction optimization; starvation; emergency mode; concurrency of request queue                                                                  |
| 3   | **Splitwise (Expense Sharing)**       | Data modeling, algorithmic core          | Strategy (equal/exact/percent splits), Factory, Observer (notifications)                                      | **Debt simplification** (min transactions — greedy/heap); concurrent expense adds; recurring expenses; groups; settle-up flow                                                  |
| 4   | **Vending Machine**                   | State transitions, money handling        | State (idle→hasMoney→dispensing), Chain of Responsibility (change/denominations)                              | Exact change unavailable; inventory refill; concurrent button presses; card payments as new state path                                                                         |
| 5   | **BookMyShow / Movie Ticket Booking** | Contention, booking flow                 | Strategy (pricing), Observer, Facade (booking orchestration)                                                  | **Seat locking**: pessimistic lock vs optimistic with timeout; double booking; payment failure → seat release (timeout/finally); waitlists                                     |
| 6   | **LRU / LFU Cache**                   | DS + design + concurrency in one         | Strategy (eviction), Decorator (metrics/TTL layers)                                                           | Code it: HashMap + DLL for LRU, freq buckets for LFU; make it thread-safe (segmented locks vs synchronized vs `ConcurrentHashMap` + care); add TTL; add write-back persistence |
| 7   | **Rate Limiter**                      | Algorithms + concurrency + API design    | Strategy (token bucket / sliding window / leaky bucket / fixed window), Factory                               | Per-user vs global; burst handling; token bucket with atomics vs lock; distributed variant discussion (Redis + Lua) — keep brief, it's LLD                                     |
| 8   | **Logger Framework**                  | Framework/library design                 | Chain of Responsibility (levels), Observer/Strategy (appenders: console/file/DB), Singleton, Builder (config) | Async logging (`BlockingQueue` + consumer thread); log rotation; formatting pluggability; never-block-the-caller guarantee                                                     |
| 9   | **Tic-Tac-Toe → Chess**               | Game loop, board modeling, extensibility | Factory (pieces), Strategy (move validation per piece), Command (moves, undo), Memento                        | N×N tic-tac-toe win check in O(1) per move; chess: piece polymorphism vs move-strategy; undo/redo; check/checkmate detection scoping                                           |
| 10  | **Snake & Ladder / Ludo**             | Clean game modeling, randomness, turns   | Factory, Strategy (dice, rules), State (player turns)                                                         | Multiple dice; pluggable board entities (snakes/ladders/mines as one abstraction); replay from move log (Command)                                                              |

### 4.2 Tier 2

**Booking / commerce / fintech (prioritize these for fintech and product companies):**

- **Food delivery (Zomato/Swiggy LLD)** — restaurant, menu, cart, order lifecycle (State), rider assignment (Strategy), notifications (Observer). Follow-ups: order state machine transitions and illegal-transition guarding; cart pricing with stacked offers (Decorator or CoR rule engine).
- **Ride hailing (Uber/Ola LLD)** — rider/driver matching (Strategy), trip state machine, fare calculation (Strategy + Decorator for surge), location as value object. Follow-up: nearest-driver structure (grid/quadtree — mention, don't build).
- **Hotel booking / Airbnb** — room inventory, date-range availability (interval overlap), reservation locking, cancellation policies (Strategy).
- **Wallet / Payments system** — account, transaction ledger (immutable, append-only), debit/credit atomicity, **idempotency keys**, transaction states (State), gateway adapters (Adapter). Fintech interviewers go deep here: double-spend prevention, optimistic locking with version column, reconciliation hooks.
- **Amazon Locker** — locker sizes, allocation (Strategy), OTP pickup, expiry (ScheduledExecutor), returns.
- **Inventory management** — SKU, warehouses, reservations vs stock, oversell prevention (atomic decrement).
- **Online auction (eBay)** — bid validation, highest-bid concurrency (CAS on `AtomicReference` or lock per item), auto-bidding (Observer + Strategy), auction close scheduling.
- **Coupon / Offer engine** — rule evaluation (Chain of Responsibility / Specification pattern), stacking rules, expiry.

**Infra-flavored LLD:**

- **Task Scheduler (cron-like)** — `ScheduledExecutorService` or DelayQueue + worker pool; Command for jobs; retry with backoff (Decorator); priorities; recurring jobs; missed-run policy.
- **Notification service** — channels (Strategy/Factory: email/SMS/push), templates, retries, rate limiting per channel, user preferences, dedup.
- **Pub-Sub / Message Queue (in-memory Kafka-lite)** — topics, partitions optional, consumer groups, offset tracking, `BlockingQueue` per topic, at-least-once delivery discussion.
- **Key-Value store (in-memory)** — TTL (lazy + sweeper thread), eviction (Strategy), snapshots (Memento), optional WAL discussion (you've built MiniDB — this maps directly).
- **Connection pool / Object pool** — `Semaphore` + `BlockingQueue`, borrow/return, validation on borrow, max-wait timeout, leak detection.
- **Thread pool from scratch** — worker threads polling a `BlockingQueue<Runnable>`, shutdown semantics (graceful vs now), rejection policies (Strategy).
- **File system (in-memory)** — Composite (File/Directory), path resolution, permissions (Proxy), search.
- **Circuit breaker** — State pattern textbook case (closed→open→half-open), failure-rate window, half-open probe concurrency.
- **URL shortener (LLD slice)** — encoding strategy, collision handling, custom aliases, expiry; keep it in-process.

**Classic OO modeling:**

- **Library Management** — catalog vs copies distinction, reservations, fines (Strategy).
- **ATM** — State + Chain of Responsibility (cash denominations) double feature.
- **Stack Overflow / Reddit** — posts, votes (one per user — enforce how?), reputation events (Observer), badges.
- **Chess extension of #9**, **Poker/Blackjack** — deck (Factory + shuffle), hand evaluation (Strategy/CoR), betting rounds (State).
- **Meeting Scheduler / Calendar** — interval conflicts, room allocation, recurring events (RRULE-lite), reminders (Observer + scheduler).
- **Text editor (mini)** — Command (undo/redo stacks), Memento (snapshots), Flyweight (characters/glyphs) discussion.
- **Cricinfo / Live scoreboard** — Observer everywhere; ball-by-ball event model; derived stats.
- **Traffic signal controller** — State + timers; intersection coordination (Mediator).
- **Coffee/Chai machine** — recipes (Builder/Template Method), ingredient inventory, concurrent dispensing (Semaphore on shared ingredients).

### 4.3 Tier 3

These test _library API design_ — the strongest senior differentiator:

- **Design a retry library** — policy (Strategy: fixed/exponential/jitter), max attempts, retryable-exception predicates, Decorator around any `Callable`, integration with circuit breaker.
- **Design a DI container (mini-Spring)** — registration API, singleton vs prototype scopes, constructor injection via reflection, circular dependency detection.
- **Design a feature-flag SDK** — flag evaluation (Strategy per rollout type: boolean/percentage/user-cohort), local cache + refresh, default fallbacks, thread safety.
- **Design a metrics library (mini-Micrometer)** — counters/gauges/timers, registry (Singleton), reporters (Observer/Strategy), lock-free counters (`LongAdder`).
- **Design an event bus (mini-Guava EventBus)** — subscribe via interface or annotation, sync vs async dispatch, error isolation between subscribers, ordering guarantees.
- **Design a config management client** — layered sources (CoR: env > file > defaults), typed access, hot reload (Observer).
- **Design a workflow/approval engine** — states, transitions, guards, pluggable actions (Command), persistence hooks.

### 4.4 Concurrency drill set (code every one of these from scratch)

1. Bounded blocking queue — with `wait/notify` **and** with `ReentrantLock`+`Condition`.
2. Producer–consumer with graceful shutdown (poison pill).
3. Thread-safe LRU cache (compare: full `synchronized` vs `ReadWriteLock` vs segment locking).
4. Token-bucket rate limiter (lock-based, then sketch the CAS version).
5. Read-write lock usage on a shared registry; then discuss `StampedLock` optimistic reads.
6. Thread-safe Singleton — all four idioms, and _why_ double-checked locking needs `volatile`.
7. Dining philosophers (lock ordering / tryLock solution) — appears at quant-adjacent and infra shops.
8. In-memory job scheduler with delay queue and worker pool.
9. Seat-booking race: demonstrate the check-then-act bug, fix with `putIfAbsent`/lock, then with optimistic versioning.
10. Concurrent counter: `synchronized` vs `AtomicLong` vs `LongAdder` — when each wins.

---

## 5. Pattern → Problem :)

| Pattern                 | Shows up in                                                                                                                                        |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Strategy                | Parking pricing/allocation, Splitwise splits, cache eviction, rate-limiter algorithms, elevator scheduling, fare/pricing, matching, retry policies |
| State                   | Vending machine, elevator, order lifecycle, ATM, circuit breaker, trip lifecycle, auction lifecycle, traffic signal                                |
| Factory                 | Vehicles, notifications, parsers, pieces, payment instruments                                                                                      |
| Observer                | Notifications, scoreboards, order tracking, stock alerts, event bus, cache invalidation                                                            |
| Builder                 | Orders, requests, config, reports                                                                                                                  |
| Decorator               | Toppings/add-ons, retries, logging/metrics wrappers, surge pricing layers, TTL on cache                                                            |
| Command                 | Undo/redo, scheduler jobs, remote ops, move replay                                                                                                 |
| Chain of Responsibility | ATM denominations, log levels, discount/rule engines, approval flows, middleware                                                                   |
| Singleton               | Lot/registry/config/pool instances (use sparingly; justify)                                                                                        |
| Adapter                 | Payment gateways, third-party SDK wrappers, legacy interfaces                                                                                      |
| Facade                  | Booking orchestration, checkout, subsystem entry points                                                                                            |
| Composite               | File system, org charts, menu categories, UI trees                                                                                                 |
| Template Method         | Report/pipeline/game-turn skeletons                                                                                                                |
| Proxy                   | Cached repository, access control, rate-limiting front                                                                                             |
| Memento                 | Editor snapshots, KV-store snapshots, game save                                                                                                    |
| Mediator                | Chat room, intersection controller, air traffic                                                                                                    |
| Flyweight               | Chess pieces, text glyphs                                                                                                                          |
