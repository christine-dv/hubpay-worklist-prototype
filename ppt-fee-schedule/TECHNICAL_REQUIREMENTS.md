# PPT Fee Schedule Data Entry Form — Technical Requirements

**Status:** Draft for engineering handoff
**Prepared for:** Pipelines team (Switchboard)
**Source of truth for exact UI behavior:** `index.html` in this folder (interactive prototype — see Section 10)
**Author context:** Reverse-engineered from a fully built HTML/JS prototype plus the original user story/AC. Where the two disagree, this document flags it explicitly and defers to the prototype.

---

## 1. Overview & Context

**What this is:** A structured data-entry form that captures a Provider Pass-Through (PPT) fee schedule for a single Order at the time the order is loaded/configured in Switchboard. The form replaces free-text or ad-hoc fee schedule communication with a structured set of rows (Vendor Type, Payment Type, Vendor Name, Fee Type, $ Amount, Max $ Amount, CNA Fee) and generates a single machine-parsed "fee text" string that downstream systems consume as the canonical, structured representation of the client-approved fee schedule.

**Why it exists:** Today, PPT fee schedules are communicated informally (e.g., email, notes) and re-keyed by hand into systems like IMS/HubPay. This causes delay, error, and inconsistent fee data at the start of retrieval activity. This form is meant to be the single point of entry so that any downstream system (e.g., HubPay's AP invoice-approval workflow, which checks vendor invoice amounts against the client-approved fee schedule) has accurate, structured fee data from the moment retrieval work begins on an order.

**Who builds/owns it:** Per the user story, this is built and hosted by the **Pipelines team** as part of the standard Order Configuration form set/workflow in Switchboard Pipelines. It is not a standalone tool — it's one form within order configuration.

**Who consumes it downstream:** At minimum, the generated fee-text string is the "machine-readable fee data" referenced in the user story — intended for consumption by systems that need to validate vendor invoices/fees against the client-approved schedule (e.g., HubPay's AP fee-approval workflow, where invoices are checked against a client-approved fee before payment). The prototype does not define an API, event, or storage contract for this hand-off — see Section 9.

**Audience for this document:** Engineers on the Pipelines team implementing this from scratch. This is not a product pitch — it specifies exact field lists, rules, and the fee-text generation algorithm precisely enough to build and unit-test against.

---

## 2. Scope

### In scope (covered by the prototype, and therefore by this spec)
- The fee schedule row-entry form itself: fields, conditional visibility, defaults, per-field validation.
- The two order-level toggles ("No PPT fee approvals" and "Approval required for all per page fees") and their effect on the form and generated text.
- Duplicate-row detection and blocking logic.
- The fee-text preview generation algorithm (Section 7) — this is the core deliverable, since it's machine-parsed downstream and must be reproduced exactly.
- The Review → Approve/Go Back/Help flow as a UI state machine.

### Explicitly NOT covered by the prototype — flagged as unvalidated/undesigned
The original acceptance criteria include several requirements that describe backend/order-lifecycle behavior. **The prototype is a static, client-side, no-backend form. It does not implement or validate any of the following, and they should be treated as open backend/integration requirements for the Pipelines team to design, not as confirmed behavior:**

- **"Fee schedule is required for Order submission (except when digital only)"** — there is no order-submission gating logic in the prototype. The receiving team must design where/how this validation is enforced against the Order submission workflow, and what "digital only" resolves to as a checkable order attribute.
- **"Only one active fee schedule can exist per order at a time; submitting a new form replaces the prior active schedule"** — the prototype's confirmation screen displays static text ("Any previously active schedule for this order has been replaced") but performs no actual versioning, persistence, or replacement. There is no data model for schedule history/versions, no order-schedule association, and no logic to deactivate a prior schedule. This entire capability needs to be designed (see Section 9).
- **RBAC / visibility** — "Visible to: Client Success users with access to order entry/configuration in Switchboard Pipelines" is not enforced or represented anywhere in the prototype; there is no auth/role check.
- **Persistence** — there is no backend, no API call, no database write. "Approve" in the prototype only transitions client-side UI state.
- **Digital-only order handling** — not represented at all in the prototype's data model.

### Also out of scope
- Editing/re-opening a previously submitted fee schedule (the prototype only supports "start a new fee schedule" from scratch after confirmation, wiping prior state).
- Any localization/currency handling beyond USD `$` formatting (see Section 9).
- A real, structured vendor directory for HIH vendor names (currently free text — see Section 9).

---

## 3. Data Model

### 3.1 Order-level state

| Field | Type | Default | Notes |
|---|---|---|---|
| `noPPT` | boolean | `false` | "No PPT fee approvals for this Order" — original AC name; prototype checkbox label is "Approval required for all PPT fees" (same semantic: no fee schedule rows apply). |
| `approvalRequiredPerPage` | boolean | `false` | "Approval required for all per page fees" — not in the original AC; added via iterative feedback. |
| `rows` | array of Row | one empty Row | See 3.2. |

### 3.2 Row fields

| Field | Type | Default | Required? | Valid values |
|---|---|---|---|---|
| `vendorType` | enum | `""` (unselected) | Always required | `Provider/Hospital`, `General HIH`, `HIH Carveout` |
| `paymentType` | enum | `"Pre+Post Pay"` | Always has a value (not user-clearable) | `Pre+Post Pay`, `Post Pay Only` for `Provider/Hospital`/`General HIH`; `Pre+Post Pay`, `Post Pay Only`, `Exclusion` for `HIH Carveout` only |
| `vendorName` | free text | `""` | Required only when `vendorType === "HIH Carveout"` | Any non-empty string after trim |
| `feeType` | enum | `"Per Chart"` | N/A (always has a value; hidden entirely when `paymentType === "Exclusion"`) | `Per Chart`, `Per Page` (`Per Page` removed from the option list — and any row currently set to it reset to `Per Chart` — whenever `approvalRequiredPerPage` is checked) |
| `amount` | decimal string (numeric input) | `""` | Required unless `paymentType === "Exclusion"` | Non-negative number, 2-decimal step; `""`/NaN fails validation |
| `cnaFee` | decimal string (numeric input) | `""` | Never required | Non-negative number; optional on every row (see 3.3) |
| `maxAmount` | decimal string (numeric input) | `""` | Never required | Non-negative number; only meaningful/visible when `feeType === "Per Page"` |

**Note on `vendorType` option changes from the original AC:** The original AC specified Vendor Type options `Provider/Physician`, `Hospital`, `General HIH`, `HIH Carveout`, `Exception`, with mutual exclusion between `Provider/Physician` and `Hospital`. The prototype has evolved this to exactly three options: `Provider/Hospital` (merged), `General HIH`, `HIH Carveout` — `Exception` was removed. **Build against the three-option list; the mutual-exclusion rule no longer applies (there's only one merged option).**

### 3.3 CNA Fee is a column, not a Fee Type

Per the current prototype (`hasCnaField()` returns `true` unconditionally), **every row** — regardless of Vendor Type, as long as `paymentType !== "Exclusion"` — exposes a CNA Fee input alongside the primary fee amount. This is a deliberate design change from an earlier iteration where CNA might have been modeled as a `feeType` option requiring a duplicate row; now a single row can carry both a primary fee (Per Chart or Per Page) and a CNA fee simultaneously.

---

## 4. Field-level Conditional Visibility & Validation

| Field | Shown when | Required when | Validation on Review |
|---|---|---|---|
| Vendor Type | Always | Always | Error "Required" if blank |
| Payment Type | Always | N/A — always has a default | none |
| Vendor Name | `vendorType === "HIH Carveout"` (else rendered as `—`, not editable) | Shown AND value blank after trim | Error "Required" |
| CNA Fee | `paymentType !== "Exclusion"` (shown for all 3 vendor types) | Never | none (blank/zero simply omitted from preview) |
| Fee Type | `paymentType !== "Exclusion"` | N/A | none |
| PPT Fee Amount | `paymentType !== "Exclusion"` | Shown AND (`amount === ""` or `parseFloat(amount)` is NaN) | Error "Required" |
| Max $ Amount | `feeType === "Per Page"` AND `paymentType !== "Exclusion"` | Never | none |

Additional dynamic behavior:
- Changing **Vendor Type** on a row: resets `paymentType` to `"Pre+Post Pay"` if the current value isn't in the new type's allowed Payment Type list; clears `vendorName` if the new type doesn't use it; clears `cnaFee` if the new type doesn't support it (currently unreachable since all types support CNA, but the code path exists — implement it as written).
- Changing **Fee Type** away from `"Per Page"` clears `maxAmount`.
- Changing **Payment Type** to `"Exclusion"` clears `amount`, `maxAmount`, `cnaFee` and resets `feeType` to `"Per Chart"` on that row (since all of those fields become hidden).
- **Amount of `0`** passes field validation (it's a valid parsed number) but is treated identically to blank in the fee-text algorithm (see `isBlankOrZero`, Section 7) — it will simply not appear in the generated text. This is a real discrepancy between "valid input" and "renders in output" that the receiving team should be aware of; it is not flagged as an error to the user.
- A row can be removed via the "✕" button unless it is the only remaining row (`rm-btn` is `disabled` when `rows.length === 1`).
- **Prototype behavior — production should reconsider:** field errors are only computed and displayed on clicking "Review" (`validateRows()`); there is no live/blur-level validation as the user types.

---

## 5. Order-level Controls

Two order-level checkboxes, both above the row table:

1. **"Approval required for all PPT fees"** (`noPPT`, maps to the AC's "No PPT fee approvals for this Order")
   - When checked: the entire row table and "+ Add row" control are hidden (not just disabled — not rendered at all). Row *data* is retained in memory but ignored. The fee text preview becomes the fixed string `"Approval required for all PPT fees"`, overriding everything else including `approvalRequiredPerPage` and all row content. Row validation and duplicate-row validation are both skipped entirely when this is checked (Review proceeds unconditionally).
   - When unchecked: form reverts to the row table with whatever row data was already in state.

2. **"Approval required for all per page fees"** (`approvalRequiredPerPage`)
   - Disabled (not interactable) whenever `noPPT` is checked, but **its underlying value is not cleared** — if a user checks `noPPT`, then unchecks it later, `approvalRequiredPerPage` retains whatever value it held before.
   - When checked: removes `"Per Page"` from the Fee Type dropdown on every row (present and future), and for every existing row currently set to `"Per Page"`, resets `feeType` to `"Per Chart"` and clears `maxAmount`. This reset is applied immediately and silently — no confirmation, and (prototype behavior) no re-run of duplicate detection after the bulk reset, meaning a mass reset could create a new duplicate combination that isn't caught until the user clicks Review (which does run the full duplicate check as a final gate).
   - When checked, appends the fixed trailing segment `"Approval required for all per page fees"` to the generated fee text (see Section 7, Step E), regardless of whether any row currently has fee data.

**Precedence:** `noPPT` unconditionally overrides everything (rows, `approvalRequiredPerPage`, duplicate checks, validation). `approvalRequiredPerPage` only has effect when `noPPT` is unchecked.

---

## 6. Duplicate-Row Validation Rules

There are **two distinct, independently-triggered** duplicate rules. Both operate only on rows that have a `vendorType` selected (rows with no vendor type chosen yet are ignored entirely by both checks).

### 6.1 Row identity key ("combo" duplicate)

```
rowKey(row):
    vendorKey = row.vendorType
    if row.vendorType == "HIH Carveout":
        vendorKey += "|" + trim(row.vendorName).toLowerCase()
    feeTypeKey = "" if row.paymentType == "Exclusion" else row.feeType
    return vendorKey + "||" + row.paymentType + "||" + feeTypeKey
```

Two rows are a **"combo" duplicate** if they produce the same `rowKey`. Match is case-insensitive and whitespace-trimmed on Vendor Name (Vendor Name only participates in the key for `HIH Carveout`; `Provider/Hospital` and `General HIH` rows are compared purely on Vendor Type + Payment Type + Fee Type, with Fee Type ignored for `Exclusion` rows).

**Important edge case to preserve or explicitly decide on:** because `feeType` is part of `rowKey`, two `Provider/Hospital` (or `General HIH`) rows with the same Payment Type but **different** Fee Type (e.g., one `Per Chart`, one `Per Page`) are **not** flagged as a combo duplicate — and since those vendor types have no Vendor Name field, the name-only rule (6.2) doesn't catch them either. Both rows are accepted. However, the fee-text vendor-grouping key in `buildFeeText()` (Section 7, Step A) does **not** include `feeType` — so these two "non-duplicate" rows will silently merge into a single vendor group in the output text, combining their Per Chart and Per Page fees into one line. This is a real inconsistency between the duplicate-guard key and the text-grouping key in the current prototype; flag for the receiving team to decide whether to align them (Section 9).

### 6.2 Vendor-name-only duplicate ("name" duplicate)

```
vendorNameOf(row):
    return trim(row.vendorName).toLowerCase() if row.vendorType == "HIH Carveout" else ""
```

Two rows are a **"name" duplicate** if they resolve to the same non-empty `vendorNameOf(...)` value, **regardless of Payment Type or Fee Type**. This only applies to `HIH Carveout` rows (the only vendor type with a Vendor Name field) — e.g., two `HIH Carveout` rows both named "Healthmark" are a duplicate even if one is `Pre+Post Pay`/`Per Chart` and the other is `Post Pay Only`/`Per Page`.

### 6.3 When each check runs

- **Live, single-row check + auto-remove** (`checkDuplicateAndRemove(rowId)`): runs on `vendorType` change, `paymentType` change, `feeType` change, on Vendor Name field `change` (blur/commit, not every keystroke), and once immediately after a new row is added. For the row in question, compares it against every *other* row (first match wins: combo checked before name). If a conflict is found: **Prototype behavior — production should use a non-blocking, in-context inline warning/toast instead** — the prototype calls a blocking browser `alert()` with a distinct message per conflict type, then unconditionally deletes the newly-changed/added row from state (no undo).
- **Full-form gate on "Review" click** (`hasDuplicateRows()`): iterates all rows once, applying both checks (combo first, then name) globally across the whole row set. If any duplicate exists anywhere, blocks navigation to Review with a blocking `alert()` and does **not** remove any row — the user must resolve it manually. This check is skipped entirely if `noPPT` is checked.

---

## 7. Fee Text Preview Generation Algorithm

This is the most important part of the spec: the output string is machine-parsed downstream, so it must be reproduced exactly, not "similarly." **Caveat for the receiving team:** this algorithm was reverse-engineered from three sample output strings provided by the stakeholder, then iteratively refined through live feedback (payment type leads the string; Provider/Hospital orders before HIH groups; Per Page gets a "pp" suffix; CNA is its own column; blank/$0 is omitted; Exclusion rows need no fee type). **It has not been validated against a formal written spec ("Requirement 3")** — see Section 9. The prototype's current, exact behavior is defined below and should be treated as the contract to implement and unit test against.

### 7.0 Helper functions

**`fmtAmount(n)`** — formats a numeric amount as a dollar string:
```
fmtAmount(n):
    num = parseFloat(n)
    if isNaN(num): return "$0"
    rounded = round(num, 2)
    if rounded is a whole number: return "$" + round(rounded)      // e.g. "$15"
    else: return "$" + rounded.toFixed(2)                          // e.g. "$15.50"
```

**`isBlankOrZero(v)`** — a value is omitted from the preview if:
```
v == "" or v == null or v == undefined or isNaN(parseFloat(v)) or parseFloat(v) == 0
```

**`vendorLabel(row)`**:
```
Provider/Hospital -> "Prov"
General HIH       -> "HIH"
HIH Carveout      -> trim(row.vendorName) or "Unnamed Vendor" if blank
```

**`paymentLabel(paymentType)`**:
```
"Pre+Post Pay"  -> "Pre+Postpay"
"Post Pay Only" -> "Postpay Only"
(anything else, e.g. "Exclusion") -> returned unchanged (not used in practice since Exclusion rows are handled separately)
```

**`vendorCategory(vendorType)`**: `"provider"` if `Provider/Hospital`, else `"hih"` (covers both `General HIH` and `HIH Carveout`).

### 7.1 Top-level algorithm

```
buildFeeText():
  1. If noPPT is true: return "Approval required for all PPT fees"   // short-circuits everything below

  2. rowsWithVendor = rows where vendorType is set
     included = rowsWithVendor where paymentType != "Exclusion"
     excluded = rowsWithVendor where paymentType == "Exclusion"
     // Note: the code's literal filter for `included` also checks (amount != "" OR feeType truthy);
     // since feeType always has a default value, this second condition is effectively always true.

  3. STEP A — group `included` rows into vendor entries:
     vendorMap = {}   // keyed by: vendorType + "|" + (vendorName.trim() if HIH Carveout else "") + "|" + paymentType
     for each row r in included:
        cnaAmt = r.cnaFee
        hasPrimary = not isBlankOrZero(r.amount)
        hasCna     = not isBlankOrZero(cnaAmt)
        if not hasPrimary and not hasCna: continue   // nothing to show for this row yet

        key = r.vendorType + "|" + (r.vendorName.trim() if r.vendorType=="HIH Carveout" else "") + "|" + r.paymentType
        entry = vendorMap.get(key) or create { vendorType: r.vendorType, vendorName: r.vendorName, paymentType: r.paymentType, fees: {} }

        if hasPrimary:
           feeStr = fmtAmount(r.amount)
           if r.feeType == "Per Page":
              feeStr += "pp"
              if not isBlankOrZero(r.maxAmount): feeStr += " (max " + fmtAmount(r.maxAmount) + ")"
           entry.fees[r.feeType] = feeStr        // keyed "Per Chart" or "Per Page"
        if hasCna:
           entry.fees["CNA"] = fmtAmount(cnaAmt)

     // NOTE: this grouping key does NOT include feeType. Two rows with the same
     // vendorType/vendorName/paymentType but different feeType merge into ONE vendor
     // entry carrying both a "Per Chart" and a "Per Page" fee (see Section 6.1's caveat).

  4. Build each vendor group's fee string, in fixed order:
     FEE_ORDER = ["Per Chart", "Per Page", "CNA"]
     for each vendorMap entry v:
        parts = []
        for ft in FEE_ORDER:
           if v.fees[ft] is undefined: continue
           parts.push( ft=="CNA" ? "CNA " + v.fees[ft] : v.fees[ft] )
        v.feesString = parts.join(", ")

  5. STEP C — merge vendor groups that share an identical (paymentType, feesString) pair,
     in first-seen order:
     mergeMap = {} keyed by paymentType + "||" + feesString
     for each vendor group g (in the order vendorMap entries were created):
        entry = mergeMap.get(key) or create { paymentType: g.paymentType, feesString: g.feesString, labels: [] }
        entry.labels.push({ text: vendorLabel(g), isCarveout: g.vendorType=="HIH Carveout", category: vendorCategory(g.vendorType) })

     for each merged entry:
        combinedLabel = labels[0].text
        for i in 1..labels.length-1:
           prev = labels[i-1]
           combinedLabel += (prev.isCarveout AND labels[i].isCarveout) ? ", " + labels[i].text
                                                                        : "+" + labels[i].text
        mergedGroup = { paymentType, feesString, vendorLabel: combinedLabel, category: labels[0].category }

  6. STEP: order merged groups — Provider/Hospital-category groups (category "provider")
     before HIH-related groups (category "hih"); stable sort (preserves original relative
     order within each category).

  7. STEP D — decide how to render payment type:
     uniquePaymentTypes = distinct paymentType values across all merged groups
     if mergedGroups is non-empty AND uniquePaymentTypes has exactly 1 value:
        // every group shares one payment type -> lead the whole string with it once
        label = paymentLabel(mergedGroups[0].paymentType)
        groupStrs = for each group: trim(vendorLabel + " " + feesString)
        nonExclusionSegments = [ join([label, ...groupStrs], " | ") ]   // ONE segment
     else:
        // mixed payment types (or zero groups) -> each group states its own payment type
        nonExclusionSegments = for each group: trim(vendorLabel + " " + paymentLabel(paymentType) + " " + feesString)
                                // one segment PER group

  8. STEP E — exclusions:
     excludedLabels = deduplicated vendorLabel(r) for each row in `excluded`, in first-seen order
     exclusionSegment = excludedLabels.length > 0
                          ? "Not approved for " + join(excludedLabels, ", ") + "."
                          : ""

  9. perPageSegment = approvalRequiredPerPage ? "Approval required for all per page fees" : ""

  10. result = join( filter_nonempty([...nonExclusionSegments, exclusionSegment, perPageSegment]), " | " )
      return result
```

If `buildFeeText()` returns `""` (no rows entered, no exclusions, `approvalRequiredPerPage` off, `noPPT` off), the UI shows a placeholder `"Enter fee details to preview…"` in a visually distinct (amber) style — this placeholder is a UI-only affordance and is never itself part of the machine-readable output; downstream consumers should expect an empty/absent string, not this placeholder text.

### 7.2 Worked examples (test cases)

These are constructed to exercise every branch of the algorithm above; expected outputs are computed by hand-tracing the algorithm and should match the prototype's live output exactly.

**T1 — single vendor, single fee, shared payment type**
Row: `Provider/Hospital`, `Pre+Post Pay`, `Per Chart`, amount `15`
→ `"Pre+Postpay | Prov $15"`

**T2 — two vendor types merge under identical fee string**
Row A: `Provider/Hospital`, `Pre+Post Pay`, `Per Chart`, amount `15`
Row B: `General HIH`, `Pre+Post Pay`, `Per Chart`, amount `15`
→ `"Pre+Postpay | Prov+HIH $15"`

**T3 — two HIH Carveout vendors, same payment type/fee, comma-joined**
Row A: `HIH Carveout`, name `Acme`, `Post Pay Only`, `Per Chart`, amount `8`
Row B: `HIH Carveout`, name `Beta`, `Post Pay Only`, `Per Chart`, amount `8`
→ `"Postpay Only | Acme, Beta $8"`

**T4 — Per Page fee with Max $ Amount**
Row: `Provider/Hospital`, `Pre+Post Pay`, `Per Page`, amount `1`, max `25`
→ `"Pre+Postpay | Prov $1pp (max $25)"`

**T5 — Per Chart fee plus CNA fee on the same row**
Row: `General HIH`, `Post Pay Only`, `Per Chart`, amount `10`, CNA `5`
→ `"Postpay Only | HIH $10, CNA $5"`

**T6 — mixed payment types (no shared lead), category ordering preserved**
Row A: `Provider/Hospital`, `Pre+Post Pay`, `Per Chart`, amount `15`
Row B: `General HIH`, `Post Pay Only`, `Per Chart`, amount `20`
→ `"Prov Pre+Postpay $15 | HIH Postpay Only $20"`

**T7 — exclusion row appended after a normal schedule**
Row A: `Provider/Hospital`, `Pre+Post Pay`, `Per Chart`, amount `15`
Row B: `HIH Carveout`, name `Gamma`, `Exclusion`
→ `"Pre+Postpay | Prov $15 | Not approved for Gamma."`

**T8 — "approval required for all per page fees" trailing segment**
Row: `Provider/Hospital`, `Pre+Post Pay`, `Per Chart`, amount `15`; `approvalRequiredPerPage = true`
→ `"Pre+Postpay | Prov $15 | Approval required for all per page fees"`

**T9 — "No PPT fee approvals" overrides everything**
`noPPT = true` (rows and `approvalRequiredPerPage` irrelevant, even if populated)
→ `"Approval required for all PPT fees"`

**T10 — empty/blank state**
No rows with any amount or CNA fee entered, both checkboxes off
→ `""` (UI shows placeholder text only; do not confuse the placeholder with actual output)

**T11 — $0 / blank amount is silently omitted**
Row: `Provider/Hospital`, `Pre+Post Pay`, `Per Chart`, amount `0`
→ `""` (row passes field validation but contributes nothing to the text — flag to product/QA if this is not the intended behavior for production)

---

## 8. Review & Submission Flow

The prototype is a 4-state client-side machine: `form → review → confirmation`, with `help` reachable from and returning to `review`.

| State | Entered from | UI | Exits |
|---|---|---|---|
| `form` | initial load; "Go Back" from `review`; "Start a new fee schedule" from `confirmation` | Row table (or hidden if `noPPT`), both checkboxes, live fee-text preview, "Review" button | → `review` (see gating below) |
| `review` | "Review" button on `form` | Read-only rendering of every row (or "Approval required for all PPT fees." if `noPPT`) plus the final fee-text preview; buttons: Help, Go Back, Approve | → `help`, → `form` (Go Back), → `confirmation` (Approve) |
| `help` | "Help" button on `review` | Static message: submit a PPI ticket and post in `#support-switchboard-cutover` in Slack; single "Back to Review" button | → `review` only |
| `confirmation` | "Approve" button on `review` | Static success message + the submitted fee text + "Start a new fee schedule" button | → `form` (fully resets all state: `noPPT=false`, rows reset to one empty row, errors cleared) |

**Gating on "Review" click (`form → review`):**
1. If `!noPPT` and `hasDuplicateRows()` is true → block with `alert()`, stay on `form` (Section 6.3).
2. Else if `noPPT` is true, or `validateRows()` passes → advance to `review`.
3. Else → stay on `form`, re-render with inline field errors shown (Section 4).

**Approve:** transitions to `confirmation` only. **Prototype behavior — production must replace this:** there is no backend call, no persistence, no activation of a real order fee schedule, and no error path (network failure, conflict, etc.) is modeled. The confirmation text ("This fee schedule is now the active PPT fee schedule... Any previously active schedule ... has been replaced") is static copy, not a real system state change — the actual replace-prior-schedule and activation logic must be designed by the receiving team (Section 9).

**Go Back:** returns to `form` with all row data, checkbox state, and errors fully intact (no state is cleared) — this part is safe to carry over directly into production.

**Help:** navigates to a static help screen. Content is exactly: "Please submit a PPI ticket describing the issue, and post in `#support-switchboard-cutover` in Slack." No ticket is actually created by the form; it's instructional text only. "Back to Review" returns to `review` with state intact.

---

## 9. Open Questions / Follow-ups for the Receiving Team

1. **Order-submission gating.** The AC states the fee schedule is required at Order submission except when digital-only. Not modeled in the prototype at all — needs design: where is this check enforced (client-side blocking Order submit, or a backend validation), how is "digital only" determined for an order, and what happens if Client Success tries to submit an order without a fee schedule on a non-digital-only order.

2. **Versioning / "one active schedule per order, replaces prior."** No data model exists for this. Needs: a persistence layer for fee schedules per order, a concept of "active" vs. superseded schedules, and the actual replace-on-resubmit logic (soft-delete/version prior record vs. hard overwrite; audit trail requirements; whether downstream consumers need to know a schedule was replaced, or just always read "current active").

3. **Backend persistence / API contract.** The prototype has zero backend integration. Needs: where fee schedules are stored, what event/API downstream systems (e.g., HubPay) call or subscribe to in order to receive the generated fee-text (and/or the structured row data — is the *string* the source of truth downstream, or should downstream consumers also get structured JSON rather than parsing text?).

4. **RBAC / visibility enforcement.** AC specifies visibility to Client Success users with order entry/configuration access. Not represented in the prototype. Needs integration with whatever role/permission system gates Switchboard Pipelines forms.

5. **HIH Carveout vendor name as a real dropdown.** Currently free text with no validation against a known vendor list, explicitly because the structured HIH vendor list doesn't exist yet — this is pending Ivy Deng's team structuring that data (per prototype assumptions). Once available, Vendor Name for `HIH Carveout` should likely become a constrained dropdown/typeahead rather than free text, which would also change/simplify the duplicate-name matching in Section 6.2 (exact-match on a canonical ID rather than case-insensitive string compare).

6. **Fee-text formula ("Requirement 3") is unconfirmed.** The algorithm in Section 7 was reverse-engineered from sample strings and then iteratively adjusted via stakeholder feedback captured in the prototype's own "assumptions" panel — it has never been checked against a written, authoritative spec. Recommend a formal sign-off pass with the stakeholder using the worked examples in Section 7.2 (and ideally the original 3 sample strings the formula was first derived from, which are not preserved in this codebase) before building this as a hard downstream contract.

7. **Inconsistency between duplicate-guard key and text-grouping key.** As noted in Section 6.1/7.1 Step 3: the duplicate check's `rowKey` includes `feeType`, but the fee-text vendor-grouping key does not. This lets two non-duplicate rows (same vendor/payment, different fee type) silently merge in the output text. Decide whether this is intended (a vendor can have both a Per Chart and a Per Page rate, expressed as two rows) or a bug to fix before this becomes the production contract.

8. **`$0` / blank amount silently omitted (Section 4, T11).** A row with `amount = 0` passes field validation but produces no visible output. Confirm this is the intended behavior (e.g., representing "no fee" explicitly as a $0 row that shouldn't render) rather than a validation gap that should instead require a non-zero amount.

9. **Localization / currency.** `fmtAmount()` hard-codes a `$` prefix and 2-decimal formatting; there is no locale, currency code, or multi-currency support. Confirm this is acceptable for all supported markets, or scope currency handling explicitly.

10. **Duplicate-alert UX.** Both duplicate paths use blocking `alert()` dialogs (Section 6.3) — a prototyping shortcut. Production should use non-blocking inline validation/toasts, and the auto-remove-on-conflict behavior for the live single-row check should be reconsidered (silently deleting a user's just-entered row without an undo is a poor pattern for production).

11. **No live/inline validation feedback while typing.** Field errors currently only surface on clicking "Review." Confirm whether production should validate on blur/change instead.

---

## 10. Reference

The authoritative, executable specification for all UI behavior (exact field order, exact copy, exact conditional rendering, and a runnable version of the algorithm in Section 7) is the interactive prototype itself:

**`index.html`** (this folder)

It is a single self-contained HTML/JS file with no dependencies — open it directly in a browser to interact with every state described in this document. Where this written document and the prototype ever diverge, treat the prototype as ground truth for *current* behavior, and treat this document's Section 9 as the list of things the prototype deliberately does not attempt to answer.
