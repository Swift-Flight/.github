# Flight

A modular server-side framework for Swift.

Dependency injection and application lifecycle at the bottom, HTTP and
WebSockets above it, real-time layers on top of those, and persistence beside
them. Six packages, each usable on its own.

```bash
flight new MyService
cd MyService && swift run MyService
# → MyService is flying, on http://127.0.0.1:8080
```

That is the skeleton: configuration, dependency injection, HTTP, and health
endpoints, with nothing else running. `--tier basics` adds a database and
`--tier demo` adds the rest.

## The packages

| Repository | What it is |
| --- | --- |
| [flight](https://github.com/Swift-Flight/flight) | The framework: container and lifecycle, configuration, HTTP, WebSockets, PubSub, Channels, Presence, actuator endpoints, and token authentication |
| [flight-data](https://github.com/Swift-Flight/flight-data) | Persistence and caching: data-source protocols, an in-memory cache, migrations, and the PostgreSQL and Valkey drivers |
| [flight-cli](https://github.com/Swift-Flight/flight-cli) | The `flight` command, the starter templates it generates, and the tutorial |
| [hangar](https://github.com/Swift-Flight/hangar) | A typed query builder and repository for PostgreSQL. Usable outside Flight |
| [swift-changeset](https://github.com/Swift-Flight/swift-changeset) | Ecto-style changesets: collect changes, validate, apply only what is valid and only what changed. No Flight dependency |
| [flight-channels-js](https://github.com/Swift-Flight/flight-channels-js) | The browser and Node client for the Channels wire protocol |

## Start here

**[The tutorial](https://github.com/Swift-Flight/flight-cli/blob/main/TUTORIAL.md)**
builds one application in three parts, each ending at a project you can
download and run: configuration and HTTP, then a database, then a real-time
chat room with presence and authentication. Every stage ends with a command
and what you should see.

If you would rather read code than prose, the
[demo](https://github.com/Swift-Flight/flight-cli/tree/main/templates/demo) is
the finished application.

## What it looks like

Registration happens at build time. A plugin scans your target for
`@Controller`, `@Service`, `@Repository` and `@Component` and generates the
wiring — adding a controller does not mean editing a list, and a misspelled
configuration key is a compile error rather than a page at 3am.

```swift
@Controller
struct UserController {
    @GetMapping("/users/:id")
    func get(_ context: RequestContext) async throws -> User {
        guard let id = context.pathParam("id").flatMap({ UUID(uuidString: $0) }) else {
            throw HTTPError(.badRequest, "user id must be a UUID")
        }
        let users = try context.resolve((any UserRepositoryProtocol).self)
        guard let user = try await users.find(byID: id) else {
            throw HTTPError(.notFound, "no user \(id)")
        }
        return user
    }
}
```

Composition is by module, and modules form a DAG the framework orders. Scopes
are explicit: a request-scoped component holds one pooled connection for that
request, and resolving one outside a scope is an error rather than a surprise
connection.

## Design

A few decisions worth knowing before you invest in it:

**Bring your own auth.** Flight validates tokens your identity provider
issued. There is no password hashing, no session store, and no token issuance
anywhere in it — any OIDC-compliant provider is configuration, not a fork. The
one security-critical primitive, signature verification, is delegated to
JWTKit.

**No hand-rolled HTTP.** The default transport wraps HummingbirdCore rather
than reimplementing HTTP/1.1 correctness, request-smuggling mitigations, and
WebSocket framing. Flight owns routing and dispatch; a transport is a module,
and any conforming one is a peer.

**Take only what you use.** Both packages ship many products behind traits, so
an application that wants the container and lifecycle resolves 7 packages
rather than 29, and one using the in-memory cache never resolves a Postgres
driver.

**Testing is the point of the injection.** Suites run real controllers, real
routing and real middleware against fakes, with no socket and no database —
which is why the demo's real-time tests exercise the actual Channels router,
PubSub fan-out, and Presence CRDT in memory.

## Status

**v0.1.x — early.** The APIs work and are tested, and they are not yet stable
across versions. Requires Swift 6.3 or later.

## License

MIT.
