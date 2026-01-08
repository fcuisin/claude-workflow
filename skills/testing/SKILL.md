# Testing - TDD Principles

## Overview

**Test-Driven Development (TDD)** is a **mandatory** practice to ensure code quality and maintainability.

**Objectives:**

- Code coverage ≥ 80%
- Fast tests (< 10s for unit tests)
- Independent and reproducible tests

## Test Pyramid

- E2E: 10% (slow)
- Integration tests: 20% (to verify external connections)
- Unit tests: 70% (fast, isolated)

## TDD - Test-Driven Development

### The Red-Green-Refactor Cycle

1. **RED** - Write a failing test

   - Define expected behavior
   - Test MUST fail (otherwise it tests nothing)

2. **GREEN** - Write minimum code to pass

   - Simplest possible code
   - No optimization
   - No generalization

3. **REFACTOR** - Improve the code
   - Remove duplication
   - Improve readability
   - Tests must still pass

### TDD Example

```
// 1. RED - Failing test
test "calculateSubtotal returns sum of line item amounts":
  invoice = new Invoice()
  invoice.addLine(LineItem(amount: 100))
  invoice.addLine(LineItem(amount: 250))

  assert invoice.calculateSubtotal() == 350
  // ❌ FAIL: method calculateSubtotal() not defined

// 2. GREEN - Minimal code
class Invoice:
  lines = []

  addLine(line):
    lines.add(line)

  calculateSubtotal():
    return lines.sum(line => line.amount)
  // ✅ PASS

// 3. REFACTOR - Improve
class Invoice:
  lines: List<LineItem> = []

  addLine(line: LineItem): void
    lines.add(line)

  calculateSubtotal(): Decimal
    return Decimal.sum(lines.map(l => l.amount))
  // ✅ PASS (improved with types)
```

### TDD Rules

1. **One test at a time**
2. **Test defines behavior** (not implementation)
3. **Minimal code to pass**
4. **Refactor after each GREEN**
5. **Never ignore a failing test**

## Test Types

### Unit Tests

**Goal:** Test a code unit in isolation

```
test "Discount can be applied to price":
  price = Price(100, "USD")
  discount = Discount(10, PERCENT)

  result = price.applyDiscount(discount)

  assert result.amount == 90
  assert result.currency == "USD"
```

**Characteristics:**

- Fast (< 1s)
- Isolated (no external dependencies)
- Deterministic (same result every time)
- Independent (execution order doesn't matter)

### Integration Tests

**Goal:** Test interaction between components

```
test "OrderRepository saves and retrieves order":
  repo = OrderRepository(database)
  order = Order(customerId: "C123", total: 99.99)

  repo.save(order)
  retrieved = repo.findById(order.id)

  assert retrieved.customerId == "C123"
  assert retrieved.total == 99.99
```

**Characteristics:**

- Test connections (DB, API, files)
- Use real dependencies or testcontainers
- Slower than unit tests

### End-to-End (E2E) Tests

**Goal:** Test complete system from user perspective

```
test "Customer can checkout and pay for order":
  browser.goto("/catalog")
  browser.click("#add-item-123")
  browser.click("#cart")
  browser.click("#proceed-checkout")
  browser.fill("#card-number", "4242424242424242")
  browser.click("#pay-now")

  assert browser.text("#order-status") contains "Payment successful"
```

**Characteristics:**

- Test complete user journey
- Slow and brittle
- Use sparingly

### Contract Tests

**Goal:** Verify contracts between services

```
test "API returns valid order schema":
  response = api.get("/orders/ORD-123")

  assert response.status == 200
  assert response.body matches OrderSchema
```

---

## Best Practices

### 1. Explicit Naming

```
// ❌ Vague names
test "test1": ...
test "product test": ...
test "it works": ...

// ✅ Descriptive names
test "calculateSubtotal returns zero for empty invoice": ...
test "checkout fails when payment is declined": ...
test "confirmation email is sent after successful payment": ...
```

### 2. Use Fixtures/Factories

```
// ❌ Repeated manual creation
test "test 1":
  product = Product(
    name: "Laptop",
    sku: "LAP-001",
    price: 999.99,
    category: "electronics",
    stock: 50,
    // ... 10 more fields
  )

// ✅ Factory
test "test 1":
  product = ProductFactory.create(category: "electronics")
```

### 3. Arrange-Act-Assert (AAA)

```
test "product price can be updated":
  // Arrange - Setup
  product = Product(price: 99.99)

  // Act - Execute
  product.updatePrice(149.99)

  // Assert - Verify
  assert product.price == 149.99
```

---

## Anti-patterns

### 1. Tests that test implementation

```
// ❌ Tests HOW (implementation)
test "placeOrder calls repository.insert":
  mock = mock(OrderRepository)
  service.placeOrder(order)
  verify mock.insert was called once

// ✅ Tests WHAT (behavior)
test "order is persisted":
  service.placeOrder(order)
  assert repository.findById(order.id) exists
```

### 2. Overly coupled tests

```
// ❌ Test knows too many internal details
test "fulfill order":
  order.fulfill()
  assert order._internalStatus == "fulfilled"
  assert order._fulfilledAt != null
  assert order._warehouseId == 456

// ✅ Test via public interface
test "fulfill order":
  order.fulfill()
  assert order.isFulfilled()
```

### 3. Tests without assertions

```
// ❌ Tests nothing
test "create product":
  service.createProduct(data)
  // No assert!

// ✅ Verify the result
test "create product":
  product = service.createProduct(data)
  assert product.id != null
  assert product.sku == data.sku
```

---

## Checklist

### Before each commit

- [ ] All tests pass
- [ ] Coverage ≥ 80%
- [ ] Explicit test names

### For each new feature

- [ ] Unit tests for business logic
- [ ] Integration tests for external connections

### For each bug fix

- [ ] Test that reproduces the bug (fails before fix)
- [ ] Fix implemented
- [ ] Test passes after fix
- [ ] Regression test added
