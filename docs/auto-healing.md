# Auto-Healing in Playwright (Java)

Last updated: 2025-08-10

This guide clarifies what “auto-healing” typically means in test automation, how Playwright approaches robustness, and
how you can optionally implement a small, non-invasive fallback utility in Java for your own project.

- What is auto-healing?
- Does Playwright have built-in auto-healing?
- How Playwright achieves robustness without mutating selectors
- Recommended practices (the Playwright way)
- Optional: a lightweight “AutoHealing” fallback utility (Java example)
- Comparison with Selenium ecosystem
- FAQs and limitations

## 1. What is auto-healing?

In some Selenium-based frameworks and vendor tools, “auto-healing” refers to mechanisms that dynamically change locators
at runtime when the original locator fails (e.g., attributes changed). These tools may guess new selectors, learn from
history/ML, or maintain multiple strategies behind the scenes.

## 2. Does Playwright have built-in auto-healing?

Short answer: Playwright does not automatically rewrite your selectors at runtime. There is no opaque "self-heal" that
silently changes your locator strings.

Instead, Playwright aims to make tests resilient by design so that you don’t need selector mutation:

- Locators are lazy and re-resolve on each action (no stale element issues like WebElement in Selenium).
- Actions auto-wait for the element to be attached, visible, and actionable.
- Locator APIs encourage semantics (getByRole/getByLabel/getByText) that are less brittle than CSS/XPath tied to
  structure.
- Strictness helps you catch ambiguity early.

See the Locators Guide for details: docs/locators.md.

## 3. How Playwright achieves robustness (no selector mutation)

Under the hood, Playwright contributes to “auto-healing-like” stability through:

- Auto-waiting and retries: Actions like click/fill re-try within the timeout, waiting for DOM and actionability
  conditions.
- Semantic engines: getByRole/getByLabel/getByText rely on the accessibility tree and visible text normalization, which
  tolerate certain DOM changes better than brittle CSS/XPath.
- Fresh resolution: Locators don’t hold on to a specific node handle, they resolve when needed, which avoids stale
  references after re-renders.
- Traceability: With tracing and snapshots, you can quickly diagnose selector issues and adjust intentionally rather
  than guessing at runtime.

This approach favors explicit, maintainable tests over hidden mutations.

## 4. Recommended practices (the Playwright way)

- Prefer semantic locators:
  - getByRole(…name=…) for interactive controls
  - getByLabel, getByPlaceholder for form fields
  - getByText for visible text
  - getByTestId for stable, explicit hooks (configurable attribute)
- Scope your locators to regions (dialogs, forms, tables) to keep them strict and unambiguous.
- Avoid brittle CSS/XPath that depend on layout and long descendant chains.
- Use filter(has/hasText) and chaining rather than deeply nested single-string selectors.
- Rely on auto-waiting and assert important states before acting.

See docs/locators.md for a complete catalog and best practices.

## 5. Optional: a lightweight “AutoHealing” fallback utility (Java example)

If your team still wants a controlled fallback (without changing Playwright core behavior), you can create a small
helper that:

- Tries the primary, semantic locator first.
- On failure or timeout, tries one or more alternative strategies you define.
- Logs what was tried, and fails clearly if nothing matched.

Important: Keep fallback lists short and intentional. Overusing fallbacks can mask real regressions.

Example utility (for demos/tests; place in your project’s test utils):

```java
package org.example;

import com.microsoft.playwright.*;

import java.time.Duration;
import java.util.Arrays;
import java.util.List;
import java.util.function.Supplier;

public class AutoHealing {
  private final Page page;
  private final int perAttemptTimeoutMs;

  public AutoHealing(Page page, int perAttemptTimeoutMs) {
    this.page = page;
    this.perAttemptTimeoutMs = perAttemptTimeoutMs;
  }

  /** Try a sequence of locator suppliers until one is actionable, then click it. */
  public void click(Supplier<Locator>... candidates) {
    attempt("click", Arrays.asList(candidates), loc -> {
      loc.click(new Locator.ClickOptions().setTimeout((double) perAttemptTimeoutMs));
      return true; // success
    });
  }

  /** Fill tries each candidate until successful. */
  public void fill(String value, Supplier<Locator>... candidates) {
    attempt("fill", Arrays.asList(candidates), loc -> {
      loc.fill(value, new Locator.FillOptions().setTimeout((double) perAttemptTimeoutMs));
      return true;
    });
  }

  private interface LocatorAction {
    boolean run(Locator locator);
  }

  private void attempt(String action, List<Supplier<Locator>> candidates, LocatorAction actionFn) {
    PlaywrightException last = null;
    for (int i = 0; i < candidates.size(); i++) {
      Locator locator = candidates.get(i).get();
      try {
        // Optional: small wait to ensure the element is visible/actionable
        locator.waitFor(new Locator.WaitForOptions().setState(WaitForSelectorState.VISIBLE).setTimeout((double) perAttemptTimeoutMs));
        boolean done = actionFn.run(locator);
        if (done) {
          System.out.println("[AutoHealing] Action '" + action + "' succeeded with candidate #" + (i + 1) + ": " + describe(locator));
          return;
        }
      } catch (PlaywrightException e) {
        last = e;
        System.out.println("[AutoHealing] Candidate #" + (i + 1) + " failed for action '" + action + "': " + e.getMessage());
      }
    }
    throw new PlaywrightException("AutoHealing failed for action '" + action + "' after " + candidates.size() + " candidates", last);
  }

  private String describe(Locator locator) {
    try {
      return locator.toString();
    } catch (Exception e) {
      return locator.getClass().getSimpleName();
    }
  }
}
```

Usage example:

```java
try(Playwright pw = Playwright.create()){
Browser browser = pw.chromium().launch();
Page page = browser.newContext().newPage();
  page.

navigate("https://example.com");

AutoHealing heal = new AutoHealing(page, 2000);

  heal.

click(
  // Primary: semantic locator
    () ->page.

getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().

setName("Continue")),
  // Fallback: test id
  ()->page.

getByTestId("continue"),
// Fallback: visible text
    ()->page.

getByText("Continue")
  );
    }
```

Notes:

- This utility does not modify Playwright internals; it simply orchestrates multiple locator attempts with time-limited
  tries and logging.
- Keep fallbacks few and stable (e.g., role -> testId -> text). Avoid brittle CSS/XPath as a generic catch-all.

## 6. Comparison with Selenium ecosystem

- Selenium + Auto-healing tools: Some third-party tools plug into Selenium to guess or re-learn selectors at runtime.
  This can be useful but also risks hiding real UI changes.
- Playwright: Encourages writing resilient selectors up front (semantics, test IDs) and relies on auto-waiting +
  retries, avoiding implicit selector mutation.

## 7. FAQs and limitations

- Does Playwright change my selector strings automatically? No.
- Can I build my own healing strategy? Yes—compose locator candidates as shown above, but prefer improving selectors
  first.
- What’s the best first fallback? Test IDs or ARIA Role+Name, because they’re intentional and maintainable.
- Will this slow down my tests? Slightly, if primary selectors fail often. Keep per-attempt timeouts short and fix flaky
  selectors.

In summary, Playwright does not ship with a hidden auto-healing engine. It provides robust locator semantics and
auto-waiting so you rarely need one. If you must, a small, explicit fallback helper like the above can be added to your
project safely.
