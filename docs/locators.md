# Playwright Locators Guide (Java)

Last updated: 2025-08-11

This guide explains Playwright locator concepts in depth for Java users and compares them with Selenium’s element model.
It covers best practices, advanced patterns, and migration tips.

- What is a Locator
- Why Locators (vs ElementHandles)
- Key concepts: auto-waiting, strictness, visibility, retryability
- Locator APIs in Java
- Building robust locators
- Advanced patterns
- Assertions and waiting
- Comparison with Selenium
- Troubleshooting and FAQs
- References

## 1. What is a Locator?

A Locator is a lazy, chainable query that resolves to one or more elements on the page at action time. It is:

- Deferred: It does not immediately query the DOM; it stores a selector strategy.
- Retryable: Actions and assertions on a Locator auto-retry until the desired state is reached or a timeout elapses.
- Stable: Not tied to a specific element instance, avoiding stale-element issues.

In Java, you create locators from a Page or FrameLocator, for example:

```java
Locator button = page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Submit"));
button.click();
```

## 2. Why Locators (vs ElementHandles)?

- ElementHandle is a direct reference to a specific DOM node at a moment in time; it can become stale.
- Locator represents a strategy to find an element when needed; Playwright re-resolves and auto-waits by default.
- Prefer Locator wherever possible. Use ElementHandle for low-level operations or when strictly necessary.

### 2.1 ElementHandle example (Java)

Use ElementHandle for low-level DOM work where a concrete handle is required (e.g., custom JS evaluation, reading layout
via boundingBox, element-only screenshots). For regular user actions (click/fill), prefer Locator.

```java
// Prefer Locator for actions (auto-waits, re-resolves):
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().

setName("Submit")).

click();

// When you need a concrete ElementHandle:
ElementHandle handle = page.querySelector("#canvas"); // or: page.locator("#canvas").elementHandle();
if(handle !=null){
  // Example 1: Low-level JS evaluation on the element
  handle.

evaluate("el => el.scrollIntoView()");

// Example 2: Read layout information
ElementHandle.BoundingBox box = handle.boundingBox();
  if(box !=null){
  System.out.

printf("Canvas size: %.0fx%.0f at (%.0f, %.0f)%n",box.width, box.height, box.x, box.y);
  }

// Example 3: Element-only screenshot
byte[] png = handle.screenshot(new ElementHandle.ScreenshotOptions().setPath(Paths.get("canvas.png")));

// Note: You can click via ElementHandle, but it lacks Locator's strictness and robust auto-waiting.
// Prefer: page.locator("#save").click();
}
```

Notes:

- Getting handles from a Locator: `ElementHandle h = page.locator(".item").elementHandle();` or multiple via
  `List<ElementHandle> hs = page.locator(".item").elementHandles();`
- Handles can become stale after DOM updates. Re-acquire if the UI re-renders.
- Most tests should use Locator APIs for stability; reserve ElementHandle for specialized needs.

## 3. Key Concepts

### 3.1 Auto-waiting

Playwright automatically waits for elements to be actionable before performing actions like click and fill. It waits
for:

- Element to be attached to DOM
- Element to be visible and enabled (unless forced)
- Stable state (no animations covering it) when applicable

Example:

```java
page.locator("text=Login").click(); // waits for visibility and actionability
```

### 3.2 Strictness

Locators are strict by default: if a selector matches more than one element and the action needs a single element,
Playwright throws an error, prompting you to refine the locator.

Refine with chaining or filters:

```java
Locator items = page.locator(".todo-item");
Locator firstToggle = items.locator("input[type=checkbox]").first();
firstToggle.click();
```

### 3.3 Visibility

Many actions implicitly require visibility. You can also assert visibility explicitly:

```java
expect(page.locator("#status")).toBeVisible();
```

Java note: Use Playwright Test assertions in JS/TS; in Java, use LocatorAssertions from @playwright/test-junit or
AssertJ/TestNG with wait loops, or use has/hasText filters to ensure desired state before acting. See section 7.

### 3.4 Retryability

Actions like click, fill, check, and assertions like toHaveText (in Playwright Test) auto-retry until timeout. In Java,
locator actions still auto-wait; pair with assertions or state checks when needed.

### 3.5 Timeouts

- Global default timeout: page.setDefaultTimeout(ms)
- Navigation timeout: page.setDefaultNavigationTimeout(ms)
- Per-call overrides often available via options

```java
page.setDefaultTimeout(10000);
page.locator("text=Save").click(new Locator.ClickOptions().setTimeout(5000));
```

## 4. Locator APIs in Java

### 4.1 Creating Locators

- CSS: `page.locator(".btn.primary")`
- Text: `page.getByText("Welcome")` (smart text engine)
- Role/ARIA: `page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Submit"))`
- Test ID: `page.getByTestId("checkout")` (see section 5.2)
- Placeholder: `page.getByPlaceholder("Email")`
- Label: `page.getByLabel("Password")`
- Alt text: `page.getByAltText("Avatar")`
- Title: `page.getByTitle("Close")`
- XPath: `page.locator("xpath=//button[normalize-space(.)='Submit']")` (supported but generally prefer role/text)

### 4.2 Chaining and Scoping

You can scope locators to parents for strictness and clarity:

```java
Locator dialog = page.getByRole(AriaRole.DIALOG, new Page.GetByRoleOptions().setName("Settings"));
Locator save = dialog.getByRole(AriaRole.BUTTON, new Locator.GetByRoleOptions().setName("Save"));
save.click();
```

### 4.3 Indexing and Filtering

- `.first()`, `.last()`, `.nth(i)`
- `.filter(new Locator.FilterOptions().setHasText("Active"))`
- `.filter(new Locator.FilterOptions().setHas(someChildLocator))`

```java
Locator rows = page.locator("table#users tr");
Locator activeRow = rows.filter(new Locator.FilterOptions().setHasText("Active"));
activeRow.getByRole(AriaRole.BUTTON, new Locator.GetByRoleOptions().setName("Edit")).click();
```

### 4.4 Combining Locators (and/or)

- and: chain calls (narrowing)
- or: use locator("selector1, selector2") or build logic in code

```java
Locator action = page.locator("button.save, button.submit");
action.click();
```

### 4.5 Frame and Shadow DOM

- Frames: `page.frameLocator("iframe#iframeId").locator("text=Continue").click();`
- Shadow DOM: CSS pierces open shadow by default in Playwright selectors; use role/text when possible. For closed shadow
  DOM, interact via exposed handles or app APIs.

### 4.6 Lists and Iteration

```java
Locator items = page.locator(".result-item");
int count = items.count();
for (int i = 0; i < count; i++) {
  Locator item = items.nth(i);
  System.out.println(item.innerText());
}
```

### 4.7 Interaction APIs

Common methods include: click, dblclick, fill, type, press, check, uncheck, selectOption, hover, dragTo, screenshot,
evaluate.

```java
page.getByLabel("Email").fill("user@example.com");
page.getByLabel("Password").fill("secret");
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Sign in")).click();
```

## 5. Building Robust Locators

### 5.1 Prefer Semantic Selectors

Use getByRole/getByLabel/getByText. Benefits:

- Accessibility aligned
- Resilient to DOM/attribute churn
- Readable tests

```java
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Add to cart")).click();
```

### 5.2 Test IDs

Add stable data-testid attributes in your app:

```html
<button data-testid="checkout">Checkout</button>
```

Use:

```java
page.getByTestId("checkout").click();
```

Configure custom test id attribute globally if needed via selectors.setTestIdAttribute("data-qa").

### 5.3 Avoid brittle CSS/XPath

- Avoid overly specific CSS (deep descendant chains)
- Avoid XPath when role/text/testId suffice
- Avoid :nth-child chains that depend on layout

### 5.4 Scope to Regions

Scope to containers (dialogs, forms, tables) to ensure uniqueness and strictness.

### 5.5 Use Filters

`.filter({ hasText })` or `.filter({ has })` to target by embedded text or children.

## 6. Advanced Patterns

### 6.1 Conditional UI

```java
Locator save = page.locator("button.save, button.submit");
if (save.count() > 0) save.first().click();
```

### 6.2 Waiting for Specific States

```java
page.locator("#spinner").waitFor(new Locator.WaitForOptions().setState(WaitForSelectorState.DETACHED));
```

### 6.3 File Uploads

```java
page.getByLabel("Upload").setInputFiles(Paths.get("/path/to/file.png"));
```

### 6.4 Assertions (Java)

Playwright Java does not bundle the JS Playwright Test assertions, but you can:

- Use LocatorAssertions from com.microsoft.playwright.assertions (when available in your version)
- Or implement polling with Awaitility/TestNG/AssertJ

Example with built-in assertions (if available):

```java
import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

assertThat(page.locator("#status")).hasText("Ready");
```

Fallback custom poll:

```java
long deadline = System.currentTimeMillis() + 5000;
while (System.currentTimeMillis() < deadline) {
  if ("Ready".equals(page.locator("#status").innerText())) break;
  Thread.sleep(100);
}
```

### 6.5 Error Diagnostics

Use `.locator("…").highlight()` during debugging (when dev tools support) or rely on tracing/snapshots. Combine with
console logs and `page.onConsoleMessage`.

## 7. Assertions and Waiting Cheat Sheet

- Visibility: ensure element is visible before action (auto-waited) or explicitly assert.
- Text content: assert via hasText when available; otherwise poll innerText().
- Count: `assertThat(list).hasCount(n)` (if available) or `list.count()`.
- URL/navigation: `page.waitForURL("**/dashboard");`

## 8. Playwright vs Selenium (Locators and Concepts)

### 8.1 Object Model

- Playwright: Locator (lazy, retryable), ElementHandle (eager, low-level).
- Selenium: By + WebElement (eager element reference). WebElement can become stale after DOM updates.

### 8.2 Waiting

- Playwright: Auto-waits for actionability for most actions; unified, built-in.
- Selenium: No global auto-wait for actionability. Use implicit waits (discouraged for flakiness) or explicit waits (
  WebDriverWait + ExpectedConditions) per step.

### 8.3 Strictness and Errors

- Playwright: Strict by default—multiple matches cause informative errors, pushing toward precise selectors.
- Selenium: By locator returns first match; ambiguity can go unnoticed until the wrong element is clicked.

### 8.4 Built-in Semantics

- Playwright: getByRole, getByLabel, getByPlaceholder, getByText, getByTestId.
- Selenium: Primarily By.id, By.cssSelector, By.xpath, By.name, By.className, By.linkText; ARIA role-based querying
  requires custom libraries.

### 8.5 Stale Elements

- Playwright: Locators re-resolve; stale-element is rare when using Locator.
- Selenium: Frequent StaleElementReferenceException after re-renders; mitigated by re-querying or waits.

### 8.6 Shadow DOM and Iframes

- Playwright: frameLocator simplifies iframe scoping; selectors pierce open shadow roots; role/text often simplify.
- Selenium: Requires switching to frame contexts; shadow DOM needs JavaScript execution or ShadowRoot API.

### 8.7 Test IDs

- Playwright: First-class getByTestId with configurable attribute.
- Selenium: Use By.cssSelector("[data-testid='…']"). Works but lacks sugar and documentation emphasis.

### 8.8 Performance and Diagnostics

- Playwright: Batching, auto-waiting, trace viewer with DOM snapshots and selector insights.
- Selenium: Diagnostics rely on logs/screenshots/vendors; no unified trace viewer.

### 8.9 Example Contrast

Playwright (Java):

```java
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Search")).click();
```

Selenium (Java):

```java
WebElement btn = driver.findElement(By.xpath("//button[normalize-space()='Search']"));
new WebDriverWait(driver, Duration.ofSeconds(10))
  .until(ExpectedConditions.elementToBeClickable(btn)).click();
```

## 9. Troubleshooting and FAQs

- My locator matches multiple elements: narrow scope with a parent locator, use .filter({ hasText }), or .nth().
- Click is intercepted/element not clickable: rely on role/text locators, ensure no overlay; wait for overlays to
  detach.
- Flaky due to animations: wait for stable state; consider disabling animations in app/test CSS for CI.
- Dynamic lists: target rows by text with filter(hasText) and then select nested buttons by role.
- I use XPath heavily: prefer getByRole/getByText/test IDs for readability and stability; keep XPath for edge cases
  only.

Q: In Playwright it’s recommended to use semantic locators first (getByRole/getByLabel/getByText). Won’t that hurt
performance vs id/name as often recommended in Selenium?
A: In practice, no—when used correctly. Key points:

- Playwright’s semantic engines are optimized and run in the browser context. On typical pages the time difference
  between a scoped getByRole/getByTestId and an id lookup is negligible compared to network, rendering, and auto-waiting
  costs.
- Scope first. The biggest performance win is scoping to a smaller region (e.g., dialog/table row) before calling
  getByRole/getByText. This reduces search space and ambiguity.
- Prefer exact matches. For roles, provide an accessible name (and use exact when appropriate) to avoid broader text
  scanning.
- Choose the right signal. If you control the app, getByTestId (or a stable id attribute) is extremely fast and stable.
  For user-facing controls, getByRole with a name is both robust and fast enough.
- Avoid large unscoped getByText on screens with lots of repeating text; scope to a container or switch to role/testId.
- Diagnose with Trace Viewer. If a particular step is slow due to selector work, the trace will show long wait/resolve
  times—optimize that one step (scope, change strategy) rather than abandoning semantic locators globally.

Rule of thumb: Use the most semantic, stable signal that uniquely identifies the element, scoped to a region. getByRole
and getByTestId are usually both fast and resilient; raw id/name selectors aren’t inherently faster in any meaningful
way for end-to-end tests unless you rely on them exclusively and unscoped on very large DOMs. See section 14.3 for more
performance tips.

## 10. References

- Official docs (Java Locators): https://playwright.dev/java/docs/locators
- ARIA roles: https://www.w3.org/TR/wai-aria-1.2/#roles
- Assertions (Java): https://playwright.dev/java/docs/test-assertions
- Trace viewer (helpful for selectors): https://playwright.dev/java/docs/trace-viewer

## 11. All Available Locator Builders and Selector Engines (Java)

This section catalogs the primary locator builders available in Playwright for Java, with brief signatures, common
options, and examples. Prefer semantic builders (getByRole/getByLabel/…) over raw CSS/XPath whenever possible.

### 11.1 Page-level builders

- page.locator(String selector)
  - Description: Creates a Locator using Playwright selector syntax (CSS by default; can use engines like text=, xpath=,
    id=, etc.).
  - Example: `page.locator(".btn.primary").click();`

- page.frameLocator(String frameSelector)
  - Description: Scopes subsequent locator queries to matching iframes.
  - Example: `page.frameLocator("iframe#payments").getByText("Continue").click();`

- page.getByRole(AriaRole role, Page.GetByRoleOptions opts)
  - Common options: name (accessible name), exact, checked, disabled, expanded, pressed, selected, includeHidden.
  - Example: `page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Submit")).click();`

- page.getByText(String text)
  - Description: Finds elements by visible text with smart normalization (supports partial by default).
  - Tip: Use surrounding parent scoping to avoid ambiguity.
  - Example: `page.getByText("Welcome back").click();`

- page.getByLabel(String text, Page.GetByLabelOptions opts)
  - Description: Targets form controls associated with a label.
  - Options: exact.
  - Example: `page.getByLabel("Email").fill("user@example.com");`

- page.getByPlaceholder(String text, Page.GetByPlaceholderOptions opts)
  - Description: Matches input elements by placeholder text.
  - Example: `page.getByPlaceholder("Search products").fill("laptop");`

- page.getByAltText(String text, Page.GetByAltTextOptions opts)
  - Description: Targets elements by alt text (e.g., images).
  - Example: `page.getByAltText("Company logo").click();`

- page.getByTitle(String text, Page.GetByTitleOptions opts)
  - Description: Targets elements by title attribute.
  - Example: `page.getByTitle("Close").click();`

- page.getByTestId(String testId)
  - Description: Targets elements by a dedicated test id attribute.
  - Example: `page.getByTestId("checkout").click();`
  - Configure custom attribute: `playwright.selectors().setTestIdAttribute("data-qa");`

### 11.2 Locator-level builders (scoped to an existing Locator)

- locator.locator(String selector)
  - Description: Further narrows within the current scope using selector syntax.
  - Example: `Locator dialog = page.getByRole(AriaRole.DIALOG); dialog.locator("button.primary").click();`

- locator.getByRole(AriaRole role, Locator.GetByRoleOptions opts)
  - Example: `region.getByRole(AriaRole.BUTTON, new Locator.GetByRoleOptions().setName("Save")).click();`

- locator.getByText(String text)
  - Example: `row.getByText("Active");`

- locator.getByLabel / getByPlaceholder / getByAltText / getByTitle / getByTestId
  - Description: Same semantics as Page-level builders but scoped to the current Locator.

### 11.3 Locator transformations and filters

- first() / last() / nth(int index)
  - Example: `page.locator(".todo-item").first().click();`

- filter(Locator.FilterOptions opts)
  - hasText: restrict by visible substring.
  - has: restrict by nested child locator.
  - Example: `rows.filter(new Locator.FilterOptions().setHasText("Active"));`

- count(), allTextContents(), allInnerTexts()
  - Examples:
    - `int n = rows.count();`
    - `List<String> texts = rows.allTextContents();`

- waitFor(Locator.WaitForOptions opts)
  - Common states: ATTACHED, DETACHED, VISIBLE, HIDDEN.
  - Example: `spinner.waitFor(new Locator.WaitForOptions().setState(WaitForSelectorState.DETACHED));`

### 11.4 Interaction methods on Locator (high level)

- click, dblclick, hover, press, fill, type, check, uncheck, selectOption, setInputFiles, dragTo, focus, blur,
  screenshot, evaluate.
- Examples:
  - `page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Sign in")).click();`
  - `page.getByLabel("Upload").setInputFiles(Paths.get("/path/to/file.png"));`

### 11.5 Selector engines (string selectors)

You can pass the following engines to page.locator(...) and locator.locator(...). When no engine prefix is provided, CSS
is used by default.

- CSS (default)
  - Example: `.inventory .item >> css=button.buy` (tip: prefer chaining over ">>")
  - Recommended approach: `page.locator(".inventory .item").locator("button.buy");`

- text=
  - Example: `page.locator("text=Log in");`
  - Notes: Use getByText where possible for readability and semantics.

- xpath=
  - Example: `page.locator("xpath=//button[normalize-space(.)='Submit']");`

- id=
  - Example: `page.locator("id=main");`

- role= (internal engine)
  - Prefer the Java API getByRole rather than raw engine syntax.

- data-testid= (through getByTestId)
  - Prefer getByTestId over raw attribute selectors; configure attribute via selectors().setTestIdAttribute.

### 11.6 Playwright-specific selector pseudo-classes and functions

These work within Playwright selector strings (CSS-engine compatible context):

- :has(<selector>)
  - Example: `page.locator(".card:has(button:has-text('Buy'))");`

- :has-text(<text>)
  - Example: `page.locator("tr:has-text('Active')");`

- :visible / :hidden / :enabled / :disabled / :checked
  - Examples: `page.locator("button:visible");`, `page.locator("input:disabled");`

- :nth-match(<selector>, <index>)
  - Example: `page.locator(":nth-match(.result-item, 3)");`

- :scope
  - Example: `container.locator(":scope > .row");`

Notes:

- Prefer Locator chaining and filter(has/hasText) over complex single-string selectors for readability and strictness.
- The old relative selector combinator ">>" is considered legacy in docs; use locator chaining instead.

### 11.7 Iframes and Shadow DOM

- Iframes: Use frameLocator to scope queries into frames.
  - `page.frameLocator("iframe#iframeId").getByText("Continue").click();`
- Shadow DOM: Playwright selectors pierce open shadow roots by default. For closed shadow roots, rely on app-provided
  handles or APIs.

### 11.8 Configuring the Test ID attribute

- Default attribute: data-testid
- Change globally:
  - `try (Playwright pw = Playwright.create()) { pw.selectors().setTestIdAttribute("data-qa"); /* ... */ }`

## 12. Quick Reference

- Semantic builders: getByRole, getByLabel, getByText, getByPlaceholder, getByAltText, getByTitle, getByTestId.
- Generic builders: locator(selector), frameLocator(selector).
- Transformations: first, last, nth, filter(has/hasText).
- Selector engines: css (default), text=, xpath=, id=, role engine (use getByRole), data-testid (use getByTestId).
- Pseudo-classes: :has, :has-text, :visible, :hidden, :enabled, :disabled, :checked, :nth-match, :scope.
- Iframes: frameLocator; Shadow DOM: open shadow pierced by default.
- Test ID attribute: playwright.selectors().setTestIdAttribute("...").
- Custom selector engines: playwright.selectors().register(name, script | path [, options]). See section 13.

## 13. Custom Locators (Custom Selector Engines)

In addition to built-in engines (css, text, xpath, id, role, data-testid), Playwright lets you register your own
selector engines. This is useful for domain-specific patterns or when you want a terse syntax for repeated structures.

Important notes

- Prefer built-in semantic locators first (getByRole, getByLabel, getByText, getByTestId). Custom engines add
  maintenance overhead.
- Engine names are case-sensitive and may only contain [a-zA-Z0-9_].
- You cannot override predefined engines like "css", "xpath", "text", etc.
- You can register before or after creating a BrowserContext. Engines registered later become available to existing
  contexts as well.
- If you pass options.setContentScript(true), the engine runs in an isolated world (content script). Otherwise, it runs
  in the main world.

13.1 Registering from a JavaScript object (inline)

```java
String myEngine = "{\n" +
  "  create(root, target) { return target.nodeName; },\n" +
  "  query(root, selector) { return root.querySelector(selector); },\n" +
  "  queryAll(root, selector) { return Array.from(root.querySelectorAll(selector)); }\n" +
  "}";
try (Playwright pw = Playwright.create()) {
  // Register a custom engine named 'tag'
  pw.selectors().register("tag", myEngine);

  Browser browser = pw.chromium().launch();
  BrowserContext context = browser.newContext();
  Page page = context.newPage();
  page.setContent("<div><span></span></div><div></div>");

  // Use it via the engine prefix: engineName=selector
  page.locator("tag=DIV").first().click();
  int count = page.locator("tag=DIV").count(); // => 2
}
```

13.2 Registering from a file path

```java
try (Playwright pw = Playwright.create()) {
  pw.selectors().register("section", Paths.get("src/test/resources/sectionselectorengine.js"));
  Page page = pw.chromium().launch().newContext().newPage();
  page.setContent("<section></section>");
  page.locator("section=whatever").click();
}
```

Example sectionselectorengine.js

```js
module.exports = {
  // Optional: return a selector string when the user picks an element in the inspector
  create(root, target) {
    // e.g., choose a stable representation of the target
    return target.nodeName.toLowerCase();
  },
  // Return the first matching element in 'root'
  query(root, selector) {
    return root.querySelector(selector);
  },
  // Return all matching elements in 'root'
  queryAll(root, selector) {
    return Array.from(root.querySelectorAll(selector));
  },
};
```

13.3 Running in the isolated world (content script)

```java
String engine = "{\n" +
  "  create(root, target) { },\n" +
  "  query(root, selector) { return window['__answer']; },\n" +
  "  queryAll(root, selector) { return window['__answer'] ? [window['__answer'], document.body, document.documentElement] : []; }\n" +
  "}";
try (Playwright pw = Playwright.create()) {
  pw.selectors().register("isolated", engine, new Selectors.RegisterOptions().setContentScript(true));
  // By default, such engines run in an isolated world and do not see window['__answer'] set in main world.
}
```

13.4 A realistic example: data-qa engine
This custom engine makes the syntax dataqa=checkout equivalent to [data-qa="checkout"]. In practice, prefer getByTestId
with selectors().setTestIdAttribute("data-qa") because it integrates deeply with Playwright. Use custom engines only
when you need special semantics.

```java
String dataqaEngine = "{\n" +
  "  create(root, target) { return target.getAttribute('data-qa') || ''; },\n" +
  "  query(root, selector) { return root.querySelector(`[data-qa=\\"${selector}\\"]`); },\n" +
  "  queryAll(root, selector) { return Array.from(root.querySelectorAll(`[data-qa=\\"${selector}\\"]`)); }\n" +
  "}";
try (Playwright pw = Playwright.create()) {
  pw.selectors().register("dataqa", dataqaEngine);
  Page page = pw.chromium().launch().newContext().newPage();
  page.setContent("<button data-qa=\"checkout\">Checkout</button>");
  page.locator("dataqa=checkout").click();
}
```

13.5 Error handling and constraints

- Unknown engine name results in: Unknown engine "name" while parsing selector name=...
- Duplicate registration throws an error: "<name> selector engine has been already registered".
- Invalid names (e.g., "$foo") throw: "Selector engine name may only contain [a-zA-Z0-9_] characters".
- Built-in names (e.g., "css") are reserved and cannot be registered over: '"css" is a predefined selector engine'.

13.6 When to use custom engines vs Selenium

- Playwright: Prefer semantic builders and test ids; use custom engines sparingly for domain-specific patterns.
- Selenium: Custom strategies are often implemented via By.cssSelector(...) wrappers or custom By implementations;
  Playwright’s register offers a more integrated way with the selector syntax (engine=selector).

## 14. Locator Strategy Priority, Performance, and Usage Guide

This section provides a practical, opinionated guide for choosing locators in Playwright Java. It covers priority order,
performance considerations, locator types, and concrete advice on when and where to use each approach.

### 14.1 Recommended Priority Order (TL;DR)

Use the highest semantic, most stable signal that uniquely identifies the element. Prefer scoping to a region before
choosing a selector.

1) getByRole(...name=...) — Primary for interactive controls ✓

- Why: Semantic, accessibility-aligned, robust across DOM refactors.
- Examples:
  - Dialog buttons, menu items, tabs, links, checkboxes, radios.
- Tips: Scope to a region: dialog.getByRole(BUTTON, name="Save").

2) getByTestId("...") — Stable, explicit hook ✓

- Why: Stable across refactors, intentional for testing, localizable apps.
- Examples: Controls without clear role/name; non-textual icons; complex widgets.
- Tips: Configure globally via selectors().setTestIdAttribute("data-qa").

3) getByLabel / getByPlaceholder — Form-centric ✓

- Why: Human-facing association; resilient if markup changes but labels remain.
- Examples: Inputs, selects, textareas.

4) getByText("...") — Visible text ✓

- Why: Simple, readable when the UI text is stable.
- Caveats: Sensitive to copy changes, i18n, and dynamic text.
- Tip: Scope to parent region to avoid ambiguity.

5) CSS (page.locator("...")) — Structural/attribute targeting •

- Why: Powerful and fast; good for layout-driven or test-id-like attributes.
- Caveats: Brittle if chained deeply; avoid long descendant chains and :nth-child.
- Tip: Prefer chaining locators over complex single strings.

6) XPath — Edge cases only •

- Why: Expressive for certain structural queries.
- Caveats: Less readable; easy to make brittle selectors; harder to debug.

Rule of thumb: Scope first, then choose the most semantic builder that uniquely identifies your target. Reserve
CSS/XPath for gaps or special cases.

### 14.2 Locator Types: Pros and Cons

- Semantic locators: getByRole, getByLabel, getByText, getByPlaceholder
  - Pros: Readable, aligned with accessibility, resilient to structural changes.
  - Cons: Requires proper ARIA/labels; getByText can be sensitive to copy changes.

- Attribute-based locators: getByTestId, CSS attr selectors
  - Pros: Stable, intentional hooks; great for complex widgets and i18n.
  - Cons: Requires adding/maintaining attributes in app code.

- Structural locators: CSS hierarchy, :has(), :nth-match(), :scope
  - Pros: Powerful composition; good within scoped regions for disambiguation.
  - Cons: Can become brittle and slow if overly complex or unscoped.

- Custom selector engines (registered engines)
  - Pros: Domain-specific shorthand; can encode business semantics.
  - Cons: Extra maintenance; avoid unless a clear, repeated pattern exists.

### 14.3 Performance Considerations

General performance best practices:

- Scope queries to a region first (dialog, form, table, list item). This reduces DOM search and ambiguity.
- Prefer locator chaining over deep, single-string selectors: region.locator(".row").locator("button.buy").
- Avoid frequent count() on large lists during polling; it queries the DOM each time.
- Use has/hasText filters thoughtfully; they are powerful but can be more expensive than simple role/testId matches.
- Prefer exact matching when possible (e.g., role name exact) to reduce text engine work.

Specific notes:

- getByRole: Usually efficient when scoped; leverages the accessibility tree heuristics.
- getByText: Normalizes whitespace and visibility; can be heavier in large containers. Scope it.
- :has and :has-text: Powerful but can be slower on large DOMs; prefer has(locator) filter when possible.
- allTextContents()/allInnerTexts(): Consolidate multiple queries into one; still O(n) in matched elements.
- waitFor and auto-waiting: Let Playwright handle actionability; avoid manual sleeps.

CI tips:

- Keep selectors strict to minimize retries.
- Use tracing to diagnose slow/ambiguous selectors.

### 14.4 When to Use What (Common Scenarios)

- Forms (inputs, selects, textareas)
  - Prefer: getByLabel, getByPlaceholder. Fallback: getByTestId, role (textbox/combobox) with name.

- Buttons, links, menus, tabs
  - Prefer: getByRole(…name=…). Fallback: getByText (scoped) or getByTestId for icon-only controls.

- Tables and lists
  - Strategy: Scope to table/list region -> filter row by hasText -> find nested control by role.
  - Example: table.locator("tr").filter(hasText("Alice")).getByRole(BUTTON, name("Edit"))

- Dialogs, popovers, side panels
  - Scope to dialog by role/name -> query children by role/testId/text.

- Icon-only buttons
  - Prefer: getByRole(BUTTON, name=accessibleName). If no accessible name, use getByTestId or add one.

- Internationalized (i18n) UIs
  - Prefer: getByRole and getByTestId to minimize text coupling.
  - If using getByText, ensure stable, localized key strings or test per locale.

- Single-page apps (frequent re-renders)
  - Prefer: Locators (not ElementHandle). Scope and semantic builders reduce flakiness.

- Shadow DOM
  - Open shadow is pierced by default. Use role/testId within the component. For closed shadow, rely on app APIs.

- Iframes
  - Use frameLocator("iframe#...") to scope; then regular builders inside.

### 14.5 Do and Don’t Cheat Sheet

Do:

- Do scope first (region -> child).
- Do prefer getByRole/getByLabel/getByTestId for stability and readability.
- Do use filter(has/hasText) for targeting rows/cards by content.
- Do configure a project-wide test id attribute early.

Don’t:

- Don’t write very long CSS with multiple descendant hops and :nth-child chains.
- Don’t rely on brittle innerText without scoping; prefer role or testId.
- Don’t overuse XPath for general queries.
- Don’t add huge timeouts; fix ambiguity instead.

### 14.6 Selenium Comparison: Priority and Performance

- Priority
  - Playwright: Semantics (role/label/testId) first; CSS/XPath as fallback.
  - Selenium: Often id/css/xpath first; ARIA-based querying requires extra libs.

- Performance & stability
  - Playwright: Auto-waiting and lazy locators reduce retries/staleness; scoped semantic queries are fast and
    maintainable.
  - Selenium: Manual waits and stale element handling add overhead; deep XPath/CSS often used, increasing brittleness.

In summary: Scope your search and favor semantic, stable signals (role, label, test id). Use text sparingly and scoped.
Reserve structural CSS/XPath for edge cases. This yields both fast and resilient tests in Playwright.
