---
title: "18 - Server-Side Swift: APIs, NIO, Vapor, and Observability"
created: 2026-06-07
updated: 2026-06-07
tags: [swift, programming-languages, server-side-swift, vapor, observability]
aliases: []
---

# 18 - Server-Side Swift: APIs, NIO, Vapor, and Observability

[toc]

> **TL;DR:** Server-side Swift is SwiftPM-first, Linux-aware, async-heavy, and observability-driven. Vapor and Hummingbird sit above lower-level networking foundations like SwiftNIO; production services need release builds, structured logging, metrics, tracing, tests, and containers.

## Real-World Example

This minimal Vapor route receives JSON and returns JSON. It shows why server-side Swift feels familiar to app developers but still requires backend discipline.

```swift
import Vapor

struct GreetingRequest: Content {
    let name: String
}

struct GreetingResponse: Content {
    let message: String
}

func routes(_ app: Application) throws {
    app.post("greet") { req async throws -> GreetingResponse in
        let input = try req.content.decode(GreetingRequest.self)
        return GreetingResponse(message: "Hello, \(input.name)")
    }
}
```

## Vocabulary

**SwiftNIO**: A low-level asynchronous networking framework used by many server-side Swift libraries.

---

**Vapor**: A popular full-featured Swift web framework.

---

**Hummingbird**: A lightweight Swift server framework with fewer dependencies.

---

**Event loop**: A loop that drives nonblocking IO callbacks and scheduled work.

---

**Backpressure**: A mechanism for preventing producers from overwhelming consumers.

---

**Structured logging**: Logs with fields such as request ID, route, status, latency, and error.

---

**Tracing**: Recording a request's path across services and async boundaries.

## Intuition

Server-side Swift is not "iOS code on Linux." It is backend engineering in Swift. You need HTTP semantics, database boundaries, migrations, deployment, observability, security, and performance testing.

The language helps with typed models, async/await, errors, and performance. It does not remove the need for timeouts, resource limits, request validation, authentication, rate limiting, and safe deployments.

## Project Shape

Keep route handlers thin. Put domain logic in services and pure types.

```text
Sources/
  App/
    routes.swift
    configure.swift
    main.swift
  Core/
    Models/
    Services/
Tests/
  CoreTests/
  AppTests/
```

## Vapor Startup

The official Swift.org Vapor guide starts with the Vapor toolbox and template.

```bash
brew install vapor
vapor new HelloVapor
cd HelloVapor
swift run
```

For CI and containers, rely on SwiftPM commands directly.

```bash
swift test
swift build -c release
```

## Observability

At minimum, log one structured event per request and one structured event per user-impacting failure. Keep secrets and PII out of logs.

```swift
req.logger.info("handled request", metadata: [
    "route": "POST /greet",
    "status": "200"
])
```

Swift.org's log-level guidance recommends restraint for libraries: `trace` and `debug` are generally safe for library internals, while applications decide production verbosity.

## Deployment

For containers, build release binaries inside a Swift image and deploy the runtime artifact.

```dockerfile
FROM swift:latest AS build
WORKDIR /workspace
COPY . .
RUN swift build -c release --static-swift-stdlib

FROM debian:stable-slim
COPY --from=build /workspace/.build/release/App /usr/local/bin/App
CMD ["App", "serve", "--hostname", "0.0.0.0"]
```

## Production Checklist

- Health check endpoint.
- Graceful shutdown.
- Request size limits.
- Timeouts for downstream calls.
- Structured logs with request IDs.
- Metrics for latency, throughput, errors, and queue depth.
- Tracing for cross-service calls.
- Database migration strategy.
- Secrets from environment or secret manager, not source.
- Linux test and release build in CI.

## Pitfalls

- **Blocking event loops**: CPU-heavy or blocking work can stall many requests.
- **No request IDs**: Debugging production without correlation IDs is slow.
- **Debug builds in containers**: Release mode is mandatory for realistic performance.
- **App code in routes**: Route handlers should orchestrate, not contain the domain.
- **Ignoring Linux differences**: Test in the OS you deploy.

## Exercises

1. Create a Vapor route that accepts JSON and returns typed JSON.
2. Add a request ID to logs.
3. Move route logic into a service and unit-test the service.
4. Write a Dockerfile that builds release mode.

## Sources

- https://www.swift.org/getting-started/vapor-web-server/
- https://docs.vapor.codes/
- https://docs.hummingbird.codes/2.0/documentation/hummingbird/
- https://www.swift.org/documentation/server/guides/
- https://www.swift.org/documentation/server/guides/libraries/log-levels.html
- https://www.swift.org/documentation/server/guides/packaging.html
- Conversation with user on 2026-06-07

## Related

- Previous: [17 - SwiftUI State, Architecture, and App Patterns](./17-swiftui-state-architecture-and-app-patterns.md)
- Next: [19 - Advanced Testing: Fakes, Async, UI Boundaries, and CI](./19-advanced-testing-fakes-async-ui-boundaries-and-ci.md)
- Earlier: [08 - SwiftPM, Testing, Compiling, and Shipping](./08-swiftpm-testing-compiling-and-shipping.md)

