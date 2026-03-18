---
applyTo: "**/*.ts,**/*.tsx,**/*.js,**/*.jsx,**/*.mts,**/*.cts"
---

# TypeScript / JavaScript Coding Standards

Apply these standards to all TypeScript and JavaScript code.

---

## TypeScript Configuration

### `tsconfig.json` — Required Compiler Options

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true,
    "esModuleInterop": true,
    "skipLibCheck": false,
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "sourceMap": true
  }
}
```

- `strict: true` is non-negotiable — enables `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, etc.
- Never use `@ts-ignore` without a comment explaining why and a linked issue to fix it
- Prefer `@ts-expect-error` over `@ts-ignore` (it fails if the error disappears)

---

## Type System Standards

### No `any` Without Justification

```typescript
// ✅ Good — typed
function processEvent(event: MouseEvent): void { ... }
function parseConfig(raw: unknown): AppConfig { ... } // use 'unknown' for truly unknown types

// ❌ Bad — lazy typing
function processData(data: any): any { ... }

// ✅ If you must use any, document why
// eslint-disable-next-line @typescript-eslint/no-explicit-any
const legacyData = response.data as any; // third-party lib has no types; tracked in ISSUE-123
```

### Use Discriminated Unions for Result Types

```typescript
// ✅ Good — explicit success/failure shape
type Result<T, E = Error> = 
  | { success: true; data: T }
  | { success: false; error: E };

async function fetchUser(id: string): Promise<Result<User>> {
  try {
    const user = await db.users.findById(id);
    if (!user) return { success: false, error: new NotFoundError(id) };
    return { success: true, data: user };
  } catch (err) {
    return { success: false, error: err instanceof Error ? err : new Error(String(err)) };
  }
}

// ❌ Bad — returning null/undefined as error signal
async function fetchUser(id: string): Promise<User | null> { ... }
```

### Prefer Readonly for Immutable Objects

```typescript
// ✅ Good
interface Config {
  readonly apiUrl: string;
  readonly timeout: number;
}

function processItems(items: ReadonlyArray<Item>): void { ... }
```

---

## Naming Conventions

| Construct | Convention | Example |
|-----------|-----------|---------|
| Variables & functions | `camelCase` | `getUserById` |
| Classes & interfaces | `PascalCase` | `UserService`, `AuthToken` |
| Types & enums | `PascalCase` | `UserRole`, `ApiError` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRIES` |
| Files | `kebab-case` | `user-service.ts` |
| Test files | `*.test.ts` or `*.spec.ts` | `user-service.test.ts` |
| Private class fields | `#fieldName` (or `_` prefix if private not supported) | `#cache` |

---

## Functions & Classes

### Functions Should Be Small and Focused

```typescript
// ✅ Good — single responsibility
async function validateEmail(email: string): Promise<boolean> {
  return EMAIL_REGEX.test(email);
}

async function createUser(input: CreateUserInput): Promise<User> {
  await validateCreateUserInput(input); // delegates validation
  const user = buildUserEntity(input);  // delegates construction
  await db.users.insert(user);
  await sendWelcomeEmail(user);
  return user;
}

// ❌ Bad — does too many things in one function
async function createUser(email: string, password: string, firstName: string, ...) {
  // 80 lines of mixed validation, construction, persistence, and email sending
}
```

### Error Handling

```typescript
// ✅ Good — typed custom errors
class ValidationError extends Error {
  constructor(
    message: string,
    public readonly field: string,
    public readonly value: unknown
  ) {
    super(message);
    this.name = 'ValidationError';
  }
}

class NotFoundError extends Error {
  constructor(resource: string, id: string) {
    super(`${resource} with id '${id}' not found`);
    this.name = 'NotFoundError';
  }
}

// ✅ Good — catch-rethrow with context added
async function getDocument(id: string): Promise<Document> {
  try {
    return await db.documents.findById(id);
  } catch (err) {
    throw new DatabaseError(`Failed to fetch document ${id}`, { cause: err });
  }
}
```

---

## Module Organization

```
src/
├── index.ts              # Public exports only
├── config/               # Configuration loading and validation
├── types/                # Shared type definitions and interfaces
├── errors/               # Custom error classes
├── middleware/            # Express/framework middleware
├── controllers/           # Request handlers (thin — delegate to services)
├── services/              # Business logic
├── repositories/          # Data access layer
├── utils/                 # Pure utility functions
└── __tests__/             # Test files mirror src/ structure
```

---

## Security-Specific TypeScript Patterns

### Input Validation with Zod

```typescript
import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email().toLowerCase().max(255),
  password: z.string().min(12).max(128),
  firstName: z.string().min(1).max(100).trim(),
  lastName: z.string().min(1).max(100).trim(),
});

type CreateUserInput = z.infer<typeof CreateUserSchema>;

// In controller/route handler
app.post('/api/users', async (req, res) => {
  const result = CreateUserSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(400).json({ error: 'Invalid input', details: result.error.flatten() });
  }
  const user = await userService.create(result.data); // result.data is fully typed
  res.status(201).json(user);
});
```

### Environment Configuration Validation

```typescript
import { z } from 'zod';

const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'staging', 'production']),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  PORT: z.coerce.number().default(3000),
});

// Validate at startup — fail fast
const env = EnvSchema.parse(process.env);
export default env;
```

---

## ESLint Configuration

```json
// .eslintrc.json — required rules
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/strict-type-checked",
    "plugin:security/recommended"
  ],
  "rules": {
    "no-eval": "error",
    "no-implied-eval": "error",
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-floating-promises": "error",
    "@typescript-eslint/no-unsafe-assignment": "error",
    "@typescript-eslint/explicit-function-return-type": "warn",
    "security/detect-object-injection": "warn",
    "security/detect-non-literal-regexp": "warn"
  }
}
```

---

## Documentation (JSDoc)

```typescript
/**
 * Retrieves a user by their unique identifier.
 *
 * @param id - The UUID of the user to retrieve
 * @returns The matching user
 * @throws {NotFoundError} if no user with the given ID exists
 * @throws {DatabaseError} if the database query fails
 *
 * @example
 * const user = await getUserById('550e8400-e29b-41d4-a716-446655440000');
 */
async function getUserById(id: string): Promise<User> { ... }
```

Every exported function, class, and interface must have a JSDoc comment with:
- Description of purpose
- Parameter descriptions (if not obvious from types)
- Return value description
- Thrown exceptions
