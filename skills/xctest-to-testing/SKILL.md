---
name: xctest-to-testing
description: Migrate XCTest-based Swift tests to Swift Testing incrementally. Use when converting XCTestCase suites, assertions, setup/teardown, expectations/callback tests, skip logic, known issues, serialized execution, and attachments to `Testing` APIs.
---

# XCTest to Swift Testing

## Scope and Preconditions

Use this skill to migrate existing XCTest tests to Swift Testing with minimal churn.

- Provide guidance and edits only. Do not assume an automatic rewrite script.
- Allow XCTest and Swift Testing to coexist during migration.
- Keep migrations target-by-target or file-by-file to reduce risk.

Before migrating:

- Confirm the project builds and tests pass in baseline state.
- Inventory XCTest-only features in the target file (`XCTestExpectation`, `XCTSkip*`, `XCTExpectFailure`, `measure`, heavy `XCTestCase` lifecycle usage).
- Pick the destination test type using the rules in `Struct-First Defaults`.

## Struct-First Defaults

Use struct-based suites by default.

- Default target shape:
  - `import Testing`
  - `struct MyTests { ... }`
  - `@Test` for each test function
- Keep as `class` when:
  - Preserve class inheritance from shared base test classes.
  - Preserve cleanup that must run in `deinit`.
  - Reduce migration churn intentionally in a narrow scope.
- Import both `XCTest` and `Testing` when a file intentionally contains mixed tests.

## Execution Model and Actor Isolation

- Assume migrated tests run on an arbitrary task, not on the main actor by default.
- Add `@MainActor` when test logic must run on the main thread.
- Use `await MainActor.run { ... }` for small thread-sensitive regions.
- Prefer explicit isolation over relying on XCTest's historical synchronous-main-thread behavior.

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
| `XCTAssertThrowsError(try f())` | `#expect(throws: (any Error).self) { try f() }` | Capture and inspect the error when needed. |
| `XCTAssertNoThrow(try f())` | `#expect(throws: Never.self) { try f() }` | |
| `try XCTUnwrap(optional)` | `try #require(optional)` | |
| `XCTFail("message")` | `Issue.record("message")` | If used inside async/network test doubles, also propagate a failure result to avoid hanging callbacks. |

Message handling:

- Preserve assertion intent first. Keep custom reason strings where they improve failure triage.
- Do not mechanically carry `file:`/`line:` parameters; Swift Testing captures source context.
- Do not force-translate `XCTAssertEqual(..., accuracy: ...)`; use an approximate comparison helper (for example, `isApproximatelyEqual` from `swift-numerics`) when needed.

## Failure Semantics (`continueAfterFailure`)

- Do not migrate `continueAfterFailure = false` directly.
- Use `try #require(...)` for gate conditions that should stop the current test early.
- Keep using `#expect(...)` for checks that should record failures but allow the test to continue.

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

## Async Callback Migration (Important)

Use the following rule:

- Use `await confirmation(...)` for event-style APIs where the callback should occur before the confirmation scope returns.
- Use continuations for callback-driven APIs that complete later (for example, `URLSession` or queue-delivered callbacks).

Key semantics of `confirmation(...)`:

- Expect one confirmation by default.
- Pass `expectedCount:` to require exact counts.
- Pass a lower-bounded range (for example, `10...`) when over-fulfill is acceptable but a minimum is required.
- Ensure confirmations happen before the confirmation closure returns.

Use this pattern:

```swift
let value = await withCheckedContinuation { continuation in
    service.load { result in
        continuation.resume(returning: result)
    }
}
#expect(value.isSuccess)
```

XCTest-to-Continuation example:

```swift
// Before (XCTest)
func testFetchUsers() {
    let expectation = XCTestExpectation(description: "fetch users")
    service.fetchUsers { result in
        XCTAssertNotNil(result.value)
        expectation.fulfill()
    }
    wait(for: [expectation], timeout: 1)
}

// After (Swift Testing)
@Test
func fetchUsers() async {
    let result = await withCheckedContinuation { continuation in
        service.fetchUsers { result in
            continuation.resume(returning: result)
        }
    }
    #expect(result.value != nil)
}
```

Use `withCheckedThrowingContinuation` when the callback carries throwing semantics.

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

## Skip and Conditional Execution Migration

Convert runtime skip calls to traits:

| XCTest | Swift Testing |
|--------|---------------|
| `try XCTSkipIf(condition)` | `@Test(.enabled(if: !condition))` or `@Suite(.disabled(if: condition))` |
| `try XCTSkipUnless(condition)` | `@Test(.enabled(if: condition))` |

Guidance:

- Prefer suite-level traits when the condition applies to all tests in a container.
- Prefer test-level traits when only one test is conditional.

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

## Known-Issue Migration

Convert `XCTExpectFailure` blocks to `withKnownIssue`.

| XCTest | Swift Testing |
|--------|---------------|
| `XCTExpectFailure("msg") { ... }` | `withKnownIssue("msg") { ... }` |
| `XCTExpectFailure(..., options: .nonStrict()) { ... }` | `withKnownIssue(..., isIntermittent: true) { ... }` |

Use overloads with `when:` and `matching:` to replace option-based conditional matching.
- For `XCTExpectFailure(_:options:)` that affects the remainder of a test, wrap the intended scope explicitly in `withKnownIssue { ... }`.

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

## Suite Parallelism and Serialization

- Assume Swift Testing runs tests in a suite in parallel by default.
- Add `@Suite(.serialized)` for suites that cannot be made concurrency-safe yet.
- Prefer refactoring shared mutable state over blanket serialization.

## Attachment Migration

| XCTest | Swift Testing |
|--------|---------------|
| `XCTAttachment(...)` + `add(...)` | `Attachment.record(...)` |

Guidance:

- Attach values that help diagnose failures.
- Prefer types that already conform to `Attachable`, `Encodable`, or `NSSecureCoding` (with Foundation imported).
- Add custom byte serialization only when default attachment behavior is insufficient.

## Unsupported or Manual Migration Matrix

| XCTest feature | Swift Testing status | Recommended migration action |
|----------------|----------------------|------------------------------|
| `XCTestExpectation` / `waitForExpectations` | Partial mapping via `confirmation` + async refactors | Use `confirmation` for scoped event confirmation; use continuations for deferred callback completion. Keep XCTest temporarily if refactor is high-risk. |
| `measure {}` performance tests | XCTest-specific | Keep as XCTest or move to dedicated benchmarking tooling. |
| `XCTExpectFailure(_:options:)` without closure | No direct 1:1 scope-wide equivalent | Wrap the affected region (or full test body) in `withKnownIssue { ... }`. |
| Heavy `XCTestCase` lifecycle coupling | Partial | Prefer helper factories and local cleanup. Keep class form if needed. |
| `tearDownWithError` behavior with strict guarantees | No direct struct equivalent | Manual migration required. Consider class-based suite retention. |
| Test ordering dependencies | Discouraged in both frameworks | Refactor tests to be order-independent before migration. |
| Shared global test doubles (`static var` handlers) | High flake risk under concurrent execution | Isolate access with a single serialization mechanism (actor/lock) and keep setup+await+teardown in one scoped helper. |

## Deterministic Migration Workflow

1. Inventory current XCTest usage in the target file.
2. Choose destination type (`struct` default, `class` by exception).
3. Convert imports, suite declaration, and `test...` methods to `@Test`.
4. Migrate setup logic (`setUp*`) to `init` or helper builders.
5. Replace assertions using the transformation table.
6. Migrate throwing assertions with typed `#expect` patterns.
7. Convert simple looped case tables to `@Test(arguments:)` where useful.
8. Migrate skip conditions and known-issue blocks to traits / `withKnownIssue`.
9. Add `@Suite(.serialized)` only where shared state cannot yet be isolated.
10. Resolve unsupported XCTest features using the `Unsupported or Manual Migration Matrix`.
11. Run verification commands and fix any regression.

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
rg -n "import XCTest|XCTestCase|XCTAssert|XCTFail|XCTUnwrap|XCTestExpectation|XCTSkip|XCTExpectFailure|XCTAttachment|measure\\(" Tests
```

For Xcode projects (non-SPM), also run:

```bash
xcodebuild -project <Project>.xcodeproj -scheme <Scheme> -destination 'platform=iOS Simulator,name=<Device>' test
```

Expected output behavior in mixed state:

- XCTest output: `Test Suite` / `Test Case` lines.
- Swift Testing output: `◇` start markers and `✔` pass lines.q
- Both outputs together are valid during incremental migration.

Common migration failures and fixes:

- Missing `@Test`: add annotations to migrated test functions.
- Mutation errors in struct tests: add `mutating` only where needed.
- Capture-before-initialization errors: use helper reference types for closure-captured mutable state.
- Incorrect lifecycle mapping: remove struct `deinit` assumptions and use local cleanup strategies.
- Main-thread-only failures: add `@MainActor` or `MainActor.run` around thread-sensitive logic.
- Lost callback assertions: replace ad-hoc async callbacks with `confirmation` or continuations based on callback timing.
