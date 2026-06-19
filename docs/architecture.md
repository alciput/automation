# Automation Studio — Technical Blueprint & Architecture

> A plain-language guide to how this project is built, why, and how to use it.
> All diagrams are ASCII so they render anywhere (GitHub, terminal, Confluence).

---

## 1. What this project is (in one paragraph)

**Automation Studio** lets *anyone* — PM, BA, developer, not only QA — build
automated browser tests. You describe what to test in plain English, an **AI
tool** (Cursor / ChatGPT) fills in the technical detail using prompts the app
generates for you, and the framework turns it into runnable **Playwright** tests.

The golden rule:

```
The APP owns structure, translation, and execution.
The AI only fills in details (selectors, phrases, scenarios).
The HUMAN reviews and approves.
```

This is called **human-in-the-loop AI**. The app never calls an AI API itself —
it produces copy‑paste prompts, you run them in your own AI tool, and paste the
result back. That keeps it simple, free, and safe.

---

## 2. The big picture (architecture blueprint)

```
                        ┌───────────────────────────────────────────┐
                        │                  HUMAN                     │
                        │   PM · BA · Developer  (no QA needed)      │
                        └───────────────┬───────────────────────────┘
                                        │ uses
                                        ▼
        ╔═══════════════════════════════════════════════════════════════╗
        ║                 WEB UI  (Next.js, localhost:3010)             ║
        ║                                                               ║
        ║   Home  ·  Build with AI  ·  Projects  ·  Test cases  ·  …    ║
        ║                                                               ║
        ║   • Build wizard  → generates AI PROMPTS to copy             ║
        ║   • Translate     → .feature  ➜  tests.json                  ║
        ║   • Projects      → status, stats, run commands             ║
        ║   • Test cases    → read scenarios in plain language        ║
        ╚════════════╤══════════════════════════════════╤═══════════════╝
                     │ copy prompt                       │ calls API routes
                     ▼                                   ▼
        ┌─────────────────────────┐        ┌──────────────────────────────┐
        │   EXTERNAL AI TOOL      │        │   FRAMEWORK CORE  (framework/)│
        │   Cursor / ChatGPT      │        │                              │
        │                         │        │  parser · translator ·       │
        │  writes:                │        │  matcher · executor ·        │
        │   • locators            │        │  loaders · validator         │
        │   • step-definitions    │        └───────────────┬──────────────┘
        │   • .feature scenarios  │                        │ reads / writes
        └────────────┬────────────┘                        ▼
                     │ paste back        ┌──────────────────────────────────┐
                     └──────────────────▶│   PROJECT DATA  (projects/<name>) │
                                         │                                  │
                                         │   project.json                   │
                                         │   step-definitions.yaml          │
                                         │   pages/<page>/<page>.feature    │
                                         │   pages/<page>/tests.json        │
                                         └─────────────────┬────────────────┘
                                                           │ runs
                                                           ▼
                                         ┌──────────────────────────────────┐
                                         │   PLAYWRIGHT RUNNER              │
                                         │   playwright/json-runner.spec.ts │
                                         │   → real browser → HTML report   │
                                         └──────────────────────────────────┘
```

---

## 3. The four layers explained

| Layer | Folder | Job | Who touches it |
|-------|--------|-----|----------------|
| **Web UI** | `src/` | Generate AI prompts, translate, show status | Everyone (browser) |
| **Framework core** | `framework/` | Parse, translate, match, execute | Maintainers |
| **Project data** | `projects/` | Your tests + config (the source of truth) | AI + human (review) |
| **Runner** | `playwright/` | Run JSON tests in a real browser | CLI / CI |

Supporting folders:

```
schemas/    → Zod schemas: the "contract" for every file shape
templates/  → starter .feature + step-definition examples
docs/        → this document + guides
storage/    → generated reports & screenshots (not committed)
```

---

## 4. The data model — 4 files per project

```
projects/coty/
├── project.json            (1) HOW to run + WHERE elements are
├── step-definitions.yaml   (2) PLAIN PHRASE → technical action
└── pages/
    └── homepage/
        ├── homepage.feature (3) WHAT to test (human/AI writes)
        └── tests.json       (4) Runnable steps (auto-generated)
```

**(1) `project.json`** — run settings + locators (element addresses):

```json
{
  "name": "coty",
  "run": { "baseUrl": "https://www.coty.com", "viewport": {"width":1280,"height":720} },
  "locators": [
    { "id": "hero-heading", "strategy": "text", "value": "WE ARE COTY", "page": "homepage" }
  ]
}
```

**(2) `step-definitions.yaml`** — the dictionary that maps a plain phrase to actions:

```yaml
steps:
  "user is on Coty home page":
    - action: navigate
      value: "{{baseUrl}}"
    - action: dismiss_cookies
```

**(3) `<page>.feature`** — plain-language scenario (Gherkin). No code here:

```gherkin
Scenario: Hero is visible
  Given user is on Coty home page
  Then I should see text "WE ARE COTY"
```

**(4) `tests.json`** — generated by `translate`; the runner reads this. You never
edit it by hand.

> Think of it like cooking:
> `.feature` = the menu order · `step-definitions.yaml` = the recipe ·
> `project.json` = the kitchen address book · `tests.json` = the prepared dish.

---

## 5. How data flows (the pipeline)

```
  WRITE              TRANSLATE                 RUN                  REPORT
 ┌───────┐          ┌──────────┐          ┌──────────┐         ┌──────────┐
 │ .feature│  ───▶  │ translate │  ───▶   │ Playwright│  ───▶  │  HTML    │
 │ (plain) │        │  engine   │         │  runner   │        │  report  │
 └───────┘          └────┬─────┘          └────┬─────┘         └──────────┘
                         │                      │
        reads project.json + step-definitions   reads tests.json
                         │                      │
                         ▼                      ▼
                    tests.json            real browser actions
                   (per page)             + screenshots on fail
```

**Step-by-step inside `translate`:**

```
  .feature scenario line
        │
        ▼
  "user is on Coty home page"
        │  1. look up phrase in step-definitions.yaml
        ▼
  [ navigate {{baseUrl}}, dismiss_cookies ]
        │  2. replace variables ({{baseUrl}}) from project.json
        ▼
  [ navigate https://www.coty.com, dismiss_cookies ]
        │  3. validate against schema (Zod)
        ▼
  tests.json  ✅
```

If a phrase has no recipe, translate fails clearly:
`No step definition for: "..."` → fix the YAML (the Build wizard has an AI prompt for this).

---

## 6. How to USE it — for non‑QA (the easy path)

```
  ┌─────────────────────────────────────────────────────────────┐
  │  npm run dev      →     open  http://localhost:3010          │
  └─────────────────────────────────────────────────────────────┘
         │
         ▼
  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │ 1. Home      │──▶│ 2. Build w/AI │──▶│ 3. Translate │──▶│ 4. Projects  │
  │  overview &  │   │  copy prompt  │   │  .feature ➜  │   │  copy run    │
  │  start CTA   │   │  → paste in   │   │  tests.json  │   │  command →   │
  │              │   │  Cursor/GPT   │   │              │   │  terminal    │
  └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
```

**The 7-step Build wizard** (each step gives you a prompt to copy):

```
 [1] Create project ─ folders + base URL          (terminal commands)
 [2] Collect selectors ─ AI finds element locators (AI prompt) ★
 [3] Step definitions ─ AI maps phrases→actions    (AI prompt) ★
 [4] Test cases ─ AI writes .feature scenarios     (AI prompt) ★
 [5] Translate & validate ─ make tests.json        (button, no AI)
 [6] Run tests ─ Playwright + report               (terminal commands)
 [7] Fix failures ─ AI suggests locator/step fixes (AI prompt) ★

   ★ = "AI discovers" mode: you only paste the website URL,
       the AI explores the site and writes the details itself.
```

---

## 7. How to USE it — the CLI (for running tests)

```
 npm run list                         # show projects & pages
 npm run validate -- --project coty   # check all files are valid
 npm run translate -- --project coty  # .feature ➜ tests.json (all pages)
 npm run test -- --project coty       # run in Playwright
 npm run report -- --project coty     # open the HTML report
```

Narrow a run:

```
 npm run test -- --project coty --page homepage
 npm run test -- --project coty --tag smoke
```

What `npm run test` does under the hood:

```
 cli.ts test ──▶ sets env TEST_PROJECT/TEST_PAGE/TEST_TAG
            ──▶ spawns: npx playwright test
            ──▶ json-runner.spec.ts loops projects/pages/test cases
            ──▶ executor.ts performs each step in the browser
            ──▶ report saved to storage/reports/<project>/html
```

---

## 8. Web app route map

```
 src/app/(dashboard)/
 ├── layout.tsx        ── global top nav (shared by all pages)
 ├── page.tsx          ──  /            Home (hero + project cards)
 ├── build/            ──  /build       Build wizard (AI prompts)   ★ hero
 ├── projects/         ──  /projects    Status, stats, run commands
 ├── test-cases/       ──  /test-cases  Read scenarios, no JSON
 └── translate/        ──  /translate   .feature ➜ tests.json + validate

 src/app/api/          ── server endpoints used by the UI
 ├── translate/        ── POST: run translation, save tests.json
 ├── validate/         ── GET : validate a project
 ├── feature/          ── GET : load .feature + tests.json from disk
 ├── projects/         ── GET : project summaries (stats)
 ├── test-cases/       ── GET : scenarios for a page
 └── build/context/    ── GET : auto-fill wizard with project data
```

---

## 9. Framework module map (`framework/`)

```
 cli.ts                 → command entry (list/validate/translate/test/report)
 gherkin-parser.ts      → reads .feature text into scenarios
 gherkin-translator.ts  → scenarios → tests.json (the pipeline brain)
 step-definitions-loader→ loads step-definitions.yaml
 step-matcher.ts        → matches a phrase to a recipe (+ {params})
 variables.ts           → fills {{baseUrl}} and {params}
 project-loader.ts      → loads + validates project.json
 page-loader.ts         → discovers/loads tests.json per page
 locator-resolver.ts    → turns a locator id into a Playwright locator
 executor.ts            → runs steps & assertions in the browser
 validator.ts           → checks all files match the schemas
 paths.ts               → where everything lives on disk
 playwright-artifacts.ts→ report paths + failure screenshots
```

Call order during a test run:

```
 json-runner.spec.ts
     └─▶ paths.listProjects()        which projects exist
     └─▶ project-loader.loadProject  config + locators
     └─▶ page-loader.loadPageTests   tests.json
     └─▶ executor.runTestCase
             └─▶ locator-resolver    id → element
             └─▶ Playwright actions  click/fill/expect…
             └─▶ on fail: playwright-artifacts.attachFailureScreenshot
```

---

## 10. Roles — who does what

```
 ┌─────────────┬───────────────────────────────────────────────┐
 │ PM / BA     │ Write .feature scenarios (Build step 4),       │
 │             │ review Test cases page                         │
 ├─────────────┼───────────────────────────────────────────────┤
 │ Developer   │ Run Build steps 2–3 (locators, step-defs),     │
 │             │ fix failures (step 7)                          │
 ├─────────────┼───────────────────────────────────────────────┤
 │ Anyone      │ Translate (step 5), run tests & read report    │
 │             │ (step 6) via Projects page commands            │
 └─────────────┴───────────────────────────────────────────────┘
```

---

## 11. Glossary (plain language)

| Term | Means |
|------|-------|
| **Gherkin / `.feature`** | Plain "Given/When/Then" sentences describing a test |
| **Locator** | A stable address for a button/link/text on the page |
| **Step definition** | The recipe: one phrase → one or more browser actions |
| **Translate** | Convert `.feature` → `tests.json` (runnable form) |
| **tests.json** | The generated, machine-readable test (never hand-edit) |
| **Autonomous / "AI discovers"** | Wizard mode where AI explores the site from just a URL |
| **Human-in-the-loop** | App makes prompts; you run AI; you approve the result |

---

## 12. One-page cheat sheet

```
 BUILD (once per site)         RUN (every time)
 ─────────────────────         ────────────────
 1. /build → copy prompts      npm run translate -- --project X
 2. paste into Cursor/ChatGPT  npm run test      -- --project X
 3. save files into projects/X npm run report    -- --project X
 4. npm run validate -- -p X   (read pass/fail in the browser)
```
