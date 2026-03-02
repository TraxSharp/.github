# Trax

Railway Oriented Programming for .NET — build business logic as trains that carry data through typed stops, with automatic error handling, persistent execution logs, scheduling, and a monitoring dashboard.

## The Idea

Most service code is a sequence of steps: validate, transform, persist, notify. But the actual logic gets buried under error handling, null checks, and scattered side effects. Trax replaces that with a train — a typed chain of steps where errors propagate automatically and side effects are managed atomically.

```csharp
public class CreateUserTrain : ServiceTrain<CreateUserRequest, User>, ICreateUserTrain
{
    protected override async Task<Either<Exception, User>> RunInternal(CreateUserRequest input)
        => Activate(input)
            .Chain<ValidateEmailStep>()
            .Chain<CreateUserInDatabaseStep>()
            .Chain<SendWelcomeEmailStep>()
            .Resolve();
}
```

If any step fails, the rest are skipped. Every execution is logged. Side effects are applied atomically or not at all. A compile-time analyzer catches broken chains before you run anything.

## Packages

| Package | What it does |
|---------|-------------|
| [Trax.Core](https://github.com/TraxSharp/Trax.Core) | The locomotive — trains, steps, Memory, railway error propagation, and the compile-time analyzer |
| [Trax.Effect](https://github.com/TraxSharp/Trax.Effect) | Full commercial service — `ServiceTrain` with execution metadata, pluggable effect providers, and DI |
| [Trax.Mediator](https://github.com/TraxSharp/Trax.Mediator) | Dispatch station — `TrainBus` routes inputs to the right train without coupling |
| [Trax.Scheduler](https://github.com/TraxSharp/Trax.Scheduler) | Timetable — manifest-based scheduling with retries, dead-letter handling, and job dependencies |
| [Trax.Dashboard](https://github.com/TraxSharp/Trax.Dashboard) | Control room — Blazor Server UI for monitoring trains, execution history, manifests, and dead letters |
| [Trax.Samples](https://github.com/TraxSharp/Trax.Samples) | Sample apps and a `dotnet new trax-server` project template |

All packages are on [NuGet](https://www.nuget.org/packages?q=Trax.Core).

## Quick Start

```bash
dotnet add package Trax.Core
dotnet add package Trax.Effect
dotnet add package Trax.Effect.Data.Postgres
```

```csharp
builder.Services.AddTraxEffects(options => options
    .AddPostgresEffect(connectionString)
    .AddServiceTrainBus(typeof(Program).Assembly)
);
```

Or scaffold a complete project with scheduling and the dashboard:

```bash
dotnet new install Trax.Samples.Templates
dotnet new trax-server -n MyApp
```

## How It Fits Together

```
Trax.Core (the locomotive)
  └── Trax.Effect (execution metadata + effect providers)
        ├── Trax.Mediator (TrainBus dispatch)
        ├── Trax.Scheduler (manifest-based timetables)
        └── Trax.Dashboard (monitoring UI)
```

Each layer is optional. Use Core alone for lightweight step chains. Add Effect for persistence and observability. Add Mediator for decoupled dispatch. Add Scheduler for recurring jobs. Add Dashboard to see it all.

## Documentation

Full docs: [traxsharp.github.io/Trax.Docs](https://traxsharp.github.io/Trax.Docs)

## License

MIT
