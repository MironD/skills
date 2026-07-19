---
name: testing
description: Use when reviewing the quality of tests in a PR — coverage, structure, test doubles, flakiness, and severity calibration for missing or weak tests.
---

# Test Quality

Load this skill alongside the task whenever you're reviewing the quality of tests in a PR. It complements the review skill: this document covers WHAT to look for in the tests themselves, while the loaded review skill owns the output format, severity labels, and the mechanics of publishing findings.

Examples are in Go (`testing`, `testify`). The baseline Go testing conventions — table-driven tests, `t.Parallel`, `t.Helper`, `httptest`, `t.TempDir`, `-race` — are covered in depth in §11 of the `go` skill and are not repeated here; they're simply applied to concrete quality questions.

---

## 1. The Test Pyramid and Choosing a Level

Not every test earns its keep. Before you judge the quality of an individual test, confirm it lives at the right level at all.

```
        /  E2E  \          — few, slow, expensive, most realistic
       / Integration \     — moderate count, exercise boundaries between components
      /   Unit tests   \   — many, fast, isolated, exercise logic
```

| What's under test | Right level | Wrong level (red flag) |
|---|---|---|
| Business logic (calculations, rules) | **Unit** | E2E test to verify a formula → `minor` |
| Persistence to the database | **Integration** | Unit with a mocked repository → `nit` (unless it's the only test) |
| Full flow of an HTTP endpoint | **Integration** | Unit with a mocked request → `minor` |
| Authorization / authentication rules | **Integration** | Unit only → `minor` (integration needed too) |
| Interaction across several services | **E2E / contract test** | Unit with everything around it mocked → `minor` |
| Pure function | **Unit** | Integration with a real DB to check a map/filter → `nit` |

### Behavior vs. Implementation

**The core principle:** a test verifies **WHAT** the code does (its behavior), not **HOW** it does it (its implementation).

**Signs you're testing implementation (red flags):**
- The test checks the **order** of method calls on a mock.
- The test checks the **number** of calls (`AssertNumberOfCalls`).
- A refactor that preserves behavior **breaks** the test.
- Mock expectations are set on internal or private methods.
- The test asserts on intermediate state rather than the final result.

**Signs you're testing behavior (good):**
- The test checks **input → output**.
- Refactoring the internal structure leaves the test **green**.
- The test reads like a **specification** of the behavior.
- The name alone tells you exactly what's being verified.

---

## 2. Coverage Analysis

### When Missing Tests Are a Problem

Not every change needs tests. But there are categories of code where the absence of tests is a **blocker**.

**Must be covered (missing → `major`/`request_changes`):**
- Business logic — calculations, rules, state transitions, validation.
- Authorization and authentication — who gets access and who doesn't.
- Payment logic — charges, deductions, refunds, discounts.
- Data mutations — creating, updating, deleting entities.
- Integrations with external services — request/response format, error handling.
- Bug fixes — every fix **must** ship with a test that reproduces the original bug.

**Tests optional (missing → `nit` or ignore):**
- Trivial getters with no logic.
- Config files, route registration (as long as integration tests cover them).
- Auto-generated code (mock generators, protobuf stubs).
- Pass-through proxy/delegation with no transformation.

### Coverage Is Not Quality

A high coverage percentage **does not** mean good tests. 100% coverage with tests that assert nothing is worse than 60% coverage with meaningful checks.

```go
// Bad: 100% coverage, 0% quality — the test verifies nothing
func TestCreateOrder(t *testing.T) {
    svc := NewOrderService(repo)
    svc.CreateOrder(ctx, validData) // no assertion at all
}

// Good: the test verifies a specific behavior
func TestCreateOrder_PersistsOrderWithPendingStatus(t *testing.T) {
    svc := NewOrderService(repo)

    order, err := svc.CreateOrder(ctx, validData)

    require.NoError(t, err)
    assert.Equal(t, StatusPending, order.Status)
    assert.Equal(t, 1500, order.TotalCents)
}
```

**What to look for in review:**
- Tests that invoke the code but never check the result.
- Tests that only assert `assert.NotNil` or `assert.True(t, true)`.
- Coverage earned by "walking through" the code rather than verifying behavior.
- No negative tests — only the happy path is covered.

---

## 3. Structure and Naming

### Arrange-Act-Assert

Every test cleanly separates three phases. That separation makes the test readable and lets a reviewer grasp what's being verified at a glance.

```go
func TestTransferMoney_InsufficientBalance_ReturnsError(t *testing.T) {
    // Arrange
    sender := NewAccount(100)
    receiver := NewAccount(0)

    // Act
    err := TransferMoney(sender, receiver, 200)

    // Assert
    assert.ErrorIs(t, err, ErrInsufficientBalance)
    assert.Equal(t, 100, sender.Balance()) // balance unchanged
}
```

**Anti-pattern: interleaved arrange-act-assert:**

```go
// Bad: asserts between act steps, unclear what's actually under test
func TestWorkflow(t *testing.T) {
    user := createUser(t)
    assert.True(t, user.Exists())          // assert #1 — why here?
    user.Activate()
    assert.True(t, user.IsActive())        // assert #2
    order := user.CreateOrder(data)
    assert.NotNil(t, order)                // assert #3
    order.Pay()
    assert.Equal(t, "paid", order.Status)  // assert #4
}
```

A test like this does too much at once. Split it into separate tests: `TestActivateUser`, `TestCreateOrder`, `TestPayOrder`.

### The Name Is the Documentation

The test name should make three things obvious: **what** is under test, **under what condition**, and **the expected result**.

```
Good:
  TestTransferMoney_InsufficientBalance_ReturnsError
  TestCreateOrder_ExpiredCoupon_IgnoresDiscount
  TestParseConfig_EmptyFile_ReturnsDefaults
  TestLogin_WrongPassword_Returns401

Bad:
  TestTransfer        — what about transfer?
  TestOrderWorks       — "works" says nothing
  TestCase1             — completely uninformative
  TestBugFix            — which bug?
```

### One Concept Per Test

One test, one **concept** — not necessarily one `assert` call. Multiple assertions are fine when they check **the same behavior** from different angles.

```go
// Good: two assertions, one concept — "order created successfully"
func TestCreateOrder_ValidData_PersistsOrder(t *testing.T) {
    order, err := svc.CreateOrder(ctx, validData)

    require.NoError(t, err)
    assert.Equal(t, StatusPending, order.Status)
    assert.Equal(t, validData.CustomerID, order.CustomerID)
}

// Bad: two distinct concepts in one test
func TestCreateOrder(t *testing.T) {
    order, err := svc.CreateOrder(ctx, validData)
    require.NoError(t, err)
    assert.Equal(t, StatusPending, order.Status)

    // ...and now we also check the update — that's a different test
    updated, err := svc.UpdateOrder(ctx, order.ID, newData)
    require.NoError(t, err)
    assert.Equal(t, newData.Name, updated.Name)
}
```

### Grouping Scenarios with `t.Run`

When several scenarios exercise the same function or method under different conditions, group them under one parent test and drive each scenario through its own subtest instead of writing N standalone top-level functions. The parent name carries the **what**; each subtest name carries the **condition and result**.

```go
func TestService_CreateIdentity(t *testing.T) {
    t.Run("shouldReturn_ConflictError", func(t *testing.T) {
        repo := NewInMemoryIdentityRepo(withExisting(existingIdentity))
        svc := NewService(repo)

        _, err := svc.CreateIdentity(ctx, duplicateData)

        assert.ErrorIs(t, err, ErrConflict)
    })

    t.Run("shouldPersist_WhenDataIsValid", func(t *testing.T) {
        repo := NewInMemoryIdentityRepo()
        svc := NewService(repo)

        identity, err := svc.CreateIdentity(ctx, validData)

        require.NoError(t, err)
        assert.Equal(t, validData.Email, identity.Email)
    })
}
```

This is an alternative to the flat `TestXxx_Condition_Result` naming above, not a replacement for it — reach for it when several scenarios share the same subject (`CreateIdentity`) so a reviewer sees its whole behavior surface at once, and `go test -run TestService_CreateIdentity -v` runs exactly that surface. For many similar cases with identical assertions and only the inputs/outputs varying, prefer a table-driven test (§4.5) over a wall of individual `t.Run` blocks.

**Rules:**
- Each subtest still needs its own Arrange-Act-Assert and its own single concept — grouping under one parent doesn't excuse mixing concerns.
- `t.Parallel()` is safe at both the parent and subtest level as long as subtests don't share mutable fixtures (see Isolation below).
- Subtest names stay condition-first (`shouldReturn_X`, `shouldPersist_X`) so `-run` patterns and `go test -v` output stay grep-able; don't repeat the parent test's name inside the subtest name.

**In review:** a parent test with a single subtest (no real grouping) is a `nit` — just use a flat test name. A parent test whose subtests share state through closures over parent-scoped variables instead of each building its own fixture is a `minor` (breaks isolation, see below).

### Isolation and Independence

Tests **must not** depend on one another. Each one has to pass on its own, regardless of execution order or what other tests did — otherwise `t.Parallel()` becomes a liability.

**Signs isolation is broken:**
- Shared mutable state between tests (a package variable, state in a singleton).
- Test A creates data that test B relies on.
- A test passes only after some other specific test.
- A test fails when run in parallel.

```go
// Bad: shared state at package scope
var testOrders []Order // shared across every test in the file!

func TestCreateOrder(t *testing.T) {
    testOrders = append(testOrders, createOrder())
    assert.Len(t, testOrders, 1) // fails if another test already appended
}

func TestListOrders(t *testing.T) {
    assert.Len(t, testOrders, 1) // depends on TestCreateOrder and run order
}
```

```go
// Good: each test builds its own data, parallel execution is safe
func TestCreateOrder(t *testing.T) {
    t.Parallel()
    repo := NewInMemoryOrderRepo()
    repo.Save(createOrder())
    assert.Len(t, repo.All(), 1)
}

func TestListOrders(t *testing.T) {
    t.Parallel()
    repo := NewInMemoryOrderRepo()
    repo.Save(createOrder())
    assert.Len(t, repo.FindAll(), 1)
}
```

---

## 4. Quality Signals and Anti-Patterns

### What Makes a Test Good

- **Tests behavior, not implementation** — it checks WHAT the code does, not HOW.
- **Reads like documentation** — understandable without reading the code under test.
- **Fails for the right reason** — it goes red when the business logic breaks, not when the order of JSON fields changes.
- **Fast and deterministic** — the same result on every run, with no dependence on the clock, the network, the filesystem, or execution order.

### 4.1. Testing Implementation Details (Brittle Tests)

The test is bound to the code's internal structure. Any refactor breaks it, even when behavior is unchanged.

```go
// Bad: the test asserts that a specific internal method is called with specific args
func TestProcessPayment(t *testing.T) {
    gw := new(mockGateway)
    gw.On("validateCard", "4111111111111111").Return(true)
    gw.On("charge", mock.Anything, mock.Anything).Return(nil)

    svc := NewPaymentService(gw)
    err := svc.ProcessPayment(payment)

    assert.NoError(t, err)
    gw.AssertExpectations(t)
}

// Good: we test the outcome — the payment went through
func TestProcessPayment_ValidCard_Succeeds(t *testing.T) {
    gw := newFakeGateway(alwaysApprove)
    svc := NewPaymentService(gw)

    result, err := svc.ProcessPayment(payment)

    require.NoError(t, err)
    assert.Equal(t, PaymentStatusApproved, result.Status)
}
```

### 4.2. Shotgun Assertions

The test checks every field of an object and every side effect, even the ones irrelevant to the behavior under test.

```go
// Bad: asserting EVERYTHING, including things unrelated to creation
func TestCreateUser(t *testing.T) {
    user := createUser(t, "John", "j@example.com")

    assert.NotEmpty(t, user.ID)
    assert.Equal(t, "John", user.Name)
    assert.Equal(t, "j@example.com", user.Email)
    assert.False(t, user.CreatedAt.IsZero())
    assert.False(t, user.UpdatedAt.IsZero())
    assert.True(t, user.IsActive)
    assert.Equal(t, "user", user.Role)
    assert.Empty(t, user.AvatarURL)
    assert.True(t, user.LastLogin.IsZero())
    assert.Zero(t, user.LoginCount)
    // ...20 more fields
}

// Good: assert what matters — the user was created with the right data
func TestCreateUser_PersistsWithDefaultRole(t *testing.T) {
    user := createUser(t, "John", "j@example.com")

    assert.Equal(t, "John", user.Name)
    assert.Equal(t, "j@example.com", user.Email)
    assert.Equal(t, "user", user.Role)
    assert.True(t, user.IsActive)
}
```

### 4.3. Tests Without Assertions

The most dangerous anti-pattern: a test that **always** passes because it checks nothing. It manufactures a false sense of safety.

```go
// Bad: no assertion at all — always green
func TestSendNotification(t *testing.T) {
    svc := NewNotificationService()
    svc.Send(user, message)
    // ...that's it. What's being verified here?
}

// Good: verify the notification was actually created
func TestSendNotification_CreatesNotificationRecord(t *testing.T) {
    svc := NewNotificationService(notificationRepo)

    svc.Send(user, message)

    got := notificationRepo.FindLastForUser(user.ID)
    require.NotNil(t, got)
    assert.Equal(t, message.Subject, got.Subject)
}
```

### 4.4. Flaky Tests

Tests that are unstable — passing sometimes, failing others. The usual culprits:

**Dependence on the real clock:**
```go
// Bad: depends on the wall clock
func TestTokenExpiry(t *testing.T) {
    token := GenerateToken(time.Now().Add(1 * time.Second))
    time.Sleep(2 * time.Second) // flaky: under load it may not elapse in time
    assert.True(t, token.IsExpired())
}

// Good: abstract the clock
func TestTokenExpiry(t *testing.T) {
    clock := NewFakeClock(time.Date(2025, 1, 1, 0, 0, 0, 0, time.UTC))
    token := GenerateToken(clock, clock.Now().Add(1*time.Hour))

    clock.Advance(2 * time.Hour)

    assert.True(t, token.IsExpired())
}
```

**Dependence on ordering:**
```go
// Bad: map key order is not guaranteed
func TestGetUserRoles(t *testing.T) {
    roles := GetUserRoles(userID)
    assert.Equal(t, []string{"admin", "editor"}, roles) // order may differ
}

// Good: compare without regard to order
func TestGetUserRoles(t *testing.T) {
    roles := GetUserRoles(userID)
    assert.ElementsMatch(t, []string{"admin", "editor"}, roles)
}
```

**Dependence on an external service:**
```go
// Bad: a real call to an external API inside a unit test
func TestFetchExchangeRate(t *testing.T) {
    rate, err := svc.FetchRate(ctx, "USD", "EUR")
    require.NoError(t, err)
    assert.Greater(t, rate, 0.0)
    // if the external API is down, the test fails through no fault of the code
}
```

### 4.5. Copy-Paste Tests Instead of Parametrization

The same test logic is repeated with different inputs — copy-paste where a table-driven test belongs.

```go
// Bad: three identical tests with different data
func TestValidateEmail_NoAt(t *testing.T)     { assert.False(t, ValidateEmail("invalid")) }
func TestValidateEmail_NoDomain(t *testing.T) { assert.False(t, ValidateEmail("user@")) }
func TestValidateEmail_NoLocal(t *testing.T)  { assert.False(t, ValidateEmail("@domain.com")) }

// Good: table-driven test (see §11 of the go skill)
func TestValidateEmail_RejectsInvalidFormat(t *testing.T) {
    t.Parallel()
    tests := []struct {
        name  string
        email string
    }{
        {"no at", "invalid"},
        {"no domain", "user@"},
        {"no local", "@domain.com"},
        {"empty", ""},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            assert.False(t, ValidateEmail(tt.email))
        })
    }
}
```

### 4.6. Testing the Framework Instead of the Business Logic

The test verifies that the ORM or HTTP client works, rather than an application business rule.

```go
// Bad: testing that GORM/the database driver can persist a row
func TestSaveUser(t *testing.T) {
    user := &User{Name: "John", Email: "j@example.com"}
    err := db.Create(user).Error
    require.NoError(t, err)

    var got User
    db.First(&got, "email = ?", "j@example.com")
    assert.Equal(t, "John", got.Name)
}

// Good: testing a business rule
func TestRegisterUser_SendsWelcomeEmail(t *testing.T) {
    mailer := &fakeMailer{}
    svc := NewRegistrationService(repo, mailer)

    err := svc.Register(ctx, "John", "j@example.com")

    require.NoError(t, err)
    require.Len(t, mailer.Sent, 1)
    assert.Equal(t, "j@example.com", mailer.Sent[0].To)
}
```

### 4.7. Over-Mocking

When everything around the code is mocked, the test really only confirms that the mocks return what you told them to return. A real bug sails right through.

```go
// Bad: everything mocked — the test checks only the "glue" between mocks
func TestCreateOrder(t *testing.T) {
    repo := new(mockOrderRepo)
    payment := new(mockPaymentService)
    notifier := new(mockNotifier)
    logger := new(mockLogger)
    cache := new(mockCache)

    repo.On("Save", mock.Anything).Return(nil)
    payment.On("Charge", mock.Anything).Return(nil)
    notifier.On("Notify", mock.Anything).Return(nil)
    logger.On("Info", mock.Anything).Return()
    cache.On("Invalidate", mock.Anything).Return(nil)

    svc := NewOrderService(repo, payment, notifier, logger, cache)
    err := svc.CreateOrder(ctx, orderData)

    assert.NoError(t, err)
    repo.AssertCalled(t, "Save", mock.Anything)
    // what did we actually verify? That the code called 5 methods in the right order
}
```

**Rule: more than 3 mocks in a single test signals a design problem.** The function under test most likely carries too many responsibilities and needs to be decomposed.

---

## 5. Edge Cases and Boundary Testing

Good tests cover more than the happy path — they cover the boundaries.

### nil and Missing Values

```go
func TestCalculateDiscount_NilPrice_ReturnsError(t *testing.T) {
    _, err := CalculateDiscount(nil, 10)
    assert.ErrorIs(t, err, ErrPriceRequired)
}
```

### Empty Collections and Strings

```go
func TestCalculateAverage_EmptySlice_ReturnsZero(t *testing.T) {
    avg := CalculateAverage([]float64{})
    assert.Equal(t, 0.0, avg) // no panic, no NaN
}

func TestParseCSV_EmptyString_ReturnsEmptyResult(t *testing.T) {
    result, err := ParseCSV("")
    require.NoError(t, err)
    assert.Empty(t, result.Rows)
}
```

### Boundary Values

Boundary values are the single most common source of bugs.

| Type | Values to check |
|---|---|
| Numbers | `0`, `-1`, `1`, `math.MaxInt`, `math.MinInt`, negatives |
| Strings | `""`, single character, maximum length, unicode |
| Collections | empty, single element, maximum count |
| Dates | epoch, far future, leap year, DST transition |

```go
func TestPaginate_Boundaries(t *testing.T) {
    t.Parallel()
    // the store holds 25 items
    tests := []struct {
        name    string
        page    int
        perPage int
        want    int
    }{
        {"first page", 1, 10, 10},
        {"last partial page", 3, 10, 5},
        {"beyond last page", 4, 10, 0},
        {"page zero falls back to first", 0, 10, 10},
        {"negative page falls back to first", -1, 10, 10},
        {"zero per page falls back to default", 1, 0, 25},
        {"one per page", 1, 1, 1},
        {"huge per page", 1, 1000, 25},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            got := repo.Paginate(tt.page, tt.perPage)
            assert.Len(t, got.Items, tt.want)
        })
    }
}
```

### Unicode and Special Characters

```go
func TestSlugify(t *testing.T) {
    t.Parallel()
    tests := []struct{ name, input, want string }{
        {"basic", "Hello World", "hello-world"},
        {"cyrillic", "Привіт Світ", "privit-svit"},
        {"cjk", "日本語テスト", "ri-ben-yu-tesuto"},
        {"double space", "hello  world", "hello-world"},
        {"html tags stripped", "<script>alert(1)</script>", "scriptalert1script"},
        {"empty", "", ""},
        {"truncated to max length", strings.Repeat("a", 1000), strings.Repeat("a", 255)},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            assert.Equal(t, tt.want, Slugify(tt.input))
        })
    }
}
```

### Concurrent Access

```go
func TestCounter_ConcurrentIncrement_IsThreadSafe(t *testing.T) {
    counter := NewCounter()
    var wg sync.WaitGroup

    for range 1000 {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Increment()
        }()
    }

    wg.Wait()
    assert.Equal(t, 1000, counter.Value())
}
```

Run tests like this with `-race` (see §11 of the `go` skill) — otherwise a data race stays invisible even while the test is green.

### Error Paths

```go
// The error path is often forgotten — and that's exactly where bugs live
func TestCreateOrder_DatabaseFailure_RollsBack(t *testing.T) {
    repo := &failingOrderRepo{failOnSave: true}
    svc := NewOrderService(repo)

    _, err := svc.CreateOrder(ctx, validData)

    require.ErrorIs(t, err, ErrOrderCreationFailed)
    assert.False(t, repo.WasCalled("Commit"))
}
```

### Timeouts and Retries

```go
func TestFetchDataWithRetry_RetriesThreeTimes(t *testing.T) {
    client := &stubHTTPClient{
        responses: []stubResponse{
            {err: context.DeadlineExceeded},
            {err: context.DeadlineExceeded},
            {body: `{"data":"ok"}`},
        },
    }

    result, err := FetchDataWithRetry(ctx, client, "https://api.example.com/data")

    require.NoError(t, err)
    assert.Equal(t, "ok", result.Data)
    assert.Equal(t, 3, client.CallCount())
}

func TestFetchDataWithRetry_AllRetriesExhausted_ReturnsError(t *testing.T) {
    client := &stubHTTPClient{alwaysErr: context.DeadlineExceeded}

    _, err := FetchDataWithRetry(ctx, client, "https://api.example.com/data")

    assert.ErrorIs(t, err, ErrServiceUnavailable)
}
```

---

## 6. Test Double Strategy

### What to Use, and When

| Test double | Purpose | When to use |
|---|---|---|
| **Stub** | Returns a canned response | You need to control what a dependency feeds in |
| **Mock** | Verifies that a method was called | The side effect matters (sending an email, writing to a queue) |
| **Fake** | A simplified but working implementation | In-memory repository, fake HTTP server |
| **Spy** | Records calls for later inspection | You need to confirm a call happened without controlling its inputs |

**Selection priority:** Fake > Stub > Spy > Mock.

Fakes are the most reliable because they carry real (if simplified) logic. Mocks are the most brittle because they're coupled to implementation details.

### Generated Mocks: `gomock` and `mockery`

Hand-writing a fake stops being pleasant once an interface has a dozen methods. For interfaces that size, generate a mock instead of writing one by hand:

- **`gomock`** (`go.uber.org/mock/gomock` + `mockgen`) — expectation-based. Calls go through an injected `*gomock.Controller`, which fails the test automatically on an unexpected or missing call.
- **`mockery`** — generates `testify/mock`-compatible mocks from an interface definition; same `.On(...)` / `.AssertExpectations(t)` API already used in the examples above.

```go
// gomock, generated via: //go:generate mockgen -source=repo.go -destination=mocks/repo_mock.go
func TestCreateOrder_PersistsOrder(t *testing.T) {
    ctrl := gomock.NewController(t)
    repo := mocks.NewMockOrderRepo(ctrl)
    repo.EXPECT().Save(gomock.Any()).Return(nil)

    svc := NewOrderService(repo)
    err := svc.CreateOrder(ctx, validData)

    require.NoError(t, err)
}
```

**What to check in review, regardless of generator:**
- The mock is generated for an **interface the consumer defines** (small, purpose-shaped), not a wide interface mocked wholesale because one method is used — that's a design smell the generator just makes convenient to ignore.
- Generated files carry a `//go:generate` directive above the source interface and are never hand-edited — a diff touching `*_mock.go` / `mocks/*.go` by hand is a red flag; the fix belongs in the source interface, then regenerate.
- The mock's return values still match the real dependency's shape (see "Mock Behavior Must Match Reality" below) — generation only guarantees the call signature compiles, not that the stubbed response is realistic.
- Generated mocks still count toward the **more-than-3-mocks-per-test** design-smell rule below. Generation makes mocks cheap to create, not free of the coupling problem they introduce.

```go
// FAKE: the best choice for a repository
type InMemoryOrderRepo struct {
    mu     sync.Mutex
    orders map[string]*Order
}

func NewInMemoryOrderRepo() *InMemoryOrderRepo {
    return &InMemoryOrderRepo{orders: make(map[string]*Order)}
}

func (r *InMemoryOrderRepo) Save(order *Order) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.orders[order.ID] = order
    return nil
}

func (r *InMemoryOrderRepo) FindByID(id string) (*Order, error) {
    r.mu.Lock()
    defer r.mu.Unlock()
    order, ok := r.orders[id]
    if !ok {
        return nil, ErrNotFound
    }
    return order, nil
}

// A test backed by the fake — clean, verifies real behavior
func TestCreateOrder_PersistsOrder(t *testing.T) {
    repo := NewInMemoryOrderRepo()
    svc := NewOrderService(repo)

    order, err := svc.CreateOrder(ctx, validData)

    require.NoError(t, err)
    persisted, err := repo.FindByID(order.ID)
    require.NoError(t, err)
    assert.Equal(t, validData.CustomerID, persisted.CustomerID)
}
```

### Diagnosing Over-Mocking

**Symptoms:**
- A test has more mock-setup lines than assertions.
- More than 3 mocks in a single test.
- Changing the call order (not the logic!) breaks the test.
- The test never catches real bugs.

**More than 3 mocks = a design problem.** If testing one method requires stubbing five dependencies, that method is taking on too many responsibilities. Extract the business logic into a pure function, or split the type.

```go
// Bad: 5 mocks — the service does too much
func TestProcessOrder(t *testing.T) {
    repo, payment, notification, logger, metrics := new(mockRepo), new(mockPayment), new(mockNotification), new(mockLogger), new(mockMetrics)
    payment.On("Charge", mock.Anything).Return(PaymentResult{Success: true}, nil)
    repo.On("Save", mock.Anything).Return(nil)
    // ...more setup

    svc := NewOrderService(repo, payment, notification, logger, metrics)
    svc.Process(ctx, order)
    // the test checks calls between mocks, not the real logic
}

// Good: the extracted business logic tests with no mocks at all
func TestCalculateOrderTotal_AppliesDiscount(t *testing.T) {
    total := CalculateOrderTotal(
        []Item{{Price: 10000, Qty: 2}, {Price: 5000, Qty: 1}},
        Discount{Percent: 10},
    )
    assert.Equal(t, 22500, total)
}
```

### Mock Behavior Must Match Reality

```go
// Bad: the mock returns something the real service never would
gateway.On("Charge", mock.Anything).Return(map[string]string{"status": "ok"}, nil)
// the real API returns a PaymentResult struct, not a map

// Good: the mock behaves like the real service
gateway.On("Charge", mock.Anything).Return(PaymentResult{
    Status:        PaymentApproved,
    TransactionID: "txn_123",
}, nil)
```

### Integration Tests for Critical Paths

Unit tests with mocks don't prove the system works as a whole. Critical paths need a real database and real HTTP.

```go
func TestOrderRepository_Save_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    db := setupTestDB(t)
    repo := NewPostgresOrderRepo(db)
    order := NewOrder(WithCustomerID("cust_1"), WithTotal(1500))

    err := repo.Save(ctx, order)
    require.NoError(t, err)

    loaded, err := repo.FindByID(ctx, order.ID)
    require.NoError(t, err)
    assert.Equal(t, order.CustomerID, loaded.CustomerID)
    assert.Equal(t, order.Total, loaded.Total)
}
```

### Contract Tests for External Services

When a service depends on an external API, contract tests guarantee that the expected response format hasn't changed out from under you.

```go
// Verify the client correctly parses the provider's real response format
func TestPaymentClient_ParsesProviderResponse(t *testing.T) {
    raw := []byte(`{
        "id": "txn_abc",
        "status": "approved",
        "amount": {"value": 1500, "currency": "USD"},
        "created_at": "2025-01-01T00:00:00Z"
    }`)

    result, err := ParsePaymentResponse(raw)

    require.NoError(t, err)
    assert.Equal(t, "txn_abc", result.TransactionID)
    assert.Equal(t, PaymentApproved, result.Status)
    assert.Equal(t, Cents(1500), result.Amount)
}
```

---

## 7. Database Testing

### Isolation via Rollback

Every test must leave the database in exactly the state it found it.

```go
func TestOrderRepository(t *testing.T) {
    db := setupTestDB(t)
    tx, err := db.Begin()
    require.NoError(t, err)
    t.Cleanup(func() { tx.Rollback() }) // whatever the outcome, changes are rolled back

    repo := NewOrderRepo(tx)
    // ... the test runs inside a transaction that never commits
}
```

### Test Data via Builders

Tests shouldn't create data through raw SQL or a constructor with dozens of parameters — that's noise, and it's brittle every time a new column appears.

```go
// Go: builder pattern for test data
type orderOption func(*Order)

func NewTestOrder(opts ...orderOption) *Order {
    order := &Order{
        ID:         uuid.NewString(),
        CustomerID: "default-customer",
        Status:     StatusPending,
        Total:      1000,
        CreatedAt:  time.Now(),
    }
    for _, opt := range opts {
        opt(order)
    }
    return order
}

func WithStatus(s Status) orderOption { return func(o *Order) { o.Status = s } }
func WithTotal(t int) orderOption     { return func(o *Order) { o.Total = t } }

// Usage: only what matters to this specific test is visible
order := NewTestOrder(WithStatus(StatusPaid), WithTotal(5000))
```

### Testing Migrations

Migrations are code that reshapes the production database. They need tests just as much as business logic does.

```go
func TestMigration0042_AddDiscountColumn(t *testing.T) {
    db := setupMigrationTestDB(t)

    // "Before" state — the column doesn't exist yet
    require.NoError(t, migrateTo(db, "0041"))
    assert.False(t, columnExists(db, "orders", "discount_cents"))

    // Forward
    require.NoError(t, migrateTo(db, "0042"))
    assert.True(t, columnExists(db, "orders", "discount_cents"))

    // Rollback
    require.NoError(t, migrateTo(db, "0041"))
    assert.False(t, columnExists(db, "orders", "discount_cents"))
}
```

### Test Queries, Not the ORM/Driver's CRUD

Test the **business queries** — the tricky `WHERE`s, the `JOIN`s, the aggregations — not the fact that the library can insert a row.

```go
func TestFindOverdue_ReturnsOnlyUnpaidPastDue(t *testing.T) {
    db := setupTestDB(t)
    repo := NewOrderRepo(db)

    seedOrder(t, db, WithStatus(StatusPending), WithDueAt(time.Now().Add(-24*time.Hour))) // overdue
    seedOrder(t, db, WithStatus(StatusPending), WithDueAt(time.Now().Add(24*time.Hour)))  // not yet due
    seedOrder(t, db, WithStatus(StatusPaid), WithDueAt(time.Now().Add(-24*time.Hour)))    // paid, not overdue
    seedOrder(t, db, WithStatus(StatusPending), WithDueAt(time.Now().Add(-7*24*time.Hour))) // overdue

    overdue, err := repo.FindOverdue(ctx)

    require.NoError(t, err)
    assert.Len(t, overdue, 2)
    for _, o := range overdue {
        assert.Equal(t, StatusPending, o.Status)
    }
}
```

---

## 8. API / HTTP Testing

### Assert the Status Code

Every API test **must** check the HTTP status — it's the first line of defense against regressions.

```go
func TestCreateOrder_ValidPayload_Returns201(t *testing.T) {
    resp := doPost(t, "/api/v1/orders", validPayload)
    assert.Equal(t, http.StatusCreated, resp.StatusCode)
}

func TestCreateOrder_InvalidPayload_Returns422(t *testing.T) {
    resp := doPost(t, "/api/v1/orders", map[string]any{})
    assert.Equal(t, http.StatusUnprocessableEntity, resp.StatusCode)
}

func TestCreateOrder_Unauthenticated_Returns401(t *testing.T) {
    resp := doPostNoAuth(t, "/api/v1/orders", validPayload)
    assert.Equal(t, http.StatusUnauthorized, resp.StatusCode)
}

func TestCreateOrder_ReadOnlyUser_Returns403(t *testing.T) {
    resp := doPostAs(t, readOnlyUser, "/api/v1/orders", validPayload)
    assert.Equal(t, http.StatusForbidden, resp.StatusCode)
}
```

### Response Body Structure

```go
func TestListOrders_ResponseStructure(t *testing.T) {
    resp := doGet(t, "/api/v1/orders", authHeaders)
    require.Equal(t, http.StatusOK, resp.StatusCode)

    var data struct {
        Items   []map[string]any `json:"items"`
        Total   int              `json:"total"`
        Page    int              `json:"page"`
        PerPage int              `json:"per_page"`
    }
    require.NoError(t, json.NewDecoder(resp.Body).Decode(&data))

    if len(data.Items) > 0 {
        order := data.Items[0]
        assert.Contains(t, order, "id")
        assert.Contains(t, order, "status")
        assert.Contains(t, order, "total")
        assert.NotContains(t, order, "internal_notes") // a sensitive field must not leak
    }
}
```

### Error Format

Errors should come back in a consistent shape — tests are what guarantee it.

```go
func TestCreateOrder_ValidationError_ReturnsStructuredError(t *testing.T) {
    resp := doPost(t, "/api/v1/orders", map[string]any{
        "customer_id": "",
        "total":       -1,
    })

    assert.Equal(t, http.StatusUnprocessableEntity, resp.StatusCode)

    var body ErrorResponse
    require.NoError(t, json.NewDecoder(resp.Body).Decode(&body))

    assert.Equal(t, "validation_error", body.Code)
    assert.Contains(t, body.Errors, FieldError{Field: "customer_id", Message: "must not be empty"})
    assert.Contains(t, body.Errors, FieldError{Field: "total", Message: "must be positive"})
}
```

### Authorization Matrix

Each endpoint deserves its own test matrix: who gets in and who doesn't.

```go
func TestDeleteOrder_Authorization(t *testing.T) {
    t.Parallel()
    tests := []struct {
        role     string
        wantCode int
    }{
        {"admin", http.StatusNoContent},
        {"manager", http.StatusNoContent},
        {"editor", http.StatusForbidden},
        {"viewer", http.StatusForbidden},
        {"guest", http.StatusUnauthorized},
    }
    for _, tt := range tests {
        t.Run(tt.role, func(t *testing.T) {
            t.Parallel()
            user := createUserWithRole(t, tt.role)
            order := createOrder(t)

            resp := doDeleteAs(t, user, "/api/v1/orders/"+order.ID)

            assert.Equal(t, tt.wantCode, resp.StatusCode)
        })
    }
}
```

### Pagination

```go
func TestListOrders_PaginationDefaults(t *testing.T) {
    seedOrders(t, 50)

    resp := doGet(t, "/api/v1/orders", authHeaders)
    var data paginatedResponse
    json.NewDecoder(resp.Body).Decode(&data)

    assert.Equal(t, 20, data.PerPage) // default
    assert.Len(t, data.Items, 20)
    assert.Equal(t, 50, data.Total)
}

func TestListOrders_PaginationCustomPage(t *testing.T) {
    seedOrders(t, 50)

    resp := doGet(t, "/api/v1/orders?page=3&per_page=20", authHeaders)
    var data paginatedResponse
    json.NewDecoder(resp.Body).Decode(&data)

    assert.Len(t, data.Items, 10) // the remainder
    assert.Equal(t, 3, data.Page)
}
```

---

## 9. Regression Testing

### A Bug Fix Must Ship With a Test That Reproduces the Bug

Without such a test, there's no guarantee the bug won't come back.

**In review:** if a PR is described as a bug fix but no tests were added or changed → `major`.

```go
// Good: a test that was red first (reproducing the bug),
// and green after the fix
func TestCalculateTotal_HandlesZeroQuantityItems(t *testing.T) {
    // Regression: #1234 — an item with quantity=0 caused a divide-by-zero
    // in the average-price calculation.
    items := []Item{
        {Price: 10000, Quantity: 2},
        {Price: 5000, Quantity: 0}, // this is the element that triggered the panic
    }

    total := CalculateTotal(items)

    assert.Equal(t, 20000, total)
}
```

### Refactoring Shouldn't Change Test Behavior

If the PR is a refactor:
- The tests **should not** change, provided they test behavior rather than implementation.
- If the tests did change, then either the refactor quietly altered behavior (which needs explaining in the PR) or the old tests were brittle (they tested implementation).

**In review:** a PR claims to be a refactor but the tests were rewritten — ask whether behavior changed. If it didn't, the old tests were testing the wrong thing; if it did, this isn't a refactor but a functional change that deserves its own description.

### When to Add a Regression Test

- Bug fix → **always**, with a test that reproduces the original bug.
- An uncovered edge case you found → add a test before fixing it.
- A production incident → a test that models the incident scenario.
- A discovered security vulnerability → a test that verifies the fix itself.

---

## 10. Test Maintenance

### DRY vs. Readability

In production code, DRY is nearly a law. In tests, **excessive DRY hurts readability**. A test should read as a self-contained document, without forcing you to hop through five helper functions.

**The balance:**
- **Extract into helpers:** test data construction (builders/factories), setup/teardown, repeated assertions for domain concepts.
- **Keep inline:** the arrange-act-assert logic itself and the concrete values under test.

```go
// Bad: over-DRY — unreadable without jumping around the codebase
func TestCreateOrder(t *testing.T) {
    result := executeStandardOrderCreation(t, scenarioHappyPath)
    assertStandardOrderResult(t, result, expectedHappyPath)
}
// to understand this test you have to read four other helpers

// Good: helpers for data, but the test logic stays inline
func TestCreateOrder_WithDiscount(t *testing.T) {
    user := createUser(t, withRole("premium"))
    product := createProduct(t, withPrice(10000))

    order, err := svc.Create(ctx, user, []Product{product}, WithCoupon("SAVE10"))

    require.NoError(t, err)
    assert.Equal(t, 9000, order.TotalCents)
    assert.Equal(t, StatusPending, order.Status)
}
```

### Custom Assertions for Domain Concepts

Repeated checks of business logic are worth pulling into custom assertion helpers — you get DRY and readability at once.

```go
func assertOrderCompleted(t *testing.T, order *Order) {
    t.Helper()
    assert.Equal(t, StatusCompleted, order.Status, "order should be completed")
    assert.False(t, order.CompletedAt.IsZero(), "completed_at should be set")
    assert.True(t, order.CompletedAt.After(order.CreatedAt), "completed_at should be after created_at")
}

func TestCompleteOrder(t *testing.T) {
    order := NewTestOrder(WithStatus(StatusPaid))

    err := svc.CompleteOrder(ctx, order.ID)

    require.NoError(t, err)
    updated, _ := repo.FindByID(order.ID)
    assertOrderCompleted(t, updated) // one line instead of three assertions
}
```

### Cleaning Up After Tests

Tests that create external state (files, DB rows, data in external services) **must** clean up after themselves — `t.TempDir()` and `t.Cleanup()` do this automatically (see §11 of the `go` skill).

```go
func TestExportReport_CreatesFile(t *testing.T) {
    dir := t.TempDir() // removed automatically after the test
    path := filepath.Join(dir, "report.csv")

    err := ExportReport(path, testData)

    require.NoError(t, err)
    content, err := os.ReadFile(path)
    require.NoError(t, err)
    assert.Contains(t, string(content), "Order ID,Total")
}
```

---

## 11. Severity Calibration

### `critical` — Blocker, `request_changes` Guaranteed

- Critical business logic (payments, authorization, data mutations) **with no tests**.
- Existing functionality's tests deleted **with no explanation** (the functionality itself wasn't removed).
- An always-green test (`assert.True(t, true)`, a test with no assertions) on a critical path.
- A payment/authorization test that doesn't exercise the failure scenarios.

### `major` — Blocker Unless Explained

- New functionality with no tests.
- A bug fix with no regression test.
- A broken (failing) test added in this same PR.
- A flaky test added in this same PR (dependence on time or ordering).
- Tests removed during a refactor without replacement.
- A test that covers only the happy path for a function that handles errors.
- An integration test swapped for a fully mocked unit test on a critical path.

### `minor` — Worth Fixing, Not a Blocker

- Test naming that doesn't follow the "what_under-which-condition_result" convention.
- A missing edge case (empty value, nil, boundary) on a non-critical path.
- A test that could be parametrized (copy-paste with different data).
- Over-mocking (>3 mocks) — a design-problem signal.
- Missing cleanup after a test (temp files, test data).
- An overly long arrange section — time for a factory/builder.
- Assertions that check implementation rather than behavior.

### `nit` — Cosmetic, Not a Blocker

- Inconsistent test-naming style within a single file.
- Swapped argument order (`expected`, `actual`).
- Redundant comments in a simple test.
- Unused test fixtures/helpers.
- Uninformative test-data names (`data1`, `data2`).

---

## 12. Test Review Checklist

This checklist is for the reviewer's own use. **Do not publish it in the PR** — it exists so you can work through the tests systematically and miss nothing.

### Coverage and Completeness
- [ ] Every new/changed piece of business logic is covered by a test
- [ ] The bug fix has a regression test that reproduces the original bug
- [ ] The happy path is covered
- [ ] Error paths are covered (invalid input, dependency failure, timeout)
- [ ] Boundary conditions are covered (nil, empty value, edge numbers)
- [ ] Authorization is checked for new endpoints

### Structure and Quality
- [ ] Tests follow Arrange-Act-Assert
- [ ] Test names are informative
- [ ] One test, one concept
- [ ] Tests are isolated (no shared mutable state)
- [ ] Tests are deterministic (no dependence on time, network, order)
- [ ] Tests verify behavior, not implementation details
- [ ] Grouped scenarios use `t.Run` with condition-first subtest names, each with its own fixture

### Anti-Patterns
- [ ] No tests without assertions
- [ ] No shotgun assertions
- [ ] No testing the framework instead of the business logic
- [ ] No over-mocking (>3 mocks per test)
- [ ] No flaky patterns (`time.Sleep`, real HTTP, order-dependent asserts)
- [ ] No copy-paste tests that could be reduced to table-driven

### Test Doubles
- [ ] The test-double choice is appropriate (fake for a repository, mock for side effects)
- [ ] Mock behavior matches the dependency's real behavior
- [ ] Generated mocks (`gomock`/`mockery`) target consumer-defined interfaces and are never hand-edited
- [ ] Critical paths have integration tests (real DB, real HTTP)
- [ ] Not everything is mocked indiscriminately

### Database Tests
- [ ] Tests are isolated via transaction rollback
- [ ] Test data is built through builders
- [ ] Business queries are tested, not basic CRUD
- [ ] Migrations have tests (forward and rollback)

### API Tests
- [ ] Status codes are checked for every scenario
- [ ] Response structure is validated
- [ ] Error format is consistent
- [ ] The authorization matrix is covered

### Maintenance
- [ ] Test code isn't over-DRY (reads without jumping between files)
- [ ] Complex test data uses builders
- [ ] Domain assertions are extracted into helpers
- [ ] Cleanup after tests (files, external state)

---

## Additional: Quick Reference

### Minimum Set of Checks for Any PR With Tests

1. **Are there assertions?** A test without them is `critical`/`major`.
2. **Is behavior being tested?** Checking the order of mock calls is a brittle-test tell.
3. **Is the error path covered?** Happy path only → `minor`/`major` depending on how critical the code is.
4. **Is there a regression test for the bug fix?** None → `major`.
5. **Is the test deterministic?** `time.Sleep`, real time, real HTTP are flakiness tells.

### Red Flags (Grab Attention Immediately)

| What you see | What it means | Severity |
|---|---|---|
| Test with no `assert`/`require` | The test verifies nothing | `major` |
| `time.Sleep` in a test | Flaky, time-dependent | `minor`/`major` |
| `mock.Anything` on every argument | The test doesn't check concrete values | `minor` |
| More than 3 mock setups | Over-mocking, a design problem | `minor` |
| `assert.True(t, true)` / a bare `assert.NotNil` | A fake test | `major` |
| A test with no matching source change | A test for nonexistent code, or new logic without coverage | `info` |
| Copy-paste blocks of tests | Should be reduced to table-driven | `nit`/`minor` |
| `t.Skip()` with no explanation | A hidden broken test | `major` |
| A deleted test with no functionality removed | Loss of coverage | `major` |
| Hand-edited `*_mock.go` / `mocks/*.go` file | Drift from the generator, will be silently overwritten | `minor` |
| Subtest names repeat the parent test name | Redundant with `t.Run` nesting, hurts `-run`/`-v` output | `nit` |
