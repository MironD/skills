---
name: architecture
description: Use when reviewing or writing code where the concern is structural — layer boundaries, coupling, SOLID violations, design patterns, API contracts, complexity — rather than syntax or line-level correctness.
---

Bring this skill alongside the task whenever the concern is the structure of the code rather than its syntax.

**Review:** this document describes WHAT to check from an architectural standpoint — layer boundaries, coupling, contracts. The output format, severity labels, and how findings are published are all set by the loaded review skill, not by this document.

**Writing code:** keep these principles in mind while you work, and run the checklist in §11 before handing the result off.

Code examples are Go. Topics already covered in depth by the `go` skill (error handling, `context`, concurrency, testing) are only referenced here — go there for the details.

---

## 1. SOLID in a Go project

### S — a single reason to change

A package or struct should have exactly one reason to change. If a change to a business rule and a change to the API response format both force edits in the same place, the responsibilities are tangled together.

**Common symptoms:** a struct with a dozen public methods serving unrelated scenarios; an HTTP handler where validation, business logic, and the database write all live in one function; a do-everything service like `OrderManager` with `Create`, `Cancel`, `Refund`, `Export`, `Notify`.

```go
// Bad: the handler validates, computes a discount, and writes to the DB itself
func (h *OrderHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    json.NewDecoder(r.Body).Decode(&req)
    if len(req.Items) == 0 {
        http.Error(w, "no items", http.StatusBadRequest)
        return
    }
    discount := 0.0
    if req.User.IsPremium {
        discount = req.Total * 0.1
    }
    order := &Order{UserID: req.User.ID, Total: req.Total - discount}
    h.db.Exec(`INSERT INTO orders ...`, order.UserID, order.Total)
    sendEmail(req.User.Email, "Order created")
    json.NewEncoder(w).Encode(order)
}

// Good: the handler only decodes the input and delegates to a use case
func (h *OrderHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid body", http.StatusBadRequest)
        return
    }
    order, err := h.createOrder.Execute(r.Context(), req.ToCommand())
    if err != nil {
        writeError(w, err)
        return
    }
    json.NewEncoder(w).Encode(order)
}
```

**Quick test:** "if the JSON response format changes, do I have to touch this type? And if the discount rule changes?" Two yes answers about the same type signal an SRP violation.

### O — open for extension, closed for modification

A new behavior should arrive as new code, not as an edit to existing code. The most frequent symptom of a violation is a `switch` or an `if/else if` chain that grows with every PR to accommodate one more type.

```go
// Bad: every new shipping type is an edit to an existing function
func ShippingCost(o Order) float64 {
    switch o.Method {
    case "standard":
        return o.Weight * 1.5
    case "express":
        return o.Weight * 3.0
    case "overnight":
        return o.Weight * 5.0
    }
    return 0
}

// Good: a new calculator registers itself; existing code stays untouched
type ShippingCalculator interface {
    Calculate(o Order) float64
    Supports(method string) bool
}

type ShippingRegistry struct {
    calculators []ShippingCalculator
}

func (r *ShippingRegistry) Calculate(o Order) (float64, error) {
    for _, c := range r.calculators {
        if c.Supports(o.Method) {
            return c.Calculate(o), nil
        }
    }
    return 0, fmt.Errorf("unsupported shipping method: %s", o.Method)
}
```

**Where to stop:** a `switch` with two stable branches that don't grow is perfectly good Go, not a reason to reach for interfaces. OCP becomes critical only once the chain grows on a regular basis.

### L — substitutability

A type that implements an interface must behave so that code written against the interface doesn't break for the concrete implementation. The classic red flag is a method that always returns a "not supported" error for one of the variants.

```go
// Bad: ReadOnlyCache implements Cache, but Set always fails
type Cache interface {
    Get(ctx context.Context, key string) ([]byte, error)
    Set(ctx context.Context, key string, val []byte, ttl time.Duration) error
}

type ReadOnlyCache struct{ source Cache }

func (c *ReadOnlyCache) Set(ctx context.Context, key string, val []byte, ttl time.Duration) error {
    return errors.New("cache is read-only") // breaks the Cache contract
}

// Good: split interfaces that reflect the type's real capabilities
type CacheReader interface {
    Get(ctx context.Context, key string) ([]byte, error)
}

type CacheWriter interface {
    CacheReader
    Set(ctx context.Context, key string, val []byte, ttl time.Duration) error
}

type ReadOnlyCache struct{ source CacheReader } // only Get — and never has to lie
```

### I — narrow interfaces

A client should not depend on methods it never calls. In Go this rule is especially natural: an interface is defined where it is consumed, not where it is implemented (see §6 of the `go` skill).

```go
// Bad: login only needs a lookup by email,
// but the dependency drags in the whole repository
type UserRepository interface {
    GetByID(ctx context.Context, id int64) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    List(ctx context.Context, f UserFilter) ([]*User, error)
    Create(ctx context.Context, u *User) error
    Update(ctx context.Context, u *User) error
    Delete(ctx context.Context, id int64) error
    ExportCSV(ctx context.Context, f UserFilter) ([]byte, error)
}

// Good: authService declares only what it actually needs
type UserByEmailFinder interface {
    GetByEmail(ctx context.Context, email string) (*User, error)
}

type authService struct {
    users UserByEmailFinder
}
```

### D — dependency inversion

Business logic depends on abstractions, not on concrete DB drivers, queue SDKs, or HTTP clients. Constructor injection and manual wiring in `main` are covered in detail in §4 of the `go` skill; what matters here is the architectural consequence.

```go
// Bad: business logic knows about Redis and S3 directly
type OrderService struct {
    cache *redis.Client
    s3    *s3.Client
}

func NewOrderService() *OrderService {
    return &OrderService{
        cache: redis.NewClient(&redis.Options{Addr: "localhost:6379"}),
        s3:    s3.New(session.New()),
    }
}

// Good: dependencies are interfaces; the concrete types are assembled in main
type LimitChecker interface {
    CheckLimit(ctx context.Context, userID int64) error
}

type OrderStorage interface {
    Save(ctx context.Context, o *Order) error
}

type OrderService struct {
    limits  LimitChecker
    storage OrderStorage
}

func NewOrderService(limits LimitChecker, storage OrderStorage) *OrderService {
    return &OrderService{limits: limits, storage: storage}
}
```

**Cost of ignoring DIP:** the business logic can't be tested without standing up real infrastructure, and swapping Redis for anything else forces edits in the service layer.

---

## 2. Coupling between packages

### Signs of tight coupling

A change in one package forces a change in another, even though the two shouldn't be so tightly bound.

1. **Constructing a dependency inline** — a service calls `sql.Open(...)` or `redis.NewClient(...)` itself instead of accepting a ready dependency through its constructor.
2. **God object** — one struct (`App`, `Container`) with dozens of fields that half the codebase depends on.
3. **Import cycle** — package `order` imports `user`, and `user` imports `order`. The Go compiler forbids this outright, so the cycle usually hides indirectly — through a shared package that both pull too much from.
4. **Feature envy** — a method on one type operates mostly on the data of another type.

```go
// Bad: Order reaches into Customer's internals to compute a discount
func (o *Order) DiscountedTotal(c *Customer) float64 {
    discount := 0.0
    if c.LoyaltyPoints > 1000 {
        discount = 0.1
    }
    if c.MemberSince.Before(time.Now().AddDate(-2, 0, 0)) {
        discount += 0.05
    }
    return o.Total * (1 - discount)
}

// Good: the knowledge of its own discount belongs to Customer
func (c *Customer) DiscountRate() float64 {
    rate := 0.0
    if c.LoyaltyPoints > 1000 {
        rate = 0.1
    }
    if c.MemberSince.Before(time.Now().AddDate(-2, 0, 0)) {
        rate += 0.05
    }
    return rate
}

func (o *Order) DiscountedTotal(rate float64) float64 {
    return o.Total * (1 - rate)
}
```

### Signs of low cohesion within a package

- A package does things that don't logically belong together (a `user` package that both creates users and generates PDF reports).
- A junk-drawer package: `util`, `helpers`, `common`, `misc` — an anti-pattern explicitly banned in §2/§17 of the `go` skill too.
- If you can't give a package a concrete name describing WHAT it provides, its cohesion is low.

### Afferent / efferent coupling

- **Afferent (Ca)** — how many packages depend on this one. High Ca = a core of the system; any change here is risky.
- **Efferent (Ce)** — how many packages this one depends on. High Ce = unstable, easily broken by other people's changes.
- **Instability = Ce / (Ca + Ce).** Stable packages (the domain) should not depend on unstable ones (infrastructure) — the same dependency rule as in §4.

### Layer boundary violations

| Violation | Severity |
|---|---|
| Transport → Database (handler runs SQL itself) | critical |
| Domain → Infrastructure (a domain type imports an ORM/DB driver) | critical |
| Domain → Transport (a domain object knows about HTTP statuses) | critical |
| Application → Transport (a use case builds a JSON response) | major |
| Infrastructure → Application (a repository calls a use case) | major |

```go
// Bad: transport reaches straight into the database
func (h *OrderHandler) List(w http.ResponseWriter, r *http.Request) {
    rows, _ := h.db.QueryContext(r.Context(),
        `SELECT id, total FROM orders WHERE user_id = $1 LIMIT 20`, userID(r))
    // ... scanning rows right inside the handler
}

// Good: transport → application → infrastructure
func (h *OrderHandler) List(w http.ResponseWriter, r *http.Request) {
    result, err := h.listOrders.Execute(r.Context(), ListOrdersQuery{UserID: userID(r)})
    if err != nil {
        writeError(w, err)
        return
    }
    json.NewEncoder(w).Encode(result)
}
```

---

## 3. Design patterns — when they help and when they hurt

### Creational

**Factory** is justified when the concrete type is decided at runtime or construction is multi-step. For an ordinary constructor it's overkill — in Go, functional options usually cover optional parameters (see §3 of the `go` skill).

**Builder** makes sense at 5+ optional fields; for 2-3 fields, just pass them to the constructor.

### Structural

**Adapter** — a wrapper around an external API behind an internal interface. Always justified at the boundary with an external system: a payment provider, a mail service, a third-party SDK.

```go
type PaymentGateway interface {
    Charge(ctx context.Context, amount Cents, token string) (Receipt, error)
}

// Adapter over a specific provider — its SDK details don't leak any further
type providerGateway struct {
    client *providersdk.Client
}

func (g *providerGateway) Charge(ctx context.Context, amount Cents, token string) (Receipt, error) {
    resp, err := g.client.CreateCharge(ctx, providersdk.ChargeRequest{
        AmountCents: int64(amount),
        Token:       token,
    })
    if err != nil {
        return Receipt{}, fmt.Errorf("charge: %w", err)
    }
    return Receipt{ID: resp.ID, Status: mapStatus(resp.Status)}, nil
}
```

**Decorator** adds behavior without changing the object itself — the natural choice for cross-cutting concerns: logging, metrics, and caching wrapped around an existing interface.

**Facade** — a simplified entry point into a complex subsystem with several dependencies.

### Behavioral

**Strategy** replaces a growing `switch`/`if-else` with polymorphism. Justified from three or more genuinely distinct behaviors (see the OCP example above).

**Observer / events** decouple the producer from the consumer. Side effects (notifications, analytics) are better emitted as an event than called directly from the main flow.

```go
func (s *OrderService) Complete(ctx context.Context, id OrderID) error {
    order, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return fmt.Errorf("find order: %w", err)
    }
    order.MarkCompleted()
    if err := s.repo.Save(ctx, order); err != nil {
        return fmt.Errorf("save order: %w", err)
    }
    s.events.Publish(ctx, OrderCompleted{OrderID: order.ID})
    // subscribers: send email, update inventory, notify partners
    return nil
}
```

**Command** encapsulates a request as an object — justified for undo/redo, task queues, and audit logs.

### Anti-patterns

| Anti-pattern | Sign | Consequence |
|---|---|---|
| Over-engineering | Abstraction on top of abstraction for "future flexibility" | Code can't be read without jumping between files |
| Pattern cargo-culting | A `Factory` for a single concrete type | Pointless indirection with no payoff |
| Needless abstraction | An interface with one implementation and no clear reason | Harder to navigate the code |
| Premature generality | A generic mechanism built for one current case | A YAGNI violation |

**Decision rule:** a pattern is justified when it solves a problem that already exists — or that will certainly appear in the next PR, not somewhere down the road.

---

## 4. Layer boundaries (hexagonal / clean architecture)

```
Transport (HTTP, gRPC, CLI)  →  Application (use cases)  →  Domain (entities, value objects)  ←  Infrastructure (DB, cache, queue)
```

**Dependency rule:** dependencies always point inward, toward the domain. Infrastructure implements interfaces declared in domain/application, never the other way around.

### Port / Adapter

**Port** — an interface declared in domain or application. **Adapter** — its implementation in infrastructure.

```go
// PORT: internal/domain/order/repository.go
type Repository interface {
    FindByID(ctx context.Context, id OrderID) (*Order, error)
    Save(ctx context.Context, order *Order) error
}

// ADAPTER: internal/infra/postgres/order_repository.go
type OrderRepository struct {
    pool *pgxpool.Pool
}

func (r *OrderRepository) FindByID(ctx context.Context, id order.OrderID) (*order.Order, error) {
    row := r.pool.QueryRow(ctx, `SELECT id, status, total FROM orders WHERE id = $1`, id)
    return scanOrder(row)
}
```

### Keeping the domain model clean

**Critical rule:** the domain package does not import an ORM, DB drivers, HTTP frameworks, or the SDKs of infrastructure services.

```go
// Bad: a domain type carries an ORM tag and persists itself
type Order struct {
    gorm.Model
    Status string
}

func (o *Order) Complete(db *gorm.DB) error {
    o.Status = "completed"
    return db.Save(o).Error // persistence has leaked into business logic
}

// Good: a clean domain; persistence is the repository's responsibility
type Order struct {
    ID     OrderID
    Status OrderStatus
}

func (o *Order) Complete() error {
    if o.Status != StatusPending {
        return fmt.Errorf("%w: order %s is %s", ErrInvalidTransition, o.ID, o.Status)
    }
    o.Status = StatusCompleted
    return nil
}
```

### Use case = one type, one method

One use case — one type with an `Execute`/`Handle` method. The reasoning is the same as in SRP: each use case changes independently, with no risk of disturbing its neighbor.

```go
type CancelOrderUseCase struct {
    orders   OrderRepository
    payments PaymentGateway
    events   EventPublisher
}

func (uc *CancelOrderUseCase) Execute(ctx context.Context, cmd CancelOrderCommand) error {
    order, err := uc.orders.FindByID(ctx, cmd.OrderID)
    if err != nil {
        return fmt.Errorf("find order: %w", err)
    }
    if err := order.Cancel(cmd.Reason); err != nil {
        return fmt.Errorf("cancel order: %w", err)
    }
    if order.IsPaid() {
        if err := uc.payments.Refund(ctx, order.PaymentID(), order.Total()); err != nil {
            return fmt.Errorf("refund: %w", err)
        }
    }
    if err := uc.orders.Save(ctx, order); err != nil {
        return fmt.Errorf("save order: %w", err)
    }
    uc.events.Publish(ctx, OrderCancelled{OrderID: order.ID, Reason: cmd.Reason})
    return nil
}
```

---

## 5. Organizing code and modules

### Grouping packages: horizontal vs. vertical

```
# HORIZONTAL (scales worse as the project grows)
handlers/ services/ repositories/

# VERTICAL (isolates changes better)
internal/order/
    handler.go, service.go, repository.go, model.go
internal/user/
    handler.go, service.go, repository.go, model.go
```

Vertical grouping keeps every file of a single business area in one directory — a change in `order` isn't scattered across three horizontal layers. For a project with three or more domains, vertical grouping wins; for a single-domain microservice, horizontal is perfectly acceptable. The ground rules for `cmd/`, `internal/`, and `pkg/` are in §2 of the `go` skill.

### Minimizing the public surface

Fewer exported names = fewer commitments and fewer breaking changes down the line.

```go
// Bad: everything is exported; implementation details leak out
type OrderService struct {
    DB    *sql.DB
    Cache *redis.Client
}

func (s *OrderService) BuildQuery(f Filter) string { /* implementation detail */ }

// Good: a minimal public surface
type OrderService struct {
    db    *sql.DB
    cache *redis.Client
}

func (s *OrderService) GetByID(ctx context.Context, id int64) (*Order, error) { /* ... */ }
func (s *OrderService) buildQuery(f Filter) string { /* private */ }
```

### Breaking cyclic dependencies

The Go compiler won't allow a direct import cycle between packages — but the usual disguised form is `order` and `user` each knowing too much about the other through a shared type or interface, which effectively reproduces the cycle at the design level.

**The fix:** extract the shared abstraction, invert the dependency through an interface on the consumer side, or merge the packages if the boundary turned out to be artificial.

```go
// package order — does not depend on the concrete user package
type UserProvider interface {
    GetByID(ctx context.Context, id int64) (*User, error)
}

type OrderService struct {
    users UserProvider // satisfied by a real user.Service or a fake in tests
}
```

### When a module is ready to split off into a service

Signs of readiness: a separate team owns the module, a different deploy cycle, different scaling requirements, minimal shared data with the rest of the system, a clear API at the boundary.

**Don't split when:** the shared data model is large, calls are mostly synchronous, or shared transactions are needed — a distributed transaction between two freshly split services is almost always more expensive than whatever the split buys you.

---

## 6. Data flow and state

### Immutability

```go
// Bad: a mutable value object — one call changes state for every holder
type Money struct {
    Amount   int64
    Currency string
}

func (m *Money) Add(other Money) {
    m.Amount += other.Amount // an unexpected mutation for the caller
}

// Good: immutable — the method returns a new value
type Money struct {
    Amount   int64
    Currency string
}

func (m Money) Add(other Money) Money {
    return Money{Amount: m.Amount + other.Amount, Currency: m.Currency}
}
```

**Where immutability is critical:** value objects, DTOs, command objects, events, configuration after initialization.

### A clear owner of state

Every piece of state has exactly one owner; everyone else gets read-only access or a copy. Symptoms of a violation: a pointer object mutated from several independent places, a global singleton with mutable state, a map shared between goroutines without synchronization (§12 of the `go` skill has both this and the correct protection pattern via a mutex or channels).

### Event-driven vs. direct call

| Criterion | Direct call | Event |
|---|---|---|
| A response is needed | Yes | No |
| Consistency | Strong | Eventual |
| Coupling | Tight | Loose |
| Debugging difficulty | Lower | Higher |

**Rule:** the core business flow is a direct call. Side effects (notifications, analytics, audit) go through an event.

### CQRS and Event Sourcing

CQRS is justified when the read and write models diverge substantially, reads greatly outnumber writes, and each side needs to be optimized separately.

Event Sourcing is justified when you need a full audit log, "time travel" through state, or when the domain model itself is naturally described as a sequence of events.

**Apply neither** to plain CRUD without complex business logic — the infrastructure cost won't pay off.

### Cache invalidation

**Patterns:** write-through (write to cache and store at the same time), cache-aside (miss → read from store → write to cache), TTL-based (expiry by time with no explicit invalidation).

**On review:** verify that every write point invalidates the corresponding cache keys. Updating the data and forgetting to invalidate is one of the most common sources of production bugs of exactly this kind.

---

## 7. API contract

### Backward compatibility

**Breaking changes** (break clients): removing a field from a response, changing a field's type, renaming a field, removing an endpoint, adding a new required request parameter.

**Non-breaking:** a new optional request parameter, a new response field, a new endpoint, a new enum value.

```go
// Bad: both fields are breaking changes
type OrderResponse struct {
    Total MoneyDTO `json:"total"` // was float64 — the type changed
    State string   `json:"state"` // was "status" — a rename
}

// Good: a backward-compatible extension; the old fields remain
type OrderResponse struct {
    Total   float64   `json:"total"`
    TotalV2 *MoneyDTO `json:"total_v2,omitempty"` // new, optional field
    Status  string    `json:"status"`
    State   string    `json:"state"` // an added synonym; the old one stays
}
```

### Consistent error format

Check for: a single error format across the whole API, a machine-readable error code, no leakage of internal details (stack trace, SQL query, disk path), consistent HTTP statuses for equivalent situations.

```json
{
    "error": {
        "code": "ORDER_NOT_FOUND",
        "message": "order 123 not found",
        "details": [{"field": "order_id", "reason": "entity does not exist"}]
    }
}
```

### Pagination, filtering, sorting

Check for: an explicit limit on `per_page` (without it, a DoS vector), cursor-based pagination for large tables, sorting only on indexed fields, validation of filter/sort values (never allow arbitrary SQL through a `sort` parameter).

### Idempotency

An idempotent operation — a repeated call with the same parameters yields the same result and does not produce the side effect a second time.

```go
func (h *PaymentHandler) Charge(w http.ResponseWriter, r *http.Request) {
    key := r.Header.Get("Idempotency-Key")
    if key == "" {
        http.Error(w, "Idempotency-Key required", http.StatusBadRequest)
        return
    }
    if existing, err := h.payments.FindByIdempotencyKey(r.Context(), key); err == nil {
        json.NewEncoder(w).Encode(existing)
        return
    }

    var req ChargeRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid body", http.StatusBadRequest)
        return
    }

    payment, err := h.gateway.Charge(r.Context(), req.Amount, req.CardToken)
    if err != nil {
        writeError(w, err)
        return
    }
    payment.IdempotencyKey = key
    if err := h.payments.Save(r.Context(), payment); err != nil {
        writeError(w, err)
        return
    }
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(payment)
}
```

**Where idempotency is critical:** financial operations, any state change, webhook handlers, and POST/PUT/PATCH endpoints a client might retry.

---

## 8. Managing complexity

### Cyclomatic complexity

Every `if`, `for`, `case`, `&&`, `||` adds +1 to a function's complexity.

| Complexity | Rating | Action |
|---|---|---|
| 1–5 | Simple | OK |
| 6–10 | Moderate | Needs solid tests |
| 11–20 | High | Refactoring advisable |
| 21+ | Very high | Refactoring required |

### Deep nesting (>3 levels)

```go
// Bad: 5 levels of nesting
func ProcessOrders(orders []Order) {
    for _, o := range orders {
        if o.IsValid() {
            for _, item := range o.Items {
                if item.InStock() {
                    if o.User.HasPermission("buy") {
                        processItem(item)
                    }
                }
            }
        }
    }
}

// Good: guard clauses strip out the needless nesting
func ProcessOrders(orders []Order) {
    for _, o := range orders {
        if !o.IsValid() || !o.User.HasPermission("buy") {
            continue
        }
        for _, item := range o.Items {
            if item.IsProcessable() {
                processItem(item)
            }
        }
    }
}
```

### Long functions (>40 lines)

A long function signals an SRP violation at the function level, not just the type level. The fix: extract logical blocks into private methods with descriptive names; a comment above a block of code often points precisely to the extraction boundary.

```go
// Good: an orchestrator function instead of an 80-line monster
func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    if err := s.validate(req); err != nil {
        return nil, fmt.Errorf("validate: %w", err)
    }
    pricing, err := s.calculatePricing(ctx, req)
    if err != nil {
        return nil, fmt.Errorf("pricing: %w", err)
    }
    if err := s.checkInventory(ctx, req.Items); err != nil {
        return nil, fmt.Errorf("inventory: %w", err)
    }
    order, err := s.persist(ctx, req, pricing)
    if err != nil {
        return nil, fmt.Errorf("persist: %w", err)
    }
    s.notifyCreated(ctx, order)
    return order, nil
}
```

### "God files" (>500 lines)

Usually a sign of several mixed responsibilities in one file. Exceptions: test files, large blocks of constants/enums, generated code (protobuf, mocks).

### Early returns instead of deep branching

```go
// Bad: arrow code — a single return at the very bottom
func ProcessPayment(p Payment) Result {
    if p.IsValid() {
        if p.Amount > 0 {
            if p.HasValidCard() {
                charge, err := gateway.Charge(p)
                if err == nil {
                    return Result{Success: true, ChargeID: charge.ID}
                }
                return Result{Success: false, Error: err.Error()}
            }
            return Result{Success: false, Error: "invalid card"}
        }
        return Result{Success: false, Error: "invalid amount"}
    }
    return Result{Success: false, Error: "invalid payment"}
}

// Good: guard clauses — flat and readable
func ProcessPayment(p Payment) Result {
    if !p.IsValid() {
        return Result{Success: false, Error: "invalid payment"}
    }
    if p.Amount <= 0 {
        return Result{Success: false, Error: "invalid amount"}
    }
    if !p.HasValidCard() {
        return Result{Success: false, Error: "invalid card"}
    }
    charge, err := gateway.Charge(p)
    if err != nil {
        return Result{Success: false, Error: err.Error()}
    }
    return Result{Success: true, ChargeID: charge.ID}
}
```

---

## 9. Recognizing technical debt

### When duplication is actually worth fixing

**Not every matching snippet is a DRY violation.** Two fragments can look identical yet describe different business concepts that merely happen to look the same right now.

**The fix is justified when:** the same business concept is duplicated 3+ times; a rule change requires edits in 3+ places at once; it's an obvious copy-paste with minimal differences.

**The fix is NOT justified when:** the resemblance is coincidental (different domains that just share a shape); the duplication lives in tests (there, test isolation matters more than DRY); it's the language's own standard boilerplate.

```go
// Signal: the same business logic duplicated across two services
func (s *OrderService) calculateTotal(items []Item) float64 {
    total := 0.0
    for _, item := range items {
        total += item.Price * float64(item.Quantity)
        if item.Quantity > 10 {
            total *= 0.95 // 5% volume discount
        }
    }
    return total
}
// InvoiceService: a copy-paste of the same formula → extract a PricingCalculator
```

### TODO / FIXME / HACK

- A new `TODO` with no ticket reference — "has a ticket been created?" is a fair question to raise at review time.
- A `FIXME` that lives in production longer than a single sprint is no longer a FIXME but acknowledged technical debt; file a ticket instead of leaving a comment.
- A PR that adds a `TODO` to code it just wrote — worth asking why it wasn't simply done now.

### Inconsistent patterns across the codebase

The worst kind of debt is the same task solved three different ways in one project. On review, watch for new code with a different approach to error handling, a different handler structure, or different naming or dependency-injection conventions.

**Rule:** new code either follows the existing pattern or refactors ALL the places to the new approach. A mix is worse than either option on its own.

### Missing abstractions

Three or more places with identical code structure signal that it's time to extract an abstraction.

```go
// Signal: the identical log+metric+error-response structure in 5 handlers
result, err := svc.DoSomething(ctx, req)
if err != nil {
    log.ErrorContext(ctx, "service error", slog.Any("err", err))
    metrics.Inc("service_error", "handler", "orders")
    writeErrorResponse(w, err)
    return
}
// → extract into handleServiceCall(ctx, w, func() (T, error) { ... }) or middleware
```

### Dead code

Unused imports, unreachable branches, commented-out code (git remembers the history — just delete it), methods with no callers, feature flags nobody turns on anymore. Dead code is noise while reading and a false signal when searching the codebase.

---

## 10. Severity of architectural issues

When run alongside the review skill, these four tiers map onto its identical 🔴/🟡/🔵/ℹ️ scale: Critical → 🔴, Major → 🟡, Minor → 🔵, Info → ℹ️.

### Critical — blocks the PR

| Issue | Why | Example |
|---|---|---|
| Layer boundary violation | Breaks the architectural contract | A handler runs SQL directly |
| A design-level dependency cycle | The packages can't be untangled or tested separately | `order` and `user` pull too much from each other |
| Broken domain purity | Domain depends on infrastructure | A domain type with ORM tags |
| Breaking API change without versioning | Breaks existing clients | Changing a response field's type |
| Shared mutable state without synchronization | Data race, unpredictable behavior | A global map with no mutex |

### Major — strongly recommend fixing

| Issue | Why | Example |
|---|---|---|
| SOLID violation | Degrades maintainability | A type with 15 unrelated public methods |
| Tight coupling | Cascading changes on the smallest edit | `redis.NewClient(...)` right inside business logic |
| No abstraction at the boundary with an external system | An external API change forces edits in business code | The payment provider's SDK scattered across 10 files |
| High complexity (>15) | The function can't be fully covered by tests | 20 branches in one method |
| Inconsistent error handling | Unpredictable behavior for the API client | Some endpoints return JSON, others plain text |

### Minor — nice to have

| Issue | Why | Example |
|---|---|---|
| Inconsistent naming | Extra cognitive load | `GetUserByID` next to `fetchUser` |
| Minor cohesion issue | A type is a little blurry | Two unrelated methods on one struct |
| Works but inelegant | Not a blocker, but worth tidying | An `if/else` where a map is the obvious fit |
| Missing early return | Needless nesting | Two levels of `if` instead of a guard clause |
| Premature abstraction | An interface for a single implementation with no reason | A `Repository` interface with one concrete type and no test fakes |

### Info — cosmetic, no action required

| Issue | Why | Example |
|---|---|---|
| Alternative pattern as FYI | Worth knowing, not worth changing | A different valid way to structure the same package |
| Naming style inconsistency within a single file | Cosmetic only | Mixed receiver names for the same type across methods |
| Non-blocking architectural observation outside PR scope | Doesn't affect this change | "This package will need splitting eventually, not yet" |

---

## 11. Architectural review checklist

### Boundaries and layers
- [ ] Is the dependency rule honored (dependencies point inward)?
- [ ] Domain doesn't import infrastructure or transport?
- [ ] Transport contains no business logic?
- [ ] A clear boundary between application and domain?
- [ ] No dependency cycles?

### SOLID
- [ ] A single reason to change (SRP)?
- [ ] New behavior added by extension, not by editing existing code (OCP)?
- [ ] Implementations honor the interface contract (LSP)?
- [ ] Interfaces aren't "fat" (ISP)?
- [ ] Dependencies inverted at the boundaries through abstractions (DIP)?

### Coupling and cohesion
- [ ] No inline construction of dependencies in business logic?
- [ ] No god objects?
- [ ] No feature envy?
- [ ] External systems isolated behind an adapter?
- [ ] Packages have concrete names (not `util`/`helpers`)?

### API
- [ ] API changes are backward compatible, or explicitly versioned?
- [ ] The error format is consistent across the whole API?
- [ ] Mutating endpoints are idempotent?
- [ ] Pagination has explicit limits?

### Complexity
- [ ] No functions with complexity > 15?
- [ ] No nesting deeper than 3 levels?
- [ ] No functions > 40 lines without reason?
- [ ] No files > 500 lines (except tests/generated)?
- [ ] Guard clauses instead of deep branching?

### Technical debt
- [ ] No business-logic duplication in 3+ places?
- [ ] Every `TODO` has a ticket?
- [ ] New code follows the project's conventions?
- [ ] No dead code?

### Data and state
- [ ] Value objects are immutable?
- [ ] Is it clear who "owns" the mutable state?
- [ ] Side effects separated from the core logic?
- [ ] Cache invalidated at every write point?

### Design patterns
- [ ] Does each pattern solve a real, existing problem?
- [ ] Abstractions aren't premature (2+ implementations or a clearly stated need)?
- [ ] The level of indirection is justified by the problem's complexity?
