---
tags:

- testing
- python
- playwright
- e2e
- ci

---

# Playwright for End-to-End Testing

Playwright drives a real browser (Chromium, Firefox, WebKit) against a real running
application — the closest a test gets to "what would a user actually experience." It
trades speed and isolation (the two things unit tests optimize for) for confidence that
the whole system, frontend included, actually works together.

## The Browser / Context / Page Model

Playwright's API has three layers, and understanding them is what makes test isolation and
speed manageable:

- **Browser** — one actual browser process (Chromium/Firefox/WebKit). Expensive to start,
  so it's typically launched once per test *session*, not once per test.
- **BrowserContext** — an isolated "incognito-like" session within a browser: its own
  cookies, local storage, cache. Cheap to create relative to a browser. Each test should
  get its own fresh context so tests never leak login state or cookies into each other.
- **Page** — a single tab within a context. Most test interactions happen on a `Page`.

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)      # one browser process
    context = browser.new_context()                  # isolated session
    page = context.new_page()                        # one tab

    page.goto("https://example.com")
    page.click("text=Sign in")

    context.close()
    browser.close()
```

!!! tip "Pro Tip"
    Launch the `Browser` once per test session and hand out a fresh `BrowserContext` per
    test — reusing the browser process is what keeps a Playwright suite fast, while a new
    context per test is what keeps tests isolated from each other's cookies/storage
    without needing a full browser restart.

## `pytest-playwright`: Fixtures Out of the Box

The `pytest-playwright` plugin provides `page`, `context`, and `browser` as ready-to-use
pytest fixtures, scoped correctly by default (browser per session, context/page per test)
— no manual setup/teardown needed:

```python
# tests/test_login.py
def test_login_succeeds(page):
    page.goto("https://app.example.test/login")
    page.get_by_label("Email").fill("ada@example.com")
    page.get_by_label("Password").fill("correct-horse-battery-staple")
    page.get_by_role("button", name="Sign in").click()

    page.wait_for_url("**/dashboard")
    assert page.get_by_text("Welcome, Ada").is_visible()
```

```bash
pip install pytest-playwright
playwright install                 # downloads browser binaries
pytest --browser chromium          # or firefox / webkit; can pass multiple times
pytest --headed                    # watch it run instead of headless
```

Because `page`/`context` are `function`-scoped by default, every test gets a clean browser
tab with no cookies or storage from a previous test — the browser-level equivalent of not
letting unit tests share mutable state (see
[Pytest Fixtures](pytest-fixtures-monkeypatch.md#unit-test-standards)).

## Locators and Auto-Waiting

Modern Playwright code is built around **locators** — an object describing *how to find*
an element, resolved fresh every time an action is performed, rather than a one-shot
reference to a specific DOM node:

```python
# Prefer role/label/text-based locators — resilient to markup/CSS changes
login_button = page.get_by_role("button", name="Sign in")
email_input = page.get_by_label("Email")
error_banner = page.get_by_text("Invalid credentials")

login_button.click()
```

Every locator action (`click`, `fill`, `check`, ...) **auto-waits**: Playwright retries
until the element is attached to the DOM, visible, stable (not mid-animation), and not
covered by another element — up to a timeout — before acting. This eliminates the entire
class of "clicked before it rendered" flakiness that plagued older Selenium-style tests.

```python
# OLD STYLE — explicit sleeps, fragile and slow
import time

driver.find_element_by_id("submit").click()
time.sleep(2)  # hope the page finished loading by then
assert "Success" in driver.page_source
```

```python
# PLAYWRIGHT — auto-waiting, no manual sleep
page.get_by_role("button", name="Submit").click()
expect(page.get_by_text("Success")).to_be_visible()  # retries until true or timeout
```

!!! warning
    `time.sleep()`/hardcoded waits in browser tests are almost always a symptom, not a fix
    — they make the suite slow (always waiting the full duration) *and* still flaky
    (sometimes the duration isn't enough on a slow CI runner). `expect(locator).to_...()`
    assertions and locator actions already retry with a sensible timeout; reach for an
    explicit `page.wait_for_selector`/`wait_for_url` only for genuinely async state that
    isn't expressed by an element appearing (e.g. waiting on a specific URL after a
    redirect chain).

## The Page Object Model (POM)

As a suite grows, repeating raw selectors (`page.get_by_label("Email")`) across dozens of
test files makes every markup change a multi-file find-and-replace. The **Page Object
Model** wraps each page/component's locators and actions behind a class, so tests read at
the level of user intent and selector changes happen in exactly one place.

```python
# pages/login_page.py
from playwright.sync_api import Page


class LoginPage:
    def __init__(self, page: Page):
        self.page = page
        self.email_input = page.get_by_label("Email")
        self.password_input = page.get_by_label("Password")
        self.submit_button = page.get_by_role("button", name="Sign in")
        self.error_banner = page.get_by_text("Invalid credentials")

    def goto(self):
        self.page.goto("https://app.example.test/login")

    def login(self, email: str, password: str):
        self.email_input.fill(email)
        self.password_input.fill(password)
        self.submit_button.click()
```

```python
# tests/test_login.py
import pytest
from pages.login_page import LoginPage


@pytest.fixture
def login_page(page):
    return LoginPage(page)


def test_invalid_credentials_show_error(login_page):
    login_page.goto()
    login_page.login("ada@example.com", "wrong-password")
    assert login_page.error_banner.is_visible()


def test_valid_login_redirects_to_dashboard(login_page):
    login_page.goto()
    login_page.login("ada@example.com", "correct-horse-battery-staple")
    login_page.page.wait_for_url("**/dashboard")
```

If the login form's markup changes, exactly one file (`login_page.py`) needs updating —
every test that uses `LoginPage` keeps working unchanged. This is the same motivation as
the [fixture-factory pattern](pytest-fixtures-monkeypatch.md#fixture-factories) in unit
tests: centralize construction logic so tests express intent, not mechanics.

## Debugging Flaky Tests: Trace, Screenshots, Video

Browser tests fail for reasons a stack trace alone rarely explains — a race condition, an
unexpected redirect, a CSS layout shift hiding an element. Playwright's built-in debugging
artifacts exist specifically for this:

```ini
# pytest.ini / pyproject.toml [tool.pytest.ini_options]
[pytest]
addopts = --tracing=retain-on-failure --screenshot=only-on-failure --video=retain-on-failure
```

- **Trace** (`--tracing=retain-on-failure`) — a full recording of the test: every action,
  network request, console log, and DOM snapshot at each step. Open it with
  `playwright show-trace trace.zip` for a timeline you can scrub through frame by frame —
  by far the fastest way to root-cause a flaky failure without reproducing it locally.
- **Screenshot** (`--screenshot=only-on-failure`) — a single image at the moment of
  failure; cheap, good for a quick "what did the page actually look like" sanity check.
- **Video** (`--video=retain-on-failure`) — a full recording of the test run; more useful
  than a screenshot for failures caused by *timing* (something appeared, then disappeared)
  rather than final state.

```python
# Manual trace capture (without pytest-playwright's flags), for one-off debugging
context = browser.new_context()
context.tracing.start(screenshots=True, snapshots=True, sources=True)

page = context.new_page()
page.goto("https://app.example.test")
# ... test actions ...

context.tracing.stop(path="trace.zip")
```

!!! tip "Pro Tip"
    Set these to `retain-on-failure` / `only-on-failure`, not `on` — capturing full traces
    and video for every passing test bloats CI artifact storage and slows the run for no
    benefit. The goal is evidence when something breaks, not a recording of the happy path.

## Network Interception and Mocking

Playwright can intercept and mock network requests at the browser level, which is what
makes it possible to test frontend behavior — loading states, error handling, empty states
— without depending on a real backend being in a specific state:

```python
def test_shows_error_banner_on_api_failure(page):
    page.route(
        "**/api/orders",
        lambda route: route.fulfill(status=500, json={"error": "internal error"}),
    )

    page.goto("https://app.example.test/orders")

    assert page.get_by_text("Something went wrong").is_visible()


def test_shows_empty_state(page):
    page.route("**/api/orders", lambda route: route.fulfill(status=200, json=[]))
    page.goto("https://app.example.test/orders")
    assert page.get_by_text("No orders yet").is_visible()
```

`page.route()` can also inspect and pass through real requests selectively — useful for
mocking one flaky third-party integration (payments, email) while letting the rest of the
app talk to its real backend, which keeps the test closer to reality than mocking
everything.

## Headless vs. Headed

- **Headless** (default, `headless=True`) — no visible browser window; faster and the
  only sane option for CI.
- **Headed** (`headless=False` or `pytest --headed`) — a real visible window; useful while
  *writing* a test, to watch exactly where it's clicking, but never wanted in CI.

```python
browser = p.chromium.launch(headless=False, slow_mo=500)  # slow_mo: ms delay per action,
                                                             # for watching test execution
```

## Running in CI

```yaml
# .github/workflows/e2e.yml
- name: Install dependencies
  run: |
    pip install -r requirements.txt
    playwright install --with-deps chromium

- name: Run e2e tests
  run: pytest tests/e2e --browser chromium -n auto
```

- `playwright install --with-deps` also installs the OS-level libraries the browser
  binaries need — CI runner images rarely have them preinstalled.
- `-n auto` (from `pytest-xdist`) parallelizes across CPU cores; each worker gets its own
  browser context, so tests need to be independent of each other (no shared login state,
  no shared test-account data) for parallel runs to be safe — the same "no test
  interdependence" rule from
  [unit testing standards](pytest-fixtures-monkeypatch.md#unit-test-standards) applies here,
  and matters even more once tests run concurrently rather than just in arbitrary order.
- Always run headless in CI — a headed browser needs a display server (or a virtual one
  like Xvfb), which is extra infrastructure for zero benefit once the suite is unattended.
- e2e suites are slow and comparatively expensive per test relative to unit tests — most
  teams run them against a small number of critical user flows (login, checkout,
  onboarding) rather than trying to cover every UI branch this way. See
  [Integration Testing](integration-testing.md#where-integration-tests-sit) for how this
  fits into the broader testing pyramid.

## Summary

- Browser (session-scoped) → BrowserContext (isolated session, per test) → Page (a tab) —
  `pytest-playwright` wires these up as fixtures automatically.
- Prefer role/label/text-based locators over CSS/XPath selectors; every locator action
  auto-waits, which is what eliminates the need for manual `sleep`/explicit waits.
- The Page Object Model centralizes selectors per page/component so a markup change means
  editing one class, not every test file that touches that page.
- Capture trace/screenshot/video only on failure — the trace viewer is the fastest path to
  root-causing flakiness.
- `page.route()` intercepts network requests to test loading/error/empty states without a
  fully-stateful real backend.
- Run headless with `--with-deps` in CI, parallelize with `pytest-xdist`, and keep e2e
  suites focused on a small number of critical flows rather than full UI coverage.

## Related Articles

- [Pytest: Fixtures, Monkeypatching, and Unit Test Standards](pytest-fixtures-monkeypatch.md)
  — the fixture mechanics `pytest-playwright` builds on, and the test-isolation habits
  that matter even more for browser tests.
- [Integration Testing](integration-testing.md) — where e2e tests sit relative to
  integration tests, and when a slow e2e test is better replaced by a faster integration
  or unit test.
