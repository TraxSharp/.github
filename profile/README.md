# Trax

**Business logic you can call, schedule, or serve as an API, with every run recorded in your Postgres.**

MIT licensed, with no commercial edition. .NET 10.
[Docs](https://traxsharp.net/docs) · [Getting started](https://traxsharp.net/docs/getting-started) ·
[Repository](https://github.com/TraxSharp/Trax) · [Samples](https://github.com/TraxSharp/Trax/tree/main/Trax.Samples) · [NuGet](https://www.nuget.org/profiles/Theauxm)

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

Nothing in the train knows how it will be run.

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

When a job fails overnight, you open the run and read what happened. [Trax.Dashboard](https://github.com/TraxSharp/Trax/tree/main/Trax.Dashboard)
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

## The repository

All of Trax lives in one repository, [TraxSharp/Trax](https://github.com/TraxSharp/Trax), one folder per layer. Every
package releases at one version from it. Each .NET folder depends only on folders above it in this table; take the
layers you need, and the trains you already wrote do not change.

| Folder | What it adds |
|---|---|
| [Trax.Core](https://github.com/TraxSharp/Trax/tree/main/Trax.Core) | Trains, junctions and the chain, with no database and no DI container |
| [Trax.Effect](https://github.com/TraxSharp/Trax/tree/main/Trax.Effect) | A recorded run for every execution (Postgres, SQLite or in memory), DI, effect providers, the state-machine engine |
| [Trax.Mediator](https://github.com/TraxSharp/Trax/tree/main/Trax.Mediator) | The train bus: run a train by handing over its input, with every chain checked at startup |
| [Trax.Scheduler](https://github.com/TraxSharp/Trax/tree/main/Trax.Scheduler) | Cron and interval schedules, retries, dead letters, and workers on other machines or in Lambda |
| [Trax.Api](https://github.com/TraxSharp/Trax/tree/main/Trax.Api) | GraphQL generated from your trains, with authentication, audit and typed clients |
| [Trax.Dashboard](https://github.com/TraxSharp/Trax/tree/main/Trax.Dashboard) | A Blazor Server UI for runs, schedules and dead letters, mounted in your app |
| [Trax.Cli](https://github.com/TraxSharp/Trax/tree/main/Trax.Cli) | The `trax` tool: scaffold a hub and trains from an OpenAPI or GraphQL schema, and state-machine codegen |
| [Trax.Samples](https://github.com/TraxSharp/Trax/tree/main/Trax.Samples) | **Start here.** Complete sample apps, and the `trax-api`, `trax-scheduler` and `trax-hub` templates |
| [Trax.Api.StateMachine](https://github.com/TraxSharp/Trax/tree/main/Trax.Api.StateMachine) | `@trax/state-machine`, the TypeScript twin of the state-machine engine |
| [Trax.Docs](https://github.com/TraxSharp/Trax/tree/main/Trax.Docs) | The documentation at [traxsharp.net/docs](https://traxsharp.net/docs), and the decision records |
| [Trax.Website](https://github.com/TraxSharp/Trax/tree/main/Trax.Website) | Source for [traxsharp.net](https://traxsharp.net) |

Trax used to be split into one repository per layer (`TraxSharp/Trax.Core` and the rest). Those are archived; their
history is in TraxSharp/Trax, under each folder.

## License

MIT. There is no commercial edition, and there will not be one.

Trax is an independent open-source project and is not affiliated with the Utah Transit Authority, Trax Retail, or any
other organization using the Trax name.
