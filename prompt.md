# Prompt: Test Explorer, a visualizer for a project's tests

Copy everything below the line into any LLM that has access to the project files (Claude Code, Cursor, Copilot Agent, etc.).
If the model has no file access, attach the test folder structure and a few sample test files.

---

## Role and goal

You are a developer and QA analyst. Your task is to build a web application from the project's automated tests that **clearly shows WHAT the tests actually verify**: which areas of the system are covered, which scenarios exist, with what data, and with what expected results. The audience is people who don't read test code (managers, analysts, new QA engineers).

## Step 1. Reconnaissance (write nothing until you understand the project)

1. Identify the stack and the test format: `package.json`, configs (`cypress.config.*`, `playwright.config.*`, `jest.config.*`, `pytest.ini`, etc.), README.
2. Find all test files and count them. For BDD, look for `*.feature`.
3. See how folders group the tests (by domain, endpoint, page). This becomes the navigation tree.
4. Read 2-3 typical files in full to understand the style: tags, `Background`, `Scenario Outline` + `Examples`, data tables, docstrings.
5. Determine what is being tested: API (HTTP methods, endpoints), UI (pages, forms), or both. Look for this in Feature titles, tags and steps.
6. Find out whether the project has restrictions (for example, "never commit anything"). **Do not run `git add/commit/push` unless asked.**
7. Pick the adaptation profile from the "Adapting to other frameworks" section below, if the tests are not Gherkin.

## Step 2. Parser to data

Write a script (dependency-free Node.js, or Python) that reads all test files and emits JSON:

```
feature:  { file, dir, title, tags[], description[], background[steps], scenarios[] }
scenario: { name, outline(bool), tags[], steps[], examples[] }
step:     { kw: Given|When|Then|And|But, text, table?, doc? }
examples: { header[], rows[][] }
```

Parser requirements (Gherkin):
- skip blank lines and `#` comments;
- tags (`@...`) belong to the next Feature/Scenario/Examples;
- support `Feature`, `Background`, `Scenario`, `Scenario Outline`/`Template`, `Examples`;
- `| a | b |` tables are either Examples or the data table of the previous step; an escaped `\|` is not a separator;
- attach docstrings `"""..."""` to their step;
- for non-Gherkin frameworks, follow the adaptation section and map the result onto the same data structure.

For every test compute:
- `short`, a human-readable suite name (strip a prefix like `api:` and a suffix like `test suite`);
- `methods`, the HTTP methods from a title like `[get, post]` (or from the code, see the adaptation section);
- `cases`, the number of real checks (for an Outline: the sum of rows across all Examples);
- `kind`, an **automatic classification** from the name, steps and file path:
  - `perm`: access rights (`permission`, `forbidden`, `403`);
  - `neg`: negative (`unsuccessful`, `invalid`, `error`, `without`, `not allowed`, ...), unless the name contains `(successful)`;
  - `pos`: everything else.
  The keyword lists are heuristics tuned to one project. **Re-derive them from the real test names of the project you are analyzing** (list the most frequent words in test titles first). State in the UI and in your final report that the classification is approximate.

## Step 2b. Plain-language steps (mandatory for non-Gherkin code)

Raw code in a step (`await expect(page.getByRole("button", { name: "Sign In" })).toBeVisible()`) defeats the purpose of the page. Every step must be **translated into a plain English sentence that a non-programmer can read**, derived mechanically from the code's AST, never guessed:

1. **Parse, don't regex.** Work on the AST (TypeScript compiler API, Python `ast`, etc.). Resolve each call by its *function*: action, locator, assertion, mock, helper call.
2. **Actions** become verb phrases: `click(x)` -> `Click <x>`; `fill(v)` -> `Enter <v> into <x>`; `check/uncheck`, `selectOption(v)` -> `Select <v> in <x>`; `press(k)`; `hover`; `goto(url)` -> `Open <url>`; `reload`; file upload; waits (`Wait for ...`); HTTP calls -> `Send GET request to "/path" with body ...`.
3. **Locators** become descriptions of what a user sees, not selectors: `getByRole("button", {name: "Sign In"})` -> `"Sign In" button`; `getByText("X")` -> `text "X"`; `getByLabel("X")` -> `element labeled "X"` (do not call it a field or button unless the code says so); `getByPlaceholder`; `getByTestId`; `locator("css")` -> `element "css"`; `.first()/.nth(1)/.last()` -> `first / 2nd / last`; `.filter({hasText})` -> `containing "..."`; chains -> `outer › inner`.
4. **Resolve names through the project.** If a locator is a Page Object property (`this.submitButton = page.getByLabel("Submit")`), show the resolved description instead of the identifier. For Page Object methods show `On the <page name>: <method name humanized> <args>` and keep an expandable "what this method does" list built by applying the same rules to the method body.
5. **Assertions** become statements of expected state: `toBeVisible` -> `<x> is visible`; `not.toBeVisible` -> `is not visible`; `toHaveText("t")` -> `has text "t"`; `toHaveValue`; `toBeChecked`; `toHaveCount(n)` -> `appears n time(s)`; `toHaveURL`; `toHaveTitle`; status codes -> `response status equals 403`; unit-test matchers (`toEqual`, `toBeNull`, `toHaveBeenCalledWith`, `rejects.toThrow`) -> `... equals ...`, `... is null`, `... was called with ...`, `... rejects with ...`. Unknown matchers fall back to the humanized matcher name plus its arguments.
6. **Values:** string literals in double quotes; numbers as is; a constant that resolves to a literal in the same file is shown as that literal; any other variable as a `<placeholder>`.
7. **Control flow:** loops -> `For each <item> in <array literal if resolvable>`; conditions -> `If <cond>` / `Else` with `===` rendered as `is`, `&&` as `and`; `test.step("title")` -> `Step: title`.
8. **Mocks / setup (unit tests):** `jest.mock("m")` -> `Mock module "m"`; `fn.mockResolvedValue(v)` -> `Make fn resolve with v`; calling the unit under test -> `Call <function name humanized> with <args>`.
9. **Keep the original.** Store the raw code of every step next to its sentence; show it as a tooltip and via a "Show code" toggle in the UI. Never drop it, so readers can verify the translation.
10. **No invention.** If a construct is not covered by the rules, show the humanized call (`Call <name> with <args>`) rather than a made-up description, and keep the raw code visible.

## Step 3. Web application

Build **one self-contained HTML file** (no server, no CDN): a `template.html` plus a build script that embeds the JSON into `index.html`. When embedding the JSON:
- use a **replacement function** in `String.replace(..., () => json)`, otherwise `$&` / `$'` sequences in the data will break the code;
- escape `<`, U+2028 and U+2029.

Required UI features:
1. **Overview (home):** cards (files, scenarios, checks including Examples, positive/negative/permission counts); a bar chart per area, split by pos/neg/perm, with a caption saying what each area covers; a summary by HTTP method; a "How to read" block.
2. **Left tree:** areas, then suites, with counters; clicking opens the section.
3. **Suite card:** title, method badges, file path, tags, `Background` ("Before every test").
4. **Scenario** (collapsible): name, tags, a kind badge (positive/negative/permission), number of variants; inside, Given/When/Then steps (plain-language sentences from Step 2b, with a "Show code" toggle) in different colors, `"quoted"` values and `<placeholders>` highlighted, Examples tables and data tables shown in full.
5. **Search** across names, steps, tags (task IDs such as `BDDTEST-LAM-1237`) and Examples values, with match highlighting.
6. **Filters** by kind: all / positive / negative / permission.
7. Light and dark theme (system default plus a toggle), responsive layout for phones, no horizontal page scroll.
8. **Area descriptions** (a "folder to title + what it covers" dictionary). Write them from folder names and Feature titles; do not invent. If unsure, keep the folder name. Write them in **English**.

Design: neutral, dense, readable; colors via CSS variables; no external libraries.

**Language: the entire generated `index.html` must be in English.** This covers every UI label, button, heading, card title, tooltip, placeholder, legend, the "How to read" block, area descriptions, empty-state messages, and the text of generated steps (for example `For each x of [...]`, `If ...`, `Else`, `Step: ...`, `mock module ...`). Do not leave any Russian or other non-English strings in `template.html` or `build.mjs`. Test titles are shown exactly as written in the source code; never translate or rewrite them. Set `<html lang="en">`.

## Step 4. Verification (mandatory)

1. Run the build and make sure it prints the file and scenario counts, and that they match `grep -c "Scenario" ...` on the sources (or the equivalent for your framework).
2. Open `index.html` in headless Chrome with a screenshot and the console log:
   `chrome --headless --window-size=1300,1500 --screenshot=out.png --virtual-time-budget=3000 --enable-logging=stderr --v=0 file://.../index.html 2>&1 | grep CONSOLE`
   There must be no console errors. Look at the screenshot yourself.
3. Check at least one section with a `Scenario Outline` (or a parametrized test), the search, and a filter.
4. Verify the area descriptions against the real folder contents.

## Step 5. Result

- Put everything in a new folder (for example `test-explorer/`): `build.mjs`, `template.html`, `index.html`.
- Do not touch existing project files and do not commit unless asked.
- In your reply, briefly state: how to open and refresh (`node test-explorer/build.mjs`), how many files/scenarios/checks were found, what is approximate (classification, area descriptions), and what could be added next (linking steps to step definitions and endpoints, publishing the page for the team).

## Adapting to other frameworks

Gherkin has an explicit structure (Given/When/Then). Other frameworks don't, so you have to **derive** "what the test verifies" from code. The principle: show the test's title, the actions it performs and the assertions it makes, never an invented description.

### Mapping table

| Concept | Gherkin | Playwright / Jest / Mocha | pytest | JUnit / TestNG |
|---|---|---|---|---|
| Suite (card) | `Feature` | `test.describe(...)` / `describe(...)` or the file | file or `class Test*` | class |
| Scenario | `Scenario` | `test('...')` / `it('...')` | `def test_*` | `@Test` method |
| Parametrized | `Scenario Outline` + `Examples` | `for` loop or `.forEach` generating tests, `test.each`, `it.each` | `@pytest.mark.parametrize` | `@ParameterizedTest` |
| Tags | `@tag` | `{ tag: ['@smoke'] }`, `@smoke` in the title, `test.describe.configure` | `@pytest.mark.x` | `@Tag` |
| Before each | `Background` | `beforeEach` / `test.beforeEach` | fixtures, `setup_method` | `@BeforeEach` |
| Steps | `Given/When/Then` | see below | comments, helper calls, `assert` | helper calls, `assert*` |

### Playwright (priority)

Playwright tests are TypeScript/JavaScript code, so a regex parser is fragile. **Prefer the real AST or Playwright's own metadata:**

1. **Best source of the test list:** run `npx playwright test --list --reporter=json`. It returns the full tree: file, suite titles, test title, tags, project, line number, with parametrized tests already expanded. Build the data from this JSON instead of parsing source text. This also gives the exact number of checks.
2. **Details (steps and assertions):** for each test, read its source by file and line number (taken from the JSON above) and extract, in order:
   - `test.step('title', ...)`: these are the real "steps"; use their titles as the step text;
   - navigation and actions: `page.goto(url)`, `.click()`, `.fill(value)`, `.selectOption()`, `.check()`, `.press()`, and API calls `request.get/post/put/patch/delete(url, { data })`;
   - assertions: `expect(...).toBe*`, `toHaveText`, `toHaveURL`, `toBeVisible`, `toHaveStatus`, `expect(response.status()).toBe(403)`, and so on.
   Map them to a Given/When/Then-like view: setup and `beforeEach` become *Given*, actions become *When*, `expect` becomes *Then*. Render the call text as the step, for example `Then expect locator "#error" toHaveText "Invalid password"`.
3. **If the project uses Page Objects:** an action like `loginPage.submit()` is not self-explanatory. Resolve the method in the page class and either show its name humanized (`login page: submit`) or inline a one-line summary of what it does. Do not guess what it does without reading it.
4. **Data:** for parametrized tests (`for (const x of cases) test(...)`, `test.each`) read the data array and show it as an Examples-like table. If the data comes from a file (JSON/CSV), read it.
5. **Type (UI vs API):** a test using `page` is UI; a test using only `request` (or `APIRequestContext`) is API. Show a badge; for API tests extract the HTTP method and URL into `methods` and the area.
6. **Areas:** use the folder structure of `testDir`, or `test.describe` titles, or projects/tags (`@smoke`, `@regression`) as an extra filter dimension.
7. **Classification** for Playwright usually keys off words in titles and assertions: `should not`, `invalid`, `error`, `denied`, `unauthorized`, `redirects to login`, status codes `4xx`/`5xx` in `expect(...status())` mean negative; role-based tests (`as viewer`, `forbidden`, `403`) mean permission. Re-derive from real titles.
8. Also surface Playwright-specific facts that matter to readers: `test.skip`, `test.fixme`, `test.fail` (known problems; show a badge), `test.slow`, retries, and which projects/browsers a test runs in.

### Jest / Mocha / Vitest

- Use the AST (`@babel/parser`, `typescript`, or `esbuild`) rather than regexes: `describe`/`it`/`test` nesting is arbitrary.
- Steps: there are none. Show the sequence of calls to the code under test plus each `expect(...)` chain. For `it.each`/`test.each`, expand the table into Examples.
- Mark `it.skip`, `it.todo`, `xit` explicitly.

### pytest

- Use `ast` (Python). Suites are files or `class Test*`; tests are `def test_*`; parametrization is `@pytest.mark.parametrize` (read the argument list into an Examples-like table).
- Steps: the test docstring, `# given/when/then` comments, and `assert` statements. Fixtures used by the test (its arguments) play the role of *Background*.
- Markers (`@pytest.mark.slow`, `skip`, `xfail`) become tags.

### Cypress (non-Gherkin) / Selenium / others

- Cypress `describe/it` follows the Jest model; `cy.visit`, `cy.get().click()`, `cy.request`, `cy.intercept` are the actions, `.should(...)` / `expect` are the assertions.
- Selenium (any language): find the framework's test runner (JUnit, NUnit, pytest) and follow its row above; actions are `driver.get`, `findElement(...).click()`, assertions are the `assert*` calls.

### Always, for any framework

- Count tests with the tool's own listing command when one exists (`--list`, `--collect-only`, `--dry-run`), and compare to your parser's count.
- If something cannot be extracted reliably, show the test title and file/line, and say so explicitly; never fabricate a description.
- Keep the same output data structure so the same `template.html` works for every framework; only the parser changes.

## Anti-patterns

- Do not write the app without reading the real tests.
- Do not invent what a test "checks" if the scenario or code does not say so.
- Do not count scenarios by eye, only with the parser and the framework's own list command.
- Do not leave an unresolved placeholder like `/*__DATA__*/` in the final HTML.
- Do not use a CDN: the file must open offline with a double click.
- Do not reuse another project's classification keywords without checking them against this project's test titles.
