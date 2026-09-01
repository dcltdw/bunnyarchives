<!-- CHECKPOINT — disaster-recovery snapshot, 20260831T202057
     State: Permissions session closed, incl. Forbidden-page explainability refinement (§15).
     Intended to be deleted once merged back into the running doc. -->

# Bunnyarchives — Requirements (Running Doc)

Architecture is out of scope. This document tracks discovered requirements as they're gathered; it should grow as more surfaces.

---

## Cross-Cutting Principles

- **One deployment serves one campaign.** Bunnyarchives is not multi-tenant; another campaign adopting it would run its own deployment.
- **Mistakes are reversible.** Every operation should be reversible by an appropriately-permissioned person. Named exceptions:
  - Under first-one-in-wins conflict handling, a refused stale edit is never applied — though it is preserved as an Awaiting Merge record (§12, §18).
  - **PII is hard-deleted** on account deletion and does not come back (§5). Everything else is soft-deleted and restorable.
  - **The root action log is append-only** — not modifiable or deletable, including by root (§15).
  - **Ban records persist**, including their retained PII copy, even after the source account is deleted. Un-banning is reversible; the ban record is not erased (§5).
  - **Legal documents (e.g. waivers) persist forever, unchanged** (§5).
- **Notifications go to email.** Keep an abstraction seam so another service could be swapped in — it's an easy barrier to create — but no other implementation is expected.
- **Players and staff are both people.** One person can be both (staff register for events, run NPCs, may play PCs). Player and staff are facets of a common "Person" concept.


## Terminology

- **Revision** — the staleness marker on an editable unit, used by conflict handling (§12).
- **Supersession** — a *new document* replacing an old one, chained by reference (§4). Not an edit to the old one.
- **Edition** — an externally published rulebook version (§1).
- **Tag** — reserved exclusively for the permission system (§15).
- **Label** — inert organizational metadata on content; never gates anything (§23).
- **Status** — the behavior-bearing lifecycle state of authored content: Draft → Working → Committed → Issued (§23).
- **Draft** — the authoring status: unfinished, owner-only (§23). *Not* the refusal artifact.
- **Awaiting Merge** — a refused edit preserved for manual resolution (§12). Formerly loosely called a "draft".
## 1. Character Creation Wizard

- Three tiers: minimal (next field only), somewhat helpful (brief reminders), very helpful (quotes rulebook sections).
- **The player chooses the tier, and can switch tiers mid-flow.**
- **Someone will need to manually annotate the rulebook** so the wizard can quote from it — the wizard must never LLM-summarize the rulebook itself.
- **The system knows which edition of the rulebook it references** — annotations are tied to a rulebook edition.
- **The current edition is a policy** (P19), changed via `PolicyAdmin`; §22 timing rules govern when a flip takes effect.
- **On an edition flip: old-edition annotations persist** (tied to their edition); **the new edition's annotation set starts empty.**
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
- **Season pass duration is configurable**: either a number of events or a fixed end date. **No grandfathering** — a change applies to outstanding passes; remediation is the existing manual refund/credit path (§22, risk R10).
- **Missed-event credit**: manual by default (staff-applied per player). Automated/policy-driven credit is a **later-phase feature**.
- **Refunds are policy-driven and entirely manual** — no automated refund flow.
- **Event cancellation**: if an entire event is canceled, staff need an easy way to record each player's choice, typically one of: full refund; partial or full donation of the money to the LARP (no refund, no future credit); credit transferred to the next event. (Illustrative, not locked: one admin screen with tickboxes covering all players in one go.)
- **Staff discounts**: always free (100% comp), no partial-discount case. Free registration is denoted by a dedicated staff tag (e.g., `StaffFreeReg`). Audit trail = the registration record itself.
- **Community fund**:
  - Single ongoing pool (not reset per event).
  - Donations accepted from anyone: a **dedicated donation page** exists, and regular registration checkout offers a **donate option**.
  - **Policy dictates whether Community Fund Managers must approve each use of funds, or players may self-select** fund use.
  - Balance visibility is a policy (§22). Like all policies it is changed via `PolicyAdmin`; Directors are expected holders of that tag.
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
- **Documents follow the content lifecycle (§23)** — Draft → Working → Committed → **Issued**. Issued is the player-visible state; only Documents reach it.
- **Documents are permanent once issued — cannot be revoked, *except* by explicit staff deletion** (see below). There is no player-facing revocation and no expiry.
- **Deletion (§5 soft-delete model) applies at two levels, with distinct meanings, both gated by a document-deletion tag:**
  - **Delete an Issuance** — fixes a mis-issuance to one character. That character loses access; other recipients are unaffected.
  - **Delete the Document** — removes it from every character who received it, in one action.
  - **Hiding it from the player is the intended effect.** This is deliberate and is the one sanctioned form of revocation.
- However, a new document **may supersede** an old one; system must handle this gracefully (supersession, not deletion — see Terminology).
  - **Supersession obscures previous versions from players.** Staff can still reach them, but not casually — e.g., a confirmation dialog, a SUPERSEDED watermark, or other high-visibility markings.

## 5. Player Profiles

- Fields: name, email, emergency contact, **dietary needs** (one field — quick-flag common items like dairy/gluten plus free text, e.g. kosher; subsumes the former *dietary restrictions* and *food allergies*), notable non-food allergies (separate field). Phone optional. **No address field, ever** (no physical mail use case in the system).
- **Emergency contact is structured**: name, relation, phone, email.
- **Non-food allergies**: scope = anything that could induce a reaction requiring an inhaler or, especially, an epi-pen. **Epi-pen-level allergies must be flagged/highlighted distinctly as true dangers**, easy for staff to spot quickly. **Epi-pen severity is self-reported by the player.**
- **Medical data (allergies, emergency contact) is tag-gated** like everything else (§15). It does **not** generally appear in reports, but any report may opt to include it. It **is** part of the offline "download all info" export (§18).
- **Minors are allowed.** At registration, every player is checked for **valid waiver(s) on file; which waivers are required is configurable** — this covers the minor case.

### Deletion Model (resolved)

- **Deletion is a "marked for deletion" bit, not removal.** It is fundamentally a **filter**, not a security control — deleted objects drop out of default views and out of reports.
- **Account deletion** is invokable by the player ("delete my account") or by staff. A **deleted account cannot be logged into.**
- **PII is hard-deleted. Everything else is soft-deleted.** Character sheets and PELs persist with the delete bit set.
  - **PII is defined (resolved — blocker cleared): contact channels (email, phone), medical data (dietary needs, allergies, epi-pen flags, meal selections incl. free text), third-party contact data (emergency contact, all fields), and credentials (password hash).**
  - **Not PII — survives deletion: name, participation record (attendance history, payment facts), waivers, and creative output** (sheets, PELs, comments, annotations — soft-deleted).
  - **PII purge happens after a 7-day hold.** **Within the hold, undelete is a simple bit-flip** restoring everything, PII included.
  - **Financial data splits.** *That a payment occurred* — amount, date, what for — **persists**. **Credit card info, including billing address, is collected during payment and deleted as soon as the transaction clears or fails.** CC info therefore exists only inside payment triggers — event registration, donations (§8), and meal plans (§21). Rationale: encryption at rest was considered and rejected, so minimizing the window in which sensitive financial data exists is the mitigation.
- **Carve-out: the audit trail retains PII past account deletion.** Hard-delete does not sweep the audit log — otherwise the "who made this change" tracing the log exists for would be destroyed. This is a named exception to the deletion promise, alongside bans.
- **Carve-out: waivers — like all legal documents — persist forever, unchanged.** The *document* persists even though it contains PII (signature, DOB for minors, guardian info); the corresponding profile *fields* still purge.
- **Undelete after purge:** root, or a holder of the **`Can Undelete Account`** tag (§15), **re-adds an email address to the account, then starts the Forgot Password flow**, which emails the user to set a new password. This is also the returning-player path: staff undelete the old person record and attach the current email, restoring characters and history.
- **Cascade:** deleting a parent marks its children deleted. **Undeleting a parent undeletes its children.** Each object also carries its own independent bit, so an object deleted separately before its parent stays deleted when the parent is restored.
- **Deleted characters do not occupy character-cap slots** (§6). Note the deliberate asymmetry with skill caps: **character caps gate creation only** ("can I make a new one?"), whereas **skill caps apply retroactively**.
- **Deleted PELs** only arise when a player submits a PEL and then deletes their account; whether a deleted PEL satisfies other criteria is not germane.
- **Visibility of deleted content** requires the **`Read Deleted Content`** tag (§15). For holders, every relevant view offers an **"also include deleted content"** toggle.
  - **This is a UI-level toggle, per page — not an underlying design concern.**
  - **The toggle is not sticky** — it does not persist across views or sessions.
  - **Filters are orthogonal to rules/tags.** Root mode does *not* auto-reveal deleted content; a root user still toggles the filter.
  - **Children of a deleted parent do not surface** unless specifically navigated to. Considered an unlikely path.
- **Deleted content is excluded from the offline export** (§18).
- **Derived content outlives its source.** Where staff have derived material from something later deleted — e.g. plot notes written from a PEL (§17) — the derived material is staff-owned and **persists**. **A provenance line replaces the deleted source**, which holders of `Read Deleted Content` can reveal via the same per-page toggle.

### Bans

- **Bans survive account deletion.** The **Ban object holds its own copy of the PII** (**name and email**), so the source object's PII can be hard-deleted while the ban record persists.
- **The system flags a probable match at registration; it does not block.** Evasion by changing name and email is a **known and accepted risk**.
- **Ban PII is visible to anyone who can see bans** — no separate gating.
- **Bans are reversible, but the retained PII is not purged on un-ban.** The ban record functions as a **ban audit log**.
- **The player is told** that deletion will not remove their information, because they are banned.

## 6. Characters

- Players start with **zero characters** (base case) — zero Active is valid at any time.
- **Maximum of 1 Active PC per player** (kind = PC only; NPCs exempt — §13).
- **PCs require staff approval** — a status axis parallel to Speculative/Active/Retired: **Awaiting Approval / Approved**. The axis applies to PCs only; NPCs are staff-created and ungated.
- Skill cap: **starting value is configurable** (default 2, global), can be raised over time; individual characters can have a different cap than the global default.
- **Global skill cap increases apply retroactively** to existing characters.
- **Adding skills is player self-service**; a system for **character advancement** exists.
- **Character self-service includes a Print option**: outputs player name, character name, event date, and character skills; more fields may be added by configuration.
- **Speculative PC cap is policy-driven**: default 3, applying to **kind = PC only** (P8). **NPCs are uncapped** — the former staff cap of 10 is removed.
- **The rulebook is the authoritative source of the skill list; bunnyarchives holds a manually-maintained Skill Catalog mirroring it** (same posture — and plausibly the same people — as the §1 rulebook annotation work). "Defined in the rulebook" means authority, not storage.
- **Character sheets reference catalog entries by identity (picklist) — never free text.** This is what makes the §16 skill-count report, the §9 prep-flag report, self-service adding, and printing work without fragmentation.
- **Catalog entries are identity-bearing — the label model again (§23):** the UUID is stable across renames; renaming propagates to every sheet; rename collisions are rejected; **a genuinely new skill gets a new UUID** (an edition renaming *Deception* → *Subterfuge* is a rename of the same entry, preserving continuity).
- **The catalog is validating**: entries carry attributes — name, requires-prep flag (§9), and **prerequisites** — and self-service enforces **the cap and the prerequisites**. (No cost attribute in the shipped system — see Advancement.)

### Advancement & Validation

- **The shipped advancement system is the cap, full stop**: the number of skills a character may hold is **set universally by staff** (P6, with per-character override), and **staff will raise that number over time** (retroactive, per §6/§22). There is no currency, no XP, no spend.
- **Priorities, verbatim intent:**
  - **Absolutely needed**: the cap system above.
  - **Nice to have**: seams such that a currency module (e.g. earn XP by attending events or making donations; XP converts to CP; CP is spent on advancement) and a cost module could be swapped in later.
  - **Not needed**: implementing any such alternative system.
  - **More important than the seams**: not complicating currency/costs just to have them. **If the seams don't survive to the end product, that's fine.**
- **Prerequisites are expressive**: a boolean structure (AND/OR) over condition atoms — required skill UUIDs, **counts** ("any 3 of …"), and **non-skill conditions**. **Condition types are pluggable sub-modules with a seam for easy add/drop.**
- **Counts range over an explicit picklist of skills, a category, or a list of categories.** Catalog entries therefore carry a **category attribute**; the **category vocabulary is a policy-managed list** (P20), while assigning categories to a skill is ordinary catalog editing.
- **Shipped non-skill condition sub-modules**: **events attended** (reads attendance records, §9), **holds a specific document** (reads Issuance records, §4 — issuance as unlock), and the **staff-manual flag** ("staff says this character qualifies" — the universal escape hatch). Note the cross-module reads these create: prerequisite validation depends on attendance and issuance data.
- **Validity violations are flagged, never auto-resolved.** A soft-deleted prerequisite, or any rule change that strands a character's existing skills, flags the sheet for staff attention; resolution is manual (edition changes ship human-applied conversion rules). Application of the §22 current-state rule.
- **No re-pricing, ever** — acquisitions are completed historical facts (§22) and are never re-processed. Moot in the shipped system (nothing has a price) but binding on any future currency module.
- **Creating a custom skill is gated by its own separate tag**, distinct from catalog editing (plot staff and the rules committee may be different people; one-tag-one-thing, §15). **Custom skills go through the §23 lifecycle like any entry** — a mid-event boon takes the quick Draft → Working → Commit hop before it's grantable.
- **Skill removal from the game = soft-delete of the catalog entry (§5 model).** The delete bit acts as a filter on every character sheet that holds the skill; **undeleting the entry instantly restores it everywhere.** Deleted skills follow the §5 rules: excluded from reports and prints, revealable via `Read Deleted Content`.
- **Catalog entries follow the §23 lifecycle** (Draft → Working → Committed; entries never reach Issued). **The player-facing picklist shows Committed entries only** — staff stage a new edition's skill changes in Working and commit them when the edition publishes.
- **Custom skills are catalog entries too**, marked custom, each with its own UUID: granting one means **"create a brand-new custom skill, or choose from the picklist of existing custom skills."** Amy's *firewalking* and Bob's *waterwalking* are fundamentally different entries; a later grant of *firewalking* to Carol reuses Amy's entry.
- **Catalog editing is gated by its own dedicated tag** (one-tag-one-thing, §15).
- **PC and NPC are one Character entity, distinguished by an immutable `kind` discriminator.** NPCs share the Active/Retired/Speculative status model; all other rules key off kind:
  - **Max 1 Active, the approval axis, and cap/prerequisite validation apply to PCs only.** An Active NPC means "in play in the world," not exclusive; NPC sheets may break player rules by design (they still reference catalog skills by identity, custom skills included).
  - **NPCs never surface in the player list (§7) or the skill-count report (§16)** — reports are over PCs unless a report explicitly states otherwise.
  - **Status transitions**: PC rules as written below; NPC statuses are staff-managed freely.
  - **Documents are issuable to NPCs** — the villain holds the letter; issuance-as-unlock (§6) works unchanged.
- **`kind` is immutable.** PC-to-villain and NPC-to-PC conversions are handled by **re-creating the character as the other kind** — the new character starts fresh; history and issuances stay with the old one, which is typically Retired.
- **NPCs are person-owned** (the creator, by default) **with ownership transfer. Transfer is free** — no dedicated permission; anyone who can edit the NPC can reassign its owner, with accountability via the audit trail (§15).
- **Owner-deletion rescue (rescue-after model)**: when an owner deletes their account, the §5 cascade soft-deletes their NPCs as usual — no blocking, no auto-transfer. **Transferring a deleted person's NPC to a new owner clears the inherited deletion**, restoring it. Zero new machinery; the cascade plus transfer already covers it.
- **Per-event casting (who portrays an NPC) is deliberately out of scope** — Cast Groups (§16) remain a schedule filter only.
- **There is no separate Dead state — Retired covers character death.**

## 7. Player Visibility (Other Players List)

- **Security-motivated**: unauthenticated users see nothing. Even among players, the list is gated to players Staff has marked as having *attended* (not merely registered) at least one event.
- **No list at all is shown to players who haven't attended an event** — explicitly to prevent a malicious actor from registering just to check whether a specific person (e.g., a stalking target) is registered.
- Default visible fields: first name, last name initial, Active PC name (unambiguous — max one Active PC, §6; NPCs never appear here).
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
  - Availability conflicts between the two mechanisms resolve as **last-in wins** — an instance of the general state-transition rule (§12), not an exception. The use case is: "Hey, John — can you mark me as available for that? Thanks!"
- **Staff member status** during an event: default value set is **NOT ARRIVED, OFF SITE, ON SITE, IN AN ENCOUNTER, IN A MODULE, RESTING, SLEEPING, HAS LEFT** — the set is configurable (more/fewer states). Surfaced on a staff-status dashboard (§16) that also works offline (§18).
- At-the-door registration: **no offline support** — payment can't reasonably be queued offline, so this flow assumes connectivity.

## 12. Staff-Only Character Notes

- Section headings (e.g., "for next event," "archive," "to do," "potential ideas," "lines & veils," "plots") are **configurable via global policy**, not by individual staff members.
- **Edit conflict handling**: **first-one-in wins**, where **"first in" means first to reach the server** — not first by wall-clock. An offline edit made earlier but synced later loses. Deliberate; write it down because someone at an event will feel wronged by it.
- **The versioned unit is declared per object type**, fine-grained by default: the **section** for sectioned objects (working-knowledge notes), the **field** for form-like objects (character sheets). Edits to *different* units of the same object never conflict.
- **Content vs. state rule**: **content edits are first-one-in-wins; state transitions and signals are last-in-wins** (idempotent where possible) — status flips, availability marks (§13), check-in marks. Refusing a status flip because someone else flipped it first is nonsense.
- **A refused edit is preserved as an *Awaiting Merge* record, scoped to its owner.** It stores: owner, target unit, **the base revision it was made against, and the complete submitted content**. **Diffs are derived on demand — never stored as the source of truth.** (A stored patch can fail to apply once the base moves on; stored full content diffs cleanly against anything, at any time.)
- **On refusal, the owner is notified by email — notice plus link only, no content.** The diff (submitted vs. current, derived on demand) is viewed in-app, keeping edit content inside the permission system (resolves R11).
- **Side-by-side in-app merge/resolution UI is deferred.** Minimum viable resolution: the emailed diff plus ordinary editing.
- Same model for offline sync conflicts (§18) — one rule across the system.

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

**Four access-relevant mechanisms exist, and only one is tags.** (1) **Status** — lifecycle states carry visibility (Draft is owner-only; Issued is player-visible; §23). (2) **Ownership** — some rules key on the owner relationship (Awaiting Merge records §12, NPCs §13). (3) **Tags** — the grant machinery below. (4) **Filters** — orthogonal to all of the above (deleted-content visibility, §5). Do not "simplify" one into another; the separation is deliberate.

- Every piece of information/action supports **granular access**; ultimately **every element of an object can carry its own CRUD permissions**, differing from its peers and from its parent.
- **"Parent" is a valid security setting**: an element may declare that whoever can perform a CRUD operation on its parent can perform it on the element too.
- **Authoring happens in three layers**: **type-level defaults** (a permission template shared by all objects of a type — where ~95% of configuration lives) → **per-object override** → **per-element override**. Each layer defaults to the one above; overrides are exceptional. Full per-element capability is retained, but it is the escape hatch, not the authoring surface.
- **Tags are the access primitive.** Tags are deliberately low-level, purpose-scoped groups (Community Fund Managers exists solely for the community fund). Tags are **self-naming** (e.g., `RegistrationApprover`, `StaffFreeReg`) and **may not be strictly hierarchical** (no implied nesting/inheritance).
- **Roles are adopted, as live objects**: a Role is a named set of tags. Granting a Role grants the set; **editing a Role changes every holder's effective tags** — this is what keeps "Director" meaningful as the org evolves. People may hold loose tags alongside Roles. **Roles do not nest.**
- **Effective permission is the AND (intersection) of all relevant tags' permissions — a DENY anywhere results in DENY.** Consequence: holding an R-only tag alongside a CRUD tag leaves the person with R only; the R tag's missing grants deny C/U/D.
  - Guardrail for that misconfiguration: **mutually exclusive tag groups** — a person may hold at most one tag from such a group.
  - **AND is retained deliberately**, with two mitigations:
    1. **Grant-time reduction warning**: when granting a tag would *reduce* the recipient's effective access anywhere, the system flags it to the granter.
    2. **Permission explainability**: any page can answer "what permissions are relevant here?" — a per-tag contribution breakdown (e.g. *Tag A grants CRU, Tag B grants R; intersection R, which is why you can't create*).
    3. **The Forbidden page explains itself**: on direct navigation to a page without R access, the person's held tags are listed. **If and only if the denial is a conflict** — some held tag would have granted R but another relevant tag denies it — **the in-conflict tags are called out**, highlighted; the person's remaining, irrelevant tags are listed separately, unhighlighted. (Self-inspection only: a person sees their own tags, never anyone else's.)
- **One tag controls permission for exactly one thing — no re-use.**
- **Granting and revoking tags is gated by a single `TagAdmin` tag** (all non-root tags). **The root tag is grantable only from root mode.** Per-tag granter lists are a future refinement (deferred). TagAdmin is knowingly the second-most-powerful tag — its holder can self-grant anything but root; the audit trail is the check.
- **Root tag**: exactly one tag has CRUD on everything. This is distinct from what Directors hold: Directors (the highest-ranking people) might carry, e.g., universal CR, and must deliberately **switch personas into the root tag, sudo-style**, for anything beyond that.

### Tag Registry

One row per tag. Every future "gated by its own tag" statement in this document must add a row. Names marked † are suggested, not yet fixed (tags are self-naming; finalize on creation).

| Tag | Gates | Ref |
|---|---|---|
| root | Everything (sudo-style mode switch) | §15 |
| `RootLogReader`† | Reading the root action log | §15 |
| `TagAdmin` | Granting/revoking all non-root tags | §15 |
| `PolicyAdmin` | Changing all policy values | §22 |
| `OpsAdmin`† | Editing the OpsParameters table | §22 |
| `Read Deleted Content` | Revealing deleted content (per-page toggle) | §5 |
| `Can Undelete Account` | Post-purge account undelete (re-attach email) | §5 |
| `DocumentDeleter`† | Deleting Documents and Issuances | §4 |
| `CatalogEditor`† | Skill Catalog editing (incl. categories) | §6 |
| `CustomSkillCreator`† | Creating custom skills | §6 |
| `Committer`† | Working → Committed transitions | §23 |
| `RegistrationApprover` | Approving registrations | §11 |
| `StaffFreeReg` | Free staff registration | §11 |
| `CommunityFundManager` | Community fund operations | §2 |
| `BanViewer`† | Seeing bans (incl. retained PII) | §5 |
| Medical-data tag(s) | Medical/emergency profile fields | §5 |

### Root Mode (resolved)

- **Root bypasses permission evaluation entirely** — it does not participate in the AND-intersection, so a DENY elsewhere cannot override it.
- **Root does not bypass filters.** Deleted-content visibility remains a toggle even in root mode (§5).
- **Explicit mode switch with visible UI change**, so the holder always knows which persona is active.
- **Idle timer: root mode reverts to the user's normal permission set after one hour.**
- **All root actions are logged. The root action log is append-only — root can neither modify nor delete it.**
- **Reading the root log is a separate tag** from holding root — e.g., an oversight committee that can review all root actions without being able to take them.
- **A root-mode user can strip another user's root tag.** Mutual demotion is a **known and accepted risk**.

### Audit Trail

- **All-actions logging is a system-wide binary setting, default ON.**
- **Retention is configurable; default 6 months.** Short retention serves "oops, undo that"; retention set to forever serves "who made this change" provenance tracing back to campaign start.
- **Volume is a real concern at long retention — compression is required thinking**, since access all the way back to campaign start is desirable.
- **An entry records: actor, timestamp, object, before/after values.**
- **Before/after values may contain tag-gated data** (e.g. medical fields). Root changing such values is highly unusual, and restorability is worth more than the leak — **accepted**.
- **The audit trail retains PII past account deletion** (named carve-out, §5).
- **The root-action log never prunes** — it is exempt from the retention setting.
- **Policy-change entries are retained for the greater of 1 year and the general retention setting** (§22).
- **Root actions are always logged, regardless of the system setting.** Protection against a malicious root actor in the moment is a minimal concern; the protection is that their actions are logged and reviewable.

### Root Tag Count & Bootstrap Lock

- **The number of root-tag holders is configurable — both minimum and maximum.**
- **The system enforces a floor of 2 on the minimum**, which **lives in code**, not configuration, and is not overridable. **A configured minimum below 2 is rejected with an explanation** — not silently coerced (§22 validation rule).
  - Example: `max 5, min 3` is honored as written. `max 5, min 1` is rejected: "minimum must be ≥ 2." The bootstrap wizard validates its file the same way.
- **The system refuses to remove the root tag from the second-to-last holder**, and **blocks a root holder from deleting their own account.**
- **At install there are zero root holders.** Until the configured minimum is reached, a **Bootstrap Lock** is in effect:
  - A large banner appears on **every** page.
  - **Player registration is disabled.**
  - **No player-facing pages are available.**
  - Bootstrap Lock lifts once the configured minimum is met (e.g., at 3 holders under a `min 3` config).

## 16. Reports (Staff)

- Pre-event: registered list; registered-and-unpaid list; door-payment-planned list; **skill-count report**: for each skill, how many Active PCs of registered players have it (e.g., of the 10 characters registered for the next event, how many have Deception?). All report visibility is policy/tag-gated.
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
- **Offline sync conflict model**: first-one-in wins (first *to the server* — §12); conflicting followups are refused and saved as Awaiting Merge records — same unified model as live-edit conflicts (§12).
- **The offline client displays a freshness indicator** — when its data snapshot was taken (e.g. "data from Friday 6:00 PM") — so staff know what they're trusting at Sunday check-in.
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

## 21. Meals

- **Feature flag: meals are optionally off**; the feature starts disabled.
- **Offered to both players and staff.** Options: **omnivore, vegetarian, vegan**.
- **Dietary needs** (field name: *dietary needs*, not *food allergies*): common ones (e.g. dairy, gluten) are easy to note; **custom entries via free text** (e.g. kosher).
- **§21 reads dietary data from the §5 profile and asks the person to confirm or update it** — it does not collect an independent copy.
- **Prices may vary across options.**
- **Multi-day events**: there may be **one price covering all meals of the event, with individual days broken out** (i.e., per-day purchase is possible).
- **Payment is collected before the event**, like registration (§8, §11). **Usually bundled with registration into a single charge**, but **adding a meal plan later, as a separate charge, is allowed.**

## 22. Policy

### Model

- A **Policy** is a named campaign setting with: **name, type, default (the default lives in code), scope, current value**.
- **Policies are values. Permissions (§15) are who-can-do-what.** They are separate concepts and **permanently separate machinery** — Permissions do not reuse the policy model (resolved).
- **Feature flags are boolean policies** in the same registry (e.g. meals §21) — no second mechanism.
- Post-bootstrap, policy values live **in the DB only** (§15 bootstrap transition).

### Authority

- **A single `PolicyAdmin` tag changes all policy values.** No per-policy authority exceptions. The earlier "balance visibility settable only by Directors" is superseded — Directors are simply expected holders of `PolicyAdmin`.

### Scope

- Legal scopes (complete for now; may be revisited): **global**; **per-role default**; **global with per-object override**.

### Timing & Retroactivity

- **Changes take effect immediately and govern future evaluations only.** Millisecond-level race conditions are explicitly not a concern.
- **General rule: current state is always evaluated against current policy; completed historical facts are never re-processed.** Skill caps and season passes are *not* exceptions under this formulation — a pass's remaining validity is current state, evaluated at each future registration; past attendance is never clawed back.
- **Declared exception: the speculative PC cap gates creation only** — existing over-cap state is never flagged (§6).
- **No grandfathering, anywhere.** This avoids memory-of-purchase-time-terms and exception handling. Consequence accepted as **R10**.

### Validation

- **Invalid values are rejected with an explanation — never silently coerced.** (This supersedes the earlier root min/max coercion behavior; see §15.)

### Policy vs. Ops vs. Secrets

- **`OpsParameters` (DB table)**: non-secret infrastructure settings — SMTP host/port/from-address, payment-processor webhook URLs, merchant IDs, and the like. **Edited via a separate ops tag, not `PolicyAdmin`** (one-tag-one-thing, §15).
- **Secret layer** (AWS Secrets Manager or equivalent — security-first posture for all credential info): SMTP password, payment API keys, every credential. Never in the DB, never in policy.
- **Must live in file/env forever** — permanent, enumerated exceptions to "the bootstrap file goes inert" (§15):
  1. The **DB connection string** (cannot live in the DB it points to).
  2. **Bootstrap credentials for the secret layer itself.**
  - Additions to this list must be enumerated here as discovered.

### Audit

- Policy changes are recorded in the general audit trail and **retained for the greater of 1 year and the general retention setting** (§15).

### Policy Registry

One row per policy. Every future "configurable" or "policy-driven" statement in this document must add a row. Defaults marked *unstated* are open elicitation targets.

| # | Policy | Type | Default | Scope | Notes |
|---|---|---|---|---|---|
| P1 | Event frequency | number/year | 4 | Global | §2 |
| P2 | Season pass duration | event count *or* end date | unstated | Global | No grandfathering (R10). §2 |
| P3 | Community Fund approval mode | enum: managers-approve / self-select | unstated | Global | §2 |
| P4 | Fund balance visibility | boolean/enum | unstated | Global | §2 |
| P5 | Required waivers | list | unstated | Global | Checked at each registration (current-state rule). §5 |
| P6 | Skill cap | number | 2 | Global + per-character override | Current-state evaluation (§6) |
| P7 | Character-print extra fields | list | name, character, event date, skills | Global | §6 |
| P8 | Speculative PC cap | number | 3 | Global (kind = PC only) | NPCs uncapped. Declared exception: creation-gating only. §6 |
| P9 | Registration approval required | boolean | unstated | Global | §11 |
| P10 | Staff status value set | list | 8 named states | Global | §13 |
| P11 | Working-knowledge section headings | list | as listed | Global | §13 |
| P12 | Audit logging enabled | boolean | on | Global | §15 |
| P13 | Audit retention | duration | 6 months | Global | Root log exempt; policy changes max(1yr, this). §15 |
| P14 | Root tag count min/max | numbers | min 2 / max unstated | Global | Floor of 2 in code; invalid values rejected loudly. §15 |
| P15 | On-deck window default | duration | unstated | Global + per-encounter override | §16 |
| P16 | Schedule phases | list | setup, lunch, dinner, strike | Global | §16 |
| P17 | Meals feature enabled | boolean | off | Global | §21 |
| P18 | Label aspect vocabulary | list | suggested list: Brainstorm, Doc, Writeup, Run Notes | Global | §23 |
| P19 | Current rulebook edition | identifier | unstated | Global | §1 |
| P20 | Skill category vocabulary | list | unstated | Global | §6 |

## 23. Content Lifecycle & Labels

### Status

- **Every authored object carries a Status**: **Draft → Working → Committed → Issued.** One consistent chain across all content types — consistency is a virtue here.
  - **Draft** — unfinished; **scoped to the owner only**.
  - **Working** (a.k.a. uncommitted) — shared; up for review.
  - **Committed** — final; staff-visible.
  - **Issued** — made public to players. **Only Documents (§4) reach Issued**; plot notes and the like stop at Committed.
- **Statuses drive behavior; labels never do.** Status is a first-class field with a small fixed value set — not implemented as labels.
- **Not all content is collaborative** — PEL submissions are single-author (their *templates* are collaborative). The chain is universal; which transitions a type uses may vary.
- **Mutability per state**: **Draft and Working are freely editable. Committed is immutable — fixing it means demoting to Working**, itself an audited status transition. **Issued is immutable forever**; supersession only (§4).
- **Working → Committed is gated by its own dedicated tag** (`Committer`†, one-tag-one-thing, §15); likely rolled into a Role.
- **Issued is entered automatically on first issuance** of a Committed document — not a precondition staff set by hand.
- Status transitions are state changes under the §12 content-vs-state rule: last-in-wins, idempotent where possible.

### Labels

- **Labels are inert organizational metadata**: searchable, filterable, and they gate nothing — visibility control is exclusively the tag system's job (§15).
- **Compound model: Base × Aspect.**
  - The **Aspect vocabulary is a policy-managed list** (P18) — e.g. *Brainstorm, Doc, Writeup, Run Notes* — covering the roles a piece of content plays from conception to end. **A default suggested list ships with the system.**
  - **Bases are proper nouns** (plot/arc names: *Foo*, *Icarus*), created by staff as needed.
  - **Adding a new base yields all its compound labels for free**: create *Foo* and *Foo Brainstorm*, *Foo Doc*, *Foo Writeup*, *Foo Run Notes* are immediately available.
- **An object may carry multiple labels.**
- **Labels are identity-bearing**: content references a label's identity (e.g. a UUID); the display string is an attribute looked up from it. **Renaming a label is therefore a single metadata edit and every labeled object follows automatically** — the correction mechanism for typo'd bases (*Ikarus* → *Icarus*).
- **Rename collisions are rejected** (§22 validation rule) — renaming a base to a string another base already owns fails loudly. Cleanup is manual: relabel the smaller set, delete the emptied base. No merge operation.
- **Rationale — free-form labels fragment**: Amy's "Icarus doc" and Bob's "Icarus writeup" should group together and won't. A controlled vocabulary with automatic compounding keeps grouping intact while staying one-click cheap.

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
| R6 | **Epi-pen-level allergy severity is self-reported.** Staff act on player-supplied data for potentially life-threatening conditions. | §5 | **Accepted — confirmed.** Self-report is the only viable mechanism; no alternative exists. Highest-consequence accepted risk in the document. |
| R10 | **No policy grandfathering.** A policy change (e.g. shortening season pass duration) can retroactively alter goods people already paid for. | §2, §22 | Accepted. Remediation is the existing manual refund/credit path; no new system machinery. |
| R12 | **Offline snapshots freeze permissions at export time.** A tag revoked Saturday doesn't reach Friday's laptop export; revocation propagates only at next export. | §15, §18 | Accepted. Snapshot-at-export semantics; nothing to build. |

**Nominated — not yet confirmed as accepted**

| # | Risk | Context |
|---|---|---|
| R7 | **At-the-door registration has no offline path** (§11), making it the flow most exposed to an outage at the worst possible moment. | §11, §18, §20 |
| R8 | **Backup retention may resurrect hard-deleted PII**, silently violating the deletion promise. | §5, §20 |
| R11 | ~~Refusal emails carry edit content outside the permission system~~ **RESOLVED**: notice-plus-link only; diff viewed in-app (§12). | §12, §18 |
| R9 | ~~Undelete after PII purge restores an identity-less stub~~ **RESOLVED**: post-purge undelete re-attaches an email and runs Forgot Password (§5). | §5 |

---

## Open Items / Explicitly Deferred to Later Phases
- Automated missed-event credit logic.
- PEL section → staff-member routing implementation (seam exists now, feature later).
- Wiki as part of bunnyarchives (plot content abstraction layer exists now; native wiki is later).
- Ad hoc peer-to-peer offline sync network.
- Per-tag granter lists (v1: single `TagAdmin`; §15).
- Side-by-side in-app merge/resolution UI for refused edits (§12).
- Full per-object edit-history viewing (current state + audit trail only, for now; audit prunes per §15 retention).
- **Operations/DR/uptime/cost sizing (§20)** — captured, deliberately unexamined for now.
- **LLM assistance (staff-facing)** — deferred wholesale; this set of ideas needs refinement before it's actionable. As sketched: an optional LLM tool reads PELs and suggests which sections should route to which plots; it **never auto-applies** (each individual suggestion requires explicit staff approval); approved suggestions are tagged in metadata as applied "via AI agent" (mirroring the metadata convention for staff-moved PEL info, §17).
