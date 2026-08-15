---
title: Testing
tags:

- testing
- overview

---

# Testing

Testing fundamentals: how to write trustworthy unit tests, how to drive real
browsers for end-to-end coverage, and how to test components together against
real dependencies without slowing the whole suite to a crawl.

## Articles

- [Pytest: Fixtures, Monkeypatching, and Unit Test Standards](pytest-fixtures-monkeypatch.md)
  — fixtures and scope, `conftest.py`, `monkeypatch`, parametrization, marks, mocking, and
  the habits (AAA, naming, test isolation) that keep a suite trustworthy.
- [Playwright for End-to-End Testing](playwright-e2e-testing.md) — the browser/context/page
  model, locators and auto-waiting, the Page Object Model, trace/screenshot/video
  debugging, network mocking, and running in CI.
- [Integration Testing](integration-testing.md) — where integration tests sit in the
  testing pyramid, real dependencies vs. test doubles, testcontainers, transactional
  rollback for DB tests, and when integration tests are worth their cost.

## Contributing

Learned a testing pattern, a debugging trick, or hit a gotcha worth keeping? Add it here
as its own file. See [Contributing](../../contributing.md) for the how-to.
