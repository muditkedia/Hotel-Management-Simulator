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


---

## 12. Validation audit — realism, logic & balance (`v20` + calibration, this branch)

A QA/hotel-consultant pass over every value, formula, timeline and setting, validated empirically with 60-day economic probes (16-room Mumbai midscale, full staff, restaurant/breakfast/laundry/wifi).

**Formula bugs fixed**
- **Facility running costs were charged twice** — once in the hourly opex line and again as departmental fixed cost in `runDepartments`. Dept-run facilities now carry their fixed cost only in their department P&L.
- **Guests ate 3–5 restaurant meals a day** — only breakfast had a once-per-day flag; lunch and dinner re-rolled every hour. Both now capped at one per day, cutting F&B revenue ~30% to a realistic 15–25% share of total revenue.
- **Two contradictory asset valuations** — `assetValue()` (loan capacity, credit rating) used ₹9L/room while `chainAssets()` (IPO) used ₹13.5L/room. Unified on the chainAssets basis.
- **Finance "Monthly Cost Structure" showed a stale flat utilities estimate** (rooms×250) instead of the metered utilities model. Now reads the meter.
- ALOS KPI zeroed out every 30th day (window reset artifact) — now falls back to the previous window.

**Missing cost realism added**
- **Fixed overheads line** (insurance, property tax, licences, AMCs, G&A, reserves): ₹550/room/day — a real hotel P&L line that was entirely absent and a major driver of the game's inflated 65% margins.
- Internet was ₹20k+/month for a 20-room hotel — halved to market rates.
- **Dismissal now costs half a month's severance** and dents team morale; instant free firing removed.

**Demand realism (the biggest imbalance)**
- The probe showed **79% occupancy at ₹10.6k ADR on a 2.56★ rating** — a rating the demand model ignored entirely, and the advance-reservation pipeline filled the book regardless. A **rating gate** now applies to both walk-in demand and OTA advance bookings (`0.45 + rating/5 × 0.62`, clamped 0.5–1.12; unrated new hotels get 0.85 intro visibility) — a 2.5★ property books meaningfully less, a 4.5★+ property earns a premium. `repFactor` spread widened (0.45–1.40) so weak reputation genuinely hurts.
- Restaurant walk-in covers now follow the hotel's rating strongly (a no-name 2.5★ draws few outside diners).
- Review propensity cut from ~30% of guests to ~15–20%, matching real post-stay OTA review rates.

**Dead settings wired or removed**
- `roomsvc24` (dead since Wave 2): now opens a 23:00–05:00 room-service window at a 15% surcharge.
- `bookingConfidence` (displayed, never read): now a ±10% demand factor.
- `rm.parity` (OTA rate parity, never read): breaking parity now costs ~12% of OTA bookings (ranking demotion) while direct guests gain goodwill.
- `cbMeters` (imperceptible contrast filter): now swaps the semantic palette to a colour-blind-safe blue/orange pair in both themes.
- Duplicate laundry pricing control removed from Policies (the department page owns it); `policies.laundry` had no remaining sim effect.

**Timelines corrected**
- Room construction 3–5 → 4–7 days; new floor 8–12 → 12–18 days (copy updated); hiring notice period 1–3 → 2–5 days (including GM auto-hires). Standard check-in default aligned to 14:00 (the arrival-curve anchor); arrivals may now land at 1–5 am, which airport/transit hotels really see.
- IPO valuation multiple raised from 4× to 8× annual profit (hotel-sector EV/EBITDA norms).
- KPI "Food Cost %" renamed to "F&B Dept Cost Ratio" with an honest definition (true food cost alone runs 28–35%).

**Post-calibration probe** (same setup): occupancy ~70–75% at 2.5★ (was ~80%), revenue −15–20%, expense model complete, F&B share realistic, laundry roughly break-even at small scale (as in real small hotels), payroll ~7% and utilities ~4% of revenue. Remaining margins (~55–65%) reflect deliberate model boundaries — an owned, debt-free property with no rent or above-property corporate overhead, on a compressed timescale — and are documented rather than hidden.

Settings verification: severance charges exactly ₹sal/2; the night room-service window produces measurable usage and revenue; every fix regression-tested (deposit ledger still exact to the rupee, 27 tabs clean, zero console errors).


---

## 13. Phase 11 — Strategic planning & operational realism (`v21`, this branch)

**Removed / consolidated (Part 1)**
- Hotel Identity panel removed — positioning still forms silently from the guest mix and steers segment demand, but players read it from operations, not a dashboard label.
- Reputation display consolidated to three measures: **Guest Satisfaction** (in-house, Overview), **Google Rating** (public, Reviews) and **Brand Reputation** (long-term, topbar/Reviews). The nine-audience card is gone; its segment-expectation mechanics remain internal.
- The duplicate Operating Policies card was deleted from the Pricing *template* itself (previously only cut at render time); Policies is the single authoritative home, cross-linked.

**Corporate Sales (Part 2.1)** — companies (name/industry/HQ/annual room-nights/preferred category/budget/term/patience) rotate into a pipeline sized by the city's business share. Pitches cost ₹25k and are scored on offered rate vs budget, Google rating, brand reputation, business facilities (conference, business centre, shuttle, restaurant, Wi-Fi), awards, corporate-guest history and front-desk pace. **Won contracts inject real corporate reservations into the v16 booking pipeline** at the negotiated rate; delivered nights and traveller satisfaction are tracked per account and decide renewal; unpitched prospects eventually sign with rivals. Renewals lift brand reputation; terminations dent corporate standing.

**Capital Planning (Part 2.2)** — a dedicated CapEx page that reuses every existing investment handler (renovation waves, floors, parking/EV, all six utility investments, security upgrades, building-system overhauls) plus one new PMS technology upgrade (+12% front-desk capacity). Each row computes capital cost, useful life, estimated annual benefit from live RevPAR/utility meters, and payback — with financing cross-linked to loans/investors.

**Awards & Certifications (Part 2.3)** — eight deterministic recognitions (Travellers' Choice, Best Business, Best Value, Green Certification, Luxury/Food/Cleanliness/Service Excellence), each earned only by meeting its published standard on **two consecutive monthly reviews**, valid 180 days, and lapsing if the standard slips at renewal. Held awards add +2% demand each, weight corporate pitches, boost brand reputation, awareness and listed-company sentiment.

**Workforce overhaul (Part 3)**
- Roster rebuilt around real hotel shifts (Morning 06–14 / Afternoon 14–22 / Night 22–06, nights spilling into the next day) as an editor over the existing hourly engine — full save compatibility, existing caps and premiums intact. Cells read "on duty / needed."
- **Auto-roster rewritten**: covers one person on every needed hour first, then fills to full need; spends contracted hours before touching the overtime pool; staffs only the hours that need cover (skeleton-crew pattern — a single 22:00 need no longer buys a whole night shift); reports honestly when coverage is impossible within legal caps ("hire ~N more"). Staff arrays are ordered so the most experienced take the shift first.
- **Hard staffing gates**: no maintenance crew on shift → repairs wait (the old 0.5 free floor removed); no F&B staff on shift → the restaurant serves no meals in-house or external and guests notice at dinner; front desk, housekeeping, security and procurement gates already existed. Stoppages surface in the Attention Center with the reason.
- **Workforce analytics**: per-department coverage gaps, weekly hours, overtime, labour ₹/day and labour per occupied room — on the Roster page and in the Daily Report.

Verified end-to-end headless: a pitched contract signs, injects corporate reservations, delivers nights and tracks traveller satisfaction; three awards earn after exactly two monthly reviews and measurably raise demand; auto-roster fully covers morning need within budget; unrostered maintenance stops repairs and an unstaffed kitchen serves zero lunches; all 28 tabs render; the full regression (deposit ledger exact, chain background sim, policies fees) stays green.

---

## 14. Internationalisation — Grand Stay (`v22`, branch `claude/aatithya-international`)

*Separate branch from the India build (`claude/aatithya-audit-jnh84c`), which is untouched.*

The game is de-localised from India-only into a three-region simulator (India · USA · Europe), Tier 1–2 cities only.

**Architecture** — one balanced base economy, localised at the display layer. All internal money stays in a single base unit (the tuned Indian economy), so gameplay balance is preserved everywhere and every legacy currency literal converts automatically. `cr()`/`cr0()` became currency-aware (symbol, per-country display divisor `k`, Indian lakh/crore vs Western K/M/B grouping). A `locStr()` pass converts static ₹-labels baked into events, objectives and achievements to the active currency; dynamic template literals (loans, valet, loyalty) now route through `cr()`.

**Per-country profile** (`COUNTRIES`): currency + display divisor; construction (`capex`), wage and starting-capital multipliers applied to the captured base constants at `applyCountry()` (re-applied on load, save-safe via `G.country`); tax model (India GST slab · US 15% occupancy/sales · Europe 10% VAT); a curated Tier 1–2 city list with per-city rate/demand/competition/mix/seasonality; local first/last-name pools; local competitor brand pools; local hotel-name generator; country-neutral seasonality (India festivals vs Western summer/year-end peaks).

**Cities** (8 each, Tier 1–2 only): India — Mumbai, Delhi NCR, Bengaluru, Hyderabad, Pune, Ahmedabad, Jaipur, Kochi. USA — New York, San Francisco, Los Angeles, Chicago (T1), Boston, Austin, Denver, Nashville (T2). Europe — Paris, Amsterdam, Frankfurt, Munich (T1), Barcelona, Rome, Vienna, Lisbon (T2).

**Wizard** — step 0 becomes "Choose your market": a country row (flags) above the city grid; picking a country re-applies its economy and swaps the city list live. Difficulty/confirm show capital and rates in the chosen currency.

**Verified** (45–60-day probes per country, headless): India ₹ / GST / Mumbai / 56% margin — **identical to the single-country build**; USA $ / 15% tax / New York / std rate ~$288, housekeeper ~$2,000/mo, build ~$59k, start ~$494k / 46% margin (realistically tighter on higher labour); Europe € / 10% VAT / Paris / ~€327 rate / 53% margin. Guest names, competitor brands, hotel names and taxes all localise; zero ₹ leaks in the USA/Europe UI; all Phase-11 systems (corporate, awards, roster gates) and the full regression stay green in every region.

**Brand** — "Aatithya / आ" retired for the generic international mark **Grand Stay** across title, topbar, wizard and landing plate.

---

## 15. International build — full QA pass (`v22` follow-up, this branch)

A 100% sweep of the international build: every tab (28), every department page, and the room/KPI modals rendered headless in all three countries and scanned for wrong-currency output, `NaN`/`undefined`/`Infinity`, and render exceptions; every formula with an absolute money constant audited for country-scaling; every `type="number"` money input audited for internal-vs-local units.

### Fixed — currency display leaks (visible in USA/Europe)
| Where | Was | Now |
|---|---|---|
| Pricing table header | `Your Rate (₹)` | `Your Rate ($/€/₹)` |
| Roster analytics column | `Labour ₹/day` | active symbol |
| Staff — auto-train budget row | `₹` prefix | active symbol |
| Utilities — per-occupied-room benchmark | `~₹450–900` hard-coded | converted via `cr0()` (`~$13–$26`, `~€15–€30`) |
| Policies — early/late fee & EV price labels | `(₹, …)` | active symbol |
| F&B free-breakfast note | `revenue is ₹0` | active symbol |
| Sidebar Pricing icon | `₹` glyph | neutral 💰 (icon is CSS-hidden anyway) |

### Fixed — money inputs now read/write local currency
Previously these inputs displayed the raw internal (India-base) number while surrounding text showed converted currency — in the USA the standard rate input read "9000" beside a "$265" market reference. All now display local currency and convert back on change (`÷ LOC.k` on render, `× LOC.k` on commit):
- **Room rate inputs** (Pricing) + Apply Rates handler
- **Policy fees** — early check-in, late checkout, EV charging (`polNum` gained a money mode)
- **Monthly auto-train budget** (step localised too: 200 in $/€, 5000 in ₹)
- **Investor raise amount** (min/step/default localised)
- **Loyalty redemption value** kept as an abstract per-point number (points are internal-scale), but the misleading ₹ prefix was replaced with an honest conversion hint: "1,000 pts ≈ $12".

### Fixed — formulas that ignored country cost levels
- **`assetValue()` / `chainAssets()`** used flat ₹13.5L/room + ₹23L/facility — a US hotel was valued at ~35% of its build cost (asset-to-build ratio 1.0 vs India's 2.79), understating loan capacity, IPO valuation and net worth. Now scaled by country `capex`; ratio is a uniform 2.79 everywhere and borrowing capacity for the same 14-room hotel is ₹1.26 Cr / $1.04M / €1.10M as expected.
- **Floor construction** (`2500000 + floors×800000`) — unscaled while room builds were 2.8× dearer; now × `capex` (all four call sites, CapEx plan included).
- **New-hotel setup cost** (₹1.5 Cr flat, with a hard-coded ₹ toast) — now × `capex` ($1.24M / €1.14M) with a converted toast.
- **Quick loan / investor buttons** (₹20L/₹50L/₹1.5Cr/₹1Cr flat) — now × `capex` so financing options stay proportionate to build costs.
- **Housekeeping KPI target** (₹350/occupied room, India-calibrated) — housekeeping payroll is wage-scaled (×3.2 US, ×3.0 EU) so the KPI failed structurally abroad. Target now scales with the housekeeping salary ratio (≈$33/€35), matching how the metric's numerator scales. GAC/maintenance/utility targets are *not* wage-scaled internally, so their India-calibrated targets remain fair and were left alone.

### Fixed — content bug (all countries)
- **Staff tab printed `undefined`** for the Procurement Manager hire card: `roleIcon()` predates the v17 procurement role. Added its icon (📦) plus a safe fallback.

### Verified clean after fixes
- Sweep: **0 issues** across 28 tabs × 3 countries + department pages + modals; no page errors.
- Money-input round-trips: typing $275 stores 9,350 internal; fee/train/investor inputs round-trip exactly (k=1 and k=34 both tested).
- KPI probe: HK cost/occupied-room passes in all three countries with honest effort; GOPPAR intentionally remains tougher in the US (real labour-cost structure).
- Full regression (`intl`, `leakcheck`, `smoke4`, `smoke5`): India economy byte-identical behaviour (₹6,900 std rate, 60% margin), deposits ledger exact, Phase-11 systems green.

---

## 16. Operations Centre — CCTV dashboard, monochrome UI, pacing (`v24`, this branch)

**UI overhaul (no mechanics touched).** The default look is now a hotel-operations-centre light theme: paper-white surfaces, black typography, thin 1px borders, flat 2–3px corners, no shadows or gradients, large whitespace. Accent colours only carry meaning — forest green (live/good/revenue-up), burgundy (critical/loss), orange (warning), dark gold (brand/awards), deep grey (info); everything else is monochrome. Dark mode remains available in Settings as a flat charcoal "security room". Existing saves migrate to the new light default once; an explicit re-pick of dark then sticks.

**CCTV — a purchasable security system.** The Cameras dashboard starts locked ("Install CCTV System to unlock live monitoring"). The core package (lobby, all floor corridors, entrance, reception — capex-scaled, and it raises the existing `p.sec.cctv` tier so guest-confidence effects apply) unlocks the dashboard; each built facility (restaurant, breakfast hall, pool, gym, spa, laundry, travel desk, parking, conference, banquet, lounge) can then be wired individually. Unbuilt facilities never appear.

**The dashboard.** One permanent primary feed — Ground Floor Lobby — plus three switchable secondary panels (floor corridors + purchased facility cams). Feeds are hourly CCTV frames, not video: each in-game hour every visible camera regenerates a monochrome silhouette scene from live simulation state — occupancy, arrivals due this hour, checkouts (real `depDay`/`depHour`), staff on duty per role from the shift roster, dirty-room counts, maintenance backlog locations, guest archetype mix (gym/spa/lounge/conf draw on business/premium/VIP in-house counts), meal periods, weekday patterns, weather (rain empties the pool, fills the lobby) and night hours (security patrols corridors and parking). A seeded RNG keyed on day+hour+camera makes each frame stable within its hour. No floating guest labels — each feed carries only a small info strip (e.g. Lobby: guests visible / check-ins active / check-outs active / waiting). One or two silhouettes per feed drift slowly via CSS animation (disabled by reduce-motion) for a live feel without a render loop. Operations gains a CCTV Control Room card mirroring purchase state.

**Pacing.** Base tick retimed from 3s to 2s per in-game hour at 1×, with a new 8× control (2×=1s, 4×=0.5s, 8×=0.25s, keyboard `8`, speed-cycle updated). The hour remains the atomic simulation step — speed only changes render cadence, so nothing skips or desynchronises; verified by full sim runs and the deposits-ledger invariant at 8×.

**Verified.** 3-country sweep (29 tabs + facility pages + modals): zero issues, no page errors; camera flow end-to-end (locked state → core purchase at $840K/₹3L → feeds live → facility cams → slot switching → hourly frame regeneration); theme migration both directions; mobile layout (feeds stack, KPI strip visible); smoke4/smoke5 economy and Phase-11 regressions green.

---

## 17. Silhouette Minimal — UI pass + logic-fix pass (`v25`, this branch)

**Visual (toward the finalised reference).** Black-pill sidebar navigation, ▲/▼ vs-yesterday deltas on the topbar Occupancy/ADR chips, and the Cameras page rebuilt as the operational centerpiece: large pinned Ground-Floor Lobby feed with floor-corridor quick chips, three switchable feeds stacked to the right, and a live-operations summary row beneath (department status board, activity timeline, today's arrivals, alerts). Lobby/restaurant/lounge scenes gained a glass window-wall with soft skyline and pendant lights; daytime lobby density raised.

**Logic fixes:**
1. **Internet billing** — `utilCompute` charged `net` unconditionally; now billed only when Wi-Fi is built (`has('wifi')`), else 0.
2. **Save/load** — `load()` never revealed the app: it parsed the save but left `#setup` visible, `#app` hidden and the game loop stopped, so "Continue Saved Game" landed on a dead screen. It now hides the landing/wizard, shows the app, and starts the loop; verified exact-state restore across a hard page reload (day, hour, cash, rooms, staff, backlog, facilities, contract all byte-equal).
3. **Front-desk gate** — a hired manager silently covered an unmanned desk 24/7 (`od===0 && !mgr`), so check-ins always proceeded. The gate is now strict: nobody rostered → guests wait in the lobby (walkout after 3 hours). Housekeeping/maintenance/F&B/security gates already keyed off `dutyList`/`onDuty` and were verified.
4. **Auto-roster F&B/security** — `rNeed('fnb')` only recognised `restaurant`; breakfast/lounge/banquet/room-service hotels never rostered F&B. Need now follows every built F&B facility with its operating hours; security threshold lowered (nights from 6 rooms; day cover for 24+ rooms or banquet properties). Building a facility or hiring staff now tops up an empty roster for that role automatically (never overwrites an existing plan).
5. **Staffing caps** — daily cap corrected from 12h to **8h** (`R_MAXDAY`); weekly hard cap stays **56h**; contracted-hours input widened to 40–56 (default 48) so overtime (1.5×, beyond contract) works as designed instead of feeling like a 46h ceiling.
6. **Maintenance** — the daily resolver computed capacity from whoever was on duty *at midnight* (when it runs) — i.e. nobody — so issues never cleared. Capacity now comes from the day's rostered maintenance hours × skill; unrostered days resolve nothing (gate). Issue spawn rates cut to ~40% of before (occupied .05→.02, idle .015→.006, wear .04→.016, common-area .08→.03).
7. **Pricing page** — three separated concerns: "Rate Control — you set the rack rates" (inputs disabled with an explanatory notice while the RMS runs), "Rate Tactics — fences & discounts you control", "Revenue Management — who sets prices". Market ref/position/demand-pull remain read-only analytics.
8. **Guest Mix Insight card removed** from Front Office (not replaced).
9. **Policy defaults** (new properties): check-in 13:00, late-checkout fee 500, early check-in ON, late checkout ON, payment terms = no deposit. Existing saves keep their chosen values.
10. **Events demand an answer** — the shared modal's leftover backdrop-dismiss handler let events be clicked away; event modals now disarm it, so one of the offered actions must be chosen (game pauses while open, as before).

**Verified.** Dedicated fix-pass suite (all 20 assertions green), save→reload→load exact-match, 3-country/29-tab sweep zero issues, smoke4 economy invariants (deposit ledger consistent under the new no-deposit default) and smoke5 Phase-11 suite green.

---

## 18. CCTV Art II + finance verification pass (`v26`, this branch)

**Camera visuals rebuilt to the reference.** New articulated silhouette figure library (walking, luggage-pulling, conversing pairs, seated diners with chairs, staff at reception laptops, waiter with tray, housekeeper pushing a towel cart, vacuuming, pool attendant with skimmer net, swimmers) with glossy-floor reflections and contact shadows. Set dressing per scene: glazed window walls with layered city skylines and palms, pendant lamps, monstera planters, a proper reception counter with laptops, dining tables with seated pairs, umbrellas/loungers/rippled pool water, corridor doors with housekeeping carts, basement parking with pillars, bay markings and car silhouettes. Figures are placed statically per hourly frame (seeded by day+hour+camera) with one slow drifting walker per feed — a CCTV still, not a video. All people counts remain driven by the same live simulation state.

**Finance & numbers audit (dedicated instrumented probes).** Verified exact: cash continuity (Δcash ≡ Σ(rev−exp) + Δdeposit-liability + intraday residuals — drift 0 over 10 days); per-day analytics identity (profit = rev − exp, every row); finHist ↔ analytics agreement; Today P&L chip vs internal counters; Net/day chip; loan mechanics (annuity EMI formula, principal amortization, 1% processing fee disclosed and expensed, cash credited net of fee); investor raises correctly gated by `invUnlocked`; deposits ledger equals outstanding booked deposits plus in-house guests' deposits (they release at checkout by design); utilities total = component sum; department P&L finite; money tabs free of NaN/Infinity. A source-grouped mutation trace (every write to cash/revToday/expToday/depositsHeld attributed to its code line) confirmed every flow pairs correctly.

**One real defect found and fixed:** guest walkouts (`checkedIn='lost'` — unmanned desk or no clean room) kept the guest's deposit in the `depositsHeld` liability forever, never refunding or releasing it. All three walkout sites now refund the deposit in full (the failure is the hotel's) and release the liability.

**Verified:** 3-country / 29-tab sweep zero issues; smoke4 economy invariants; smoke5 Phase-11 suite; v25 20-assertion logic suite; save→hard-reload→load exact match.

---

## 19. Streamlining pass — spec items 5–9 + construction (`v27`, this branch)

Focused on "reduced unnecessary complexity / smarter automation" half of the Phase-Next spec. No simulation depth removed.

5. **Random Events/Situations removed.** `triggerEvent` is now a no-op; no pop-up situation cards, timers or forced interruptions. Operational pressure surfaces organically through the existing Attention Center, staffing gates, maintenance backlog, reviews, utilities and finance — the game is simulation-driven, not event-driven. `showEvent`/`autoResolveEvent` remain defined (harmless) but are never invoked.

6. **Pricing overhaul — one clean control.** The page had three overlapping pricing systems (v13 per-room positioning, v17 global 3-mode/5-strategy, v12 tactics). Consolidated to the spec model: a single "Revenue Management — who sets prices" card with two modes — **Player Managed** (you set rack rates) and **Manager Managed** — and, under Manager, exactly five positioning options (20% Below / 10% Below / Match / 10% Above / 20% Above Market). One authoritative daily repricer anchors each rate to the live market reference: `rate = marketRate × (1 + position%) × (1 + organic-nudge%)`, where the bounded ±12% nudge reads demand, on-the-books pace, festival windows, competitor index, rating and reputation — every change logged and explained. Removed the duplicate per-room positioning card and the redundant "Yield autopilot" toggle. Booking fences (early-bird, corporate rate, weekend min-stay, day-of-week, last-minute yield, OTA parity) kept as a distinct "Rate Tactics" card. Legacy 3-mode/5-strategy state migrates automatically (manager/ai → auto; strategy → nearest positioning).

7. **Construction costs −20%.** Scaled at the cost source: `_BASE.build` (rooms), `_BASE.qcost` (quality fit-out), `_BASE.fbuild` (all facilities/departments) and `_BASE.comp` (components) × 0.8, re-derived through `applyCountry` so every country stays consistent; floor construction (2.5M+0.8M/floor → 2.0M+0.64M) and parking bays (45k → 36k/space) edited directly. Daily operating costs and consumable inventory left unchanged (not construction).

8. **Contract hours removed.** One standard schedule everywhere — 56 h/week, 8 h/day. Payroll baseline is `salary / (4.33 × 56)`; overtime only accrues past 56 h/week (never, given the hard cap), so OT is effectively retired while the 15% night premium stays. The contracted-hours input, `p.contract` dependence and OT-below-56 logic are gone from the roster UI and payroll math.

9. **Manager-controlled rostering.** Per-hotel toggle on the Roster page: **Player-managed** (edit shifts by hand) or **Manager-managed** (enabled once a General Manager is on staff). Under Manager-managed, the GM auto-fills every shift to need each day at rollover — front office, housekeeping, maintenance and F&B coverage maintained, staffing scaled to occupancy, overtime minimised — and the player can switch back to manual anytime.

**Verified.** Dedicated v27 suite (16/16 real checks): events never fire over 20 days; payroll OT = 0 under standard load; roster contract control gone + standard-schedule wording; management toggle present and delegating auto-refills a cleared roster at rollover; pricing shows two modes + five options with no legacy strategy buttons; Match-Market snaps to reference, +20% lifts above; construction −20% confirmed against pre-v27 effective bases. Full regression green — 3-country/29-tab sweep (0 issues), smoke4 economy invariants, smoke5 Phase-11 suite. (A tiny pre-existing rounding drift in the non-default 'partial'-deposit ledger is present in the committed build too — not a v27 regression; the default policy is 'none'.)

*Deferred to the next build (per scope split): strategic modules — Corporate Sales workflow (#1), Supplier Marketplace (#2), expanded HQ (#3), long-term progression (#4) — and the deep code-hygiene pass (#10).*
