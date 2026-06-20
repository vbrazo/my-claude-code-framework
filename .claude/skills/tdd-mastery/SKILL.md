---
name: tdd-mastery
description: Test-driven development — the red-green-refactor loop, applied across JavaScript, Python, and Go
---

# Test-driven development

## The loop: red, green, refactor

1. **Red** — write a failing test that captures the behavior you want
2. **Green** — write just enough code to make it pass
3. **Refactor** — tidy up while the tests stay green

No production code goes in without a failing test ahead of it, and each lap through the loop should take only a few minutes.

## Structuring a test

Reach for arrange-act-assert every time:

```
Arrange: Set up test data and dependencies
Act:     Execute the behavior under test
Assert:  Verify the expected outcome
```

Name a test for what it pins down — `test_<unit>_<scenario>_<expected_result>` or `it("should <behavior> when <condition>")`.

## Jest / Vitest

```typescript
describe("OrderService", () => {
  it("should apply discount when order exceeds threshold", () => {
    const order = createOrder({ items: [{ price: 150, qty: 1 }] });
    const result = applyDiscount(order, { threshold: 100, percent: 10 });
    expect(result.total).toBe(135);
  });

  it("should throw when applying discount to empty order", () => {
    const order = createOrder({ items: [] });
    expect(() => applyDiscount(order, defaultDiscount)).toThrow(EmptyOrderError);
  });
});
```

Mock with `vi.fn()` / `jest.fn()`, favor dependency injection over module mocking, and lean on `beforeEach` for shared setup — never carry mutable state between tests.

## pytest

```python
@pytest.fixture
def db_session():
    session = create_test_session()
    yield session
    session.rollback()

def test_create_user_stores_hashed_password(db_session):
    user = UserService(db_session).create(email="a@b.com", password="secret")
    assert user.password_hash != "secret"
    assert verify_password("secret", user.password_hash)

@pytest.mark.parametrize("input,expected", [
    ("", False),
    ("short", False),
    ("ValidPass1!", True),
])
def test_password_validation(input, expected):
    assert validate_password(input) == expected
```

Catch exceptions with `pytest.raises`, share fixtures through `conftest.py`, and tag slow tests with `@pytest.mark.slow`.

## Go

```go
func TestParseConfig(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    Config
        wantErr bool
    }{
        {"valid yaml", "port: 8080", Config{Port: 8080}, false},
        {"empty input", "", Config{}, true},
        {"invalid port", "port: -1", Config{}, true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseConfig([]byte(tt.input))
            if (err != nil) != tt.wantErr {
                t.Errorf("ParseConfig() error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            if !tt.wantErr && got != tt.want {
                t.Errorf("ParseConfig() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

Default to table-driven tests, call `t.Helper()` in test utilities, and only pull in `testify/assert` if the team already leans on it.

## Levels of testing

| Level | Scope | Speed | Dependencies |
|-------|-------|-------|-------------|
| Unit | Single function/class | <100ms | None (mock all) |
| Integration | Module boundaries | <5s | Real DB, real FS |
| E2E | Full user flow | <30s | Full stack |

Aim for roughly 70% unit, 20% integration, 10% e2e.

## Coverage

- Hold the line at **80% line coverage minimum** in CI
- Watch branch coverage, not just line coverage
- Leave out generated code, type definitions, and config files
- Don't write tests to chase a number — test behavior

```bash
# Jest/Vitest
vitest run --coverage --coverage.thresholds.lines=80 --coverage.thresholds.branches=75

# pytest
pytest --cov=src --cov-fail-under=80 --cov-branch

# Go
go test -coverprofile=cover.out -coverpkg=./... ./...
go tool cover -func=cover.out
```

## Mocking

- Mock at the boundaries — HTTP clients, databases, filesystems, clocks
- Never mock the unit you're testing
- For repositories, prefer in-memory fakes over mocks
- Assert on behavior, not on how many times a mock was called
- Reset shared mocks with `t.Cleanup` / `afterEach`

## What to avoid

- Asserting on implementation details rather than behavior
- Tautological tests that still pass after you delete the code
- Mutable state shared across test cases
- Tolerating flaky tests instead of fixing them
- Reaching in to test private methods directly
- Sprawling setup that buries what the test is actually checking
