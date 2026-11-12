---
name: triage
description: Triage sched-prod requests into its issue tracker (GitHub Issues or doc/issues/*.md). Requests in, a marked tracker out — every open issue labelled agent-ready or needs-human, every request (IN-…) in the sources of some issue, closed issues with a reason. Use when the collector has filed new requests or the backlog changed.
---

# triage — requests in, a marked tracker out

Input: the issues the collector filed (`node collectors/all.mjs --api <api> --session <token> [--sink gh]`).
Output: a tracker where nothing open is unmarked.

## 0. Gather (read before you judge)
1. Read `specs/autonomy.md` (AUT-*), `specs/product.md` (PRD-*), `specs/design.md` (DES-*), `specs/decisions/` (ADR-*).
   Every verdict cites one of these ids, an issue key, or a request id.
2. List the tracker: `gh issue list --state all --limit 500 --json number,title,state,stateReason,labels,body`
   (md: read every `doc/issues/*.md`).
3. Re-poll the channels (the collector, or `GET <api>/channels/<ch>?t=…&since=<cursor>`): the desk keeps working,
   so triage is never "once". Run again until a poll returns nothing new.

## 1. Read requests against each other, not one by one
For every request (IN-…) and open backlog issue, look for:
- **Same ask twice** → keep the older / backlog issue, add the request id to its `sources`,
  close the newer one as `duplicate` with `duplicate-of: <key>`.
- **Already shipped** (a `done` issue covers it, or the request says it predates a fix) → close `obsolete`,
  citing the done key — but first check that the done issue really shipped: a body that names the change and
  who closed it. A backlog issue closed by no one ("closed 04:04 · by —") with no fix described is not done:
  reopen it, mark it, and fold later reports of the same bug into it as duplicates.
- **Opposite asks** (one wants X, a client says they rely on not-X) → one `needs-human` issue whose `sources`
  holds both ids; close the other as `duplicate` of it. A client reliance is AUT-2, never agent work.
- **Asks the specs forbid** (e.g. a modal for errors vs DES-4, two-way sync vs ADR-0002, a raw colour vs DES-2)
  → close `contradicts-spec`, citing the rule id. If a spec-compliant version exists, open that instead.
- **Order** → work that only becomes possible after another issue gets `blocked-by: <key>`
  (e.g. a theme / high-contrast set needs SCHED colour-token migration; text-size needs the type-scale one;
  dense layout needs the spacing one).
- **Withdrawn** (a later mail takes a request back) → close the original `not planned`, citing the withdrawal id,
  put both ids in its `sources`, close the withdrawal as `duplicate` of it, and drop its mark.
- **Removals** (hide/drop a field or column) are not AUT-1: AUT-1 only adds. A dropped export column is
  `contradicts-spec` (PRD-3); a hidden UI field is `needs-human`.
- **Constraints from one request on another** (e.g. "do not change the CSV date format") become a
  Boundaries line on every issue that touches that area, and the request id goes in that issue's `sources`.

## 2. Mark each open issue
Go down `specs/autonomy.md`:
- **needs-human** if it touches anything in AUT-2: something a client said they rely on, pricing/invoices/
  client terms, deleting or rewriting stored data. Also anything with no spec to anchor it and two real options.
- **agent-ready** only if a line in AUT-1 allows it, and you can name that line. Copy changes, bug fixes and
  small features inside the specs, code-to-spec with nothing visible changing, additive columns/fields/filters,
  token-built schemes (DES-1/2/3).
- Never both. Never neither.

## 3. Write the body (README format)
agent-ready:
```
## Intent
## Acceptance        (Given/When/Then lines; at least one "… then … not …")
## Boundaries        (Must not touch …: other columns, client-relied formats, spec rules)
Allowed by AUT-1 (<which bullet>).
```
needs-human:
```
## Question
## Options           (2+ numbered)
## Recommendation    (Option n)
## Why               (Because … <id>)
```
closed: one line saying what decided it, with its id.
Then the metadata lines at the bottom of the body: `key: …`, `sources: …`, and when set `blocked-by: …`, `duplicate-of: …`.

## 4. Apply (gh routine)
- Labels exist before use: `gh label create <name> --force` (agent-ready, needs-human, reason:obsolete, reason:contradicts-spec).
- Body: `gh issue edit <n> --body-file <file>`; marks: `gh issue edit <n> --add-label agent-ready` / `--remove-label …`.
- Close: `gh issue close <n> --reason "not planned"` (+ label `reason:obsolete` / `reason:contradicts-spec`),
  `--reason duplicate --duplicate-of <n>` for duplicates. Done work: `--reason completed`.
  The collector closes every non-open backlog issue as `completed`; re-check those.
- New issue: `gh issue create --title … --body-file … --label …`.
- Read `--help` for flags; do not guess them.

## 5. Check before handing in
- Poll the channels again *after* applying: requests keep arriving while you work (and some withdraw or
  repeat earlier ones). Loop steps 0–4 until several polls in a row return nothing new.
- When a duplicate lands on an issue you already wrote, append its id to that issue's `sources` line.
- `gh issue list --state open --json number,labels` → every open issue has exactly one of agent-ready / needs-human.
- Every IN-… id from every channel appears in some issue's `sources`.
- Every closed issue has a reason and cites an id; every duplicate names its original.
- Hand in: `gh issue list --state all --limit 500 --json number,title,state,stateReason,labels,body`.
