---
name: solid-principles
description: Principes SOLID. Use when reviewing code quality or refactoring.
---

# Principes SOLID

The SOLID principles are **mandatory** for all code governed by this documentation.
They ensure code that is maintainable, testable, and scalable over time.

## SRP – Single Responsibility Principle

### Definition

**A class should have only one reason to change.**

Each class, method, or module must have a single, clearly defined responsibility.

### Violation Indicators

- Class names containing “and” or “or”
- Methods performing multiple unrelated actions
- Classes that are difficult to name precisely
- Tests requiring extensive mocking to isolate behavior

### Example

**❌ Violation:**

```typescript
class UserService {
  // Multiple responsibilities: validation, persistence, and email sending
  async createUser(userData: UserData) {
    // Validation logic
    if (!userData.email.includes("@")) {
      throw new Error("Invalid email");
    }

    // Database persistence
    const user = await db.users.insert(userData);

    // Email notification
    await emailClient.send({
      to: user.email,
      subject: "Welcome!",
      body: `Hello ${user.name}...`,
    });

    return user;
  }
}
```

**✅ Correct:**

```typescript
class UserValidator {
  validate(userData: UserData): ValidationResult {
    if (!userData.email.includes("@")) {
      return { valid: false, error: "Invalid email" };
    }
    return { valid: true };
  }
}

class UserRepository {
  async save(userData: UserData): Promise<User> {
    return await db.users.insert(userData);
  }
}

class WelcomeEmailSender {
  async send(user: User): Promise<void> {
    await emailClient.send({
      to: user.email,
      subject: "Welcome!",
      body: `Hello ${user.name}...`,
    });
  }
}

class UserService {
  constructor(
    private validator: UserValidator,
    private repository: UserRepository,
    private emailSender: WelcomeEmailSender
  ) {}

  async createUser(userData: UserData): Promise<User> {
    const validation = this.validator.validate(userData);
    if (!validation.valid) throw new Error(validation.error);

    const user = await this.repository.save(userData);
    await this.emailSender.send(user);

    return user;
  }
}
```

### Benefits

- **Testability:** Each unit can be tested in isolation
- **Maintainability:** Changes are localized
- **Reusability:** Components are independent
- **Readability:** Responsibilities are explicit

---

## OCP – Open/Closed Principle

### Definition

**Software entities should be open for extension but closed for modification.**

New behavior should be added without altering existing code.

### Violation Indicators

- Conditional logic based on type or mode
- Frequent edits to the same class to add features
- Feature additions requiring modification of stable code

### Example

**❌ Violation:**

```typescript
class PaymentProcessor {
  processPayment(amount: number, method: string) {
    if (method === "credit_card") {
      // Credit card logic
      this.chargeCreditCard(amount);
    } else if (method === "paypal") {
      // PayPal logic
      this.chargePayPal(amount);
    } else if (method === "crypto") {
      // Crypto logic (requires modifying this class!)
      this.chargeCrypto(amount);
    }
  }
}
```

**✅ Correct:**

```typescript
interface PaymentMethod {
  charge(amount: number): Promise<PaymentResult>;
}

class CreditCardPayment implements PaymentMethod {
  async charge(amount: number): Promise<PaymentResult> {
    // Credit card specific logic
    return { success: true, transactionId: "..." };
  }
}

class PayPalPayment implements PaymentMethod {
  async charge(amount: number): Promise<PaymentResult> {
    // PayPal specific logic
    return { success: true, transactionId: "..." };
  }
}

// New payment method without modifying existing code
class CryptoPayment implements PaymentMethod {
  async charge(amount: number): Promise<PaymentResult> {
    // Crypto specific logic
    return { success: true, transactionId: "..." };
  }
}

class PaymentProcessor {
  constructor(private paymentMethod: PaymentMethod) {}

  async processPayment(amount: number): Promise<PaymentResult> {
    return await this.paymentMethod.charge(amount);
  }
}
```

### Benefits

- **Safe extension:** New features via new classes
- **Stability:** Existing code remains untouched
- **Regression safety:** No unintended side effects
- **Scalability:** Behavior grows without risk

---

## LSP – Liskov Substitution Principle

### Definition

**Objects of a derived type must be substitutable for objects of the base type
without altering the correctness of the program.**

Subtypes must honor the contracts of their base types.

### Violation Indicators

- Subclasses throwing undocumented exceptions
- Runtime type checks to change behavior
- Overridden methods that break expectations
- Stronger preconditions or weaker postconditions

### Rules

1. Preconditions must not be strengthened
2. Postconditions must not be weakened
3. Invariants must be preserved
4. Historical behavior must remain compatible

### Example

**❌ Violation:**

```typescript
class Rectangle {
  constructor(protected width: number, protected height: number) {}

  setWidth(width: number) {
    this.width = width;
  }
  setHeight(height: number) {
    this.height = height;
  }
  getArea(): number {
    return this.width * this.height;
  }
}

class Square extends Rectangle {
  // Violates LSP: changes behavior unexpectedly
  setWidth(width: number) {
    this.width = width;
    this.height = width; // Side effect!
  }
  setHeight(height: number) {
    this.width = height;
    this.height = height; // Side effect!
  }
}

// This breaks when using Square instead of Rectangle
function expandRectangle(rect: Rectangle) {
  rect.setWidth(5);
  rect.setHeight(4);
  // Expected area: 20
  // Actual area with Square: 16 (broken!)
  return rect.getArea();
}
```

**✅ Correct:**

```typescript
interface Shape {
  getArea(): number;
}

class Rectangle implements Shape {
  constructor(private width: number, private height: number) {}

  setWidth(width: number) {
    this.width = width;
  }
  setHeight(height: number) {
    this.height = height;
  }
  getArea(): number {
    return this.width * this.height;
  }
}

class Square implements Shape {
  constructor(private side: number) {}

  setSide(side: number) {
    this.side = side;
  }
  getArea(): number {
    return this.side * this.side;
  }
}

// Now each shape behaves correctly according to its contract
function calculateArea(shape: Shape): number {
  return shape.getArea();
}
```

### Benefits

- **Safe polymorphism**
- **Clear contracts**
- **Predictable behavior**
- **Reliable test doubles**

---

## ISP – Interface Segregation Principle

### Definition

**Clients should not be forced to depend on interfaces they do not use.**

Prefer multiple focused interfaces over a single general-purpose one.

### Violation Indicators

- Interfaces with many unrelated methods
- Empty or unsupported method implementations
- Clients depending on unused functionality

### Example

**❌ Violation:**

```typescript
// Fat interface forcing all implementations to support all operations
interface UserRepository {
  // Read operations
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  findAll(): Promise<User[]>;

  // Write operations
  create(userData: UserData): Promise<User>;
  update(id: string, userData: Partial<UserData>): Promise<User>;
  delete(id: string): Promise<void>;

  // Analytics operations
  getUserStats(): Promise<UserStats>;
  getActiveUsersCount(): Promise<number>;
  generateReport(): Promise<Report>;
}

// Read-only service forced to implement write methods
class UserDisplayService {
  constructor(private userRepo: UserRepository) {}

  async displayUser(id: string) {
    const user = await this.userRepo.findById(id);
    // Only needs read access, but depends on entire interface
  }

  // Forced to have access to dangerous operations
  // userRepo.delete() is available but shouldn't be!
}

// Analytics service depends on CRUD methods it doesn't need
class AnalyticsService {
  constructor(private userRepo: UserRepository) {}

  async getDashboardData() {
    return await this.userRepo.getUserStats();
    // Depends on create/update/delete methods it never uses
  }
}
```

**✅ Correct:**

```typescript
// Segregated interfaces - each client depends only on what it needs
interface UserReader {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  findAll(): Promise<User[]>;
}

interface UserWriter {
  create(userData: UserData): Promise<User>;
  update(id: string, userData: Partial<UserData>): Promise<User>;
  delete(id: string): Promise<void>;
}

interface UserAnalytics {
  getUserStats(): Promise<UserStats>;
  getActiveUsersCount(): Promise<number>;
  generateReport(): Promise<Report>;
}

// Full repository implements all interfaces
class UserRepository implements UserReader, UserWriter, UserAnalytics {
  async findById(id: string): Promise<User | null> {
    /* ... */
  }
  async findByEmail(email: string): Promise<User | null> {
    /* ... */
  }
  async findAll(): Promise<User[]> {
    /* ... */
  }
  async create(userData: UserData): Promise<User> {
    /* ... */
  }
  async update(id: string, userData: Partial<UserData>): Promise<User> {
    /* ... */
  }
  async delete(id: string): Promise<void> {
    /* ... */
  }
  async getUserStats(): Promise<UserStats> {
    /* ... */
  }
  async getActiveUsersCount(): Promise<number> {
    /* ... */
  }
  async generateReport(): Promise<Report> {
    /* ... */
  }
}

// Services depend only on what they need
class UserDisplayService {
  constructor(private userReader: UserReader) {}

  async displayUser(id: string) {
    const user = await this.userReader.findById(id);
    // Only has read access - safer and clearer intent
  }
}

class UserManagementService {
  constructor(private userReader: UserReader, private userWriter: UserWriter) {}

  async updateUserEmail(id: string, email: string) {
    const user = await this.userReader.findById(id);
    if (!user) throw new Error("User not found");
    return await this.userWriter.update(id, { email });
  }
}

class AnalyticsService {
  constructor(private analytics: UserAnalytics) {}

  async getDashboardData() {
    return await this.analytics.getUserStats();
    // Clean dependency - only analytics methods
  }
}
```

### Benefits

- **Loose coupling**
- **Simpler implementations**
- **Cleaner mocks**
- **Safer evolution**

---

## DIP – Dependency Inversion Principle

### Definition

**High-level modules must not depend on low-level modules.
Both must depend on abstractions.**

**Abstractions must not depend on details.
Details must depend on abstractions.**

### Violation Indicators

- Direct instantiation of concrete classes
- Infrastructure imports in business logic
- Strong coupling to frameworks
- Tests requiring real external systems

### Example

**❌ Violation:**

```typescript
import { MySQLDatabase } from "./mysql-database";
import { SendGridEmailService } from "./sendgrid-email-service";

class OrderService {
  // High-level module depends on low-level concrete implementations
  private database = new MySQLDatabase();
  private emailService = new SendGridEmailService();

  async createOrder(orderData: OrderData) {
    const order = await this.database.saveOrder(orderData);
    await this.emailService.sendConfirmation(order);
    return order;
  }
}

// Impossible to test without real MySQL and SendGrid!
```

**✅ Correct:**

```typescript
// Domain-owned abstractions
interface OrderRepository {
  save(orderData: OrderData): Promise<Order>;
}

interface EmailNotifier {
  sendOrderConfirmation(order: Order): Promise<void>;
}

// High-level module depends only on abstractions
class OrderService {
  constructor(
    private orderRepository: OrderRepository,
    private emailNotifier: EmailNotifier
  ) {}

  async createOrder(orderData: OrderData): Promise<Order> {
    const order = await this.orderRepository.save(orderData);
    await this.emailNotifier.sendOrderConfirmation(order);
    return order;
  }
}

// Infrastructure implementations depend on domain abstractions
class MySQLOrderRepository implements OrderRepository {
  async save(orderData: OrderData): Promise<Order> {
    // MySQL-specific implementation
    return await db.orders.insert(orderData);
  }
}

class SendGridEmailNotifier implements EmailNotifier {
  async sendOrderConfirmation(order: Order): Promise<void> {
    // SendGrid-specific implementation
    await sendGrid.send({...});
  }
}

// Easy to test with mocks
class InMemoryOrderRepository implements OrderRepository {
  private orders: Order[] = [];

  async save(orderData: OrderData): Promise<Order> {
    const order = { id: this.orders.length + 1, ...orderData };
    this.orders.push(order);
    return order;
  }
}
```

### Benefits

- **High testability**
- **Implementation flexibility**
- **Isolation of business logic**
- **Reusable abstractions**

---

## Validation Checklist

### Before each commit

#### SRP

- One clear responsibility per class
- Methods do one thing (< ~30 lines)
- No ambiguous naming

#### OCP

- New behavior added via extension
- No conditional logic based on types

#### LSP

- Subtypes respect base contracts
- No undocumented behavioral changes

#### ISP

- Interfaces are small and focused
- Clients depend only on what they use

#### DIP

- Use cases depend on interfaces
- Interfaces belong to the domain
- Dependencies injected via constructors
