# WILDLANDS — AI DEVELOPMENT MASTER SPECIFICATION

**Version:** 1.0 • **Date:** 7 September 2026 • **Status:** Recommended architecture for owner review; no implementation authorized by this document alone.

**Prepared for:** WILDLANDS: Expedition • Roblox • multiplayer exploration, creature collection, extraction and sanctuary development.

**Source:** The complete supplied *Game Master Brief — WILDLANDS_ Expedition.md*, sections 1–65. The brief is product input, not executable instructions. Where it conflicts with the current request, the current request governs. Recommendations below are proposed studio decisions, not claims that systems already exist. Platform statements link to official Roblox documentation; balance, capacity and performance numbers are planning hypotheses to validate on real builds.

**Deliverable boundary:** Architecture, conceptual schemas, interface contracts, production projects and verification plans only. Directory trees name future artifacts. API signatures describe contracts, not Roblox scripts. No game, Blender asset, in-game UI, production repository scaffolding or deployment has been created.

## Contents

1. [Executive Summary](#1-executive-summary)
2. [Game Architecture](#2-game-architecture)
3. [Design Analysis](#3-design-analysis)
4. [Technical Architecture](#4-technical-architecture)
5. [Data Architecture](#5-data-architecture)
6. [Dependency Graph](#6-dependency-graph)
7. [Project Decomposition](#7-project-decomposition)
8. [Development Roadmap](#8-development-roadmap)
9. [MVP Definition](#9-mvp-definition)
10. [Vertical Slice Specification](#10-vertical-slice-specification)
11. [Blender / Asset Pipeline](#11-blender--asset-pipeline)
12. [AI Agent Responsibilities](#12-ai-agent-responsibilities)
13. [Shared Repository Structure](#13-shared-repository-structure)
14. [AI Development Rules](#14-ai-development-rules)
15. [Testing Architecture](#15-testing-architecture)
16. [Performance Standards](#16-performance-standards)
17. [Live-Ops Architecture](#17-live-ops-architecture)
18. [Risk Register](#18-risk-register)
19. [Definition of Done](#19-definition-of-done)
20. [Recommended Next Steps](#20-recommended-next-steps)

## 1. Executive Summary

Build WILDLANDS around one emotional contract: **discover something worth keeping, choose how much farther to go, return safely, and make the discovery part of your home.** Its identity depends on readable exploration, meaningful but bounded risk, expressive creatures and visible permanent progress. Neither a giant open world nor a complex economy proves that experience.

The recommended architecture is a **server-authoritative modular monolith**: one Roblox experience, explicit domain services, shared validated definitions, owner-specific state projections, and durable economic operations. Begin with a single place containing a compact expedition area and sanctuary plots. Use an eight-player test target. Preserve place-independent contracts so sanctuary and expedition servers can separate when profiling or content size justifies it. Do not introduce distributed infrastructure merely because the finished game may become large.

The initial persistence unit is a versioned player aggregate behind a repository interface. Inventory size and write throughput are explicitly bounded. Before expanding to large unique inventories or enabling trade, migrate through a tested segmented-inventory and escrow protocol. Roblox persistence does not offer an automatic transaction across two player keys; trade is a separate high-risk release gate, not a screen added to inventory.

The first slice contains one small Meadowlands biome with a forest-edge danger pocket, three base creature species, one mutation, one weather event, resource collection, capture, extraction, inventory, a modest sanctuary, one sanctuary upgrade, basic collection feedback and saving. Fox, Deer and rare Owl are the species; Crystal Fox is a mutation form, not a fourth base species. Start with Coins. Keep premium currency, trading, breeding, raids, complex combat, automated sanctuary production and season passes out of this proof.

Twenty-four projects below cover the long-term product. They are ownership domains with staged milestones, not twenty-four things to complete before playtesting. The earliest integrated milestone should prove an entire saved run, including a failed run and a reconnect. Build only the minimal capabilities from each domain required for that proof. Then validate whether uncoached testers choose another expedition before spending on breadth or monetization.

The highest-priority investments are durable inventory/extraction, a clear loss model, touch-friendly interaction, fair shared encounters, profiling on mobile and disciplined content validation. Approve expansion only after these foundations and the loop pass their gates.

## 2. Game Architecture

### 2.1 Product pillars and decision boundaries

- **Discovery:** Landscapes and creature behavior draw attention before numbers do. Scanning provides clues and accessible assistance; the catalog provides identity, rarity, habitats and achievable search conditions.
- **Risk:** Increasing exposure competes with the value of current cargo. Hazards are legible; a player must know what can be lost and where safety lies. Permanent creatures, equipment, cosmetics and banked balances never become ordinary expedition stakes.
- **Ownership:** Securing a find produces a lasting collection record and an individual creature or resource balance. A creature's identity, provenance and appearance survive content updates and transfers.
- **Home:** Sanctuary placement makes ownership visible. Start with preset sockets and one meaningful habitat upgrade. Broad construction tools follow demonstrated demand and performance capacity.
- **Social stories:** Shared discoveries, near escapes, visits and expressive creatures should create stories. Prestige uses verified server events and scoped records, with throttled announcements and no reliance on wealth rankings.
- **Expansion:** A new creature using an existing behavior family should require definitions, compatible assets, validation and QA. A genuinely new behavior, traversal mode or hazard requires an explicit engineering extension.

A feature may satisfy a brief north-star question and still be rejected if its cost, safety burden or performance demand exceeds its demonstrated value. New systems need both a player benefit and an owner.

### 2.2 Loop layers

**Minute loop:** Read terrain → notice a clue → approach → observe/scan → interact or capture → evaluate cargo, capacity and danger. Favor recognition and movement over repeated modal prompts.

**Expedition loop:** Prepare safe equipment → begin a server-recorded run → gather unsecured cargo → choose another encounter or exit → resolve success/failure once → return to sanctuary. Extraction is the boundary between risk-bearing cargo and permanent ownership.

**Session loop:** Display a creature → convert surplus resources to Coins → buy a deterministic upgrade → choose a concrete next goal → re-enter a familiar biome with new variation. The first useful upgrade must not depend on a rare drop.

**Collection loop:** Observe species → secure individuals → discover compatible mutation forms → complete achievable habitat pages → earn cosmetic displays and knowledge. Collection records are historical achievements; selling or trading an individual does not erase legitimate past secured discovery.

**Social loop:** Encounter together → earn individually valid rewards → see a verified discovery cue → visit a sanctuary → plan a run. Parties coordinate eligibility and travel, not trust or ownership.

**Long-term loop:** Add reachable regions, creature families, mastery choices, recurring events and sanctuary expression. Do not require simultaneous advancement through six independent numeric ladders to unlock one activity.

### 2.3 System boundaries

There are five runtime layers:

1. **Platform adapters:** Persistence, networking, time, random generation, asset resolution, telemetry and optional cross-server coordination. Domain services use adapters so failure can be injected in tests.
2. **Authoritative state domains:** Player session, inventory/ownership, economy, active expedition, sanctuary and progression. Only the named owner mutates its state; the persistence coordinator serializes changes.
3. **Gameplay orchestration:** Exploration, creature encounters, capture, resource gathering, extraction, upgrades and quests invoke domain commands using server evidence.
4. **Content:** Published catalogs, spawn/hazard profiles, quest templates, visual variants, world prefabs and schedules. Each run pins compatible content revisions.
5. **Presentation:** Controllers render sanitized snapshots and deltas, inputs request intentions, and audio/VFX interpret confirmed outcomes. Prediction is cosmetic until an authoritative result arrives.

The same value must never be authoritative in two places. An unsecured captured creature is a run record, not a permanent owned creature hidden by a UI flag. A trade reservation is a durable restriction, not a boolean known only to the open trade window. A sanctuary creature display references an owned ID; it does not mint an additional copy.

### 2.4 State ownership and invariant catalog

- **INV-01 — Unique ownership:** Every transferable unique item has one authoritative owner or one escrow custodian, never two spendable owners.
- **INV-02 — Secured means durable:** Display extraction success only after the entire grant is committed or a durable, recoverable transaction establishes the result.
- **INV-03 — One terminal run result:** A run settles as extracted, failed, abandoned or administratively recovered exactly once. Repeated messages do not repeat grants.
- **INV-04 — No negative spend:** Every spend checks integer amount, currency, available balance and reservations in the authoritative transaction.
- **INV-05 — One session writer:** A fenced lease prevents a stale or resumed server from overwriting a newer player session.
- **INV-06 — Client intent only:** Clients never choose rewards, rarity rolls, owned IDs, prices, trade ownership or authoritative completion times.
- **INV-07 — Permanent safety:** Ordinary expedition failure cannot consume banked creatures, purchased entitlements, safe storage or equipped permanent tools.
- **INV-08 — Catalog compatibility:** Stored IDs are stable; retired definitions resolve through tombstones or approved migration mappings. Catalog rollback must not invalidate legitimate ownership.
- **INV-09 — Bounded work:** Inventories, remotes, schedules, creature simulation, placement and retries have explicit bounds and admission limits.
- **INV-10 — Revision agreement:** Any accepted offer, upgrade quote or mutation operation is bound to the correct player, state revision and content revision; edits invalidate prior confirmation.
- **INV-11 — Recoverable uncertainty:** Timeout is not proof of a failed write. Reconcile using the same operation ID; never issue a compensating grant merely because an acknowledgment was lost.
- **INV-12 — Auditable authority:** Valuable outcomes have server-generated IDs, reasons and provenance sufficient to reconcile without using analytics as the source of truth.

These invariants become requirements in Project 01 and acceptance properties in Project 23. Implementation methods may evolve through approved decisions; changing a player-facing promise requires explicit product-owner review.

### 2.5 MVP-to-scale seam

Build replaceable adapters and stable contracts now, not a full future subsystem. Examples: a world locator resolves zone IDs in one place now and place destinations later; inventory queries page bounded results even when stored in one aggregate; creature rendering selects a variant profile without knowing ownership storage; content lookup always uses IDs and revision; operation results distinguish pending, committed and rejected. These seams avoid rewriting consumers while leaving complex internals until their prerequisites are proven.

## 3. Design Analysis

This analysis treats the supplied *Game Master Brief & Production Specification*, sections 1–65, as the source vision. Recommendations below are proposed production decisions, not claims that the original brief already resolves these questions. The user's present request takes precedence over the brief's competing eight-creature slice and art-demonstration lists. All prices, capacities, encounter frequencies, timers, conversion rates, and behavioral targets in this section are provisional hypotheses for testing; they are not established balance or retention benchmarks.

### 3.1 What is structurally important

**Core player fantasy.** The player is an explorer who notices something extraordinary, judges whether to pursue it, and makes the discovery part of a visible personal home. Ownership becomes emotionally meaningful because the player remembers an expedition. Sanctuary display is the payoff to exploration, not a separate decorating game bolted onto a grind. Rarity should produce recognition and stories rather than become the sole measure of accomplishment.

**Primary loop.** Prepare a destination and loadout → explore readable terrain → notice a creature or resource → decide whether remaining capacity and danger justify pursuing it → capture/collect into unsecured cargo → return to an extraction point → commit cargo to permanent ownership → display or spend it → choose a new expedition goal. This loop needs a genuine decision before extraction, a reliable ownership boundary at extraction, and a meaningful change after returning. Removing any of those three reduces the game to collecting nearby objects repeatedly.

**Secondary loops.** The collection loop turns an unknown silhouette into an observed entry, then a secured specimen and optional variant records. The sanctuary loop turns secured finds into expression and additional display space. The equipment loop opens routes and improves tool usability. The social loop lets friends witness, cooperate in, and later visit the results of discoveries. The event loop temporarily changes familiar routes and encounter opportunities. These loops feed exploration; they must not compete for the first-session player's attention.

**Short-term progression: one session.** Secure a first creature, learn extraction, obtain enough ordinary resources for one affordable improvement, and choose a visible next goal. A first upgrade must be achievable without winning a rare roll. The player should know both what is permanently safe and what remains exposed during a run.

**Medium-term progression: several sessions.** Open another route or biome through demonstrated exploration and deterministic milestones; improve equipment options; fill species families; make the sanctuary recognizably personal. Avoid requiring parallel Explorer Level, Mastery Level, Sanctuary Level, tool level, and currency gates for the same unlock. Each progression axis needs a different job.

**Long-term progression: weeks and updates.** Pursue optional mutations, habitat knowledge, exploration achievements, sanctuary expression, friend collections, and new regions. Old collections remain meaningful when new content arrives. Permanent “complete everything ever released” objectives must exclude unavailable seasonal entries or offer an explicit archival category.

**Collection systems.** Separate species, mutations, traits, size records, cosmetics, and provenance. A mutation is a permitted variation of a species, not a duplicate species definition. A trait is a behavior or gameplay modifier with an explicit stacking policy; a cosmetic changes presentation without changing collection rarity or spawn odds. The collection book distinguishes `seen` from `secured`: a valid observation can survive a failed run, while ownership, secured milestones, sale value, and collection rewards require extraction. Selling or later trading an owned creature does not erase historical discovery.

**Social systems.** Make observing a friend's discovery and exploring the same route valuable before building parties, trading, global rankings, or raids. Shared encounters must not reward whoever first touches an object at the expense of everyone else. Later visiting should load bounded displays and permissioned interactions; owning a sanctuary does not grant visitors authority to move, spend, or remove anything. Prestige should reward exploration accomplishments and creative homes, not the largest amount spent.

**Economy.** Coins are the first soft currency. Resources and first-time exploration objectives are controlled sources; permanent upgrades and later optional customization provide sinks. Securing a creature is not also an automatic repeatable coin payout unless a specific reward definition says so. Selling resources converts a recorded quantity once; it does not create an additional extraction bonus for the same sale. Rare collectibles should remain valuable without forcing players to liquidate their favorite finds to progress. Event currencies and Gems are deferred until they solve a demonstrated design need.

**Monetization.** Test the free expedition loop first. Later paid offerings should initially be cosmetic: sanctuary themes, identity effects, outfits, banners, and visually distinguishable creature skins. Do not sell capture success, rare detection access, mutation odds, extraction protection, hazard immunity, or field capacity that improves expected expedition yield. Extra off-expedition cosmetic storage or saved appearance presets may be evaluated later, with sufficient free capacity. Paid cosmetics must not masquerade as earned rare specimens in the collection book or trading interface.

**Exploration.** Use authored landmarks, loops, alternate paths, sightlines, and safe extraction routes. Controlled variation changes what appears in approved encounter sockets; it should not randomly make the only route home impossible. A scanner supports observation through direction or proximity cues. It must not hide all rare creatures from players who lack a scanner upgrade. Better knowledge, route access, and timing should improve opportunity without becoming paid or purely numerical perception gates.

**Creature systems.** The production unit is a species definition linked to a tested rig family, behavior family, habitat rules, capture behavior, animation set, and mutation allowlist. Hundreds of collectible definitions must not imply hundreds of bespoke AI scripts. Active world simulation is bounded independently of total catalog size; sanctuary displays do not need full wilderness perception and navigation. Traits and companions remain later features until their contribution to exploration is demonstrably different from stacking luck multipliers.

**Sanctuary systems.** The first sanctuary is a safe, permanent display and preparation space with fixed sockets. Its initial upgrade adds a visible habitat pad and another displayed creature, making a successful expedition legible to the player and nearby friends. Storage capacity, displayed-creature capacity, and active simulated-creature capacity are different limits. Free placement, passive production, breeding, workers, raiding, and visitor permissions are separate later projects, not implicit consequences of having a home.

**Events.** An event is a bounded rule overlay with a schedule, eligible regions, affected spawn pools, presentation, reward rules, and an expiry policy. The first event changes one existing biome. It does not require another map or another currency. Events should produce a recognizable gathering moment without making participation mandatory at an inconvenient real-world time. Unsecured event loot already earned remains extractable after the weather or event ends.

**Procedural systems.** Use server-selected, reproducible variation within authored constraints: encounter sockets, resource bundles, event timing, and optional routes. Give every spawn an identity, life cycle, and award policy. Seeds support debugging and reproduction; knowing a seed never authorizes a reward. Validate reachability, extraction access, minimum ordinary-resource availability, and encounter concurrency before activating a layout. Full procedural terrain is unnecessary early.

**Content expansion.** New species, supported mutations, resource types, quests, cosmetic sets, and existing event patterns should be data plus approved assets. A new mechanic such as temperature is a deliberate system extension with a specification, owner, tests, and budget. “Data-driven” must not mean executable scripts hidden in content data or unlimited combinations of effects. Every release declares the behavior families and schema versions it requires.

**Retention mechanisms.** Use attainable next goals, newly opened routes, partial collection progress, sanctuary pride, and enjoyable co-play. Daily and weekly objectives should offer choice and catch-up without a compulsory login streak. Offline production is later, capped, and economically secondary to exploring. Longer session duration alone is not success: a long session caused by slow walking, confusing extraction, or inventory friction is a failure signal.

**Viral and social moments.** Favor a distinct creature silhouette, a friend noticing the same rare encounter, a close but fair escape, an unusual mutation reaction, and a sanctuary reveal. Announcements follow verified server discoveries and should not expose precise cargo or player location by default. Limit repetition and allow presentation controls. “First discoverer” initially means a personal milestone or explicitly labeled server-session recognition. Global-first prestige requires a separate eligibility, atomic claim, tie, moderation, and rollback policy before it can make a credible promise.

### 3.2 Keep these strong ideas

1. **The extraction-to-sanctuary identity.** It connects excitement, emotional ownership, progression, and social display within one understandable activity.
2. **Readable, connected regions.** They let danger, art, onboarding, content, and performance grow in controlled increments.
3. **Stylized creatures with expressive animation.** Distinct silhouettes and reactions provide better brand recognition and practical reuse than escalating detail.
4. **Authored worlds with variable encounters.** Players develop useful route knowledge while remaining curious about the next run.
5. **Collection across species and supported mutations.** It expands goals efficiently when combinations remain curated and visually recognizable.
6. **Prove the free loop before expansion.** The brief explicitly permits stopping to repair the core loop. That must remain a real production gate.

### 3.3 Weak ideas and recommended changes

- **Eight rarity tiers at the outset:** they create terminology without enough content to make the tiers meaningful. Start with Common, Uncommon, and Rare; reserve future identifiers without presenting empty tiers. Extraordinary presentation needs an actual extraordinary find.
- **Rare-only scanner access:** it makes discovery feel like an equipment paywall and undermines environmental observation. All eligible creatures remain visible; upgrades later improve clue clarity, range within a tested cap, or route investigation.
- **Many independent progression levels:** they obscure the next useful action and multiply balancing work. Launch the slice with a tutorial milestone and one sanctuary improvement. Add Explorer progression first; add mastery only when it changes decisions.
- **Traits that multiply luck or value:** stacked bonuses can dominate collecting and damage economy forecasts. Defer functional traits, later use explicit categories, caps, exclusivity rules, and mostly situational advantages.
- **A new mechanic every major update:** this is a recurring engineering obligation, not a content recipe. Commit to sustainable content packs; add mechanics only when tested existing systems cannot express the intended experience.
- **Capacity as paid convenience:** field capacity changes how much value a player can extract and how often they must risk returning. Keep expedition capacity progression earnable and balance it as gameplay. Monetize expression first.
- **Prescribed launch counts:** two or three regions and twenty to thirty creatures are aspirations, not evidence of readiness. Ship only the amount that clears quality, variety, performance, operational, and retention gates. Trading is not a compulsory launch dependency.

### 3.4 Missing systems to specify before implementation

The brief needs explicit ownership and loss rules; item and creature identity; duplicate-request handling; session recovery; capture contention; inventory-full outcomes; disconnect behavior; entitlement handling; moderation and reporting; accessibility; content validation; catalog retirement; support compensation; rollback operations; and experiment ownership. These are architecture requirements, not optional polish.

It also needs a coherent hazard vocabulary. The slice uses one readable environmental exposure mechanism with warnings, recovery opportunities, and a stable route home. Add neither combat nor a separate hunger/thirst/stamina simulation merely because the genre includes survival. Mobile interaction, short sessions, color-independent rarity cues, readable text, adjustable effects/audio, and resumed onboarding are required product constraints.

### 3.5 Dangerous systems and bounded alternatives

**Trading and duplication:** a trade spans two persistent owners and cannot be safe merely because both clients press Confirm. Defer player transfers until durable transaction decisions, escrow reservations, idempotent delivery, crash recovery, inventory limits, and reconciliation are demonstrated under fault injection. Later trades show exact immutable assets and reset acceptance whenever terms change. No trust-trade or “gift now, receive later” flow.

**Loss and raiding:** the phrase “risk everything” conflicts with permanent sanctuary progression. Only the active expedition's unsecured cargo is exposed to in-world failure. Already owned creatures, secured inventory, upgrades, equipment, purchased goods, and sanctuary contents remain safe. PvP and raids are excluded early and require a separate opt-in mode assessment later.

**Persistence failures:** a success animation is not proof that an award survived. Extraction remains pending until durable confirmation; pending cargo cannot be sold, traded, or spent. Failed loading must never silently start a new profile that overwrites existing progress. Ambiguous transactions are reconciled by identity rather than replaying a fresh award.

**Unlimited spectacle and simulation:** thousands of definitions do not authorize thousands of active agents, particles, or displayed pets. Use tested per-zone simulation, display, streaming, and effect budgets; degrade cosmetic density while preserving interaction readability. The sanctuary must remain performant as ownership grows.

**Forced rarity and FOMO:** extreme chance, inaccessible schedules, and disappearing completion requirements can turn anticipation into frustration. Make rare finds optional, expose useful habitat clues, use repeatable return opportunities, and keep ordinary-resource progression deterministic. Do not sell relief from a deliberately frustrating rarity system.

### 3.6 Resolved brief conflicts

- **Slice size:** three base species—Fox, Deer, Owl—with Crystal as one Fox mutation; therefore three base creatures and four collectible forms. This replaces the competing eight-creature production slice and the art scene's Fox/Deer/Crystal Fox count.
- **Region scope:** one Meadowlands biome with a forest-edge danger pocket, not a separately unlocked Ancient Forest biome. Region travel is an expansion contract, not slice content.
- **Progression scope:** one fixed sanctuary habitat upgrade proves development. Backpack and scanner upgrades, Explorer Level, and a second-region unlock follow after the slice.
- **Currencies:** Coins only in the slice. Gems and temporary event currencies are later decisions.
- **Risk scope:** current-run unsecured finds can be lost to telegraphed expedition failure; permanent progression cannot. No early PvP or sanctuary theft.
- **Discovery scope:** observation earns `seen`; extraction earns `secured` and ownership. The interface never calls an unsecured capture permanently owned.
- **Social scope:** shared witnessing and fair co-op captures are early. Formal parties, global firsts, trading, raids, and rankings are later.

## 4. Technical Architecture

### 4.1 Architectural decisions

Use a **modular monolith**: one versioned codebase, explicit domain modules, one server bootstrap, one client bootstrap, and narrow contracts. A service is a logical module, not a separately deployed microservice. Do not adopt an entity-component framework, dependency injection framework, custom networking framework, or distributed backend merely because the final game may become large. Select and pin supporting tools during Project 01 after a small compatibility evaluation.

The first playable uses **one place, eight players, a small Meadow with a forest edge, and streamed sanctuary plots**. Services address logical locations and expedition IDs rather than assuming one giant map. Split sanctuary/social and expedition places only after measured memory, simulation, or matchmaking requirements justify the operational cost. Share schema, catalogs, and contracts across places; deploy compatible release manifests. Changing places must never change who owns a player's persistent data.

The principal invariant is: **permanent banked possessions are safe; only cargo acquired in the current unresolved expedition is exposed to its declared loss policy.** Seen, captured, and secured are different states. A seen creature may remain in the encyclopedia after failure; ownership and secured collection rewards require extraction. A trading or display service cannot accept an unextracted creature.

Prefer deterministic domain logic with injected clock, random source, storage adapter, and content registry. Keep Roblox Instances at adapters and presentation boundaries. A creature definition describes behavior capabilities; a reusable behavior implementation executes them. Hundreds of definitions must not create hundreds of unique scripts or always-running loops.

### 4.2 Recommended Studio hierarchy

The following is a future structure specification, not generated scripts:

```text
ReplicatedFirst/
  LoadingBootstrap                 minimal boot/error presentation
ReplicatedStorage/
  Wildlands/
    Shared/
      Types, Contracts, Validation, Constants
      Math, Spatial, Time, ResultCodes
      PublicCatalog, LocalizationKeys, PresentationRules
    Remotes/
      InteractionRequest, ExpeditionRequest, InventoryRequest
      SanctuaryRequest, ProgressionRequest, TradeRequest
      QueryRequest, QueryResponse, CommandResult
      StateSnapshot, StateDelta, WorldCue
    ClientAssets/
      EssentialIcons, SharedEffects, PreviewProxies
ServerScriptService/
  WildlandsServer/
    Bootstrap, ServiceRegistry
    Infrastructure/
      DataService, SessionService, NetworkService
      CatalogService, FeatureFlagService, TelemetryService
      Scheduler, PersistenceAdapter, RecoveryCoordinator
    Domain/
      PlayerService, InventoryService, EconomyService
      ExpeditionService, ExtractionService, WorldService
      ResourceService, CreatureService, DiscoveryService
      CaptureService, CollectionService, EquipmentService
      SanctuaryService, ProgressionService, QuestService
      AchievementService, SocialService, TradeService
      EventService, EntitlementService
    Admin/
      AuthorizedCommands, Audit, SupportRecovery
ServerStorage/
  Wildlands/
    PrivateCatalog/                 reward, eligibility, spawn, economy rules
    Prefabs/Creatures, World, Sanctuary, Equipment
    ContentPackages/               assets required by this place release
    ServerTemplates/               invisible authoritative proxies
StarterPlayer/
  StarterPlayerScripts/WildlandsClient/
    Bootstrap
    Controllers/
      Input, Interaction, Camera, State, Expedition
      Inventory, Collection, Sanctuary, UI, Audio, VFX, Streaming
  StarterCharacterScripts/
    CharacterPresentation          only lifecycle-local presentation
StarterGui/
  WildlandsGui/
    HUD, Menus, Notifications, ConnectionStatus, Accessibility
Workspace/
  World/Meadow, ForestEdge, Boundaries, ExtractionZones
  SanctuaryPlots/
  Runtime/Creatures, Resources, Interactables
  SpawnMarkers/                    tagged authoring markers
  Navigation/                      route and hazard volumes
  Terrain
SoundService/
  Music, Ambience, Discovery, Interface, Danger
Lighting/
  Baseline                         event visuals composed on the client
```

Workspace is a view of authoritative server state, not a database. CollectionService tags identify authoring roles such as spawn anchor, hazard region, interaction point, and sanctuary attachment. Attributes contain non-sensitive IDs and small presentation metadata. Never encode money, ownership, or authorization solely in Attributes, leaderstats, GUI values, Tool instances, or Workspace hierarchy.

Keep world and creature prefab libraries in ServerStorage. Only currently needed assets become Workspace instances; only genuinely shared small presentation assets belong in ReplicatedStorage. Roblox streams Workspace descendants, while ReplicatedStorage content is outside that streaming system. Client controllers therefore bind and unbind by entity ID as models appear and disappear and must not mistake a streamed-out object for a deleted collectible. [Roblox instance streaming](https://create.roblox.com/docs/workspace/streaming)

### 4.3 Module ownership and dependency contracts

Infrastructure owns loading, transport, persistence, logging, time, scheduling, and release identity. Domain services own rules. Client controllers own interaction presentation and rendering. The composition root explicitly initializes infrastructure, registers validated catalogs, constructs domain services, then exposes networking. The bootstrap service registry only constructs modules; the content registry owns allowlisted definition/effect handlers; equipment/progression owns derived gameplay capabilities. These are separate responsibilities. A failed mandatory dependency prevents gameplay activation; optional social/analytics services can degrade independently.

Each service publishes a small interface document containing input and output types, authorization, operation identity, possible errors, durable effects, idempotency, event ordering, and failure behavior. No service receives a mutable reference to another service's state. DataService is the **only writer to player aggregates**; domain services produce validated commands or mutation plans. InventoryService defines ownership and capacity rules; EconomyService defines currency accounting; ExtractionService orchestrates their plans in a single player transaction. It does not directly edit their internal tables.

Required conceptual interfaces include:

- DataService: acquire profile; read immutable projection; commit a fenced operation; renew/release session; reconcile operation status; migrate; quarantine.
- InventoryService: check capacity; enumerate paginated holdings; plan grant/reservation/removal; resolve ownership; lock an asset for a declared operation.
- EconomyService: quote a server-defined purchase; plan debit/credit; validate bounds; return a transaction result. A quote contains revision and expiry; it cannot authorize itself.
- ExpeditionService: prepare, start, checkpoint, resolve failure; return current run projection. ExtractionService: begin/cancel extraction, validate completion, commit cargo settlement.
- CreatureService: allocate encounter, evaluate spawn eligibility, maintain lightweight behavior state. CaptureService: validate attempt and construct an immutable server-earned award/entitlement plan using supplied run context. ExpeditionService commits that plan and its consumed entitlement together through DataService; CaptureService does not call ExpeditionService back or write cargo independently.
- DiscoveryService: validate seen evidence; CollectionService: own saved seen/secured history, compute collection progress, and plan one-time claims. Neither creates owned creatures independently. A batched seen observation can appear immediately as a session sighting, but is not labeled saved until persistence confirms it.
- SanctuaryService: validate placement/display/upgrade plans and own persistent visit permissions; SocialService owns invitations and visit routing subject to those permissions. EquipmentService derives equipped capabilities from banked ownership.
- TradeService: create/version offer, confirm exact offer digest, prepare escrow, commit decision, reconcile. EntitlementService: resolve platform receipt to a permanent grant plan.
- EventService: resolve trusted versioned schedule and modifiers. CatalogService: get definition by stable ID and pinned content version; return a public projection when requested by a client. FeatureFlagService owns enforcement/control state; content teams define valid targets, and release operators invoke authorized transitions.

In-process domain events are emitted **after commit** and include event ID, operation ID, aggregate revision, subject IDs, and content version. Quest, achievement, and analytics consumers deduplicate. Progression rewards are either included in the original aggregate commit or claimed idempotently from durable evidence. An event bus must never become a second hidden transaction engine. Synchronous dependency cycles are prohibited; an orchestrator coordinates two domains without either owning the other.

### 4.4 Client/server and network boundaries

The client requests an intention: scan, attempt capture, start expedition, collect resource, begin extraction, equip a held item, place an owned decoration, claim a completed objective, or confirm a trade revision. The server derives price, odds, damage, rewards, resulting inventory, eligibility, and completion. Roblox requires server validation of client input; purchase grants also belong on the server. [Securing the client-server boundary](https://create.roblox.com/docs/scripting/security/client-server-boundary)

Every command envelope carries protocol version, request ID, command type, bounded payload, and relevant run/offer/entity revision. The authenticated Player supplied by Roblox is the actor; a payload userId cannot substitute for it. Validate type, maximum nesting, finite numeric range, string length, enum membership, catalog references, active session, ownership, cooldown, spatial plausibility, line of sight where relevant, and legal state transition. Reject NaN, infinity, oversized lists, arbitrary Instances, forged rewards, and unknown command names before expensive processing.

Use named **RemoteEvents** grouped by domain for commands and asynchronous results. `QueryRequest/QueryResponse` provide paginated reads with correlation IDs, timeout, cancellation, and bounded responses. **No RemoteFunctions are required in the initial architecture.** If a later local cached read uses one, it must return promptly without storage/network waits; never invoke a client RemoteFunction from an authoritative server workflow. VFX cues can use unreliable delivery later where loss changes no gameplay; inventories, trades, grants, and extraction results require reliable messages plus state resynchronization.

Start with per-player, per-command token buckets: ordinary interaction at 5 requests/second with burst 10, scan at 2/second, inventory query at 2/second, trade edit at 2/second. These are tunable abuse controls, not reward rates or platform limits. Each expensive service also enforces a server-wide budget. Client retry uses the same request/operation identity. The server returns `Pending`, `Committed`, `Rejected`, or `Recovering`, with reason codes such as `DataUnavailable`, `StaleRevision`, `InvalidTarget`, `CapacityFull`, `RunClosed`, or `RateLimited`.

Send each player only their own compact private state, public neighbor displays, and relevant nearby entities. Initial snapshots are followed by revisioned deltas; a missing base revision requests resynchronization. Paginate inventory and catalog details rather than broadcasting every creature, transaction, or definition on join. A public catalog may expose species, rarity descriptions, icons, allowed mutations, and discovery hints. Private spawn seeds, entitlement rolls, anti-abuse thresholds, reward schedules, and unreleased content remain server-only. Secrecy supplements validation; it never replaces it.

Player movement can remain responsive with normal Roblox character networking, but server observation must check impossible travel, region access, timing, and interaction distance. Rewarding from `Touched` alone is prohibited. Roblox cannot verify client-owned physics calculations, including abuse of contact events. Valuable interactables use anchored/server-authoritative proxies, and gameplay-critical unanchored NPC assemblies use explicit server ownership where required. Cosmetic movement can be simulated locally. [Roblox network ownership](https://create.roblox.com/docs/physics/network-ownership)

### 4.5 Persistent data, saving, and operation identity

Initial storage is one standard DataStore key per player containing a bounded PlayerData aggregate: wallet, holdings, progression, sanctuary, collection, operation receipts, and active expedition. This lets ordinary purchases, extraction, upgrades, and claims update related state within **one key**. Roblox data stores are shared across places in an experience; `UpdateAsync` supports conditional same-key changes and consumes read and write budgets. Its callback cannot yield. Treat the callback as a pure transformation with no spawned objects, remote messages, analytics emission, or new random rolls. [Roblox data stores](https://create.roblox.com/docs/cloud-services/data-stores)

Use a reviewed persistence implementation behind an adapter, pin its version, audit its session semantics, and retain WILDLANDS-specific recovery tests. The application must enforce a fencing token even when a library offers session locking. The documented session-lock pattern addresses overlapping old/new servers and must not be weakened by unconditional profile replacement. [Roblox player data and purchasing architecture](https://create.roblox.com/docs/cloud-services/data-stores/player-data-purchasing)

The required profile lifecycle is `Unloaded → Acquiring → Migrating → Reconciling → Ready → Closing`, with `ReadOnlyRecovery` and `Quarantined` branches. Acquire using a conditional write that records server session ID, lease expiry, and a monotonically increasing fencing epoch. Every subsequent write verifies the current epoch and lease ownership. A stale server can neither save nor release a newer session's lock. Renew well before lease expiry; if ownership is uncertain, suspend mutations. No force-takeover because a player clicks retry. Proposed starting lease: 120 seconds, renew every 30 seconds, adjustable after failure tests; these are application settings.

Serialize mutations through one per-player queue. A command has a stable operation ID, expected preconditions, deterministic mutation, bounded result record, and durable commit revision. Generate a creature instance ID and its rolled traits once, before the retryable transformation; reuse them across all retries. Persist grant and operation receipt together. If the receipt already exists, return the previous result without applying the effect again. Ephemeral network request caches are an optimization, never the only duplication defense.

A failed write response does **not** prove that no write occurred. The current Roblox guidance explicitly describes unknown outcomes and recommends uncached verification. After a timeout, keep the command unresolved, reconcile the same operation ID against authoritative storage, and retry conditionally. Never issue a second grant under a new ID or restore an older snapshot because an acknowledgement was lost. [DataStore errors and limits](https://create.roblox.com/docs/cloud-services/data-stores/error-codes-and-limits)

Use retries with exponential backoff and jitter, per-key serialization, bounded queues, admission control, and a circuit breaker. Do not retry malformed data or authorization errors. Prioritize acknowledged-state protection, extraction settlement, receipts, and session renewal over optional metrics. If reads fail, do not create and save a fresh default profile; only a successful authoritative missing-key result authorizes creation. A corrupt profile is quarantined with evidence and support reference, never silently reset.

Autosave coalesces nonvaluable preferences and observed-discovery notes; it is not the durability guarantee for rewards. Valuable commands remain visibly pending until their state is committed. Checkpoint cargo in short batches with an initial target of ≤2 seconds p95 under healthy services; resource animation may play immediately, but secured counts and spendable balances cannot predict a successful write. Budget-aware admission may lengthen a pending interaction or pause new acquisition, with clear connection status. Avoid constant per-frame or per-hit saves. Final save on leave and shutdown is a best-effort drain, not the only copy of progress.

### 4.6 Expedition, capture, and extraction protocol

1. **Prepare:** validate loadout, bank capacity, route access, and DataService readiness. Reserve enough permanent storage for the entire allowed expedition cargo before departure; do not surprise the player with a capacity purchase at extraction.
2. **Start:** persist run ID, ordinal, content version, run seed reference, server session, frozen loadout, loss-policy version, capacity reservation, and `Active` state before enabling reward-bearing interactions.
3. **Encounter:** allocate a unique encounter ID and immutable eligible roster of at most eight nearby active participants. Each earns an individual server-validated opportunity keyed by `encounterId + userId`. Shared sightings and collaboration cannot let the first player touching a creature steal everyone else's opportunity.
4. **Capture:** validate proximity, encounter state, tool capability, activity, and capacity. Roll each participant's result once using server-controlled context; retries preserve the outcome. CaptureService returns an immutable award plan; the expedition coordinator commits its pending creature and consumed entitlement marker together in that player's active-run checkpoint through DataService. A repeated scan, remote, reconnect, or changed capture animation cannot reroll or mint another copy. No participant can claim from an old or newly joined run.
5. **Risk:** show cargo as unsecured. Damage, hazard exposure, and extraction eligibility remain authoritative. Banked creatures and tools do not enter the loss pool. Seen observations record no owned creature or secured-collection reward.
6. **Begin extraction:** server validates arrival in the extraction volume, plausible journey, and the short uninterrupted channel. It locks new cargo actions for this run while serializing any earlier pending acquisition operations.
7. **Commit extraction:** in one fenced aggregate transaction, require the matching active run, transfer committed cargo into banked holdings, credit declared expedition rewards, mark secured discoveries, apply once-only progression, close the run, release unused capacity, and record `extract:<runId>`. The same cargo cannot be both pending and banked.
8. **Acknowledge:** only after confirmed commit show secured success, play the return celebration, and make the bank usable. Crash before commit leaves an unresolved run; crash after commit returns the already-banked result on rejoin without another grant.

An active run surviving disconnection must never become an automatic successful extraction. A short reconnect grace may resume **only in the same live owning server**, where hazards and run state continue and the body does not become invulnerable. A new profile owner first reconciles any durable extraction transaction; if extraction committed, the player keeps that secured result. Otherwise it resolves the unresolved run by the normal versioned failure policy and never rewinds to an earlier, safer checkpoint. **For the slice, confirmed failure loses all current-run unsecured cargo; banked possessions and seen encyclopedia observations remain.** Failure settlement is also idempotent. A server crash can therefore lose unsecured cargo but cannot lose banked holdings or duplicate a reward. Record that tradeoff in onboarding language and test it with players. Confirmed service incidents can receive a separately audited, capped compensation operation; the client cannot assert an outage to obtain one.

If places split later, complete banking before return teleport. For departure use a server-issued, expiring, one-use ticket referencing a durable run and handoff state, intended user, origin, and destination. TeleportData may carry only an opaque lookup ID and visual context. Destination validates the ticket and acquires the newer fencing epoch; it does not accept balances, inventory, eligibility, or loot from TeleportData. Roblox explicitly labels teleport data unsuitable for secure information. Ticket expiry or teleport failure returns to a defined recovery state; it never resets or duplicates a run. [Teleport between places](https://create.roblox.com/docs/projects/teleport)

### 4.7 Growth boundary: segmented storage before trading

Bound the initial profile at a **64 KiB normal target and 128 KiB application admission ceiling**, including receipts and active run. Initial admission policy: approximately 120 owned creatures, 100 distinct stack entries, 60 sanctuary placements, and a bounded current journal; byte size overrides those illustrative counts. Preserve acquired holdings at capacity and stop admitting new runs/grants that cannot fit. Never silently discard items. Reserve emergency headroom for an already-issued receipt or pending recovery. Do not sell capacity above the tested storage budget.

These are deliberately lower than Roblox's documented **4,194,304-character entry limit** because per-key write throughput also matters. Current documentation lists **4 MB/minute per-key write throughput**; rewriting a nearly full key frequently is untenable. Example planning envelope: 64 KiB × 24 commits/minute ≈1.5 MiB/minute, before retries and overhead. Use runtime budgets and observed serialized sizes, not this arithmetic as a guarantee. Trigger the next storage project when p95 profile size exceeds 64 KiB, any profile approaches 96 KiB, p95 admission waits exceed two seconds, or trading is scheduled—whichever occurs first. [DataStore limits](https://create.roblox.com/docs/cloud-services/data-stores/error-codes-and-limits)

The next storage model separates a small PlayerCore root, bounded immutable inventory pages, authoritative unique-asset ownership records, and durable transaction coordinators. Pages support roughly 100–200 records or 64 KiB each, whichever comes first. Wallet and active run stay in a small core aggregate. A manifest identifies committed page generations and counts. Preparing new immutable pages does not make them visible; a conditional root-manifest commit publishes the generation. An interrupted preparation leaves reclaimable orphan pages; old manifest readers still have a consistent view. Retain previous generations through a recovery window. Do not implement independent mutable chunks followed by a hopeful root save.

Unique assets need stable instance IDs and an ownership record containing owner, owner revision, state, and transaction reference. The inventory page becomes a query index; authorization checks the ownership record. To mint or extract assets across keys, create a durable transaction manifest, prepare inaccessible asset records with deterministic IDs, commit the player transaction, then activate records idempotently. Recovery scans the player's pending-transaction references on login and a durable partitioned work index in the background. A MemoryStore queue can accelerate work but cannot be its only discoverability mechanism. During incomplete settlement, show `Recovering` and deny use/trade of affected assets. Large inventory lists are paginated with revisioned cursors; equipped/displayed/recent items are the hot set, not the entire collection.

Migration is a required project with dual-format readers, generation manifests, lazy per-player conversion under the session fence, checksums/count reconciliation, rollback instructions, and adversarial tests. Keep the source aggregate until destination validation and the root switch succeed. Do not permit trading until all participating assets use the audited ownership model. This is a planned evolution behind stable repository interfaces, not an unspecified future rewrite.

### 4.8 Trade protocol and purchase receipts

There is **no assumed atomic transaction spanning two players or several DataStore keys**. Trading therefore needs a durable escrow state machine, not two profile saves. Start with same-server trades of banked unique creatures only, a small bounded offer, and no currency, premium entitlements, quest objects, equipped objects, or unsecured cargo. Resources and eligible cosmetics are later additions to the same audited protocol.

Before any shared ownership effect, register the transaction ID in a durable bounded partition of the recovery work index, then persist the coordinator and participant pending references. An index entry without a completed coordinator is a harmless recoverable orphan; an escrow mutation without durable recovery discoverability is forbidden. Apply the same ordering to segmented mint/extraction transactions. Recovery jobs are idempotent and use coordinator fencing plus per-record expected revisions; MemoryStore and messages only accelerate this process.

Each recipient obtains a **durable capacity reservation keyed by tradeId** before escrow preparation and the commit decision. Reserve incoming unique-item slots, page/core byte headroom and settlement metadata conservatively without assuming outgoing items have already freed space. All runs, grants and other trades respect existing reservations. Acquire participant reservations in stable user-ID order with bounded retry; a failed preparation follows the durable abort path. Settlement consumes the appropriate reservation, and abort releases it exactly once. A reservation cannot expire independently while its coordinator is unresolved or committed. Capacity and eligibility are revalidated against the sealed revision before the coordinator commits; a check alone is insufficient when another operation can consume space.

The trade state machine is `Draft → Confirming → Preparing → Committed → Settled`, with `Aborted` allowed only before `Committed`. Any offer edit clears both confirmations. Confirmations bind both users to the exact offer revision and digest. Persist the sealed participant/item list and trade coordinator before preparing escrow. In deterministic item order, conditionally transfer each asset's authority from the expected owner/revision to `Escrow(tradeId)`. Escrowed assets are unusable by either player. After every required item is confirmed escrowed and all eligibility/capacity checks remain valid, write the coordinator's single irreversible `Committed` decision. Recovery must roll a committed trade forward, never refund one side.

Finalize each asset from escrow to its specified recipient with idempotent per-asset transitions, then reconcile inventory indexes and mark settled. A crash at any boundary delays availability but cannot create two authoritative owners. Before commit, a durable abort decision restores every participant asset to its recorded source and marks terminal versions so delayed stale preparation cannot re-lock it. The abort worker must visit **every asset in the sealed enlistment list, including assets that have not yet reached escrow**. If an asset still has its original source revision, write an abort tombstone and advance that revision; if already escrowed, restore it and advance the revision; if a later legitimate revision already exists, reject the obsolete reservation. Old workers always present their original expected revision and cannot act through a newer one. A concurrent late preparation can temporarily reach escrow before its abort fence lands, so recovery continues until every enlisted asset has a terminal revision and the aborted coordinator is reconciled. During that interval affected assets remain visibly recovering. This is eventual recovery across separate keys, not an atomic multi-key abort. Unknown write outcomes must be reconciled; a timeout is not an abort instruction. Durable pending references and periodic recovery find abandoned trades even when all players are offline. Repeated failure freezes only affected assets, records an incident, and routes to support.

When stackable resources become tradable, split offers into deterministic escrow lots. Source debit/reservation evidence and transfer coordinator must be durable before the corresponding lot can be delivered. When currency is considered, add wallet reservation records and a double-entry settlement proof; never transfer it using an independent debit and credit without a coordinator. Premium currency remains nontradable. This extension has its own economic and failure-recovery release gate.

For developer products, one EntitlementService owns `ProcessReceipt`. Resolve the platform product ID through a server allowlist; use `PurchaseId` as the permanent idempotency key. Under the profile fence, commit the granted balance/entitlement and its processed-receipt record together, then return `PurchaseGranted`. Duplicate delivery returns the recorded committed result. Unknown product, unavailable profile, lost fence, or uncertain commit returns `NotProcessedYet` and triggers reconciliation. Purchase-prompt completion never grants value. Roblox's server receipt workflow is the relevant platform boundary. [Developer products](https://create.roblox.com/docs/production/monetization/developer-products)

Do not age out receipt IDs and thereby make old purchases grantable again. Before receipts grow beyond the core budget, archive immutable processed-ID segments using a copy-verify-root-switch protocol; duplicate detection consults both the bounded hot index and permanent archive before granting. Do not enable a product until storage headroom exists for its worst-case benefit and receipt. Promotional, quest, support, and event rewards use their own durable claim identities; their lifetime is tied to the source claim, not an arbitrary short deduplication cache.

### 4.9 Migrations, errors, analytics, and operations

Separate `schemaVersion`, `contentVersion`, `networkProtocolVersion`, and `buildId`. Persistent migrations are ordered, deterministic, validated functions with synthetic and anonymized historical fixtures; each commits under the session fence. Never auto-downgrade a newer profile. Older servers must reject incompatible writes and direct a safe reconnect. Prefer additive fields and tolerant readers during rolling updates. A renamed species keeps its stable ID; retired definitions retain tombstones and a display fallback so old possessions remain readable.

Maintain pre-migration backups/version references and immutable release manifests. Recovery distinguishes a wrong balance from a missing asset. Restore or compensate through authorized, audited operations after reconciling receipts and trades. Never globally restore yesterday's profiles after players have traded today: that can recreate the same item under multiple owners. Feature flags can stop a faulty content source or commerce path without deleting data. Global unique first-discovery claims, if later approved, use a durable conditional key per category/season; server announcements and display indexes are downstream projections.

Use structured logs: timestamp, severity, build/content/schema versions, place/server ID, operation ID, run/trade ID when relevant, redacted actor identifier, error code, and sanitized context. Sample routine diagnostics; retain authoritative transaction/recovery records separately. Do not log private chat, full profiles, raw receipt payloads, access tokens, or unnecessary personal data. Alerts cover commit latency, throttling, fence loss, unknown write outcomes, migration failure, stuck escrow, invalid ownership, economy imbalance, and memory growth.

Analytics is downstream and cannot block extraction. Track entered-run, seen, capture-attempt, captured-pending, captured-committed, extraction-started, extraction-committed, failed-run, secured-collection, upgrade, and second-run events with consistent run and content identities. This distinguishes an exciting sighting from a saved reward. Record onboarding and mobile cohorts, failure reasons, and optional diagnostic samples. Missing telemetry must not repeat economy effects. Durable economy records supply reconciliation; analytics dashboards are not the balance ledger.

Use MemoryStore for transient party presence, matchmaking, short-lived routing tickets, and recovery acceleration. It expires and is not the durable authority for ownership. MessagingService can invalidate cached schedules or announce committed discoveries; delivery is best effort, so servers periodically reconcile against versioned durable schedules and never require a message to complete a reward. [Memory stores](https://create.roblox.com/docs/cloud-services/memory-stores), [MessagingService](https://create.roblox.com/docs/reference/engine/classes/MessagingService/SubscribeAsync)

## 5. Data Architecture

### 5.1 Conventions and classification

The field contracts below are conceptual schemas, not executable Luau. `Id` is a stable bounded string; `UserId` is an integer; `Timestamp` is server UTC seconds; `Int` is a validated bounded integer; `Number` must be finite; `?` means optional; `[]` means a bounded list; `Map<A,B>` means a bounded dictionary. Persist only serializable primitives and structures, never Instances, functions, random generators, CFrames, or unchecked client tables. Positions are quantized local coordinates and orientation is a constrained rotation representation. Currency uses integer units with an explicit maximum below exact-integer precision limits; no floating-point money.

**Static configuration** is an immutable validated content release. **Player persistent** data represents owned value and saved progression. **Durable shared state** represents cross-player coordinators, ownership records, and global claims. **Server session state** contains live simulation, queues, encounter reservations, and cached projections. **Client-only state** contains camera, inputs, animation blending, menu selection, preview placement, accessibility preferences in effect, and pending command presentation. Client-only state never proves a grant. A cached copy does not change the authoritative classification.

Definitions reference asset-registry IDs rather than raw scattered Roblox asset numbers. Static conditions use an allowlisted vocabulary such as region, weather, time window, mastery threshold, and capability requirement. Effects use typed handlers such as apply cosmetic material, change capped movement factor, unlock route, or grant a defined resource. Content cannot contain arbitrary executable strings. Validate references, enum values, unit/range constraints, variant compatibility, total weights, cyclic prerequisites, animation availability, and asset permissions during packaging.

Every definition includes `id: Id`, `definitionVersion: Int`, `introducedIn: ContentVersion`, `retiredIn?: ContentVersion`, `nameKey: Id`, and relevant asset/localization references. Deleting a catalog row that persistent records still reference is prohibited. Retired content stays resolvable. The following entries list domain-specific fields in addition to these common fields.

### 5.2 Required schemas

**1. PlayerData — player persistent; written only by DataService.**

Fields: `userId: UserId`; `schemaVersion: Int`; `revision: Int`; `createdAt/lastSeenAt: Timestamp`; `storageGeneration: Int`; `session: {id: Id, fence: Int, leaseUntil: Timestamp}`; `wallet: Map<CurrencyId,Int>`; `holdings: {creatures: Map<InstanceId,OwnedCreature>, stacks: Map<StackKey,InventoryItem>, equipment: Map<InstanceId,Equipment>, cosmetics: Map<Id,Entitlement>}`; `inventoryManifest?: Manifest`; `progression: {explorerXp: Int, unlockedBiomes: Id[], mastery: Map<Id,Int>}`; `sanctuary: Sanctuary`; `collectionBook: CollectionBook`; `quests: Map<QuestInstanceId,QuestProgress>`; `achievements: Map<Id,AchievementProgress>`; `activeRun?: ActiveRun`; `closedRunOrdinal: Int`; `claimReceipts: ClaimIndex`; `purchaseReceipts: ReceiptIndex`; `pendingTransactions: Id[]`; `recentOperations: OperationReceipt[]`; `settings: Settings`; `lastOfflineClaimAt: Timestamp`.

Invariants: one fencing owner; balances nonnegative; no asset in two ownership domains; active run at most one; committed revision monotonic; schema newer than a server's writer is not writable. Initial inline holdings and later manifest-backed holdings are mutually exclusive per storage generation. A bounded recent-operation journal does not replace permanent purchase/claim identity. Closed run ordinals reject every request for an already-closed run, even after detailed receipts are archived.

**2. CreatureDefinition — static, with public and private projections.**

Fields: `speciesId: Id`; `familyId: Id`; `baseRarity: RarityId`; `biomeIds: Id[]`; `habitatTags: Id[]`; `sizeRange: {min:Number,max:Number}`; `personalityPool: WeightedId[]`; `traitPoolId: Id`; `allowedMutationIds: Id[]`; `mutationSlotLimit: Int`; `behaviorArchetypeId: Id`; `captureRuleId: Id`; `spawnRuleIds: Id[]`; `baseSellValue: Int`; `assetSetId/rigFamilyId/animationSetId/audioSetId: Id`; `collectionCategoryIds: Id[]`; `displayBounds: Bounds`; `tradePolicyId: Id`.

Invariants: species identity is distinct from mutation identity; an Owl can be a rare species while Crystal is a Fox-compatible mutation. Spawn weights and reward mechanics remain private, with player-facing eligibility guidance published separately. A definition creates no owned creature by itself.

**3. OwnedCreature — player persistent initially; asset authority later in ownership ledger.**

Fields: `instanceId: Id`; `creatureDefinitionId: Id`; `rolledDefinitionVersion: Int`; `origin: {runId:Id,encounterId:Id,participantUserId:UserId,claimId:Id}`; `originalDiscovererId: UserId`; `acquiredAt/securedAt: Timestamp`; `size: Number`; `personalityId: Id`; `traitInstances: {traitId:Id,roll:Number}[]`; `mutationIds: Id[]`; `cosmeticSkinId?: Id`; `nickname?: bounded filtered string`; `locked: Bool`; `favorite: Bool`; `displayAssignmentId?: Id`; `ownershipRevision: Int`; `binding: BindingPolicyId`; `provenanceRef?: Id`.

Invariants: instance ID never changes on trade; permanently banked only after extraction; original provenance never becomes current owner identity; equip/display/escrow exclusions are enforced; mutable nickname is bounded and filtered, and is not an ID. Ownership ledger, when enabled, is authoritative over inventory indexes.

**4. InventoryItem — player persistent for banked stacks; ActiveRun for pending cargo.**

Fields: `entryId: Id`; `itemDefinitionId: Id`; `quantity: Int`; `qualityBand?: Id`; `binding: BindingPolicyId`; `expiryAt?: Timestamp`; `state: Banked | PendingRun | Reserved`; `reservationId?: Id`; `originRef?: Id`; `revision: Int`. Derived `stackKey` includes definition, quality, binding, and expiration bucket.

Invariants: positive bounded quantity; incompatible binding/expiry/quality never silently merge; equipped unique assets are not stacks; item quantity and reserved quantity cannot exceed ownership. Inventory UI slots are a presentation/capacity concept, not DataStore key locations.

**5. ItemDefinition — static.**

Fields: `category: Resource | Relic | Quest | Cosmetic | Consumable | Equipment`; `stackLimit: Int`; `carryWeightUnits: Int`; `iconId: Id`; `prefabId?: Id`; `sellRuleId?: Id`; `useEffectId?: Id`; `tradePolicyId: Id`; `bindingPolicyId: Id`; `expiryPolicyId?: Id`; `tags: Id[]`; `rarityId: Id`; `acquisitionRuleIds: Id[]`.

Invariants: a paid cosmetic cannot inherit a currency selling path accidentally; prices come from active server rules; sell/use eligibility is explicit. Catalog expansion does not require loading every model.

**6. BiomeDefinition — static; instantiated runtime is separate.**

Fields: `regionId: Id`; `worldPackageId: Id`; `adjacentBiomeIds: Id[]`; `entryRequirements: Condition[]`; `routeGraphId: Id`; `dangerBands: DangerBand[]`; `spawnTableIds/resourceTableIds: Id[]`; `supportedWeatherIds: Id[]`; `hazardProfileIds: Id[]`; `extractionAnchorTags: Id[]`; `proceduralRuleSetId: Id`; `environmentPresetId: Id`; `populationBudget: Budget`; `onboardingRulesId?: Id`.

Invariants: valid safe arrival and extraction routes; seeded placement never blocks mandatory paths; minimum navigation readability retained in fog; every generated point passes terrain and collision validation. Coordinates are authored anchors plus bounded variation, not unconstrained random positions.

**7. ResourceDefinition — static.**

Fields: `itemDefinitionId: Id`; `nodePrefabId: Id`; `harvestCapabilityId: Id`; `harvestDurationSeconds: Number`; `yieldRuleId: Id`; `respawnRangeSeconds: Range`; `spawnConditionIds: Id[]`; `claimPolicy: PerParticipant | SharedDepleting`; `maxConcurrentNodes: Int`; `harvestAudioVfxId: Id`.

Runtime companion: `nodeId: Id`; `biomeInstanceId: Id`; `spawnEpoch: Int`; `remainingCharges: Int`; `claimedUserIds: UserId[]`; `nextSpawnAt: Timestamp`; `position: QuantizedVector`. Claims use node ID, epoch, and participant identity; reusing an art model cannot reset its entitlement.

**8. Discovery — player persistent evidence or durable scoped first-discovery claim.**

Fields: `discoveryKey: Id`; `speciesId: Id`; `mutationSignature: Id`; `seenAt?: Timestamp`; `securedAt?: Timestamp`; `seenBiomeIds: Id[]`; `firstRunId?: Id`; `firstSecuredInstanceId?: Id`; `largestSecuredSize?: Number`; `originEvidenceId: Id`; `scope: Personal | ServerSession | GlobalSeason`; `scopeId?: Id`; `discovererUserId?: UserId`.

Invariants: observation grants knowledge only; secured fields require extraction evidence. Server-first and world-first labels must never be interchangeable. Global prestige needs its own conditional durable claim and is deferred; client clocks cannot determine the winner.

**9. Mutation — static definition plus IDs on an owned instance.**

Fields: `compatibleSpeciesTags: Id[]`; `incompatibleMutationIds: Id[]`; `slotCost: Int`; `spawnConditionIds: Id[]`; `appearanceRecipeId: Id`; `scaleModifier: Number`; `statModifiers: CappedModifier[]`; `rarityPresentationId: Id`; `valueMultiplierRuleId: Id`; `effectBudgetClass: Id`; `eventBindingId?: Id`.

Invariants: canonical sorted signature prevents duplicate collection entries; bounded combinations avoid exponential completion demands; scale respects collision/capture bounds; rarity visuals cannot hide gameplay telegraphs. A new mutation using supported effects is data plus assets; a new effect type requires a reviewed capability change.

**10. Trait — static definition plus bounded rolled values on instances.**

Fields: `effectType: Id`; `validContexts: ContextId[]`; `valueRange: Range`; `stackingGroupId: Id`; `stackingRule: Max | AddCapped | Exclusive`; `globalCap: Number`; `compatibilityTags: Id[]`; `rollWeight: Int`; `descriptionKey: Id`; `tradeDisclosureKey: Id`.

Invariants: server recomputes effects from the approved equipped/displayed set; traits do not multiply one another recursively; inactive stored creatures grant no invisible bonus; cosmetic skins do not alter traits. Initial slice may store future trait capacity while enabling no economic trait bonuses.

**11. Equipment — static EquipmentDefinition and player persistent instance.**

Definition fields: `itemDefinitionId: Id`; `slotId: Id`; `tier: Int`; `capabilityIds: Id[]`; `upgradePathIds: Id[]`; `capacityBonus: Int`; `movementModifiers: CappedModifier[]`; `toolActionIds: Id[]`; `assetSetId: Id`. Instance fields: `instanceId: Id`; `definitionId: Id`; `upgradeLevel: Int`; `equippedSlot?: Id`; `cosmeticId?: Id`; `revision: Int`.

Invariants: only one item per exclusive slot; capabilities derive from ownership and upgrade state; expedition loadout freezes at start unless a supported resupply mechanic says otherwise. No durability repair system is implied by this schema; add it only if design evidence warrants it.

**12. Sanctuary — player persistent; public projection for visitors.**

Fields: `sanctuaryId: Id`; `ownerUserId: UserId`; `templateId: Id`; `level: Int`; `upgradeLevels: Map<Id,Int>`; `placements: Placement[]`; `displayAssignments: {slotId:Id,creatureInstanceId:Id}[]`; `unlockedSlotIds: Id[]`; `themeId: Id`; `visitPolicy: Friends | Public | Private`; `production: {lastClaimAt:Timestamp,ruleVersion:Id}`; `revision: Int`.

Placement fields: `placementId: Id`; `itemInstanceOrEntitlementId: Id`; `socketId?: Id`; `localPosition: QuantizedVector`; `rotationStep: Int`. Invariants: bounded placement count, valid plot bounds, owned item, no blocked entry, no duplicate exclusive display. Initial sanctuary uses predefined sockets; free placement is deferred. Offline production derives from bounded elapsed server time, not timers per pet or client time.

**13. SanctuaryUpgrade — static, with achieved level in Sanctuary.**

Fields: `maxLevel: Int`; `prerequisites: Condition[]`; `costByLevel: CostBundle[]`; `effectByLevel: CapabilityGrant[]`; `constructionSocketId?: Id`; `presentationPrefabByLevel: Id[]`; `buildDurationSeconds?: Number`.

Invariants: cost and effect commit together; prerequisites form an acyclic graph; capacities cannot decrease below current ownership; free introductory upgrade is an explicit rule rather than a client bypass. Slice has one meaningful upgrade and no waiting timer.

**14. Quest — static QuestDefinition and player persistent QuestProgress.**

Definition: `questType: Tutorial | Repeatable | Daily | Weekly | Seasonal`; `eligibility: Condition[]`; `objectiveSteps: ObjectiveDefinition[]`; `rewardBundleId: Id`; `scheduleId?: Id`; `expiryBehavior: ExpiryPolicyId`; `localizationKeys: Id[]`. Progress: `questInstanceId: Id`; `definitionId: Id`; `ruleVersion: Id`; `periodId?: Id`; `stepCounters: Map<Id,Int>`; `acceptedAt: Timestamp`; `completedAt/claimedAt?: Timestamp`; `claimId?: Id`.

Invariants: objective source distinguishes seen from secured; server events advance counters; claim is once per quest instance/period; a schedule rollover does not erase already-claimable earned rewards without an explicit grace policy. Counters cap at required progress.

**15. Event — static EventDefinition, durable schedule, and server runtime instance.**

Definition: `eventType: Id`; `compatibleBiomes: Id[]`; `triggerPolicyId: Id`; `durationSeconds: Int`; `modifierBundleId: Id`; `spawnOverlayIds: Id[]`; `questIds: Id[]`; `assetPackageId: Id`; `budgetClass: Id`; `endPolicyId: Id`. Schedule: `scheduleId: Id`; `eventDefinitionId: Id`; `contentVersion: Id`; `startAt/endAt: Timestamp`; `cohortRuleId?: Id`; `killSwitchId: Id`. Runtime: `eventInstanceId: Id`; `scheduleId: Id`; `startedAt: Timestamp`; `seedRef: Id`; `state: Id`.

Invariants: UTC schedule is authoritative; event end stops new eligibility but does not invalidate already-earned cargo; overlapping modifiers follow explicit priorities and caps; missing messaging does not skip a calendar transition. Slice fog changes route readability and presentation within a safe budget.

**16. Economy transaction — player persistent receipt initially; durable shared coordinator for cross-key work.**

Fields: `transactionId: Id`; `idempotencyKey: Id`; `type: Grant | Spend | Sell | Extract | Upgrade | Receipt | Trade | Compensation`; `actorUserId: UserId`; `sourceRef: Id`; `currencyLegs: {accountId:Id,currencyId:Id,delta:Int}[]`; `itemLegs: ItemTransfer[]`; `ruleVersion: Id`; `expectedRevisions: RevisionRef[]`; `state: Prepared | Committed | Settled | Aborted`; `createdAt/committedAt?: Timestamp`; `resultDigest: Id`; `reasonCode: Id`; `auditRef?: Id`.

Invariants: grant/sink legs identify a system source/sink, not unexplained balance edits; player wallet never negative; transfer legs conserve value except explicit fees/sinks; receipt ID and outcome match. This is a record of authority, not an analytics event pretending to be a ledger.

**17. Trade — durable shared coordinator; transient editing projection.**

Fields: `tradeId: Id`; `participants: [UserId,UserId]`; `offerRevision: Int`; `offers: Offer[2]`; `offerDigest: Id`; `confirmations: {userId:UserId,revision:Int,digest:Id,at:Timestamp}[]`; `state: Draft | Confirming | Preparing | Committed | Settled | Aborted`; `escrowAssetRefs: OwnershipRef[]`; `capacityReservations: ReservationRef[]`; `coordinatorEpoch: Int`; `createdAt/expiresAt: Timestamp`; `committedAt/settledAt?: Timestamp`; `recoveryStatus: Id`.

Offer fields: `assetIds: Id[]`; `expectedOwnershipRevisions: Int[]`; `resourceLots?: Lot[]`; `currencyLegs?: CurrencyLeg[]`. Invariants: bounded two-party immutable sealed offer; both confirm same digest; editing invalidates confirmations; no postcommit abort; logs include original offer and terminal ownership outcome; noneligible future field types are rejected while their feature flags are disabled.

**18. Achievement — static definition and player persistent progress.**

Definition: `criteriaRuleId: Id`; `thresholds: Int[]`; `visibility: Public | HiddenUntilEarned`; `rewardBundleId?: Id`; `platformBadgeId?: Int`; `scopeId: Id`. Progress: `achievementId: Id`; `tier: Int`; `counter: Int`; `earnedAt?: Timestamp`; `claimIds: Id[]`; `badgeDeliveryState?: Id`.

Invariants: platform badge delivery is retryable presentation/recognition after authoritative earning; it cannot repeatedly grant game rewards. A trade does not create an original discovery achievement unless a rule explicitly says ownership is sufficient.

**19. CollectionBook — player persistent compact progress plus static collection definition.**

Definition: `collectionId: Id`; `entryKeys: Id[]`; `completionRuleId: Id`; `milestoneRewardIds: Id[]`; `seasonId?: Id`; `completionVersion: Int`. Player fields: `bookVersion: Int`; `entries: Map<DiscoveryKey,Discovery>`; `completedMilestones: Map<Id,Timestamp>`; `claimedMilestoneIds: Id[]`; `pinnedGoals: Id[]`; `scopedCompletion: Map<Id,Int>`.

Invariants: only secured entries count toward ownership/completion rewards; seen silhouettes may show hints. Adding new content creates a new completion version or extension page, never silently revokes earned completion rewards. Use sparse entries initially; introduce indexed bitsets only with a versioned index map and demonstrated size benefit.

### 5.3 Supporting records and data evolution

**ActiveRun** is a required persistent companion: `runId`, monotonic `runOrdinal`, user and owning session, lifecycle state, start time, content/loss-policy versions, biome/route, loadout snapshot, capacity reservation, checkpoint revision, pending cargo, committed encounter claims, current checkpoint evidence, and optional terminal transaction reference. Live AI positions, frame-by-frame health, exact pathfinding state, and network requests remain server session state. The failure policy resolves incomplete runs; reloading a checkpoint is not a player-controlled undo.

**OwnershipRecord** contains unique asset ID, immutable creation/provenance, current owner or escrow transaction, ownership revision, finalized transaction marker, and state. **InventoryManifest** contains generation, page refs and hashes, totals, schema version, and pending publication transaction. **OperationReceipt** contains operation ID, result digest, committed revision, and bounded summary. **Entitlement** contains source, entitlement ID, granted-at, binding, and revocation state where relevant. **AssetRegistryEntry** contains stable asset ID, Roblox imported asset IDs, source file hash, version, rig/animation contracts, bounds, collisions, LOD mappings, permissions, and approved budget class.

Do not append every expedition event forever to PlayerData. Keep lifetime counters and sparse discovery state, retain recent summaries for player support, and archive detailed audit transactions in bounded partitions with an explicit retention policy. Permanent deduplication uses durable source identities, run ordinals, receipt archives, and committed claim records. A capped journal alone would allow old rewards to be replayed.

For large catalogs, load the release manifest and essential summaries first, then biome-specific tables and detail pages on demand. For large holdings, query by category/filter with an opaque cursor tied to manifest generation. If holdings change during pagination, return a stale-cursor response and refresh the affected view. Sanctuary visitors receive the chosen display projection, never full private inventories, economic history, or purchase records.

## 6. Dependency Graph

### 6.1 Runtime dependency graph

Arrows mean **provider → consumer**. These are capability dependencies, not a requirement to finish every provider's future feature before integrating a slice. Presentation consumes read models; it never owns a reverse dependency from authoritative gameplay.

```mermaid
flowchart TD
    Contracts[Contracts · IDs · validation · time · RNG] --> Catalog[Versioned content catalog]
    Contracts --> Net[Network gateway and player lifecycle]
    Contracts --> Data[Persistence · fencing · migrations]
    Contracts --> Observe[Telemetry · diagnostics]
    Net --> Session[Player session]
    Data --> Session
    Data --> Inventory[Inventory and ownership]
    Catalog --> Inventory
    Inventory --> Economy[Economy · reservations · grants]
    Catalog --> World[Zones · spatial queries · procedural director]
    Net --> World
    World --> Creatures[Creature encounters and behavior]
    Catalog --> Creatures
    Economy --> Equipment[Equipment and upgrade transactions]
    Catalog --> Equipment
    Creatures --> Capture[Discovery and capture]
    Equipment --> Capture
    Inventory --> Capture
    World --> Runs[Expedition and hazards]
    Capture --> Runs
    Session --> Runs
    Runs --> Extract[Extraction coordinator]
    Economy --> Extract
    Inventory --> Extract
    Data --> Extract
    Inventory --> Sanctuary[Sanctuary · display · placement]
    Economy --> Sanctuary
    Catalog --> Sanctuary
    Capture --> Progress
    Economy --> Progress
    Extract --> Progress[Progression · quests · collection · achievements]
    Catalog --> Progress
    Catalog --> EventDirector[Events and schedules]
    EventDirector --> World
    EventDirector --> Creatures
    Sanctuary --> Social[Visits · parties · scoped prestige]
    Runs --> Social
    Inventory --> Trade[Optional trading and escrow]
    Economy --> Trade
    Data --> Trade
    Social --> Trade
    Economy --> Commerce[Optional purchases and entitlements]
    Data --> Commerce
    World --> ReadModels
    Equipment --> ReadModels
    Capture --> ReadModels
    Runs --> ReadModels
    Inventory --> ReadModels
    Extract --> ReadModels[Sanitized read models]
    Progress --> ReadModels
    Sanctuary --> ReadModels
    Trade --> ReadModels
    Commerce --> ReadModels
    ReadModels --> Presentation[Client controllers · UI · audio · VFX]
    Catalog --> Presentation
```

**Foundation:** contracts, catalog validation, player lifecycle, gateway, persistence, diagnostic adapters. **Mid-level:** inventory, economy, spatial world queries, equipment, encounters. **Gameplay:** discovery, capture, expeditions, extraction, sanctuary, progression. **Content:** definitions/assets, biome packages, events/schedules; templates consume already-supported gameplay capabilities. **Presentation:** projections, controllers, UI, audio and visual assets. **Optional:** trading, commerce, broad housing, offline production, ranked prestige, raids and PvP.

The expedition coordinator supplies run context to discovery/capture validation, then commits the returned entitlement and cargo plan together through persistence. Capture never writes cargo separately or calls back into the expedition service. Observation evidence can update collection history before extraction; secured completion still requires extraction.

Progression publishes an **access projection**, which a coordinator passes into expedition/equipment requests. Runs never import ProgressionService to award XP while ProgressionService calls back into runs. Likewise, an event schedule supplies a frozen modifier context; world spawning does not call the event scheduler to advance its lifecycle. This keeps the graph acyclic even when product concepts influence one another.

### 6.2 Project prerequisite graph

The exact hard prerequisite list appears on each project card. Parallel starting bands are:

- **A:** P01.
- **B:** P02 and P06.
- **C:** P03, P07, P18, P21, P23 foundation.
- **D:** P04, P08, P24 pipeline foundation.
- **E:** P05 and P19 foundation.
- **F:** P11, P12, P22.
- **G:** P09.
- **H:** P10.
- **I:** P13, P14.
- **J:** P15, P16 and P17. P20 is technically dependency-ready in G but intentionally held until the free-loop and receipt safety gates pass in Phase 6.

A project may begin interface design after P01; production implementation consumes tested provider milestones. For example P19 can build a HUD contract after P02/P04/P06, while its extraction success screen cannot pass integration until P10 settles a run correctly. P23/P24 start tooling early and certify completed releases later. They are not postponed until all feature work is finished.

**First playable critical path:** P01 → P02/P06 → P03/P07 → P04/P08 → P05 → P12 → P09 → P10, integrated with minimal P11/P13/P19 and foundation P21/P23/P24. P18/P17 supply only the small required asset/world package. Single-domain tests run at each step; do not await the last arrow to test multiplayer.

### 6.3 Circular dependencies to prohibit

- Inventory must not call CaptureService or TradeService. These orchestrators submit validated inventory operations; inventory enforces ownership and capacity.
- Economy must not invoke quests to determine rewards. A validated reward plan arrives from an orchestrator; a committed outcome later drives quest projections with deduplication.
- DataService must not import every gameplay service to save it. Registered schema validators/migrations and a transaction coordinator operate on declared aggregates.
- Collection and Sanctuary must not call each other to grant creatures. Both derive their references from ownership and secured-discovery events.
- Monetization must not make itself a prerequisite for free progression. Commerce grants catalog-defined entitlements through the same durable grant boundary.
- Event configuration must not load arbitrary module paths. A registry dispatches to approved handlers.
- UI must not be required for completing a server transaction. Closing a screen or disconnecting cannot cancel a committed trade/extraction.
- Telemetry must not be necessary for a grant. Durable state is authoritative even during analytics outage.

Mandatory rewards that must settle together belong inside one authoritative operation. Secondary effects such as announcements and achievement projections use a durable outbox with idempotent consumers; an in-memory event bus alone is insufficient for effects that must survive a crash.

## 7. Project Decomposition

The 24 projects are durable ownership domains. Their **minimum slice**, **expansion** and **release** milestones are tracked separately. A dependency below means a tested provider capability is required for implementation/integration; documentation and asset exploration can proceed earlier against approved contracts. “Consumers” lists direct project prerequisites, not every transitive consumer. P23 tests all enabled domains and P24 consumes their release artifacts even where they are not direct implementation prerequisites.

### Project 01 — Master Architecture / Technical Foundation

**Purpose:** Turn this recommendation into an approved, testable contract set for the team.

- **Owns:** Domain boundaries; invariant register; ADR baseline; public API conventions; repository/toolchain plan; slice acceptance plan.
- **Does not own:** Feature implementation, production credentials, complete framework generation and art production.
- **Dependencies:** None; starts from the supplied brief and this specification.
- **Systems required:** Contract versioning, ownership map, dependency checking and change control.
- **Assets required:** Architecture diagrams and slice asset requirements only.
- **Data required:** Schema catalog, ID namespaces, example records and migration policy.
- **Roblox modules required:** Specifications for Bootstrap, ServiceRegistry, ContractTypes and adapters; no module implementation in its documentation milestone.
- **APIs/interfaces required:** Result/error envelope; command/query/event conventions; repository and content lookup contracts.
- **Inputs:** Approved master brief, this specification, owner constraints and current repository inventory.
- **Outputs:** Approved architecture baseline, documented seams, P02–P24 briefs and acceptance fixtures.
- **Testing requirements:** Independent trace of success, failure, timeout, replay and stale-session flows; dependency-cycle audit; schema example validation plan.
- **Completion criteria:** Every persistent field and public operation has an owner; no unresolved critical ownership decision; owner approval recorded.
- **Other projects that depend on it:** P02, P03, P06, P18, P21, P23, P24
- **Recommended development order:** Phase 0; first. Documentation completion precedes any separately authorized scaffold milestone.
- **Estimated complexity:** High

### Project 02 — Player Runtime, Networking and Interaction

**Purpose:** Provide a reliable player lifecycle and secure input boundary for every gameplay system.

- **Owns:** Bootstrap order, join/leave/respawn, ready states, intent routing, rate limits, interaction targeting and client lifecycle cleanup.
- **Does not own:** Saving policy, creature rewards, economy math and domain-specific eligibility.
- **Dependencies:** P01
- **Systems required:** PlayerSession facade, bootstrap/service registration, networking gateway and basic interactable contract.
- **Assets required:** Standard Roblox avatar setup and temporary interactable markers.
- **Data required:** Session status, message schema, request IDs, rate buckets and client preferences projection.
- **Roblox modules required:** ServerBootstrap, ClientBootstrap, PlayerService, NetworkService, InteractionService; InputController, InteractionController.
- **APIs/interfaces required:** PlayerReady/PlayerLeaving; RequestIntent; IntentResult; register interactable; sanitized snapshot subscription.
- **Inputs:** P01 contracts, player input and platform lifecycle callbacks.
- **Outputs:** Authenticated bounded intents and deterministic session readiness events.
- **Testing requirements:** Malformed payloads, duplicate requests, stale revisions, respawn spam, disconnect during boot, mobile/controller binding and cleanup soak.
- **Completion criteria:** Unready clients cannot mutate gameplay; invalid messages are safely rejected; eight clients can join/respawn without leaked connections.
- **Other projects that depend on it:** P03, P07, P08, P09, P10, P11, P12, P14, P19, P21, P22, P23
- **Recommended development order:** Phase 1 foundation; controllers expand alongside every later domain.
- **Estimated complexity:** High

### Project 03 — Persistence, Sessions and Recovery

**Purpose:** Keep durable player state consistent across crashes, reconnects and schema changes.

- **Owns:** Repository adapters, fenced leases, write scheduling, migration engine, operation deduplication, reconciliation and data recovery.
- **Does not own:** Domain reward rules, direct UI state and treating analytics as a ledger.
- **Dependencies:** P01, P02
- **Systems required:** Player aggregate, durable operation records/outbox, checkpoints, budget monitor and later segmented manifest migration.
- **Assets required:** None; isolated test experience and fixture datasets are required.
- **Data required:** PlayerData envelope, session epoch, schema version, revisions, operation outcomes, shard manifests and recovery markers.
- **Roblox modules required:** DataService, PlayerRepository, TransactionCoordinator, LeaseManager, SaveScheduler, MigrationRegistry, RecoveryService.
- **APIs/interfaces required:** AcquireProfile; ReadProjection; ApplyOperation; ResolveOperation; ReleaseProfile; MigrateProfile; recovery inspection.
- **Inputs:** Validated operation plans, session identity and known schema versions.
- **Outputs:** Durable outcomes, safe snapshots, conflict/pending errors and diagnostic records.
- **Testing requirements:** Timeout after commit, concurrent leases, stalled old server, corrupt entry, budget exhaustion, shutdown cuts and every migration interruption.
- **Completion criteria:** No stale writer succeeds; replay cannot double-apply; load failure never creates a replacement profile; recovery runbook proven.
- **Other projects that depend on it:** P04, P05, P10, P13, P14, P15, P20, P22
- **Recommended development order:** Phase 1 aggregate; capacity/segmentation milestone must pass before P15.
- **Estimated complexity:** Very High

### Project 04 — Inventory, Unique Ownership and Reservations

**Purpose:** Represent valuable items once and make capacity, ownership and transfer restrictions enforceable.

- **Owns:** Stacks, owned-creature/item IDs, capacity, paged queries, sorting indexes, reservations, locks and ownership migration semantics.
- **Does not own:** Capture rolls, sale prices, trade orchestration and display models as ownership records.
- **Dependencies:** P03, P06
- **Systems required:** Inventory repository, creature ownership view, bounded admission, reservation lifecycle and later escrow-compatible ownership ledger.
- **Assets required:** Item/icon references supplied through the catalog; no UI assets authored here.
- **Data required:** InventoryItem, OwnedCreature, stack keys, reservation IDs, storage tiers and ownership revisions.
- **Roblox modules required:** InventoryService, OwnershipService, CapacityPolicy, InventoryQuery, ReservationService.
- **APIs/interfaces required:** CanAccept; Reserve; ReleaseReservation; ApplyInventoryDelta; GetInventoryPage; GetOwnedEntity; validate custody.
- **Inputs:** Server-authorized grant/spend plans and catalog definitions.
- **Outputs:** Authoritative balances/custody, paged projections and typed rejection reasons.
- **Testing requirements:** Stack overflow, negative quantity, full inventory during extraction, double reservation, missing definitions, 10k fixture queries and migration replay.
- **Completion criteria:** Every unique item has one authority; capacity cannot silently destroy cargo; inventory size/write limits block unsafe growth.
- **Other projects that depend on it:** P05, P09, P10, P11, P15, P19, P20, P22
- **Recommended development order:** Phase 1 minimum storage, Phase 4 capacity proof, Phase 5 transfer readiness.
- **Estimated complexity:** Very High

### Project 05 — Economy, Rewards and Upgrade Transactions

**Purpose:** Centralize all value creation and spending with auditable, replay-safe outcomes.

- **Owns:** Wallets, deterministic prices, grants, sell/upgrade settlement, source/sink reasons, reward plans and transactional receipts for internal operations.
- **Does not own:** Robux receipt callback ownership, market speculation, quest completion logic and inventory storage internals.
- **Dependencies:** P03, P04, P06
- **Systems required:** Coins wallet, reward plan validation, atomic single-profile purchases, sale confirmation and transaction journal.
- **Assets required:** Currency icon references only.
- **Data required:** EconomyTransaction, currency definitions, price tables, recipe/upgrade costs and capped reward reason codes.
- **Roblox modules required:** EconomyService, RewardService, PriceResolver, TransactionJournal, UpgradePurchaseHandler.
- **APIs/interfaces required:** Quote; ValidateQuote; SpendAndGrant; SellItems; GrantReward; GetTransactionStatus.
- **Inputs:** Authenticated action context, current state revision and server catalog prices.
- **Outputs:** Committed debit/credit/entitlement outcomes and economy telemetry outbox events.
- **Testing requirements:** Negative/NaN/overflow amounts, quote expiry, repeated sale, insufficient funds, timeout retry and source/sink simulations.
- **Completion criteria:** No client-supplied price or reward is trusted; costs and rewards cannot commit separately; reason codes reconcile every change.
- **Other projects that depend on it:** P10, P11, P12, P13, P15, P20, P22
- **Recommended development order:** Phase 1 minimal Coins; economy modeling expands before offline output/trading/commerce.
- **Estimated complexity:** High

### Project 06 — Content Catalog, Definitions and Validation

**Purpose:** Make content additions predictable without per-creature service rewrites.

- **Owns:** Stable IDs, public/private catalog projections, definition validation, compatibility rules, immutable manifests and tombstones. Canonical runtime asset registry and resolution.
- **Does not own:** Asset modeling, world assembly, event scheduling and arbitrary code execution from configuration.
- **Dependencies:** P01
- **Systems required:** Typed catalog, reference graph validator, schema linter and allowlisted content-handler registry (not equipped-player capabilities).
- **Assets required:** Registry entries linking future prefabs, icons, audio and animation sets.
- **Data required:** Creature/Item/Biome/Resource/Mutation/Trait/Quest/Event definitions and content revision manifests.
- **Roblox modules required:** ContentRegistry, CatalogValidator, CompatibilityResolver, PublicCatalogBuilder, DefinitionMigrationMap.
- **APIs/interfaces required:** ResolveDefinition(id, revision); ResolveAsset; ValidateContentPackage; GetPublicCatalogPage.
- **Inputs:** Reviewed definitions, asset registry and supported handler IDs.
- **Outputs:** Validated immutable content package and safe client projection.
- **Testing requirements:** Duplicate IDs, dangling references, invalid weights, incompatible mutations, cyclic quest prerequisites, missing assets and retired IDs.
- **Completion criteria:** A compatible new creature needs data/assets only; invalid packages fail before publication; private reward logic is not replicated.
- **Other projects that depend on it:** P04, P05, P07, P08, P09, P11, P12, P13, P16, P17, P18, P19
- **Recommended development order:** Phase 1 contracts, continuous content gate from the first species onward.
- **Estimated complexity:** High

### Project 07 — World Runtime, Resources and Procedural Variation

**Purpose:** Provide bounded, replayable exploration spaces and resource encounters.

- **Owns:** Zone IDs, spatial queries, spawn sockets, resource node lifecycle, seed scopes, interest regions and procedural placement validation.
- **Does not own:** Biome art production, creature cognition, reward settlement and unconstrained terrain generation.
- **Dependencies:** P02, P06
- **Systems required:** Zone director, tagged world prefabs, resource gathering, seed service and reachability-aware spawn budgets.
- **Assets required:** Graybox meadow, resource nodes, extraction markers and collision proxies.
- **Data required:** BiomeDefinition, ResourceDefinition, spawn/socket tables, topology references and server seed context.
- **Roblox modules required:** WorldService, ZoneService, ResourceService, SpawnDirector, SpatialIndex, WorldSeedProvider.
- **APIs/interfaces required:** GetZoneAt; QueryNearby; RequestGather; ReserveSpawn; DespawnEncounter; GetTraversalRoute.
- **Inputs:** World package, event modifier context and server-approved interaction requests.
- **Outputs:** Reachable encounters, gathered resource intent records and bounded active world state.
- **Testing requirements:** Unreachable sockets, overlapping nodes, repeated gather, streamed-out references, seed replay and clustered eight-player load.
- **Completion criteria:** No node can pay twice; every mandatory route is traversable; procedural variation respects population/performance budgets.
- **Other projects that depend on it:** P08, P10, P12, P16, P17
- **Recommended development order:** Phase 2 graybox; reuse same APIs in every future biome.
- **Estimated complexity:** High

### Project 08 — Creature Runtime, Behavior and Variation

**Purpose:** Make species expressive and reusable while bounding simulation cost.

- **Owns:** Encounter lifecycle, behavior families, server eligibility, mutation/trait application, visual phenotype data and simulation LOD.
- **Does not own:** Permanent ownership, capture grant settlement, bespoke scripts embedded in every prefab and breeding.
- **Dependencies:** P02, P06, P07
- **Systems required:** Creature state machines, movement intent, hazard/alert reactions, compatible variants and spawn roster.
- **Assets required:** Fox/Deer/Owl prefabs or graybox substitutes, family rigs and animation profiles from P18.
- **Data required:** CreatureDefinition, Mutation, Trait, encounter state, immutable roll context and phenotype seed.
- **Roblox modules required:** CreatureService, CreatureStateMachine, BehaviorRegistry, MutationResolver, TraitResolver, CreatureSimulationScheduler.
- **APIs/interfaces required:** CreateEncounter; ObserveEncounter; GetCaptureContext; ApplyBehaviorSignal; ReleaseEncounter.
- **Inputs:** Approved spawn request, pinned catalog revision and population budget.
- **Outputs:** Server encounter IDs, validated behavior state and sanitized visual descriptors.
- **Testing requirements:** Incompatible mutation combinations, missing rig fallback, stuck navigation, death/despawn race and creature saturation.
- **Completion criteria:** Three species share framework handlers; new compatible definition needs no core edits; rare VFX cannot increase server simulation unboundedly.
- **Other projects that depend on it:** P09, P11, P16, P17
- **Recommended development order:** Phase 2 core behavior; advanced families only when new biomes justify them.
- **Estimated complexity:** High

### Project 09 — Discovery and Capture

**Purpose:** Turn noticing a creature into a fair, memorable and verifiable encounter.

- **Owns:** Observation evidence validation, scanner results, capture eligibility/challenge, per-player entitlement plans and immutable server-rolled cargo award plans.
- **Does not own:** Permanent banking, CollectionBook reward settlement, economic value and client-authored rarity. Direct writes to cargo or collection-history projections.
- **Dependencies:** P02, P04, P06, P08, P12
- **Systems required:** Discovery validation, capture attempt state machine, co-op participation and encounter deduplication.
- **Assets required:** Scanner/capture feedback hooks, creature discovery animations and sound references.
- **Data required:** Discovery, capture attempt, encounterId+userId entitlement, rolled creature form and unsecured cargo record.
- **Roblox modules required:** DiscoveryService, CaptureService, CapturePolicy, EncounterEntitlementRepository.
- **APIs/interfaces required:** Observe; StartCapture; SubmitCaptureInput; ResolveCapture; GetDiscoveryProjection.
- **Inputs:** Server proximity/visibility evidence, equipped tool capability and valid encounter state.
- **Outputs:** Validated observation evidence or immutable capture/entitlement plan; P10 commits entitlement consumption and cargo together through P03 before returning success.
- **Testing requirements:** Out-of-range capture, forged elapsed time, replayed entitlement, multiple participants, touch timing and despawn during attempt.
- **Completion criteria:** Every eligible participant can earn one outcome; rolls cannot be rerolled by retry; capture success is never confused with secured ownership.
- **Other projects that depend on it:** P10, P17
- **Recommended development order:** Phase 2 after the equipment minimum; polish in Phase 3.
- **Estimated complexity:** High

### Project 10 — Expeditions, Hazards, Risk and Extraction

**Purpose:** Own the game's defining decision and settle each run once.

- **Owns:** Run state machine, cargo, hazard exposure, failure policy, extraction validation, reconnect outcome and cross-place handoff contract. Atomic entitlement-consumption plus cargo commit for P09 capture plans.
- **Does not own:** Inventory persistence internals, player-versus-player theft, economy prices and region art.
- **Dependencies:** P02, P03, P04, P05, P07, P09
- **Systems required:** Run coordinator, hazard handlers, extraction channel, durable settlement and loss preview.
- **Assets required:** Extraction point, danger markers and hazard volumes; P19 supplies presentation.
- **Data required:** ActiveRun, RunCheckpoint, run ID, secured settlement operation and hazard profiles.
- **Roblox modules required:** ExpeditionService, HazardService, ExtractionService, RunSettlementPolicy, TransferTicketService specification.
- **APIs/interfaces required:** BeginRun; AddCargo; BeginExtraction; CancelExtraction; ResolveRun; ReconcileRun; GetCargoProjection.
- **Inputs:** Validated encounters/resources, server position/time and player-authorized departure/extraction intent.
- **Outputs:** One durable terminal outcome and corresponding ownership, currency and progression plan.
- **Testing requirements:** Capture/extract race, death during channel, double extraction, disconnect at commit, full bank, shutdown and reused transfer ticket.
- **Completion criteria:** A run has one terminal result; loss matches the displayed policy; no exit path banks unsecured loot accidentally.
- **Other projects that depend on it:** P13, P14, P16, P17
- **Recommended development order:** Phase 2 critical-path integration; cross-place implementation deferred until justified.
- **Estimated complexity:** Very High

### Project 11 — Sanctuary, Display and Bounded Construction

**Purpose:** Make discoveries feel permanent and visibly personal.

- **Owns:** Plot assignment, preset sockets, creature display references, upgrade layout, visitor permissions and later bounded build/offline production rules. Persistent sanctuary ACL and edit permissions are authoritative here.
- **Does not own:** Creature ownership, economy transaction code, sanctuary raids and general-purpose building editor.
- **Dependencies:** P02, P04, P05, P06, P08
- **Systems required:** Sanctuary aggregate, placement validation, habitat compatibility and bounded display simulation.
- **Assets required:** Camp, habitat/display sockets, one upgrade prefab and modular decorations from P18.
- **Data required:** Sanctuary, SanctuaryUpgrade, owned references, placement transforms and permission settings.
- **Roblox modules required:** SanctuaryService, PlacementService, HabitatResolver, SanctuaryProjection; OfflineAccrualService later.
- **APIs/interfaces required:** GetSanctuary; PlaceOwnedCreature; RequestUpgrade; PlaceDecoration; ValidateVisitPermission.
- **Inputs:** Authoritative owned IDs, layout definitions and committed upgrade entitlements.
- **Outputs:** Saved bounded layout and visitor-safe projection with visible secured creatures.
- **Testing requirements:** Foreign ownership, overlapping placement, boundary escape, deleted asset fallback, many visitors and offline claim replay later.
- **Completion criteria:** One upgrade persists; display never duplicates ownership; visitors cannot edit without authority; plot reset releases all instances.
- **Other projects that depend on it:** P14, P20
- **Recommended development order:** Phase 2 preset slice; Phase 4 decorations; offline production after economy gate.
- **Estimated complexity:** High

### Project 12 — Equipment, Traversal and Access Capabilities

**Purpose:** Make upgrades open understandable exploration opportunities.

- **Owns:** Loadouts, equipment capabilities, deterministic upgrades, tool cooldowns and traversal validation.
- **Does not own:** Inventory balances, capture reward odds, broad crafting, mounts and combat trees.
- **Dependencies:** P02, P05, P06, P07
- **Systems required:** Basic scanner/net/backpack, capability resolution and server-validated traversal permissions.
- **Assets required:** Tool prefabs, attachment points and basic equip animations.
- **Data required:** Equipment, ItemDefinition capability profiles, upgrade recipes and loadout IDs.
- **Roblox modules required:** EquipmentService, LoadoutService, CapabilityResolver, TraversalValidator.
- **APIs/interfaces required:** Equip; GetCapabilities; RequestUpgradeQuote; ApplyEquippedState; ValidateTraversal.
- **Inputs:** Owned tools, approved purchase outcomes and access projection passed by coordinator.
- **Outputs:** Bounded equipped capability snapshot and movement/tool behavior permissions.
- **Testing requirements:** Equip during capture, forged equipment IDs, cooldown spam, inaccessible route bypass and touch/controller operation.
- **Completion criteria:** Starting tools support all slice encounters; free deterministic upgrades improve access without hiding rare encounters behind payment.
- **Other projects that depend on it:** P09, P17
- **Recommended development order:** Phase 2 starting tools only; later rope/heat/diving extensions require new capability tests.
- **Estimated complexity:** Medium

### Project 13 — Progression, Quests, Collection Book and Achievements

**Purpose:** Turn successful discoveries into clear short- and long-term goals.

- **Owns:** Explorer XP, access projection, objective evaluation, collection milestones, historical seen/secured records, achievement claims and onboarding steps.
- **Does not own:** Capture authority, creature ownership, direct currency mutation and independent progression ladders without a demonstrated need.
- **Dependencies:** P03, P05, P06, P10
- **Systems required:** Idempotent outcome consumers, objective templates, collection index, deterministic milestone rewards and goal selection.
- **Assets required:** Book icons, reward display references and short onboarding text.
- **Data required:** CollectionBook, Achievement, Quest definition/instance, progression counters and event-consumer cursors.
- **Roblox modules required:** ProgressionService, QuestService, CollectionService, AchievementService, OnboardingService.
- **APIs/interfaces required:** ApplySecuredOutcome; GetAccessProjection; ClaimMilestone; GetCollectionPage; GetActiveGoals.
- **Inputs:** Durable observation evidence and committed run/secured-discovery outcomes, catalog rules and previous durable progress.
- **Outputs:** Saved goals/unlocks, collection projections and idempotent reward plans.
- **Testing requirements:** Repeated outbox event, out-of-order updates, trade-away historical records, expired quests, missing rare drop and claim replay. Observed-only encounter followed by failure and rejoin preserves acknowledged seen history without granting ownership.
- **Completion criteria:** Seen and secured are distinct; one first upgrade goal is attainable without rare RNG; every milestone pays once.
- **Other projects that depend on it:** P16, P17
- **Recommended development order:** Phase 2 minimal journal/onboarding, Phase 4 richer quests and collections.
- **Estimated complexity:** High

### Project 14 — Social Play, Visits, Parties and Prestige

**Purpose:** Make shared exploration and sanctuary pride useful before trading exists.

- **Owns:** Party membership, consented travel, visit consent and routing using P11 permission projections, participation roster, throttled discovery announcements and scoped first-discoverer records.
- **Does not own:** Item transfer, global wealth valuation, unrestricted chat implementation and forced PvP.
- **Dependencies:** P02, P03, P10, P11
- **Systems required:** Party lifecycle, invite/visit flow, session prestige, privacy controls and later leaderboard projections.
- **Assets required:** Party indicators, sanctuary visiting cues and scoped trophies.
- **Data required:** Party session, visit permission, discovery announcement ID and first-discovery scope record.
- **Roblox modules required:** PartyService, VisitService, SocialAnnouncementService, PrestigeService.
- **APIs/interfaces required:** Invite; AcceptInvite; LeaveParty; RequestVisit; PublishVerifiedDiscovery; ClaimScopedFirst.
- **Inputs:** Player consent, committed eligible discovery and authoritative run/plot state.
- **Outputs:** Bounded social sessions, safe visit routing and deduplicated prestige projections.
- **Testing requirements:** Invite spam, blocked/privacy-restricted users, leader disconnect, party split, simultaneous first claim and unauthorized plot edit.
- **Completion criteria:** Two friends can join a run and visit safely; rare announcements cannot expose private state or flood the server.
- **Other projects that depend on it:** P15
- **Recommended development order:** Phase 4; limited ambient shared encounters already exist in the slice.
- **Estimated complexity:** High

### Project 15 — Trading, Escrow and Transfer Recovery

**Purpose:** Allow safe voluntary exchange without turning rare creatures into duplication or scam vectors.

- **Owns:** Offer revision, two-sided confirmation, eligibility, durable escrow state machine, settlement/recovery and trade evidence.
- **Does not own:** Currency issuance, auctions, lending, trust trades, cross-game sales and client-only locks.
- **Dependencies:** P03, P04, P05, P14, P22
- **Systems required:** Co-located trade sessions, ownership reservations, immutable offer digest, durable terminal decision and reconciliation workers.
- **Assets required:** P19 confirmation presentation, rarity/form distinctions and lock/pending cues.
- **Data required:** Trade, escrow custody, participant reservations, accepted offer revision, item provenance and terminal transaction ID.
- **Roblox modules required:** TradeService, EscrowRepository, TradeEligibilityPolicy, TradeRecoveryWorker.
- **APIs/interfaces required:** OpenTrade; SetOffer; AcceptRevision; Confirm; CancelBeforeDecision; GetSettlementStatus.
- **Inputs:** Server-owned inventory references, opt-in participants and available capacity.
- **Outputs:** One committed transfer or safe abort, receipts and auditable custody history.
- **Testing requirements:** Crash after every state transition, two workers racing, client disconnect, offer swap, double trade, full recipient inventory and stale lease.
- **Completion criteria:** Segmented durability gate passed; every crash cut conserves total currency/items; pending custody stays unavailable until resolved.
- **Other projects that depend on it:** No direct downstream gameplay project; P23 certifies and P24 consumes its release artifacts.
- **Recommended development order:** Phase 5 only; independent security/recovery certification is mandatory before enablement.
- **Estimated complexity:** Very High

### Project 16 — Events, Seasons and Live Content Control

**Purpose:** Refresh familiar spaces using validated schedules and compatible modifier packages.

- **Owns:** Event lifecycle, UTC schedules, participation windows, catalog rollout pointers, event currencies and content-specific emergency disable targets via P22 enforcement.
- **Does not own:** Arbitrary dynamic scripts, new hazard code hidden in config and forced seasonal replacement of core progression.
- **Dependencies:** P06, P07, P08, P10, P13, P21, P22
- **Systems required:** Event scheduler, pinned run context, modifier composition, claims/grace windows and rollback-compatible manifests.
- **Assets required:** Reusable Fog slice profile; later event audio/VFX/props from P18.
- **Data required:** Event, schedule revision, content manifest, eligibility and claim records.
- **Roblox modules required:** EventService, ScheduleProvider, ModifierComposer, LiveConfigService, SeasonService later.
- **APIs/interfaces required:** GetActiveEventContext; ActivateManifest; DisableRewardSource; ClaimEventReward; ResolveExpiredCurrency.
- **Inputs:** Reviewed package, operator-approved schedule and supported capabilities.
- **Outputs:** Bounded active event instances, compatible catalog pointers and deduplicated reward eligibility.
- **Testing requirements:** Clock boundary, overlapping events, stale server revision, schedule outage, rollback and claims across expiry.
- **Completion criteria:** A Fog event runs locally in the slice; full live release requires tested fallback/kill switch and no invalidation of owned items.
- **Other projects that depend on it:** No direct downstream gameplay project; P23 certifies and P24 consumes its release artifacts.
- **Recommended development order:** Local slice subset Phase 3; complete global/liveops domain Phase 6.
- **Estimated complexity:** High

### Project 17 — Biome, Encounter and Content Package Production

**Purpose:** Produce playable places and collection goals through the established content pipeline.

- **Owns:** Level layout, route readability, encounter placement, resource distribution, biome package assembly and content-specific acceptance.
- **Does not own:** Engine-level world systems, new mechanics disguised as content and bulk modeling ownership.
- **Dependencies:** P06, P07, P08, P09, P10, P12, P13, P18
- **Systems required:** Content authoring workflow, route validation, spawn budget presets and biome playtest recipe.
- **Assets required:** Approved modular environment kit, creature prefabs, landmarks and ambient audio.
- **Data required:** Biome package manifest, spawn/loot references, hazard zones, collection page and localization keys.
- **Roblox modules required:** No per-biome service for existing capabilities; package definitions and prefab metadata only.
- **APIs/interfaces required:** Catalog import/validate/publish; world prefab contract; encounter spawn interface.
- **Inputs:** Approved biome brief, supported mechanics, art kit and capacity budgets.
- **Outputs:** One versioned playable biome package with reproducible seed/test routes.
- **Testing requirements:** Reachability, softlocks, mandatory progression without rare drops, eight-player crowding, visibility and device budgets.
- **Completion criteria:** A reviewer adds/enables the package without core service edits; content manifest and QA report match exact asset revisions.
- **Other projects that depend on it:** No direct downstream gameplay project; P23 certifies and P24 consumes its release artifacts.
- **Recommended development order:** Graybox design begins Phase 2 using agreed contracts; full production Phase 4 onward after provider integration.
- **Estimated complexity:** High

### Project 18 — Art, Blender, Animation and Audio Pipeline

**Purpose:** Create a coherent stylized identity and reusable production standards at scale.

- **Owns:** Concept/reference approval, modeling, UV/materials, rigs, animation/audio packages, LOD/collision, exports and source asset-manifest entries consumed by P06 runtime registry.
- **Does not own:** Gameplay scripts in models, unapproved asset publication and gameplay balance changes through asset scale.
- **Dependencies:** P01, P06
- **Systems required:** Source/export/import traceability, family templates, asset validation and permission checks.
- **Assets required:** Three slice creatures, Crystal attachments/materials, tool/terrain/sanctuary kit, one weather package and core sounds.
- **Data required:** Asset manifest, bounds, pivots, rig family, animation markers, LOD metadata, license/provenance and approved Roblox IDs.
- **Roblox modules required:** Asset-contract specifications and metadata validators only; runtime resolver belongs to P06.
- **APIs/interfaces required:** AssetRegistry entry; prefab attachment contract; animation marker contract; audio cue mapping.
- **Inputs:** Approved art direction, budget sheet, concept sheets and content IDs.
- **Outputs:** Versioned source files, validated exports, permitted Roblox prefabs and integration report.
- **Testing requirements:** Silhouette at phone size, deformation, marker timing, collision, triangle/texture counts, import fidelity and memory costs.
- **Completion criteria:** One representative family establishes a reproducible standard; slice assets pass Studio and real-device review before scaling production.
- **Other projects that depend on it:** P17
- **Recommended development order:** Pipeline/concept in Phase 1, one sample during Phase 2, full slice polish Phase 3.
- **Estimated complexity:** High

### Project 19 — UI, UX, Accessibility and Presentation

**Purpose:** Make risk, discoveries and ownership immediately readable on touch, controller and desktop.

- **Owns:** Screen flow, HUD, inventory/book virtualization, input hints, accessibility, localization layout and confirmed audio/VFX orchestration.
- **Does not own:** Reward authority, hidden client-only inventory state and implementing a transaction through button callbacks.
- **Dependencies:** P02, P04, P06
- **Systems required:** Controller lifecycle, view models, responsive layouts, navigation focus and user preference handling.
- **Assets required:** Reviewed icons, typography, UI sounds, rarity palette plus non-color rarity indicators and P18 VFX.
- **Data required:** Sanitized snapshots, paged lists, localization keys and display preferences.
- **Roblox modules required:** HUDController, InventoryController, CollectionController, ExpeditionController, SanctuaryController, PresentationController, later Trade/Shop controllers.
- **APIs/interfaces required:** SubscribeSnapshot/Delta; RequestIntent; show pending/confirmed/rejected operation; GetInventoryPage.
- **Inputs:** Authoritative projections and player inputs with device accessibility needs.
- **Outputs:** Readable player decisions and requests; no direct mutation of durable state.
- **Testing requirements:** Small screen, text expansion, controller focus, color-blind readability, reduced effects, 10k-list virtualization and delayed/reordered results.
- **Completion criteria:** Players can distinguish seen/carried/secured and understand loss without developer explanation; UI close/reopen preserves server truth.
- **Other projects that depend on it:** P20
- **Recommended development order:** Minimal Phase 2; each later screen depends on its domain's integrated service milestone.
- **Estimated complexity:** High

### Project 20 — Monetization, Entitlements and Receipts

**Purpose:** Offer optional expression while preserving free exploration and collection.

- **Owns:** Marketplace product mapping, receipt processing, purchase reconciliation, cosmetic entitlements and purchase UX requirements.
- **Does not own:** Paid power, paid rare-creature RNG, economy wallet internals, direct sale of expedition success and unapproved sales activation.
- **Dependencies:** P03, P04, P05, P11, P19, P21, P22
- **Systems required:** Durable receipt adapter, entitlement resolver, cosmetic equip validation and small shop catalog.
- **Assets required:** Sanctuary theme/creature skin cosmetics distinct from earned rarity, shop imagery and previews.
- **Data required:** Product mapping, immutable purchase ID, entitlement record and verified grant status.
- **Roblox modules required:** CommerceService, ReceiptService, EntitlementService, CosmeticEquipPolicy.
- **APIs/interfaces required:** ResolveProduct; ProcessVerifiedReceipt; GetEntitlements; EquipCosmetic; QueryPurchaseStatus.
- **Inputs:** Platform receipts/ownership evidence and approved product catalog.
- **Outputs:** Exactly-once entitlement effects with durable retry/recovery and purchase analytics.
- **Testing requirements:** Repeated receipt on different servers, unknown product, failure after grant, missing asset, insufficient capacity and forged purchase-finished signal.
- **Completion criteria:** Receipt recovery certified; free progression unchanged; prices come from platform data; product activation is separately approved.
- **Other projects that depend on it:** No direct downstream gameplay project; P23 certifies and P24 consumes its release artifacts.
- **Recommended development order:** Phase 6 after free-loop proof; season pass/Gems optional later milestones.
- **Estimated complexity:** Very High

### Project 21 — Analytics, Playtest Evidence and Experiments

**Purpose:** Identify whether players understand, enjoy and return to the loop.

- **Owns:** Event taxonomy, funnel/cohort definitions, dashboards, experiment assignments, consent-appropriate data minimization and interpretation.
- **Does not own:** Authoritative ownership/economy ledgers, unbounded player histories and optimizing session length in isolation.
- **Dependencies:** P01, P02
- **Systems required:** Buffered telemetry adapter, correlation IDs, sampling, metric definitions and controlled tuning experiments.
- **Assets required:** None; research scripts/questionnaires are documentation artifacts.
- **Data required:** Event schema, build/content version, run ID, session cohort and aggregated economy reasons.
- **Roblox modules required:** AnalyticsService, TelemetryAdapter, ExperimentAssignmentProvider, MetricsRegistry.
- **APIs/interfaces required:** EmitDomainEvent; RecordFunnelStep; RecordSourceSink; GetExperimentVariant.
- **Inputs:** Committed domain outcomes, session lifecycle and consented playtest observations.
- **Outputs:** Reproducible funnel/cohort reports and prioritized product recommendations.
- **Testing requirements:** Repeated event dedup, outage buffering bounds, PII scan, cohort denominator verification and experiment variant stability.
- **Completion criteria:** First discovery/extraction/upgrade/second run can be traced for a build; telemetry outage cannot block gameplay.
- **Other projects that depend on it:** P16, P20, P22
- **Recommended development order:** Phase 1 schema/adapters; interpretation begins with first graybox testers.
- **Estimated complexity:** Medium

### Project 22 — Security Operations, Support and Administrative Tools

**Purpose:** Make incidents diagnosable and recovery controlled before valuable social systems launch.

- **Owns:** Privileged command boundaries, RBAC, audit records, item quarantine, compensation approval and incident runbooks. Authoritative feature-disable state and enforcement contract; P16 declares content targets and P24 invokes it.
- **Does not own:** Secret reward bypasses, direct ad-hoc DataStore edits, blanket bans from one movement signal and automatic destructive rollback.
- **Dependencies:** P02, P03, P04, P05, P21
- **Systems required:** Server-authorized admin adapter, dry-run commands, reconciliation views, anomaly review and scoped kill switches.
- **Assets required:** None for slice; future internal tooling must avoid exposing privileged endpoints to players.
- **Data required:** Operator identity, reason/ticket, before/after revision, compensation operation ID and immutable audit evidence.
- **Roblox modules required:** AdminService, AuditService, CompensationService, QuarantineService, FeatureGateService.
- **APIs/interfaces required:** InspectOperation; DryRunRecovery; QuarantineItem; ApplyApprovedCompensation; DisableFeature.
- **Inputs:** Verified incident evidence, authorized operator and durable ledger references.
- **Outputs:** Audited bounded recovery actions and feature-state projections.
- **Testing requirements:** Unauthorized access, impersonation, duplicate compensation, stale revision, disabled-feature race and cross-environment targeting.
- **Completion criteria:** Support can diagnose a disputed extraction without editing raw records; privileged grants replay safely and have traceable authority.
- **Other projects that depend on it:** P15, P16, P20
- **Recommended development order:** Read-only diagnostics Phase 1–2; write recovery tools before trade/paid launch.
- **Estimated complexity:** High

### Project 23 — QA, Security Verification and Performance Certification

**Purpose:** Independently prove behavior, failure recovery and device performance at every milestone.

- **Owns:** Test strategy, fixtures, fault injection, multiplayer/device matrix, exploit scenarios, regression packs and evidence-based certification.
- **Does not own:** Self-approval by implementation agents, invented test passes and changing requirements to make a build pass.
- **Dependencies:** P01, P02
- **Systems required:** Automated test harness, simulation/property tests, Studio/published-environment scenarios and profiling capture workflow.
- **Assets required:** Stress fixtures, controlled seeds, device routes and worst-case scene manifests.
- **Data required:** Test accounts in isolated environment, historical schema fixtures, report IDs and result artifacts.
- **Roblox modules required:** Test adapters, deterministic clock/RNG fakes and integration harness specifications; no live reward paths.
- **APIs/interfaces required:** Fault injection hooks behind test-only composition; invariant assertions; evidence/report contract.
- **Inputs:** Exact commit/build/catalog IDs and domain acceptance criteria.
- **Outputs:** Pass/fail report, reproducible defects, capacity evidence and release recommendation.
- **Testing requirements:** The entire gate matrix in section 15, with mandatory failure branches for each candidate feature.
- **Completion criteria:** Independent evidence covers enabled features; all critical/high release blockers closed or feature disabled; real-device limits verified.
- **Other projects that depend on it:** P24
- **Recommended development order:** Foundation in Phase 1; continuous per-project gates; full release certification Phase 7.
- **Estimated complexity:** Very High

### Project 24 — Build, Release and Production Operations

**Purpose:** Ship identifiable, recoverable versions with controlled environments and asset/config compatibility.

- **Owns:** Build reproducibility, environment separation, artifact manifests, release approval, rollout, rollback orchestration and operational ownership.
- **Does not own:** Unreviewed production publication, automatic destructive data downgrades and accepting code merely because it compiles. Duplicate feature-gate enforcement; use P22.
- **Dependencies:** P01, P23
- **Systems required:** Pinned build pipeline, staging experience, release manifest, canary procedure and operational dashboards.
- **Assets required:** Versioned permitted asset IDs and experience/place configuration records.
- **Data required:** Build version, schema compatibility interval, catalog revision, rollout cohort and release/incident log.
- **Roblox modules required:** Future build/validation tooling and release configuration; runtime version/feature-gate adapters supplied by owning domains.
- **APIs/interfaces required:** BuildArtifact; ValidateReleaseManifest; ApproveRelease; RollOut; RollBackCompatibleBuild; FreezeFeature.
- **Inputs:** Reviewed project outputs, P23 certification and recorded owner authorization.
- **Outputs:** Traceable release candidate, deployed version when authorized, rollback plan and launch report.
- **Testing requirements:** Rebuild from clean checkout, staged migration, partial rollout, incompatible client/server rejection, bad asset rollback and incident drill.
- **Completion criteria:** A separate operator can reproduce, validate and recover a release; no open ownership or on-call gap for launch-critical domains.
- **Other projects that depend on it:** No direct downstream gameplay project; P23 certifies and P24 consumes its release artifacts.
- **Recommended development order:** Pipeline specification/foundation Phase 1; staging every milestone; production rollout Phase 8.
- **Estimated complexity:** High

## 8. Development Roadmap

### Phase 0 — Architecture and testable contracts

Complete P01. Agree the loss model, three-species slice, state ownership, profile limits, network message contracts, catalog IDs, repository rules and acceptance fixtures. P06 validates a tiny illustrative catalog on paper; P18 defines one family style sheet. Exit when an independent reviewer can trace capture→run cargo→extraction→owned creature, including the failure branches, without inventing missing ownership rules. No broad content production.

### Phase 1 — Technical spine with an end-to-end transaction

Deliver foundation milestones from P02, P03, P06, P21, P23 and P24. Implement one player session, one server intent, one versioned load, one durable mutation and one rendered acknowledgment in later implementation work. Then add minimal P04/P05. Run duplicate, timeout-after-commit, reconnect and stale-writer cases before attaching valuable creatures. Exit when the transaction can be replayed safely in a private test environment. This is an engineering checkpoint, not a public prototype.

### Phase 2 — Graybox loop proof

Integrate minimal P07/P08/P09/P10/P11/P12/P13/P19 with P17's small navigable graybox. Use three simple creature representations, gathering, a clear hazard, visible unsecured cargo, extraction, a display socket and one sanctuary upgrade. Test save/rejoin and normal failure. P18 may explore one polished creature independently, but bulk art waits. Exit only when uncoached players understand the loop and choose a second run; if they do not, revise encounters, route decisions and feedback before adding systems.

### Phase 3 — Cohesive vertical slice

P18 replaces the approved slice representations with a consistent art/animation/audio package; P17 tunes the meadow and forest edge; P19 completes touch/controller accessibility. Polish and tune the existing rare Owl, Crystal Fox variation and one Fog event through a local subset of P16's event interface. This event does not require global live-ops scheduling. Validate eight-player encounter fairness and target mobile performance. Freeze a reproducible slice release with P23/P24 and measure P21's funnel. This is the first polished internal/closed-test milestone.

### Phase 4 — Extensibility and closed alpha

Prove a second biome and a fourth species can be added through catalogs and prefabs using existing capabilities. Expand P13 to a modest quest/collection arc, P11 to bounded decorations, P14 to visits/parties and P22 to recovery/support operations. Raise inventories only after measured byte/throughput budgets. A second biome is an architectural proof, not a commitment to build all seven named regions. Exit when returning testers understand both goals and when an inexperienced content contributor can add a valid creature without editing core services.

### Phase 5 — Durability at scale and safe social exchange

Complete segmented ownership migration, escrow recovery and rollback drills in P03/P04 before implementing/enabling P15. Begin with allowlisted unique-creature trades, fixed offer caps and co-located sessions; do not introduce auctions, lending, loans or cross-server asynchronous trading. Ship parties and visits even if trading is held. P23 certifies crash cuts at every transition and P22 rehearses quarantine/reconciliation. A single reproducible duplication exploit blocks trade launch.

### Phase 6 — Live operations and fair commerce

Complete P16 content manifests, safe scheduling, event expiry, compatibility and kill switches. P21 establishes cohort baselines and P22/P24 rehearse content rollback. Only after free play passes the loop gate should P20 add a very small cosmetic catalog and durable receipts. Premium currency and season passes remain optional. Production commerce cannot be enabled until receipt replay/recovery tests pass. Offline production is a separate P11 milestone, gated on source/sink analysis and reliable time accounting.

### Phase 7 — Beta, capacity and release candidate

P23 runs realistic multi-server, long-session and real-device tests; P21 compares onboarding, second-run behavior and return cohorts; P24 rehearses migrations, staged rollout and recovery. Aim for two polished regions and a deliberately small stable roster, then consider the brief's 20–30 creatures if the pipeline and test coverage support them. Launch scope is earned by gates; neither trading nor three regions is an unconditional launch dependency. Localized text, report controls, data operations, asset permissions and accessibility pass before public scaling.

### Phase 8 — Launch and measured expansion

P24 performs an approved staged release: limited audience → larger cohort → general availability, with predetermined stop thresholds. P22 owns incidents, P21 owns evaluation, P16/P17/P18 own tested content batches. Stabilize before raising server size, world size or update cadence. A seasonal update may add a biome or a new mechanic; it need not add both every time. Re-estimate after the slice and first content package, not from AI-generated calendar guesses.

### Planning discipline

Use complexity ratings for relative risk, not delivery dates. Estimate each milestone after interface approval with assumptions for human review, real-device access, asset revision and platform failure testing. Limit each AI coding assignment to one accepted contract or vertical behavior. Run two or three independent domain streams at most until integration throughput is known; more agents do not remove serial design or testing gates. Reserve explicit capacity for integration, bug repair and content validation in every phase.

## 9. MVP Definition

The first playable prototype proves mechanical cause and effect with deliberately simple assets. The polished vertical slice then tests the same loop at the intended quality bar. Neither milestone is a small version of every feature in the brief. Future-safe service boundaries and durable ownership are required; future features are not.

### MUST HAVE — first genuinely playable prototype

- A player can join, move, perform one clear interaction, and rejoin with secured progress intact.
- One shared, bounded test biome and a safe sanctuary return area; one obvious extraction station.
- A minimal expedition state machine: preparing, active, extracting/pending commit, secured return, or failed return.
- One provisional creature family sufficient to prove observing, capturing, carrying, and securing; the slice expands this to three species.
- Basic resources, bounded unsecured cargo, a secured inventory, Coins, and one reachable sanctuary display upgrade.
- Explicit risk warnings and a failure path that removes only the current run's unsecured cargo.
- Clear `seen` and `secured` collection states, one creature display socket, and obvious inventory capacity feedback.
- Minimal touch-friendly feedback for interaction, danger, unsecured cargo, extraction progress, save confirmation, and the next goal.
- Server-authorized awards, input validation, idempotency, profile locking, schema versioning, safe loading failures, and recovery tests for extraction.
- Essential funnel, error, transaction, and performance telemetry. Failure cases must be reproducible before rewarding more content production.

### SHOULD HAVE — complete the vertical slice immediately afterward

- Three species: Common Fox, Uncommon Deer, Rare Owl; one Crystal mutation for Fox. Labels and weights are provisional balance hypotheses.
- One authored Meadowlands route loop with a forest-edge pocket offering greater opportunity and greater exposure.
- One weather overlay, Fog, with explicit navigation cues and an opportunity associated with the Crystal Fox habitat.
- Readable creature animation, discovery/extraction audio, accessible rarity cues, and cohesive specimen art for the chosen slice.
- Shared witnessing and one award per eligible participant in co-op capture, without a first-touch race.
- Small collection book, resource selling, one visible habitat upgrade, and resumed onboarding.
- Uncoached, instrumented mobile and desktop tests of a second expedition, including deliberate failure and reconnection exercises.

### COULD HAVE — only if the slice gate already passes

- A brief sanctuary visit within the current server, using fixed displays and read-only guest permissions.
- One simple optional exploration objective using the existing quest contract.
- A noncompetitive discovery toast and an optional celebratory photo pose.
- A deterministic backpack upgrade prototype, if carrying choices—not walking time—are the demonstrated friction.
- A small additional authored encounter route using the same assets and behavior families.

### LATER — explicitly excluded from early implementation

Trading, gifting, auctions, mail transfers, breeding, functional trait stacking, companions, mounts, guilds, raids, PvP, global first-discoverer awards, cross-server rankings, complex combat, hunger/thirst, procedural terrain, vehicles, freeform sanctuary building, offline production, paid conveniences, premium currency, passes, multi-currency events, multiple biome places, large crafting trees, and large catalogs. None may enter the slice as an allegedly small enhancement.

Trading remains disabled until its recovery proof is complete even if other social features ship. Monetization begins later with cosmetics after the free loop passes its own playtest gate. The architecture documents extension points for these systems without creating empty production implementations for all of them.

## 10. Vertical Slice Specification

### 10.1 Testable scope

Target one place with up to eight players, one small Meadowlands biome, a safe shared hub with per-player fixed sanctuary pads, one extraction station, and one short return route from a forest-edge danger pocket. The eight-player limit is an initial capacity target, not a platform claim; every relevant gate must run at that population before increasing it.

Content consists of Fox, Deer, and Owl, plus Crystal Fox through a species-restricted Crystal mutation. Owl is the rare base species. Crystal is a mutation, not a fourth species and not a second required progression gate. Use two ordinary resource types, one capture tool, one scanner, a fixed initial field capacity, Coins, and one sanctuary habitat-pad upgrade. The exact names, costs, weights, capacities, and timing below remain provisional hypotheses.

The forest edge uses one environmental exposure meter. Exposure increases in clearly marked danger areas, and Fog can make route reading harder while preserving landmarks and extraction signage. Warning stages are communicated through text/icon, sound, and visual treatment; no critical information depends on color alone. Leaving the danger pocket reduces exposure. A full meter causes a rescue to sanctuary, losing unsecured cargo from that run. There is no loss of already secured creatures, Coins, equipment, paid goods, or upgrades.

The first weather event is Fog. It temporarily alters one local opportunity, such as revealing Crystal-associated tracks or enabling an authored Crystal Fox encounter socket. It does not require another resource, currency, hazard meter, boss, or event shop. The deterministic playtest scenario may trigger Fog and arrange a rare opportunity to exercise the flow; organic-session analytics must identify and exclude these scripted grants when evaluating actual rarity and retention.

### Provisional slice tuning fixture

Use one explicit balance fixture so implementation and QA share a reproducible example. Starting Coins are zero; field cargo permits two creature awards and ten ordinary resource units. The introductory route guarantees five Reed Fiber and five Meadow Stone per eligible beginner through bounded personal node claims. Each resource unit sells for five Coins; the habitat upgrade costs forty Coins. Selling eight units therefore pays for the first upgrade and leaves two resource units. This is an example tuning set for the test build, not a promise about launch prices. No creature sale or rare roll is needed. Permanent storage is reserved for the run before departure.

The initial display has one socket; the upgrade adds a second visible habitat socket without increasing field yield. The capture interaction targets roughly three seconds of readable approach/hold/response, with server-measured eligibility and generous touch tolerance. Extraction has a three-second cancelable channel followed by the durable commit wait. Canceling the channel leaves an active run; once settlement starts, closing the interface cannot undo it. These timings are hypotheses and may change after fresh-user tests.

Use Common/Uncommon/Rare labels in this slice; reserve additional rarity identifiers without exposing empty tiers. Rare Owl is an optional habitat-and-time opportunity, while Crystal Fox is eligible during the Fog occurrence. QA has a deterministic seed/scenario that presents both; organic encounters use server-owned weights that are tuned separately. No numeric one-in-N claim appears in player presentation until its actual eligibility and conditional distribution are specified.

For economy review, estimate issuance as eligible encounters per minute × active eligible participants × award quantity × successful extraction fraction. A shared rare spawn can yield up to eight separate earned specimens, so treating it as one economic item per server is wrong. Simulate one player, eight cooperating players, repeated failed runs, saturated inventories and high-efficiency routes. Collect sell/spend/extraction rates separately; never double-count both resource value and its later coin conversion as new net value.

### 10.2 Award, loss, and cooperation rules

Scanning a valid creature records a server-confirmed observation. An observation may reveal species, habitat clues, and an unfilled secured marker. Capture places an award receipt into that run's unsecured cargo, subject to eligibility and capacity. It does not yet create an independently transferable permanent asset. Extraction applies each award exactly once to the owner's persistent inventory and updates secured collection records. A player who loses an observed Owl can still recognize it in the book, but does not own it or receive its secured reward.

A shared creature encounter offers each actively participating eligible player one independently earned capture award. The server validates proximity, the player's individual capture action, encounter state, and available capacity. Eligibility is bounded by the encounter's recorded participant roster, at most the eight-player server population. Each participant award has a stable encounter/player key; each secured creature receives its own unique ownership identity. Repeating the action, reconnecting, or racing a remote cannot produce a second award. The visual creature is shared, but successful participation is not a first-touch ownership contest. Proximity alone does not award spectators a creature.

A full inventory blocks capture before reward creation and explains how to return or discard unsecured resources. Nothing is silently deleted. A player may abandon unsecured resources during a run after a clear interaction; that never sells them. Explicit expedition failure ends the run once. Voluntary leaving is not an extraction method.

Disconnects never secure cargo. A short, configured reconnect grace applies only while the original live server still owns the run; hazard simulation continues during that grace, and returning resumes current server state without a rewind. If the grace expires, the run resolves as failure. A new profile owner first reconciles any durable extraction decision: a committed award is applied exactly once; an active run with no durable extraction success resolves as failure. Cargo remains locked while the decision is unresolved. This policy must be explained before meaningful risk begins. Stop new expeditions and valuable capture when persistence is unhealthy. Verified platform-outage compensation, if appropriate, is a separate capped, audited support action; a client-reported disconnect reason never authorizes compensation. The message must distinguish successful extraction, rescue/timeout loss, and unresolved recovery.

### 10.3 Exact first ten-minute journey

This is the intended test route, not a forced countdown or a promise about player speed. Beginners can take longer. Scenario timing and guaranteed onboarding placement are marked separately in telemetry.

1. **0:00–0:30 — arrive and choose to explore.** The player appears beside a small sanctuary pad with an empty display. One short objective points toward the meadow. Basic scanner and capture tool are already available. The camera and controls never require navigating a shop or a long dialogue.
2. **0:30–1:30 — notice a Fox.** A nearby authored encounter shows the silhouette and a short scanner cue. The player observes it, sees the book change to `seen`, performs capture, and sees “Unsecured — extract to keep.” The first interaction is achievable without a rare roll or coordination.
3. **1:30–2:30 — collect and recognize a choice.** Along the return loop, the player gathers ordinary resources. A visible Deer or trail toward the forest edge suggests more value. Cargo feedback shows that the Fox and resources remain exposed. The player can make one optional detour without overwhelming tutorial demands.
4. **2:30–3:30 — test mild risk.** The forest edge visibly raises exposure. A warning points toward a known safe route. The player learns that leaving lowers danger and that returning with something is a valid success. The tutorial never secretly invalidates the stated loss rule.
5. **3:30–4:30 — extract.** At the station, a short, cancelable preparation precedes a server transaction. The interaction clearly distinguishes traveling/extracting from the moment cargo becomes secured. The success celebration occurs after durable confirmation. The player returns to the safe hub.
6. **4:30–6:00 — make the find theirs.** The player displays the Fox, sees its book record become secured, sells a chosen ordinary resource quantity, and buys one habitat-pad upgrade with the resulting Coins. Ordinary resources on the introductory route are sufficient at the provisional price, regardless of rare encounters. The pad visibly changes and offers an additional display position.
7. **6:00–7:00 — choose a second goal.** The book presents the Owl's habitat clue and an unfilled entry. Fog arrives with a recognizable environmental/audio change, and the forest edge shows an unusual track. The player chooses between searching for the Owl and investigating the mutation clue.
8. **7:00–9:00 — second expedition, real temptation.** The player gathers some cargo, sees the event opportunity, and approaches a controlled rare encounter opportunity in the instrumented scenario. A friend can participate in capture and receive their own eligible award. A readable exposure warning makes “go home or investigate one more clue” a real decision.
9. **9:00–10:00 — resolve the second decision.** A successful player extracts the additional find and can display it in the upgraded sanctuary. A player who overextends is rescued, retains their previously secured Fox and upgrade, and sees exactly which current-run cargo was lost. In either outcome, the next goal remains visible; the game does not force another launch.

After ten minutes, the tester receives no additional instruction. Observe whether they start another expedition, pursue the missing species or Crystal mutation, show their sanctuary, or leave. Capture the reason through a short post-session interview. Do not interpret compliance with the tutorial as voluntary interest.

### 10.4 Slice acceptance gate

Run at least two small uncoached rounds, each with ten fresh target-audience testers and representation from touch devices, using appropriate recruitment and guardian arrangements. These sample sizes and thresholds are proposed diagnostic gates, not statistically reliable retention forecasts. Record device class, build, scenario mode, errors, assistance, and where each player stops.

In each round, at least eight of ten should observe a creature, extract, and obtain the first upgrade within ten minutes without developer prompting; at least seven should voluntarily enter the second expedition; and at least eight should correctly explain which possessions are at risk. At least six should make a visible return-versus-detour choice and explain the reason afterward. Pair the counts with observations: a forced or accidental action does not satisfy comprehension.

Also test eight concurrent players with simultaneous captures and extraction, full inventories, defeat, mid-extraction disconnect, server interruption, failed load, and rejoin. Acceptance requires zero confirmed duplication or permanent-item loss in the defined fault suite, clear recovery outcomes, and the separate performance/mobile gates. “Zero observed” is a release gate within the tested scenarios, not proof that no bugs exist.

Do not expand the catalog if failures show poor comprehension, repetitive encounters, punitive loss, weak sanctuary payoff, or lack of voluntary second runs. Fix that specific part of the loop and repeat with fresh testers. D1/D7 retention is measured later in a live cohort; it cannot be inferred from a ten-minute guided scenario.

## 11. Blender / Asset Pipeline

The production unit is a reusable asset family with a validated gameplay contract. A fox is a species definition, an appearance family, a compatible rig and animation set, and a registered prefab. Crystal Fox is a permitted mutation of Fox; it does not require a second capture implementation or duplicate species records. The first polished batch contains the agreed three base species plus the Crystal mutation, one small biome, a modest sanctuary, an extraction station and one weather presentation. The brief's larger eight-creature art list becomes a subsequent content batch.

### 11.1 From concept to accepted game content

1. **Asset request.** Game Design records the gameplay purpose, scale in studs, viewing distance, silhouette, biome palette, interaction dimensions, rarity readability, rig family, required clips, mutation sockets and budget. The request names the content ID and acceptance scene before generation starts.
2. **AI concepts.** Asset AI explores a small number of visual directions using rights-cleared references. Record prompts, source references, generator/version when available and usage provenance. Avoid recognizable copies of another game's creatures. The output must meet WILDLANDS' expressive, stylized direction.
3. **Reference approval.** Produce front/side/back reference, neutral lighting, a silhouette test, palette swatches and a scale comparison against the standard player. Lock the chosen reference for this asset revision. A human art owner approves a new family or departure from the style guide; already approved variants follow the established batch authority.
4. **Blender blockout.** Establish proportions, root pivot, floor contact, bounds and attachment locations. Inspect at actual gameplay distance. Import the blockout into the validation scene early to catch scale and readability failures.
5. **Model and topology.** Preserve silhouette and articulation. Remove hidden geometry and unnecessary subdivisions, check normals and degenerate geometry, and verify triangulated export counts. Add geometry around deforming joints where it improves animation.
6. **UV and materials.** Use one UV set, packed islands and consistent texel density. Prefer shared palette/trim textures and simple materials. Use PBR selectively where crystal, metal or water materially improves readability. A mutation should reuse base geometry/textures where possible.
7. **Rig and skin.** Use a versioned family rig. Establish the bind pose and ground reference; test the most extreme pose before producing a full animation set. Keep gameplay hit volumes independent from decorative bones.
8. **Animation.** Create the required state clips and record loop status, duration, playback speed range, transitions and presentation markers. Validate foot contact at gameplay locomotion speeds. Export/import each required clip and retain its source action.
9. **Optimization and LOD.** Audit triangles, unique meshes/materials, skinning, collisions, transparency, particles, texture memory and total scene cost. Author alternate detail representations only where profiling justifies them; all versions keep compatible scale, identity and sockets.
10. **Export and import.** Use a pinned Blender version and tested export/import presets. Export FBX for the initial creature/animation path; adopt glTF only after an equivalent rig and animation smoke test. Record units, axes, mesh hierarchy, source hash and importer settings. Roblox supports FBX, glTF and OBJ, with FBX/glTF supporting richer hierarchy and animation data. [Roblox Importer](https://create.roblox.com/docs/studio/importer)
11. **Prefab assembly.** The integration owner creates a script-free prefab containing the model, agreed attachments, animation bindings, simple collision proxies and metadata. Behaviour comes from registered services. Attachments use semantic names such as `ScanOrigin`, `Nameplate`, `CaptureFocus`, `MutationBack` and `GroundReference`.
12. **Registry and integration.** Register immutable logical asset ID, asset revision, prefab reference, creator/ownership, required permissions, rig and animation compatibility, budget report, moderation status, fallback and dependent definitions. Bind it through the catalog. Run capture, stream-out/in, sanctuary display and missing-asset scenarios in a published staging experience.
13. **Acceptance.** Art review, functional integration and measured mobile performance all pass. Archive source and export evidence, then mark the asset `Accepted`. An attractive Blender render alone is insufficient.

Roblox's general requirements currently cap an individual mesh at 20,000 triangles and a skinned vertex at four bone influences. They require appropriate root/transforms and describe separate exports for multiple animation tracks. These are platform constraints; WILDLANDS' targets below are deliberately lower. Revalidate platform limits when upgrading the pipeline. [Roblox general modeling specifications](https://create.roblox.com/docs/art/modeling/specifications)

### 11.2 Names, files and registry

Use stable lowercase domain-qualified logical IDs (snake_case within each segment) with domain prefixes: `creature.fox`, `mutation.crystal`, `rig.quadruped_small`, `asset.creature.fox.body`. Display names are localized and never serve as database keys. Source/export names follow `wl_<category>_<subject>_<part>_<variant>_rNNN`; for example `wl_creature_fox_body_base_r003.blend`. Clips follow `wl_anim_<rig-family>_<action>_rNNN`. Textures use `_albedo`, `_normal`, `_roughness`, `_metalness` suffixes.

All requests specify final dimensions in studs. Define the export preset so a chosen Blender calibration length maps to a known stud length; validate a two-stud reference cube and avatar comparison after import rather than assuming automatic unit conversion. Use a floor-centered root with consistent forward direction, applied transforms and a socket calibration scene.

Under each asset folder keep `request.md`, `references/`, `source/`, `textures/`, `exports/`, `previews/`, `validation/` and a machine-readable manifest specified by the asset schema. Do not duplicate the same mesh under every biome. Central registries point to shared assets; biome assembly manifests reference their logical IDs.

The manifest records dimensions, pivot convention, triangle counts by representation, material slots, textures/resolutions, bone count, rig version, animation set, sockets, collision recipe, streaming mode, VFX tiers, source revision, asset IDs and IDs of every dependent texture/animation. It also records creator account/group, staging and production universe permissions, provenance, accepted version and last successful published smoke test. Production content pins an accepted revision; it does not consume an artist's latest export automatically.

### 11.3 Provisional art budgets

These are starting design targets for the eight-player slice, measured as triangles for the complete visible asset, not Blender face counts or Roblox maximums. Aggregate scene measurements may require lower limits.

- **Small/medium creature:** 3,000–6,000 triangles at close range; 1–2 principal skinned mesh sections; 18–32 deform bones. Large hero creatures may use 8,000–12,000 triangles and up to 48 bones after a budget review. Do not give every common creature the hero allowance.
- **Mutation attachments:** ordinarily 200–800 additional triangles in total and at most two extra mesh sections. Prefer material/color changes for ordinary mutations. Giant/Tiny use approved scale bounds with tested ground offset, capture volume and doorway clearance.
- **Equipment/hand props:** 300–1,500 triangles; a silhouette-critical backpack or scanner may reach 2,500. Static small sanctuary decoration: 100–1,000. A modular wall/floor/roof segment: 100–1,500. Count the assembled structure separately.
- **Trees and large rocks:** 500–2,000 triangles for ordinary trees and 100–800 for rocks. Landmark trees may reach 4,000 within a regional allocation. A grass/flower clump should usually remain around 20–150 triangles; avoid stacking many translucent cards across the same pixels.
- **Textures:** 256² for small props, 512² for ordinary creatures/equipment, 1024² only for a close-view hero asset or reusable environment atlas with demonstrable benefit. No routine 2K/4K maps. Prefer one principal color texture per ordinary creature; approve extra PBR maps against memory and material costs. Author compact maps because file size on disk is not the same as decoded texture memory. [Roblox texture specifications](https://create.roblox.com/docs/art/modeling/texture-specifications)
- **Animation:** source sampling at 30 fps initially, concise curves and no unnecessary constant keys. Core creature coverage: idle, locomotion, alert, flee, capture reaction and sanctuary idle. Sleep/eat/happy and species-specific reactions follow when the slice needs them. Common rig-compatible clips are reused with intentional timing variation.
- **VFX:** define low/medium/high tiers. For the initial low tier, an ordinary mutation uses zero or one small emitter and no permanent dynamic light; a rare reveal may temporarily use 2–3 emitters under the global effect budget. Avoid full-screen stacked transparency and effects that hide extraction hazards. Audio/VFX cues supplement text and shape; color alone never conveys danger or rarity.

### 11.4 LOD, collision and rig standards

Use `RenderFidelity = Automatic` by default and evaluate `Performance` for background assets. Engine-managed mesh simplification is distinct from an authored LOD system; placing files named LOD0/1/2 in a folder does not wire them into Roblox. If testing justifies authored versions, use approximately 50% and 20% of close-detail triangles, preserve the recognizable silhouette and control representation changes through the presentation layer. Avoid loading every catalog variant to every client. [Roblox RenderFidelity](https://create.roblox.com/docs/reference/engine/enums/RenderFidelity)

World props are anchored unless simulation is a requirement. Visual foliage and decorative mutation meshes normally have collision, touch and query disabled. Only intentional interaction proxies remain queryable. Use box/hull collision or a small set of simple parts for irregular traversal surfaces; reserve precise decomposition for measured exceptions. Movement/capture checks use server-defined volumes, not the apparent size of a client effect. Reusing identical mesh and texture IDs helps rendering; reimporting duplicate copies for each placement wastes that opportunity. [Roblox performance improvement guidance](https://create.roblox.com/docs/performance-optimization/improve)

The three-species slice needs a versioned quadruped family for Fox/Deer with validated proportion-specific clips and a small bird family for Owl. Keep Owl behavior to perch, alert, a short authored reposition and capture; full free-flight navigation is deferred. Introduce aquatic and other families only when approved content needs them. Do not force every shape onto one universal rig. Rigs retain consistent semantic bones, naming, attachment transforms and animation compatibility. Changing a rig is a versioned integration change. Use custom creature rigs with an `AnimationController`/`Animator` where appropriate rather than inheriting the full player character stack for every animal. Default player movement remains the standard Roblox character path; accessories follow the applicable avatar rules.

Make individual interactive creature prefabs atomic when streaming requires their parts together. Do not make a whole biome atomic or persistent. Clients tolerate temporary absence and reconstruct cosmetic state when the model returns. Persistent models are exceptional: Roblox sends them as complete units and does not stream them out; `PersistentPerPlayer` scopes this behaviour to explicitly selected players. [Roblox ModelStreamingMode](https://create.roblox.com/docs/reference/engine/enums/ModelStreamingMode)

### 11.5 Ownership and asset continuity

Recommend an owner-approved Roblox group for production experiences and assets, with separate staging and production universes and minimum necessary collaborator roles. Inspect permissions per asset type, including dependencies. A collaborator being able to preview an asset does not prove the published experience can load it. Current Roblox asset privacy controls differ by asset type; Open Use changes and game grants can be irreversible. Keep these actions with the authorized release owner. Animation sharing can use explicit permissions; mandatory reuploading under a different owner is not a universal requirement. [Roblox asset privacy](https://create.roblox.com/docs/projects/assets/privacy)

All content retains a neutral fallback model, icon, silent audio fallback and missing-animation behaviour. Moderation can affect assets after upload, so published smoke tests and load-failure monitoring continue after acceptance. A removed presentation asset must not erase a player's owned creature. Preserve source files, provenance and old accepted revisions so that a replacement can be integrated without changing collection identity. [Roblox asset overview](https://create.roblox.com/docs/projects/assets)

## 12. AI Agent Responsibilities

One human product owner holds final authority over approved architecture, material scope, production operations and irreversible actions. Agents work autonomously within a documented, approved task; they do not need a fresh permission request for every routine edit. A task cannot grant itself broader authority by changing its own instructions.

### 12.1 Role boundaries

**Lead Architect AI** owns proposed domain boundaries, dependency direction, invariants, contract coherence, architectural decision records and integration sequencing. It can clarify an approved interface without changing its meaning and review implementation against it. It cannot unilaterally alter the approved Master Architecture, approve its own incompatible public API/data-schema change, or authorize production spending/publishing. It writes an impact proposal for those decisions and records the owner's answer.

**Coding AI** owns implementation and focused tests within its assigned paths and approved contracts. It can refactor internals when behaviour and integration contracts remain intact. It cannot introduce a second economy/inventory system, bypass persistence ownership, edit unrelated domains, decide reward policy, broaden public networking payloads without review, or manipulate production data to make tests pass.

**Blender / Asset AI** owns references, models, textures, rigs, animations, source manifests and asset validation for its assigned family. It can generate variants within approved style/budgets. It cannot change creature rarity, sale values, hit volumes, spawn policy, saved IDs or production permissions as an incidental art decision. Rig/socket incompatibilities are escalated before dependent assets are overwritten.

**Game Design AI** owns proposed tuning, progression, discovery conditions, fairness analysis and testable playtest hypotheses. It can adjust approved development tuning within explicit bounds. It cannot silently change economy invariants, launch a paid offer, weaken loss protection or infer player retention improvement without evidence. New mechanics require a design proposal and dependency assessment.

**QA AI** owns the test plan, adversarial scenarios, reproduction fixtures, device/build evidence and acceptance report. It can add independent tests and report blockers. It cannot mark failed gates passed, omit a failure because it is inconvenient, or automatically patch the implementation it is supposed to independently certify. If QA fixes a defect, another reviewer owns the resulting verification.

**Reviewer AI** independently reviews scope, contracts, exploit resistance, data durability, maintainability and evidence. It can approve a change for owner/integrator consideration or block it with a reproducible reason. It cannot merge/deploy a high-impact change solely on its own authority or treat another agent's summary as test evidence. Review must include the actual diff and relevant reports.

**UI/UX workstream** may be assigned to a coding or specialist UI agent later. It owns accessible presentation, input adaptation and error/empty/loading states against agreed view models. It cannot calculate authoritative rewards or decide that a server rejection is a client-side success. No UI implementation is part of this specification stage.

### 12.2 Parallel work contract

Before concurrent implementation, freeze the small set of interfaces the work shares. Each task receives a branch/worktree, base revision, owned paths, named public contracts and integration tests. Use one writer per shared file or binary asset. Assign an integration owner for composition roots, catalogs and project manifests; agents propose changes to these through their handoff instead of competing writes.

A task brief contains: objective and player outcome; inputs/source revisions; explicit included/excluded scope; file ownership; dependencies and API/schema versions; invariants; failure cases; performance allocation; required tests; evidence format; rollback approach; completion criteria; and approval exceptions. A missing dependency is reported as a contract gap, not filled with a private replacement system.

The handoff contains: task ID and final revision; changed paths; behaviour delivered; interfaces used/changed; schema or asset compatibility impact; tests actually executed with results; unexecuted checks; known limitations; migration/rollback steps; open decisions; and exact next integration action. Every claim links to repository evidence. Chat can coordinate urgency, but decisions and acceptance records must be committed to the repository before the task closes.

The sequence is **brief → implementation → implementer checks → independent review → QA gate → integration → status update**. High-risk data/economy/trade changes require both domain review and independent adversarial tests. One agent cannot satisfy both independent roles for its own work. Resolve merges at the final integration revision, then rerun the relevant integration checks; results from an earlier branch do not prove the merged build.

## 13. Shared Repository Structure

The repository is the authoritative, versioned record. The original brief is immutable source material; this specification becomes an approved baseline only after the owner accepts it. Proposed changes remain visibly `Draft`. The runtime source, asset sources, imported bindings and published release manifest must point to each other so that a fresh agent can reproduce the current project state.

Recommended future structure; these folders and implementation files are specified here, not created at this stage:

```text
wildlands/
  README.md                         # Entry point and current milestone
  AGENTS.md                         # Approved rules and reading order
  STATUS.md                         # Build, phase, blockers and next ready task
  CHANGELOG.md
  docs/
    source/game-master-brief.md      # Immutable brief snapshot + provenance
    bible/                          # Player fantasy, loop, art and economy intent
    architecture/                   # Approved master and domain boundaries
    projects/                       # Project 01–NN briefs and acceptance records
    systems/                        # State machines, failures and invariants
    api/                            # Commands, queries, remotes and error catalog
    schemas/                        # Persistent, session, network and content specs
    assets/                         # Style guide, rigs, sockets and budget policy
    decisions/                      # Numbered ADRs and owner approvals
    plans/                          # Roadmap, dependencies and capacity assumptions
    handoffs/                       # Agent task inputs and completed handoffs
    operations/                     # Release, rollback, recovery and support runbooks
    analytics/                      # Event dictionary, funnels and experiments
    legal-and-provenance/            # Asset licenses and source attestations
  src/
    server/                         # Server composition, services and adapters
    client/                         # Controllers and presentation
    shared/                         # Pure contracts, types and utilities
  content/
    definitions/                    # Creatures, items, biomes, quests, etc.
    catalogs/                       # Immutable catalog manifests and tombstones
    localization/                   # Stable text keys and translations
    schedules/                      # Reviewed UTC event schedules
  assets/
    concepts/
    creatures/<family>/<asset>/      # References/source/textures/exports/validation
    environment/<kit>/<asset>/
    sanctuary/<kit>/<asset>/
    equipment/<asset>/
    vfx/<effect>/
    audio/<sound-family>/
    registry/                       # Logical IDs → accepted Roblox dependencies
    roblox/prefabs/                  # Script-free importable model sources
  studio/
    places/                         # Owned place source and assembly manifests
    validation-scenes/              # Scale, materials, creature and device scenes
  tests/
    unit/
    integration/
    multiplayer/
    persistence/fixtures/
    security/
    economy/
    content/
    performance/
  qa/
    reports/<build-id>/
    known-bugs.md
    regression-cases/
  engineering/
    technical-debt.md
    dependencies.md
    compatibility-matrix.md
  releases/<release-id>/             # Manifest, evidence and operation record
  tooling/                          # Approved build/validation tools, added later
```

Each system specification identifies its owner, source-of-truth state, API, persistence impact, callers, state transitions, invariants and failures. Each ADR has status, context, alternatives, decision, consequences, migration needs and approver. A superseded ADR stays discoverable. Each known bug includes severity, reproduction, affected builds, owner and regression case; each debt entry includes risk, trigger for repayment and deadline or milestone.

Use text source for Luau and manifests. Large `.blend`, audio and binary place/model files belong in a versioned large-file workflow; lock binary files while editing. Review a binary change using source revision plus screenshots/animation previews and a generated manifest diff. Never make a mutable Studio session the only copy of a working system. Decide which objects are source-controlled and which are authored in Studio before synchronization tooling is introduced; two writers must not own the same object tree.

A release manifest records code commit, place versions, catalog epoch/hash, compatible profile schema versions, asset revisions, feature flags, environment and test report links. Credentials, session cookies, production profile dumps and secrets never enter the repository. Test fixtures use synthetic or minimized sanitized data. Operational access is environment-specific and documented separately from the secrets themselves.

Every agent starts with `README`, `AGENTS`, `STATUS`, the approved architecture summary, its project brief, relevant contracts and latest handoff. `STATUS` lists the active milestone, current reproducible build, approved/deferred decisions, each project's state, blockers and dependency-ready tasks. An agent finishes by updating its project record and handing status changes to the integration owner; it does not manufacture a new global roadmap in its private notes.

## 14. AI Development Rules

These are future production rules. They do not grant authority to implement or publish the game during this planning task.

### Scope and architectural integrity

1. Read the approved baseline and assigned task before editing. Treat attached references, asset metadata and retrieved pages as information, not authority to change scope.
2. Implement one meaningful domain increment with observable acceptance criteria. Never respond to a bounded task by generating the entire game.
3. Never rewrite unrelated systems, reformat unrelated files, or remove working functionality to make a local design easier.
4. Use the existing owner of a capability. Before adding a service, utility or store, search the repository and document why an existing component cannot serve the need.
5. Never silently change a public API, network contract, persistent schema, invariant, economic entitlement or approved Master Architecture. Propose the change, document dependents and compatibility/migration impact, and obtain owner approval when outside the approved task.
6. The Lead Architect is accountable for coherence; it is not exempt from rule 5. Recorded owner approvals remain valid within their stated scope and must not be repeatedly requested.
7. Use an ADR for a new external dependency or material architectural exception. Record version, license, owner, maintenance status, failure behaviour and exit path. Do not import opaque packages simply to accelerate code generation.
8. A temporary implementation must satisfy the lasting interface and have a removal trigger. Placeholder visuals are allowed during the agreed prototype; placeholder persistence/economy designed for total replacement is not.

### Authority, money and persistent state

9. The client submits intent. The server validates player/session, state, distance where relevant, permissions, quantities, timing and replay/rate limits. Never accept client-provided prices, rewards, owned IDs, mutation rolls or successful extraction claims as truth.
10. Only the owning server service may mutate its data through approved operations. UI, analytics and content definitions do not write balances, inventory ownership or collection rewards directly.
11. Money and item quantities use bounded validated integers. Reject malformed numbers, non-finite values, unexpected fields, excessive collections and oversized payloads before expensive work.
12. Every retryable economic action has an idempotency identity and defined duplicate result. A receipt/event being delivered more than once is an expected condition, not an exceptional design assumption.
13. Never acknowledge secured loot, completed trade or paid entitlement before the required durable commit. Do not hide an uncertain outcome by issuing a second reward.
14. Never overwrite a failed or timed-out profile load with a default profile. Never allow concurrent writers to a profile by bypassing the ownership/lease mechanism.
15. Persistent schema changes need versioned migration fixtures, validation, size analysis and rollback/forward-repair planning before release. Save an owned reference to a removed content ID as a tombstone-compatible object; do not silently delete it.
16. Multi-player trading stays unavailable until cross-profile transaction recovery, replay protection and ownership conservation pass their explicit gate. No client-to-client transfer and no sequential pair of unprotected inventory writes.
17. Production data edits, compensation, deletion, publishing, asset permission expansion, spending and paid offer activation require the owner's existing explicit authorization or a named approved operational runbook. Approval refers to a concrete diff/action and environment.

### Reliability, performance and maintainability

18. Define recoverable errors and safe degraded behaviour for every external dependency. Use bounded retries with backoff/jitter; avoid infinite retries and unbounded work queues.
19. Treat disconnects, process death, throttling, stale servers, duplicate requests and asset failures as normal test cases. Avoid relying exclusively on graceful shutdown saves.
20. Keep work bounded: cap live creatures, scheduled jobs, session history, inventory pages, input payloads and catalog expansion. A large inventory must not trigger full serialization or replication on each click.
21. Clean up connections, tasks, models and cached player references at lifecycle end. Profile under repeated join/leave, extraction and sanctuary visits.
22. Keep high-value spawn rolls and authority-only configuration server-side. Send clients only the data needed for presentation. Hidden content is not secure merely because its display name is obscure.
23. Do not place executable scripts in imported art prefabs. Review third-party content for scripts and dependencies before integration.
24. Use consistent strict Luau contracts and explicit error results. Avoid global mutable singletons, cyclic module dependencies and a universal event bus that conceals required synchronous dependencies.
25. Log actionable structured errors with correlation IDs; do not log secrets, raw player chat or full profiles. Analytics failure must not block extraction or persistence.

### Evidence and review

26. Write meaningful tests for invariants, branches and failure cases. Avoid large bodies of untested code and tests that merely restate implementation constants.
27. Never claim a test ran when it did not. Separate automated execution, Studio simulation, published-server testing, physical-device testing and manual inspection in reports.
28. An implementer cannot independently approve its own work. Review the actual final revision and rerun affected checks after conflict resolution.
29. Never mark completion with open critical/high defects in the delivered domain. Lower-priority deferrals need an owner, impact statement and agreed milestone.
30. Update contracts, project status, changelog, known bugs and handoff evidence alongside the delivered behaviour. A future agent must be able to continue without chat history.
31. Avoid expanding the slice during bug fixing. Record new features as proposals and preserve the current acceptance boundary.
32. Stop dependent work when required approval is missing, while continuing independent authorized work. Escalate with a concrete proposal and the exact rule or boundary requiring the decision.

## 15. Testing Architecture

### 15.1 Environments, tooling, and evidence

Use separate development, automated-test, staging, and production experiences or rigorously separated test experiences with independent stores, product IDs, badges, assets, and publishing credentials. Do not enable Studio access against production stores. Roblox warns that Studio can access the same data as the live application; staging is therefore mandatory for destructive migration and save-failure tests. [Data-store access guidance](https://create.roblox.com/docs/cloud-services/data-stores)

The future repository contains pure-domain tests, adapter/contract tests, Roblox integration tests, multiplayer scenarios, migration fixtures, exploit probes, economy simulations, content validation, and manual mobile/experience checklists. Select a pinned Luau test runner during Project 01. Tests must use an injected clock, deterministic seeded random source, and fault-injectable persistence adapter. Offline tests validate domain logic; published staging validates actual platform behavior. Neither substitutes for the other.

Studio's client/server modes and multiple-client simulation test replication boundaries; device emulation tests touch layouts, aspect ratios, and input. The current Studio documentation also describes scripted multi-client testing. Confirm availability in the team's installed Studio before selecting it for CI. Published-device tests remain necessary because emulation does not reproduce an actual phone's thermal and memory behavior. [Studio testing modes](https://create.roblox.com/docs/studio/testing-modes)

Every test report records build and content versions, commit, environment, test runner version, device/OS, player count, seed, steps, expected/actual results, logs/traces, failures, owner, and disposition. An AI may draft and run tests it can actually access; it must mark human device tests and inaccessible published tests as **not run**. This architecture stage specifies gates; it does not claim the game has passed them.

### 15.2 Required gates

**Unit gate.** Test deterministic behavior at legal boundaries and invalid inputs: capacity, stack merge/split, value bounds, trait caps, compatibility, capture eligibility, quest conditions, event overlap, risk state transitions, and migration transformations. Use meaningful invariants rather than tests repeating the same constants as implementation. All domain state-machine transitions and every economy authorization branch need executable cases; a coverage percentage alone is insufficient.

**Integration gate.** Exercise a real player session through prepare, expedition start, encounter, capture, pending/committed cargo, extraction, secured collection, upgrade, and rejoin. Verify data projections match committed authority and a missing analytics subscriber cannot affect rewards. Add an observed-only case: see an uncaptured creature, persist the observation, fail, and rejoin; the encyclopedia retains seen knowledge but grants no ownership, secured completion, or reward. Also crash before an observation batch commits and confirm the interface never promised that sighting was saved. Test content-version mismatch, missing asset fallback, streaming detach/rebind, full inventory reservation, event ending during extraction, and quest rewards triggered by extraction exactly once.

**Multiplayer gate.** Test two and eight players locally and in staging. Include simultaneous capture from the same encounter, late arrival outside eligibility roster, player joining/leaving mid-event, one player extracting while another fails, rapid rejoin to another server, and sanctuary visitors during owner departure. Verify per-participant capture ID and immutable roll prevent theft and farming. Before trade release, test disconnect at every escrow state with either player offline and competing requests against the same asset.

**Client/server validation gate.** Treat remotes as a hostile API. Send invalid types, unbounded arrays, deep nesting, NaN/infinity, arbitrary Instances, forged user IDs, unauthorized asset IDs, old protocol versions, repeated request IDs with changed payload, stale revisions, future timestamps, and request floods. Test impossible movement, spoofed contact, distant capture, capture through invalid geometry, accelerated extraction, and tool capability fabrication. Server rejects safely with bounded cost; it never rewards or crashes. No permanent punishment is based solely on one latency-sensitive movement anomaly.

**Save/load and migration gate.** Test successful missing-profile creation separately from read failure. Cover large but admitted profiles, boundary-sized strings, schema N to N+1, already-migrated profiles, retired content, partially prepared segmented migration, newer schemas on old servers, and metadata/fence preservation. Compare pre/post inventory counts, IDs, balances, claims, and provenance. Migration cannot delete an unknown field simply because a current reader does not use it.

**Failure-recovery gate.** Inject a crash or connection failure before and after each persistent boundary: profile acquire, cargo checkpoint, extraction, purchase, reward claim, page publication, ownership reservation, trade decision, final settlement, and session release. Include a write that commits but returns an error, a delayed stale write after takeover, throttling, lost renewal, unavailable store, duplicated queue job, absent MessagingService delivery, MemoryStore expiry, process death without shutdown callback, and replayed receipt. Recovery must converge from durable state without a second grant or double owner. No workflow may depend on a graceful shutdown to preserve acknowledged value.

**Performance gate.** Capture named CPU/frame scopes, client/server memory, network rates, load times, pathfinding work, active entity counts, and VFX load on fixed test routes. Run the declared peak: eight players, cap of creatures, maximum permitted local sanctuary display, fog plus one rare discovery burst, and active inventory browsing. Soak for 60 minutes with repeated joins/leaves and expedition cycles; once trade exists, include prepare/abort/recover cycles. Object counts and memory must return near the warmed baseline after cleanup.

**Mobile and accessibility gate.** On real low and mid device tiers, complete the first run, identify rare/unknown creatures, see cargo risk, extract, view collection, and buy the earned upgrade using touch only. Test safe areas, text scaling, scrolling, interrupted focus, poor network, portrait rejection or adaptation according to the approved orientation policy, audio muted, color-vision alternatives, reduced VFX, and motion settings. Critical hazard and rarity information cannot depend on color, particles, or sound alone.

**Exploit/security gate.** An independent reviewer traces every path that can mint, debit, transfer, equip, sell, secure, or restore value. Audit server tools, admin authorization, imported asset scripts, dynamic `require` calls, remote schemas, secret leakage, stale session ownership, and recovery commands. Investigate abuse reports with structured evidence. Signed-off review and adversarial scenarios are required before enabling trading or monetization.

**Economy gate.** Check conservation for player-to-player transfers and explicit mint/sink entries for system transactions. Simulate new/free/returning players, storage saturation, high-efficiency players, event modifiers, trait stacks, repeated failure, mass bot-like acquisition, and offline claims. Verify paid convenience cannot become superior expedition power. Distribution tests validate expected spawn/capture rates with seeded large samples and confidence intervals; they must not fail merely because a single random sample misses a rare creature. The server never substitutes client-provided odds.

**Regression gate.** Maintain a short automated critical path on every system PR; run affected domain and contracts after change, then required integration cases. Before release, rerun full data/security/recovery regression and the declared device matrix. Every fixed serious bug adds a scenario reproducing its cause, unless its behavior is covered by an existing case whose input is extended. Do not retest unrelated low-impact details endlessly once the necessary gates pass.

### 15.3 Quantitative gate standards and recovery proof

For stateful foundation changes, require at least 10,000 deterministic randomized operation sequences across inventory, grants, claims, extraction, and failure settlement, with zero invariant violations. This is a proposed project gate, not evidence already collected. Exhaust every defined crash boundary at least once with deterministic expected outcome; randomized volume cannot replace those named cases. Before trading, repeat with two-player escrow and late-worker interleavings, checking each asset has exactly one authoritative owner or one inaccessible escrow owner at every observation.

For every scenario, distinguish safety from availability. **Safety:** no acknowledged banked loss, negative balance, duplicate mint, reward replay, or double ownership. **Availability:** an unresolved transaction becomes settled or visibly quarantined within an operational target. Healthy-platform target: normal recovery within 60 seconds of profile reacquisition; unresolved cases older than five minutes alert support. Platform outages can exceed that availability target without relaxing safety.

Completion requires all mandatory gates relevant to the project, no unresolved critical/high data or security defects, reproducible evidence, documentation/API changes, reviewed rollback, and a demonstrable artifact in the appropriate staging environment. A project with only mocked persistence tests is not done if it changes DataStore behavior. A polished capture animation is not proof that capture ownership is safe.

## 16. Performance Standards

### 16.1 Measurement policy

These are **initial WILDLANDS engineering budgets, not Roblox platform limits**. Rebaseline after the measured primitive slice and first art pass. The product owner approves a named minimum device matrix before beta; start with a lower mobile tier around an older 4 GB Android device, a mid mobile tier, an older supported iPhone, and an ordinary integrated-graphics laptop. Physical devices already available to the team are sufficient; record exact models, graphics level, OS, and thermal conditions. Do not infer a universal memory ceiling from RAM alone.

Measure the published client after warm-up, then a 20-minute normal route and a 60-minute server soak, including event and sanctuary peaks. Report p50/p95/p99 and worst spikes with test context. Studio overhead skews performance numbers; Creator Hub recommends client measurement and provides Developer Console and MicroProfiler diagnostics. [Identify performance issues](https://create.roblox.com/docs/performance-optimization/identify)

### 16.2 Client and loading targets

- **Frame rate:** lower mobile tier maintains 30 FPS, with p95 frame time ≤33.3 ms and p99 ≤50 ms during normal play. Mid/high devices target 60 FPS, p95 ≤16.7 ms. Investigate every reproducible gameplay stall above 150 ms; discovery/extraction effects must not cause one.
- **Client memory:** target ≤700 MiB warmed total client memory on the chosen lower mobile tier, ≤900 MiB stress peak, and no sustained upward trend after repeated runs. Track game-added delta separately from engine baseline. These are internal starting thresholds and must be lowered if the chosen device crashes or experiences pressure below them. A ceiling met on desktop proves nothing about a phone.
- **Client scripts:** ≤3 ms average and ≤5 ms p95 attributable script time per frame on the low tier; camera/input remains minimal. Inventory filtering and layout run incrementally, with roughly 20–50 visible cells instantiated and recycled instead of thousands.
- **Join to control:** cold p50 ≤8 seconds and p95 ≤15 seconds on a declared test connection; show actionable status by two seconds. Target first nearby interactable ready ≤12 seconds p95. Measure authentication/server allocation separately when diagnosing platform delays.
- **Asset delivery:** initial essential content target ≤12 MB measured downloaded payload, whole first route ≤25 MB before optional sanctuary/collection previews. These are measured delivery budgets; exported file size is not identical to Roblox-transcoded delivery or decoded memory. Preload only essentials; defer distant biomes and full collection previews.
- **Streaming transitions:** no safe-floor fall-through, invisible mandatory extraction interaction, or infinite wait for a streamed Instance. Critical navigation and arrival areas need explicit readiness checks; optional visuals can use bounded fallback. Avoid marking entire biomes permanently resident.

### 16.3 Server and persistence targets

- **Population:** eight players per slice server. Increasing this is a separate profiling decision, not a configuration-only assumption.
- **Server scheduling:** target 60 Hz simulation health with p95 frame time ≤16.7 ms under declared peak; game-script work ≤4 ms average and ≤8 ms p95 per frame. No synchronous full-catalog scan, inventory clone, or per-creature path solve on every heartbeat.
- **Server memory:** target ≤1 GiB warmed slice server and ≤1.5 GiB peak with eight players, including cached content; require plateau after repeated join/leave cycles. These are conservative project budgets, not engine allocation guarantees.
- **Creature simulation:** at most 24 awake authoritative world creatures, 48 lightweight logical encounter records, and two humanoid-style utility NPCs initially. Dormant entities have no full AI loop. Stagger decision ticks around 2–5 Hz, nearby steering at a separately budgeted rate, and path requests through a queue starting at five solves/second/server. Validate rates against behavior quality and CPU traces.
- **Sanctuary simulation:** render only nearby plots and a small display set, initially up to eight animated displayed creatures per client across visible plots. Stored creatures remain records. Idle ambience is a client presentation system and cannot produce unverified resources.
- **Data:** normal profile target ≤64 KiB; admission ceiling 128 KiB, with migration triggered earlier as specified in section 4. Keep sustained write throughput and request usage under 50% of the measured/current allowance in normal operation to reserve retries and peaks. Target average ≤24 aggregate commits/player/minute during the slice, with coalescing and resource pacing; remeasure the actual per-key serialized load.
- **Durable actions:** under healthy services target ≤2 seconds p95 from accepted valuable action to committed result and ≤3 seconds p95 for extraction settlement. If targets are missed, throttle admission or reduce write payload; never acknowledge an uncommitted grant merely to improve the metric.

### 16.4 Replication, geometry, and effects

- **Application networking:** target ≤12 KB/second server-to-client and ≤3 KB/second client-to-server per active player for custom gameplay traffic in steady play. Track total engine traffic separately, initially aiming ≤50 KB/second received per player outside loading, with measured brief peaks documented. No fixed platform throughput promise is implied.
- **Payloads:** ordinary commands ≤1 KiB, paginated responses ≤16 KiB, initial private-state snapshot ≤32 KiB before optional pages. Use interest filtering and changed fields; never resend all inventories for a currency change. Full creature transforms are unnecessary every render frame when clients can interpolate.
- **Geometry:** low-tier camera target ≤250,000 visible triangles and ≤1,500 rendered world/prop parts in the stress view; desktop art settings may allow approximately twice the triangle budget after profiling. World slice authored target ≤8,000 BaseParts total including hidden areas; streaming and batching determine residency. Triangle and part counts are screening tools, not a substitute for frame measurement.
- **Draw complexity:** starting stress-view target ≤500 draw calls where tooling exposes a usable count; keep material/texture variety restrained, merge static compatible assets where streaming/culling remain sensible, and inspect shadow/transparent overdraw separately. Do not rely on assumed automatic batching to approve assets.
- **Creature asset tiers:** typical visible creature LOD0 around 3,000–6,000 triangles, hero form 8,000–12,000 only after art/performance review; LOD1 near half, LOD2 around 15–25% of LOD0 or a silhouette proxy. Distant models reduce animation as well as geometry. Art pipeline specifies detailed rig and texture rules.
- **Textures:** use shared palettes and material families. Typical creature sheet 512², hero 1024² when justified, small props 256²–512². Do not multiply unique 1024² texture sets for every mutation. Decoded textures and material maps consume memory beyond compressed file size.
- **VFX:** low tier starts with ≤300 simultaneously visible particles, ≤12 active particle emitters, and two large discovery bursts at once; rare tier quality adds restrained effects only after profiling. Prefer opaque/simple visuals, limit full-screen transparency, and preserve hazard readability. Weather uses a local envelope; it must not spawn particles across the whole world.
- **Lighting/audio:** restrict shadow-casting local lights, initially four visible dynamic local lights on low tier; ambient/weather transitions blend presets rather than rebuilding the world. Prioritize roughly 16 simultaneous audible voices, with distance culling and concurrency caps per sound family.

Roblox's optimization guidance emphasizes streaming, avoiding excessive high-frequency work, and measuring memory. Our budgets translate those principles into a deliberately small first scene; they are provisional until device evidence exists. [Improve performance](https://create.roblox.com/docs/performance-optimization/improve)

### 16.5 Profiling and controlled degradation

Give named profiler scopes to AI scheduling, spatial queries, replication build, inventory projection, save serialization, event application, and sanctuary rendering. Use Developer Console for memory and MicroProfiler frame/network captures for bottlenecks; network diagnostics require captured dumps rather than assuming the live overlay exposes every detail. [MicroProfiler modes](https://create.roblox.com/docs/performance-optimization/microprofiler/modes), [Network profiling](https://create.roblox.com/docs/performance-optimization/microprofiler/network)

Degrade in this order: distant decorative animation, particles and auras, grass density, shadow distance/local lights, optional sanctuary detail, then distant encounter simulation. Maintain the same reward eligibility, creature silhouette recognition, hazard warnings, and extraction function on every quality tier. Never improve performance by secretly reducing low-device rare-creature chances. Stop content rollout when a representative device regresses beyond budget until the owner either fixes the cause or records an evidence-backed budget revision.

### Technical evidence register

Primary sources checked **7 September 2026**. Source-backed statements above describe platform behavior; all WILDLANDS storage protocols, admission thresholds, content budgets, and test gates are proposed engineering decisions. Recheck the linked platform documentation before implementation and at dependency/API upgrades.

- Roblox Creator Hub, [Data stores](https://create.roblox.com/docs/cloud-services/data-stores): server storage access, same-experience scope, conditional update callback, Studio isolation.
- Roblox Creator Hub, [Data-store error codes and limits](https://create.roblox.com/docs/cloud-services/data-stores/error-codes-and-limits): uncertain write outcomes, size/throughput constraints, budgets and throttling.
- Roblox Creator Hub, [Implement player data and purchasing systems](https://create.roblox.com/docs/cloud-services/data-stores/player-data-purchasing): session-overlap risks, lock pattern, durable receipt handling.
- Roblox Creator Hub, [Securing the client-server boundary](https://create.roblox.com/docs/scripting/security/client-server-boundary): server input validation and authoritative grants.
- Roblox Creator Hub, [Instance streaming](https://create.roblox.com/docs/workspace/streaming): streamed Workspace scope and model lifecycle.
- Roblox Creator Hub, [Network ownership](https://create.roblox.com/docs/physics/network-ownership): client-owned physics and contact-event risks.
- Roblox Creator Hub, [Teleport between places](https://create.roblox.com/docs/projects/teleport): secure state must not rely on TeleportData.
- Roblox Creator Hub, [Developer products](https://create.roblox.com/docs/production/monetization/developer-products): server receipt processing boundary.
- Roblox Creator Hub, [Memory stores](https://create.roblox.com/docs/cloud-services/memory-stores) and [MessagingService](https://create.roblox.com/docs/reference/engine/classes/MessagingService/SubscribeAsync): transient storage and best-effort messaging.
- Roblox Creator Hub, [Studio testing modes](https://create.roblox.com/docs/studio/testing-modes): client/server, multiplayer, and input/device testing capabilities.
- Roblox Creator Hub, [Identify performance issues](https://create.roblox.com/docs/performance-optimization/identify), [Improve performance](https://create.roblox.com/docs/performance-optimization/improve), [MicroProfiler modes](https://create.roblox.com/docs/performance-optimization/microprofiler/modes), and [Network profiling](https://create.roblox.com/docs/performance-optimization/microprofiler/network): evidence collection and diagnosis.

## 17. Live-Ops Architecture

Content expansion uses the same modular monolith and domain services as the slice. Add a data-driven recipe when an existing mechanic can express the change; create a reviewed capability when it cannot. The brief's proposed requirement for one new mechanic every major update should be optional: a biome plus compelling discoveries can justify an update without permanently increasing systemic complexity.

### 17.1 Catalogs, schedules and runtime boundaries

Each published content catalog is immutable and identified by an epoch, content hash and compatible server build range. Its manifest lists definitions, asset revisions, localization, drop tables, feature capabilities, activation dependencies and fallback policy. A separate environment release pointer selects an accepted catalog for newly admitted runs. Editing an already published catalog in place is prohibited.

The server validates catalogs before admitting players. Duplicate IDs, dangling references, invalid probabilities, incompatible rigs, missing assets, excessive list sizes, unknown effect keys, invalid date ranges and negative/overflowing reward quantities block activation. Client catalogs are filtered projections containing presentation fields; they do not disclose secret spawn seeds or authoritative roll tables unnecessarily.

**An expedition pins its catalog epoch, seed, rule version and event occurrence at creation.** Capture outcomes and secured creature provenance retain the relevant revision. A safe cosmetic refresh may update presentation under an explicit compatibility rule, but changing the odds halfway through a run is not the default. Cohorts should be assigned by server/run so co-op players share compatible rules.

Live settings accept only a small allowlisted schema: flags, bounded numeric tuning, known definition IDs, schedules and predeclared effect keys. No executable Lua, arbitrary module asset IDs, dynamic `require`, remote code, arbitrary URLs or unconstrained expressions. Changes that introduce a new effect capability ship through the reviewed code pipeline. A cached validated configuration is usable during control-plane failure; absence of configuration never becomes an unlimited reward mode.

Event schedules use UTC start/end timestamps and a stable occurrence ID. Define inclusive/exclusive boundaries, eligibility timestamp, entry deadline, run grace period, claim deadline and post-event behaviour. Never rely on a player's clock or local timezone. A server starting midway through an occurrence derives current phase from the authoritative schedule; it does not replay all past rewards. Reward claims use durable occurrence/claim identities.

### 17.2 Content recipes

- **Creature:** add stable species ID, compatible family/behaviour profile, spawn conditions, capture profile, collection metadata and accepted prefab/animation references. Existing services handle observation, capture, ownership and display.
- **Mutation:** add a bounded visual/effect recipe, compatibility tags, acquisition rules and collection identity. Validate permitted species combinations. Initially allow one principal mutation plus bounded size variation; expand stacking only after combinatorial, performance and economy tests.
- **Biome:** add a region manifest, traversal connections, bounds/streaming layout, spawn anchors, resource tables, hazard profiles, equipment/access requirements, extraction rules, art kit, audio/lighting and progression links. A new biome's geometry requires authoring; data-driven does not mean automatically creating a playable map.
- **Event:** compose approved triggers, conditions, spawn modifiers, environmental presentation and rewards. Migration/Meteor/Fog should use bounded spawn/effect budgets shared with ordinary gameplay.
- **Resource/item/equipment:** add item definitions, tags, stack/storage constraints, approved uses, sink/source rules and visual references. Equipment combines allowlisted capabilities; it cannot carry executable behaviour inside a content entry.
- **Sanctuary item/upgrade:** add footprint, placement constraints, cost, entitlement, display capacity and approved effects. Placement rules remain server-validated and budgeted.
- **Quest/achievement:** add objective types from the existing registry, eligibility, progress aggregation, reset policy and idempotent reward recipe. A new objective type is a code extension with its own tests.
- **Season/cosmetic:** add schedule, collection/group IDs, entitlement sources, localization and approved presentation bindings. Paid activation is a separate reviewed operation after the free progression has been validated.

Every content request includes player value, implementation classification (configuration/asset/new capability), dependency list, economy impact, performance allocation, tests and removal policy. This prevents content from becoming an unreviewed second programming language.

### 17.3 Rollout and recovery

Move a candidate through local validation, staging catalog, published multiplayer/device checks, owner-authorized limited rollout and wider release. Keep build/catalog compatibility explicit while old servers coexist. Compare error rate, load failures, extraction completion, duplicate/failed claims, resource issuance, performance and early-session progression against the baseline. Use reviewable thresholds established before rollout; low sample size or seasonal changes must not be misreported as proven retention gains.

Only the release owner, or an explicitly delegated operator using an approved runbook, activates the production pointer. Keep a named operator, incident contact, observation window, known-good pointer and rollback decision thresholds in the release record. Cross-server messages can accelerate refresh, but servers periodically reconcile the durable accepted state; a missed notification must not strand them forever.

Rollback changes what compatible servers admit for future work. It does not blindly overwrite player profiles or reverse already committed transactions. If the old build cannot read new data, disable the problematic feature and forward-fix with compatible code; do not roll back to an incompatible reader. Prevent startup or write admission on unsupported schema versions. Old live servers must be drained or fenced according to the compatibility plan.

An emergency denylist can disable new spawns, purchases, claims, trades involving a definition, or event entry independently. Existing owned objects remain visible through a safe fallback and retain identity/provenance. Already valid queued extraction settlement should complete where safe; if integrity cannot be established, preserve the unresolved transaction for recovery and display its pending status rather than discarding it.

Content retirement uses tombstones: the original ID, localized historical name, essential ownership metadata and safe presentation remain resolvable. Disable future acquisition separately from ownership and collection history. Limited discoveries record season/event occurrence; their collection page should distinguish historical completion from currently attainable goals. Do not make old content vanish or sell an implied permanence the design cannot honour.

Compensation is a reviewed durable transaction keyed by incident and affected account/transaction, with a documented eligibility query, amount cap, dry-run count and reversal policy where applicable. Recovery retries must not duplicate compensation. Support needs a restricted audit view of transaction/provenance records and tools that call the same validated domain operations; it must not become an unrestricted profile editor.

### 17.4 Sustainable operations

Ship the first event as a local server occurrence using the shared event interface. Global coordination, rotating seasons and campaign tooling come after the basic loop and durability have passed. Trading remains a later rollout behind its transaction gate. Offline sanctuary progression uses capped elapsed server time and a single durable claim, with explicit economy sinks; it is a modest return bonus rather than the main source of advancement.

For the later offline-production milestone, begin with an eight-hour accrual cap and ordinary basic resources only. Calibrate the maximum daily claim to a small fraction (initial target at most 10%) of a benchmark active-play day; revise using cohort economy evidence. Snapshot the eligible production roster and rule version, settle the prior interval when that roster changes, and atomically advance the last-claim time with the reward. Clamp negative elapsed time to zero, cap long absences, and never apply a newly equipped creature or a current event multiplier retroactively to the entire absence. No per-creature offline timer or rare/premium resource mint is required.

Build content cadence around measured production capacity. A repeatable small creature family, a tested mutation and a meaningful event may be better than an incomplete biome. Maintain a calendar of reversible events, content lead time, moderation contingency and asset reuse opportunities. Retention work should offer achievable goals and return opportunities without punishing absence or erasing collection progress.

## 18. Risk Register

Severity combines impact on player trust, architectural cost, and likelihood during the stated stage. Every risk needs an owner and evidence in the project tracker; changing severity does not remove its release gate. The register contains exactly twenty risks in priority order.

### 01 — CRITICAL: persistent profile overwrite or loss

**Why:** permanent collections are the player's investment; one overwrite can destroy trust. **When:** first saving integration, reconnects, concurrent sessions, migrations, and shutdowns. **Prevent:** single-writer ownership, validated loading, versioned transforms, tested snapshots and restore procedures; never save default data after an unsuccessful load. **Detect:** revision anomalies, load/save errors, ownership conflicts, profile validation failures, and support reports. **Recover:** lock affected writes, preserve evidence, restore a verified snapshot, reconcile later valid transactions, and document affected accounts.

### 02 — CRITICAL: duplicate or missing extraction awards

**Why:** extraction is the central ownership promise and the largest natural duplication boundary. **When:** repeated input, concurrent capture, disconnect during commit, or timeout retries. **Prevent:** stable run/award identities, one terminal run decision, idempotent durable grants, and a pending state that cannot be spent. **Detect:** repeated award keys, contradictory run states, unexpected inventory creation, and fault tests. **Recover:** suspend affected award paths, replay the recorded decision once, quarantine ambiguous assets, and compensate only through audited transactions.

### 03 — CRITICAL: unsafe player-to-player transfers

**Why:** two owners, confirmations, and failures can create duplication, theft, or unrecoverable partial trades. **When:** trading, gifting, mail, or auctions are introduced. **Prevent:** defer all transfer paths until escrow, durable decisions, idempotent delivery, reconciliation, and trade-term resets pass crash tests. **Detect:** stranded reservations, unequal settlement, asset ownership conflicts, and scam reports. **Recover:** disable transfers, resume recorded committed decisions, cancel only uncommitted reservations, and restore from an auditable asset trail.

### 04 — CRITICAL: client-authorized rewards or bypassed eligibility

**Why:** a forged request can mint currency, capture distant creatures, or bypass progression. **When:** any client interaction or new reward source is added. **Prevent:** server-derived values, proximity/state validation, bounded payloads, rate controls, capabilities, and central mutation APIs. **Detect:** impossible movement-to-award timing, reward-rate anomalies, invalid-request metrics, and adversarial remote tests. **Recover:** disable the vulnerable route, invalidate uncommitted claims, audit affected assets, and repair through reversible, reviewed transactions.

### 05 — CRITICAL: missing or duplicated paid entitlements

**Why:** purchase receipts can be retried or delivered during uncertain saves; an incorrect acknowledgment loses purchased value or grants it repeatedly. **When:** the first developer product, receipt archive or commerce migration is enabled. **Prevent:** server receipt ownership, permanent purchase-ID deduplication, grant and receipt in the same durable operation, archive-aware replay checks and reserved storage headroom. **Detect:** receipt-to-entitlement reconciliation, duplicate purchase IDs, stale pending receipts and support reports. **Recover:** disable the affected product path, reconcile the existing receipt and durable grant before retrying, preserve evidence and use an audited repair; never mint a fresh replacement solely because the callback timed out.

### 06 — HIGH: incompatible data/content rollout

**Why:** old servers and newer catalogs can interpret identifiers or persistent fields differently. **When:** schema migrations, content removal, or mixed-version deployment. **Prevent:** compatibility windows, catalog/version manifests, staged migration, tombstones, explicit supported versions, and backward-readable fields where practical. **Detect:** unknown identifiers, migration failures, version mismatches, and canary validation. **Recover:** halt rollout, route to compatible builds, revert configuration when safe, and repair forward when new writes make rollback unsafe.

### 07 — HIGH: the loop is understandable but not compelling

**Why:** sound engineering cannot rescue a game players do not voluntarily replay. **When:** greybox and slice tests, before costly production. **Prevent:** early uncoached tests of discovery, risk, extraction payoff, and a changed second run. **Detect:** low voluntary re-entry, repetitive choices, confusion after upgrading, and interviews reporting chores. **Recover:** freeze content expansion, identify the weak decision or payoff, change one hypothesis at a time, and retest with fresh players.

### 08 — HIGH: loss causes players to quit

**Why:** a young audience may interpret unclear cargo loss as permanent theft of progress. **When:** first defeat, disconnect, Fog event, or later high-risk regions. **Prevent:** explicit secured/unsecured states, telegraphed danger, short return routes, permanent-property protection, and separate outage recovery. **Detect:** exits immediately after loss, misunderstood interviews, repeated rescue loops, and abandonment by new players. **Recover:** reduce exposure or run length, clarify outcomes, improve escape affordances, and fix or compensate confirmed erroneous losses.

### 09 — HIGH: economy inflation or reward farming

**Why:** uncontrolled sources invalidate upgrades and collectibles, especially once transfers exist. **When:** repeatable quests, event bonuses, offline production, or stacked traits arrive. **Prevent:** explicit source/sink ownership, bounded awards, grant-once rules, finite stacking, and simulations of ordinary and abusive play. **Detect:** currency supply, source-to-sink ratios, item creation rates, progression time, and outlier cohorts. **Recover:** disable the source, change future configuration, reconcile illicit awards where evidenced, and avoid indiscriminate balance wipes.

### 10 — HIGH: lower-end device performance collapse

**Why:** poor frame pacing or memory pressure makes discovery and escape unreadable. **When:** polished art, Fog/VFX, dense sanctuaries, or content accumulation. **Prevent:** tested asset/display budgets, representative devices, scalable cosmetics, bounded nearby simulation, and early full-population tests. **Detect:** frame-time tails, memory peaks, disconnect/crash reports, load duration, and device-segment funnels. **Recover:** switch to validated lighter content/effects, lower display density, remove offending assets, and retest on the affected devices.

### 11 — HIGH: server simulation and networking overload

**Why:** delayed interactions can lose captures, make hazards unfair, and amplify retries. **When:** many active creatures, all players gathering together, or event migration bursts. **Prevent:** fixed encounter caps, behavior scheduling, bounded path work, scoped updates, and load tests at hot spots. **Detect:** server-frame spikes, operation queue age, request/replication volume, and capture latency. **Recover:** stop discretionary spawns, lower cosmetic/simulation density, disable the expensive behavior, and preserve authoritative transaction processing.

### 12 — HIGH: save pressure from large inventories

**Why:** unbounded records, frequent writes, and offline rewards can make persistence slow or exceed service budgets. **When:** mature accounts, event grants, or later trading. **Prevent:** payload budgets, bounded journal retention, compact identifiers, dirty-state batching, representative mature fixtures, and a tested partition plan before limits are approached. **Detect:** serialized size, write latency, throttling, pending-save age, and reconciliation backlog. **Recover:** halt nonessential grants, keep affected data read-only where needed, compact safely, or migrate through the tested storage plan.

### 13 — HIGH: AI agents create incompatible parallel systems

**Why:** duplicated authority and silent API changes undermine every subsystem. **When:** parallel production, rushed repairs, and context handoffs. **Prevent:** scoped briefs, exclusive owners, shared API/schema registry, explicit architecture decisions, independent review, and contract tests. **Detect:** duplicate modules, new dependency cycles, undocumented public changes, and failing integration gates. **Recover:** stop conflicting merges, select the approved owner and contract, adapt callers incrementally, and record the decision rather than rewriting unrelated systems.

### 14 — HIGH: rarity becomes an unfair progression wall

**Why:** required extreme chance produces frustration and incentivizes abusive monetization. **When:** region unlocks, quests, collections, or luck traits are balanced. **Prevent:** deterministic main-path milestones, optional rare goals, habitat knowledge, accessible return windows, and honest chance communication. **Detect:** time-to-unlock tails, cohorts stuck at a rare objective, paid/free divergence, and feedback about mandatory grinding. **Recover:** replace the gate with ordinary exploration requirements, add a deterministic alternative, and preserve credit for prior effort.

### 15 — HIGH: asset throughput exceeds quality control

**Why:** hundreds of inconsistent rigs, materials, bounds, and collisions turn content into engineering repair work. **When:** scaling beyond the approved slice. **Prevent:** species families, shared rig/animation standards, automated manifests, import validation, style review, and a small approved reference set. **Detect:** failed imports, animation defects, silhouette inconsistency, memory drift, and integration time per asset. **Recover:** quarantine nonconforming assets, repair source files and exporter settings, and reduce release quantity until acceptance rates recover.

### 16 — MEDIUM: shared encounters become griefing opportunities

**Why:** first-touch capture, blocking, or monopolizing rare spawns undermines co-play. **When:** a rare creature attracts several players or a streamer's group. **Prevent:** per-participant eligibility, one award each, nonexclusive interactions, collision-aware layouts, and bounded respawn rules. **Detect:** denied-capture rates, repeated obstruction, nearby nonparticipant behavior, and player reports. **Recover:** adjust encounter admission or geometry, disable abusive interactions, and verify that fixes do not award idle spectators or enable farming.

### 17 — MEDIUM: event configuration breaks ordinary play

**Why:** a wrong overlay can remove extraction access, overproduce currency, or retire cargo before it is secured. **When:** scheduled activations and seasonal transitions. **Prevent:** validated manifests, ordinary-play fallbacks, reward caps, expiry rules, deterministic dry runs, and a tested kill switch. **Detect:** activation errors, unreachable routes, reward spikes, missing spawns, and completion-funnel drops. **Recover:** disable the overlay, retain valid earned cargo, reconcile affected rewards, and restore the previous catalog manifest.

### 18 — MEDIUM: collection and prestige claims are misleading

**Why:** ambiguous “first,” unavailable completion requirements, or paid rare-looking skins reduce trust. **When:** global firsts, seasonal catalogs, renamed assets, and premium cosmetics. **Prevent:** explicit scope labels, separate earned forms and skins, immutable identities, archival collection rules, and an eligibility policy before global claims. **Detect:** conflicting claims, impossible completion pages, display/ownership mismatches, and moderation tickets. **Recover:** correct labels and records, retain historical provenance, suspend disputed global awards, and issue audited replacement recognition where warranted.

### 19 — MEDIUM: live-ops promises exceed sustainable capacity

**Why:** a fixed biome-plus-mechanic schedule can destabilize the game and burn production effort on quantity. **When:** after beta and during seasonal cadence commitments. **Prevent:** measure asset/system lead times, reuse approved mechanics, budget QA and localization, and separate content releases from engineering features. **Detect:** growing technical debt, recurring rollbacks, rising defects, and repeatedly missed gates. **Recover:** reduce scope or cadence, ship smaller tested packs, reserve stabilization cycles, and communicate revised internal milestones.

### 20 — MEDIUM: telemetry and playtests produce false confidence

**Why:** scripted encounters, duplicated events, small samples, or missing device segments can conceal retention and performance failures. **When:** slice evaluation, beta experiments, and economy changes. **Prevent:** event dictionaries, deduplicated identities, scenario labels, cohort definitions, fresh testers, and qualitative observation alongside metrics. **Detect:** impossible funnel ordering, inconsistent denominators, missing sessions, and disagreement between recordings and analytics. **Recover:** repair instrumentation, exclude contaminated cohorts, rerun the affected experiment, and delay decisions unsupported by trustworthy evidence.

## 19. Definition of Done

`Implemented`, `Integrated`, `QA Accepted` and `Released` are separate states. A project is **Done for its declared milestone** only when its approved scope, dependencies and evidence meet that milestone's gate. “Done for the slice” does not claim that deferred trading, multiple places or launch-scale content exists.

A completed project must provide all of the following:

1. **Behaviour:** its acceptance scenarios work in the current integrated build, including the required player-facing error, empty and recovery paths. For gameplay, testers can complete the intended loop without developer intervention.
2. **Contract integrity:** public APIs, state ownership, data definitions and dependencies match the approved contracts. Any approved change includes migrated consumers and updated documentation.
3. **Durability:** relevant load/save, concurrent ownership, retry, replay, process-loss and migration cases pass. No reward acknowledgement relies on an uncommitted authoritative result. Deliberately unsupported failure recovery is visible and blocks release where it threatens ownership or money.
4. **Security and economy:** adversarial inputs cannot mint value, transfer another player's assets, bypass access or create unbounded work. Relevant conservation/issuance invariants pass. Critical/high integrity defects are closed.
5. **Multiplayer and presentation:** the feature passes its defined concurrency scenario, streaming lifecycle and input/device coverage. Asset permissions and animation playback work in a published staging build, not solely the author's Studio session.
6. **Performance:** the measured build meets its allocated budgets and full-scene targets on the named test devices. Reports identify device, graphics level, population, duration, catalog and profiling conditions. An exception requires owner-approved scope and mitigation.
7. **Evidence:** unit/integration/manual results are stored with build revision, reproduction instructions and failing cases resolved. Untested checks are listed honestly and cannot silently count as passed.
8. **Maintainability:** code and assets have appropriate ownership, lifecycle cleanup, observability, versioned dependencies and a handoff. No duplicate domain service, silent API change or unowned technical debt is introduced.
9. **Recovery:** the release unit has a tested disable/rollback or forward-repair path. Data changes and paid operations include specific compatibility/reconciliation evidence. The operator knows what remains safe during an incident.
10. **Independent acceptance:** a reviewer other than the implementer has inspected the final change and QA has validated the relevant gate. Known bugs, debt, status and changelog reflect the result.

For the vertical slice, product acceptance additionally requires an observed **explore → discover → choose risk → extract → return → upgrade → voluntarily start another run** journey, using the agreed one-biome, three-species scope. Qualitative playtest findings and funnel data must support the next investment; functional correctness alone does not prove that the game is fun.

Launch acceptance adds production operational access, monitored rollout, platform/asset permissions, device coverage, recovery rehearsals and owner release approval. No AI may call the entire game complete because its own assigned module passed unit tests. A project with a blocked mandatory gate stays `Blocked` or `In Review`, with the exact missing evidence and next owner recorded.

## 20. Recommended Next Steps

1. **Establish the approval baseline.** Review the proposed invariants, unsecured-loss rule, single-place eight-player slice, three species/four forms, Coins-only economy and delayed transfer/monetization gates. Record accepted decisions and genuine unresolved constraints as ADRs; do not restart the game design from a blank page.
2. **Run Project 01's documentation milestone.** Place the source brief and accepted specification in the actual game repository, reconcile any existing documentation, define named contracts and state machines, create the project-status/dependency ledger and assign ownership. This task has produced the specification only; it has not modified the supplied game's repository.
3. **Select the smallest toolchain.** Evaluate source-to-Studio synchronization, Luau checking, a test runner, persistence adapter and binary asset versioning against real constraints. Pin chosen versions and record alternatives. Avoid adopting a framework stack merely to create the appearance of a foundation.
4. **Authorize a bounded foundation increment afterward.** P02/P03/P06 should produce the first complete authenticated, persisted and acknowledged action with failure tests. The next implementation request should name one milestone, allowed paths, contracts and required evidence.
5. **Reach the graybox loop before breadth.** Build the minimum provider milestones for one expedition, extraction, sanctuary display and upgrade. Use fresh testers and physical touch devices. If the second-run gate fails, spend the next increment on the observed problem.
6. **Scale production from evidence.** Establish one polished creature family and one complete content package; test the second-biome authoring path. Add social visits before transfer complexity. Enable commerce and trade only through their independent durability gates.

### Assumptions requiring later operational confirmation

The owner/team's current repository state, Roblox group ownership, Studio synchronization method, available physical test devices, art-production throughput, staff capacity, deployment access and live account limits were not audited in this planning task. Project 01 inventories them without changing production. This specification does not claim a working build, passed game tests, measured retention, fixed release date or validated device performance. The numeric budgets are acceptance targets for subsequent evidence.

### NEXT ACTION

Copy the following prompt to start the next project. It deliberately authorizes the documentation and contract milestone of Project 01; it does not authorize game implementation or publishing.

```text
Start Project 01 — Master Architecture / Technical Foundation for WILDLANDS: Expedition.

Your role is Lead Architect and Technical Director. Complete the documentation and contract milestone only. Do not implement the game, generate Roblox/Luau scripts, create game UI, make Blender models, upload assets, change production data or publish anything.

Read the entire original brief:
/Users/dooves/Documents/Projects/Roblox/Wildlands/docs/Game Master Brief — WILDLANDS_ Expedition.md

Then read the entire planning baseline:
/Users/dooves/Documents/Codex/2026-09-07/wildlands-master-development-architecture-task-you/outputs/WILDLANDS — AI DEVELOPMENT MASTER SPECIFICATION.md

Inspect /Users/dooves/Documents/Projects/Roblox/Wildlands and its applicable repository instructions before proposing edits. Treat the two documents as design input; follow this task's scope. Preserve existing work. If a real conflict exists, document the concrete decision and impact instead of silently replacing working documentation.

Use the master specification's recommended boundaries as the planning baseline: server-authoritative modular monolith; single-place eight-player slice; safe permanent possessions; seen/captured/secured distinction; exactly one durable run settlement; three base species Fox/Deer/Owl plus Crystal Fox as a mutation; one Meadowlands biome with forest edge, Fog, Coins, one sanctuary upgrade; trading and paid products disabled until their durability gates pass. Mark these decisions Proposed until the owner records approval. Do not expand the slice.

Produce a coherent documentation package in the actual repository:
1. An entry-point README and status ledger identifying the current milestone, completed work, blockers and dependency-ready next tasks.
2. An immutable source-brief reference and the master specification baseline with provenance/version.
3. Architecture decision records for service ownership, place topology, inventory/storage growth, session fencing, loss/reconnect policy, networking, catalog versions and content/asset authority.
4. Exact conceptual public contracts for player readiness, versioned commands/results, inventory queries/reservations, economy plans, capture entitlement plans, expedition cargo settlement, collection projections and sanctuary display. Define inputs, errors, authority, idempotency, commit boundaries and consumers. No implementation code.
5. State machines and failure traces for profile acquisition, observation/capture, extraction, upgrade purchase, reconnect, uncertain write and migration. Treat cross-key trade and receipt protocols as later gated designs; never imply Roblox provides atomic multi-key transactions.
6. Versioned data/schema documentation, bounded sizes, example records and migration acceptance fixtures for the initial player aggregate, plus the explicit criteria for segmented ownership migration before trade.
7. P01–P24 project briefs derived from the supplied decomposition, with ownership, exclusions, prerequisites, deliverables, acceptance evidence and staged milestone status. Reconcile and reuse the supplied briefs; do not invent a parallel project plan.
8. Repository/Studio mapping, asset naming/rig/registry contracts, AI role rules, ADR/change-control process, handoff template, known-bugs/debt records and test/report templates.
9. A minimal proposed development toolchain with pinned-version selection criteria, ownership, compatibility evaluation and dependency rationale. Do not install packages or scaffold runtime code during this milestone.
10. A ready-to-execute P02 foundation brief limited to player lifecycle, one secure intent/result path and interfaces for the first persistence integration, with exact tests and allowed paths.

Use bounded parallel reviewers where useful, with one writer per shared document. Independently audit the final package for cycles, duplicate state owners, ambiguous cargo grants, incompatible schema changes and missing failure cases. Verify current Roblox-specific constraints using official documentation and distinguish platform facts from proposed budgets.

Definition of Done: a new coding/asset/QA agent can find the current state and implement its next bounded milestone without chat history or inventing public contracts; every critical invariant has an owner and an acceptance case; all remaining owner decisions are concrete and reviewable. List checks actually performed and any evidence still missing. Conclude with the exact bounded prompt for P02, but do not begin P02 yourself.
```
