<!-- CHECKPOINT — disaster-recovery snapshot, 20260822T042236
     Session state as of the deletion/root elicitation pass.
     Intended to be deleted once merged back into the running doc. -->

# Bunnyarchives — Requirements (Running Doc)

Architecture is out of scope. This document tracks discovered requirements as they're gathered; it should grow as more surfaces.

---

## Cross-Cutting Principles

- **One deployment serves one campaign.** Bunnyarchives is not multi-tenant; another campaign adopting it would run its own deployment.
- **Mistakes are reversible.** Every operation should be reversible by an appropriately-permissioned person. Named exceptions:
  - Under first-one-in-wins conflict handling, a refused stale edit is never applied — though it is preserved as a draft (§12, §18).
  - **PII is hard-deleted** on account deletion and does not come back (§5). Everything else is soft-deleted and restorable.
  - **The root action log is append-only** — not modifiable or deletable, including by root (§15).
  - **Ban records persist**, including their retained PII copy, even after the source account is deleted. Un-banning is reversible; the ban record is not erased (§5).
- **Notifications go to email.** Keep an abstraction seam so another service could be swapped in — it's an easy barrier to create — but no other implementation is expected.
- **Players and staff are both people.** One person can be both (staff register for events, run NPCs, may play PCs). Player and staff are facets of a common "Person" concept.

## 1. Character Creation Wizard

- Three tiers: minimal (next field only), somewhat helpful (brief reminders), very helpful (quotes rulebook sections).
- **The player chooses the tier, and can switch tiers mid-flow.**
- **Someone will need to manually annotate the rulebook** so the wizard can quote from it — the wizard must never LLM-summarize the rulebook itself.
- **The system knows which version of the rulebook it references** — annotations are tied to a rulebook version.
- Unanswered questions route through a query mechanism (email/Discord/other/combination) to humans, **and bunnyarchives tracks the state of that query** (open/answered/ignored), not just fire-and-forget.
  - Answers may arrive through any mechanism; the tracking ticket is updated to match. **The player can mark a ticket DONE; all other status changes are Staff-only.**

## 2. Events

- Held ~4x/year by default — **frequency is configurable**, default is 4. Events are currently single-day Saturdays in practice, but **the system must not enforce any day-of-week constraint**.
- Seam for multi-day events (3-day, 5-day, etc.) exists from the start, even though events are 1-day for now.
- **Multi-day events affect the entire event model**, not just scheduling — including:
  - Check-in can happen at any time during the event (e.g., a player may check in on Day 2).
  - **Attending any part of a multi-day event counts as having attended it** (relevant to §7, §10).
  - Registering/paying for only part of a multi-day event is **not** needed functionality.
- **Events have a capacity** — waitlist mechanics in §3.

## 3. Registration & Payment

- RSVP + payment; payment may be detached from RSVP (e.g., season passes).
- **Capacity & waitlist**: when a spot frees up, the system waits a short delay (in case the opening was a mistake), then notifies the first person on the waitlist via email and moves them from waitlist to registered-but-unpaid. How registration status is captured is a future design question.
- **Season pass duration is configurable**: either a number of events or a fixed end date.
- **Missed-event credit**: manual by default (staff-applied per player). Automated/policy-driven credit is a **later-phase feature**.
- **Refunds are policy-driven and entirely manual** — no automated refund flow.
- **Event cancellation**: if an entire event is canceled, staff need an easy way to record each player's choice, typically one of: full refund; partial or full donation of the money to the LARP (no refund, no future credit); credit transferred to the next event. (Illustrative, not locked: one admin screen with tickboxes covering all players in one go.)
- **Staff discounts**: always free (100% comp), no partial-discount case. Free registration is denoted by a dedicated staff tag (e.g., `StaffFreeReg`). Audit trail = the registration record itself.
- **Community fund**:
  - Single ongoing pool (not reset per event).
  - Donations accepted from anyone: a **dedicated donation page** exists, and regular registration checkout offers a **donate option**.
  - **Policy dictates whether Community Fund Managers must approve each use of funds, or players may self-select** fund use.
  - Balance visibility policy can only be set by Directors.
  - Donor identity **and donation amounts** are visible only to a dedicated staff tag: **Community Fund Managers**. This tag exists solely for this purpose.
- **Paying for another player's registration**:
  - Beneficiary always receives confirmation.
  - If a specific person paid, they also receive confirmation.
  - If paid via community fund, confirmation instead routes to Community Fund Managers.
- **Registration transfer to a future event**:
  - Default: no extra charge regardless of price difference.
  - Overridable by policy.
  - Some special events may disallow transfers entirely, or require additional payment to transfer in.

## 4. Documents (Player-Facing)

- **Documents are communal artifacts owned by Staff.** They are *issued to characters*, not owned by the receiving player.
- **One document may be issued to many characters**, simultaneously or at different times. Model this as **Document** (the artifact) and **Issuance** (character X received document Y, on date Z).
- **Content is identical across all issuances.** No per-recipient variation; no templating.
- **Issuance targets a Character, not a Person.** A player whose character retires no longer has access to that character's documents through a new character. Retrieval is character-scoped.
- **Recipients are not visible to each other.** A player cannot learn who else received the same document.
- Issued at check-in (letters, rumors, other materials); retrievable by players at will after the fact.
- **Documents aren't limited to check-in**: additional documents may be issued over the course of an event, and optionally between events as well.
- **For all documents, regardless of when/how issued, retrieval by the player should be easy** — a single consistent place/mechanism, not split by delivery timing.
- **Documents are permanent once issued — cannot be revoked, *except* by explicit staff deletion** (see below). There is no player-facing revocation and no expiry.
- **Deletion (§5 soft-delete model) applies at two levels, with distinct meanings, both gated by a document-deletion tag:**
  - **Delete an Issuance** — fixes a mis-issuance to one character. That character loses access; other recipients are unaffected.
  - **Delete the Document** — removes it from every character who received it, in one action.
  - **Hiding it from the player is the intended effect.** This is deliberate and is the one sanctioned form of revocation.
- However, a new document **may supersede** an old one; system must handle this gracefully (i.e., versioning/supersession, not deletion).
  - **Supersession obscures previous versions from players.** Staff can still reach them, but not casually — e.g., a confirmation dialog, a SUPERSEDED watermark, or other high-visibility markings.

## 5. Player Profiles

- Fields: name, email, emergency contact, dietary restrictions, food allergies, notable non-food allergies. Phone optional. **No address field, ever** (no physical mail use case in the system).
- **Emergency contact is structured**: name, relation, phone, email.
- **Non-food allergies**: scope = anything that could induce a reaction requiring an inhaler or, especially, an epi-pen. **Epi-pen-level allergies must be flagged/highlighted distinctly as true dangers**, easy for staff to spot quickly. **Epi-pen severity is self-reported by the player.**
- **Medical data (allergies, emergency contact) is tag-gated** like everything else (§15). It does **not** generally appear in reports, but any report may opt to include it. It **is** part of the offline "download all info" export (§18).
### Deletion Model (resolved)

- **Deletion is a "marked for deletion" bit, not removal.** It is fundamentally a **filter**, not a security control — deleted objects drop out of default views and out of reports.
- **Account deletion** is invokable by the player ("delete my account") or by staff. A **deleted account cannot be logged into.**
- **PII is hard-deleted. Everything else is soft-deleted.** Character sheets and PELs persist with the delete bit set.
  - 🚧 **BLOCKER: "PII" is not yet defined.** Deliberately parked — the boundary is influenced by too many other open factors and a cleaner line is expected to emerge later. **§5 cannot be finished until this is drawn.** Known so far: **name is NOT PII**; **attendance history, payment records, and waivers persist.**
- **Cascade:** deleting a parent marks its children deleted. **Undeleting a parent undeletes its children.** Each object also carries its own independent bit, so an object deleted separately before its parent stays deleted when the parent is restored.
- **Deleted characters do not occupy character-cap slots** (§6). Note the deliberate asymmetry with skill caps: **character caps gate creation only** ("can I make a new one?"), whereas **skill caps apply retroactively**.
- **Deleted PELs** only arise when a player submits a PEL and then deletes their account; whether a deleted PEL satisfies other criteria is not germane.
- **Visibility of deleted content** requires a **`can see deleted objects` tag** (§15). For holders, every relevant view offers an **"also include deleted content"** toggle.
  - **The toggle is not sticky** — it does not persist across views or sessions.
  - **Filters are orthogonal to rules/tags.** Root mode does *not* auto-reveal deleted content; a root user still toggles the filter.
- **Deleted content is excluded from the offline export** (§18).

### Bans

- **Bans survive account deletion.** The **Ban object holds its own copy of the PII** (**name and email**), so the source object's PII can be hard-deleted while the ban record persists.
- **The system flags a probable match at registration; it does not block.** Evasion by changing name and email is a **known and accepted risk**.
- **Ban PII is visible to anyone who can see bans** — no separate gating.
- **Bans are reversible, but the retained PII is not purged on un-ban.** The ban record functions as a **ban audit log**.
- **The player is told** that deletion will not remove their information, because they are banned.
- **Minors are allowed.** At registration, every player is checked for **valid waiver(s) on file; which waivers are required is configurable** — this covers the minor case.

## 6. Characters

- Players start with **zero characters** (base case) — zero Active is valid at any time.
- **Maximum of 1 Active character per player.**
- **Characters require staff approval** — likely as a status axis parallel to Speculative/Active/Retired: **Awaiting Approval / Approved**.
- Skill cap: **starting value is configurable** (default 2, global), can be raised over time; individual characters can have a different cap than the global default.
- **Global skill cap increases apply retroactively** to existing characters.
- **Adding skills is player self-service**; a system for **character advancement** exists.
- **Character self-service includes a Print option**: outputs player name, character name, event date, and character skills; more fields may be added by configuration.
- **Speculative character cap is policy-driven**: default 3 for players, default 10 for staff (staff create NPCs).
- **Skill list itself is defined in the rulebook** (not a bunnyarchives-managed catalog, at least initially).
- NPCs share the same Active/Retired/Speculative status model as PC characters.
- **There is no separate Dead state — Retired covers character death.**

## 7. Player Visibility (Other Players List)

- **Security-motivated**: unauthenticated users see nothing. Even among players, the list is gated to players Staff has marked as having *attended* (not merely registered) at least one event.
- **No list at all is shown to players who haven't attended an event** — explicitly to prevent a malicious actor from registering just to check whether a specific person (e.g., a stalking target) is registered.
- Default visible fields: first name, last name initial, Active character name (unambiguous — max one Active character, §6).
- Players can opt to hide name, character, or both.
- Hidden entries are omitted from the list, but a **discreet summary line** appears at the bottom (e.g., "+2 more players with hidden info").
- Players can opt in to publish their email on this list.

### 7a. Related Staff Controls (surfaced by the above)
- Ban a player — **blocks both login and registration**.
- Disallow registration from a specified IP list.
- Global on/off switch for New Player Registration (default **Off** until the site is ready).
- When registration is On, a policy toggle for whether new registrations require manual approval (to catch banned/malicious actors re-registering under a new email). **Approval is performed by a dedicated tag** (name TBD; tags are self-naming, e.g. `RegistrationApprover` — see §15).

## 8. Payment Data Handling

- Payment info (including address) is **never stored** — used transiently at time of purchase, then immediately deleted. In-memory retention window kept to the strict minimum.

## 9. Check-In

- Staff verifies payment and marks attendance at event start; system separately supports marking/unmarking attendance at any time.
- **Any check-in mistake must be reversible.**
- **Check-in must work offline** (§18).
- Check-in "what to prepare" items:
  - Can live on the **player object** (e.g., "have $15 refund check ready for John") or the **character object** (e.g., "this character always receives 2 Rumors").
  - **Prepare-items are typically not free text** (free-text materials are handouts, i.e. documents §4) — they are a **list of items and desired quantities** (quantities can vary wildly), i.e. the item-collection building block (§19).
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
- Deadline: PELs are always accepted, but a deadline may exist as an **incentive** (not a hard cutoff) for on-time submission. **The incentive itself is staff-set and out of system scope.**
- **PELs cannot be edited once submitted** — but a new submission within the incentive deadline is allowed and **clobbers** the previous one.
- Staff have their own PEL variant — different template, same underlying process.
- **A person who both played a PC and NPC'd at the same event** (expected to be very rare) submits **one PEL of each kind** — the player-facing one and the staff-facing one.

## 11. Staff & Roles

- **Directors** are effectively root-level admins, but structurally are just one staff tag among many (not a hardcoded special case). See §15 for the root tag / sudo distinction.
- Access model (see §15): granular permissions attach to **tags**; Directors assign people to tags. Effective permission is the **AND of all relevant tag permissions — a DENY anywhere wins**.
- Staff can register for events at no cost (`StaffFreeReg` tag, §3); can indicate availability for early setup, late strike, or partial-day-only presence — **both self-reported and staff-lead-assignable** (both mechanisms exist).
  - Availability conflicts between the two mechanisms resolve as **last-in wins** — deliberately unlike the first-one-in-wins edit model (§12). The use case is: "Hey, John — can you mark me as available for that? Thanks!"
- **Staff member status** during an event: default value set is **NOT ARRIVED, OFF SITE, ON SITE, IN AN ENCOUNTER, IN A MODULE, RESTING, SLEEPING, HAS LEFT** — the set is configurable (more/fewer states). Surfaced on a staff-status dashboard (§16) that also works offline (§18).
- At-the-door registration: **no offline support** — payment can't reasonably be queued offline, so this flow assumes connectivity.

## 12. Staff-Only Character Notes

- Section headings (e.g., "for next event," "archive," "to do," "potential ideas," "lines & veils," "plots") are **configurable via global policy**, not by individual staff members.
- **Edit conflict handling**: **first-one-in wins**. Any subsequent update against a now-stale version is **refused**, and the refused edit is preserved/saved as a draft so the user can manually merge. This is the same model used for offline sync conflicts (§18) — one consistent rule across the system.

## 13. PC Status Changes

- Staff can move Active → Retired. Players can also move Active → Retired themselves. **This overlap is intentional.**
- If Staff retires a player's character:
  - The player **cannot reverse it themselves**.
  - The player **must be notified**.
  - Staff **can reverse it** (general "mistakes are reversible" principle applies).
- **Retired → Active is possible but expected to be rare** — its most common use is undoing a mistaken retirement. (Staff-retired characters remain staff-only to reverse, per above.)
- Only players can move Speculative → Active (subject to the approval axis, §6).

## 14. Plots, Encounters, Modules

- **Plots**: long-running threads spanning multiple events.
- **Encounters**: a single occurrence within a plot, scoped to **exactly one event** — the same underlying plot thread can produce a separate encounter (with its own priority) at each event. E.g., a P3 encounter at Event 1 generates strong player interest, so the "next chapter" encounter in the same plot is scored P1 at Event 2.
  - Priority: P1 (must run) / P2 (should run) / P3 (nice if it runs) / P4 (fine if it doesn't).
  - Scheduled via specific time or general label ("morning," "afternoon," "anytime").
  - Status during event: **Running**, **Delayed** (rescheduled within the *same* event), or **Deferred** (not run this event; intended for a future event, likely but not necessarily the next one).
  - **Deferred encounters seed the next event's encounter list**, under a section marked "Deferred from last event."
  - Reports: "encounters that ran," "...that ran delayed," "...that were deferred."
- **Modules**: run in designated named spaces; may need multiple staff and one or more briefing handouts; have expected run time, setup/strike assignments and timing.
  - Can be **recurring within a single event** (multiple showings same day) — not recurring across events.
- **Comments** (encounters & modules):
  - **Append-only audit trail** is the default mechanism.
  - A separate **"working knowledge" section is freely editable** (not audit-trailed) for living/collaborative notes.
- Both encounters and modules have a short summary field for use in reports.

## 15. Access Control (Cross-Cutting)

- Every piece of information/action supports **granular access**; ultimately **every element of an object can carry its own CRUD permissions**, differing from its peers and from its parent.
- **"Parent" is a valid security setting**: an element may declare that whoever can perform a CRUD operation on its parent can perform it on the element too.
- **Tags are the access primitive.** Tags are deliberately low-level, purpose-scoped groups (Community Fund Managers exists solely for the community fund). Tags are **self-naming** (e.g., `RegistrationApprover`, `StaffFreeReg`) and **may not be strictly hierarchical** (no implied nesting/inheritance).
- **Roles are dropped for now.** They may return as groupings of tags — possibly orthogonal groupings — once their purpose is clearer (see deferred list).
- **Effective permission is the AND (intersection) of all relevant tags' permissions — a DENY anywhere results in DENY.** Consequence: holding an R-only tag alongside a CRUD tag leaves the person with R only; the R tag's missing grants deny C/U/D.
  - Guardrail for that misconfiguration: **mutually exclusive tag groups** — a person may hold at most one tag from such a group.
- **Root tag**: exactly one tag has CRUD on everything. This is distinct from what Directors hold: Directors (the highest-ranking people) might carry, e.g., universal CR, and must deliberately **switch personas into the root tag, sudo-style**, for anything beyond that.

### Root Mode (resolved)

- **Root bypasses permission evaluation entirely** — it does not participate in the AND-intersection, so a DENY elsewhere cannot override it.
- **Root does not bypass filters.** Deleted-content visibility remains a toggle even in root mode (§5).
- **Explicit mode switch with visible UI change**, so the holder always knows which persona is active.
- **Idle timer: root mode reverts to the user's normal permission set after one hour.**
- **All root actions are logged. The root action log is append-only — root can neither modify nor delete it.**
- **Reading the root log is a separate tag** from holding root — e.g., an oversight committee that can review all root actions without being able to take them.
- **A root-mode user can strip another user's root tag.** Mutual demotion is a **known and accepted risk**.

### Root Tag Count & Bootstrap Lock

- **The number of root-tag holders is configurable — both minimum and maximum.**
- **The system enforces a floor of 2 on the minimum. A configured minimum below 2 is ignored and 2 is used.** This floor is not overridable.
  - Example: `max 5, min 3` is honored as written. `max 5, min 1` is silently treated as `min 2`.
- **The system refuses to remove the root tag from the second-to-last holder**, and **blocks a root holder from deleting their own account.**
- **At install there are zero root holders.** Until the configured minimum is reached, a **Bootstrap Lock** is in effect:
  - A large banner appears on **every** page.
  - **Player registration is disabled.**
  - **No player-facing pages are available.**
  - Bootstrap Lock lifts once the configured minimum is met (e.g., at 3 holders under a `min 3` config).

## 16. Reports (Staff)

- Pre-event: registered list; registered-and-unpaid list; door-payment-planned list; **skill-count report**: for each skill, how many Active characters of registered players have it (e.g., of the 10 characters registered for the next event, how many have Deception?). All report visibility is policy/tag-gated.
- Encounter reports: all encounters for the event (one per page, **assume PDF output**, multi-page encounters supported); all summaries; summaries filtered by assigned staff member.
- Live-event dashboard: current time, what's running, what's on-deck, who's running what, who's on-deck for what. Plus the **staff-status dashboard** (§11).
  - **On-deck window** (e.g., 30 min vs 2 hr lead time) is a **staff-set policy** (possibly gated to specific tag(s)), with per-encounter override.
  - **Encounters and modules have separate default on-deck window values.**
  - Schedule phases (setup, lunch, dinner, strike) configurable by policy.
- Live queries: "my assigned encounters," "current schedule," "my scheduled encounters," "all P#-remaining encounters," "my assigned encounters with no comments yet."
  - **These queries should also be available before the event** (not just live during it) — e.g., a staff member checking "what am I assigned to?" while still prepping.
- **Gantt-like scheduling chart**: visualizes the schedule of all encounters, optionally filterable by priority, by staff member, by both, and ideally by a **group of cast members** — a group is simply a **list of staff names**, not a tag.
- **Overscheduling / double-booking detection**: e.g., John is scheduled 10–11, Becky is scheduled 11–12, and John+Becky are scheduled together 10:30–11:30 — all three overlapping entries should be surfaced together as the conflicting set, not just the pair that directly overlaps.
  - **Space double-booking is flagged too** — two modules/encounters in the same named space at overlapping times — not just people.
- Post-event: PEL reading, gated per the configurable tier policy (§10).

## 17. Cross-Event Plot Continuity

- Staff working on **any** event of a plot should easily see **all prior events'** encounters for that plot + associated player/staff PEL responses, to inform planning.
- **Staff can move/route PEL info onto plots**; such moves carry applied-by metadata. (This is the manual mechanism the deferred LLM assistance would feed suggestions into.)

## 18. Offline Support

- "Download all info for upcoming event" — includes **full historical plot archive**, not just the current/relevant threads, since staff may be asked about anything at any time. The export includes tag-gated data such as medical info (§5).
- **Must work offline**: check-in (§9); reading/printing plots and encounters; looking up archive data; entering staff comments on encounters; the running/on-deck encounter dashboards; the staff-status dashboard (§11).
- Delayed sync of offline-entered data on reconnect.
- **Offline sync conflict model**: first-one-in wins; conflicting followups are refused and saved as drafts for manual merge — same unified model as live-edit conflicts (§12).
- Later phase: ad hoc peer-to-peer laptop network for on-site sync without internet.

## 19. Items Catalog

- A catalog of items (props/game items) exists as a first-class concept.
- **Item IDs are unique and player-friendly** — human-readable, not opaque, e.g. a Tier 2 Rogue item called "Hot Tip" → `Rogue2HotTip`.
- **Printing support**:
  - One-off printing of a single item.
  - Batch printing driven by an input file — format may be a spreadsheet, `.md`, or `.txt` (multiple input formats to support, not just one).
  - **Layout varies by item**: items may print at different sizes/densities per sheet — e.g., some items 2-across × 5-up on 8.5x11, others 1-across × 6-up. Layout is a per-item (or per-item-type) property, not a single fixed template.
- **Item collections/quantities are reusable as a building block** across multiple other parts of the system — a named/specified group of items + counts can be attached wherever needed. **An item can belong to multiple collections simultaneously** (not exclusive to one). Examples given:
  - What a player/character should receive at check-in (§9).
  - What's required for a given encounter or module.
- **Metadata**: each item has an associated **metadata dict field**, starting **empty** by default (no required/predefined keys) — open-ended, to be populated as needs emerge.

## 20. Disaster Recovery, Uptime & Cost *(parked — captured, not yet examined)*

> Raw thought, recorded verbatim in substance; nothing here is decided. The sizing guesses below are **hypotheses to test**, not requirements. The requirements-shaped questions underneath them are the parts to actually resolve.

**Initial hypothesis (unvalidated)**
- A small single instance (e.g. AWS nano-class) is likely sufficient for steady state.
- Estimated peak: **~100 simultaneous connections, each pulling on the order of hundreds of KB.**
- Ability to **scale out behind a load balancer** is desirable — not for steady state, but so a peak can be absorbed quickly.
- Hosting on a major cloud would supply baseline availability; **open question is what that costs.**

**Questions to resolve**
- **Load shape, not just load size.** With ~4 events/year (§2), utilization is extremely spiky: near-idle most of the year, with sharp peaks at registration open, at check-in (§9), and at PEL submission (§10). Is the cost model better served by always-on small, or by something that scales to near-zero between events?
- **What uptime is actually required, and when?** Availability likely matters far more during an event than between events. Is there a single SLO, or a "game-day" tier and an "off-season" tier?
- **Recovery objectives.** How much data loss is tolerable (RPO), and how fast must service return (RTO)? These plausibly differ by data class: a lost PEL is annoying, a lost payment record is not.
- **Backup scope and restore rehearsal.** What's backed up, how often, retained how long, and has a restore ever been tested? Interacts with the 7-day delete queue (§5) — backups must not silently resurrect purged data.
- **Cost ceiling and who pays.** A volunteer-run LARP has a real budget limit; that limit is a requirement and should be written down as a number.
- **Third-party dependencies.** Payment processing (§8) and email notifications (cross-cutting principles) are external services with their own availability. What's the behavior when they're down mid-event?
- **Who operates this?** Ongoing patching, monitoring, and incident response require a human. Identifying that role (and its bus factor) is a requirement, not an implementation detail.

**Cross-references**
- **§18 (offline support) is already a partial DR strategy.** Much of the event-critical surface is required to work offline, which meaningfully lowers the cost of an outage *during* an event — but not the cost of losing data.
- **§11 at-the-door registration explicitly has no offline path**, so it is the flow most exposed to an outage at the worst possible moment.

---

## Known & Accepted Risks

Consolidated register of risks that have been surfaced and **deliberately accepted** rather than mitigated. Anything added here should be an explicit decision, not an oversight.

**Explicitly accepted**

| # | Risk | Context | Note |
|---|---|---|---|
| R1 | **Mutual root demotion.** A root-mode user can strip another user's root tag; two disagreeing holders can each demote the other. | §15 | Accepted. Partially bounded by the min-2 floor and second-to-last-holder refusal, which prevent total lockout but not a power struggle. |
| R2 | **Ban evasion is trivial.** The ban record matches on name and email; both are easily changed. The system flags a probable match but does not block. | §5 | Accepted. Enforcement is ultimately social — staff recognizing the person. |
| R3 | **Ban PII has no separate gating.** Anyone who can see bans sees the retained name and email, including for accounts otherwise deleted. | §5 | Accepted. |
| R4 | **Document deletion is revocation in practice.** Hiding a document from the player is the intended effect, despite the general "documents cannot be revoked" principle. | §4 | Accepted deliberately; §4 wording amended to name the exception. |
| R5 | **Root bypasses permission evaluation entirely.** No tag configuration can constrain a root-mode user. | §15 | Accepted. Mitigated by the append-only log, the separate log-reading tag, the visible mode indicator, and the 1-hour idle revert. |
| R6 | **Epi-pen-level allergy severity is self-reported.** Staff act on player-supplied data for potentially life-threatening conditions. | §5 | Inherited from original doc. **Flagged for confirmation** — this is the highest-consequence accepted risk in the document. |

**Nominated — not yet confirmed as accepted**

| # | Risk | Context |
|---|---|---|
| R7 | **At-the-door registration has no offline path** (§11), making it the flow most exposed to an outage at the worst possible moment. | §11, §18, §20 |
| R8 | **Backup retention may resurrect hard-deleted PII**, silently violating the deletion promise. | §5, §20 |
| R9 | **Undelete after PII purge restores an identity-less stub** — characters and PELs return, orphaned from a person who can no longer log in. | §5 |

---

## Open Items / Explicitly Deferred to Later Phases
- Automated missed-event credit logic.
- PEL section → staff-member routing implementation (seam exists now, feature later).
- Wiki as part of bunnyarchives (plot content abstraction layer exists now; native wiki is later).
- Ad hoc peer-to-peer offline sync network.
- **Roles** as a grouping above tags (§15) — reintroduce only when their purpose is clearer.
- **Operations/DR/uptime/cost sizing (§20)** — captured, deliberately unexamined for now.
- **LLM assistance (staff-facing)** — deferred wholesale; this set of ideas needs refinement before it's actionable. As sketched: an optional LLM tool reads PELs and suggests which sections should route to which plots; it **never auto-applies** (each individual suggestion requires explicit staff approval); approved suggestions are tagged in metadata as applied "via AI agent" (mirroring the metadata convention for staff-moved PEL info, §17).
