# Trax

Trax structures .NET business logic as typed pipelines. A pipeline is a **train**. Each step is a
**junction** with one job, a typed input and a typed output. If a junction fails, the train skips
the remaining junctions and returns the exception, so there is no try/catch between steps.

Once logic lives in a train, the rest of the framework has something to operate on: the same class
can be a GraphQL field, a cron job, a Lambda invocation and a row in a dashboard without being
rewritten for any of them. Each of those is a package on top of the one below. Core on its own
pulls in no DI container, no database and no ASP.NET.

Requires .NET 10. Documentation: [traxsharp.net/docs](https://traxsharp.net/docs).

## A train

```csharp
public class ValidateEmailJunction : Junction<CreateUserRequest, ValidatedEmail>
{
    public override Task<ValidatedEmail> Run(CreateUserRequest input) =>
        input.Email.Contains('@')
            ? Task.FromResult(new ValidatedEmail(input.Email))
            : throw new ValidationException("Invalid email format");
}
```

```csharp
[TraxMutation]
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

Each junction's output is stored by type in the train's `Memory` and handed to the next junction
that asks for that type. `Trax.Core.Analyzers` fails the build when a junction needs a type nothing
upstream produces.

The two attributes are optional. With them, that train is also this:

```graphql
mutation {
  dispatch {
    createUser(input: { email: "a@b.com", name: "Ada" }, mode: QUEUE) {
      externalId
      workQueueId
    }
  }
}
```

`mode: RUN` executes it and returns the output; `QUEUE` hands it to the scheduler. Either way the
run is recorded.

## Packages

| Package | Adds |
|---|---|
| `Trax.Core` | Trains, junctions, Memory, the analyzer |
| `Trax.Effect` | DI-resolved junctions, persisted execution records |
| `Trax.Mediator` | `TrainBus`: dispatch by input type instead of naming a train |
| `Trax.Scheduler` | Manifests, timetables, retries, dead letters, job dependencies |
| `Trax.Api.GraphQL` | The HotChocolate schema built from your trains and entities |
| `Trax.Dashboard` | Blazor Server operations UI, mounted at `/trax` |

Persistence is `Trax.Effect.Data.Postgres`, `.Sqlite` or `.InMemory`.

## GraphQL

`[TraxQuery]` and `[TraxMutation]` put a train on the schema; the input record becomes the field's
arguments. `[TraxQueryModel]` on an EF Core entity exposes the table with cursor pagination,
filtering, sorting and projection. Relationships across schemas resolve through batched DataLoaders.
`AddTraxGraphQL()` registers all of it; `UsePersistedOperations()` restricts the server to an
approved operation list.

Auth is `Trax.Api.Auth` plus a scheme: API key, JWT, Cognito or OIDC. `[TraxAuthorize]` takes
policies and roles and works on trains and on entities, including when an entity is reached through
a navigation property. A surface with neither `[TraxAuthorize]` nor `[TraxAllowAnonymous]` fails at
startup with the offending types named, which keeps an ungated endpoint from reaching production by
accident.

## Scheduling

A manifest records which train runs, on what schedule, with what input, and how it retries.
Schedules are 6-field cron (second granularity) or helpers like `Every.Minutes(5)` and
`Cron.Daily(hour: 3)`. `Exclude.DaysOfWeek()`, `Exclude.Dates()`, `Exclude.DateRange()` and
`Exclude.TimeWindow()` skip weekends, holidays and maintenance windows without logging a misfire.
Retries write new rows rather than mutating old ones; jobs that exhaust them go to the dead letter
queue. Manifests can depend on other manifests.

## Running trains elsewhere

The scheduler dispatches and executes in-process by default. One call moves execution:

| Call | Where queued trains run |
|---|---|
| `UseRemoteWorkers()` | an HTTP endpoint you host |
| `UseSqsWorkers()` | Lambda, fed by SQS |
| `UseLambdaWorkers()` | Lambda, invoked directly |

`UseRemoteRun()` and `UseLambdaRun()` do the same for on-demand runs, so an API box can serve
`mode: RUN` without hosting the train code. Postgres holds manifests, metadata and the work queue in
every topology; the trains themselves are unchanged.

## Execution records

Every run writes a row: state, start and end, serialized input and output, the junction that failed,
its exception and stack trace, the parent run, and the host. Host environment is detected at startup
(Lambda, ECS, Kubernetes, Azure App Service, or a plain server), so a metadata row tells you which
machine executed it.

`Trax.Dashboard` reads those rows and adds the operational controls: cancel a run from another
server, see which junction a run is on, set per-group concurrency caps and dispatch priority, toggle
effect providers at runtime.

## Also in the box

- **State machines.** `Trax.Effect.StateMachine` holds a multi-step flow as a snapshot that a C#
  backend and a TypeScript client both understand. Every operation returns a typed result instead of
  throwing, and an instance rebuilds from stored JSON. Both engines run the same conformance fixtures.
- **Real-time.** SignalR and RabbitMQ broadcaster sinks, plus `[TraxBroadcast]` for the built-in
  `onTrainCompleted` subscription.
- **Architecture guards.** The `*.Testing` packages ship the conventions as NUnit base fixtures with
  the tests already written; you supply configuration.
- **CLI.** `trax generate` scaffolds trains from a GraphQL SDL or an OpenAPI spec. `trax machine`
  generates and checks state-machine artifacts.

## Start

```bash
dotnet new install Trax.Samples.Templates
dotnet new trax-scheduler -n MyApp    # scheduler + dashboard
dotnet new trax-api -n MyApp.Api      # GraphQL API
```

Or `dotnet add package Trax.Core` and write a train.
[Getting Started](https://traxsharp.net/docs/getting-started) walks one example from Core-only to
the full stack. There are inlay-hint extensions for
[VS Code](https://marketplace.visualstudio.com/items?itemName=Trax.Core.trax-hints) and Rider that
show `TIn -> TOut` on every link in a chain.

## Repositories

| Repo | Contents |
|---|---|
| [Trax.Core](https://github.com/TraxSharp/Trax.Core) | Trains, junctions, Memory, the analyzer, the IDE plugins |
| [Trax.Effect](https://github.com/TraxSharp/Trax.Effect) | `ServiceTrain`, metadata, data and effect providers, broadcasters, state machines |
| [Trax.Mediator](https://github.com/TraxSharp/Trax.Mediator) | `TrainBus`, train discovery, concurrency limiting, authorization |
| [Trax.Scheduler](https://github.com/TraxSharp/Trax.Scheduler) | Manifests, timetables, dead letters, SQS and Lambda runners |
| [Trax.Api](https://github.com/TraxSharp/Trax.Api) | GraphQL, auth, audit, persisted operations, typed clients |
| [Trax.Dashboard](https://github.com/TraxSharp/Trax.Dashboard) | The Blazor Server UI |
| [Trax.Cli](https://github.com/TraxSharp/Trax.Cli) | The `trax` global tool |
| [Trax.Samples](https://github.com/TraxSharp/Trax.Samples) | Samples and the `dotnet new` templates |
| [Trax.Docs](https://github.com/TraxSharp/Trax.Docs) | Documentation and decision records |
| [Trax.Website](https://github.com/TraxSharp/Trax.Website) | Source for [traxsharp.net](https://traxsharp.net) |

Packages are on [NuGet](https://www.nuget.org/profiles/TraxSharp), versioned per repo by Semantic
Release.

The samples cover deployment shapes: one process, a split API and scheduler, distributed workers
polling a job table, ephemeral runners over HTTP, a real-time chat service, and a multi-schema
library app that consumes every architecture guard.

## Documentation

- [Getting Started](https://traxsharp.net/docs/getting-started)
- [SDK Reference](https://traxsharp.net/docs/sdk-reference)
- [Trax vs Quartz.NET vs Hangfire](https://traxsharp.net/docs/reference/comparison)
- [Benchmarks](https://traxsharp.net/docs/reference/benchmarks) — what a train costs over a plain method call
- [API Security](https://traxsharp.net/docs/api-security), [Supply Chain Security](https://traxsharp.net/docs/supply-chain-security)
- [Migrating from ChainSharp](https://traxsharp.net/docs/reference/migration)

## License

MIT.

## Trademark & Brand Notice

Trax is an open-source .NET framework provided by TraxSharp. This project is an independent
community effort and is not affiliated with, sponsored by, or endorsed by the Utah Transit
Authority, Trax Retail, or any other entity using the "Trax" name in other industries.
