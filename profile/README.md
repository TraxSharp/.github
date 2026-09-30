# Trax

**Business logic you can call, schedule, or serve as an API, with every run recorded in your Postgres.**

MIT licensed, with no commercial edition. .NET 10.
[Docs](https://traxsharp.net/docs) · [Getting started](https://traxsharp.net/docs/getting-started) ·
[Samples](https://github.com/TraxSharp/Trax.Samples) · [NuGet](https://www.nuget.org/profiles/Theauxm)

A **train** is a typed pipeline of small steps called **junctions**. Each junction takes a typed input and returns a
typed output. When one throws, the rest are skipped and the train returns the exception, so the route reads top to
bottom with no error handling between steps.

```csharp
public class RecalculateLeaderboardTrain
    : ServiceTrain<RecalculateLeaderboardInput, RecalculateLeaderboardOutput>,
        IRecalculateLeaderboardTrain
{
    protected override Task<Either<Exception, RecalculateLeaderboardOutput>> Junctions() =>
        Chain<AggregateScoresJunction>()
            .Chain<RankPlayersJunction>()
            .Resolve();
}
```

Each junction's output is stored by type, and the next junction declares what it needs. Because every step is
declared up front, the host checks the whole chain at startup and refuses to start if a junction's input can never be
provided.

## One train, four ways to run it

Nothing in the train knows how it will be run. Adapted from the
[game server sample](https://github.com/TraxSharp/Trax.Samples/tree/main/samples/LocalWorkers):

| | How | Package |
|---|---|---|
| **Call it** | `trains.RunAsync<RecalculateLeaderboardOutput>(new RecalculateLeaderboardInput { Region = "na" })` from a controller or service | `Trax.Mediator` |
| **Schedule it** | `.Schedule<IRecalculateLeaderboardTrain>("leaderboard-na", input, Every.Minutes(5), o => o.MaxRetries(3))`. Failed runs retry with backoff, then land in the dead-letter queue. | `Trax.Scheduler` |
| **Serve it** | `[TraxMutation(GraphQLOperation.Queue)]` and `[TraxAuthorize(Roles = "Admin")]` on the class make it a typed GraphQL mutation. An exposed train that does not say who may call it stops the host from starting. | `Trax.Api.GraphQL` |
| **Move it** | `AddTraxWorker()` on other machines polls Postgres and claims jobs one worker at a time, with no broker. `UseLambdaWorkers(...)` hands them to AWS Lambda instead. | `Trax.Scheduler` |

## Every run leaves a record

API calls, background jobs and cron schedules go through the same pipeline, and each run gets a row in your database:

```text
trax.metadata  id 88213
TrainState        Failed
StartTime         2026-09-29 03:15:00.412
EndTime           2026-09-29 03:15:02.087
Input             { "region": "na" }
FailureJunction   RankPlayersJunction
FailureException  TimeoutException
FailureReason     Score service did not answer within 2s
ManifestId        14  (leaderboard-na)
HostName          worker-2
```

When a job fails overnight, you open the run and read what happened. [Trax.Dashboard](https://github.com/TraxSharp/Trax.Dashboard)
shows the same rows, and so does a SQL query. [What gets recorded](https://traxsharp.net/docs/effect/metadata).

## Start

```bash
dotnet new install Trax.Samples.Templates
dotnet new trax-scheduler -n MyApp    # scheduler and dashboard
dotnet new trax-api -n MyApp.Api      # GraphQL API
dotnet new trax-hub -n MyApp.Hub      # both in one process
```

Or `dotnet add package Trax.Core` and write a train. [Getting started](https://traxsharp.net/docs/getting-started)
takes one example from Core alone to the full stack.

## Repositories

Each code repo is one layer and depends only on repos above it in this table. Take the layers you need; the trains you already wrote
do not change.

| Repo | What it adds |
|---|---|
| [Trax.Core](https://github.com/TraxSharp/Trax.Core) | Trains, junctions and the chain, with no database and no DI container |
| [Trax.Effect](https://github.com/TraxSharp/Trax.Effect) | A recorded run for every execution (Postgres, SQLite or in memory), DI, effect providers, the state-machine engine |
| [Trax.Mediator](https://github.com/TraxSharp/Trax.Mediator) | The train bus: run a train by handing over its input, with every chain checked at startup |
| [Trax.Scheduler](https://github.com/TraxSharp/Trax.Scheduler) | Cron and interval schedules, retries, dead letters, and workers on other machines or in Lambda |
| [Trax.Api](https://github.com/TraxSharp/Trax.Api) | GraphQL generated from your trains, with authentication, audit and typed clients |
| [Trax.Dashboard](https://github.com/TraxSharp/Trax.Dashboard) | A Blazor Server UI for runs, schedules and dead letters, mounted in your app |
| [Trax.Cli](https://github.com/TraxSharp/Trax.Cli) | The `trax` tool: scaffold a hub and trains from an OpenAPI or GraphQL schema, and state-machine codegen |
| [Trax.Samples](https://github.com/TraxSharp/Trax.Samples) | **Start here.** Complete sample apps, and the `trax-api`, `trax-scheduler` and `trax-hub` templates |
| [Trax.Docs](https://github.com/TraxSharp/Trax.Docs) | The documentation at [traxsharp.net/docs](https://traxsharp.net/docs), and the decision records behind cross-repo rules |
| [Trax.Website](https://github.com/TraxSharp/Trax.Website) | Source for [traxsharp.net](https://traxsharp.net) |

## Where Trax stops

What it does not do today, so you can decide before you install it.

- **A crash restarts the train.** If a process dies halfway through a run, the run is marked failed and a scheduled
  train is retried from its first junction. Junctions that call other systems should be safe to repeat. Resuming at
  the step that failed is being built.
- **Postgres in production.** The scheduler coordinates workers with Postgres row locks and advisory locks. SQLite
  works for a single process and in-memory storage for tests. There is no SQL Server or MySQL provider.
- **Not a message bus.** Trax runs work and records it. It does not carry messages between services through a broker
  or coordinate sagas across them.
- **One language, one database.** Trains are C#, and one Postgres database coordinates them.
- **No tracing export yet.** Runs are recorded in the database and logged through `ILogger`, not emitted as
  OpenTelemetry spans.

## Documentation

- [Getting started](https://traxsharp.net/docs/getting-started) and the [templates](https://traxsharp.net/docs/reference/templates)
- [SDK reference](https://traxsharp.net/docs/sdk-reference)
- [Running trains on other machines](https://traxsharp.net/docs/scheduler/remote-execution)
- [How Trax compares](https://traxsharp.net/docs/reference/comparison) and [what a train costs](https://traxsharp.net/docs/reference/benchmarks)
- [API security](https://traxsharp.net/docs/api-security) and [supply-chain security](https://traxsharp.net/docs/supply-chain-security)

## License

MIT, in every repo. There is no commercial edition, and there will not be one.

Trax is an independent open-source project and is not affiliated with the Utah Transit Authority, Trax Retail, or any
other organization using the Trax name.
