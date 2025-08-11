# Playwright-Java Automation Framework Documentation

## 1. Overall Architecture and Project Structure

Playwright-Java is a modern browser automation library that provides a high-level API for controlling Chromium, Firefox,
and WebKit browsers. The architecture follows a client-server model where the Java client communicates with a separate
driver process that controls the browsers.

### 1.1 Core Architectural Components

- **Playwright**: The main entry point for the framework, providing access to browser types
- **BrowserType**: Represents a browser type (Chromium, Firefox, WebKit) and provides methods to launch browser
  instances
- **Browser**: Represents a browser instance and provides methods to create contexts
- **BrowserContext**: Represents an isolated browser session with its own cookies, cache, and storage
- **Page**: Represents a browser page/tab and provides methods for interacting with web pages
- **Frame**: Represents a frame within a page
- **ElementHandle**: Represents a DOM element in the page
- **Locator**: Modern way to locate and interact with elements, replacing ElementHandle for most use cases
- **JSHandle**: Represents a JavaScript object in the page

### 1.2 Project Structure

This repository is the Playwright-Java library itself. It does not contain application-specific Page Objects ("pages").
Those are typically maintained in your own automation project that consumes this library.

Where to maintain pages:

- In your consumer automation project, keep Page Object Models under src/main/java/.../pages (recommended) and your test
  classes under src/test/java/.../tests.
- In this repository, any demo/sample code (including example pages or tests) belongs under the examples module.

Recommended structure for your automation project (where you maintain pages):

```
src/
├── main/
│   ├── java/
│   │   └── com/
│   │       └── yourcompany/
│   │           ├── base/           # Base classes and framework foundation
│   │           │   ├── BasePage.java
│   │           │   ├── BaseTest.java
│   │           │   └── PlaywrightFactory.java
│   │           ├── config/         # Configuration management
│   │           │   ├── ConfigManager.java
│   │           │   └── Environment.java
│   │           ├── pages/          # Page Object Models (maintain your pages here)
│   │           │   ├── LoginPage.java
│   │           │   └── HomePage.java
│   │           ├── utils/          # Utility classes
│   │           │   ├── Listeners.java
│   │           │   ├── Reporter.java
│   │           │   └── Waits.java
│   │           └── data/           # Test data management
│   │               ├── DataProvider.java
│   │               └── TestData.java
│   └── resources/                  # Configuration files, test data
│       ├── config.properties
│       └── testdata.json
└── test/
    └── java/
        └── com/
            └── yourcompany/
                └── tests/          # Test classes
                    ├── LoginTests.java
                    └── HomePageTests.java
```

Note for this repository (library and examples):

```
examples/
└── src/
    ├── main/
    │   └── java/                    # Example/demo code (optional pages can live here)
    └── test/
        └── java/                    # Example tests
```

This clarifies that you maintain your application pages in your own project under src/main/java/.../pages, while this
repository focuses on the Playwright-Java library code and ships only examples under the examples module.

## 2. High-Level Architectural Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Test Framework Layer                          │
│  ┌─────────────┐  ┌────────────┐  ┌───────────┐  ┌───────────────┐  │
│  │ Test Classes│  │Test Runners│  │ Listeners │  │ Data Providers│  │
│  └─────────────┘  └────────────┘  └───────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────────┐
│                        Page Object Layer                             │
│  ┌─────────────┐  ┌────────────┐  ┌───────────┐  ┌───────────────┐  │
│  │  Page       │  │ Components │  │ Fragments │  │ Base Page     │  │
│  │  Objects    │  │            │  │           │  │ Classes       │  │
│  └─────────────┘  └────────────┘  └───────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────────┐
│                        Framework Core Layer                          │
│  ┌─────────────┐  ┌────────────┐  ┌───────────┐  ┌───────────────┐  │
│  │ Playwright  │  │ Browser    │  │ Context   │  │ Configuration │  │
│  │ Factory     │  │ Manager    │  │ Manager   │  │ Manager       │  │
│  └─────────────┘  └────────────┘  └───────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────────┐
│                        Playwright API Layer                          │
│  ┌─────────────┐  ┌────────────┐  ┌───────────┐  ┌───────────────┐  │
│  │ Playwright  │  │ Browser    │  │ Browser   │  │ Page          │  │
│  │ Instance    │  │ Type       │  │ Context   │  │               │  │
│  └─────────────┘  └────────────┘  └───────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────────┐
│                        Driver Process Layer                          │
│  ┌─────────────┐  ┌────────────┐  ┌───────────┐                     │
│  │ Connection  │  │ Transport  │  │ Channel   │                     │
│  │             │  │            │  │ Owner     │                     │
│  └─────────────┘  └────────────┘  └───────────┘                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────────┐
│                        Browser Instances                             │
│  ┌─────────────┐  ┌────────────┐  ┌───────────┐                     │
│  │ Chromium    │  │ Firefox    │  │ WebKit    │                     │
│  │             │  │            │  │           │                     │
│  └─────────────┘  └────────────┘  └───────────┘                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 2.1 Component Definitions by Layer

### Framework Core Layer

- Playwright Factory: A framework-owned helper responsible for creating and managing lifecycle-scoped instances of
  Playwright, Browser, BrowserContext, and Page. It centralizes initialization, configuration application (e.g.,
  headless, video, viewport), and cleanup across tests. Example: PlaywrightFactory in section 3.1.1.
- Browser Manager: A framework component (often part of the factory) that decides which browser to launch (
  Chromium/Firefox/WebKit), applies launch options, and exposes a ready Browser instance. It may implement ThreadLocal
  scoping for parallel runs.
- Context Manager: Creates and configures BrowserContext objects to isolate sessions. It applies context options (
  viewport, storage state, permissions, tracing/video) and provides new Pages. It also owns closing contexts and pages
  after use.
- Configuration Manager: Central configuration access point that reads properties (browser, headless, baseUrl, timeouts,
  output paths). It provides typed getters and sensible defaults so the rest of the framework remains decoupled from
  property sources. Example: ConfigManager in section 3.2.1.

### Playwright API Layer

- Playwright Instance: The top-level client entry point created via Playwright.create(). It exposes BrowserType
  handles (chromium(), firefox(), webkit()) and coordinates the connection to the driver process.
- Browser Type: A descriptor for a specific browser family (Chromium/Firefox/WebKit) that can launch Browser processes
  with given LaunchOptions. It does not represent a running browser itself.
- Browser: A handle to a launched browser process. From a Browser you create isolated BrowserContext instances. Closing
  the Browser ends all contexts/pages within it.
- BrowserContext: An isolated browser session (akin to an incognito profile) with its own cookies, storage, cache, and
  permissions. Contexts host Pages, support tracing/video, and enable strong test isolation and parallelism.
- Page: A single tab within a BrowserContext. Provides high-level actions (navigate, click, fill, assertions), modern
  Locator API, and auto-waiting behavior for robust interactions.

### Driver Process Layer

- Connection: The client-side controller that maintains the session with the Playwright driver process. It routes
  requests/responses/events between the Java API objects and the underlying driver.
- Transport: The low-level messaging channel used by Connection to send and receive protocol messages to/from the
  Playwright driver (e.g., over stdio/WebSocket depending on runtime). It ensures reliable, ordered delivery.
- Channel Owner: A base abstraction for API objects that are bound to a protocol channel (e.g., Browser, BrowserContext,
  Page). It tracks identity, lifecycle, and message dispatch to the correct remote endpoint.

### Browser Instances

- Chromium: Google Chromium-based engine used to power Chrome/Edge-like automation. Offers CDP features and broad web
  compatibility.
- Firefox: Mozilla Firefox engine. Provides parity features through Playwright’s unified API with Firefox-specific
  capabilities where applicable.
- WebKit: Apple’s WebKit engine (Safari family). Enables cross-engine coverage to catch engine-specific issues and
  ensure true cross-browser validation.

## 3. Detailed Module Explanations

### 3.1 Base Module

The Base module provides the foundation classes for the framework.

#### 3.1.1 PlaywrightFactory

This class is responsible for initializing Playwright, launching browsers, and creating contexts and pages.

```java
public class PlaywrightFactory {
  private static ThreadLocal<Playwright> playwrightThreadLocal = new ThreadLocal<>();
  private static ThreadLocal<Browser> browserThreadLocal = new ThreadLocal<>();
  private static ThreadLocal<BrowserContext> contextThreadLocal = new ThreadLocal<>();
  private static ThreadLocal<Page> pageThreadLocal = new ThreadLocal<>();

  public static Playwright getPlaywright() {
    if (playwrightThreadLocal.get() == null) {
      playwrightThreadLocal.set(Playwright.create());
    }
    return playwrightThreadLocal.get();
  }

  public static Browser getBrowser() {
    if (browserThreadLocal.get() == null) {
      // Read browser type from configuration
      String browserType = ConfigManager.getInstance().getBrowser().toLowerCase();

      // Launch browser based on configuration
      switch (browserType) {
        case "chromium":
          browserThreadLocal.set(getPlaywright().chromium().launch(
            new BrowserType.LaunchOptions()
              .setHeadless(ConfigManager.getInstance().isHeadless())
          ));
          break;
        case "firefox":
          browserThreadLocal.set(getPlaywright().firefox().launch(
            new BrowserType.LaunchOptions()
              .setHeadless(ConfigManager.getInstance().isHeadless())
          ));
          break;
        case "webkit":
          browserThreadLocal.set(getPlaywright().webkit().launch(
            new BrowserType.LaunchOptions()
              .setHeadless(ConfigManager.getInstance().isHeadless())
          ));
          break;
        default:
          throw new IllegalArgumentException("Invalid browser type: " + browserType);
      }
    }
    return browserThreadLocal.get();
  }

  public static BrowserContext getContext() {
    if (contextThreadLocal.get() == null) {
      // Create a new context with specific options
      contextThreadLocal.set(getBrowser().newContext(
        new Browser.NewContextOptions()
          .setViewportSize(1920, 1080)
          .setRecordVideoDir(ConfigManager.getInstance().getVideoRecordingPath())
          .setAcceptDownloads(true)
      ));
    }
    return contextThreadLocal.get();
  }

  public static Page getPage() {
    if (pageThreadLocal.get() == null) {
      pageThreadLocal.set(getContext().newPage());
    }
    return pageThreadLocal.get();
  }

  public static void closePlaywright() {
    if (pageThreadLocal.get() != null) {
      pageThreadLocal.get().close();
      pageThreadLocal.remove();
    }
    if (contextThreadLocal.get() != null) {
      contextThreadLocal.get().close();
      contextThreadLocal.remove();
    }
    if (browserThreadLocal.get() != null) {
      browserThreadLocal.get().close();
      browserThreadLocal.remove();
    }
    if (playwrightThreadLocal.get() != null) {
      playwrightThreadLocal.get().close();
      playwrightThreadLocal.remove();
    }
  }
}
```

#### 3.1.2 BasePage

The BasePage class provides common functionality for all page objects.

```java
public abstract class BasePage {
  protected Page page;
  protected String pageUrl;
  protected String pageTitle;

  public BasePage(Page page) {
    this.page = page;
  }

  // Common methods for all pages
  public String getPageTitle() {
    return page.title();
  }

  public void navigateTo() {
    page.navigate(pageUrl);
  }

  // Wrapper methods for common Playwright actions
  protected void click(String selector) {
    page.click(selector);
  }

  protected void fill(String selector, String text) {
    page.fill(selector, text);
  }

  protected String getText(String selector) {
    return page.textContent(selector);
  }

  protected boolean isVisible(String selector) {
    return page.isVisible(selector);
  }

  // Modern Locator-based methods
  protected Locator getLocator(String selector) {
    return page.locator(selector);
  }

  protected void clickLocator(Locator locator) {
    locator.click();
  }

  protected void fillLocator(Locator locator, String text) {
    locator.fill(text);
  }

  // Wait methods
  protected void waitForSelector(String selector) {
    page.waitForSelector(selector);
  }

  protected void waitForNavigation(Runnable action) {
    page.waitForNavigation(() -> action.run());
  }
}
```

#### 3.1.3 BaseTest

The BaseTest class provides the foundation for all test classes.

```java
public class BaseTest {
  protected Playwright playwright;
  protected Browser browser;
  protected BrowserContext context;
  protected Page page;

  @BeforeClass
  public void setUp() {
    playwright = PlaywrightFactory.getPlaywright();
    browser = PlaywrightFactory.getBrowser();
    context = PlaywrightFactory.getContext();
    page = PlaywrightFactory.getPage();
  }

  @AfterClass
  public void tearDown() {
    PlaywrightFactory.closePlaywright();
  }

  @BeforeMethod
  public void beforeMethod(Method method) {
    // Set up tracing for each test method
    context.tracing().start(new Tracing.StartOptions()
      .setScreenshots(true)
      .setSnapshots(true)
      .setSources(true));
  }

  @AfterMethod
  public void afterMethod(ITestResult result) {
    // Stop tracing and save trace file with test name
    String testName = result.getMethod().getMethodName();
    context.tracing().stop(new Tracing.StopOptions()
      .setPath(Paths.get("traces/" + testName + ".zip")));

    // Take screenshot on failure
    if (result.getStatus() == ITestResult.FAILURE) {
      String screenshotPath = "screenshots/" + testName + ".png";
      page.screenshot(new Page.ScreenshotOptions().setPath(Paths.get(screenshotPath)));
    }
  }
}
```

### 3.2 Config Module

The Config module handles configuration management for the framework.

#### 3.2.1 ConfigManager

```java
public class ConfigManager {
  private static ConfigManager instance;
  private Properties properties;

  private ConfigManager() {
    properties = new Properties();
    try (InputStream input = getClass().getClassLoader().getResourceAsStream("config.properties")) {
      properties.load(input);
    } catch (IOException e) {
      e.printStackTrace();
    }
  }

  public static synchronized ConfigManager getInstance() {
    if (instance == null) {
      instance = new ConfigManager();
    }
    return instance;
  }

  public String getBrowser() {
    return properties.getProperty("browser", "chromium");
  }

  public boolean isHeadless() {
    return Boolean.parseBoolean(properties.getProperty("headless", "true"));
  }

  public String getBaseUrl() {
    return properties.getProperty("baseUrl", "https://example.com");
  }

  public Path getVideoRecordingPath() {
    String path = properties.getProperty("videoRecordingPath", "videos");
    return Paths.get(path);
  }

  public int getDefaultTimeout() {
    return Integer.parseInt(properties.getProperty("defaultTimeout", "30000"));
  }
}
```

### 3.3 Pages Module

The Pages module contains Page Object Models for different pages of the application.

#### 3.3.1 LoginPage

```java
public class LoginPage extends BasePage {
  // Locators
  private String usernameInput = "#username";
  private String passwordInput = "#password";
  private String loginButton = "#login-button";
  private String errorMessage = ".error-message";

  // Constructor
  public LoginPage(Page page) {
    super(page);
    this.pageUrl = ConfigManager.getInstance().getBaseUrl() + "/login";
    this.pageTitle = "Login Page";
  }

  // Page actions
  public HomePage login(String username, String password) {
    fill(usernameInput, username);
    fill(passwordInput, password);

    // Wait for navigation after clicking login button
    waitForNavigation(() -> click(loginButton));

    return new HomePage(page);
  }

  public boolean isErrorMessageDisplayed() {
    return isVisible(errorMessage);
  }

  public String getErrorMessage() {
    return getText(errorMessage);
  }

  // Modern Locator-based approach
  public HomePage loginWithLocators(String username, String password) {
    Locator usernameLocator = page.locator(usernameInput);
    Locator passwordLocator = page.locator(passwordInput);
    Locator loginButtonLocator = page.locator(loginButton);

    usernameLocator.fill(username);
    passwordLocator.fill(password);

    // Wait for navigation after clicking login button
    waitForNavigation(() -> loginButtonLocator.click());

    return new HomePage(page);
  }
}
```

### 3.4 Utils Module

The Utils module contains utility classes for the framework.

#### 3.4.1 Waits

```java
public class Waits {
  private Page page;

  public Waits(Page page) {
    this.page = page;
  }

  public void waitForPageLoad() {
    page.waitForLoadState(LoadState.DOMCONTENTLOADED);
    page.waitForLoadState(LoadState.NETWORKIDLE);
  }

  public void waitForElement(String selector, WaitForSelectorState state) {
    page.waitForSelector(selector, new Page.WaitForSelectorOptions()
      .setState(state));
  }

  public void waitForElementToBeVisible(String selector) {
    waitForElement(selector, WaitForSelectorState.VISIBLE);
  }

  public void waitForElementToBeHidden(String selector) {
    waitForElement(selector, WaitForSelectorState.HIDDEN);
  }

  public void waitForURL(String url) {
    page.waitForURL(url);
  }

  public void waitForURLToContain(String urlPart) {
    page.waitForURL("**/" + urlPart + "**");
  }

  public void waitForResponse(String url, Runnable action) {
    page.waitForResponse(url, action);
  }
}
```

#### 3.4.2 Reporter

```java
public class Reporter {
  private static final Logger logger = LogManager.getLogger(Reporter.class);

  public static void log(String message) {
    logger.info(message);
  }

  public static void logError(String message) {
    logger.error(message);
  }

  public static void logWarning(String message) {
    logger.warn(message);
  }

  public static void takeScreenshot(Page page, String name) {
    try {
      String screenshotPath = "screenshots/" + name + ".png";
      page.screenshot(new Page.ScreenshotOptions().setPath(Paths.get(screenshotPath)));
      log("Screenshot saved to: " + screenshotPath);
    } catch (Exception e) {
      logError("Failed to take screenshot: " + e.getMessage());
    }
  }

  public static void saveTrace(BrowserContext context, String name) {
    try {
      String tracePath = "traces/" + name + ".zip";
      context.tracing().stop(new Tracing.StopOptions()
        .setPath(Paths.get(tracePath)));
      log("Trace saved to: " + tracePath);
    } catch (Exception e) {
      logError("Failed to save trace: " + e.getMessage());
    }
  }
}
```

## 4. How Playwright-Java Handles Sessions, Parallel Execution, and Browser Context Lifecycles

### 4.1 Session Management

Playwright uses a hierarchical model for session management:

1. **Playwright Instance**: The top-level object that provides access to browser types
2. **Browser Instance**: Represents a browser process
3. **BrowserContext**: Represents an isolated browser session (similar to an incognito window)
4. **Page**: Represents a browser tab within a context

Each BrowserContext is isolated from others, with its own:

- Cookies
- Local Storage
- Session Storage
- Cache
- Browser extensions
- Authentication state

This isolation is crucial for running tests in parallel without interference.

```
// Example of session management
try (Playwright playwright = Playwright.create()) {
    Browser browser = playwright.chromium().launch();

    // Create first isolated context
    BrowserContext context1 = browser.newContext();
    Page page1 = context1.newPage();
    page1.navigate("https://example.com");
    page1.fill("#username", "user1");

    // Create second isolated context (no shared state with context1)
    BrowserContext context2 = browser.newContext();
    Page page2 = context2.newPage();
    page2.navigate("https://example.com");
    // Username field will be empty here, not "user1"

    // Clean up
    context1.close();
    context2.close();
    browser.close();
}
```

### 4.2 Parallel Execution

Playwright-Java supports parallel execution through several mechanisms:

1. **Thread-Safe Design**: The core Playwright API is designed to be thread-safe
2. **Isolated Contexts**: Each test can use its own BrowserContext for isolation
3. **ThreadLocal Storage**: Framework can use ThreadLocal to store browser instances per thread

```java
public class PlaywrightFactory {
  private static ThreadLocal<Playwright> playwrightThreadLocal = new ThreadLocal<>();
  private static ThreadLocal<Browser> browserThreadLocal = new ThreadLocal<>();
  private static ThreadLocal<BrowserContext> contextThreadLocal = new ThreadLocal<>();
  private static ThreadLocal<Page> pageThreadLocal = new ThreadLocal<>();

  // Methods to get thread-specific instances
  // ...
}
```

When running tests in parallel with TestNG:

```java

@Test(threadPoolSize = 3, invocationCount = 3, timeOut = 30000)
public void parallelTest() {
  // Each thread gets its own Playwright, Browser, Context, and Page instances
  Page page = PlaywrightFactory.getPage();
  page.navigate("https://example.com");
  // Test actions...
}
```

### 4.3 Browser Context Lifecycle

The lifecycle of a BrowserContext typically follows these stages:

1. **Creation**: Created from a Browser instance with specific options
2. **Configuration**: Set up with options like viewport size, geolocation, permissions
3. **Usage**: Used to create and interact with Pages
4. **Cleanup**: Closed to release resources

```
// Example of BrowserContext lifecycle
try (Playwright playwright = Playwright.create()) {
    Browser browser = playwright.chromium().launch();

    // 1. Creation
    BrowserContext context = browser.newContext(
        new Browser.NewContextOptions()
            .setViewportSize(1920, 1080)
            .setLocale("en-US")
            .setTimezoneId("America/New_York")
            .setGeolocation(41.890221, 12.492348)
            .setPermissions(Arrays.asList("geolocation"))
    );

    // 2. Configuration
    context.setDefaultTimeout(30000);
    context.setExtraHTTPHeaders(new HashMap<String, String>() {{
        put("Accept-Language", "en-US");
    }});

    // Start tracing
    context.tracing().start(new Tracing.StartOptions()
        .setScreenshots(true)
        .setSnapshots(true));

    // 3. Usage
    Page page = context.newPage();
    page.navigate("https://example.com");
    // Interact with the page...

    // 4. Cleanup
    context.tracing().stop(new Tracing.StopOptions()
        .setPath(Paths.get("trace.zip")));
    context.close();
    browser.close();
}
```

## 5. Comparison with Selenium-TestNG or Selenium-JUnit Projects

### 5.1 Architecture Differences

| Feature             | Playwright                                          | Selenium                                            |
|---------------------|-----------------------------------------------------|-----------------------------------------------------|
| **Architecture**    | Client-server model with a dedicated driver process | Client-server model with separate WebDriver servers |
| **Browser Control** | Direct control via CDP (Chrome DevTools Protocol)   | Control via WebDriver protocol                      |
| **Process Model**   | Single driver process for all browsers              | Separate WebDriver server for each browser          |
| **Browser Support** | Chromium, Firefox, WebKit (Safari)                  | Chrome, Firefox, Edge, Safari, IE                   |
| **Mobile Support**  | Emulation only                                      | Real devices via Appium                             |
| **API Design**      | Modern, fluent API with async support               | Traditional, more verbose API                       |
| **Auto-Waiting**    | Built-in auto-waiting for elements                  | Requires explicit waits                             |

### 5.2 Execution Model Differences

| Feature                | Playwright                                         | Selenium                                             |
|------------------------|----------------------------------------------------|------------------------------------------------------|
| **Browser Launch**     | Fast browser launch with persistent browser option | Slower browser launch                                |
| **Test Isolation**     | BrowserContext for isolated sessions               | New browser instance or clearing cookies             |
| **Parallel Execution** | Efficient with shared browser instances            | Resource-intensive with separate WebDriver instances |
| **Resource Usage**     | Lower memory footprint                             | Higher memory footprint                              |
| **Speed**              | Generally faster execution                         | Generally slower execution                           |
| **Stability**          | More stable with auto-waiting                      | Less stable, requires explicit waits                 |

### 5.3 Design Pattern Differences

| Feature              | Playwright                             | Selenium                                     |
|----------------------|----------------------------------------|----------------------------------------------|
| **Element Location** | Locator API with built-in retry logic  | FindElement/FindElements with explicit waits |
| **Page Objects**     | Modern approach with Locator objects   | Traditional approach with WebElement         |
| **Waits**            | Auto-waiting built into actions        | Explicit waits with WebDriverWait            |
| **Actions**          | High-level actions (click, fill, etc.) | Low-level actions via Actions class          |
| **Assertions**       | Built-in assertions with retry logic   | Requires external assertion libraries        |

### 5.4 Code Comparison

#### Selenium Example:

```java
public class LoginPage {
  private WebDriver driver;
  private WebDriverWait wait;

  // Locators
  private By usernameInput = By.id("username");
  private By passwordInput = By.id("password");
  private By loginButton = By.id("login-button");

  public LoginPage(WebDriver driver) {
    this.driver = driver;
    this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
  }

  public void login(String username, String password) {
    wait.until(ExpectedConditions.visibilityOfElementLocated(usernameInput));
    driver.findElement(usernameInput).clear();
    driver.findElement(usernameInput).sendKeys(username);

    wait.until(ExpectedConditions.visibilityOfElementLocated(passwordInput));
    driver.findElement(passwordInput).clear();
    driver.findElement(passwordInput).sendKeys(password);

    wait.until(ExpectedConditions.elementToBeClickable(loginButton));
    driver.findElement(loginButton).click();

    wait.until(ExpectedConditions.urlContains("dashboard"));
  }
}
```

#### Playwright Example:

```java
public class LoginPage extends BasePage {
  // Locators
  private String usernameInput = "#username";
  private String passwordInput = "#password";
  private String loginButton = "#login-button";

  public LoginPage(Page page) {
    super(page);
  }

  public void login(String username, String password) {
    // Auto-waiting built in, no explicit waits needed
    page.fill(usernameInput, username);
    page.fill(passwordInput, password);
    page.click(loginButton);

    // Wait for navigation to complete
    page.waitForURL("**/dashboard");
  }

  // Modern Locator-based approach
  public void loginWithLocators(String username, String password) {
    Locator usernameLocator = page.locator(usernameInput);
    Locator passwordLocator = page.locator(passwordInput);
    Locator loginButtonLocator = page.locator(loginButton);

    usernameLocator.fill(username);
    passwordLocator.fill(password);
    loginButtonLocator.click();

    page.waitForURL("**/dashboard");
  }
}
```

### 5.5 Configuration and Test Execution Flow Differences

#### Selenium Configuration:

```java
public class WebDriverFactory {
  public static WebDriver createDriver() {
    String browser = System.getProperty("browser", "chrome");

    switch (browser.toLowerCase()) {
      case "chrome":
        WebDriverManager.chromedriver().setup();
        return new ChromeDriver();
      case "firefox":
        WebDriverManager.firefoxdriver().setup();
        return new FirefoxDriver();
      case "edge":
        WebDriverManager.edgedriver().setup();
        return new EdgeDriver();
      default:
        throw new IllegalArgumentException("Browser not supported: " + browser);
    }
  }
}

public class BaseTest {
  protected WebDriver driver;

  @BeforeMethod
  public void setUp() {
    driver = WebDriverFactory.createDriver();
    driver.manage().window().maximize();
    driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(10));
  }

  @AfterMethod
  public void tearDown() {
    if (driver != null) {
      driver.quit();
    }
  }
}
```

#### Playwright Configuration:

```java
public class PlaywrightFactory {
  public static Playwright getPlaywright() {
    return Playwright.create();
  }

  public static Browser getBrowser() {
    String browserName = System.getProperty("browser", "chromium");
    Playwright playwright = getPlaywright();

    switch (browserName.toLowerCase()) {
      case "chromium":
        return playwright.chromium().launch(
          new BrowserType.LaunchOptions().setHeadless(false)
        );
      case "firefox":
        return playwright.firefox().launch(
          new BrowserType.LaunchOptions().setHeadless(false)
        );
      case "webkit":
        return playwright.webkit().launch(
          new BrowserType.LaunchOptions().setHeadless(false)
        );
      default:
        throw new IllegalArgumentException("Browser not supported: " + browserName);
    }
  }

  public static BrowserContext createContext(Browser browser) {
    return browser.newContext(
      new Browser.NewContextOptions()
        .setViewportSize(1920, 1080)
        .setRecordVideoDir(Paths.get("videos/"))
    );
  }
}

public class BaseTest {
  protected Playwright playwright;
  protected Browser browser;
  protected BrowserContext context;
  protected Page page;

  @BeforeMethod
  public void setUp() {
    playwright = PlaywrightFactory.getPlaywright();
    browser = PlaywrightFactory.getBrowser();
    context = PlaywrightFactory.createContext(browser);
    page = context.newPage();
  }

  @AfterMethod
  public void tearDown() {
    if (context != null) {
      context.close();
    }
    if (browser != null) {
      browser.close();
    }
    if (playwright != null) {
      playwright.close();
    }
  }
}
```

## 6. Conclusion

Playwright-Java provides a modern, powerful framework for browser automation with several advantages over traditional
Selenium-based frameworks:

1. **Improved Architecture**: Client-server model with a single driver process
2. **Better Performance**: Faster execution and lower resource usage
3. **Enhanced Stability**: Auto-waiting and retry mechanisms built-in
4. **Modern API**: Fluent, easy-to-use API with powerful features
5. **Isolated Contexts**: Better support for parallel execution
6. **Advanced Features**: Built-in tracing, video recording, and network interception

When designing a Playwright-Java automation framework, focus on:

1. **Clean Architecture**: Separate concerns with proper layering
2. **Thread Safety**: Use ThreadLocal for parallel execution
3. **Page Object Pattern**: Use modern Locator-based approach
4. **Configuration Management**: Flexible, environment-aware configuration
5. **Reporting and Debugging**: Leverage Playwright's tracing and screenshot capabilities

By following these guidelines, you can create a robust, maintainable, and efficient test automation framework using
Playwright-Java.

## 7. Optional: Playwright MCP Server (AI Agent Integration)

### 7.1 What is the Playwright MCP Server?

The Playwright MCP server is an optional, separate component that exposes Playwright browser automation capabilities
over the Model Context Protocol (MCP). MCP is a protocol designed to let AI assistants or agent frameworks call tools in
a structured way. With the Playwright MCP server, an AI agent can request browser actions (for example, open a page,
click, fill, capture screenshots) via standardized tool calls.

### 7.2 Do you need it for Playwright‑Java?

- Short answer: No, not for typical automation or testing use cases.
- This repository provides the Playwright‑Java library and example tests. You do not need the MCP server to write or run
  Java tests using the standard Playwright API.
- Consider the MCP server only if you’re integrating AI assistants/agents that must control a real browser using
  Playwright as a tool backend.

### 7.3 How it relates to this repository

- Separation of concerns: The MCP server is not part of the Playwright‑Java library and is not used by the examples in
  this repo.
- Different runtime: The official MCP server implementations are typically separate processes/services. They communicate
  with AI agents over MCP and with browsers via Playwright. Your Java tests continue to use the Java API directly and
  are unrelated to the MCP runtime.
- Not a replacement for the driver: The MCP server is unrelated to the internal Playwright driver process used by this
  library; it does not replace the Java API, test runners, or Browser/Context/Page lifecycles described above.

### 7.4 When to consider the MCP server

- You are building an AI assistant or agent that needs safe, constrained access to a real browser for tasks like
  navigation, form filling, scraping, or validation.
- You want a protocol‑based boundary (MCP) between the agent runtime and browser control, with explicit tool definitions
  and auditing.

### 7.5 High‑level usage flow (for AI/agent integrations)

1. Run an MCP server that exposes Playwright capabilities as tools (separate from your Java tests).
2. Configure your AI assistant/agent framework to connect to that MCP server and call its tools.
3. The MCP server executes requested browser actions using Playwright and returns structured results (text, artifacts
   like screenshots, etc.).

Note: Exact setup, available tools, authentication, and network/security controls depend on the specific MCP server
implementation and agent framework you use. Refer to the MCP server’s README and official Playwright documentation for
up‑to‑date instructions.

### 7.6 Security and operational considerations

- Treat the MCP server as a powerful automation endpoint; restrict access and scope of allowed operations.
- Run it in a controlled environment (e.g., container or sandbox) with appropriate network/file system permissions.
- Log, audit, and rate‑limit tool calls from agents; prefer least privilege.

### 7.7 References

- Official Playwright documentation (Automation concepts, Browser/Context lifecycles, Tracing, etc.).
- Playwright MCP server repository/documentation (for installation, configuration, and tool reference).

If you are not building AI/agent features, you can safely ignore this section. Your Playwright‑Java automation will work
as described in the earlier sections without any MCP components.

## 8. Playwright Trace Viewer: What It Is, Why You Need It, and How It Compares to Selenium

Note: A standalone version of this guide is available at docs/trace-viewer.md

### 8.1 What is the Playwright Trace Viewer?

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

### 8.2 Why you need it

Trace Viewer is particularly valuable for:

- Debugging flaky tests: Reproduce the exact sequence and timing of actions, waits, and network events without rerunning
  the test.
- CI triage: Collect a single artifact per test (trace.zip) that teammates can download and inspect locally with full
  context.
- Root-cause analysis: Inspect console errors, failed network requests, locator resolutions, and timing information
  alongside screenshots.
- Faster feedback: Instead of adding ad-hoc logging or guesswork, open the trace and immediately see what happened.

### 8.3 Enabling tracing in Playwright Java

You can control tracing per BrowserContext. Typical patterns:

1) Record a trace for the whole test and save at the end

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

2) Record only on failure or on first retry

- If you use a test framework (JUnit/TestNG) with custom listeners, you can start tracing in @BeforeEach and only save
  the trace in @AfterEach if the test failed.
- In Playwright Test (JS/TS), this is built-in via config (trace: 'on-first-retry' or 'retain-on-failure'); in Java,
  implement similar logic in your test harness.

3) Set a default traces directory (optional)

- You can set BrowserType.LaunchOptions.setTracesDir(...) so intermediate chunks are stored predictably during long
  runs. Still pass StopOptions.setPath(...) to name the final .zip.

### 8.4 Viewing traces

- Using the CLI (recommended):
  - Install Playwright CLI (bundled with Playwright):
    - Run: npx playwright show-trace trace.zip
  - This opens an interactive UI to replay all actions, inspect network, console, and locator details.
- Using the Web UI:
  - You can also open trace.zip in the hosted Trace Viewer (see official docs) when internet and security policies
    allow.

Tip: Attach the trace.zip to your CI job artifacts so it’s available for download when tests fail.

### 8.5 Good practices

- Enable screenshots and DOM snapshots along with tracing for maximum context.
- Retain traces only on failure to control artifact size; enable always-on tracing temporarily while diagnosing
  flakiness.
- Name traces consistently by test name and timestamp to simplify triage.
- Consider also enabling video on the BrowserContext for a complementary, real-time view.

### 8.6 How does Trace Viewer compare to Selenium?

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

### 8.7 Troubleshooting

- Large trace files: Use retain-on-failure patterns or stop/start tracing around critical parts only.
- Missing data: Ensure you called .setScreenshots(true) and .setSnapshots(true) when starting tracing; also consider
  .setSources(true) for code links.
- CLI not found: Ensure Node.js is available so npx playwright show-trace works, or invoke the Playwright CLI
  distributed with your environment.

