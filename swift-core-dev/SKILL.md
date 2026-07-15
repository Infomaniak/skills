---
name: swift-core-dev
description: Use when writing, editing, or reviewing Swift code in any Swift project: apps, frameworks, packages, extensions, or CLIs. Enforces pragmatic modern Swift conventions: structured concurrency, InfomaniakConcurrency, InfomaniakDI, actors, Sendable, typed errors, Swift Testing, clear boundaries, and testable design. Trigger keywords: Swift, core, async/await, actor, Sendable, InfomaniakConcurrency, InfomaniakDI, Swift Testing, XCTest, Package.swift, xcodebuild.
---

# Swift Core Dev

Write modern, review-ready Swift that matches the current codebase. Inspect the
project structure, build files, tests, existing naming, and architecture before
editing. Follow explicit local conventions first; use this skill as the default
when the project is silent or inconsistent.

Read project documentation before changing behavior: `README`, `AGENTS.md`,
`CONTRIBUTING`, `docs/`, architecture notes, and CI/build scripts. When a change
invalidates documented behavior, update the matching doc in the same change.

## Golden rules

1. Optimize for clarity at the call site (Swift API Design Guidelines). Full
   words over abbreviations: `maximumConcurrentOperations`, not `maxOps`.
2. Value types first: model data as `struct`/`enum`. Reach for `class`/`actor`
   only for identity or shared mutable state.
3. Concurrency is explicit. Prefer `async`/`await`, structured concurrency,
   `actor`, isolated state, and `InfomaniakConcurrency` when available or
   appropriate to add. Avoid locks, semaphores, and `@unchecked Sendable`.
4. Errors are typed enums, mapped deliberately at system boundaries.
5. Keep files small and split along feature / domain / service / platform /
   persistence boundaries before they get hard to review. Keep files under 1,000
   lines; split earlier when a type has multiple responsibilities. Put
   extensions in `Type+Feature.swift`.
6. Document reasons, not syntax. Add comments only when code is doing something
   non-obvious that cannot be guessed by reading the implementation.
7. Never log, commit, or store bearer/refresh/ID tokens, account identifiers,
   private URLs, or user data. Tests use redacted strings and mocked sessions.

## Types and modeling

Use predictable, value-oriented models:

- Public value types conform to the conformances they actually need, in a stable
  order: `Codable, Equatable, Identifiable, Sendable`.
- DTOs, request/response models, persistence payloads, and other boundary types
  are `Sendable` by default. Only omit `Sendable` when the type cannot safely
  cross concurrency domains, and make that boundary explicit.
- Provide an explicit `public init(...)` for public structs (do not rely on the
  synthesized internal memberwise init across module boundaries).
- Use `let` for stored properties; derive secondary values as computed
  properties instead of storing duplicated state.
- Model behavior as small structs behind a protocol rather than free functions
  or type-level helpers. A protocol-backed struct can be injected and swapped for
  a mock/fake in tests, which type-level helpers cannot.
- Model closed sets as enums with associated values and give them a
  `rawValue`/`init(rawValue:)` bridge when they cross a string boundary such as
  an API, database, URL, or file format.

```swift
public struct UserProfile: Codable, Equatable, Identifiable, Sendable {
    public let id: UUID
    public let displayName: String
    public let email: String

    public init(id: UUID, displayName: String, email: String) {
        self.id = id
        self.displayName = displayName
        self.email = email
    }
}
```

## Errors

- Define domain errors as `enum ...: Error, Equatable, LocalizedError, Sendable`
  with an `errorDescription` switch when they are user- or log-facing.
- Use associated values to carry context, not string interpolation into a single
  case.
- At network, persistence, UI, extension, and public API boundaries, map errors
  deliberately. Preserve domain errors callers can act on; map `URLError`,
  `DecodingError`, database errors, and platform errors into the boundary's
  expected vocabulary instead of leaking implementation details by accident.
- Validate inputs with `guard ... else { throw }` early; prefer throwing over
  returning optionals when the caller cannot proceed.

## Concurrency

This is the highest-signal area. Prefer boring, structured concurrency over
custom scheduling.

- Prefer `async`/`await` and structured concurrency (`async let`,
  `withThrowingTaskGroup`). Use `InfomaniakConcurrency` where it fits and the
  project can accept the dependency instead of hand-rolling concurrency
  primitives, limiters, gates, queues, or retry helpers. Write a local primitive
  only when the project has a behavior the library does not provide, and cover it
  with focused concurrency tests.
- Use `actor` to protect shared mutable state (permit counts, waiter queues,
  test probes). Do not guard state with `NSLock`/`DispatchSemaphore`.
- Mark cross-boundary types `Sendable`. Constrain generics with
  `<T: Sendable>` and closures with `@Sendable` when they escape into tasks.
- Respect cancellation: call `try Task.checkCancellation()` at suspension-worthy
  points, bridge blocking waits with `withCheckedThrowingContinuation` +
  `withTaskCancellationHandler`, and always resume every continuation exactly
  once on every path (success, throw, cancel).
- Release resources on all exit paths. If acquiring a permit, file handle,
  transaction, or observation token, release it in success, error, and
  cancellation paths.
- Bound fan-out. Use task groups deliberately and cap concurrent work when
  touching network, disk, CPU-heavy parsing, databases, or platform APIs. Do not
  spawn unbounded tasks.
- Do NOT reach for `@unchecked Sendable`. If you think you need it, isolate the
  state in an actor or make the type a value type instead. If it is truly
  unavoidable, justify it in a comment.

## Dependency injection and testability

- Depend on protocols, not concretions, at meaningful boundaries: networking,
  persistence, clocks, UUID generation, file systems, platform services, and
  analytics. Service protocols should usually be `Sendable`.
- New networking or persistence behavior gets a client/service method and a
  mock/fake for tests, not ad hoc `URLSession`, SQL, or file-system calls spread
  through unrelated call sites.
- Split public-facing behavior behind small `Sendable` interfaces when that
  boundary would make the code easier to test or extract into a library later.
  Avoid protocol noise for private one-off helpers.
- Inject collaborators through initializers with sensible defaults so production
  code stays terse and tests stay hermetic:
  `session: URLSession = .shared`, `date: Date = Date()`,
  `timeZone: TimeZone = .current`.
- Do not introduce `static let shared = Type()` singletons. Register shared
  services through dependency injection, preferably `InfomaniakDI` when available
  or appropriate to add, so tests can replace them cleanly.
- Prefer typed request/response flows and dedicated API clients over hand-built
  HTTP in feature code. Keep transport details at the network boundary.
- Keep persistence behind protocols or repositories so tests can use in-memory,
  temporary-file, or fake variants.

## Modern Swift idioms to prefer

- Optional binding shorthand: `if let cursor { ... }`, `guard let nextCursor`.
- Key paths in higher-order functions: `map(\.id)`, `Set(items.map(\.id))`.
- Use standard library initializers such as `Dictionary(uniqueKeysWithValues:)`
  when they express intent more clearly than manual loops.
- Use explicit, safe encoding APIs such as `Data(string.utf8)` instead of forced
  string conversions or implicit encodings.
- Prefer `switch` over chained `if` for closed enums; keep cases exhaustive with
  no `default` when the set is fixed, so new cases surface as compiler errors.

## Testing

- New unit tests use Swift Testing when the project supports it: `import Testing`,
  `@Suite`, `@Test`, `#expect`, `#require`. Use XCTest when the target, platform,
  or existing suite requires it. UI automation usually stays on XCTest.
- Keep tests parallel by default. Mark suites/tests serialized only when they
  touch unavoidable shared process state such as global URLProtocol handlers,
  environment variables, singletons, keychains, or static caches.
- When the project uses `InfomaniakDI`, mock DI-registered services by
  registering test doubles in the dependency container. This usually requires
  explicit test scaffolding: register mocks at the beginning of the test or suite
  and reset/restore registrations afterward so tests stay isolated.
- Use temporary directories with `defer { try? FileManager.default.removeItem(at:) }`
  for filesystem tests; never write into the repo, real app group, or user data.
- Clean up every resource a test creates, locally or remotely: files,
  directories, database rows, server-side objects, accounts, jobs, queues, and
  registrations. Register cleanup immediately after creation with `defer`, test
  teardown hooks, or the framework's cleanup API so failures do not leak state.
- Drive concurrency tests with actor-based gates/probes and timeouts rather than
  sleeps-as-assertions.
- No live network in the default test path. Mock `URLSession`/URLProtocol and use
  redacted token strings.
- Add or update tests for every meaningful behavior change.

## Validation (required before claiming completion)

Use the repository's documented validation path first: README, AGENTS,
CONTRIBUTING, Makefile, scripts, Fastlane, CI config, or package manager docs.
If no project-specific command exists, choose the narrowest command that proves
the change:

```sh
# Swift Package Manager
swift build
swift test
swift test --filter <SuiteOrTestName>

# Xcode project or workspace: first discover schemes, then build/test the target
xcodebuild -list -project <Project>.xcodeproj
xcodebuild -list -workspace <Workspace>.xcworkspace
xcodebuild build -scheme <Scheme> -destination '<destination>'
xcodebuild test -scheme <Scheme> -destination '<destination>'
```

For multi-platform projects, validate every platform affected by runtime
behavior. For generated projects (Tuist, XcodeGen, Bazel, Buck, custom scripts),
use the checked-in documented workflow. When iterating on a focused change, run a
filtered Swift test suite such as `swift test --filter <SuiteOrTestName>` for the
specific suite or test first, then run broader validation when the change risk
requires it. If the full suite is too expensive or cannot run, run the closest
targeted validation and say exactly what was skipped and why.

## Commit hygiene

Use Conventional Commits (`feat:`, `fix:`, `test:`, `refactor:`, `docs:`). Keep
changes small and focused; do not mix unrelated refactors, formatting churn, or
dependency bumps with feature work. Never commit build artifacts, DerivedData,
caches, `.DS_Store`, credentials, or fixtures with real data.
