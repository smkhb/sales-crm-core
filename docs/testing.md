# Testing

The test suite is **unit-level, fast and dependency-free**. Every use case is tested in isolation against in-memory doubles — no database, no HTTP, no mocking framework.

## Running the tests

```bash
pnpm test        # single run
pnpm test:watch  # watch mode
```

Vitest is configured with `globals: true` and `vite-tsconfig-paths`, so the `@/` alias from `tsconfig.json` works inside tests.

## Strategy

- **One test file per use case, colocated with it** — e.g. `register-salesperson.test.ts` sits next to `register-salesperson.ts`. When the HTTP layer arrives, entities and handlers follow the same colocated pattern.
- **Assertions on `Either`**: success cases expect `value.isRight()` and the payload; failure cases expect `isLeft()` and `instanceof` the specific error.
- **Business rules are explicit test names**, e.g. *"should not allow a non-manager to register a salesperson"*, making the suite double as executable documentation of the business rules in the README.
- **Events are verified indirectly** through their handlers' side effects, which keeps tests honest about the dispatch flow.

## Test doubles (`tests/`)

| Double                  | File                              | Notes                                                          |
| ----------------------- | --------------------------------- | -------------------------------------------------------------- |
| In-memory repositories  | `tests/repos/in-memo-*.repo.ts`   | Implement the repository ports; persist aggregates in arrays    |
| `FakeHasher`            | `tests/encryptography/fake-hasher.ts` | `hash(plain) -> plain-hashed`; compare checks the suffix     |
| `FakeEncrypter`         | `tests/encryptography/fake-encrypter.ts` | Returns `token-{sub}-{role}`, deterministic and inspectable |
| Factories               | `tests/factories/make-*.ts`       | Valid aggregate builders with `@faker-js/faker` defaults        |

Factories accept a partial override and return a ready-to-use aggregate, so each test only declares what it cares about:

```ts
const salesperson = makeSalesperson({}, new UniqueEntityID("executor-01"));
const client = makeClient({ salesRepID: salesperson.id });
```

## Standard test skeleton

```ts
describe("Register Salesperson Use Case", () => {
  it("should allow a manager to register a new salesperson", async () => {
    const manager = makeSalesperson({ role: SalespersonRole.manager });
    const repo = new InMemorySalespersonsRepo([manager]);

    const response = await new RegisterSalespersonUseCase(repo, new FakeHasher())
      .execute({ executorID: manager.id.toString(), /* ... */ });

    expect(response.isRight()).toBe(true);
    expect(repo.items).toHaveLength(2);
  });

  it("should not allow a non-manager to register a salesperson", async () => {
    // arrange: salesperson (non-manager) executor
    // act:     execute the use case
    // assert:  response.isLeft() && response.value instanceof NotAllowedError
  });
});
```

## Conventions

- Tests import via the `@/` alias, never relative paths across layers.
- Repos are constructed with initial items where supported, making the *arrange* step obvious.
- No snapshot tests, no network, no clock — `Date` defaults are set by the aggregates themselves.

Current status: **18 test files, 70 passing tests** (plus intentionally skipped WIP cases).
