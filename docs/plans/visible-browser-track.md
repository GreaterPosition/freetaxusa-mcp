# Plan: visible-browser track

Status: **approved for build**. Integration branch: `visible-browser-track`.
This document is public and must stay generic. No taxpayer names, numbers, paths or account details belong here.

## 1. Goal

Add a second, opt-in way to run this server:

- a **visible Chrome window** that the user logs into by hand,
- so the agent **never receives credentials**,
- the user can **take over the window at any time**,
- and the server can **never e-file, sign or pay**.

The existing `embedded` (email/password) and `hermes` (cookie broker) paths stay intact and behave exactly as today.

## 2. Non-goals

- Removing or rewriting upstream auth paths or tools. Upstream behaviour is the default.
- Submitting a return, extension or payment. These are blocked in code (section 6).
- Committing any taxpayer data. Return data, logs, snapshots and screenshots live outside the repo, in directories set
  by environment variables.

## 3. Current state (upstream, at `7746882`)

| Area | File | Notes |
|---|---|---|
| Entry | `src/index.ts` | stdio transport |
| Tools registry | `src/server.ts` | `server.tool(...)` (SDK `^1.12`), `wrapHandler` applies `filterPII` to all output |
| Browser | `src/browser/context.ts` | Singleton persistent Chromium, headless by default, `acquirePageLock` mutex, `BASE_URL`/`AUTH_URL` |
| Navigation | `src/browser/navigation.ts`, `src/types/sections.ts` | SID discovery from sidebar `a[href*="sid="]`, fallback SID table |
| Forms | `src/browser/forms.ts` | `readFormFields`, `fillFieldByLabel`, `selectByLabel`, `clickSaveAndContinue` |
| Auth | `src/tools/session.ts`, `src/hermes-client.ts` | `authenticate` (embedded or Hermes), `get_session_status` |
| Working tools | `page.ts`, `personal.ts`, `overview.ts` | read page, save & continue, navigate, taxpayer info, filing status, summary, refund |
| Stubs | `income.ts`, `deductions.ts`, `review.ts`, `filing.ts` | return `not_implemented` |
| PII | `src/security/pii-filter.ts` | SSN / account masking, the only unit test |

Known gaps:

- No generic fill/click tools.
- `readFormFields` includes `input[type=password]` values. They must be excluded.
- There are no field ids, so labels are the only handle.

## 4. Mode selection

`FREETAXUSA_AUTH_MODE = embedded | hermes | visible`. When unset, current behaviour applies: Hermes if configured,
otherwise embedded.

| | `visible` (new) | `embedded` / `hermes` (existing) |
|---|---|---|
| Browser | Installed Google Chrome (`channel: "chrome"`), always headed, maximized, `viewport: null` | Bundled Chromium, `FREETAXUSA_HEADLESS` |
| Launch flags | `ignoreDefaultArgs: ["--enable-automation"]`, `--disable-blink-features=AutomationControlled`, `--start-maximized` (a human logs in and watches) | unchanged |
| Login | `open_login` → user logs in, including MFA → `wait_for_login` polls for an authenticated page | `authenticate` |
| `authenticate` tool | **Not registered** | unchanged |
| Profile dir | `FREETAXUSA_USER_DATA_DIR` or `~/.freetaxusa-mcp/browser-profile-visible/` (0700) | unchanged |
| Signal handling | `handleSIGINT/SIGTERM/SIGHUP: false`; server closes the context itself | unchanged |
| Window closed by user | Session dropped; next call says to run `open_login` | n/a |

Reference implementation: `tax-docs-fetch-mcp`, in `src/browser/context.ts` (`SessionManager`,
`VISIBLE_BROWSER_OPTIONS`) and `src/tools/session.ts` (`open_login`, `wait_for_login`, `close_window`). **Copy and
adapt it. Do not add a dependency on that package.** Use `async-mutex` or the existing lock.

Authenticated-page check: the URL is on `www.freetaxusa.com/taxes<YEAR>/` and not `auth.freetaxusa.com`, and the
sidebar nav (`a[href*="sid="]`) is present.

## 5. Target file layout (new or changed)

```
src/
  config.ts                 # NEW  read all env vars once (mode, dirs, year), validated with zod
  browser/
    context.ts              # CHANGED  delegate to a mode-specific launcher; keep exports stable
    visible-session.ts      # NEW  visible SessionManager (copied/adapted)
    guard.ts                # NEW  e-file / submit / pay guard. Single choke point
    takeover.ts             # NEW  user-input detection, pause/resume state
    actions.ts              # NEW  primitives implementation (fill/select/check/radio/click/screenshot)
    forms.ts                # CHANGED  exclude password fields; return stable field keys + page errors
  audit/
    action-log.ts           # NEW  JSONL action log (PII-masked) + screenshot paths
  returns/
    schema.ts               # NEW  zod schema for a return data file (versioned)
    plan.ts                 # NEW  data file -> ordered FillAction[] (pure, no browser)
    apply.ts                # NEW  executes actions page by page with read-back verification
  sections/
    index.ts                # NEW  registry: section id -> SectionMap
    income.ts               # NEW  W-2, 1099-INT/DIV/B/MISC/NEC/G, 1099-DA/crypto
    business.ts             # NEW  Schedule C (multiple), Form 8829, Form 4562 items, Schedule E
    deductions.ts           # NEW  itemized (Sch A), QBI, Form 2441, estimated payments
    state.ts                # NEW  state return entries (PA first; generic shape)
  verify/
    return-pdf.ts           # NEW  download filed PDF, extract line items
    compare.ts              # NEW  compare extracted lines vs data file -> diff report
  tools/
    session.ts              # CHANGED  register open_login/wait_for_login/close_window in visible mode; authenticate otherwise
    actions.ts              # NEW  MCP tools for primitives + pause/resume
    fill.ts                 # NEW  plan_fill / apply_fill / snapshot_page
    verify.ts               # NEW  download_return_pdf / verify_return
tests/unit/
  guard.test.ts  takeover.test.ts  plan.test.ts  apply.test.ts  schema.test.ts  sections.*.test.ts  compare.test.ts
tests/fixtures/
  pages/*.html              # SYNTHETIC or fully sanitized page snapshots only
  returns/sample-return.json  # fictional taxpayer
```

## 6. E-file guard (stream B), non-negotiable

- Every click, navigation, form submit and keyboard Enter issued by the server goes through `guard.assertAllowed(target)`.
- **Blocked by accessible name or text** (case-insensitive regex list in `guard.ts`): `file my return`, `e-?file`,
  `submit (my )?return`, `transmit`, `sign (and|&) file`, `pay(ment)? (now|today)`, `pay (with|by)`,
  `submit payment`, `authorize`, `i agree.*(file|submit)`, `place order`, `purchase`, `upgrade`, `deluxe`,
  `add (state|audit)`.
- **Blocked by URL**: the final filing section. Resolve it by sidebar name ("Filing", "File", "Final Steps"), plus
  any URL matching `/(efile|e-file|submit|payment|checkout|purchase)/i`.
- **Blocked by page content**: any page whose heading matches `/(sign|file|submit) (your|my) return|payment/i`. This
  covers all actions on that page except `read_current_page` and `screenshot`.
- A blocked action throws `EFILE_BLOCKED`. The tool returns `{ success: false, error: "efile_blocked",
  message: "This step must be done by the user in the browser window." }`.
- There is no override flag, env var or tool parameter. Tests must cover every pattern and a set of near-miss
  allowed labels ("Save and Continue", "Continue", "Add a W-2", "Edit").

## 7. Takeover (stream C)

- On every page load, `context.addInitScript` registers `keydown`, `mousedown` and `wheel` listeners. Ignore events
  where `isTrusted === false`, since those come from the agent. The listeners set `window.__userActiveAt = Date.now()`.
- Before each action, read `__userActiveAt`. If it changed since the last agent action, set state to `paused_by_user`
  and return that without acting.
- Tools: `pause` and `resume`. While paused, read-only tools (`read_current_page`, `screenshot`,
  `get_session_status`) still work.

## 8. Primitives (stream C)

All primitives take `target: { label?: string; name?: string; role?: string; index?: number }`, resolved with
Playwright `getByLabel` / `getByRole` (exact match first, then case-insensitive). Every call returns
`{ success, before, after, pageTitle, sid, errors[] }`.

`fill_field(target, value)`, `select_option(target, option)`, `set_checkbox(target, checked)`,
`choose_radio(groupLabel, option)`, `click(target)` (guarded), `screenshot()` (saved to `FREETAXUSA_LOG_DIR`, path
returned), `snapshot_page()` (sanitized HTML to `FREETAXUSA_SNAPSHOT_DIR`; strips input values, SSN/EIN/account
patterns, names in the header).

## 9. Action log (stream C)

`FREETAXUSA_LOG_DIR/<session-start-iso>.jsonl`. Each line is:

`{ ts, tool, mode, sid, pageTitle, target, before, after, result, screenshot? }`

`before` and `after` pass through `filterPII`. The log is never written inside the repo. The server refuses to start
in visible mode if `FREETAXUSA_LOG_DIR` resolves inside the repo.

## 10. Return data file (stream D)

`returns/schema.ts` uses zod with `schemaVersion: 1`. The top-level shape is:

```ts
{
  schemaVersion: 1, taxYear: number,
  filingStatus: "single"|"married_joint"|"married_separate"|"head_of_household"|"qualifying_surviving_spouse",
  taxpayers: { primary: Person, spouse?: Person }, dependents: Person[],
  w2: W2[], forms1099: { int: F1099Int[], div: F1099Div[], b: F1099B[], misc: F1099Misc[], nec: F1099Nec[],
                         g: F1099G[], da: F1099Da[] },
  scheduleC: Array<{ owner: "primary"|"spouse", name, ein?, address, businessCode, accountingMethod: "cash",
                     grossReceipts, expenses: Record<ScheduleCLine, number>, otherExpenses: {desc, amount}[],
                     assets: Asset[], homeOffice?: Form8829 }>,
  scheduleE: Array<{ address, type, daysRented, rents, expenses: Record<ScheduleELine, number>, depreciation }>,
  itemized?: { mortgageInterest: {lender, amount}[], realEstateTaxes, stateLocalIncomeTax, other: {desc, amount}[] },
  credits: { dependentCare?: {providers: Provider[], expenses, employerBenefits} },
  estimatedPayments: { federal: Payment[], state: Payment[] },
  state: Array<{ code: string, residency: "full"|"part"|"non", entries: Record<string, unknown> }>,
  sources: Record<string /* json-pointer */, string /* source document name */>
}
```

- Money values are numbers in dollars with two decimals. Every leaf value should have a `sources` entry so
  `verify_return` can cite the source document.
- `plan.ts` is pure. It turns the data into `FillAction[]` using `sections/*`, ordered by FreeTaxUSA's flow. Each
  action is `{ sectionId, pageMatcher, steps: Step[], readBack: Check[] }`.
- `apply.ts` navigates to the section, matches the page by title or heading, runs the steps through the primitives
  (and so through the guard and takeover check), reads back every value, then runs `save_and_continue`. It stops on
  the first mismatch, validation error, guard block or pause. It can resume from an action index.
- Tools:
  - `plan_fill(dataFile)` is a dry run. It returns an action list plus counts.
  - `apply_fill(dataFile, { section?, fromIndex?, maxPages? })`.

## 11. Section maps (streams E1–E3)

- Each section exports a `SectionMap` with:
  - `id`
  - `entry` (how to reach it: sidebar name or SID)
  - `pages` (title matcher → field label map)
  - `repeat` (how to add another W-2 or 1099)
  - `toActions(data)`
- Build and test them against **fixtures in `tests/fixtures/pages/`**, which must be synthetic or fully sanitized.
  Real snapshots captured by the user stay in `FREETAXUSA_SNAPSHOT_DIR`, outside the repo. Developers read them
  locally and hand-write synthetic fixtures from them.
- Expect FreeTaxUSA's interview to branch: yes/no gates before detail pages. Model gates as steps.

## 12. Post-file verification (stream F)

- `download_return_pdf` is used after the user has e-filed. It navigates to the account's prior-return or print page
  and saves the PDF to `FREETAXUSA_RETURNS_DIR`. It's read-only and passes the guard.
- `verify_return(dataFile, pdfPath)` extracts form lines (1040, Schedules 1/2/3/A/B/C/D/E/SE, 8829, 4562, 8995, 2441,
  state) with a PDF text extractor such as `pdfjs-dist` or `unpdf`. It compares them to the data file and to derived
  totals. It returns `{ matched[], mismatched[{line, expected, actual, source}], missing[] }`.

## 13. Work streams and order

| Stream | Scope | Starts | Merges after |
|---|---|---|---|
| A | `config.ts`, visible session, mode-aware tool registration, password-field exclusion | immediately | first |
| B | `guard.ts` + exhaustive tests | immediately | A |
| D | `returns/schema.ts`, `plan.ts`, `apply.ts` against a fake page interface | immediately | A |
| F | `verify/*` + tests with a synthetic PDF | immediately | D (schema) |
| C | primitives, takeover, action log, `snapshot_page` | after A's interfaces land | A, B |
| E1/E2/E3 | section maps for income, business and deductions/credits/state | after D's schema and the first user snapshots | C, D |
| G | docs (README, CLAUDE.md, this plan, PROJECT_STRUCTURE.md), CI scripts, final security review | continuous; final pass last | all |

Git workflow:

- Each stream works on `feature/<stream>-<topic>` in a worktree under `../freetaxusa-mcp.worktrees/`, branched from
  `visible-browser-track`.
- Each stream merges back with `--no-ff` after `npm run build && npm test` pass.
- Use Conventional Commits.
- Do not push to `main` and do not open PRs upstream without the user's go-ahead.

## 14. Live-site rules

1. **Agents never log in** and never type credentials. Live work needs the user at the keyboard.
2. Snapshot capture: the user logs in through `open_login`, then the agent walks the sidebar and calls
   `snapshot_page` on each page.
3. The first real fill is `plan_fill`, then `apply_fill` one section at a time with the user watching.
4. Filing, signing and paying are done only by the user.

## 15. Done when

- `npm run build && npm test` are green.
- Guard tests cover every blocked pattern and the allowed near-misses.
- Visible mode on a real machine:
  - login handoff works
  - user takeover pauses the agent and resume continues it
  - `plan_fill` on the sample return works
  - `apply_fill` on one income page works with read-back
- Upstream modes behave exactly as before (existing test plus a smoke test of tool registration per mode).
- Docs are updated. No personal data is in the repo (`git grep` checks for SSN/EIN/currency patterns pass).
