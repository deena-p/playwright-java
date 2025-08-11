# Playwright Trace Viewer Guide (Java)

Last updated: 2025-08-10

This standalone guide explains what the Playwright Trace Viewer is, why it’s valuable, how to enable tracing in
Playwright Java, how to open and analyze traces, best practices, a comparison with Selenium, and troubleshooting tips.

- What is the Trace Viewer
- Why you need it
- Enabling tracing in Playwright Java
- Viewing traces
- Good practices
- How it compares to Selenium
- Troubleshooting
- Screenshots vs Snapshots

## 1. What is the Playwright Trace Viewer?

Playwright Trace Viewer is an interactive, time-travel debug tool that lets you replay your test execution step-by-step
after the run completes, either locally or in CI. A trace is a compact artifact (typically a .zip) that includes:

- Network activity and console logs
- Screenshots and DOM snapshots per action
- Source code timeline of actions and assertions
- Video (if enabled)
- Attachments (e.g., arbitrary files, locator descriptions)

You can open a trace using the Playwright CLI or the web-based viewer to visually inspect each action, see what was on
the screen, inspect selectors, and understand why a test failed.

References:

- Official docs: https://playwright.dev/java/docs/trace-viewer
- CLI reference: https://playwright.dev/docs/trace-viewer#viewing-the-trace

## 2. Why you need it

Trace Viewer is particularly valuable for:

- Debugging flaky tests: Reproduce the exact sequence and timing of actions, waits, and network events without rerunning
  the test.
- CI triage: Collect a single artifact per test (trace.zip) that teammates can download and inspect locally with full
  context.
- Root-cause analysis: Inspect console errors, failed network requests, locator resolutions, and timing information
  alongside screenshots.
- Faster feedback: Instead of adding ad-hoc logging or guesswork, open the trace and immediately see what happened.

## 3. Enabling tracing in Playwright Java

You can control tracing per BrowserContext. Typical patterns:

### 3.1 Record a trace for the whole test and save at the end

```java
import com.microsoft.playwright.*;
import java.nio.file.Paths;

public class TraceExample {
  public static void main(String[] args) {
    try (Playwright pw = Playwright.create()) {
      Browser browser = pw.chromium().launch(new BrowserType.LaunchOptions().setHeadless(true));
      BrowserContext context = browser.newContext();

      // Start tracing with screenshots, DOM snapshots and source
      context.tracing().start(new Tracing.StartOptions()
          .setScreenshots(true)
          .setSnapshots(true)
          .setSources(true));

      Page page = context.newPage();
      page.navigate("https://example.com");
      page.getByRole(AriaRole.LINK, new Page.GetByRoleOptions().setName("More information"))
          .click();

      // Stop and export the trace
      context.tracing().stop(new Tracing.StopOptions()
          .setPath(Paths.get("trace.zip")));

      context.close();
      browser.close();
    }
  }
}
```

### 3.2 Record only on failure or on first retry

- If you use a test framework (JUnit/TestNG) with custom listeners, start tracing in @BeforeEach and only save the trace
  in @AfterEach if the test failed.
- In Playwright Test (JS/TS), this is built-in via config (trace: 'on-first-retry' or 'retain-on-failure'); in Java,
  implement similar logic in your test harness.

### 3.3 Set a default traces directory (optional)

- You can set BrowserType.LaunchOptions.setTracesDir(...) so intermediate chunks are stored predictably during long
  runs. Still pass StopOptions.setPath(...) to name the final .zip.

## 4. Viewing traces

- Using the CLI (recommended):
  - Install Playwright CLI (bundled with Playwright):
    - Run: npx playwright show-trace trace.zip
  - This opens an interactive UI to replay all actions, inspect network, console, and locator details.
- Using the Web UI:
  - You can also open trace.zip in the hosted Trace Viewer (see official docs) when internet and security policies
    allow.

Tip: Attach the trace.zip to your CI job artifacts so it’s available for download when tests fail.

## 5. Good practices

- Enable screenshots and DOM snapshots along with tracing for maximum context.
- Retain traces only on failure to control artifact size; enable always-on tracing temporarily while diagnosing
  flakiness.
- Name traces consistently by test name and timestamp to simplify triage.
- Consider also enabling video on the BrowserContext for a complementary, real-time view.

## 6. How does Trace Viewer compare to Selenium?

Selenium (WebDriver) is a long-standing standard for browser automation. While Selenium 4 added modern features (BiDi,
DevTools integrations, improved Grid, OpenTelemetry support), it does not ship with a first-class, built-in trace viewer
that provides the same unified time-travel experience as Playwright’s Trace Viewer.

High-level comparison:

- Unified time-travel artifact:
  - Playwright: First-class tracing artifact (trace.zip) with actions, screenshots, DOM snapshots, network, console, and
    code timeline in a single UI.
  - Selenium: Typically relies on a combination of logs, screenshots, and (optionally) videos captured via third-party
    libraries or vendor services. No official, integrated time-travel viewer across all actions by default.

- Auto-waiting and step context:
  - Playwright: Actions emit detailed steps with auto-waiting context embedded in the trace; you can see exactly
    why/when an action waited or retried.
  - Selenium: You infer from logs and code; auto-waiting is not built-in across all actions. You often manage waits
    yourself or with helper libraries.

- CI triage workflow:
  - Playwright: Store trace.zip as a CI artifact; teammates open it locally with playwright show-trace and immediately
    replay the run.
  - Selenium: Commonly depends on vendor platforms (e.g., cloud grids) for session replays, or custom pipelines
    assembling logs/screenshots/videos.

- DevTools/BiDi insights:
  - Playwright: Network and console events are integrated into the trace viewer UI.
  - Selenium: DevTools/BiDi events can be captured but are not visualized in a unified, standardized viewer by default.

- Ecosystem/vendor lock-in:
  - Playwright: Trace Viewer is open-source and works locally without a paid service.
  - Selenium: Comparable, polished replay UIs typically come from third-party services/grids; capabilities vary.

Conclusion: If reproducible, actionable debugging and fast CI triage are priorities, Playwright’s Trace Viewer provides
an out-of-the-box, integrated solution. In Selenium ecosystems, you can approximate similar visibility, but it usually
requires assembling multiple tools and services.

## 7. Troubleshooting

- Large trace files: Use retain-on-failure patterns or stop/start tracing around critical parts only.
- Missing data: Ensure you called .setScreenshots(true) and .setSnapshots(true) when starting tracing; also consider
  .setSources(true) for code links.
- CLI not found: Ensure Node.js is available so npx playwright show-trace works, or invoke the Playwright CLI
  distributed with your environment.

## 8. Screenshots vs Snapshots in Trace Viewer

Understanding the difference helps you choose which options to enable when starting tracing:

- Screenshot
  - What it is: A rasterized image (pixels) of the page at a point in time.
  - How to enable: Tracing.StartOptions.setScreenshots(true).
  - What you get in the viewer: Thumbnail images along the timeline and a full-size pixel-perfect capture for selected
    steps.
  - Strengths:
    - Captures exactly what the user would see (fonts, GPU effects, canvas, cross-origin iframes rendered to pixels).
    - Helpful for visual glitches, rendering issues, and CSS/layout regressions.
  - Limitations:
    - Not interactive: you can’t inspect DOM or selectors within a screenshot.
    - Resolution/viewport dependent; increases trace.zip size with each image captured.

- Snapshot (DOM snapshot)
  - What it is: A serialized capture of the page’s DOM (and related state) that the Trace Viewer can deterministically
    re-render.
  - How to enable: Tracing.StartOptions.setSnapshots(true).
  - What you get in the viewer: The “snapshot” rendering that you can interact with—inspect elements, view selectors,
    and see before/after state for each action.
  - Strengths:
    - Interactive: inspect locators, evaluate element states, understand why an action waited/retried.
    - Great for root-cause analysis of selector issues, timing, and state transitions.
  - Limitations:
    - Security constraints may blank or stub cross-origin iframes and certain resources.
    - Rendering may not be pixel-identical (fonts/GPU effects can differ from real screenshot).

How they complement each other

- Use snapshots for step-by-step, interactive debugging and selector inspection.
- Use screenshots for faithful visual evidence of what the browser actually painted.
- Enabling both often gives the best triage experience: setScreenshots(true) + setSnapshots(true).

Tips

- If artifact size is a concern, start with snapshots only; add screenshots temporarily while investigating visual
  issues.
- For apps heavy on cross-origin iframes or canvas/WebGL, screenshots provide visual truth where DOM snapshots may be
  limited.
