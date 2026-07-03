# AATITHYA — Architecture & Systems Audit

*Full read-through of `index.html` (7,500 lines, single-file app). This document is the Phase 1–9 deliverable: architecture, system audit, bugs, redundancies, realism gaps, and a prioritized roadmap. Line numbers reference the baseline commit.*

---

## 1. Overall Architecture

### 1.1 Shape of the codebase

The entire game is one HTML file:

| Region | Lines | Content |
|---|---|---|
| CSS | 7–219 | Base theme + first responsive layer |
| HTML shell | 221–292 | Setup wizard, app shell (topbar / sidebar / `#main`), modal, toasts |
| JavaScript | 294–7498 | Everything else, in **13 sequential layers** |

The JS is organized as an **accretion of versioned layers**, each appended below the previous one:

| Layer | Lines (approx) | Adds |
|---|---|---|
| v1 | 300–1516 | Core engine: cities/localities/difficulty data, rooms, staff, facilities, guests, demand, hourly tick, events, 10 tabs, save/load, wizard |
| v2 | 1518–2379 | Booking channels, room components, loyalty tiers, guest DB / returning guests, fog-of-war competitor intel, analytics, reports, log, policies tab, HQ/chain |
| v3 | 2381–3019 | Empty-plot start, name generator, check-in hour curve, maintenance backlog, F&B quality, direct-booking marketing, Operations tab, hamburger nav |
| v4 | 3021–3865 | Inventory/procurement, marketing campaigns, weather, regional names, guest routines, loans/investors, construction projects, automation rules, competitor entry/exit |
| v5 | 3868–4173 | Hotel-class model, unified demand engine, credit-based housekeeping, weighted-touchpoint satisfaction, GST, macro-economy, USALI P&L, training flywheel, mobile CSS |
| v6 | 4176–4523 | Review pools (sentiment-pure), more names, guest facility-usage tracking, autosave |
| v7 | 4526–4895 | Departments as business units (P&L, delegation, pricing), living front office, advanced HR (burnout/resignations), department pages |
| v8 | 4898–5152 | Goals/score/achievements, Attention Center, notifications inbox, settings/accessibility, onboarding, room-status board, 7-day outlook |
| v9 | 5155–5368 | Economy rebalance (×1.5 costs), loan/investor caps, room-quantity builder, landing screen, light theme, event choice gating |
| v10 | 5371–5744 | 7×24 shift roster + fatigue, loyalty tab, chart engine, OTB calendar, asset lifecycle/PM, F&B covers model, room standardization, valet tiers, nav regroup |
| v11 | 5747–6223 | Roster-gated workforce (on-duty), FO merge into Front Desk, wear bands/renovation center, loyalty ledger, review variety, interactive charts, contracts/notice periods/loan fees, light-mode polish |
| v12 | 6226–6935 | Guest personas/memories/dealbreakers, real reservations pipeline (escrow/no-shows/waitlist), revenue-management autopilot, room lifecycle states, live floor view, overtime/night premiums, competitor strategies/acquisitions, supplier tiers, contractor/permits, multi-dimensional reputation & identity, explainable finance, tooltips, guided tutorial, mobile bottom-nav |
| v13 | 6938–7497 | Roster caps overhaul, channel progression rework, chart engine v2, earned investor pitching, pricing delegation, 10-tier goals, training v2, loyalty v3, reports drill-down, competitor density, final mobile pass |

### 1.2 The extension mechanism (and its consequences)

Layers extend the game through three techniques:

1. **Override by redeclaration** — `function scheduleDay(){…}` declared again in a later layer. Function hoisting means the *last* declaration in the file wins. There are **4 declarations of `scheduleDay`**, 4 of `finalizeStay`, 4 of `dayRollover`, 3 of `tickHour`, 3 of `genGuest`, 3 of `pickChannel` (+1 reassignment), 3 of `guestRoutine` (+1 reassignment), 3 of `renderTab` (+5 wraps), 2 of `roomDetail`, 2 of `newGame`, etc. Every superseded body is **dead code that still ships** (roughly 25–30% of the JS).
2. **Wrap by reassignment** — `const _v10DR=dayRollover; dayRollover=function(){…_v10DR()…}`. `dayRollover` ends up wrapped **seven deep** (v5→v6→v7→v9→v10→v11→v12→v13 chain). Execution order within a day rollover is consequently very hard to reason about (see §6, timing bugs).
3. **Rendered-HTML string surgery** — v13 patches *other tabs' rendered HTML* with regex/`indexOf` (e.g. `tabFinance` strips the old "What changed" card by scanning for its heading; `tabStaff` injects 🎓 buttons by regexing `<td>NN%</td></tr>`). This is the most fragile pattern in the codebase: any wording change silently breaks the patch.

**State model.** One global `G` (game state) with `G.properties[]`; `P()` returns the active property. Persistence is `JSON.stringify(G)` into a single `localStorage` slot, with schema backfill on load via the `augmentInit`/`augmentProp`/`ensureV10`/`ensureV12` chain — this part is genuinely well done and keeps old saves loadable.

**Loop.** `setInterval(80ms)`; 1 in-game hour = 3000ms ÷ speed. `tickHour` handles arrivals → departures → housekeeping → maintenance → guest mood/routines → random events → hourly cost accrual. `dayRollover` (at hour 24) settles finances, runs the daily subsystems, then `scheduleDay` generates the next day's arrivals.

**Rendering.** Full innerHTML re-render of `#main` per tab via `renderTab()`; `refreshLive()` re-renders a hardcoded subset of tabs each tick (`overview, rooms, guests, finance` — **stale**: newer tabs like Operations, Reservations, Roster never live-refresh). Event handling is a mix of inline `onclick="window.fn(...)"` globals (~120 of them) and `wireTab()` (which only wires the Pricing tab).

### 1.3 Verdict

The layered idiom was a rational way to grow a single file without regressions, and the backfill chain keeps saves compatible. But at 13 layers it has passed the point of maintainability: most core functions have 2–4 dead ancestors, day-rollover ordering is emergent rather than designed, and at least five bugs found below are *interaction bugs between layers* (a later layer wrapping a reference an even-later layer replaced). The single most valuable structural investment is **flattening each overridden function to one authoritative implementation** (no behavior change), then extracting the layers into modules: `data`, `engine` (demand/booking/stay), `departments`, `staff`, `finance`, `ui`.

---

## 2. Cross-System Map (what feeds what)

Working connections (verified in code):

```
Procurement (INV stock) ──▶ Housekeeping quality (cleanScore −25 on stock-out)
                        ──▶ F&B quality (−22) ──▶ satisfaction.fnb
                        ──▶ Maintenance speed (spares)
Roster coverage ──▶ cleanCapacity / FO capacity / repair speed / F&B covers
Housekeeping ──▶ room readiness ──▶ check-in success ──▶ walkouts ──▶ reputation
Satisfaction ──▶ reviews (star bands, aspect-consistent text) ──▶ rating
Rating/reputation ──▶ repFactor & compFactor ──▶ computeDemand ──▶ bookings
Wear (cond) ──▶ satisfaction & review flavour ──▶ renovation loop
Marketing ──▶ awareness/segBoost/directBoost ──▶ channel mix ──▶ commission costs
Loyalty ──▶ repeat pool, discounts, perks ──▶ member satisfaction delta
Finance ──▶ loans/EMI/investor share ──▶ cash ──▶ everything gated on cash
Macro (inflation/air traffic/supply/GST) ──▶ costs & demand
```

Systems that are **isolated or only half-connected**:

- **Non-active properties are frozen.** `tickHour`, `dayRollover`, `accrueHourlyCost`, `scheduleDay` all operate on `P()` only. A second hotel earns nothing, spends nothing, and receives no guests until you switch to it — its staff salaries are free, its rooms never wear. The HQ/chain layer aggregates stats over properties that don't actually simulate. **This is the largest realism/integration gap in the game.**
- **Reservations (v12) run parallel to bookings.** `genResv` creates named reservations with deposits, but those people never arrive; on their day they just add a scalar to `computeDemand`, and `scheduleDay` invents *different* guests. Two booking systems coexist (see bug B7).
- **Room components (`COMPONENTS`, 27 items)** still influence satisfaction (`compScore`, soundproofing, fastwifi) but became **unreachable** when v10 replaced `roomDetail` with the standardization view — no UI can install a component any more. Dead-but-load-bearing system.
- **Security staffing** only matters via one rare night-incident roll (`v11Security`); it has no connection to reviews, guest mix, or events.
- **Weather** affects demand and pool/sightseeing routines but never housekeeping, maintenance (monsoon damage), or arrivals.

---

## 3. Confirmed Bugs

Ordered by impact. "FIXED" marks items corrected in the Wave-1 commit accompanying this audit; the rest are roadmap items.

| # | Bug | Location | Detail |
|---|---|---|---|
| **B1** | **Guests stranded forever at an unmanned desk** — FIXED | `tickHour` (l. 3275) + v11 `checkInGuest` (l. 5790) | The due-arrivals filter requires `a.arrHour===G.hour`. When nobody is rostered and no manager exists, `checkInGuest` marks the guest waiting (`fdWait`, "walks out after 3 hours") — but the guest is never re-processed on later hours, so the walkout logic is unreachable; the guest silently evaporates at day rollover with the room still reserved. |
| **B2** | **Room build quality silently broken** — FIXED | v9 `v9ConfirmRooms` (l. 5203) vs v10/v13 `_buildRoomNow` wraps | `v9ConfirmRooms` calls the *closure-captured original* `buildRoom`, bypassing the v10/v13 wrappers that set the type-implied furnishing tier. The v4 original then reads a `#q_<rt>` select that no longer exists → **every room is built at "Basic" quality**, while the v13 modal displays (and cash-checks against) the higher tier price. Deluxe/suite/exec rooms under-deliver forever. |
| **B3** | **`bump()` mutates the wrong guest** — FIXED | l. 890 | `pick(occ).guest.sat = clamp(pick(occ).guest.sat + amt …)` — two independent `pick()` calls: guest A's satisfaction is overwritten with guest B's ± amt. Every event choice that uses `bump` (most of them) corrupts a random guest's satisfaction. |
| **B4** | **Simulation state mutated from render path** — FIXED | v12 `compFactor` wrap (l. 6602) | The wrap decays competitor campaigns (`c.camp -= 1/24`) inside `compFactor`, which is called from `computeDemand` *and* from Reviews/Market/Overview renders. Campaign duration therefore depends on how often the player looks at certain tabs. |
| **B5** | **ID counters not persisted** — FIXED | `save()`/`load()` | `roomSeq/staffSeq/maintSeq/projSeq/resvSeq` are module globals reset to 1 on page reload. After loading a save, new rooms/staff/issues receive IDs that collide with existing ones — `roomDetail(id)`, `fixIssue(id)`, arrival `roomId` mapping and dismiss buttons then act on the wrong object. |
| **B6** | **Mobile bottom-nav "More" does nothing** — FIXED | v12 (l. 6879) | Toggles `#side.classList('open')`, but the drawer CSS is driven by `#app.navopen`. Same wrong class in two other nav helpers. On phones the "More" button is a dead control. |
| **B7** | **Reservation deposits create money from nothing** — FIXED (Wave 2) | v12 `genResv` (l. 6350–6385) | Deposits are added to `p.escrow` (a display number) but never to cash at booking; on cancellation/no-show, `G.cash += keep` credits money that was never collected. Small but steady phantom revenue, and `escrow` misrepresents liabilities. |
| **B8** | **F&B revenue is double/triple-counted** — FIXED (Wave 2) | v5 `finalizeStay` (l. 4026) + v7 `guestRoutine` `use()` + v7 `runDepartments` + v10 `runFnB` | A guest pays per-meal during the stay (dept pricing), then `finalizeStay` adds a *second* lump-sum breakfast/lunch/dinner "extra" for the same nights, `runDepartments` adds external covers, and `runFnB` re-derives restaurant P&L and applies deltas on top. Restaurant economics are inflated and unexplainable. |
| **B9** | **Two competitor AIs run simultaneously** | v4 `competitorAI` (daily) + v12 `compBrain` (weekly) | Rivals renovate/reprice under both systems; v4's market-entry cap (`comps<7`) contradicts v13's `ensureCompDensity` (refills to ≥10), so exits are instantly refilled and the entry event never fires. |
| **B10** | **`roomDetail` misreports tier & equipment** — FIXED (Wave 2) | v10 (l. 5627) | Reads `QUALITY[r.quality||r.q||0]` — `r.q` is the quality *object*, so the index lookup fails and every room displays "Basic". Also reads `r.comps` (never written; components live in `r.comp`) so installed equipment never lists. |
| **B11** | **Game-over summary immediately overwritten** — FIXED (Wave 2) | v8 wrap of `gameOver` (l. 4992) | Wrap shows the performance summary, then calls the original which replaces `#mbox` with the insolvency modal — the summary is never visible. |
| **B12** | **`runFnB` reads `p._demandBreak.final`** — FIXED (Wave 2) | l. 5598 | Field is named `total`; `final` is always undefined → table-turns term is constant. |
| **B13** | **Autosave captures mid-rollover state** — FIXED (Wave 2) | v6 wrap (l. 4523) | The autosave sits deep in the wrap chain, so v9–v13 post-rollover steps (assets, F&B recompute, OTB, agency costs, overtime deduction) run *after* the save. Loading an autosave replays a day that half-happened. |
| **B14** | **`G.guestDB` grows without bound** — FIXED (pruned) | `writeGuestDB` | Every guest ever is retained; long games bloat the save and the return-pool scan. |
| **B15** | **`chain.loyaltyMembers` counts every guest** — FIXED (Wave 2) | `writeGuestDB` (l. 1797) | Incremented on *first* stay, so HQ's "Loyalty Members" KPI and the old milestone count all unique guests; the Loyalty tab computes real members differently. Two contradictory definitions on screen. |
| **B16** | **`refreshLive` tab list stale** — FIXED (Wave 5) | l. 948 | Live re-render only covers `overview/rooms/guests/finance`; Operations, Reservations, Roster, Analytics show stale data during play until re-clicked. |
| **B17** | **Check-in time defaults disagree** | `augmentProp` (13) vs `checkinWeights`/reports (14) | Cosmetic off-by-one in the arrival curve and report display. |
| **B18** | **Mobile FAB speed button is dead** — FIXED | v12 (l. 6883) | Calls `cycleSpeed&&cycleSpeed()` — never defined; the ⏩ button is a no-op. `cycleSpeed` implemented in v14. |

---

## 4. Redundancies (one mechanic, multiple implementations)

| Area | Duplicates | Authoritative version should be |
|---|---|---|
| **Policies UI** | Pricing tab still renders the v1 "Operating Policies" card (earlyCI/lateCO/freeUpg/generosity) duplicating the Policies tab — FIXED (removed from Pricing) | Policies tab |
| **Valet** | v4 `enableValet` shopitem (₹2.5L one-time, wrong price display) *and* v10 tiered valet card, both on Expansion — FIXED (legacy shopitem removed) | v10 tiered valet |
| **Room status displays** | Rooms grid + v8 Room Status Board + v12 Room Lifecycle board (3 color legends, 2 on the same Operations tab) | One lifecycle board on Operations; Rooms tab keeps the interactive grid |
| **Satisfaction driver cards** | Reviews tab (v5 wrap) and Analytics tab (v11) render the same `compAvg` table | Analytics |
| **Milestones vs Goals** | Expansion tab "Milestones" card duplicates the Goals page (with older thresholds) | Goals page |
| **Training** | HQ `trainAll` (chain), v5 `trainProperty` (card already removed by v13, function dead), v13 per-employee + auto-budget | v13 system; HQ button should feed it |
| **Booking pipelines** | v10 `otbGen` scalar OTB → replaced by v12 `genResv`, but `scheduleDay` still invents all actual arrivals | Merge: reservations should *be* the arrivals (see roadmap R2) |
| **Competitor AI** | v4 `competitorAI` + v12 `compBrain` + v13 `ensureCompDensity` | One strategist AI with entry/exit rules |
| **Direct-booking boost** | v3 `marketDirect` (dead global, UI removed) + v4 marketing campaigns | Marketing campaigns |
| **Dead code** | ~2,000 lines of superseded function bodies (v1–v4 versions of the core loop, `maybeRequestEvent` no-op, `maturity()`, `svgLine`, `spark` users, `COMPONENTS` UI, `upgradeRoomQ`, `buyComponent`, `renovateRoom` v3, `addParking/addEV` v2, `G.brand`, `repTrend`) | Delete during flattening |

---

## 5. Dead / no-effect settings and controls

- `policies.earlyFee` / `lateFee` — collected in UI, **never charged anywhere**.
- `policies.earlyCI` / `lateCO` / `freeUpg` — toggles with **zero simulation effect** (upgrades are governed by loyalty `upg`; early check-in exists only as a random event).
- `policies.parking === 'reserve'` — behaves identically to complimentary in `assignParking`.
- `policies.freeCancel` — legacy field superseded by `cancelType`, still seeded in `newGame`.
- Settings → `cbMeters` (colour-blind meters) only tweaks contrast of two classes; doesn't recolor bars.
- Mobile FAB ⏩ (B18, fixed), bottom-nav More (B6, fixed).
- `AGENCY` tier for `shuttle`/`traveldesk` renders on department pages, but `deptCapacity` for those keys ignores the agency multiplier except through the generic default branch — capacity effect is negligible.

---

## 6. Realism Audit (hotel logic vs game logic)

**Already strong** (worth preserving): construction takes time with permits/contractors/overruns; staff have notice periods, contracts, overtime at 1.5×, night premiums, burnout→resignations; procurement has lead times, supplier reliability, stock-outs that cascade into housekeeping and F&B; GST fence at ₹7,500; USALI departmental P&L; class-based guest expectations; service-recovery paradox; loyalty as a costed liability.

Gaps, in priority order:

1. **Multi-property freeze** (§2). A chain where only the hotel you're looking at exists breaks the entire expansion pillar. Every property needs at least a daily coarse simulation (revenue/cost/reputation drift from its own staffing & pricing) even when not active.
2. **Bookings vs reservations split** (B7). Real hotels sell *future* room-nights; here, the reservation book is decorative and demand is re-rolled nightly. Merging them (reservations become the arrivals; walk-ins fill the gap) would also fix deposit accounting and make overbooking/no-shows meaningful.
3. **F&B accounting** (B8): one revenue path — per-use department pricing — with covers/capacity limits; remove the checkout lump-sum and the `runFnB` re-derivation.
4. **Deposits/escrow**: collect deposit into cash at booking, hold as liability, net at checkout; cancellations then naturally retain the right amount.
5. **Housekeeping consumes inventory per room cleaned** (currently consumption is occupancy-scaled, disconnected from cleans), and laundry backlog should delay *specific* rooms, not add abstract strain.
6. **Events should respect state more**: "Power Outage" ignores whether you have a generator (no such asset), "Festival surge" changes prices permanently with no revert, VIP arrival isn't an actual guest.
7. **Renovation should scale with wear** and room count offline should affect sellable inventory forecasts (it does affect availability, but the reservation pipeline ignores rooms under renovation).
8. **Rate parity toggle** (`rm.parity`) has no effect on OTA behavior.

---

## 7. UI Audit

- **Navigation** (post-v13 grouping) is sound: Home / Operate / Commercial / Property / Money / Company. Issues: Attention Center still deep-links to removed `frontoffice` tab (redirects, but no nav item highlights); `reservations` sits under Commercial via insertion order, not the GROUPS config.
- **Overview** is overloaded: v8 attention center + score strip, v5 class/demand/GOPPAR strip, v1 four KPI cards, v12 reputation-by-audience card. The reputation card duplicates Market/Reviews content — better on Reviews.
- **Finance** is the longest page in the game (KPIs, chart, financing, channel table, P&L, balance sheet, cost structure, USALI, department P&L, capital actions, trends, investor relations, what-changed) — needs sub-tabs (P&L / Financing / Channels / Departments).
- **Charts**: v13 engine (zero-axis, tooltips, legend toggles) is good; the finance bar chart from v1 remains as a second, inferior chart style on the same page.
- **Tooltips** (`TIPS`) cover ~20 concepts; KPI topbar deep-links work (v13 remap).
- **Mobile**: six injected style layers with four different breakpoints (980/900/1200/520). Final result mostly works (drawer + bottom nav + collapsible cards), with the two dead controls noted (B6 fixed, B18 open). Tables rely on `min-width` + horizontal scroll, acceptable. The landing/wizard renders fine on small screens.
- **Light theme** exists twice (v9 base + v11 refinement); v11 wins; a few components (event modal icon chips, roster grid tint) still assume dark background.

---

## 8. Code Quality Notes

- Naming is terse but internally consistent (`P()`, `G`, `cr()`, `row()`); the biggest readability cost is **not** naming, it's the 13-layer override archaeology — a reader cannot know which `finalizeStay` runs without scanning the whole file.
- The v13 pattern of regex-editing rendered HTML (`tabFinance` replace, `tabStaff` train buttons, `cutCard`) must not be extended; each new UI change through this path compounds fragility.
- `try{}catch(e){}` swallows errors in ~60 places — during flattening, keep the defensive wrappers only at layer boundaries that still exist.
- Every `onclick` handler is a `window.*` global (~120). Fine for a single file, but any modularization must start by inventorying them.

---

## 9. Prioritized Roadmap

**Wave 1 — surgical bug fixes (done in this branch, appended as a `v14` hotfix layer + 4 in-place one-liners, following the file's own extension idiom):**
1. B1 stranded-lobby guests, B2 room build quality/pricing, B3 `bump()`, B4 campaign decay in render path, B5 save/load ID counters, B6 mobile More button, B14 guestDB pruning; UI consolidation: duplicate Operating Policies card and legacy valet shopitem removed.

**Wave 2 — money is money (done in this branch, `v15` layer + in-place edits):**
2. F&B unified into the department model (B8): checkout lump-sums removed; in-house guests pay per use, external covers are menu-aware and seats×turns capacity-gated; `runFnB` is now a derived covers view instead of a parallel revenue path (also resolves B12).
3. Deposit accounting (B7): deposits are collected into cash at booking, held in a visible `depositsHeld` liability, netted off the bill at checkout, refunded on walkouts and forfeited per cancellation policy on no-shows (settled before every `scheduleDay`); the virtual reservation pipeline no longer credits uncollected deposits.
4. Autosave moved to the true end of rollover (B13); HQ loyalty-member count computed from real membership (B15); insolvency screen keeps the final summary reachable (B11); `roomDetail` shows the true furnishing tier and equipment (B10).

**Wave 3 — one booking engine (done in this branch, `v16` layer):**
5. Reservations ARE the arrivals: advance bookings collect real deposits at booking, materialise into actual guests on their day (rooms held, usual-room requests honoured), no-shows and cancellations settle per cancellation policy, and walk-in demand fills only what the book leaves over. The old scalar OTB demand boost is retired — booked rooms are supply, not a demand multiplier.

**Wave 4 — the chain is real (done in this branch):**
6. Non-active properties now always accrue their real costs (payroll, base utilities, facilities) — no more free hotels — and drift downhill in reputation while unmanaged. With an HQ Regional Operations Manager, they run a daily simulation driven by their **own** market (city/locality demand, season), pricing vs their market, housekeeping/front-desk staffing adequacy, and room condition — producing revenue, reviews, wear and analytics. Hiring the regional manager is now a real decision, not free money.

**Wave 5 — consolidation & flattening (done in this branch):**
7. **Flattened**: 75 dead ancestor function declarations (1,571 lines) removed with a parser-verified, behavior-neutral pass (only top-level declarations shadowed by a later declaration of the same name — hoisting guarantees the last wins). Competitor AI consolidated: the daily loop keeps tactical price drift and entry/exit only; strategic plays (renovations, campaigns, facilities, acquisitions) belong solely to the weekly `compBrain` strategist, and the entry cap now agrees with the density floor. Room-status boards merged (v12 lifecycle board is the one display); Expansion milestones folded into the Goals page; `freeCancel` seed and the no-op `reserve` parking option removed. **Dead policies implemented**: early check-in and late checkout now charge their fees (or delight when complimentary), guests made to wait for standard check-in lose satisfaction, and the complimentary-upgrades toggle grants real upgrades.
8. Finance split into sections (Everything / P&L / Channels / Financing / Departments) via a depth-scanned block filter; reputation-by-audience moved from Overview to Reviews; the old v1 bar chart removed in favour of the interactive chart engine; `refreshLive` now covers the newer live tabs (closes B16).

**Wave 6 — realism polish:** inventory tied to cleans, event-state coherence (generator asset, price-surge revert, VIP as real guest), rate-parity effect, weather↔operations links, security↔reviews link.


---

## 10. Phase 10 — Enterprise layer (`v17`, this branch)

Implemented as extensions of existing systems (no parallel mechanics):

1. **Revenue Management System** — three control modes (Manual / Revenue Manager / AI) on the existing `p.rm`; five named strategies (Aggressive Occupancy, Maximum Profit, Balanced, Premium Positioning, Budget Leader) with floors/caps; decisions driven by demand forecast, advance pace, weekends, festivals, competitor price index, rating and reputation — **every rate change is logged with its reasons** on the Pricing tab. Manager mode adds skill-scaled judgement noise; `managerPricing` folded in (one pricing brain). Direct-booking incentive knob added.
2. **Guest memory** — deepened on the v12 persona system: per-guest preference counters (breakfast/restaurant/laundry/spa/parking), upgrade & discount history, usual-room requests honoured at materialisation (+satisfaction), happy repeaters budget higher, past upgrades raise expectations, Gold/Platinum members weighted in the return pool. Guest Memory card on Front Desk.
3. **Utilities** — metered model (electricity, water, gas, internet, waste, generator fuel) with per-department consumption replaces the flat util+energy line in `accrueHourlyCost`; six efficiency investments permanently cut specific utilities; failures (power/water/internet) hit operations unless mitigated (generator, water storage). Dedicated Utilities tab with trends.
4. **Preventive maintenance** — building systems (HVAC, pumps, wiring, elevator, kitchen, laundry machines, generator) with condition, service age, interval and risk score; overdue systems break down into the existing maintenance backlog at ~3× the service cost with system-specific operational fallout; per-asset auto-service.
5. **Security department** — CCTV/access/fire investments + staffing → guest-confidence score; low-probability incidents (theft, noise, medical, false alarm, vandalism) whose frequency and severity shrink with the score; VIP/premium guests read security posture at check-in.
6. **HQ** — Regional Operations Manager makes the chain real (see Wave 4); brand-standards audits lift the weakest property; loyalty KPI now computed from real membership.
7. **IPO & capital markets** — six listing requirements, bank selection (fee/valuation trade-off), 7-day roadshow with sentiment, listing raises 25% float; daily share price from profit trend + sentiment, quarterly results vs expectations, dividends and secondary offerings, under-water-stock pressure.
8. **AI managers** — managers carry personalities (7 archetypes); the highest-skill manager acts as GM, steering manager-mode pricing strategy, department service models, preventive-maintenance automation and overload hiring — all logged, all overridable by switching to Player control.
9. **KPI cockpit** — 20 professional hospitality KPIs (ADR, RevPAR, TRevPAR, GOPPAR, ALOS, GAC, repeat rate, direct ratio, food/labour cost %, HK/maintenance/utility unit costs, complaint rate, GSI, NPS, check-in wait, turnaround, response time), each clickable for definition, target, 30-day trend and improvement levers — computed live from simulation data.

Verified headless: 51 reservation guests materialised over 14 days; deposit liability exact to the rupee including future reservations; RMS log populated with reasons; utilities/systems/security accruing; background chain revenue flowing; IPO reaches listing; 26 tabs + KPI drill-down render with zero errors.


---

## 11. Visual identity (`v19`, this branch)

The frame moved off the generic dark-navy/gold dashboard look onto a subject-grounded identity — **"Night Audit / Day Ledger"**:

- **Dark (Night Audit)**: ink-green ground with aged-brass accent, Palatino-family serif for display type (page titles, stats, brand, clock), double-rule section headings, squared 3–4px corners, and an iconless editorial sidebar (section labels + brass rule for the active entry) instead of the emoji app-drawer.
- **Light (Day Ledger)**: a fully designed warm-paper theme — ivory grounds, deep green-black ink, brass, with every component re-themed at the token level (pills, toggles, inputs, notices, reviews, charts tooltips, modal, wizard, bottom-nav, FAB) rather than the previous washed-out inversion.
- Token-level implementation: the v19 stylesheet is appended last in the cascade and redefines the CSS custom properties, so hundreds of existing templates restyle automatically; hardcoded navy leftovers (bar tracks, hover rows, pills, switches, icon chips) get spot overrides.
- Landing screen redesigned as a framed register plate (single-theme by intent); fixed a real mobile bug found during review — the setup wizard used flex-centering with `overflow:auto`, clipping its top on phones (`.wiz{margin:auto}`).

Verified with themed screenshots (landing, wizard, dark overview/finance, light overview/pricing, mobile) and the full regression suite: zero console errors, all 27 tabs render, economy invariants unchanged.
