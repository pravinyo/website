---
title: "TDD Fundamentals: Red-Green-Refactor"
date: 2026-09-02 10:00:00 +0530
categories: [Blogging, Article, ai-engineering]
tags: [softwareengineering, backenddevelopment, python, test-driven-development, ai-assisted-development]
author: pravin_tripathi
readtime: true
media_subpath: /assets/img/ai-assisted-development/
attachment_path: /assets/document/attachment/ai-assisted-development/
mermaid: true
image:
  path: header.png
  width: 1600
  height: 900
  alt: Generated using ChatGPT
---

## TL;DR

- TDD (Test-Driven Development) means writing a failing test *before* writing any implementation code — this forces you to think about correctness first. The engine is Red-Green-Refactor: write a failing test (🔴), make it pass with minimal code (🟢), then clean up (🔵).
- The Arrange-Act-Assert (AAA) pattern gives every test a clear, readable structure that is easy to review, debug, and extend.
- **Fixtures** eliminate setup duplication across tests, **parametrize** runs one test against many inputs, and **mocking** lets you test code in isolation by replacing real external dependencies (APIs, databases, file systems) with controlled fakes.
- **Coverage** tells you which lines your tests actually exercise, and GitHub Actions automates running your full suite on every push — turning testing discipline into an enforced team habit rather than a personal one.
- Tests are a **machine-readable specification**. That property matters for humans, and — as this series builds toward — it's the seed of the more formal spec-driven workflows in posts 2 and 3.

---

## Prerequisites

- Basic Python knowledge: classes, functions, exceptions
- A terminal where you can run `pip install pytest pytest-mock pytest-cov`
- Basic understanding of HTTP APIs (what a request and response are) — needed for the mocking section
- No prior testing experience required

---

## A Running Example: PortfolioLedger

This series follows one system end to end so each post builds on real state, not a new toy each time. It's a simplified trading platform:

1. **This post** — `PortfolioLedger`, a single account's buy/sell/position tracking (in-memory, unit-tested), plus `PriceQuoteService`, the external quote API it depends on for live valuation.
2. **Next post** — a nightly PySpark job that takes many accounts' raw broker trade logs and settles them into the clean records `PortfolioLedger` actually consumes in production.
3. **Third post** — a multi-plan brokerage commission engine bolted onto that same settlement pipeline.

Everything here is illustrative, not a real brokerage's accounting rules — but the shapes (average cost, insufficient-quantity errors, external price lookups) are the ones you'll actually hit building this kind of system.

---

## Part 1: The Core Loop

### What Is TDD and Why Does It Exist?

Most developers write code first, then write tests (if they write them at all). TDD reverses that order deliberately.

The reason is simple: **when you write the test first, you define what "correct" looks like before any code exists**. You're forced to think about the interface, edge cases, and expected behaviour before getting absorbed in implementation details.

This discipline matters enormously in an AI-assisted workflow. When you give an AI tool a vague prompt, you get vague code. When you give it a precise test file that defines exactly what the code must do, you get a clear implementation target — and you can verify the result immediately. We'll build directly on that idea for the rest of this series.

### Software Test Types

Before the loop, understand the landscape. There are three levels of testing, each with a different scope:

```
┌──────────────────────────────────────────────────────────┐
│                     TEST PYRAMID                         │
│                                                          │
│                        /\                                │
│                       /  \   End-to-End Tests            │
│                      / E2E \  (slow, test full system)   │
│                     /──────\                             │
│                    /        \  Integration Tests         │
│                   / Integr.  \ (medium, test components) │
│                  /────────────\                          │
│                 /              \  Unit Tests             │
│                /   Unit Tests   \ (fast, test one thing) │
│               /──────────────────\                       │
└──────────────────────────────────────────────────────────┘
```

| Type | What it tests | Speed | Example |
|------|--------------|-------|---------|
| **Unit** | One function or class in isolation | Very fast (ms) | `PortfolioLedger.buy()` |
| **Integration** | Multiple components working together | Moderate (seconds) | Ledger + live `PriceQuoteService` |
| **End-to-End** | The whole system from user's perspective | Slow (minutes) | Nightly settlement job → ledger → statement |

TDD primarily works at the **unit** level. This post starts there, then works its way up to isolating integration-style dependencies with mocks and wiring everything into CI.

### The Red-Green-Refactor Loop

TDD has one engine, used repeatedly:

```mermaid
graph LR
    A["🔴 Write Failing Test"] --> B["🟢 Write Minimal Code"]
    B --> C["🔵 Refactor"]
    C --> A
```

**🔴 Red** — Write a test for behaviour that does not exist yet. Run it. It must fail. If it passes, either the behaviour already exists or the test is wrong.

**🟢 Green** — Write the *minimum* amount of code to make the test pass. No more. Do not add features, do not anticipate future requirements. Just pass the test.

**🔵 Refactor** — Clean up the code while keeping all tests green. Remove duplication, improve naming, extract abstractions. The tests are your safety net — if you accidentally break something, they tell you immediately.

Repeat this loop for every new behaviour you want to add.

### The Arrange-Act-Assert (AAA) Pattern

Every well-written test has three parts:

```
Arrange  →  Set up the objects and state needed for the test
Act      →  Call the function or method being tested
Assert   →  Check that the result matches what you expected
```

This structure makes tests readable at a glance. When a test fails, the three-part structure tells you exactly where to look.

```python
def test_buy_increases_position_quantity():
    # Arrange — create a fresh ledger
    ledger = PortfolioLedger()

    # Act — perform the operation
    ledger.buy("RELIANCE", quantity=10, price=2000.0)

    # Assert — verify the outcome
    assert ledger.position("RELIANCE").quantity == 10
```

---

## Part 2: A Complete TDD Walkthrough — PortfolioLedger

Here is the full TDD workflow for `PortfolioLedger`: a class that tracks buys and sells for a single account, computes weighted-average cost, and reports realized profit/loss. We write every test before any implementation exists.

### Step 1 — Write all the tests (Red)

```python
# test_portfolio_ledger.py
import pytest
from portfolio_ledger import PortfolioLedger  # does not exist yet — that's intentional


def test_new_symbol_has_no_position():
    """A symbol never traded has no open position."""
    ledger = PortfolioLedger()
    assert ledger.position("RELIANCE") is None


def test_buy_creates_position():
    """A single buy opens a position at the buy price."""
    ledger = PortfolioLedger()
    ledger.buy("RELIANCE", quantity=10, price=2000.0)
    position = ledger.position("RELIANCE")
    assert position.quantity == 10
    assert position.avg_cost == 2000.0


def test_buy_averages_cost_across_multiple_buys():
    """A second buy at a different price recomputes the weighted average cost."""
    ledger = PortfolioLedger()
    ledger.buy("RELIANCE", quantity=10, price=2000.0)
    ledger.buy("RELIANCE", quantity=10, price=2400.0)
    position = ledger.position("RELIANCE")
    assert position.quantity == 20
    assert position.avg_cost == pytest.approx(2200.0)


def test_sell_reduces_quantity():
    """Selling part of a position reduces the remaining quantity."""
    ledger = PortfolioLedger()
    ledger.buy("RELIANCE", quantity=10, price=2000.0)
    ledger.sell("RELIANCE", quantity=4, price=2450.0)
    assert ledger.position("RELIANCE").quantity == 6


def test_sell_computes_realized_pnl():
    """Realized P&L accumulates (sell_price - avg_cost) * quantity across sells."""
    ledger = PortfolioLedger()
    ledger.buy("RELIANCE", quantity=10, price=2000.0)
    ledger.sell("RELIANCE", quantity=4, price=2450.0)
    # (2450 - 2000) * 4 = 1800
    assert ledger.realized_pnl == pytest.approx(1800.0)


def test_sell_more_than_held_raises_value_error():
    """Selling more than the open quantity raises ValueError."""
    ledger = PortfolioLedger()
    ledger.buy("RELIANCE", quantity=5, price=2000.0)
    with pytest.raises(ValueError, match="Insufficient quantity"):
        ledger.sell("RELIANCE", quantity=10, price=2450.0)


def test_buy_rejects_non_positive_quantity():
    """Buying a zero or negative quantity raises ValueError."""
    ledger = PortfolioLedger()
    with pytest.raises(ValueError, match="quantity must be positive"):
        ledger.buy("RELIANCE", quantity=0, price=2000.0)


def test_buy_rejects_non_positive_price():
    """Buying at a zero or negative price raises ValueError."""
    ledger = PortfolioLedger()
    with pytest.raises(ValueError, match="price must be positive"):
        ledger.buy("RELIANCE", quantity=10, price=-1.0)
```

Run `pytest test_portfolio_ledger.py` now — all 8 tests fail with `ModuleNotFoundError`. That's the 🔴 Red phase.

### Step 2 — Write minimal code to pass (Green)

```python
# portfolio_ledger.py
from dataclasses import dataclass


@dataclass
class Position:
    """A single symbol's open position within a ledger."""
    quantity: int
    avg_cost: float


class PortfolioLedger:
    """Tracks buys, sells, weighted-average cost, and realized P&L for one account."""

    def __init__(self) -> None:
        """Initialise an empty ledger."""
        self._positions: dict[str, Position] = {}
        self.realized_pnl: float = 0.0

    def position(self, symbol: str) -> Position | None:
        """Return the current open position for a symbol, or None if flat."""
        return self._positions.get(symbol)

    def buy(self, symbol: str, quantity: int, price: float) -> None:
        """Record a buy, updating the weighted-average cost for the symbol.

        Args:
            symbol: Ticker symbol, e.g. "RELIANCE".
            quantity: Number of shares bought. Must be positive.
            price: Price per share. Must be positive.

        Raises:
            ValueError: If quantity or price is not positive.
        """
        if quantity <= 0:
            raise ValueError(f"quantity must be positive, got {quantity}")
        if price <= 0:
            raise ValueError(f"price must be positive, got {price}")

        existing = self._positions.get(symbol)
        if existing is None:
            self._positions[symbol] = Position(quantity=quantity, avg_cost=price)
            return

        total_cost = (existing.quantity * existing.avg_cost) + (quantity * price)
        total_quantity = existing.quantity + quantity
        self._positions[symbol] = Position(
            quantity=total_quantity,
            avg_cost=total_cost / total_quantity,
        )

    def sell(self, symbol: str, quantity: int, price: float) -> None:
        """Record a sell, reducing the position and accumulating realized P&L.

        Args:
            symbol: Ticker symbol.
            quantity: Number of shares sold. Must be positive and <= held quantity.
            price: Price per share. Must be positive.

        Raises:
            ValueError: If quantity exceeds the open position, or inputs are non-positive.
        """
        if quantity <= 0:
            raise ValueError(f"quantity must be positive, got {quantity}")
        if price <= 0:
            raise ValueError(f"price must be positive, got {price}")

        existing = self._positions.get(symbol)
        if existing is None or quantity > existing.quantity:
            held = existing.quantity if existing else 0
            raise ValueError(
                f"Insufficient quantity: holding {held}, requested {quantity}"
            )

        self.realized_pnl += (price - existing.avg_cost) * quantity
        remaining = existing.quantity - quantity
        if remaining == 0:
            del self._positions[symbol]
        else:
            self._positions[symbol] = Position(quantity=remaining, avg_cost=existing.avg_cost)
```

Run `pytest test_portfolio_ledger.py` — all 8 tests pass. 🟢 Green.

### Step 3 — Refactor (Blue)

The validation logic (`quantity <= 0`, `price <= 0`) is already duplicated between `buy` and `sell`. In a larger codebase, this is where you'd extract a `_validate_trade_inputs(quantity, price)` helper — the tests stay green throughout because they only assert on public behaviour, not on how validation is implemented internally.

### Mapping Tests to CRUD Operations

TDD maps naturally onto CRUD (Create, Read, Update, Delete) operations. For `PortfolioLedger`:

| CRUD | What to test |
|------|-------------|
| **Create** | A `buy` on a new symbol opens a position at the buy price |
| **Read** | `position(symbol)` returns the correct quantity and average cost |
| **Update** | A second `buy` recomputes the weighted average; a `sell` reduces quantity and accrues P&L |
| **Delete** | Selling the full quantity removes the position entirely |
| **Validation** | Non-positive quantity/price, and selling more than is held, all raise the right exceptions with meaningful messages |
| **Edge cases** | Selling the exact remaining quantity, buying the same symbol repeatedly at different prices, a flat (never-traded) symbol |

---

## Part 3: Beyond a Single Class

`PortfolioLedger` is self-contained — no external dependencies, no shared state, no setup complexity. Real-world code is messier. In production, `PortfolioLedger` doesn't get its current prices from a hardcoded dict — it calls a **quote service** to value open positions:

- Multiple tests need the same object set up in the same way → repeated `__init__` calls in every test
- Your code calls external APIs, databases, or file systems → tests become slow, flaky, and environment-dependent
- You want to test dozens of input/output combinations → copying the same test function with different values is unsustainable

The rest of this post solves all three problems, then wires the result into CI.

### 1. Fixtures — Shared Setup Without Repetition

A **fixture** is a function that creates and returns a test object. pytest injects it into any test that declares it as a parameter. When a test suite has 20 tests that all need a `PriceQuoteService` instance, one fixture replaces 20 identical `service = PriceQuoteService(...)` lines.

**Basic fixture**

```python
# test_price_quote_service.py
import pytest
from price_quote_service import PriceQuoteService


@pytest.fixture
def service() -> PriceQuoteService:
    """Provide a fresh PriceQuoteService instance for each test."""
    return PriceQuoteService(api_key="test-key")


def test_service_is_created(service: PriceQuoteService) -> None:
    """Fixture injects the service — no manual setup needed."""
    assert service is not None


def test_service_has_correct_api_key(service: PriceQuoteService) -> None:
    assert service.api_key == "test-key"
```

**Fixture scope — controlling how often setup runs**

By default, fixtures run once per test. For expensive setup (e.g., a database connection), you can share the fixture across multiple tests in the same module or session:

```python
@pytest.fixture(scope="module")   # once per test module (file)
def db_connection():
    conn = create_test_db_connection()
    yield conn        # yield instead of return allows teardown
    conn.close()      # teardown runs after all tests in module finish


@pytest.fixture(scope="function") # default — once per test function
def fresh_ledger():
    return PortfolioLedger()
```

**Yield fixtures for teardown**

Use `yield` instead of `return` to run cleanup code after the test completes:

```python
@pytest.fixture
def temp_trade_log(tmp_path):
    """Create a temporary trade-log CSV and clean it up after the test."""
    file = tmp_path / "trades.csv"
    file.write_text("symbol,quantity,price\nRELIANCE,10,2450.0\nTCS,5,3800.0\n")
    yield file          # test runs here
    # cleanup happens automatically because tmp_path manages the directory
```

### 2. Parametrize — Testing Many Inputs Without Repetition

`@pytest.mark.parametrize` runs the same test logic with multiple input/output combinations. Instead of writing 10 nearly-identical test functions, you write one:

```python
import pytest
from unittest.mock import patch
from price_quote_service import PriceQuoteService


@pytest.fixture
def service() -> PriceQuoteService:
    return PriceQuoteService(api_key="test-key")


@pytest.mark.parametrize("symbol, expected_name", [
    ("RELIANCE", "Reliance Industries"),
    ("TCS",      "Tata Consultancy Services"),
    ("INFY",     "Infosys Limited"),
    ("HDFC",     "HDFC Bank"),
    ("WIPRO",    "Wipro Limited"),
])
def test_get_company_name(
    service: PriceQuoteService,
    symbol: str,
    expected_name: str,
) -> None:
    """Verify company name lookup for multiple NSE symbols."""
    with patch("price_quote_service.requests.get") as mock_get:
        # Arrange — configure the mock to return the expected name
        mock_get.return_value.json.return_value = {"name": expected_name}

        # Act
        result = service.get_company_name(symbol)

        # Assert
        assert result == expected_name
```

This generates **5 separate tests** — one per symbol — each with its own pass/fail result. When one fails, you know exactly which symbol caused it.

### 3. Mocking — Isolating External Dependencies

A **mock** is a fake object that replaces a real dependency during a test. It lets you:

- Test your code without making real network calls
- Control what external services return (including errors)
- Verify that your code calls external services correctly

**The problem mocking solves**

```python
# ❌ Without mocking — this test makes a real HTTP request
def test_get_price_without_mock(service):
    result = service.get_price("RELIANCE")
    assert result == 2450.0
    # Problems: slow, needs internet, fails if API is down,
    #           may cost money, result may change over time
```

**Mocking with `unittest.mock.patch`**

```python
from unittest.mock import patch, MagicMock
import pytest
from price_quote_service import PriceQuoteService


@pytest.fixture
def service() -> PriceQuoteService:
    return PriceQuoteService(api_key="test-key")


def test_get_price(service: PriceQuoteService) -> None:
    """Verify get_price parses the API response correctly."""
    with patch("price_quote_service.requests.get") as mock_get:
        # Configure the mock — define what the fake API returns
        mock_get.return_value.json.return_value = {
            "symbol": "RELIANCE",
            "last_price": 2450.0,
        }
        mock_get.return_value.status_code = 200

        result = service.get_price("RELIANCE")

        assert result == 2450.0
        # Optionally verify the mock was called with the right arguments
        mock_get.assert_called_once_with(
            "https://api.example.com/quote/RELIANCE",
            headers={"Authorization": "Bearer test-key"},
        )


def test_api_failure_raises_connection_error(service: PriceQuoteService) -> None:
    """Verify get_price propagates network failures — the ledger must not silently mis-value a position."""
    with patch("price_quote_service.requests.get") as mock_get:
        # Simulate a network failure
        mock_get.side_effect = ConnectionError("Network unreachable")

        with pytest.raises(ConnectionError, match="Network unreachable"):
            service.get_price("RELIANCE")


def test_api_returns_404_raises_value_error(service: PriceQuoteService) -> None:
    """Verify that a 404 response raises a meaningful ValueError."""
    with patch("price_quote_service.requests.get") as mock_get:
        mock_get.return_value.status_code = 404
        mock_get.return_value.json.return_value = {"error": "Symbol not found"}

        with pytest.raises(ValueError, match="Symbol not found: INVALID"):
            service.get_price("INVALID")
```

**The implementation being tested**

```python
# price_quote_service.py
import requests


class PriceQuoteService:
    """Client for fetching live stock quotes from a broker's quote API."""

    BASE_URL = "https://api.example.com"

    def __init__(self, api_key: str) -> None:
        """Initialise the service with an API key."""
        self.api_key = api_key
        self._headers = {"Authorization": f"Bearer {api_key}"}

    def get_price(self, symbol: str) -> float:
        """Fetch the last traded price for a given NSE stock symbol.

        Args:
            symbol: NSE ticker symbol (e.g., "RELIANCE", "TCS").

        Returns:
            The last traded price as a float.

        Raises:
            ValueError: If the symbol is not found (404 response).
            ConnectionError: If the network request fails.
        """
        url = f"{self.BASE_URL}/quote/{symbol}"
        response = requests.get(url, headers=self._headers)

        if response.status_code == 404:
            raise ValueError(f"Symbol not found: {symbol}")

        return response.json()["last_price"]
```

### 4. Test Coverage — Measuring What You Haven't Tested

Coverage tells you what percentage of your code is executed when your test suite runs. A line not executed by any test is a line that could contain an untested bug.

```bash
pip install pytest-cov

# Run tests with coverage report in terminal
pytest --cov=price_quote_service --cov-report=term-missing test_price_quote_service.py

# Generate an HTML report you can browse
pytest --cov=price_quote_service --cov-report=html test_price_quote_service.py
# Open htmlcov/index.html in a browser
```

Sample output:

```
Name                     Stmts   Miss  Cover   Missing
--------------------------------------------------------
price_quote_service.py     18      2    89%   31, 35
--------------------------------------------------------
TOTAL                      18      2    89%
```

Lines 31 and 35 are not covered — those are the untested paths. Add a test for each, and coverage climbs to 100%.

**What coverage doesn't tell you:** 100% coverage does not mean bug-free code. It means every line was *executed*, not that every logical branch was *verified correctly*. Use coverage as a floor ("nothing goes untested"), not a ceiling ("100% means done").

```python
def value_position(quantity: float, price: float) -> float:
    return quantity * price

# This achieves 100% coverage:
def test_value_position():
    assert value_position(10, 2450.0) == 24500.0

# But value_position(-10, 2450.0) silently returns a negative valuation
# for a short position the ledger never intended to support — untested.
```

### 5. Pre-Flight Checklist Before Every Release

Before merging or deploying, verify:

```
Pre-flight Test Checklist
─────────────────────────
□ pytest passes with zero failures
□ Coverage is above team threshold (commonly 80–90%)
□ No tests are marked xfail (expected failures) without an issue ticket
□ No tests are skipped without a documented reason
□ All mocks are correctly scoped (not leaking between tests)
□ No hardcoded credentials in test files
□ Integration tests pass against a staging environment
```

### 6. CI/CD — Automating the Test Suite with GitHub Actions

Manual test runs are forgotten. Automated pipelines are not. GitHub Actions runs your full test suite on every push and pull request.

**`.github/workflows/test.yml`**

```yaml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        python-version: ["3.11", "3.12"]   # test on multiple Python versions

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-cov pytest-mock

      - name: Run tests with coverage
        run: |
          pytest --cov=. --cov-report=xml --cov-fail-under=80

      - name: Upload coverage report
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
          fail_ci_if_error: false
```

The `--cov-fail-under=80` flag causes the CI job to fail if coverage drops below 80%. This enforces the coverage floor automatically.

**Integration test job (optional second job)**

```yaml
  integration-test:
    runs-on: ubuntu-latest
    needs: test          # only runs if unit tests pass

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -r requirements.txt
      - run: pytest tests/integration/ -v
        env:
          DATABASE_URL: postgresql://postgres:testpassword@localhost/testdb
```

---

## 🤖 AI in Development

### Encode requirements as tests before prompting AI

The most effective way to use AI for implementation is to write your tests first, then pass them to the AI as the specification.

**Without TDD (vague prompt):**
```
Prompt: "Write a PortfolioLedger class in Python"

Result: AI writes something that compiles and runs basic cases,
        but may miss quantity/price validation, may average cost
        incorrectly across buys, may not raise ValueError on
        overselling — you have no contract to verify against.
```

**With TDD (tests-first prompt):**
```
Prompt: "Implement a PortfolioLedger class in Python 3.11 that passes
         all the following tests. Use type hints and docstrings.
         [paste test_portfolio_ledger.py]"

Result: AI has a precise correctness contract. Every test either
        passes or fails — no ambiguity. You can verify the output
        in seconds by running pytest.
```

The key insight: **tests are a machine-readable specification**. They tell AI not just *what* to build, but exactly *how correct looks*. In the next two posts, that same insight gets pushed further — first by having an AI persona write the tests too from plain-English rules, then by formalizing the requirements themselves into a versioned specification document that both humans and AI agents read directly.

### Use AI to generate edge-case tests

AI also excels at generating test cases you haven't thought of, once an implementation exists. The workflow:

1. Write your implementation
2. Prompt AI: *"Review this function and suggest pytest test cases for edge cases, error conditions, and boundary values I may have missed: [paste code]"*
3. Review each suggested test manually before adding it
4. Add the ones that test real behaviour; discard the ones that always pass or test the wrong thing

This combines AI's broad pattern-matching with your domain judgment.

**Prompt template for test generation**

```
Context: I am writing pytest tests for the following Python function.
Function: [paste function]
Existing tests: [paste current test file]

Task: Suggest 5 additional test cases that cover:
  1. Edge cases not in my existing tests
  2. Error conditions (invalid inputs, network failures, empty collections)
  3. Boundary values (zero, negative, maximum)
  4. Concurrency or ordering issues if relevant

Format: Provide each test as a complete pytest function with
        AAA structure, type hints, and a docstring explaining
        what behaviour it verifies.
```

### AI pitfall: mocks that never verify anything

AI-generated mocks often configure a mock object but forget to assert it was actually called:

```python
# ❌ AI-generated test — configures mock but never verifies it was used
def test_sends_trade_confirmation(notification_service):
    with patch("notification_service.smtp.send") as mock_send:
        mock_send.return_value = True
        notification_service.notify("user@example.com", "Your trade executed")
        # No assertion! This test passes even if notify() never calls smtp.send

# ✅ Correct — verify the mock was called with the right arguments
def test_sends_trade_confirmation(notification_service):
    with patch("notification_service.smtp.send") as mock_send:
        mock_send.return_value = True
        notification_service.notify("user@example.com", "Your trade executed")
        mock_send.assert_called_once_with(
            to="user@example.com",
            subject="Trade Notification",
            body="Your trade executed",
        )
```

### Reviewing AI-generated tests

Ask, for every AI-suggested test:

1. Does this test actually assert anything meaningful, or does it always pass?
2. Does the test name describe the behaviour being tested?
3. Is the AAA structure clear?
4. Are edge cases covered, or only the happy path?

A common AI mistake: generating tests that don't actually test failure cases, resulting in a green suite that misses real bugs.

---

## ⚠️ Common Mistakes

**Mistake 1: Writing tests after the code**
The most common deviation from TDD. Tests written after implementation tend to test what the code *does*, not what it *should* do. They confirm existing behaviour rather than define required behaviour.

**Mistake 2: Skipping the Red phase**
If you write a test and it immediately passes without writing any implementation, the test is either testing already-existing behaviour or it's wrong. Always confirm the test fails first.

**Mistake 3: Writing too much code in the Green phase**
The temptation is to write the "complete" solution immediately. Resist. Write the minimum code to pass the current test, then add the next test for the next behaviour. This keeps each change small and reversible.

**Mistake 4: Testing implementation details instead of behaviour**
A test that checks `ledger._positions` directly (a private attribute) is fragile — it breaks whenever the internal implementation changes. Test the public interface: `ledger.position()`, `ledger.buy()`, `ledger.sell()`.

**Mistake 5: Not using `pytest.raises` for exception tests**

```python
# ❌ Wrong — this test will always pass even if no exception is raised
def test_bad():
    try:
        ledger.sell("RELIANCE", quantity=1000, price=2450.0)
    except ValueError:
        pass  # silently passes even if sell didn't raise

# ✅ Correct — test fails if ValueError is NOT raised
def test_good():
    with pytest.raises(ValueError):
        ledger.sell("RELIANCE", quantity=1000, price=2450.0)
```

**Mistake 6: Patching the wrong module path**
Mock the name as it's *imported in the module under test*, not where it's defined.

```python
# price_quote_service.py does: import requests
# ✅ Correct — patch it where it's used
patch("price_quote_service.requests.get")

# ❌ Wrong — patching the source doesn't affect price_quote_service.py
patch("requests.get")
```

**Mistake 7: Forgetting that mocks are per-test by default**
Mocks created inside a `with patch(...)` block are automatically restored when the block exits. If you see a mock leaking between tests, you're likely not using the context manager or fixture correctly.

**Mistake 8: 100% coverage complacency**
Coverage measures execution, not correctness — see the `value_position()` example above. Treat coverage gaps as a todo list, not the definition of "done."

**Mistake 9: Slow fixtures in function scope**
If a fixture takes 2 seconds to set up and you have 100 tests using it, your suite takes 200 seconds unnecessarily. Use `scope="module"` or `scope="session"` for expensive setup that doesn't need to be fresh per test.

**Mistake 10: Not testing the negative paths**
Most developers write tests for the happy path (things that should work). The most valuable tests are often for the paths that should *fail* — invalid inputs, network errors, overselling — because those are where real bugs hide.

---

## 💡 Pro Tips

1. **Name tests as sentences.** `test_sell_more_than_held_raises_value_error` reads as a sentence. This makes test output self-documenting.
2. **One assertion per test (where possible).** A test named `test_sell` that checks quantity, realized P&L, and a notification side effect is really three tests pretending to be one.
3. **Use `pytest.raises(ExceptionType, match="pattern")`.** The `match` parameter is a regex that checks the exception message. This catches cases where code raises the right exception type but with a misleading message.
4. **Run pytest with `-v` for verbose output**, `-x` to stop on first failure, and `-k "keyword"` to run only matching tests — all invaluable while iterating on a specific feature.
5. **Keep tests fast.** Unit tests should run in milliseconds. If a test takes seconds, it's probably doing I/O (network, disk, database) — mock it.
6. **Use `conftest.py` for shared fixtures.** Fixtures defined there are automatically available to all test files in the same directory and subdirectories, with no imports needed.
7. **Name fixtures after what they provide, not how they're created.** `authenticated_user` is better than `setup_user_with_token`.
8. **Separate unit tests from integration tests.** Put them in different directories (`tests/unit/` vs `tests/integration/`). Run unit tests on every save, integration tests only before commits or in CI.

---

## 📚 References

- [pytest official documentation](https://docs.pytest.org/) — The complete pytest reference
- [pytest.raises documentation](https://docs.pytest.org/en/stable/reference/reference.html#pytest.raises) — How to test exceptions correctly
- [pytest fixtures documentation](https://docs.pytest.org/en/stable/how-to/fixtures.html) — The complete fixture reference
- [pytest-mock documentation](https://pytest-mock.readthedocs.io/) — The `mocker` fixture and `MagicMock` patterns
- [unittest.mock — Python docs](https://docs.python.org/3/library/unittest.mock.html) — The standard library mock reference
- [pytest-cov documentation](https://pytest-cov.readthedocs.io/) — Coverage measurement and reporting
- [GitHub Actions — Building and testing Python](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-python) — Official CI setup guide
- [TDD by Example — Kent Beck](https://www.oreilly.com/library/view/test-driven-development/0321146530/) — The book that formalised TDD as a practice
- [Real Python — Getting Started with Testing in Python](https://realpython.com/python-testing/) — Practical intro to Python testing
- [Python type hints — PEP 484](https://peps.python.org/pep-0484/) — Type annotation reference for writing annotated functions

---

## ➡️ Next Steps

You now have the full professional TDD toolkit: the Red-Green-Refactor loop, AAA structure, fixtures, parametrize, mocks, coverage, and CI — all exercised against a single-account `PortfolioLedger`. Real trading platforms don't stop at one account: every night, trades from many accounts across many brokers need to be ingested, validated, and settled before `PortfolioLedger` ever sees them.

**Next post:** [Agentic TDD: Multi-Agent Workflows for Zero-Code Test Generation →](/posts/agentic-tdd-zero-code/)

In the next post, we build that nightly settlement pipeline — but instead of hand-writing the test code the way we did here, you'll write the requirements in plain English, and a QA-persona AI agent will generate the `pytest` suite, with a second Architect-persona agent implementing against it.
