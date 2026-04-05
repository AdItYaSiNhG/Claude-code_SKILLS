---
name: design-an-interface
description: >
  Design a TypeScript interface, API contract, or module public API through
  structured thinking: use cases first, then shape, then constraints. Produces
  a clean, minimal interface before any implementation is written.
  Triggers: "design an interface", "design this API", "what should the interface
  look like", "design the public API", "API contract", "design the types",
  "interface design", "design before implementation".
source: adapted from mattpocock/skills
version: "1.0.0"
---

# Design an Interface Skill

## Why interface-first

Writing implementation before interface produces interfaces that leak
implementation details. Starting from the caller's perspective produces
interfaces that are a pleasure to use — minimal, composable, unsurprising.

---

## Step 1 — Understand the use cases

Before designing anything, list every way the caller will use this module.
Concrete usage examples are more valuable than abstract requirements.

```typescript
// Write usage examples BEFORE writing the interface
// These become your acceptance criteria and your documentation

// Use case 1: basic fetch
const user = await userService.getById("123");

// Use case 2: create with validation
const newUser = await userService.create({ name: "Alice", email: "a@b.com" });

// Use case 3: batch fetch (don't make N calls)
const users = await userService.getMany(["1", "2", "3"]);

// Use case 4: error handling
try {
  await userService.getById("unknown");
} catch (err) {
  if (err instanceof NotFoundError) { ... }
}
```

---

## Step 2 — Design the minimal interface

Start with the absolute minimum that satisfies the use cases. Every addition
has a cost — complexity for the implementer and cognitive load for the caller.

```typescript
// First draft — minimal
interface UserService {
  getById(id: string): Promise<User>;
  getMany(ids: string[]): Promise<User[]>;
  create(input: CreateUserInput): Promise<User>;
}

// Ask for each method:
// - Is this caller need or implementer convenience?
// - Could two methods be unified?
// - Does this expose implementation details?
```

**Interface design principles:**

| Principle | Good | Bad |
|-----------|------|-----|
| Deep module | `cache.get(key)` | `cache.checkExpiry(); cache.fetchFromStore(); cache.deserialise()` |
| Stable abstraction | `repo.findByEmail(email)` | `repo.query("SELECT * WHERE email = ?", email)` |
| No leaking | `service.createUser(input)` | `service.insertUserRow(dbConn, input)` |
| Minimal surface | 3 methods, used always | 10 methods, 7 used rarely |

---

## Step 3 — Design the types

```typescript
// Input types — only what's required, strict validation
interface CreateUserInput {
  name:  string;   // required
  email: string;   // required — will be validated
  role?: UserRole; // optional — defaults to "user"
  // NOT: id (generated), createdAt (set by system), passwordHash (internal)
}

// Output types — what the caller actually needs
interface User {
  id:        string;
  name:      string;
  email:     string;
  role:      UserRole;
  createdAt: Date;
  // NOT: passwordHash, internalFlags, dbRowVersion
}

// Enum/union types — exhaustive, named
type UserRole = "user" | "admin" | "viewer";

// Error types — typed errors for recoverable cases
class NotFoundError extends Error {
  constructor(public readonly resource: string, public readonly id: string) {
    super(`${resource} not found: ${id}`);
    this.name = "NotFoundError";
  }
}
```

---

## Step 4 — Review against constraints

```
[ ] Can every use case from Step 1 be expressed through this interface?
[ ] Does any method expose implementation details (SQL, HTTP, cache)?
[ ] Are any two methods really the same method with a different name?
[ ] Can the interface be implemented in-memory for testing? (if not, it's leaking)
[ ] Are error cases typed and documented?
[ ] Are all required params required, and all optional params optional?
[ ] Will this interface still make sense in 6 months if the implementation changes?
```

---

## Step 5 — Write the interface file

```typescript
// src/domains/users/IUserService.ts

export interface CreateUserInput {
  name:  string;
  email: string;
  role?: UserRole;
}

export interface User {
  id:        string;
  name:      string;
  email:     string;
  role:      UserRole;
  createdAt: Date;
}

export type UserRole = "user" | "admin" | "viewer";

/**
 * User management operations.
 * All methods throw NotFoundError if the resource does not exist.
 * All methods throw ValidationError if input fails validation.
 */
export interface IUserService {
  /** Fetch a single user. Throws NotFoundError if not found. */
  getById(id: string): Promise<User>;

  /** Fetch multiple users. Missing IDs are silently omitted. */
  getMany(ids: string[]): Promise<User[]>;

  /** Create a new user. Throws ConflictError if email already exists. */
  create(input: CreateUserInput): Promise<User>;
}
```

---

## Step 6 — Write a stub implementation for testing

```typescript
// src/domains/users/InMemoryUserService.ts
// Implements the interface — proves it's testable without a real DB

export class InMemoryUserService implements IUserService {
  private users = new Map<string, User>();

  async getById(id: string): Promise<User> {
    const user = this.users.get(id);
    if (!user) throw new NotFoundError("User", id);
    return user;
  }

  async getMany(ids: string[]): Promise<User[]> {
    return ids.map(id => this.users.get(id)).filter((u): u is User => !!u);
  }

  async create(input: CreateUserInput): Promise<User> {
    const user: User = { id: crypto.randomUUID(), role: "user", createdAt: new Date(), ...input };
    this.users.set(user.id, user);
    return user;
  }
}
```

If the stub is hard to implement, the interface is leaking too much.
