---
name: xctest-to-testing
description: Migrates XCTest code to Swift Testing framework. Use this skill when converting existing XCTest tests to the newer Swift Testing framework.
metadata:
  short-description: Migrate XCTest tests to Swift Testing with struct-first defaults
---

# XCTest to Swift Testing

## Scope and Preconditions

This skill is for migrating existing XCTest tests to Swift Testing in incremental steps.

- Output is guidance and edits only. This skill does not include an automatic rewrite script.
- XCTest and Swift Testing can coexist in the same package during migration.
- Keep migrations target-by-target or file-by-file to reduce risk.

Before migrating:

- Confirm the project builds and tests pass in baseline state.
- Inventory XCTest-only features in the target file (`XCTestExpectation`, `measure`, heavy `XCTestCase` lifecycle usage).
- Pick the destination test type using the rules in `Struct-First Defaults`.

## Struct-First Defaults

Use this default unless one of the exceptions below applies.

- Default target shape:
  - `import Testing`
  - `struct MyTests { ... }`
  - `@Test` for each test function
- Keep as `class` when:
  - you must preserve class inheritance patterns used by shared base test classes,
  - you need `deinit` cleanup semantics directly on the test container,
  - or migration scope is intentionally minimal and retaining class form reduces churn.

## Assertion Transformations

| XCTest | Swift Testing | Notes |
|--------|---------------|-------|
| `XCTAssert(condition)` | `#expect(condition)` | |
| `XCTAssertTrue(condition)` | `#expect(condition)` | |
| `XCTAssertFalse(condition)` | `#expect(!condition)` | |
| `XCTAssertNil(value)` | `#expect(value == nil)` | |
| `XCTAssertNotNil(value)` | `#expect(value != nil)` | If value is used later, prefer `let value = try #require(value)`. |
| `XCTAssertEqual(a, b)` | `#expect(a == b)` | |
| `XCTAssertNotEqual(a, b)` | `#expect(a != b)` | |
| `XCTAssertIdentical(a, b)` | `#expect(a === b)` | |
| `XCTAssertNotIdentical(a, b)` | `#expect(a !== b)` | |
| `XCTAssertGreaterThan(a, b)` | `#expect(a > b)` | |
| `XCTAssertGreaterThanOrEqual(a, b)` | `#expect(a >= b)` | |
| `XCTAssertLessThan(a, b)` | `#expect(a < b)` | |
| `XCTAssertLessThanOrEqual(a, b)` | `#expect(a <= b)` | |
| `try XCTUnwrap(optional)` | `try #require(optional)` | |
| `XCTFail("message")` | `Issue.record("message")` | Use when control flow hits an impossible path. |

Message handling:

- Preserve assertion intent first. Keep custom reason strings where they improve failure triage.
- Do not mechanically carry `file:`/`line:` parameters; Swift Testing captures source context.

## Error Handling Patterns

### Type-Only Throw Check

```swift
#expect(throws: (any Error).self) {
    try service.validate()
}
```

### Specific Error Value Check

```swift
#expect(throws: ValidationError.invalidFormat) {
    try parser.parse("1.0.")
}
```

### Error Inspection with Additional Assertions

Use this when XCTest previously used `XCTAssertThrowsError(..., { error in ... })`.

```swift
#expect {
    try command.validate()
} throws: { error in
    guard let validationError = error as? ValidationError else {
        Issue.record("Expected ValidationError")
        throw error
    }
    return validationError.description == "Bundle Identifiers cannot be empty."
}
```

Guidance:

- Return `true` only when the inspected error satisfies your assertion.
- Re-throw unknown error types so the test fails with useful context.
- Reserve `Issue.record(...)` for impossible branches or extra diagnostics.

## Lifecycle and Test Structure Migration

| XCTest pattern | Swift Testing target | Notes |
|----------------|----------------------|-------|
| `import XCTest` | `import Testing` | |
| `final class MyTests: XCTestCase` | `struct MyTests` (default) | Keep class only when needed (see exceptions above). |
| `func testSomething()` | `@Test func testSomething()` | |
| `override func setUp()` | `init()` or helper factory | Preserve sync/async/throws behavior as needed. |
| `override func setUpWithError() throws` | `init() throws` or throwing helper | |
| `override func setUp() async throws` | `init() async throws` or async helper | |
| `override func tearDown()` | local cleanup strategy | In struct suites, prefer local scope cleanup (`defer`, helper object lifetime). |
| `override func tearDownWithError() throws` | manual migration | No direct 1:1 in struct suites; refactor cleanup paths. |

Important:

- Do not blindly map `tearDown` to `deinit` for struct suites. Structs do not support `deinit`.
- If cleanup must run when test container is destroyed, keep class-based tests and use `deinit` there.

## Struct-Based State Management

Use property mutability intentionally:

- `let` for dependencies that should never be reassigned.
- `var` for subject-under-test state that tests mutate.
- `@Test mutating` only when the test needs to mutate `self` properties.

Example:

```swift
struct BumpCoreTests {
    private let service: Service
    private var config: Config

    init() {
        self.service = Service()
        self.config = Config()
    }

    @Test mutating func updatesVersion() {
        service.apply(to: &config)
        #expect(config.version == "1.0.0")
    }
}
```

### Closure Capture Pattern for Mutable Logs or Buffers

Use a helper reference type when an escaping closure must capture mutable state during initialization.

```swift
struct CommandTests {
    final class LogCapture {
        var logs: [String] = []
    }

    private let logCapture: LogCapture
    private let command: Command

    init() {
        let logCapture = LogCapture()
        let command = Command(logger: { logCapture.logs.append($0) })
        self.logCapture = logCapture
        self.command = command
    }
}
```

## Parameterized Tests with `@Test(arguments:)`

Convert loop-based case tables when it improves readability and reporting.

```swift
@Test(arguments: [
    (input: "1.0.0", expected: true),
    (input: "1.0.", expected: false),
    (input: "1.0.0.1", expected: true)
])
func validatesVersion(input: String, expected: Bool) {
    #expect(Validator.isValid(input) == expected)
}
```

Use parameterization when:

- each case is independent,
- the same assertion logic repeats,
- per-case reporting is valuable.

Avoid parameterization when:

- each case needs different setup/teardown shape,
- or each case has different multi-step assertions.

## Unsupported or Manual Migration Matrix

| XCTest feature | Swift Testing status | Recommended migration action |
|----------------|----------------------|------------------------------|
| `XCTestExpectation` / `waitForExpectations` | No direct 1:1 replacement pattern | Refactor around `async` APIs where possible. Keep XCTest temporarily if refactor is high-risk. |
| `measure {}` performance tests | XCTest-specific | Keep as XCTest or move to dedicated benchmarking tooling. |
| Heavy `XCTestCase` lifecycle coupling | Partial | Prefer helper factories and local cleanup. Keep class form if needed. |
| `tearDownWithError` behavior with strict guarantees | No direct struct equivalent | Manual migration required. Consider class-based suite retention. |
| Test ordering dependencies | Discouraged in both frameworks | Refactor tests to be order-independent before migration. |

## Deterministic Migration Workflow

1. Inventory current XCTest usage in the target file.
2. Choose destination type (`struct` default, `class` by exception).
3. Convert imports, suite declaration, and `test...` methods to `@Test`.
4. Migrate setup logic (`setUp*`) to `init` or helper builders.
5. Replace assertions using the transformation table.
6. Migrate throwing assertions with typed `#expect` patterns.
7. Convert simple looped case tables to `@Test(arguments:)` where useful.
8. Resolve unsupported XCTest features using the `Unsupported or Manual Migration Matrix`.
9. Run verification commands and fix any regression.

Stop conditions:

- Do not force migration when unsupported XCTest features dominate the file.
- Do not rewrite architecture just to satisfy framework parity.

## Post-Migration Verification

Run after each migration batch:

1. Build:
```bash
swift build
```

2. Run all tests:
```bash
swift test
```

3. Run targeted tests for migrated scope when debugging failures:
```bash
swift test --filter <TargetName>
```

4. Check for leftover XCTest usage intentionally or accidentally:
```bash
rg -n "import XCTest|XCTestCase|XCTAssert|XCTFail|XCTUnwrap|XCTestExpectation|measure\\(" Tests
```

Expected output behavior in mixed state:

- XCTest output: `Test Suite` / `Test Case` lines.
- Swift Testing output: `◇` start markers and `✔` pass lines.
- Both outputs together are valid during incremental migration.

Common migration failures and fixes:

- Missing `@Test`: add annotations to migrated test functions.
- Mutation errors in struct tests: add `mutating` only where needed.
- Capture-before-initialization errors: use helper reference types for closure-captured mutable state.
- Incorrect lifecycle mapping: remove struct `deinit` assumptions and use local cleanup strategies.
