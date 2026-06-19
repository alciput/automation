# Locators, Selectors & Complex Test Cases

A simple guide for anyone building tests with this framework (PM, BA, developers).

**Start here if you are new:** [build-automation-from-scratch.md](./build-automation-from-scratch.md) — full workflow and AI prompts.

---

## The big picture

You have **4 files** that work together:

```
.feature              →  What to test (QA writes)
step-definitions.yaml →  How to automate (engineer writes)
project.json          →  Where elements are on the page (engineer writes)
tests.json            →  Output for Playwright (auto-generated)
```

**Translate does not find selectors.** It only converts Gherkin text into JSON using rules you already defined.

```
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────┐
│  QA: .feature   │     │  Engineer: step-defs │     │  tests.json │
│  human language │ ──► │  + project.json      │ ──► │  for runner │
└─────────────────┘     └──────────────────────┘     └─────────────┘
```

---

## Who does what?

| Role | File | Example |
|------|------|---------|
| **PM / BA / Developer** | `pages/login/login.feature` | `Given user is on login page` |
| **Developer (or AI)** | `step-definitions.yaml` | Maps that phrase → `navigate`, `click`, `fill` |
| **Developer (or AI)** | `project.json` → `locators` | `email-input` → `getByLabel('Email')` |
| **Tool** | `tests.json` | Generated when you click **Translate** |

Test authors never write CSS, XPath, or `#id` selectors in `.feature` files.

---

## Where do selectors live?

All real selectors are in **`project.json`** under `locators`:

```json
{
  "name": "coty",
  "run": { "baseUrl": "https://www.coty.com" },
  "locators": [
    {
      "id": "hero-heading",
      "strategy": "text",
      "value": "WE ARE COTY",
      "page": "homepage"
    },
    {
      "id": "footer-contact",
      "strategy": "text",
      "value": "Contact Us",
      "page": "homepage"
    }
  ]
}
```

### Locator fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Short name used in step definitions (e.g. `email-input`) |
| `strategy` | Yes | How Playwright finds the element (see table below) |
| `value` | Yes | The selector value |
| `page` | No | Which page this belongs to (for documentation) |

### Supported strategies

| Strategy | When to use | Example `value` |
|----------|-------------|-----------------|
| `role` | Buttons, links, headings | `link[name='Our Brands']` |
| `label` | Form fields with labels | `Email address` |
| `testid` | App has `data-testid` | `login-submit` |
| `text` | Visible text on page | `Contact Us` |
| `css` | Last resort | `.btn-primary` |
| `xpath` | Last resort | `//button[@type='submit']` |

**Best order:** `role` → `label` → `testid` → `text` → `css` / `xpath`

---

## How step definitions use locators

In **`step-definitions.yaml`**, you reference locator **ids**, not raw selectors:

```yaml
steps:
  "user is on Coty home page":
    - action: navigate
      value: "{{baseUrl}}"

assertions:
  "I should see the Coty hero section":
    - type: visible
      target: hero-heading        # ← id from project.json
    - type: text
      target: hero-heading
      expected: "WE ARE COTY"
```

### Gherkin → JSON example

**QA writes** (`homepage.feature`):

```gherkin
@TC-COTY-001
Scenario: Homepage loads with hero section
  Given user is on Coty home page
  Then I should see the Coty hero section
```

**After translate** (`tests.json`):

```json
{
  "steps": [
    { "action": "navigate", "value": "https://www.coty.com" }
  ],
  "assertions": [
    { "type": "visible", "target": "hero-heading" },
    { "type": "text", "target": "hero-heading", "expected": "WE ARE COTY" }
  ]
}
```

**At run time**, `hero-heading` becomes `page.getByText('WE ARE COTY')`.

---

## Complex test cases (multi-field forms)

One Gherkin line can become many Playwright steps.

**QA writes one line:**

```gherkin
Then user fill signup form with "<firstname>" "<lastname>" "<email>" "<password>"
```

**Engineer defines the mapping once** (`step-definitions.yaml`):

```yaml
"user fill signup form with {firstname} {lastname} {email} {password}":
  - action: fill
    target: firstname-input
    value: "{firstname}"
  - action: fill
    target: lastname-input
    value: "{lastname}"
  - action: fill
    target: email-input
    value: "{email}"
  - action: fill
    target: password-input
    value: "{password}"
```

**Engineer registers all fields** (`project.json`):

```json
"locators": [
  { "id": "firstname-input", "strategy": "label", "value": "First name", "page": "signup" },
  { "id": "lastname-input",  "strategy": "label", "value": "Last name",  "page": "signup" },
  { "id": "email-input",     "strategy": "label", "value": "Email",      "page": "signup" },
  { "id": "password-input",  "strategy": "label", "value": "Password",   "page": "signup" }
]
```

QA keeps writing simple Gherkin. Engineers maintain locators and step definitions.

---

## How to find selectors (discovery)

Translate **cannot** discover selectors. Use these tools **once per element**, then add to `project.json`.

### Option 1: Playwright Codegen (recommended)

```bash
npx playwright codegen https://www.coty.com
```

1. Browser opens with a recorder.
2. Click elements on the page.
3. Copy generated locators (prefer `getByRole`, `getByLabel`).
4. Add them to `project.json` with a clear `id`.

### Option 2: Playwright HTML report (after a failed test)

```bash
npm run test -- --project coty
npm run report -- --project coty
```

Open the report → failed step → trace → inspect element.

### Option 3: Browser DevTools

1. Right-click element → Inspect.
2. Prefer accessible attributes (`aria-label`, `name`, role).
3. Avoid long CSS paths when possible.

---

## Step-by-step: add a new complex test

### 1. Discover locators

Run codegen on your site. Collect ids for every field and button.

### 2. Update `project.json`

Add all locators to the `locators` array.

### 3. Update `step-definitions.yaml`

Add phrases for new Given/When/Then steps. Reference locator ids in `target`.

### 4. Write `.feature` file

QA writes Gherkin under `projects/<name>/pages/<page>/`.

### 5. Translate

**Web UI:** open dashboard → pick project & page → **Translate**

**CLI:**

```bash
npm run translate -- --project coty --page signup
```

### 6. Run tests

```bash
npm run test -- --project coty --page signup
npm run report -- --project coty
```

---

## Folder structure (reminder)

```
projects/coty/
  project.json                 # baseUrl + locators + variables
  step-definitions.yaml        # Gherkin phrase → actions
  pages/
    homepage/
      homepage.feature         # QA source
      tests.json               # generated
    signup/
      signup.feature
      tests.json
```

Only **top-level** folders under `projects/` are runnable (`coty`, `supplier`, `my-site`). Do **not** create `projects/my-site/coty/` or nest one project inside another — the runner ignores those paths.

---

## Common questions

### Does translate find selectors automatically?

**No.** Translate only maps Gherkin to JSON using `step-definitions.yaml` and checks structure. Selectors must already exist in `project.json`.

### Do I put selectors in the `.feature` file?

**No.** QA uses human phrases. Engineers put selectors in `project.json`.

### Can I use raw CSS in JSON?

Yes, as a fallback in `target`:

- `#login-btn`
- `.error-message`
- `//div[@class='alert']`

Prefer locator ids in `project.json` for maintainability.

### What if a locator id is missing?

The test may fail at **run time** when Playwright cannot find the element. Add the locator to `project.json` and re-translate if needed.

### What about dynamic data (emails, passwords)?

Use **variables** in `project.json`:

```json
"variables": {
  "testEmail": "qa@example.com",
  "testPassword": "{{ENV_TEST_PASSWORD}}"
}
```

Use `{{testEmail}}` in step `value` fields. `{{ENV_*}}` reads from environment variables.

### One Gherkin line, many actions?

Yes. Define one step definition that expands to multiple `fill`, `click`, or `wait` steps (see signup form example above).

---

## Quick reference

| I want to… | Do this |
|------------|---------|
| Write a test case | Edit `.feature` file |
| Map Gherkin to actions | Edit `step-definitions.yaml` |
| Define selectors | Edit `project.json` → `locators` |
| Generate JSON | Translate (UI or `npm run translate`) |
| Find selectors on a page | `npx playwright codegen <url>` |
| Run tests | `npm run test -- --project <name>` |
| View report | `npm run report -- --project <name>` |

---

## See also

- [README](../README.md) — setup and CLI commands
- `templates/signup-success.feature` — multi-field example
- `templates/signup-success.step-definitions.yaml` — companion step definitions
- `projects/coty/` — working demo against [coty.com](https://www.coty.com)
