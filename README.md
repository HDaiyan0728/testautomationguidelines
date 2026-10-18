# Hybrid Test Automation Framework: The Complete Guide

Welcome to the definitive architectural guide for our Hybrid Test Automation Framework.

This framework is designed to test **Web, Mobile, Desktop, and API** applications from a single, unified structure. It is built on rigorous system design principles, prioritizing maintainability, scalability, and absolute **anti-flakiness**.

*Based on the Test Automation Framework Guide by Hasin Md. Daiyan.*

---

## 1. Core Philosophy: The Test Pyramid & Risk

Test each behavior at the lowest level that gives you enough confidence.

### The Test Pyramid

UI tests are slow and break easily. API and unit tests are fast and stable.

- **End-to-End/UI:** Few, slow tests that verify real user journeys using tools like Playwright and Appium. These break most often.
- **API/Integration/Contract:** More numerous, fast tests that verify business rules, permissions, and schemas directly.
- **Unit/Component:** The largest number of tests. These are the fastest and are written by developers to test isolated logic.

> Do not test business rules through the UI when the API can check them in milliseconds.

### Risk Analysis

You cannot automate everything. Automate based on:

**Risk = Likelihood × Impact**

- **Critical (15–25):** Automate first. Cover every business rule with API/Unit tests. Add 1–2 E2E journeys per platform.
- **High (10–14):** Automate soon. Use API tests for business rules and at least one UI path.
- **Medium (5–9):** Automate the main path once it is stable.
- **Low (1–4):** Keep manual or exploratory testing.

---

## 2. Architecture & The Dependency Rule

The framework is a stack of layers. A layer may call the layers below it, **never the layers above**.

When calls go in every direction, a change anywhere can break everything. With one-directional dependencies, a locator change only affects the page layer, and a tool switch only affects the driver layer.

### The Layers (Top to Bottom)

1. **Tests:** Describe behavior and check outcomes. No locators, URLs, or sleeps.
2. **Step Definitions (BDD):** Translate Gherkin sentences into framework calls. No business logic.
3. **Facades:** Execute one business flow across several pages or APIs.
4. **Strategies:** Provide interchangeable ways to perform one job, such as UI login versus API session creation.
5. **Pages / API Clients:** Own locators and endpoints. Offer actions such as `loginAs()`.
6. **Base Page:** Standardizes the open-then-wait sequence for every page using the Template Method pattern.
7. **Driver Interface & Adapters:** Hide Playwright and Appium behind a small interface. Own all waiting.
8. **Driver Factory:** Reads the configuration and creates the appropriate adapter.

*Cross-cutting concerns such as Configuration, Logging, and Test Data are used by every layer but live independently.*

---

## 3. Folder Structure & Naming

The folder tree makes the architecture visible. A new team member should be able to identify where a file belongs without asking.

### Folder Structure

```text
automation-framework/
├── framework/                 # Reusable code (no tests here)
│   ├── config/                # Singleton config manager
│   ├── drivers/               # Interface & adapters (web/, mobile/, desktop/)
│   ├── factory/               # Driver and Page factories
│   ├── pages/                 # Page Objects (base/, web/, mobile/, desktop/)
│   ├── facades/               # Business flows spanning multiple pages
│   ├── strategies/            # Interchangeable behaviors
│   ├── api/                   # API clients, models, schemas
│   ├── data/                  # Repositories and builders
│   └── listeners/             # Observers for reporting/evidence
├── tests/                     # Test scenarios
│   ├── features/              # Gherkin files (BDD only)
│   ├── steps/                 # Step definitions (BDD only)
│   ├── hooks/                 # Setup, teardown, and fixtures
│   ├── web/
│   ├── mobile/
│   ├── desktop/
│   └── api/
├── testdata/                  # Non-secret JSON, Excel, CSV files
├── config/                    # Non-secret environment files (qa.env)
├── docs/
│   └── adr/                   # Architecture Decision Records
└── output/                    # Reports, screenshots, traces (Git ignored)
```

### Naming Rules

- **Page class:** Noun + `Page` for web or `Screen` for native apps.
  - Examples: `WebLoginPage`, `NativeLoginScreen`
- **Facade:** Business noun + `Facade`.
  - Example: `OrderFacade`
- **Strategy:** Behavior + `Strategy`.
  - Example: `ApiSessionStrategy`
- **Repository:** Data noun + `Repository`.
  - Example: `UserRepository`
- **API Client:** Service + `Api`.
  - Example: `OrdersApi`
- **Test Name:** Describes the behavior.
  - Example: `registered customer can log in`

---

## 4. Design Principles (SOLID & System Design)

Test code grows quickly. Without clear principles, every new test makes the next one harder to write.

### Single Responsibility Principle (SRP)

A class has one reason to change. `LoginPage` changes only when the login screen changes. It does not read Excel files or create drivers.

### Open/Closed Principle

Add behavior with new classes rather than editing existing ones. A new SSO login implementation should be a new strategy class, leaving `LoginFacade` unchanged.

### Liskov Substitution Principle

A subtype must work wherever its parent type works. Every `LoginView` implementation (Web, iOS, Android) should support `loginAs()`.

### Interface Segregation Principle

Keep interfaces small. Desktop applications have no URLs, so `goTo()` does not belong in the base driver interface. It belongs in a separate `Navigable` interface.

### Dependency Inversion Principle

Depend on interfaces, not concrete classes. `LoginFacade` should depend on a `LoginStrategy`, not a specific UI class.

### DRY (Don't Repeat Yourself)

Remove repeated knowledge, not necessarily every repeated line. A locator should live in one page object.

### KISS & YAGNI

Keep it simple. Do not build desktop adapters until there is a desktop application that needs them.

---

## 5. Configuration & Secrets

One place should answer three questions:

- Where am I running?
- On which platform?
- Against which URL?

### Singleton Pattern

A configuration manager loads settings once and provides the same read-only configuration object to every part of the framework.

### Configuration Cascade

Later sources override earlier ones:

1. Safe defaults in code
2. Non-secret environment file
3. Environment variables (local/CI)
4. Command-line arguments

### Fail Fast

If a required value, such as `BASE_URL`, is missing, stop with a clear error message before any tests run.

### Security

- Never commit secrets.
- Locally, use a Git-ignored `.env` file.
- In CI, use the secret store.
- Treat logs, traces, and screenshots as sensitive information.

---

## 6. The Driver Layer (Interface & Factory)

This is the heart of the hybrid framework.

### The Interface

Lists what any supported platform can do:

- `click()`
- `type()`
- `textOf()`
- `waitVisible()`
- `screenshot()`

### The Adapters

**Playwright**

Drives browsers and automatically waits for elements. Adapter methods can therefore remain thin.

**Appium (Android/iOS/Windows)**

Does not automatically wait in the same way. The adapter adds explicit waits using `WebDriverWait` and `ExpectedConditions` exactly once, so no page object or test needs a fixed sleep.

### The Factory

Reads `PLATFORM` from the configuration and returns the appropriate adapter.

Tests and page objects never create drivers directly.

---

## 7. Patterns in Action (POM, Facade, Strategy)

### Base Page (Template Method)

Standardizes the page-opening sequence:

`navigate() → waitVisible(readyMarker)`

Child pages only define their URL and ready marker.

### Page Object Model (POM)

A page object is the only place that knows a screen's locators. Tests communicate with it through plain actions such as `loginAs()`.

In a hybrid framework, each platform has its own page class. For example:

- `WebLoginPage`
- `NativeLoginScreen`

All implementations follow a shared interface, such as `LoginView`.

### Facade

Combines multiple screen steps into one business action.

For example, `OrderFacade.placeOrder(user, sku, address)` calls the Cart, Product, and Checkout pages in the correct order.

The test reads like a sentence rather than a sequence of low-level UI operations.

### Strategy

Allows behavior to be swapped at runtime.

For example:

- Use `UiLoginStrategy` for the 10 tests that specifically test the login screen.
- Use `ApiSessionStrategy` for the other 300 tests that simply need an authenticated session.

This avoids unnecessary UI login steps and makes the test suite faster.

---

## 8. Test Data Management

Shared, hard-coded data is a major cause of flaky tests.

### Repository

The repository is the only code that reads test data.

Tests request data by meaning:

```java
users.get("admin");
```

They should never request data by physical location, such as "row 4 of sheet 2".

### Source

A source is a reader for one data format or system:

- JSON
- Excel
- CSV
- API
- Database

JSON is best for static reference data, while APIs are useful for creating data separately for each test.

### Builder

Creates complex objects with safe defaults. The test only sets the values it cares about.

### Data-Driven Testing

One test body runs multiple times, once for each row of data.

Example: login validation cases.

---

## 9. Anti-Flakiness & Synchronization

A flaky test passes and fails against the same code. This trains the team to ignore the test suite.

### No Fixed Sleeps

Never use:

- `Thread.sleep()`
- `waitForTimeout()`

### Wait for State

Wait for something meaningful, such as:

- A visible message
- An enabled button
- The exact network response, such as `POST /api/orders` returning `201`

Start listening for the network response **before** clicking the button.

### Isolate Test Data

Every test gets its own unique email or ID. Never allow tests to share accounts or mutable test data.

### Test Order

Every test must be able to run independently, in any order, and in parallel.

### Retries

Retry infrastructure failures, such as a session failing to start.

Never automatically retry:

- Assertions that already have built-in retry behavior.
- State-changing actions, such as payment submissions, because retrying could create duplicate orders.

---

## 10. Observability & Reporting

A good failure report answers four questions quickly:

- What failed?
- Where did it fail?
- When did it fail?
- Why did it fail?

### The Observer Pattern

When a test finishes, an event is published. Listeners react to that event.

**ScreenshotOnFailure**

Ignores passing tests and saves a screenshot when a test fails.

**TraceAndLogCollector**

Saves traces and console logs.

**ReportWriter**

Generates JUnit XML and HTML reports.

**ChatNotifier**

Alerts the team when a smoke test fails.

### Single Output Folder

Everything goes into `output/`, including:

- Reports
- Screenshots
- Traces
- Videos

### Mask Secrets

Redact passwords, tokens, cookies, and other sensitive values from all logs and evidence.

---

## 11. Parallel Execution & CI/CD

Parallel execution makes the test suite faster. CI/CD ensures it runs automatically as part of development.

### Parallel Safety Checklist

- Every test uses unique data.
- Each test or worker has its own browser context or device session.
- Output filenames cannot collide between workers.
- Shared resources, such as a payment sandbox, use locks or resource pools.
- Worker count matches the capacity of the backend and available devices.

### CI Pipelines

**Pull Request**

Run a fast, blocking smoke suite and API tests.

**Merge to Main**

Run broader regression tests.

**Nightly**

Run the full browser/device matrix, randomize test order, and execute a flaky-test quarantine job.

---

## 12. Review Checklist & Definition of Done

Use this checklist during every pull request review.

### Pull Request Review Checklist

- [ ] The test checks a meaningful outcome.
- [ ] It uses the lowest test level that provides enough confidence.
- [ ] Locators live in page objects, not in tests or facades.
- [ ] There are no fixed sleeps anywhere.
- [ ] Test data is unique to the test.
- [ ] No state-changing action is retried automatically.
- [ ] No secrets or personal data appear in code, logs, or evidence.
- [ ] A failure produces enough evidence for diagnosis.
- [ ] The test passes independently, in random order, and in parallel.
- [ ] New test IDs (e.g., `data-testid`) are added to the application in the same PR.

### Definition of Done

**Local Use**

A new teammate can clone, install, configure, and investigate a failure without asking anyone about undocumented rules.

**Diagnostics**

Failures save evidence to `output/`. Reports show the environment, platform, and execution duration.

**Maintenance**

Important architectural decisions have ADRs (Architecture Decision Records). Runtime, tool, and driver versions are pinned, with a defined update plan.

**Automatic Enforcement**

Use lint rules or CI checks to detect:

- Fixed sleeps
- Positional XPath expressions
- Secret patterns
- Missing test suite tags

---

## Final Principle

Build the framework so that tests describe **what the application should do**, while the framework handles **how to interact with the application reliably**.
