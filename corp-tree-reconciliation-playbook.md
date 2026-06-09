# Corporate Tree Reconciliation Playbook

## Scope

Reconciles a client-provided corporate entity list against an Intapp party's Corporate Structure — covering the Tree, Shareholders, Beneficial Owners, Management/Board, and Close Affiliates. Identifies what already exists, what needs to be created, and what requires human judgement. Resolves ID conflicts before any bulk decisions. Checks parent chain integrity at the point of creation. Produces a full reconciliation report with a parent mismatch summary.

## Goals
- Minimise duplicate party creation by matching at a high name threshold before offering to create
- Use provided IDs as a safeguard and conflict alarm, not as a primary matching signal
- Ensure no entity is created without its direct parent being resolved first
- Produce a complete audit trail — every provided entity accounted for in the final report

---

## Test Fixtures

- **Party 1767** — Gazprom Pipeline, LLC. Tree rooted at Innospec Inc. (042006515). Use Scenario 1 (flat names) for a clean baseline run.
- **Party 1764** — GAZPROM TEPLOENERGO MO, OOO. Two trees + Board + UBOs loaded. Use Scenario 5 (hierarchy with IDs) for the most complex run.

---

## Playbook Variables

### Hardcoded
- `{threshold}` = `0.95` — minimum name similarity score for auto-accept. Applies to both entity name and parent name comparisons. Configurable per run.
- `{matchSkill}` = `corp-tree-match` — scoring skill
- `{interpretSkill}` = `corp-tree-interpret` — classification skill

### Per-Run Variables

#### Party
- `{partyId}` — Intapp system ID of the anchor party
- `{partyName}` — display name of the anchor party
- `{provider}` — `externalDataProvider` value from the party response
- `{sourcesLoaded}` — list of non-null Corporate Structure sections found in the response

#### Entity Lists
- `{candidatePool}` — flat array of all existing nodes extracted in Step 1: `{ id, name, parentId, source }`
- `{providedList}` — parsed array of user-supplied entities: `{ name, id?, parentName?, parentId? }`
- `{signalTable}` — output of `corp-tree-match`: one signal row per provided entity
- `{bucketTable}` — output of `corp-tree-interpret`: signal rows + Bucket + Interpretation columns
- `{resolvedList}` — `{providedList}` enriched with resolved IDs after Phase 1 completes

#### Session State
- `{idConflicts}` — rows from `{bucketTable}` where Top ID Match = Yes and name score < threshold, or parent IDs disagree
- `{matches}` — rows with Bucket = Match and no ID conflict
- `{noMatches}` — rows with Bucket = No Match
- `{judgements}` — rows with Bucket = Judgement and not in `{idConflicts}`
- `{outcomes}` — map of `providedName → { resolvedId, outcome, parentMatch, notes }` built throughout Phase 1

---

## Overall Behavior Rules
- **Silent computation:** All tool calls, signal scoring, candidate pool construction, and bucket classification produce zero user-facing output. Only defined display outputs and prompts are rendered.
- **No narration:** Never describe what you are about to do, what a tool call returned, or what computation just ran.
- **Exit checks are silent:** Exit checks are internal invariant assertions. If one fails, output only: `⚠️ Exit check failed: <reason>` and halt.
- **GO TO is unconditional:** When a step says GO TO Step N, proceed immediately — no confirmation, no summary.
- **Step labels are internal:** Never render step numbers or phase labels in user-facing output.

---

## Execution Rules (Non-Negotiable)
- NEVER narrate tool calls, reasoning, data processing, or intermediate results
- NEVER explain what you are about to do or just did
- NEVER output any text between receiving tool results and the next defined step output
- The ONLY permitted outputs are: (a) Progress Header + Status, (b) defined display output, (c) defined user prompt
- NEVER render step or substep labels in output

---

## Progress Display

### Phases
```
Fetch & Parse  ›  Match & Score  ›  ID Conflicts  ›  Reconcile  ›  Hierarchy Check  ›  Report
```

### State Icons
| Position | Icon | Bold? |
|---|---|---|
| Completed phases | ☑ | Yes |
| Current phase | ▣ | Yes |
| Upcoming phases | ☐ | No |

### Phase Emojis
| Phase | Emoji |
|---|---|
| Fetch & Parse | 🔍 |
| Match & Score | ⚙️ |
| ID Conflicts | 🔑 |
| Reconcile | ✏️ |
| Hierarchy Check | 🌲 |
| Report | 📋 |

### Rendering Contract

With content below:
```
{Progress Header}

{emoji} *{Status text}*

---

{step content}
```

Pure processing (no content):
```
{Progress Header}

{emoji} *{Status text}*
```

### Example — Pure Processing (Step 1)
```
☑ **Fetch & Parse**  ›  ▣ **Match & Score**  ›  ☐ ID Conflicts  ›  ☐ Reconcile  ›  ☐ Hierarchy Check  ›  ☐ Report

⚙️ *Scoring 16 entities against 32 candidates…*
```

### Example — Display + Prompt (Step 5)
```
☑ **Fetch & Parse**  ›  ☑ **Match & Score**  ›  ☑ **ID Conflicts**  ›  ▣ **Reconcile**  ›  ☐ Hierarchy Check  ›  ☐ Report

✏️ *13 clean matches found — confirm to accept*

---

| Provided Entity | Matched Entity (ID) | ...
```

---

## Phase 1 — Reconcile

---

### Step 1: Fetch Party and Build Candidate Pool

| Field | Value |
|---|---|
| **Type** | Processing |
| **Progress Header** | `▣ **Fetch & Parse**  ›  ☐ Match & Score  ›  ☐ ID Conflicts  ›  ☐ Reconcile  ›  ☐ Hierarchy Check  ›  ☐ Report` |
| **Status** | 🔍 *Fetching party {partyId}…* |
| **Notification Rule** | No output except Progress Header + Status |

- Call `intappCommon:get_party` with `party_id` = `{partyId}`
- Set `{partyName}`, `{provider}`, `{sourcesLoaded}` from response
- Flatten all non-null Corporate Structure sections into `{candidatePool}` (see Appendix A)
- If `corporateTrees` is null or empty → **GO TO** Step END

**Exit check:** `{candidatePool}` contains at least one node.

---

### Step 2: Parse Provided Entity List

| Field | Value |
|---|---|
| **Type** | Processing |
| **Progress Header** | `▣ **Fetch & Parse**  ›  ☐ Match & Score  ›  ☐ ID Conflicts  ›  ☐ Reconcile  ›  ☐ Hierarchy Check  ›  ☐ Report` |
| **Status** | 🔍 *Parsing provided entity list…* |
| **Notification Rule** | No output except Progress Header + Status |

Detect input format and extract into `{providedList}`. See Appendix B for format detection rules.

**Exit check:** `{providedList}` contains at least one entity with a non-empty name.

---

### Step 3: Run Matching and Classification

| Field | Value |
|---|---|
| **Type** | Processing |
| **Progress Header** | `☑ **Fetch & Parse**  ›  ▣ **Match & Score**  ›  ☐ ID Conflicts  ›  ☐ Reconcile  ›  ☐ Hierarchy Check  ›  ☐ Report` |
| **Status** | ⚙️ *Scoring {n} entities against {m} candidates…* |
| **Notification Rule** | No output except Progress Header + Status |

- Run `corp-tree-match` with `{candidatePool}` and `{providedList}` → set `{signalTable}`
- Run `corp-tree-interpret --threshold {threshold}` on `{signalTable}` → set `{bucketTable}`
- Partition `{bucketTable}` into `{idConflicts}`, `{matches}`, `{noMatches}`, `{judgements}` (see Appendix C)

**Exit check:** Every row in `{providedList}` has exactly one corresponding row in `{bucketTable}`.

---

### Step 4: Resolve ID Conflicts (one by one)

| Field | Value |
|---|---|
| **Type** | Analysis and User Prompt |
| **Progress Header** | `☑ **Fetch & Parse**  ›  ☑ **Match & Score**  ›  ▣ **ID Conflicts**  ›  ☐ Reconcile  ›  ☐ Hierarchy Check  ›  ☐ Report` |
| **Status** | 🔑 *{n} ID conflict(s) to resolve* |
| **Notification Rule** | Render opening line, then one prompt card per conflict |

If `{idConflicts}` is empty → **GO TO** Step 5.

Open with:
> "There are {n} entity IDs you provided that exist in the corporate tree already with conflicting names. Let's resolve these."

For each conflict in `{idConflicts}`, render:
```
ID:             {providedId}
Existing Name:  {topNameMatch}
Provided Name:  {providedName}

Would you like to:
A) Keep the existing name
B) Use your provided name (edit existing)
C) Create as new entity (we will make the ID unique)
```

On each decision:
- **A** → record `{providedName}` as Matched to `{topNameMatch}`, existing name preserved. Remove from `{idConflicts}`, add to `{matches}`.
- **B** → flag existing node for rename, record as Matched. If rename is actioned: re-call `intappCommon:get_party`, re-run Steps 1–3 for all remaining unresolved entities.
- **C** → drop the provided ID, remove from `{idConflicts}`, add to `{noMatches}`.

**Exit check:** `{idConflicts}` is empty before proceeding.

---

### Step 5: Bulk Accept Clear Matches

| Field | Value |
|---|---|
| **Type** | Display → User Prompt |
| **Progress Header** | `☑ **Fetch & Parse**  ›  ☑ **Match & Score**  ›  ☑ **ID Conflicts**  ›  ▣ **Reconcile**  ›  ☐ Hierarchy Check  ›  ☐ Report` |
| **Status** | ✏️ *{n} clean match(es) found — confirm to accept* |
| **Notification Rule** | Render match table then prompt |

If `{matches}` is empty → **GO TO** Step 6.

Render:

| Provided Entity | Matched Entity (ID) | ID Match | Name Match | Provided Parent | Matched Parent (ID) | Parent ID Match | Parent Name Match | Source |
|---|---|---|---|---|---|---|---|---|

Then prompt:
> "Accept all {n} matches? Type **accept** to confirm, or list any row numbers you'd like to review individually."

- **accept** → record all as Matched in `{outcomes}`. **GO TO** Step 6.
- **Row numbers listed** → move those rows to `{judgements}`, record remainder as Matched. **GO TO** Step 6.

**Exit check:** All rows in `{matches}` have an outcome recorded in `{outcomes}`.

---

### Step 6: Bulk Create / Skip No Matches

| Field | Value |
|---|---|
| **Type** | Display → User Prompt |
| **Progress Header** | `☑ **Fetch & Parse**  ›  ☑ **Match & Score**  ›  ☑ **ID Conflicts**  ›  ▣ **Reconcile**  ›  ☐ Hierarchy Check  ›  ☐ Report` |
| **Status** | ✏️ *{n} unmatched entit(y/ies) — add or skip* |
| **Notification Rule** | Render no-match list then prompt |

If `{noMatches}` is empty → **GO TO** Step 7.

For each entity in `{noMatches}`, render the interpretation and offer:
- *No match found. No parent info provided.* → **Add** or **Skip**
- *No match found. Parent also not found.* → **Add** or **Skip**
- *No match found. Parent matches — can be added as new child under [X].* → **Add under [X]** or **Skip**

For each **Add** decision, perform immediate parent check before calling `intappCommon:create_party` (see Appendix D).

Call `intappCommon:create_party` (see Appendix E) for each approved entity and any missing ancestors (root-first).

Record all outcomes in `{outcomes}`. **GO TO** Step 7.

**Exit check:** All rows in `{noMatches}` have an outcome recorded in `{outcomes}`.

---

### Step 7: Judgement Calls (one by one)

| Field | Value |
|---|---|
| **Type** | Display → User Prompt |
| **Progress Header** | `☑ **Fetch & Parse**  ›  ☑ **Match & Score**  ›  ☑ **ID Conflicts**  ›  ▣ **Reconcile**  ›  ☐ Hierarchy Check  ›  ☐ Report` |
| **Status** | ✏️ *{n} judgement call(s) to review* |
| **Notification Rule** | One prompt card per entity |

If `{judgements}` is empty → **GO TO** Step 8.

Open with:
> "There are {n} entities that need a closer look. Let's go through them one at a time."

Render one card per entity. Card format varies by interpretation type:

**Name matches, parent contradicts:**
```
Entity:          {providedName}
Name score:      {score}%
Tree parent:     {treeParentName} ({treeParentId})
Provided parent: {providedParentName}

A) Accept match — keep existing tree placement
B) Accept match — move to provided parent
C) Create as new entity under provided parent
D) Skip
```

**Fuzzy name, parent corroborates:**
```
Entity:          {providedName}
Name score:      {score}%
Best match:      {topNameMatch} ({topNameMatchId})
Parent:          {providedParentName} — matches tree ✓

A) Accept as same entity
B) Create as new entity
C) Skip
```

**Fuzzy name, no corroboration:**
```
Entity:          {providedName}
Name score:      {score}%
Best match:      {topNameMatch} ({topNameMatchId})
No parent info provided.

A) Accept as same entity
B) Create as new entity
C) Skip
```

Any Create decision triggers the immediate parent check from Appendix D.

Record all outcomes in `{outcomes}`. 

**Exit check:** All rows in `{judgements}` have an outcome recorded in `{outcomes}`.

---

## Phase 2 — Hierarchy Check

---

### Step 8: Enrich Provided List with Resolved IDs

| Field | Value |
|---|---|
| **Type** | Processing |
| **Progress Header** | `☑ **Fetch & Parse**  ›  ☑ **Match & Score**  ›  ☑ **ID Conflicts**  ›  ☑ **Reconcile**  ›  ▣ **Hierarchy Check**  ›  ☐ Report` |
| **Status** | 🌲 *Building enriched entity list…* |
| **Notification Rule** | No output except Progress Header + Status |

For every entity in `{providedList}` with outcome Matched or Created, fill in the resolved `id` from `{outcomes}`. Store as `{resolvedList}`.

**Exit check:** Every Matched or Created entity in `{outcomes}` has a non-empty resolved ID in `{resolvedList}`.

---

### Step 9: Compare Parent Chains and Flag Mismatches

| Field | Value |
|---|---|
| **Type** | Display → User Prompt |
| **Progress Header** | `☑ **Fetch & Parse**  ›  ☑ **Match & Score**  ›  ☑ **ID Conflicts**  ›  ☑ **Reconcile**  ›  ▣ **Hierarchy Check**  ›  ☐ Report` |
| **Status** | 🌲 *Comparing parent chains…* |
| **Notification Rule** | Render mismatch summary if any found, else proceed silently |

For every resolved entity in `{resolvedList}` that has a provided parent, compare:
- Provided parent (resolved to ID via `{resolvedList}`)
- Tree node's actual `parentId` from `{candidatePool}`

If they differ → flag as mismatch.

If no mismatches → **GO TO** Step COMPLETE.

Render mismatch summary:
```
⚠ Parent mismatch: {entityName}
   Provided parent: {providedParentName} ({providedParentId})
   Tree parent:     {treeParentName} ({treeParentId})
```

For each mismatch, prompt:
- **A)** Keep tree placement — tree is correct, client data is outdated
- **B)** Move in tree — client data reflects a real ownership change
- **C)** Note for follow-up — flag in report, decide later

Record decisions. **GO TO** Step COMPLETE.

**Exit check:** All mismatches have a recorded decision.

---

### Step COMPLETE

| Field | Value |
|---|---|
| **Type** | Processing → Display |
| **Progress Header** | `☑ **Fetch & Parse**  ›  ☑ **Match & Score**  ›  ☑ **ID Conflicts**  ›  ☑ **Reconcile**  ›  ☑ **Hierarchy Check**  ›  ▣ **Report**` |
| **Status** | 📋 *Generating reconciliation report…* |

Render report header:
```
Reconciliation Report
─────────────────────────────────────────────
Party:          {partyName}  (Party ID: {partyId})
Date:           {today}
Provider:       {provider}
Input format:   Scenario {n}
Threshold:      {threshold * 100}%
Sources loaded: {sourcesLoaded}

Matched:                   {n}
Created:                   {n}
Skipped:                   {n}
Skipped (ancestor blocked):{n}
Flagged for rename:        {n}
Parent mismatches found:   {n}
Total reviewed:            {n}
```

Render entity table:

| Provided Name | Provided ID | Resolved ID | Outcome | Parent Match | Notes |
|---|---|---|---|---|---|

**Outcome values:** `Matched` / `Created (ID: {x})` / `Skipped` / `Matched — flagged for rename` / `Skipped — ancestor not resolved`

---

### Step END

| Field | Value |
|---|---|
| **Type** | Display |
| **Progress Header** | *(none)* |
| **Status** | *(none)* |

Render:
> "Reconciliation could not proceed. Reason: {reason}"

Triggered when: party not found, candidate pool empty, or provided list empty.

---

## Appendix A: Building the Candidate Pool

Flatten all non-null Corporate Structure sections from the `get_party` response into `{candidatePool}`:

**Tree nodes** — for each tree in `corporateTrees`, recurse `rootCompany.subCompanies`:
```
walk(node, parentId=null):
  append { id: node.externalId, name: node.name, parentId: parentId, source: "Tree" }
  for each child in node.subCompanies: walk(child, node.externalId)
```
Skip duplicate `externalId` values across trees (first occurrence wins).

**Shareholders** — `shareholders.shareholders[]` → `{ id, name, parentId: "", source: "Shareholder" }`

**Beneficial Owners** — `beneficialOwners.beneficialOwners[]`, recurse nested `beneficialOwners`:
```
walkUBO(ubo, parentId=""):
  append { id: ubo.id, name: ubo.name, parentId: parentId, source: "Beneficial Owner" }
  for each child in ubo.beneficialOwners: walkUBO(child, ubo.id)
```

**Management/Board** — `boardMembers.boardMembers[]` → `{ id, name, parentId: "", source: "Management/Board" }`

**Close Affiliates** — `closeAffiliates.closeAffiliates[]` → `{ id, name, parentId: "", source: "Close Affiliate" }`

Skip any section that is null.

---

## Appendix B: Input Format Detection

| Scenario | Format | Fields extracted |
|---|---|---|
| 1 | Plain text, one name per line | `name` only |
| 2 | Excel: Entity Name, Parent Entity | `name`, `parentName` |
| 3 | Excel: Entity Name, Relationship Type | `name`, `relationship` (used as source hint) |
| 4 | Excel: Entity Name, Parent Name | `name`, `parentName` |
| 5 | Excel: Entity ID, Entity Name, Parent ID, Parent Name | `name`, `id`, `parentName`, `parentId` |

**Relationship → source hint (Scenario 3):**

| Value | Preferred source |
|---|---|
| GUO, Ultimate Owner | Tree (root) |
| Subsidiary, Child | Tree |
| Shareholder | Shareholders |
| Beneficial Owner, UBO | Beneficial Owners |
| Board, Director, Management | Management/Board |
| Affiliate, Close Affiliate | Close Affiliates |
| blank / unknown | All sources |

---

## Appendix C: Bucket Partitioning Rules

After running `corp-tree-interpret`, partition `{bucketTable}` as follows:

**`{idConflicts}`** — rows where:
- `Top ID Match = Yes` AND `Top Name Match Score < {threshold}`, OR
- `Top ID Match = Yes` AND `Top ID Match Parent ID ≠ Top Name Match Parent ID`

**`{matches}`** — rows where `Bucket = Match` AND not in `{idConflicts}`

**`{noMatches}`** — rows where `Bucket = No Match`

**`{judgements}`** — rows where `Bucket = Judgement` AND not in `{idConflicts}`

---

## Appendix D: Immediate Parent Check

Triggered before every `create_party` call during Steps 6 and 7.

1. Does the provided parent name/ID exist in `{candidatePool}`? → proceed
2. Has the provided parent already been created this session (in `{outcomes}`)? → proceed
3. Parent not found anywhere → halt and surface:

```
⚠ Before creating [{entityName}], its parent needs to be resolved:
   Parent: {parentName} — not found in tree and not yet created.

   → Add [{parentName}] first | Skip (blocks [{entityName}] and all its children)
```

If the parent needs to be created, apply this check recursively to the parent before creating it. Stop as soon as a node that exists is found. If user skips any ancestor → mark that ancestor and all descendants as `Skipped — ancestor not resolved` in `{outcomes}`. Do not create anything below a skipped node.

---

## Appendix E: create_party API Body

```json
{
  "name": "{entityName}",
  "partyType": "Company",
  "externalDataProvider": "None",
  "includeCorporateTree": false,
  "includeInSearchStrategyValidation": true
}
```

- For person entities (board members, UBOs) → `"partyType": "Person"`
- If party type is uncertain → call `my-custom-intapp-mcp4:get_party_types` first and present options to the user
- Record returned `id` and `partyId` in `{outcomes}`

