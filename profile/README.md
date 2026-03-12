# Trax

Composable pipelines for .NET — build business logic as a typed chain of steps where errors short-circuit automatically. Start with zero infrastructure, then add execution logging, scheduling, and a monitoring dashboard as you need them.

## The Problem

Most service code is a sequence of steps: validate, transform, persist, notify. But the actual logic gets buried under try/catch blocks, null checks, and scattered side effects.

## The Fix

Trax replaces that with a **train** — a typed pipeline of steps where each step's output feeds the next. If any step fails, the rest are skipped automatically. No try/catch required.

```csharp
public class CreateUserTrain : Train<CreateUserRequest, User>
{
    protected override async Task<Either<Exception, User>> RunInternal(CreateUserRequest input)
        => Activate(input)
            .Chain<ValidateEmailStep>()
            .Chain<CreateUserInDatabaseStep>()
            .Chain<SendWelcomeEmailStep>()
            .Resolve();
}
```

A compile-time Roslyn analyzer catches type mismatches between steps before you ever run the code.

## Use Only What You Need

Trax is a stack of independent layers. Each one is a standalone package that builds on the one below it. **You stop at whatever layer solves your problem.**

```
dotnet add package Trax.Core            # Just pipelines — no DI, no database, no infrastructure
dotnet add package Trax.Effect          # + execution logging, DI, pluggable storage
dotnet add package Trax.Mediator        # + decoupled dispatch (callers don't know which train runs)
dotnet add package Trax.Scheduler       # + cron schedules, retries, dead-letter queues
dotnet add package Trax.Dashboard       # + Blazor monitoring UI that mounts into your app
```

### Trax.Core — Type-safe pipelines

You have a sequence of steps and you want them composed with type safety and automatic error propagation. That's it. No database. No DI container. Just `Activate → Chain → Resolve`.

Good for: validation pipelines, data transformations, CLI tools, anywhere you'd write nested try/catch.

```bash
dotnet add package Trax.Core
```

```csharp
var result = await train.Run(input); // Either<Exception, TOutput>
```

### Trax.Effect — Execution logging and DI

Wraps every train run with persistent metadata — state, timing, inputs, outputs, errors. Steps are resolved from your DI container. Pick a storage provider and every execution becomes a queryable record.

Good for: web APIs, services where you need to know what ran and why it failed.

```bash
dotnet add package Trax.Effect
dotnet add package Trax.Effect.Data.Postgres  # or Trax.Effect.Data.InMemory
```

```csharp
builder.Services.AddTrax(trax =>
    trax.AddEffects(effects =>
        effects.UsePostgres(connectionString)
    )
);
```

### Trax.Mediator — Decoupled dispatch

Your controller or parent train shouldn't reference concrete train types. `TrainBus` scans your assemblies, builds a cargo-to-train mapping, and dispatches by input type.

Good for: larger apps where multiple callers trigger trains, or where trains trigger other trains.

```bash
dotnet add package Trax.Mediator
```

```csharp
builder.Services.AddTrax(trax =>
    trax.AddEffects(effects =>
            effects.UsePostgres(connectionString)
        )
        .AddMediator(typeof(Program).Assembly)
);

// In a controller or another train:
var user = await trainBus.Send<CreateUserRequest, User>(request);
```

### Trax.Scheduler — Background job scheduling

Cron-based and interval-based scheduling with retries, dead-letter handling, and dependent jobs. Every scheduled run is a normal train execution — same logging, same visibility, same dashboard.

Good for: recurring jobs, ETL pipelines, nightly reports, periodic cleanup — anywhere you'd reach for Hangfire or Quartz.

```bash
dotnet add package Trax.Scheduler
```

### Trax.Dashboard — Monitoring UI

A Blazor Server dashboard that mounts directly into your existing ASP.NET Core app. No separate deployment. Browse executions, inspect failures, view schedules, toggle effect providers at runtime.

Good for: any app using Trax.Effect that needs operational visibility without building custom admin pages.

```bash
dotnet add package Trax.Dashboard
```

```csharp
builder.Services.AddTraxDashboard();
// ...
app.UseTraxDashboard();
// Dashboard available at /trax
```

## Quick Start

The fastest way to get a full project with scheduling and the dashboard:

```bash
dotnet new install Trax.Samples.Templates
dotnet new trax-server -n MyApp
```

## Packages

| Package | Purpose |
|---------|---------|
| [Trax.Core](https://github.com/TraxSharp/Trax.Core) | Trains, steps, Memory, error propagation, compile-time analyzer |
| [Trax.Effect](https://github.com/TraxSharp/Trax.Effect) | `ServiceTrain`, execution metadata, pluggable effect providers, DI |
| [Trax.Effect.Data.Postgres](https://github.com/TraxSharp/Trax.Effect) | PostgreSQL storage provider |
| [Trax.Effect.Data.InMemory](https://github.com/TraxSharp/Trax.Effect) | In-memory storage provider (dev/testing) |
| [Trax.Mediator](https://github.com/TraxSharp/Trax.Mediator) | `TrainBus` — route inputs to trains by type |
| [Trax.Scheduler](https://github.com/TraxSharp/Trax.Scheduler) | Manifest-based scheduling, retries, dead-letter handling |
| [Trax.Api.GraphQL](https://github.com/TraxSharp/Trax.Api) | GraphQL API layer for train operations |
| [Trax.Dashboard](https://github.com/TraxSharp/Trax.Dashboard) | Blazor Server monitoring UI |
| [Trax.Samples](https://github.com/TraxSharp/Trax.Samples) | Sample apps and `dotnet new trax-server` template |

All packages are on [NuGet](https://www.nuget.org/packages?q=Trax.Core).

## Documentation

Full docs: [traxsharp.github.io/Trax.Docs](https://traxsharp.github.io/Trax.Docs)

## License

MIT
