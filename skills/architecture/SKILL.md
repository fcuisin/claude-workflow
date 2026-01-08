# Back-end Architecture Patterns

Master proven backend architecture patterns including Clean Architecture, Hexagonal Architecture, and Domain-Driven Design to build maintainable, testable, and scalable systems.

## When to Use

- Designing new backend systems from scratch
- Refactoring monolithic applications for better maintainability
- Establishing architecture standards for your team
- Migrating from tightly coupled to loosely coupled architectures
- Implementing domain-driven design principles
- Creating testable and mockable codebases
- Planning microservices decomposition

## Core Concepts

### 1. Clean Architecture (Uncle Bob)

**Layers (dependency flows inward):**

- **Entities**: Core business models
- **Use Cases**: Application business rules
- **Interface Adapters**: Controllers, presenters, gateways
- **Frameworks & Drivers**: UI, database, external services

**Key Principles:**

- Dependencies point inward
- Inner layers know nothing about outer layers
- Business logic independent of frameworks
- Testable without UI, database, or external services

### 2. Hexagonal Architecture (Ports and Adapters)

**Components:**

- **Domain Core**: Business logic
- **Ports**: Interfaces defining interactions
- **Adapters**: Implementations of ports (database, REST, message queue)

**Benefits:**

- Swap implementations easily (mock for testing)
- Technology-agnostic core
- Clear separation of concerns

### 3. Domain-Driven Design (DDD)

**Strategic Patterns:**

- **Bounded Contexts**: Separate models for different domains
- **Context Mapping**: How contexts relate
- **Ubiquitous Language**: Shared terminology

**Tactical Patterns:**

- **Entities**: Objects with identity
- **Value Objects**: Immutable objects defined by attributes
- **Aggregates**: Consistency boundaries
- **Repositories**: Data access abstraction
- **Domain Events**: Things that happened

## Clean Architecture Pattern

### Directory Structure

```
src/
├── domain/              # Entities & business rules
│   ├── entities/
│   │   ├── user.ts
│   │   └── order.ts
│   ├── value-objects/
│   │   ├── email.ts
│   │   └── money.ts
│   └── ports/          # Abstract interfaces
│       ├── user-repository.port.ts
│       └── payment-gateway.port.ts
├── application/        # Application business rules
│   ├── create-user.use-case.ts
│   ├── process-order.use-case.ts
│   └── send-notification.use-case.ts
├── adapters/           # Interface implementations
│   ├── repositories/
│   │   ├── postgres-user.repository.ts
│   │   └── redis-cache.repository.ts
│   ├── controllers/
│   │   └── user.controller.ts
│   └── gateways/
│       ├── stripe-payment.gateway.ts
│       └── sendgrid-email.gateway.ts
└── infrastructure/     # Framework & external concerns
    ├── database.ts
    ├── config.ts
    └── logger.ts
```

### Implementation Example

```typescript
// domain/entities/user.ts
export class User {
  /** Core user entity - no framework dependencies */
  constructor(
    public readonly id: string,
    public email: string,
    public name: string,
    public readonly createdAt: Date,
    private _isActive: boolean = true
  ) {}

  get isActive(): boolean {
    return this._isActive;
  }

  deactivate(): void {
    /** Business rule: deactivating user */
    this._isActive = false;
  }

  canPlaceOrder(): boolean {
    /** Business rule: active users can order */
    return this._isActive;
  }
}

// domain/ports/user-repository.port.ts
import { User } from "../entities/user";

export interface IUserRepository {
  /** Port: defines contract, no implementation */
  findById(userId: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<User>;
  delete(userId: string): Promise<boolean>;
}

// use-cases/create-user.use-case.ts
import { User } from "../domain/entities/user";
import { IUserRepository } from "../domain/ports/user-repository.port";
import { randomUUID } from "crypto";

export interface CreateUserRequest {
  email: string;
  name: string;
}

export interface CreateUserResponse {
  user: User | null;
  success: boolean;
  error?: string;
}

export class CreateUserUseCase {
  /** Use case: orchestrates business logic */
  constructor(private readonly userRepository: IUserRepository) {}

  async execute(request: CreateUserRequest): Promise<CreateUserResponse> {
    // Business validation
    const existing = await this.userRepository.findByEmail(request.email);
    if (existing) {
      return {
        user: null,
        success: false,
        error: "Email already exists",
      };
    }

    // Create entity
    const user = new User(
      randomUUID(),
      request.email,
      request.name,
      new Date(),
      true
    );

    // Persist
    const savedUser = await this.userRepository.save(user);

    return {
      user: savedUser,
      success: true,
    };
  }
}

// adapters/repositories/postgres-user.repository.ts
import { IUserRepository } from "../../domain/ports/user-repository.port";
import { User } from "../../domain/entities/user";
import { Pool } from "pg";

export class PostgresUserRepository implements IUserRepository {
  /** Adapter: PostgreSQL implementation */
  constructor(private readonly pool: Pool) {}

  async findById(userId: string): Promise<User | null> {
    const client = await this.pool.connect();
    try {
      const result = await client.query("SELECT * FROM users WHERE id = $1", [
        userId,
      ]);
      return result.rows[0] ? this.toEntity(result.rows[0]) : null;
    } finally {
      client.release();
    }
  }

  async findByEmail(email: string): Promise<User | null> {
    const client = await this.pool.connect();
    try {
      const result = await client.query(
        "SELECT * FROM users WHERE email = $1",
        [email]
      );
      return result.rows[0] ? this.toEntity(result.rows[0]) : null;
    } finally {
      client.release();
    }
  }

  async save(user: User): Promise<User> {
    const client = await this.pool.connect();
    try {
      await client.query(
        `INSERT INTO users (id, email, name, created_at, is_active)
         VALUES ($1, $2, $3, $4, $5)
         ON CONFLICT (id) DO UPDATE
         SET email = $2, name = $3, is_active = $5`,
        [user.id, user.email, user.name, user.createdAt, user.isActive]
      );
      return user;
    } finally {
      client.release();
    }
  }

  async delete(userId: string): Promise<boolean> {
    const client = await this.pool.connect();
    try {
      const result = await client.query("DELETE FROM users WHERE id = $1", [
        userId,
      ]);
      return result.rowCount === 1;
    } finally {
      client.release();
    }
  }

  private toEntity(row: any): User {
    /** Map database row to entity */
    return new User(row.id, row.email, row.name, row.created_at, row.is_active);
  }
}

// adapters/controllers/user.controller.ts
import { Request, Response, Router } from "express";
import { CreateUserUseCase } from "../../use-cases/create-user.use-case";

export class UserController {
  router = Router();

  constructor(private readonly createUserUseCase: CreateUserUseCase) {
    this.setupRoutes();
  }

  private setupRoutes(): void {
    this.router.post("/users", this.createUser.bind(this));
  }

  /** Controller: handles HTTP concerns only */
  async createUser(req: Request, res: Response): Promise<void> {
    try {
      const { email, name } = req.body;

      const response = await this.createUserUseCase.execute({ email, name });

      if (!response.success) {
        res.status(400).json({ error: response.error });
        return;
      }

      res.status(201).json({ user: response.user });
    } catch (error) {
      res.status(500).json({ error: "Internal server error" });
    }
  }
}
```

## Hexagonal Architecture Pattern

```typescript
// Core domain (hexagon center)
import { Order } from "./entities/order";
import { OrderRepositoryPort } from "./ports/order-repository.port";
import {
  PaymentGatewayPort,
  PaymentResult,
} from "./ports/payment-gateway.port";
import { NotificationPort } from "./ports/notification.port";

export interface OrderResult {
  success: boolean;
  order?: Order;
  error?: string;
}

export class OrderService {
  /** Domain service - no infrastructure dependencies */
  constructor(
    private readonly orders: OrderRepositoryPort,
    private readonly payments: PaymentGatewayPort,
    private readonly notifications: NotificationPort
  ) {}

  async placeOrder(order: Order): Promise<OrderResult> {
    // Business logic
    if (!order.isValid()) {
      return { success: false, error: "Invalid order" };
    }

    // Use ports (interfaces)
    const payment = await this.payments.charge({
      amount: order.total,
      customerId: order.customerId,
    });

    if (!payment.success) {
      return { success: false, error: "Payment failed" };
    }

    order.markAsPaid();
    const savedOrder = await this.orders.save(order);

    await this.notifications.send({
      to: order.customerEmail,
      subject: "Order confirmed",
      body: `Order ${order.id} confirmed`,
    });

    return { success: true, order: savedOrder };
  }
}

// Ports (interfaces)
export interface OrderRepositoryPort {
  save(order: Order): Promise<Order>;
  findById(orderId: string): Promise<Order | null>;
}

export interface PaymentGatewayPort {
  charge(request: {
    amount: Money;
    customerId: string;
  }): Promise<PaymentResult>;
}

export interface NotificationPort {
  send(message: { to: string; subject: string; body: string }): Promise<void>;
}

// Adapters (implementations)
import Stripe from "stripe";

export class StripePaymentAdapter implements PaymentGatewayPort {
  /** Primary adapter: connects to Stripe API */
  private stripe: Stripe;

  constructor(apiKey: string) {
    this.stripe = new Stripe(apiKey, { apiVersion: "2024-12-18.acacia" });
  }

  async charge(request: {
    amount: Money;
    customerId: string;
  }): Promise<PaymentResult> {
    try {
      const charge = await this.stripe.charges.create({
        amount: request.amount.cents,
        currency: request.amount.currency,
        customer: request.customerId,
      });

      return {
        success: true,
        transactionId: charge.id,
      };
    } catch (error) {
      return {
        success: false,
        error: error instanceof Error ? error.message : "Payment failed",
      };
    }
  }
}

export class MockPaymentAdapter implements PaymentGatewayPort {
  /** Test adapter: no external dependencies */
  async charge(request: {
    amount: Money;
    customerId: string;
  }): Promise<PaymentResult> {
    return {
      success: true,
      transactionId: "mock-123",
    };
  }
}
```

## Domain-Driven Design Pattern

```typescript
// Value Objects (immutable)
export class Email {
  /** Value object: validated email */
  private constructor(public readonly value: string) {}

  static create(value: string): Email {
    if (!value.includes("@")) {
      throw new Error("Invalid email");
    }
    return new Email(value);
  }

  equals(other: Email): boolean {
    return this.value === other.value;
  }
}

export class Money {
  /** Value object: amount with currency */
  constructor(
    public readonly amount: number, // cents
    public readonly currency: string
  ) {}

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error("Currency mismatch");
    }
    return new Money(this.amount + other.amount, this.currency);
  }

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }

  get cents(): number {
    return this.amount;
  }
}

// Entities (with identity)
import { DomainEvent } from "./domain-event";

enum OrderStatus {
  PENDING = "PENDING",
  SUBMITTED = "SUBMITTED",
  PAID = "PAID",
  SHIPPED = "SHIPPED",
}

export class Order {
  /** Entity: has identity, mutable state */
  private _items: OrderItem[] = [];
  private _status: OrderStatus = OrderStatus.PENDING;
  private _events: DomainEvent[] = [];

  constructor(public readonly id: string, public readonly customer: Customer) {}

  get items(): readonly OrderItem[] {
    return this._items;
  }

  get status(): OrderStatus {
    return this._status;
  }

  get events(): readonly DomainEvent[] {
    return this._events;
  }

  addItem(product: Product, quantity: number): void {
    /** Business logic in entity */
    const item = new OrderItem(product, quantity);
    this._items.push(item);
    this._events.push(new ItemAddedEvent(this.id, item));
  }

  total(): Money {
    /** Calculated property */
    return this._items.reduce(
      (sum, item) => sum.add(item.subtotal()),
      new Money(0, "USD")
    );
  }

  submit(): void {
    /** State transition with business rules */
    if (this._items.length === 0) {
      throw new Error("Cannot submit empty order");
    }
    if (this._status !== OrderStatus.PENDING) {
      throw new Error("Order already submitted");
    }

    this._status = OrderStatus.SUBMITTED;
    this._events.push(new OrderSubmittedEvent(this.id));
  }

  clearEvents(): void {
    this._events = [];
  }
}

// Aggregates (consistency boundary)
export class Customer {
  /** Aggregate root: controls access to entities */
  private _addresses: Address[] = [];
  private _orderIds: string[] = []; // Order IDs, not full objects

  constructor(public readonly id: string, public readonly email: Email) {}

  get addresses(): readonly Address[] {
    return this._addresses;
  }

  addAddress(address: Address): void {
    /** Aggregate enforces invariants */
    if (this._addresses.length >= 5) {
      throw new Error("Maximum 5 addresses allowed");
    }
    this._addresses.push(address);
  }

  get primaryAddress(): Address | undefined {
    return this._addresses.find((a) => a.isPrimary);
  }
}

// Domain Events
export class OrderSubmittedEvent implements DomainEvent {
  public readonly occurredAt: Date;

  constructor(public readonly orderId: string, occurredAt?: Date) {
    this.occurredAt = occurredAt || new Date();
  }
}

export class ItemAddedEvent implements DomainEvent {
  public readonly occurredAt: Date;

  constructor(
    public readonly orderId: string,
    public readonly item: OrderItem,
    occurredAt?: Date
  ) {
    this.occurredAt = occurredAt || new Date();
  }
}

// Repository (aggregate persistence)
export class OrderRepository {
  /** Repository: persist/retrieve aggregates */

  async findById(orderId: string): Promise<Order | null> {
    /** Reconstitute aggregate from storage */
    // Implementation here
    return null;
  }

  async save(order: Order): Promise<void> {
    /** Persist aggregate and publish events */
    await this.persist(order);
    await this.publishEvents(order.events);
    order.clearEvents();
  }

  private async persist(order: Order): Promise<void> {
    // Persist to database
  }

  private async publishEvents(events: readonly DomainEvent[]): Promise<void> {
    // Publish domain events to event bus
  }
}
```

## Best Practices

1. **Dependency Rule**: Dependencies always point inward
2. **Interface Segregation**: Small, focused interfaces
3. **Business Logic in Domain**: Keep frameworks out of core
4. **Test Independence**: Core testable without infrastructure
5. **Bounded Contexts**: Clear domain boundaries
6. **Ubiquitous Language**: Consistent terminology
7. **Thin Controllers**: Delegate to use cases

## Common Pitfalls

- **Framework Coupling**: Business logic depends on frameworks
- **Fat Controllers**: Business logic in controllers
- **Missing Abstractions**: Concrete dependencies in core
- **Over-Engineering**: Clean architecture for simple CRUD
