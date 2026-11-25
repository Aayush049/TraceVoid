# TraceVoid — Architecture Decision Records (ADR)
> *Last Updated:* 2026-11-25  
> *Status:* Living Document — Updated as decisions are finalized

---

## ADR-001: Project Naming

| Field | Value |
|---|---|
| **Status** | ✅ Decided |
| **Decision** | **TraceVoid** everywhere |
| **Package** | `com.aayush.tracevoid` |
| **App Label** | TraceVoid |
| **Service Name** | TraceVoidVpnService |
| **Application Class** | TraceVoidApp |
| **Database Class** | TraceVoidDatabase |

---

## ADR-002: Module Strategy — Single Module with Strict Package Boundaries

| Field | Value |
|---|---|
| **Status** | ✅ Decided |
| **Decision** | **Single `:app` module** with enforced package-level Clean Architecture |

### Why Single Module (Not Multi-Module)

**Multi-module** (`:core:network`, `:core:database`, `:domain`, `:feature:*`) sounds impressive on paper, but for a **single-developer portfolio project** it introduces significant overhead with marginal benefit:

| Factor | Single Module | Multi-Module |
|---|---|---|
| **Gradle config overhead** | 1 `build.gradle.kts` | 8-12 files, each needing plugin/dependency setup |
| **Hilt wiring complexity** | Standard `@HiltAndroidApp` | Needs `@EntryPoint` bridges, component dependencies across modules |
| **Build times** | Fast (single compilation unit) | Faster *incremental* builds, but slower *clean* builds |
| **Refactoring friction** | Move a file | Move a file + update module dependency graph |
| **Architecture enforcement** | Package discipline + code review | Compile-time boundary enforcement |
| **Recruiter impression** | ✅ Clean Architecture is visible in package tree | ✅ Shows modularization skill |

### Our Approach: Strict Package Discipline

The package structure already enforces clean architecture:

```
com.aayush.tracevoid/
├── core/        → Infrastructure (VPN, Room, DI) — no Android UI imports allowed
├── domain/      → Pure Kotlin models & use cases — zero Android framework imports
└── feature/     → Compose UI + ViewModels — depends on domain/, delegates to core/
```

**Rules enforced by convention (and later by lint/detekt rules):**
1. `domain/` imports **nothing** from `core/` or `feature/`
2. `feature/` imports from `domain/` (models, use cases) but **never** directly from `core/`
3. `core/` provides implementations injected via Hilt — `feature/` only sees `domain/` interfaces

> [!NOTE]
> If the project grows beyond a solo endeavor (e.g., open-source contributors), we can extract modules later. The package structure makes this a mechanical refactor, not an architectural one.

---

## ADR-003: Tracker Rule Source — DuckDuckGo Tracker Radar (Primary)

| Field | Value |
|---|---|
| **Status** | ✅ Decided |
| **Decision** | **DuckDuckGo Tracker Radar** as primary source, supplemented with curated EasyList domains |

### Why DuckDuckGo Tracker Radar

| Criteria | DuckDuckGo Tracker Radar | EasyList | AdGuard |
|---|---|---|---|
| **Format** | JSON (per-domain files, easy to parse) | Text filter syntax (complex regex) | Mixed filter + JSON |
| **Categorization** | ✅ Rich (24 categories: Analytics, Advertising, Session Replay, etc.) | ❌ No categories | Partial |
| **Maintenance** | Actively maintained by DDG via automated crawls | Community maintained | Company maintained |
| **License** | CC BY-NC-SA 4.0 (fine for non-commercial) | GPL | GPL |
| **Domain count** | ~5,000 tracker entities mapping to ~50,000+ domains/subdomains | ~70,000 filters (many are URL patterns, not pure domains) | ~40,000 |
| **False positive rate** | Low — categories like CDN, Payment, SSO are marked non-tracking | Higher — broad pattern matching | Medium |

### Implementation Plan

```
assets/
└── tracker_rules/
    ├── trackerradar_domains.json    # Pre-processed: {domain → category} flat map
    └── version.txt                  # Rule version for future update checking
```

**Pipeline:**
1. **Build-time preprocessing:** A Gradle task (or manual script) downloads the latest Tracker Radar repo, flattens per-domain JSON files into a single `{domain: category}` map, filters out non-blocking categories (CDN, Online Payment, Federated Login), and outputs `trackerradar_domains.json`
2. **First launch:** Room `RoomDatabase.Callback.onCreate()` reads the asset file and bulk-inserts into `TrackerRuleEntity` table
3. **Runtime:** On app start, all rules are loaded into the in-memory `DomainTrie` for $O(k)$ matching

### Categories We BLOCK (high-risk tracking behavior)

| Category | Action | Rationale |
|---|---|---|
| Ad Motivated Tracking | 🔴 BLOCK | Core privacy threat — cross-site profiling |
| Advertising | 🔴 BLOCK | Ad delivery + beacons |
| Analytics | 🔴 BLOCK | Telemetry & usage tracking |
| Session Replay | 🔴 BLOCK | Records user interactions — severe privacy risk |
| Action Pixels | 🔴 BLOCK | Conversion tracking pixels |
| Audience Measurement | 🔴 BLOCK | Demographic profiling |
| Social - Share | 🟡 WARN | Can break share functionality — user choice |
| Tag Manager | 🔴 BLOCK | Dynamically injects multiple trackers |
| Third-Party Analytics Marketing | 🔴 BLOCK | Marketing + lead profiling |
| Unknown High Risk Behavior | 🔴 BLOCK | Suspicious unclassified domains |
| Malware | 🔴 BLOCK | Malicious payloads |

### Categories We ALLOW (functional, non-tracking)

| Category | Action | Rationale |
|---|---|---|
| CDN | ✅ ALLOW | Static asset delivery — blocking breaks sites |
| Online Payment | ✅ ALLOW | Payment gateways must function |
| Federated Login / SSO | ✅ ALLOW | Login flows — blocking locks users out |
| Consent Management Platform | ✅ ALLOW | Cookie consent banners — required by law |
| Non-Tracking | ✅ ALLOW | Explicitly non-tracking infrastructure |
| Fraud Prevention | ✅ ALLOW | Security tooling — blocking weakens app security |
| Support Chat Widget | 🟡 WARN | Useful but can track — user choice |
| Embedded Content | 🟡 WARN | Videos/widgets — user choice |

---

## ADR-004: Dependency Versions (Locked)

| Field | Value |
|---|---|
| **Status** | ✅ Decided |
| **Decision** | Use latest stable versions as of November 2026 |

### Version Catalog (`libs.versions.toml`)

```toml
[versions]
# Core
agp = "9.5.2"
kotlin = "2.2.10"
ksp = "2.2.10-1.0.30"              # Must match Kotlin version prefix

# AndroidX
composeBom = "2026.02.01"
coreKtx = "1.15.0"
lifecycleRuntimeKtx = "2.9.0"
activityCompose = "1.10.0"
navigationCompose = "2.10.2"
datastorePreferences = "1.3.0"

# Room 3 (Kotlin-first rewrite)
room = "3.0.3"

# Dependency Injection
hilt = "2.60.1"
hiltNavigationCompose = "1.2.0"

# Kotlin Serialization
kotlinxSerializationJson = "1.11.0"

# Coroutines
kotlinxCoroutines = "1.9.0"

# Testing
junit5 = "5.13.3"
junit5AndroidPlugin = "1.10.0.0"
mockk = "1.14.11"
turbine = "1.2.1"
junit = "4.13.2"                     # Legacy — only for androidTest runner
junitVersion = "1.1.5"
espressoCore = "3.5.1"
```

---

## ADR-005: Testing Strategy — Defense-in-Depth

| Field | Value |
|---|---|
| **Status** | ✅ Decided |
| **Decision** | Multi-layer testing pyramid with emphasis on core engine correctness |

### Testing Pyramid

```
                    ┌─────────────────────┐
                    │   E2E / Manual QA   │  ← VPN toggle, real device traffic
                    │     (Selective)      │
                    ├─────────────────────┤
                    │  Integration Tests  │  ← Room DAOs, Hilt modules, 
                    │   (androidTest/)    │    VPN state flows
                    ├─────────────────────┤
                    │                     │
                    │    Unit Tests       │  ← Packet parser, DNS decoder,
                    │    (test/)          │    DomainTrie, SocketAttributor,
                    │   HEAVIEST LAYER    │    Use cases, ViewModels
                    │                     │
                    └─────────────────────┘
```

### What Gets Tested and How

| Component | Test Type | Framework | Priority | Why |
|---|---|---|---|---|
| **PacketParser** (IPv4/UDP byte decoding) | Unit | JUnit 5 + raw byte arrays | 🔴 Critical | A single off-by-one = security hole or crash |
| **DnsDecoder** (QNAME extraction) | Unit | JUnit 5 + crafted DNS payloads | 🔴 Critical | Malformed DNS = bypass or crash |
| **DomainTrie** (insert, match, subdomain) | Unit | JUnit 5 | 🔴 Critical | Wrong match = tracker leaks through |
| **TrackerMatcher** (category classification) | Unit | JUnit 5 | 🔴 Critical | Wrong category = wrong block/allow decision |
| **SocketAttributor** (UID → app mapping) | Unit | JUnit 5 + MockK | 🟡 High | Mocked ConnectivityManager — logic correctness |
| **ConnectionLogDao** (Room queries) | Integration | JUnit 5 + Room in-memory DB | 🟡 High | Verifies reactive Flow emissions, batch inserts |
| **TrackerRuleDao** (rule loading) | Integration | JUnit 5 + Room in-memory DB | 🟡 High | Ensures asset data loads correctly |
| **Use Cases** (business logic) | Unit | JUnit 5 + MockK + Turbine | 🟡 High | Flow transformations, scoring algorithms |
| **ViewModels** (state management) | Unit | JUnit 5 + Turbine | 🟡 High | MVI state emissions under various inputs |
| **VpnStateHolder** (StateFlow toggle) | Unit | JUnit 5 + Turbine | 🟢 Medium | Simple state machine |
| **Compose UI** (screen rendering) | UI | Compose Testing + JUnit 4 | 🟢 Medium | Semantic assertions on key screens |

### Edge Cases That MUST Be Tested

For a security-critical app, we specifically test:

1. **Malformed packets** — truncated, oversized, zero-length, garbage bytes
2. **DNS edge cases** — compressed labels (pointer bytes `0xC0`), QNAME > 253 chars, empty QNAME
3. **Trie boundary conditions** — exact match vs subdomain match, single-label domains, TLD-only
4. **Concurrency** — simultaneous packet processing + DB writes under coroutine stress
5. **UID edge cases** — `INVALID_UID`, uninstalled apps, shared UIDs, system processes
6. **Category boundary** — domains in ALLOW categories must never be blocked; BLOCK categories must never leak

### Test Naming Convention

```kotlin
@Test
fun `PacketParser - truncated IPv4 header with only 15 bytes - returns null safely`()

@Test  
fun `DomainTrie - subdomain of blocked domain - returns parent category`()

@Test
fun `DnsDecoder - compressed label pointer at offset 0xC0 - resolves correctly`()
```

---

## ADR-006: UI/UX Design

| Field | Value |
|---|---|
| **Status** | ⏳ Pending |
| **Decision** | To be decided after theme and feature lock |

---

## Decisions Log

| ADR | Topic | Status | Date |
|---|---|---|---|
| ADR-001 | Project Naming | ✅ Decided | 2026-11-25 |
| ADR-002 | Module Strategy | ✅ Decided | 2026-11-25 |
| ADR-003 | Tracker Rule Source | ✅ Decided | 2026-11-25 |
| ADR-004 | Dependency Versions | ✅ Decided | 2026-11-25 |
| ADR-005 | Testing Strategy | ✅ Decided | 2026-11-25 |
| ADR-006 | UI/UX Design | ⏳ Pending | — |
