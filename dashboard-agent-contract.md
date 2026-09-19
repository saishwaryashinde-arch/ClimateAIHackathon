# Dashboard <> Agent Contract

Status: draft for team sync · Last updated: 2026-09-19
Dashboard owner: Jillian · Agent owner: [name]

## Principle
The dashboard decides. The agent explains.
Only `evaluate()` makes eligibility decisions and totals. The agent never recomputes them; it quotes what the dashboard publishes.

## The context object

```js
window.dashboardContext = {
  batch: {
    items,              // number, all records in the loaded batch
    eligibleKg,         // number, after estimated-weight buffer
    modeledCredit,      // number, dollars at the demo rate
    rejectedOrReview    // number
  },
  reasonCounts: {       // full batch, not the filtered view
    REJECTED_OUT_OF_STATE: 0,
    REJECTED_NON_QUALIFYING_DISPOSITION: 0,
    GEO_AMBIGUOUS: 0
  },
  reasonStats: {        // one entry per rejection code, published from evaluated records
    GEO_AMBIGUOUS: {count: 0, totalKg: 0, modeledDollars: 0, recoverable: true}
    // out-of-state and nonqualifying disposition use recoverable: false
  },
  dataQuality: {       // full batch
    estimatedWeightCount: 0,
    missingEvidenceCount: 0, // eligible records without an evidence ID
    missingOriginCount: 0
  },
  activeFilter: "all",  // "all" | "eligible" | "rejected"
  selectedItem: null,   // or:
  // { id, category, donorState, disposition, weightG, weightIsEstimated,
  //   decision, reasonCode, failedRules: [] }
  selectedReason: null  // or a reason code, set when a sidebar reason is clicked
};
```

Field names match the CSV export. If the export uses a different name for a field, the export name wins.

## Update events
The dashboard dispatches `dashboard:context` on `window` after any of these:
- batch loaded or replaced
- filter changed
- row selected or drawer closed
- sidebar rejection reason clicked or cleared

The agent listens for the event and reads `window.dashboardContext`. No polling.

```js
window.addEventListener('dashboard:context', () => { /* agent reads context */ });
```

## What the agent may do
- Explain why the selected item got its decision, using `reasonCode` and `failedRules`
- Say how many items share a reason, using `reasonCounts`
- Suggest a next step for a reason group (see advice map)
- Estimate modeled credit for a group as `weight x demo rate`, labeled "modeled" and "illustrative"

## What the agent may not do
- Decide or change eligibility
- State a number that is not in the context object or derived from it
- Promise recovered dollars for a group that cannot be recovered
- Cite statute sections or regulator rules unless the team has verified them
- Present itself as legal advice

If a field is `null` or a question is out of scope, the agent uses the fallback wording:
"[Fallback wording agreed at sync]"

## Advice map (fill in at sync)

| Reason | Recoverable per item? | Advice direction |
|---|---|---|
| REJECTED_OUT_OF_STATE | No | Intake routing: capture donor origin earlier, accept only qualifying-origin bags |
| REJECTED_NON_QUALIFYING_DISPOSITION | No (past decision) | Upstream sorting and partner disposition options |
| GEO_AMBIGUOUS | Yes, if origin is supplied | Capture origin at intake, re-run the record |
| [other] | | |

## Canonical wording
- Product name: [ThreadTrace.io or TraceBack, decide at sync]
- Chatbot type: [rule-based / live model], stated honestly in the UI
- Disclaimer: [single approved text]

## Open decisions
| Decision | Options | Owner | Resolve by |
|---|---|---|---|
| Agent reacts to row click | Auto-message vs. "Ask about TB-1010" chip | | |
| Agent scope with filter active | Full batch vs. filtered view | | |
| Modeled dollar estimates | Allowed vs. counts only | | |

## Current audit-guide prototype

The guide uses the full loaded batch, including when the intake table is filtered. The dashboard publishes `dashboard:context` after batch replacement, filter changes, row selection, and drawer close. The guide reads that object and does not call `evaluate()`.

The published context extends the draft shape with `batch.version`, `batch.eligibleItems`, `quality`, `reasonGroups`, `reasonStats`, and `dataQuality`. `reasonStats.totalKg` is raw applied weight, including category benchmarks where needed; `modeledDollars` is that weight × the demo rate. It is not earned credit or a promise of recovery. Only origin-unclear records have `recoverable: true`, conditional on source proof of California origin and the other checks. `reasonGroups` retains buffered conditional-credit and example-ID details. The dashboard derives these fields from evaluated records; the guide only explains them.

The guide treats `GEO_AMBIGUOUS` as reviewable when original California origin is supplied. Out-of-state donor and past nonqualifying-disposition decisions are not recoverable by relabeling. Weight-only dollar illustrations are never presented as earned credit. Evidence IDs in the synthetic sample are mock references rather than verified proof.
