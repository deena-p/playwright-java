# Playwright Java API Testing Documentation

## 1. Purpose and Scope

This document describes how to use Playwright Java for API testing, including:

- Core architecture and components for HTTP API automation
- Project structure recommendations
- High-level architectural diagram for API testing
- Component definitions and responsibilities
- Canonical examples (GET/POST, auth, multipart, file ops, assertions)
- Best practices for reliability, parallelism, and CI
- Mixing API and UI testing in one harness
- A detailed comparison with RestAssured (commonalities and differences)

This complements docs like PLAYWRIGHT_JAVA_FRAMEWORK_GUIDE.md (UI/browser) by focusing on API testing with Playwright’s
APIRequestContext.

## 2. High-Level Architecture for API Testing

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            Test Framework Layer                          │
│  ┌───────────────┐  ┌────────────┐  ┌──────────────┐  ┌───────────────┐ │
│  │ Test Classes  │  │ Test Runner│  │ Fixtures/DI  │  │ Reporters/CI  │ │
│  └───────────────┘  └────────────┘  └──────────────┘  └───────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
                                   │
┌──────────────────────────────────────────────────────────────────────────┐
│                          API Abstraction Layer                           │
│  ┌────────────────┐  ┌─────────────────────┐  ┌───────────────────────┐ │
│  │ API Client(s)  │  │ Request Builders    │  │ Response Validators   │ │
│  │ (per service)  │  │ (auth, headers, etc)│  │ (JSON, schema, status)│ │
│  └────────────────┘  └─────────────────────┘  └───────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
                                   │
┌──────────────────────────────────────────────────────────────────────────┐
│                           Playwright API Layer                           │
│  ┌──────────────┐  ┌─────────────────────┐  ┌─────────────────────────┐ │
│  │ Playwright   │  │ APIRequestContext   │  │ APIResponse             │ │
│  │ Instance     │  │ (per test/fixture)  │  │ (body, headers, etc.)  │ │
│  └──────────────┘  └─────────────────────┘  └─────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
                                   │
┌──────────────────────────────────────────────────────────────────────────┐
│                          HTTP(S) Backend Services                        │
│    REST APIs, GraphQL, file endpoints, auth providers, etc.              │
└──────────────────────────────────────────────────────────────────────────┘
```

## 3. Component Definitions

- Playwright: Top-level entry point created via Playwright.create(). Provides access to browser types and the API client
  factory (Playwright.request()).
- APIRequest: Accessor from Playwright to create new APIRequestContext instances.
- APIRequestContext: A lightweight HTTP client encapsulating baseURL, extraHTTPHeaders, storageState (cookies/auth), and
  default request options. Designed for test-level isolation and parallelism.
- APIResponse: Represents the response (status, ok(), headers, body(), text(), json()). Must be disposed when
  appropriate in very long runs to free memory.
- APIRequestOptions: Request-specific options (headers, data, form parameters, multipart, method overrides, timeouts).
- Storage State: Cookies and local storage data persisted to/reused from a JSON file, enabling UI->API or API->UI
  authenticated flows.
- Route Interception (optional for hybrid): BrowserContext/Page.route can mock, modify, or assert traffic during UI
  tests.

## 4. Recommended Project Structure (API-focused)

Your test project (consumer of Playwright Java):

```
src/
├── main/
│   └── java/
│       └── com/yourcompany/
│           ├── api/
│           │   ├── client/            # API client wrappers per service
│           │   │   ├── UsersClient.java
│           │   │   └── OrdersClient.java
│           │   ├── model/             # DTOs for requests/responses (optional)
│           │   │   ├── UserRequest.java
│           │   │   └── UserResponse.java
│           │   ├── support/           # Builders, serializers, schema utils
│           │   │   └── JsonUtils.java
│           │   └── BaseApiTest.java   # Sets up APIRequestContext fixture
│           └── config/
│               └── ConfigManager.java
└── test/
    └── java/
        └── com/yourcompany/tests/api/
            ├── UsersApiTests.java
            └── OrdersApiTests.java
```

## 5. Creating and Using an APIRequestContext

### 5.1 Basic setup with baseURL and headers

```java
import com.microsoft.playwright.*;
import com.microsoft.playwright.options.*;

import java.util.Map;

public class BaseApiTest {
  protected Playwright playwright;
  protected APIRequestContext api;

  @org.testng.annotations.BeforeClass
  public void setUp() {
    playwright = Playwright.create();
    api = playwright.request().newContext(new APIRequest.NewContextOptions()
      .setBaseURL("https://api.example.com")
      .setExtraHTTPHeaders(Map.of(
        "Accept", "application/json",
        "X-Client", "tests"
      ))
      .setTimeout(30_000));
  }

  @org.testng.annotations.AfterClass
  public void tearDown() {
    if (api != null) api.dispose();
    if (playwright != null) playwright.close();
  }
}
```

### 5.2 GET and JSON assertions

```java
APIResponse res = api.get("/users/42");
assert res.

ok();
assert res.

status() ==200;
String contentType = res.headers().get("content-type");
assert contentType !=null&&contentType.

contains("application/json");

com.google.gson.JsonObject json = res.json().getAsJsonObject();
assert json.

get("id").

getAsInt() ==42;
  assert json.

get("active").

getAsBoolean();
```

### 5.3 POST with JSON body

```java
import com.google.gson.Gson;

record CreateUser(String name, String email) {
}

var body = new CreateUser("Ava", "ava@example.com");
APIResponse res = api.post("/users",
  RequestOptions.create()
    .setHeader("Content-Type", "application/json")
    .setData(new Gson().toJson(body))
);
assert res.

status() ==201;
```

### 5.4 Form data and multipart upload

```java
// application/x-www-form-urlencoded
APIResponse res1 = api.post("/login",
    RequestOptions.create()
      .setForm(Map.of("username", "john", "password", "secret"))
  );
assert res1.

ok();

// multipart/form-data with file
APIResponse res2 = api.post("/files",
  RequestOptions.create()
    .setMultipart(Map.of(
      "meta", new Multipart().setContentType("application/json").setData("{\"tag\":\"docs\"}"),
      "file", Multipart.fromFile("/path/to/report.pdf")
    ))
);
assert res2.

status() ==200;
```

### 5.5 Timeouts and retries (simple pattern)

```java
int attempts = 0;
APIResponse res = null;
RuntimeException last = null;
while(attempts++ < 3){
  try{
res =api.

get("/health",RequestOptions.create().

setTimeout(5_000));
  if(res.

ok())break;
  }catch(
RuntimeException ex){
last =ex;
  }
    try{
    Thread.

sleep(500L*attempts);
  }catch(
InterruptedException ignored){
  Thread.

currentThread().

interrupt();
  }
    }
    if(res ==null||!res.

ok()){
  throw(last !=null?last :new

RuntimeException("/health not OK after retries"));
  }
```

### 5.6 Authentication

- Bearer token:

```java
APIRequestContext api = playwright.request().newContext(new APIRequest.NewContextOptions()
  .setBaseURL("https://api.example.com")
  .setExtraHTTPHeaders(Map.of(
    "Authorization", "Bearer " + System.getenv("API_TOKEN")
  )));
```

- Basic auth:

```java
String basic = java.util.Base64.getEncoder().encodeToString("user:pass".getBytes());
APIRequestContext api = playwright.request().newContext(new APIRequest.NewContextOptions()
  .setExtraHTTPHeaders(Map.of("Authorization", "Basic " + basic)));
```

- Cookie-based via storageState produced by a UI sign-in:

```java
// After performing UI login in a BrowserContext, save storage state to file
context.storageState(new BrowserContext.StorageStateOptions().

setPath(Paths.get("auth.json")));

// Reuse cookies for API requests
APIRequestContext api = playwright.request().newContext(new APIRequest.NewContextOptions()
  .setStorageStatePath(Paths.get("auth.json"))
  .setBaseURL("https://app.example.com"));
```

### 5.7 File download via API

```java
APIResponse res = api.get("/export/csv");
assert res.

ok();

byte[] bytes = res.body();
java.nio.file.Files.

write(java.nio.file.Paths.get("build/export.csv"),bytes);
```

## 6. Best Practices

- Context-per-test: Create an APIRequestContext per test or per test class to ensure isolation and parallel safety.
- BaseURL: Configure setBaseURL to simplify path-only requests and reduce duplication.
- Default headers: Use setExtraHTTPHeaders for common headers; override per request with RequestOptions.setHeader.
- Timeouts: Set reasonable defaults (e.g., 30s) and tighter, explicit timeouts for probes/health checks.
- Retries: Implement idempotent retries with backoff for flaky networks; avoid retrying non-idempotent operations unless
  safe.
- JSON handling: Prefer robust JSON parsing (Gson/Jackson); centralize in utilities.
- Schema validation: For schema checks, integrate your preferred validator (e.g., everit-org JSON Schema). Playwright
  does not ship schema validation out of the box.
- Secrets: Pull tokens/credentials from env vars or a secure vault; avoid hardcoding.
- CI artifacts: On failure, log response status, headers, and a truncated body; store bodies to files if large.
- Mixed UI+API: Use storageState to bridge auth between UI and API tests; consider route mocking during UI flows for
  determinism.

## 7. Parallel Execution

- APIRequestContext is lightweight and supports parallel creation. Do not share a single context across unrelated tests.
- If you also run browser tests in parallel, keep API contexts separate from BrowserContext instances unless you
  intentionally share storageState.
- Use thread-safe factories (e.g., ThreadLocal) or your test runner’s lifecycle hooks to scope contexts per thread/test.

## 8. Mixing API and UI Tests

Common hybrid patterns:

- UI bootstrap, API validate: Sign-in via UI once, save storageState, call APIs to validate downstream effects.
- API setup, UI verify: Create entities via API for fast test data setup, then navigate UI to validate display.
- Network mocking: In UI flows, use BrowserContext.route to mock or assert calls; for pure API tests, prefer
  APIRequestContext to hit the backend directly.

Example UI->API handoff:

```java
// After UI creates an order, validate by API
APIResponse res = api.get("/orders/" + orderId);
assert res.

ok();

var json = res.json().getAsJsonObject();
assert"CONFIRMED".

equals(json.get("status").

getAsString());
```

## 9. Troubleshooting

- 4xx/5xx errors: Log request path, headers (minus secrets), and body; confirm baseURL and environment URLs.
- TLS/Cert issues: Configure JVM truststore or disable strict checks only in non-prod suites.
- Large responses: Stream or handle as bytes; avoid printing entire payloads in CI logs.
- Flaky endpoints: Add targeted retries with exponential backoff and jitter; coordinate with teams to stabilize.

## 10. Comparison: Playwright API vs RestAssured

### 10.1 Conceptual Mapping

| Concept              | Playwright (API)                                | RestAssured                                                 |
|----------------------|-------------------------------------------------|-------------------------------------------------------------|
| Client/Context       | APIRequestContext (per test/fixture)            | RequestSpecification (default or custom per test)           |
| Request Options      | RequestOptions (headers, data, form, multipart) | Given()/spec().headers(), body(), formParams(), multiPart() |
| Response             | APIResponse                                     | Response                                                    |
| Base URL             | setBaseURL on context                           | RestAssured.baseURI/basePath or spec.baseUri/basePath       |
| Auth                 | Headers, storageState (cookies)                 | auth().oauth2(), preemptive basic, etc.                     |
| Assertions           | Via test framework + JSON libs                  | Built-in then().statusCode(), body(), hamcrest matchers     |
| Interception/Mocking | BrowserContext.route (UI context)               | Filters (limited), otherwise external mocking               |
| Schema Validation    | External library                                | RestAssured module (json-schema-validator)                  |
| Parallelism          | Context-per-test, thread-safe                   | Specs per test/thread; thread-safe                          |
| UI+API Integration   | First-class (shared storageState, route)        | Not applicable (no UI automation)                           |

### 10.2 Common Features

- Base URL configuration, default headers
- GET/POST/PUT/PATCH/DELETE with JSON/form/multipart
- Auth via headers/tokens/cookies
- Response inspection (status, headers, body)
- Test runner agnosticism (JUnit/TestNG)
- Parallel-friendly when contexts/specs are per test

### 10.3 Key Differences

- Purpose: RestAssured is specialized for API testing with expressive BDD assertions; Playwright is a full-stack browser
  automation tool that includes a capable HTTP client for API tests and hybrid UI+API workflows.
- Assertions: RestAssured provides fluent then().body() with matchers and optional schema validators; Playwright relies
  on your test/assertion libraries (e.g., AssertJ, Hamcrest, JSON schema lib) — explicit but flexible.
- Mocking/Interception: Playwright shines in UI tests via route interception/mocking and end-to-end visibility;
  RestAssured focuses on direct HTTP calls and offers Filters for cross-cutting concerns but no browser network layer
  interception.
- State Bridging: Playwright can persist/reuse browser auth state for API calls (storageState), uniquely enabling UI↔API
  continuity; RestAssured does not interact with browser state.
- Tooling Ecosystem: RestAssured has first-class JSON schema validation add-ons and logging; with Playwright you
  assemble best-of-breed JSON libs and validators.

### 10.4 When to Choose Which

- Use Playwright API when:
  - You already use Playwright for UI and want unified tooling and auth state reuse.
  - You need to combine UI flows with backend validations in one run.
  - You want route mocking alongside API calls for deterministic E2E tests.

- Use RestAssured when:
  - You need rich, fluent API assertions with built-in matchers and schema validators.
  - Your scope is purely backend API testing with no browser automation needs.
  - Your team already standardizes on RestAssured and related plugins.

- Use both:
  - Keep a dedicated backend contract suite in RestAssured and run hybrid E2E validation in Playwright where UI and API
    meet.

## 11. Examples: RestAssured Equivalents in Playwright

- RestAssured style:

```java
given()
  .

baseUri("https://api.example.com")
  .

header("Accept","application/json")
.

when()
  .

get("/users/42")
.

then()
  .

statusCode(200)
  .

body("id",equalTo(42));
```

- Playwright equivalent:

```java
APIRequestContext api = playwright.request().newContext(new APIRequest.NewContextOptions()
  .setBaseURL("https://api.example.com")
  .setExtraHTTPHeaders(Map.of("Accept", "application/json")));
APIResponse res = api.get("/users/42");
assert res.

status() ==200;
var json = res.json().getAsJsonObject();
assert json.

get("id").

getAsInt() ==42;
```

## 12. CI/CD Considerations

- Environment selection: Parameterize baseURL and tokens via environment variables or system properties.
- Artifacts: Save response bodies for failed tests to build artifacts for triage.
- Rate limits: Add polite backoff and identify with custom headers (X-Client) where allowed.
- Secrets: Use CI secret stores; mask logs.

## 13. References

- Official docs: https://playwright.dev/java/docs/api/class-apirequestcontext
- Request options: https://playwright.dev/java/docs/api/class-apirequest
- Storage state: https://playwright.dev/java/docs/auth#persist-authentication-state
- Route mocking: https://playwright.dev/java/docs/network#modify-responses
- RestAssured: https://rest-assured.io/

---

Tip: For UI concepts (Browser/Context/Page, parallelism, tracing), see docs/PLAYWRIGHT_JAVA_FRAMEWORK_GUIDE.md and
docs/trace-viewer.md.
