---
tags:

- testing
- python
- integration-testing
- pytest
- docker

---

# Integration Testing

Integration tests verify that two or more real components actually work together — a
service and its database, a client and a real HTTP API, a producer and a consumer — in
places where a unit test's mocks might quietly agree with each other while the real
systems wouldn't. They're the middle layer of the testing pyramid: pricier than unit tests,
cheaper than a full end-to-end browser run.

## Where Integration Tests Sit

```text
        /\
       /  \      e2e (few) — full app, real browser, slowest, most brittle
      /----\
     /      \    integration (some) — real DB/queue/HTTP, no UI
    /--------\
   /          \  unit (many) — pure logic, mocked dependencies, fastest
  /------------\
```

The classic "testing pyramid" argues for many fast unit tests, a moderate number of
integration tests, and few, carefully chosen e2e tests (the shape the diagram implies).
Some teams instead favor a **testing trophy** — a bigger integration layer than pyramid
orthodoxy suggests, on the reasoning that integration tests catch the bugs that actually
happen in practice (wiring, serialization, real query behavior) more often than either
unit tests or e2e tests do, per test written. Which shape fits better is genuinely
context-dependent (see [When Integration Tests Earn Their
Cost](#when-integration-tests-earn-their-cost) below) — but both agree on the same point:
integration tests exist specifically to catch what mocks can't.

A unit test proves your code calls `repository.save(user)` correctly, assuming
`repository.save` behaves as mocked. An integration test proves that when your code calls
the *real* repository against a *real* database, the row actually lands with the right
types, constraints, and encoding. Both catch real bugs; neither substitutes for the other.

## Real Dependencies vs. Test Doubles

Three flavors of "not the real thing," worth naming precisely because the differences
matter when choosing one:

- **Stub** — returns canned, fixed responses; no logic, no verification of how it was
  called. `def get_price(): return 100`.
- **Mock** — like a stub, but records calls so the test can *assert on interaction*
  (call count, arguments, order). This is what `unittest.mock.Mock` gives you — see
  [Mocking with `unittest.mock`](pytest-fixtures-monkeypatch.md#mocking-with-unittestmock-pytest-mock).
- **Fake** — a working, simplified implementation of the real thing (an in-memory dict
  standing in for a database), with real behavior, not canned answers.

```python
# Fake — actually behaves like a repository, just backed by a dict instead of a DB
class FakeUserRepository:
    def __init__(self):
        self._users: dict[int, dict] = {}
        self._next_id = 1

    def save(self, user: dict) -> int:
        user_id = self._next_id
        self._users[user_id] = user
        self._next_id += 1
        return user_id

    def get(self, user_id: int) -> dict | None:
        return self._users.get(user_id)
```

Integration tests are precisely the layer where the choice of double matters most:

| Approach | Speed | Fidelity | When it's the right call |
|---|---|---|---|
| Mock/stub | Fastest | Lowest — assumes the real thing behaves as mocked | Unit tests; integration tests for an unreliable third party |
| In-memory fake (SQLite for Postgres, `fakeredis`) | Fast | Medium — same interface, different engine internals | Fast feedback loop, when engine-specific behavior isn't under test |
| Real dependency via testcontainers | Slower | Highest — the actual engine, actual constraints/behavior | Whenever engine-specific SQL, migrations, or concurrency semantics matter |

!!! warning
    SQLite-as-a-Postgres-stand-in is a common shortcut, and a common source of "passes in
    tests, fails in production" bugs: JSON column behavior, `ON CONFLICT` semantics, window
    functions, and case-sensitivity all differ between the two engines. It's a reasonable
    speed trade-off for logic that doesn't touch those areas — not a safe universal
    substitute for "test against Postgres."

## Testcontainers: Real Dependencies in CI

**Testcontainers** spins up real, disposable Docker containers (Postgres, Redis, Kafka,
...) for the duration of a test run — the same engine that runs in production, without a
shared, stateful test environment that different test runs (or different developers)
would otherwise step on each other's toes in.

```python
import pytest
from testcontainers.postgres import PostgresContainer
import sqlalchemy


@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:16") as container:
        yield container


@pytest.fixture(scope="session")
def db_engine(postgres_container):
    engine = sqlalchemy.create_engine(postgres_container.get_connection_url())
    run_migrations(engine)  # apply the real schema once per session
    return engine
```

`scope="session"` here is deliberate — starting a fresh container per test would be
correct but painfully slow (containers take real wall-clock seconds to boot). The
container starts once for the whole run; per-test *data* isolation is handled separately,
below.

See [Docker Fundamentals](../devops-tools/docker.md) for the container mechanics
testcontainers builds on, and
[Fixture Scope](pytest-fixtures-monkeypatch.md#scope-how-often-a-fixture-runs) for why a
mutable, session-scoped fixture like this needs its *data* reset carefully between tests.

## Test Data Setup, Teardown, and Transactional Rollback

A session-scoped database means every test shares the same schema and, without care, the
same rows — which reintroduces exactly the test-interdependence problem
[unit test standards](pytest-fixtures-monkeypatch.md#avoid-test-interdependence) warn
against, just at the database layer. The standard fix is to wrap each test in a
transaction that's rolled back at the end, so the database is untouched no matter what the
test did to it:

```python
@pytest.fixture
def db_session(db_engine):
    connection = db_engine.connect()
    transaction = connection.begin()
    session = sqlalchemy.orm.Session(bind=connection)

    yield session

    session.close()
    transaction.rollback()   # undoes every write this test made
    connection.close()


def test_save_user_persists_email(db_session):
    repo = UserRepository(db_session)
    repo.save(User(email="ada@example.com"))

    assert repo.find_by_email("ada@example.com") is not None
    # rollback happens automatically after this test returns —
    # the next test starts from a clean table
```

This is fast (no container restart, no `TRUNCATE` between tests) and fully isolates each
test's writes, with one caveat: code under test that itself calls `commit()` or opens a
*second* connection (a background job, a separate process) won't see the uncommitted data
inside the wrapping transaction, and won't be undone by rolling it back either. For tests
that specifically need to exercise commit boundaries or true concurrency, fall back to
`TRUNCATE`-and-reseed between tests instead of transactional rollback.

```python
@pytest.fixture(autouse=True)
def clean_tables(db_engine):
    yield
    with db_engine.begin() as conn:
        for table in reversed(metadata.sorted_tables):  # respect FK order
            conn.execute(table.delete())
```

!!! tip "Pro Tip"
    Build test data through the same factory functions/fixtures the unit tests use (see
    [Fixture Factories](pytest-fixtures-monkeypatch.md#fixture-factories)) rather than
    hand-rolled `INSERT`s per integration test file. It keeps "what does a valid user look
    like" defined in exactly one place, so a new required column doesn't mean updating
    dozens of integration tests individually.

## Testing Against Other Services: Redis, Message Queues

The same testcontainers pattern extends to any real dependency:

```python
from testcontainers.redis import RedisContainer


@pytest.fixture(scope="session")
def redis_container():
    with RedisContainer() as container:
        yield container


@pytest.fixture
def redis_client(redis_container):
    client = redis.Redis(
        host=redis_container.get_container_host_ip(),
        port=redis_container.get_exposed_port(6379),
    )
    yield client
    client.flushall()  # per-test cleanup, cheap for Redis specifically
```

For a queue/broker (Kafka, RabbitMQ), the integration test typically verifies the
*contract* your code has with the broker — a message published in the expected format is
consumable, offsets/acks behave as the consumer code assumes — which matters most for
systems built around
[event-driven architecture](../architecture/event-driven-architecture.md), where
[idempotent consumer behavior](../databases/idempotency.md) is exactly the kind of thing
that's easy to get subtly wrong and that a mock would never catch.

## Contract Testing, Briefly

When two services are developed by different teams (or deployed independently), an
integration test against a real running instance of the other service isn't always
practical in CI. **Contract testing** (e.g. Pact) is the alternative: the consumer defines
the request/response shape it expects, the provider's test suite verifies it still
satisfies that expectation — so both sides can evolve independently while a broken
contract is caught in CI on *either* side, without either team standing up the other
team's full service to test against. It's a specialized tool worth reaching for
specifically at service boundaries owned by different teams; for a single team's own
internal service-to-database or service-to-service calls within the same codebase, a
regular integration test against the real dependency is simpler and gives more direct
confidence.

## When Integration Tests Earn Their Cost

Integration tests are worth their slower runtime and setup complexity when:

- The bug they'd catch is specifically about the *interaction* — an ORM query that's
  semantically wrong only against the real engine, a migration that doesn't apply cleanly,
  serialization that loses precision going through a real message broker.
- The dependency has behavior that's genuinely hard to fake correctly (transaction
  isolation levels, upsert/conflict semantics, full-text search).
- Mocking the dependency would mean re-implementing enough of its real behavior in the
  mock that the mock itself becomes a maintenance burden and a source of false confidence.

They're a poor trade when:

- The logic under test doesn't actually depend on the real engine's behavior — a pure
  calculation or validation rule wrapped in a DB-backed repository call is better tested as
  a plain unit test against a fake, saving the container startup cost for tests that
  actually need it.
- The same behavior is already covered end-to-end by a Playwright test that exercises the
  full stack — see [Playwright End-to-End Testing](playwright-e2e-testing.md). Duplicating
  coverage across every layer of the pyramid doesn't add confidence proportional to the
  extra CI minutes it costs.
- The suite is slow enough that developers start skipping it locally and only find out it
  failed after pushing — at that point, either the container startup needs
  session-scoping (as above), or some of those tests genuinely belong one layer down as
  faster unit tests against a fake.

!!! note
    This is a judgment call, not a formula — "does this integration test protect against a
    bug that unit tests structurally can't see, and is that risk worth its runtime cost"
    is the actual question, and the honest answer changes as a codebase and its dependency
    versions change.

## Summary

- Integration tests sit between unit tests (mocked, fast, many) and e2e tests (real
  browser, slow, few) — they verify real components actually work together, without a
  full UI.
- Fakes (working simplified implementations) sit between mocks/stubs (canned responses)
  and the real dependency in both fidelity and speed — pick based on what behavior the
  test actually needs to exercise.
- Testcontainers give real Postgres/Redis/Kafka instances in CI, session-scoped for
  startup speed, with per-test isolation handled separately via transactional rollback or
  table truncation.
- Contract testing solves the "two independently-deployed services" version of this
  problem when standing up a real integration test against the other service isn't
  practical.
- Write an integration test when the interaction itself is the risk; prefer a unit test
  with a fake when it isn't, and prefer relying on e2e coverage when the same path is
  already exercised end-to-end.

## Related Articles

- [Pytest: Fixtures, Monkeypatching, and Unit Test Standards](pytest-fixtures-monkeypatch.md)
  — the fixture mechanics (scope, factories, `conftest.py`) that integration test setups
  build directly on.
- [Playwright End-to-End Testing](playwright-e2e-testing.md) — the layer above integration
  tests in the pyramid, and where duplicated coverage stops paying for itself.
- [Docker Fundamentals](../devops-tools/docker.md) — the container mechanics testcontainers
  is built on.
- [Idempotency](../databases/idempotency.md) — exactly the kind of retry/at-least-once
  behavior that's worth an integration test against a real broker rather than a mock.
- [Event-Driven Architecture](../architecture/event-driven-architecture.md) — the
  consumer/producer systems where integration tests against a real broker matter most.
