# GEM: Gym Equipment Management

Gym members scan a QR code on a piece of equipment and report it broken. The gym manager sees and works those reports on a dashboard.

## MVP scope

A manager can add equipment and print QR labels. A member can scan a label and submit a repair request with an optional photo. The manager can log in, see open requests, and mark them resolved.

Anything beyond this (maintenance schedules, analytics, multi-location, staff roles) is out of scope until the MVP ships. Don't add it unprompted.

Work is tracked as GitHub issues grouped into milestones M1–M6. Check the issue you're working on before starting.

## Stack

- **.NET 10 (LTS)**, pinned in `global.json`. Don't use .NET 11 or 11.x NuGet packages.
- **Blazor Web App** (ASP.NET Core) in `src/GymEquipment`
- **PostgreSQL 17** via Docker Compose for local dev
- **EF Core** with `Npgsql.EntityFrameworkCore.PostgreSQL`
- **ASP.NET Core Identity** for manager login (M5)
- **QRCoder** for QR codes, **QuestPDF** for label sheets (M3)
- **MudBlazor or Radzen** for dashboard UI components (picked in M5)
- **Terraform + Azure** for deployment, in `infra/` (M2)

## Architecture decisions

- **Single project.** Keep everything in `src/GymEquipment` until there's a clear reason to split. No layered/onion architecture, repositories, or MediatR.
- **Render modes:** static server-side rendering is the default. Use `@rendermode InteractiveServer` only on pages that need interactivity (the manager dashboard, forms with live behavior). The public report page must stay static SSR so it loads fast on poor gym Wi-Fi and needs no persistent connection.
- **No JavaScript frameworks.** Avoid custom JS unless there's no Blazor way to do it.
- **DbContext:** register with `AddDbContextFactory<AppDbContext>` and inject `IDbContextFactory<AppDbContext>` into components. Create a short-lived context per operation with `await using`. Never inject `AppDbContext` directly into a component.
- **IDs:** entities use `Guid` primary keys. The equipment Guid goes in QR URLs (`/report/{id}`) so they can't be guessed.
- **Time:** always `DateTime.UtcNow`. Npgsql rejects non-UTC values for `timestamptz`.
- **Enums** are stored as strings (`HasConversion<string>()`) so the database stays readable.
- **Migrations** are generated with `dotnet ef` and committed. Never edit an applied migration; add a new one.

## Layout

```
global.json
docker-compose.yml
src/GymEquipment/
  Data/              entities, AppDbContext
  Components/Pages/  routable pages
  Components/Layout/ layout and nav
  Migrations/
infra/               Terraform (M2)
.github/workflows/   CI
```

## Commands

The developer works on **Windows with PowerShell** and Docker Desktop (WSL 2). Give commands that work in PowerShell.

```powershell
docker compose up -d                               # start Postgres
dotnet watch --project src/GymEquipment            # run with hot reload
dotnet build src/GymEquipment                      # build
dotnet ef migrations add <Name> --project src/GymEquipment
dotnet ef database update --project src/GymEquipment
```

Local connection string lives in `appsettings.Development.json` (local Docker credentials only). Never commit real secrets. Use user secrets or environment variables for anything else.

## Workflow

- **One issue per pull request.** Branch from `main` as `issue-<number>-<short-name>`.
- Run `dotnet build` (and tests, once they exist) before opening a PR. CI must pass.
- PR descriptions say what changed and include `Closes #<number>`.
- Keep PRs small and focused. Note any follow-up work as a suggested new issue rather than expanding scope.

## The developer is learning C# and Blazor

The developer has about 5 years of backend experience in Go and PostgreSQL but is new to C#, .NET, and frontend work. They review every PR to learn.

- Prefer clear, idiomatic code over clever code.
- In each PR description, add a short **"New concepts"** section explaining any .NET or Blazor patterns introduced (for example, dependency injection, component lifecycle, render modes, EF migrations). Comparisons to Go are welcome where they help.
