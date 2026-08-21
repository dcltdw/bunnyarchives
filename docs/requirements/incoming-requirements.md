# Bunnyarchives — Requirements (Running Doc)

Architecture is out of scope. This document tracks discovered requirements as they're gathered; it should grow as more surfaces.

---

## 1. Character Creation Wizard

- Three tiers: minimal (next field only), somewhat helpful (brief reminders), very helpful (quotes rulebook sections).
- **Someone will need to manually annotate the rulebook** so the wizard can quote from it — the wizard must never LLM-summarize the rulebook itself.
- Unanswered questions route through a query mechanism (email/Discord/other/combination) to humans, **and bunnyarchives tracks the state of that query** (open/answered/ignored), not just fire-and-forget.

## 2. Events

- Held ~4x/year by default — **frequency is configurable**, default is 4. "Saturday only" is **descriptive of current practice only** — not a system-enforced constraint.
- Seam for multi-day events (3-day, 5-day, etc.) exists from the start, even though events are 1-day for now.
- **Multi-day events affect the entire event model**, not just scheduling — including:
  - Check-in can happen at any time during the event (e.g., a player may check in on Day 2).

## 3. Registration & Payment

- RSVP + payment; payment may be detached from RSVP (e.g., season passes).
- **Season pass duration is configurable** (not hardcoded to "next 4 events").
- **Missed-event credit**: manual by default (staff-applied per player). Automated/policy-driven credit is a **later-phase feature**.
- **Staff discounts**: always free (100% comp), no partial-discount case. Audit trail = the registration record itself.
- **Community fund**:
  - Single ongoing pool (not reset per event).
  - Balance visibility policy can only be set by Directors.
  - Donor identity is visible only to a dedicated staff tag: **Community Fund Managers**. This tag exists solely for this purpose.
- **Paying for another player's registration**:
  - Beneficiary always receives confirmation.
  - If a specific person paid, they also receive confirmation.
  - If paid via community fund, confirmation instead routes to Community Fund Managers.
- **Registration transfer to a future event**:
  - Default: no extra charge regardless of price difference.
  - Overridable by policy.
  - Some special events may disallow transfers entirely, or require additional payment to transfer in.

## 4. Documents (Player-Facing)

- Issued at check-in (letters, rumors, other materials); retrievable by players at will after the fact.
- **Documents aren't limited to check-in**: additional documents may be issued over the course of an event, and optionally between events as well.
- **For all documents, regardless of when/how issued, retrieval by the player should be easy** — a single consistent place/mechanism, not split by delivery timing.
- **Documents are permanent once issued — cannot be revoked.**
- However, a new document **may supersede** an old one; system must handle this gracefully (i.e., versioning/supersession, not deletion).

## 5. Player Profiles

- Fields: name, email, emergency contact, dietary restrictions, food allergies, notable non-food allergies. Phone optional. **No address field, ever** (no physical mail use case in the system).
- **Emergency contact is structured** (not free text) — fields TBD (likely name/phone/relationship).
- **Non-food allergies**: scope = anything that could induce a reaction requiring an inhaler or, especially, an epi-pen. **Epi-pen-level allergies must be flagged/highlighted distinctly as true dangers**, easy for staff to spot quickly.

## 6. Characters

- Players start with **zero characters** (base case) — zero Active is valid at any time.
- Skill cap: **starting value is configurable** (default 2, global), can be raised over time; individual characters can have a different cap than the global default.
- **Global skill cap increases apply retroactively** to existing characters.
- **Speculative character cap is policy-driven**: default 3 for players, default 10 for staff (staff create NPCs).
- **Skill list itself is defined in the rulebook** (not a bunnyarchives-managed catalog, at least initially).
- NPCs share the same Active/Retired/Speculative status model as PC characters.

## 7. Player Visibility (Other Players List)

- **Security-motivated**: unauthenticated users see nothing. Even among players, the list is gated to players Staff has marked as having *attended* (not merely registered) at least one event.
- **No list at all is shown to players who haven't attended an event** — explicitly to prevent a malicious actor from registering just to check whether a specific person (e.g., a stalking target) is registered.
- Default visible fields: first name, last name initial, Active character name.
- Players can opt to hide name, character, or both.
- Hidden entries are omitted from the list, but a **discreet summary line** appears at the bottom (e.g., "+2 more characters with hidden info").
- Players can opt in to publish their email on this list.

### 7a. Related Staff Controls (surfaced by the above)
- Ban a player.
- Disallow registration from a specified IP list.
- Global on/off switch for New Player Registration (default **Off** until the site is ready).
- When registration is On, a policy toggle for whether new registrations require manual approval (to catch banned/malicious actors re-registering under a new email).

## 8. Payment Data Handling

- Payment info (including address) is **never stored** — used transiently at time of purchase, then immediately deleted. In-memory retention window kept to the strict minimum.

## 9. Check-In

- Staff verifies payment and marks attendance at event start; system separately supports marking/unmarking attendance at any time.
- **Any check-in mistake must be reversible.**
- Check-in "what to prepare" items:
  - Can live on the **player object** (e.g., "have $15 refund check ready for John") or the **character object** (e.g., "this character always receives 2 Rumors").
  - Recurring character items are **dictated by the character sheet**, not re-confirmed each event.
  - **Report**: "what items need to be ready at check-in?" — aggregates from both player and character objects so Staff can prep.
  - **Report**: for skills requiring prep, shows how many and for which characters.

## 10. Post-Event Letter (PEL) System

- Template authored by Staff; PEL becomes available only to players who *attended* the event (not merely registered).
- Made of multiple sections/questions; answers may feed into various collated reports.
- **Routing mechanism** to send a player's section answer to one or more staff members exists as a **seam now, implemented in a later phase**.
- Visibility tiers:
  - Not fixed at two levels — **fully configurable**. Any piece of PEL info can be gated to a **staff tag**.
  - Staff tags **may not be strictly hierarchical** (i.e., tags don't necessarily nest/inherit).
- Deadline: PELs are always accepted, but a deadline may exist as an **incentive** (not a hard cutoff) for on-time submission.
- Staff have their own PEL variant — different template, same underlying process.

## 11. Staff & Roles

- **Directors** are effectively root-level admins, but structurally are just one staff tag among many (not a hardcoded special case).
- Access model (see also §15): granular permissions grouped into **roles**; Directors assign people to roles rather than toggling individual permissions. **DENY overrides always win** regardless of which assigned role grants access.
- Staff can register for events at no cost; can indicate availability for early setup, late strike, or partial-day-only presence — **both self-reported and staff-lead-assignable** (both mechanisms exist).
- At-the-door registration: **no offline support** — payment can't reasonably be queued offline, so this flow assumes connectivity.

## 12. Staff-Only Character Notes

- Section headings (e.g., "for next event," "archive," "to do," "potential ideas," "lines & veils," "plots") are **configurable via global policy**, not by individual staff members.
- **Edit conflict handling**: **first-one-in wins**. Any subsequent update against a now-stale version is **refused**, and the refused edit is preserved/saved as a draft so the user can manually merge. This is the same model used for offline sync conflicts (§16) — one consistent rule across the system.

## 13. PC Status Changes

- Staff can move Active → Retired. Players can also move Active → Retired themselves. **This overlap is intentional.**
- If Staff retires a player's character:
  - The player **cannot reverse it themselves**.
  - The player **must be notified**.
  - Staff **can reverse it** (general "mistakes are reversible" principle applies).
- Only players can move Speculative → Active.

## 14. Plots, Encounters, Modules

- **Plots**: long-running threads spanning multiple events.
- **Encounters**: a single occurrence within a plot, scoped to **exactly one event** — the same underlying plot thread can produce a separate encounter (with its own priority) at each event. E.g., a P3 encounter at Event 1 generates strong player interest, so the "next chapter" encounter in the same plot is scored P1 at Event 2.
  - Priority: P1 (must run) / P2 (should run) / P3 (nice if it runs) / P4 (fine if it doesn't).
  - Scheduled via specific time or general label ("morning," "afternoon," "anytime").
  - Status during event: **Running**, **Delayed** (rescheduled within the *same* event), or **Deferred** (not run this event; intended for a future event, likely but not necessarily the next one).
  - Reports: "encounters that ran," "...that ran delayed," "...that were deferred."
- **Modules**: run in designated named spaces; may need multiple staff and one or more briefing handouts; have expected run time, setup/strike assignments and timing.
  - Can be **recurring within a single event** (multiple showings same day) — not recurring across events.
- **Comments** (encounters & modules):
  - **Append-only audit trail** is the default mechanism.
  - A separate **"working knowledge" section is freely editable** (not audit-trailed) for living/collaborative notes.
- Both encounters and modules have a short summary field for use in reports.

## 15. Access Control (Cross-Cutting)

- Every piece of information/action should support **granular, role-based access**.
- Directors assign staff to **roles**; roles bundle many individual permissions (rather than staff flipping many individual switches).
- Roles/permissions may overlap; a **DENY at any level always wins** over any grant.
- Staff tags (see §10, §11) are the building block used across visibility policy, PEL section gating, and report access — **not strictly hierarchical**.

## 16. Reports (Staff)

- Pre-event: registered list; registered-and-unpaid list; door-payment-planned list; skill-count-at-event-by-active-character report. All report visibility is policy/tag-gated.
- Encounter reports: all encounters for the event (one per page, **assume PDF output**, multi-page encounters supported); all summaries; summaries filtered by assigned staff member.
- Live-event dashboard: current time, what's running, what's on-deck, who's running what, who's on-deck for what.
  - **On-deck window** (e.g., 30 min vs 2 hr lead time) is a **staff-set policy** (possibly gated to specific tag(s)), with per-encounter override.
  - **Encounters and modules have separate default on-deck window values.**
  - Schedule phases (setup, lunch, dinner, strike) configurable by policy.
- Live queries: "my assigned encounters," "current schedule," "my scheduled encounters," "all P#-remaining encounters," "my assigned encounters with no comments yet."
  - **These queries should also be available before the event** (not just live during it) — e.g., a staff member checking "what am I assigned to?" while still prepping.
- **Gantt-like scheduling chart**: visualizes the schedule of all encounters, optionally filterable by priority, by staff member, by both, and ideally by a **group of cast members** (not just individuals).
- **Overscheduling / double-booking detection**: e.g., John is scheduled 10–11, Becky is scheduled 11–12, and John+Becky are scheduled together 10:30–11:30 — all three overlapping entries should be surfaced together as the conflicting set, not just the pair that directly overlaps.
- Post-event: PEL reading, gated per the configurable tier policy (§10).

## 17. Cross-Event Plot Continuity

- Staff working on **any** event of a plot (Event 3 was just an illustrative example, not special-cased) should easily see **all prior events'** encounters for that plot + associated player/staff PEL responses, to inform planning.

## 18. LLM Assistance (Staff-Facing)

- Optional LLM tool to read PELs and suggest which sections should route to which plots.
- **Never auto-applies** — each individual suggestion requires explicit staff approval.
- Approved suggestions are tagged in metadata as applied "via AI agent" (mirrors the existing metadata convention for staff-moved PEL info).

## 19. Offline Support

- "Download all info for upcoming event" — includes **full historical plot archive**, not just the current/relevant threads, since staff may be asked about anything at any time.
- Delayed sync of offline-entered data on reconnect.
- **Offline sync conflict model**: first-one-in wins; conflicting followups are refused and saved as drafts for manual merge — same unified model as live-edit conflicts (§12).
- Later phase: ad hoc peer-to-peer laptop network for on-site sync without internet.

## 20. Items Catalog

- A catalog of items (props/game items) exists as a first-class concept.
- **Item IDs are unique and player-friendly** — human-readable, not opaque, e.g. a Tier 2 Rogue item called "Hot Tip" → `Rogue2HotTip`.
- **Printing support**:
  - One-off printing of a single item.
  - Batch printing driven by an input file — format may be a spreadsheet, `.md`, or `.txt` (multiple input formats to support, not just one).
  - **Layout varies by item**: items may print at different sizes/densities per sheet — e.g., some items 2-across × 5-up on 8.5x11, others 1-across × 6-up. Layout is a per-item (or per-item-type) property, not a single fixed template.
- **Item collections/quantities are reusable as a building block** across multiple other parts of the system — a named/specified group of items + counts can be attached wherever needed. **An item can belong to multiple collections simultaneously** (not exclusive to one). Examples given:
  - What a player/character should receive at check-in.
  - What's required for a given encounter or module.
- **Metadata**: each item has an associated **metadata dict field**, starting **empty** by default (no required/predefined keys) — open-ended, to be populated as needs emerge.

---

## Open Items / Explicitly Deferred to Later Phases
- Automated missed-event credit logic.
- PEL section → staff-member routing implementation (seam exists now, feature later).
- Wiki as part of bunnyarchives (plot content abstraction layer exists now; native wiki is later).
- Ad hoc peer-to-peer offline sync network.

