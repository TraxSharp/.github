# Trax

A .NET framework for business logic that has to do more than run once. You write a unit of work
as a typed pipeline — a **train** of **junctions**, each with one job — and every layer above treats
that train as its unit: a GraphQL schema generated from the trains you tag, auth that fails startup
rather than ship an open endpoint, cron timetables with retries and dead letters, execution history
persisted per run, workers on SQS or Lambda or any HTTP endpoint, and a dashboard that mounts into
the app you already have. Each layer is a package on top of the one below, so you stop where your
problem stops: Core alone is pipelines with no DI, no database and no ASP.NET.

Requires **.NET 10**. Full documentation: **[traxsharp.net/docs](https://traxsharp.net/docs)**.

## What it looks like

A junction does one thing:

```csharp
public class ValidateEmailJunction : Junction<CreateUserRequest, ValidatedEmail>
{
    public override Task<ValidatedEmail> Run(CreateUserRequest input) =>
        input.Email.Contains('@')
            ? Task.FromResult(new ValidatedEmail(input.Email))
            : throw new ValidationException("Invalid email format");
}
```

A train declares the route. Junction outputs wire to the next junction's input by type, out of the
train's `Memory`. If one fails, the train switches to the left track and the rest never run, so
there is no try/catch and no null-checking in between:

```csharp
[TraxMutation(Description = "Creates a user")]
[TraxAuthorize(Roles = Roles.Admin)]
public class CreateUserTrain : ServiceTrain<CreateUserRequest, User>, ICreateUserTrain
{
    protected override Task<Either<Exception, User>> Junctions() =>
        Chain<ValidateEmailJunction>()
            .Chain<CreateUserInDatabaseJunction>()
            .Chain<SendWelcomeEmailJunction>()
            .Resolve();
}
```

Those two attributes are the whole difference between a train you call yourself and one that is a
gated, typed field on your GraphQL schema. `Trax.Core.Analyzers` checks the chain at compile time:
a junction whose input nothing upstream produces is a build error, not a 2am surprise.

## The vocabulary

The metaphor is consistent across the whole API surface, so it is worth thirty seconds:

| Term | Meaning |
|---|---|
| **Train** | the pipeline — it follows a route and always reaches a destination |
| **Junction** | a point on the route where work happens; continue right (success) or switch left (failure) |
| **Memory** | the cargo the train carries between junctions, wired by type |
| **ServiceTrain** | a train with the equipment bolted on: DI, execution tracking, lifecycle hooks |
| **Manifest** | a scheduled job definition — which train, on what timetable, with what cargo |

## Pick your layer

| Layer | Package | What it adds |
|---|---|---|
| Core | `Trax.Core` | Trains, junctions, Memory, railway error propagation, the chain analyzer |
| Effect | `Trax.Effect` | DI-resolved junctions and a persisted execution record per run |
| Mediator | `Trax.Mediator` | `TrainBus`: dispatch by input type, so callers never name a concrete train |
| Scheduler | `Trax.Scheduler` | Manifests, timetables, retries, dead letters, job dependencies |
| API | `Trax.Api.GraphQL` | A HotChocolate schema generated from your trains and entities |
| Dashboard | `Trax.Dashboard` | A Blazor Server operations UI at `/trax` |

## What you get around a train

**A schema that writes itself.** `[TraxQuery]` and `[TraxMutation]` turn a train into a typed field,
its input record becoming the field's arguments and its output the return type. `[TraxQueryModel]`
on an EF Core entity exposes the table with cursor pagination, filtering, sorting and projection.
Cross-schema edges get batched DataLoaders instead of N+1s. `AddTraxGraphQL()` wires the lot, and
`UsePersistedOperations()` locks the schema to an approved operation list.

**Auth with a posture you cannot forget.** API-key, JWT, Cognito and OIDC schemes ship as packages
and register in a line. `[TraxAuthorize]` gates a train or an entity by policy and role, and the
gate on an entity holds even when a request reaches it through a navigation property. Any exposed
surface that declares neither `[TraxAuthorize]` nor `[TraxAllowAnonymous]` fails at host startup, by
name — a forgotten gate cannot ship as a public endpoint.

**Timetables with an audit trail.** A manifest names the train, the schedule, the cargo and the
retry policy. Cron down to the second or `Every.Minutes(5)`; `Exclude.DaysOfWeek()`,
`Exclude.Dates()` and `Exclude.TimeWindow()` skip weekends, holidays and maintenance hours as
intentional skips rather than misfires. Failures retry, then dead-letter for review. Retries never
mutate a row, so the history is the record.

**Somewhere else to run it.** `UseRemoteWorkers()`, `UseSqsWorkers()` and `UseLambdaWorkers()` move
dispatch off the scheduler; `UseRemoteRun()` and `UseLambdaRun()` move direct execution off the API.
Postgres stays the source of truth in every topology and the train code never changes — the only
thing you are choosing is which machine picks up the work.

**Every run as a row.** State, timing, serialized input and output, which junction failed and why,
the parent run it hung off, and the host that executed it — auto-detected as Lambda, ECS,
Kubernetes, Azure App Service or a plain server. Store it in Postgres, SQLite or memory. The
dashboard reads execution history, shows which junction a run is sitting on, cancels a run across
servers, sets per-group concurrency and priority, and toggles effect providers at runtime.

**Real time, when you want it.** `Trax.Effect.Broadcaster.SignalR` and `.RabbitMQ` push execution
events as they happen, and `[TraxBroadcast]` publishes a train's lifecycle to the built-in
`onTrainCompleted` GraphQL subscription without a custom hook.

**State machines that cross the wire.** `Trax.Effect.StateMachine` models a multi-step flow as a
serializable snapshot a C# backend and a TypeScript client agree on: illegal (state, data)
combinations are unrepresentable, every operation returns a typed result instead of throwing, and an
instance rebuilds from storage so a user can start on one device and finish on another. The engine
exists in both languages and is held identical by shared conformance fixtures.

**Conventions you can enforce.** `Trax.Core.Testing`, `Trax.Effect.Data.Testing`,
`Trax.Mediator.Testing` and `Trax.Api.GraphQL.Testing` ship the architectural rules as base fixtures
with the `[Test]` methods already written. Subclass, configure, `dotnet test`.
([reference](https://traxsharp.net/docs/reference/architecture-guards))

## Start

A working project with PostgreSQL, scheduling and the dashboard:

```bash
dotnet new install Trax.Samples.Templates
dotnet new trax-scheduler -n MyApp    # scheduler + dashboard
dotnet new trax-api -n MyApp.Api      # GraphQL API
```

Already have a schema? The CLI scaffolds the trains from it:

```bash
dotnet tool install --global Trax.Cli
trax generate --schema ./schema.graphql --output ./MyApp --name MyApp   # or an OpenAPI spec
trax machine new --name checkout                                        # state-machine artifacts
```

Or add one package and write a train:

```bash
dotnet add package Trax.Core
```

Then read [Getting Started](https://traxsharp.net/docs/getting-started), which walks the same
example from Core-only to the full stack. Inlay-hint extensions for
[VS Code](https://marketplace.visualstudio.com/items?itemName=Trax.Core.trax-hints) and Rider /
ReSharper ("Trax.Core Chain Hints" in the JetBrains Marketplace) show the `TIn -> TOut` of every
link in a chain inline.

## Repositories

| Repo | What's in it |
|---|---|
| [Trax.Core](https://github.com/TraxSharp/Trax.Core) | Trains, junctions, Memory, the chain analyzer, the IDE plugins |
| [Trax.Effect](https://github.com/TraxSharp/Trax.Effect) | `ServiceTrain`, execution metadata, data providers, effect providers, broadcasters, the state-machine engine |
| [Trax.Mediator](https://github.com/TraxSharp/Trax.Mediator) | `TrainBus`, train discovery, concurrency limiting, train authorization |
| [Trax.Scheduler](https://github.com/TraxSharp/Trax.Scheduler) | Manifests, timetables, dead letters, the SQS and Lambda runners |
| [Trax.Api](https://github.com/TraxSharp/Trax.Api) | The GraphQL layer, auth, audit, persisted operations, typed clients |
| [Trax.Dashboard](https://github.com/TraxSharp/Trax.Dashboard) | The Blazor Server monitoring UI |
| [Trax.Cli](https://github.com/TraxSharp/Trax.Cli) | The `trax` global tool |
| [Trax.Samples](https://github.com/TraxSharp/Trax.Samples) | Runnable samples and the `dotnet new` templates |
| [Trax.Docs](https://github.com/TraxSharp/Trax.Docs) | Documentation content and the decision records |
| [Trax.Website](https://github.com/TraxSharp/Trax.Website) | Source for [traxsharp.net](https://traxsharp.net) |

Every package is on [NuGet](https://www.nuget.org/profiles/TraxSharp). Versions are cut by Semantic
Release from conventional commits, per repo.

## Samples

[Trax.Samples](https://github.com/TraxSharp/Trax.Samples) covers deployment shapes rather than toy
snippets: everything in one process, a separate API, distributed workers polling a job table,
ephemeral runners pushed over HTTP, a real-time chat service, a test-runner hub, and a multi-schema
library app (Bookworm) that is the reference consumer of every architecture guard.

## Documentation

- [Getting Started](https://traxsharp.net/docs/getting-started) — Core-only through full stack
- [SDK Reference](https://traxsharp.net/docs/sdk-reference) — every public API, method by method
- [Trax vs Quartz.NET vs Hangfire](https://traxsharp.net/docs/reference/comparison) — what each one is actually for
- [Benchmarks](https://traxsharp.net/docs/reference/benchmarks) — the overhead a train costs you, measured
- [API Security](https://traxsharp.net/docs/api-security) and [Supply Chain Security](https://traxsharp.net/docs/supply-chain-security) — read before you deploy
- [Migrating from ChainSharp](https://traxsharp.net/docs/reference/migration)

## License

MIT.

## Trademark & Brand Notice

Trax is an open-source .NET framework provided by TraxSharp. This project is an independent
community effort and is not affiliated with, sponsored by, or endorsed by the Utah Transit
Authority, Trax Retail, or any other entity using the "Trax" name in other industries.
