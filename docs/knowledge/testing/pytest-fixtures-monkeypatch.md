---
tags:

- testing
- python
- pytest
- fixtures
- mocking
- fundamentals

---

# Pytest: Fixtures, Monkeypatching, and Unit Test Standards

`pytest` is the de facto unit-testing framework for Python — plain `assert` statements
instead of a `self.assertX` API, a fixture system for setup/teardown, and a plugin
ecosystem (`pytest-mock`, `pytest-cov`, `pytest-xdist`) that covers almost everything a
non-trivial test suite needs. This article covers the mechanics and the habits that keep a
growing test suite fast, isolated, and readable a year later.

## Test Discovery

By default, `pytest` collects tests using these rules:

- Files: `test_*.py` or `*_test.py`.
- Classes: `Test*` (no `__init__` method — pytest can't collect classes with one).
- Functions/methods: `test_*`.

```text
tests/
  unit/
    test_pricing.py      # collected
    pricing_helpers.py    # NOT collected — no test_ prefix
  conftest.py             # shared fixtures, not a test file itself
```

`pytest.ini`, `pyproject.toml` (`[tool.pytest.ini_options]`), or `setup.cfg` can override
the discovery patterns (`python_files`, `python_classes`, `python_functions`) and set a
default `testpaths` so `pytest` doesn't need to be pointed at a directory every time.

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers"
```

`--strict-markers` is worth turning on immediately: without it, a typo in
`@pytest.mark.slwo` (instead of `slow`) is silently accepted instead of failing — see
[Marks](#marks-skip-xfail-and-custom-marks) below.

## Fixtures

A fixture is a function decorated with `@pytest.fixture` that provides setup (and,
optionally, teardown) for a test. Tests request a fixture by naming it as a parameter —
`pytest` resolves and injects it automatically; there's no manual wiring.

```python
import pytest


@pytest.fixture
def user() -> dict:
    return {"id": 1, "name": "Ada", "email": "ada@example.com"}


def test_user_has_email(user):
    assert "@" in user["email"]
```

### Teardown with `yield`

A fixture that `yield`s instead of `return`s runs the code after the `yield` as teardown,
after the test finishes (pass or fail):

```python
import pytest


@pytest.fixture
def db_connection():
    conn = connect_to_test_db()
    yield conn
    conn.rollback()
    conn.close()
```

This is the standard pattern for anything that needs cleanup — file handles, network
connections, temp state in an external service. Teardown runs even if the test raises, so
resources don't leak just because an assertion failed.

### Scope: How Often a Fixture Runs

By default a fixture runs once *per test function that requests it*. `scope` controls that:

```python
@pytest.fixture(scope="function")  # default — fresh instance per test
def fresh_list():
    return []


@pytest.fixture(scope="class")     # once per test class
def class_resource():
    return ExpensiveResource()


@pytest.fixture(scope="module")    # once per test file
def db_schema():
    return create_schema()


@pytest.fixture(scope="session")   # once for the entire test run
def docker_network():
    return spin_up_network()
```

| Scope | Recreated | Typical use |
|---|---|---|
| `function` (default) | Every test | Anything mutable, or where test isolation matters |
| `class` | Once per test class | Shared read-only setup for a group of related tests |
| `module` | Once per file | A DB schema, a loaded config, an expensive parse |
| `session` | Once per test run | A docker-compose stack, a browser process, a test container |

!!! warning
    Wider scope means faster tests but more shared mutable state. A `session`-scoped
    fixture that returns a mutable object (a list, a dict, an ORM session) and gets
    *mutated* by one test silently leaks that mutation into every other test that uses it
    — usually surfacing as "this test fails only when run after test X." If a fixture's
    value needs to be mutated per test, keep it `function`-scoped, or have a wider-scoped
    fixture hand out a fresh child/transaction per test (see the transactional-rollback
    pattern in [Integration Testing](integration-testing.md)).

### `conftest.py`: Sharing Fixtures Across a Tree

Fixtures defined in a `conftest.py` file are automatically available to every test in that
file's directory *and subdirectories* — no import needed. This is what lets fixtures be
shared without every test file importing from a `fixtures.py` module.

```text
tests/
  conftest.py            # fixtures available to ALL tests below
  unit/
    conftest.py           # fixtures available only to tests under unit/
    test_pricing.py
  integration/
    conftest.py           # fixtures available only to tests under integration/
    test_api.py
```

```python
# tests/conftest.py
import pytest


@pytest.fixture
def settings():
    return load_test_settings()
```

```python
# tests/unit/test_pricing.py — no import of `settings` needed
def test_discount_applies(settings):
    assert compute_price(100, settings) == 90
```

This layered structure is deliberate: put broadly-useful fixtures (config, a fake clock) in
the top-level `conftest.py`, and put fixtures specific to one test category (an HTTP test
client for API tests, a browser fixture for e2e tests) in that category's own
`conftest.py` — so unit tests never even see fixtures they have no business depending on.

### Fixture Composition

Fixtures can depend on other fixtures, and pytest resolves the dependency graph:

```python
@pytest.fixture
def db_connection():
    conn = connect_to_test_db()
    yield conn
    conn.close()


@pytest.fixture
def user_repository(db_connection):
    return UserRepository(db_connection)


def test_save_user(user_repository):
    user_repository.save(User(name="Ada"))
    assert user_repository.count() == 1
```

`user_repository` doesn't need to know how `db_connection` is built — it just declares the
dependency, and `pytest` builds the chain in order, tearing it down in reverse.

### Autouse Fixtures

`autouse=True` makes a fixture apply to every test in its scope without being explicitly
requested — useful for cross-cutting setup like resetting global state or freezing time:

```python
@pytest.fixture(autouse=True)
def reset_global_cache():
    yield
    _cache.clear()  # runs after every test, whether it requested this or not
```

Use sparingly — implicit behavior that every test picks up without asking for it is easy
to forget about when debugging a failure two months later.

### Fixture Factories

A regular fixture returns one fixed value. When a test needs *several* differently
configured instances, a fixture that returns a **factory function** is the standard
pattern:

```python
import pytest


@pytest.fixture
def make_user():
    created = []

    def _make_user(**overrides):
        defaults = {"id": 1, "name": "Ada", "role": "admin"}
        user = {**defaults, **overrides}
        created.append(user)
        return user

    yield _make_user
    # teardown could clean up everything in `created` here if needed


def test_admin_and_guest(make_user):
    admin = make_user(role="admin")
    guest = make_user(id=2, role="guest")
    assert admin["role"] != guest["role"]
```

!!! tip "Fixture vs. factory-as-fixture"
    Reach for a plain fixture when a test needs *one* instance with sensible defaults.
    Reach for a factory fixture the moment a test needs more than one instance, or needs to
    vary construction per call — trying to force that through fixture parametrization
    (below) usually ends up more convoluted than a factory function would have been.

## Parametrization

`@pytest.mark.parametrize` runs the same test body against multiple inputs, reported as
separate test cases rather than one test with a loop — so a failure on one input doesn't
hide the others, and the failure output shows exactly which input failed.

```python
import pytest


@pytest.mark.parametrize(
    "amount, discount, expected",
    [
        (100, 0, 100),
        (100, 10, 90),
        (100, 100, 0),
        (0, 50, 0),
    ],
)
def test_apply_discount(amount, discount, expected):
    assert apply_discount(amount, discount) == expected
```

This runs as four independent tests: `test_apply_discount[100-0-100]`,
`test_apply_discount[100-10-90]`, etc. Stacking two `@parametrize` decorators multiplies
combinations (a Cartesian product) — useful for exhaustively covering small input spaces,
but grows fast, so use it deliberately rather than by default.

```python
@pytest.mark.parametrize("currency", ["USD", "EUR", "GBP"])
@pytest.mark.parametrize("amount", [0, 100, -1])
def test_format_amount(amount, currency):
    ...  # 9 test cases: every amount × every currency
```

Fixtures can be parametrized too, which parametrizes every test that uses them:

```python
@pytest.fixture(params=["sqlite", "postgres"])
def db_backend(request):
    return connect(request.param)


def test_insert_and_read(db_backend):
    ...  # runs once against sqlite, once against postgres
```

## Marks: `skip`, `xfail`, and Custom Marks

```python
import sys
import pytest


@pytest.mark.skip(reason="not implemented yet")
def test_future_feature():
    ...


@pytest.mark.skipif(sys.platform == "win32", reason="posix-only path handling")
def test_symlink_behavior():
    ...


@pytest.mark.xfail(reason="known bug, see JIRA-1234")
def test_currently_broken():
    assert buggy_function() == 42
```

- `skip` / `skipif` — don't run the test at all, under the given (optional) condition.
- `xfail` — run it, but expect it to fail; a failure is reported as `xfail` (not a suite
  failure), and if it unexpectedly *passes* that's reported as `XPASS`, which is worth
  paying attention to (`strict=True` turns an unexpected pass into a real failure — useful
  once you trust the suite enough to want to know the moment a "known bug" gets fixed).

Custom marks group tests for selective runs (`pytest -m slow`, `pytest -m "not slow"`):

```python
@pytest.mark.slow
def test_full_data_pipeline():
    ...
```

Register custom marks in `pyproject.toml` — combined with `--strict-markers`, an
unregistered mark (typically a typo) fails the run instead of silently doing nothing:

```toml
[tool.pytest.ini_options]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks integration tests that hit a real dependency",
]
```

## `monkeypatch`: Safely Patching for the Duration of a Test

`monkeypatch` is a built-in fixture that patches something — an attribute, an environment
variable, a dict entry, `sys.path` — and **automatically reverts it after the test**, even
if the test fails. That automatic revert is the entire reason to prefer it over patching
things by hand.

```python
def test_uses_env_var(monkeypatch):
    monkeypatch.setenv("API_KEY", "test-key-123")
    assert load_api_key() == "test-key-123"
    # API_KEY is restored to whatever it was (or unset) after this test
```

```python
# Patching an attribute/function
def test_send_email_calls_provider(monkeypatch):
    calls = []
    monkeypatch.setattr(
        "myapp.email.provider.send",
        lambda to, subject, body: calls.append((to, subject)),
    )
    send_welcome_email("ada@example.com")
    assert calls == [("ada@example.com", "Welcome!")]
```

```python
# Patching a dict entry
def test_feature_flag_enabled(monkeypatch):
    monkeypatch.setitem(FEATURE_FLAGS, "new_checkout", True)
    assert is_enabled("new_checkout") is True


# Patching sys.path — useful for testing import-time behavior in isolation
def test_plugin_discovery(monkeypatch, tmp_path):
    monkeypatch.syspath_prepend(str(tmp_path))
    ...


# Deleting an env var / attribute entirely, raising if it doesn't exist
def test_missing_api_key_raises(monkeypatch):
    monkeypatch.delenv("API_KEY", raising=False)
    with pytest.raises(ConfigError):
        load_api_key()
```

!!! tip "Pro Tip"
    `monkeypatch.setattr("myapp.email.provider.send", fn)` patches by *string path*, which
    matters more than it looks: patch the name as it's **looked up at call time**, not
    where it's originally defined. If `myapp/notifications.py` does
    `from myapp.email.provider import send`, that module now has its own local binding —
    patching `myapp.email.provider.send` won't affect it. The fix is to patch
    `myapp.notifications.send` instead, i.e. patch where the name is *used*, not where it
    lives.

## Mocking with `unittest.mock` / `pytest-mock`

`monkeypatch` is best for "replace this with a fixed value or simple stand-in." When a
test needs to assert *how* something was called — call count, arguments, call order —
`unittest.mock.Mock`/`MagicMock` (or the `pytest-mock` plugin's `mocker` fixture, which is
the same thing with automatic teardown built in) is the right tool.

```python
from unittest.mock import MagicMock


def test_notifies_on_signup():
    notifier = MagicMock()
    signup_service = SignupService(notifier=notifier)

    signup_service.register("ada@example.com")

    notifier.send.assert_called_once_with(
        to="ada@example.com", template="welcome"
    )
```

With `pytest-mock`'s `mocker` fixture — same API as `unittest.mock`, but scoped and
auto-reverted like `monkeypatch`, so nothing needs a manual `patch.stop()`:

```python
def test_payment_provider_called(mocker):
    charge = mocker.patch("myapp.billing.provider.charge", return_value={"id": "ch_1"})

    result = process_payment(amount=1000)

    charge.assert_called_once_with(amount=1000)
    assert result["id"] == "ch_1"
```

`Mock`/`MagicMock` also support `side_effect` for raising or returning different values on
successive calls — useful for simulating a flaky dependency or a retry sequence:

```python
def test_retries_on_transient_failure(mocker):
    call = mocker.patch(
        "myapp.client.fetch",
        side_effect=[TimeoutError, TimeoutError, {"status": "ok"}],
    )
    result = fetch_with_retry()
    assert result == {"status": "ok"}
    assert call.call_count == 3
```

!!! warning
    `mocker.patch("module.ClassName")` replaces the *entire class*, which is a much
    blunter instrument than patching one method. Prefer patching the smallest thing that
    makes the test meaningful — a single method or function — so the test doesn't
    accidentally stop exercising real code it was meant to cover. Over-mocking is the
    single most common reason a "green" unit test suite still misses real bugs: mock
    enough of the system and you're testing the mocks, not the code.

## `tmp_path` and `tmp_path_factory`

Tests that touch the filesystem should never write into the repo or a shared temp
directory — `tmp_path` gives each test its own unique, auto-cleaned-up directory:

```python
def test_writes_report(tmp_path):
    output_file = tmp_path / "report.csv"
    generate_report(output_file)
    assert output_file.exists()
    assert "total" in output_file.read_text()
```

`tmp_path` is `function`-scoped (a fresh directory per test). For a directory shared across
several tests in a session — e.g. a wider-scoped fixture that needs its own workspace —
use the `tmp_path_factory` fixture instead:

```python
@pytest.fixture(scope="session")
def shared_workspace(tmp_path_factory):
    return tmp_path_factory.mktemp("workspace")
```

## Unit Test Standards

A few habits that make the difference between a test suite that's trusted and one that
gets `# skip` comments piled onto it over time:

**Arrange / Act / Assert (AAA).** Structure every test in three visible blocks — set up
the world, perform the one action under test, check the outcome. It keeps tests readable
at a glance and makes it obvious when a test is doing too much.

```python
def test_discount_cannot_exceed_total():
    # Arrange
    order = Order(items=[Item(price=50), Item(price=30)])

    # Act
    order.apply_discount(percent=150)

    # Assert
    assert order.total == 0  # clamped, never negative
```

**One assertion *concept* per test** — not literally one `assert` statement (asserting
`result.status == "ok"` and `result.id is not None` in the same test is fine, they're one
concept: "the operation succeeded"), but one *behavior* per test. A test named
`test_create_user` that also asserts email-sending, audit-logging, and cache-invalidation
side effects fails for three unrelated reasons under one confusing name — split it into
`test_create_user_persists_record`, `test_create_user_sends_welcome_email`, etc.

**Name tests after behavior, not implementation.** `test_apply_discount_clamps_at_zero`
tells you what broke from the test name alone, in a CI log, without opening the file.
`test_case_3` or `test_apply_discount_2` doesn't.

**Avoid test interdependence.** Each test must pass regardless of what ran before it and
in whatever order `pytest` decides to run the suite — pass `-p no:randomly` off and
`pytest-randomly` on occasionally in CI specifically to catch order-dependent tests, since
they otherwise hide until someone reorders the suite or runs a single test in isolation. A
test that only passes when run after another test is a bug in the test, not a quirk to
work around.

```python
# BAD — depends on module-level state mutated by test order
created_users = []

def test_create_user():
    created_users.append(create_user("ada"))
    assert len(created_users) == 1  # fails if another test runs first


# GOOD — self-contained via a fixture
def test_create_user(make_user):
    user = make_user("ada")
    assert user.name == "ada"
```

**Keep unit tests fast and dependency-free.** A unit test should not open a real network
connection, hit a real database, or sleep. The moment a test needs a real Postgres
instance or an HTTP call to a real service, it has become an integration test — see
[Integration Testing](integration-testing.md) for where that boundary belongs and how to
set those tests up without slowing down the whole suite.

## Summary

- `pytest` discovers `test_*.py` files and `test_*` functions automatically; configure
  `testpaths` and `--strict-markers` in `pyproject.toml` so typos and misplaced files fail
  loudly instead of silently.
- Fixtures are requested by name as test parameters; `yield`-based fixtures get automatic
  teardown; scope (`function`/`class`/`module`/`session`) trades isolation for speed.
- `conftest.py` shares fixtures down a directory tree without imports — put broad fixtures
  high, category-specific fixtures in that category's own `conftest.py`.
- `monkeypatch` patches attributes/env vars/dict items/`sys.path` with automatic revert;
  patch where a name is *used*, not where it's defined.
- `unittest.mock`/`pytest-mock` are for asserting call behavior (count, args, order); don't
  reach for them when a simple `monkeypatch.setattr` return value would do.
- `@pytest.mark.parametrize` turns one test body into many independently-reported cases;
  custom marks need registering (with `--strict-markers`) to catch typos.
- AAA structure, one behavior per test, behavior-based naming, and zero interdependence
  between tests are what keep a suite trustworthy as it grows.

## Related Articles

- [Playwright End-to-End Testing](playwright-e2e-testing.md) — pytest fixtures applied to
  browser automation, further up the testing pyramid.
- [Integration Testing](integration-testing.md) — where fixtures meet real dependencies
  (test containers, real databases) instead of mocks.
- [Python Tips & Tricks](../python/python-tips.md) — GIL, threading, and multiprocessing
  background relevant to testing concurrent code.
- [FastAPI's Event Loop](../web-development/fastapi-event-loop.md) — relevant when unit
  testing async route handlers and services.
