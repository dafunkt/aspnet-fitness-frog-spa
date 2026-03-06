# CLAUDE.md — Fitness Frog SPA

This file provides AI assistant context for the `aspnet-fitness-frog-spa` repository.

## Project Overview

A fitness activity tracking application consisting of:
- **ASP.NET Web API 5** backend exposing a REST API
- **Pre-compiled Angular SPA** frontend (bundles checked into source control)
- **Entity Framework 6** for database access via a shared library

The project is structured as a Treehouse course exercise demonstrating REST API design, dependency injection, and repository patterns on .NET Framework 4.6.1.

---

## Repository Structure

```
aspnet-fitness-frog-spa/
├── README.md
└── src/
    ├── Treehouse.FitnessFrog.Shared/      # Shared class library
    │   ├── Data/
    │   │   ├── BaseRepository.cs          # Generic CRUD repository
    │   │   ├── ActivitiesRepository.cs
    │   │   ├── EntriesRepository.cs
    │   │   ├── Context.cs                 # EF DbContext
    │   │   └── DatabaseInitializer.cs     # Seed data (DropCreateDatabaseIfModelChanges)
    │   └── Models/
    │       ├── Activity.cs
    │       └── Entry.cs                   # Includes IntensityLevel enum
    │
    └── Treehouse.FitnessFrog.Spa/         # ASP.NET Web API project
        ├── App_Start/
        │   ├── WebApiConfig.cs            # Routing + JSON serialization
        │   └── SimpleInjectorWebApiInitializer.cs  # DI container wiring
        ├── Controllers/
        │   ├── EntriesController.cs       # Full CRUD
        │   ├── ActivitiesController.cs    # Read-only
        │   └── IntensitiesController.cs   # Read-only (enum values)
        ├── Dto/
        │   └── EntryDto.cs               # Request/response contract + validation
        ├── index.html                    # Angular SPA entry point
        ├── client-app-config.json        # { "useInMemoryData": false }
        └── Web.config                    # EF + IIS config
```

---

## Technology Stack

| Layer | Technology | Version |
|-------|------------|---------|
| Runtime | .NET Framework | 4.6.1 |
| Web API | ASP.NET Web API | 5.2.3 |
| ORM | Entity Framework | 6.1.3 |
| DI Container | SimpleInjector | 4.0.8 |
| JSON | Newtonsoft.Json | 6.0.4 |
| Frontend | Angular (pre-compiled) | — |
| Database | SQL Server LocalDB | mssqllocaldb |

---

## Domain Models

### `Entry` (core entity)
```csharp
public class Entry {
    public int Id { get; set; }
    public DateTime Date { get; set; }
    public int ActivityId { get; set; }
    public Activity Activity { get; set; }
    public decimal Duration { get; set; }         // precision 5,1 via Fluent API
    public IntensityLevel Intensity { get; set; }
    public bool Exclude { get; set; }
    [MaxLength(200)] public string Notes { get; set; }
}

public enum IntensityLevel { Low = 1, Medium = 2, High = 3 }
```

### `Activity` (lookup entity)
```csharp
public class Activity {
    public int Id { get; set; }
    [MaxLength(100)] public string Name { get; set; }
    [JsonIgnore] public IList<Entry> Entries { get; set; }  // prevents circular refs
}
```

---

## API Endpoints

Base path: `/api/`

| Method | Route | Description | Success Code |
|--------|-------|-------------|--------------|
| GET | `/api/entries` | List all entries | 200 |
| GET | `/api/entries/{id}` | Get single entry | 200 |
| POST | `/api/entries` | Create entry | 201 Created |
| PUT | `/api/entries/{id}` | Update entry | 204 No Content |
| DELETE | `/api/entries/{id}` | Delete entry | 200 |
| GET | `/api/activities` | List all activities | 200 |
| GET | `/api/intensities` | List intensity levels | 200 |

**JSON format:** All property names are **camelCase** (configured via `CamelCasePropertyNamesContractResolver`).

**Validation errors** return `400 Bad Request` with ModelState error details.

---

## Key Conventions

### Repository Pattern
- `BaseRepository<T>` provides `Add`, `Update`, `Delete`, `Get`, `GetList`
- Each operation calls `SaveChanges()` immediately — no explicit Unit of Work
- All repositories are registered with **scoped (per-request) lifetime** via SimpleInjector
- `EntriesRepository.GetList()` eager-loads `Activity` and orders by `Date DESC, Id DESC`
- `ActivitiesRepository.GetList()` orders by `Name`

### DTO Pattern
- `EntryDto` is the only DTO — used for both POST and PUT request bodies
- Contains a `ToModel()` method that converts to the `Entry` domain model
- Nullable properties (`int?`, `decimal?`, `bool?`) allow partial validation
- Required fields: `Date`, `ActivityId`, `Duration`, `Intensity`

### Dependency Injection
- SimpleInjector is initialized via `WebActivator.PostApplicationStartMethod` in `SimpleInjectorWebApiInitializer.cs`
- Registration order: `Context` (scoped) → `EntriesRepository` (scoped) → `ActivitiesRepository` (scoped)
- Controllers receive repositories via constructor injection

### JSON Serialization
- `ReferenceLoopHandling.Ignore` prevents circular reference errors
- `CamelCasePropertyNamesContractResolver` applied globally
- XML formatter is **removed** — API is JSON-only

### Controller Patterns
```csharp
// Standard error returns
return NotFound();          // 404
return BadRequest(ModelState);  // 400 with validation detail
return StatusCode(HttpStatusCode.NoContent);  // 204

// Standard success returns
return Created(locationUri, entry);  // 201 with Location header
return Ok(entries);                  // 200
```

---

## Database Setup

- **Database name:** `FitnessFrog`
- **Connection:** `(localdb)\MSSQLLocalDB` with Integrated Security
- **Initializer:** `DropCreateDatabaseIfModelChanges<Context>` — database is **dropped and recreated** whenever the EF model changes
- **Seed data:** 10 activities (Basketball, Biking, Hiking, Kayaking, Pokemon Go, Running, Skiing, Swimming, Walking, Weight Lifting) and 10 sample entries from July 2017
- **No EF Migrations** — schema is managed by the initializer
- Singular table naming (PluralizingTableNameConvention removed)

---

## Development Workflow

### Prerequisites
- Visual Studio 2015+ or MSBuild
- SQL Server LocalDB (installed with Visual Studio)
- .NET Framework 4.6.1 SDK

### Running Locally
1. Open `aspnet-fitness-frog-spa.sln` (or the `.csproj` files) in Visual Studio
2. Build the solution — NuGet packages restore automatically
3. Run the Spa project via IIS Express (port **51487**)
4. The database is created/seeded automatically on first request
5. Navigate to `http://localhost:51487` for the Angular SPA
6. API is available at `http://localhost:51487/api/`

### Rebuilding the Frontend
The Angular SPA frontend source code is **not present** in this repository — only pre-compiled webpack bundles are committed. To rebuild the frontend you would need a separate Angular CLI project. The hash-named bundles in the Spa project root are the production output.

### No Test Projects
There are no automated tests in this repository. Testing is manual via the browser SPA or an API client (Postman, curl, etc.).

---

## Configuration Files

### `Web.config` (Spa project)
- EF database initializer and connection factory settings
- Assembly binding redirects for Web API, MVC, and related packages
- IIS handler for extensionless URLs (required for Web API routing)
- `<compilation debug="true" targetFramework="4.6.1">` — debug mode on

### `Web.Debug.config` / `Web.Release.config`
- Transform files for environment-specific overrides
- Release transform disables debug compilation

### `client-app-config.json`
```json
{ "useInMemoryData": false }
```
Controls whether the Angular app uses real backend API (`false`) or in-memory mock data (`true`).

---

## Adding New Features

### Adding a New Controller
1. Create `Controllers/XyzController.cs` in the Spa project
2. Inherit from `ApiController`
3. Register any new repositories in `SimpleInjectorWebApiInitializer.cs`
4. Follow existing patterns: inject repository via constructor, use `Ok()`, `NotFound()`, `BadRequest()` return helpers

### Adding a New Model
1. Add the class to `Treehouse.FitnessFrog.Shared/Models/`
2. Add a `DbSet<T>` to `Context.cs`
3. Add Fluent API configuration in `Context.OnModelCreating()` if needed
4. Create a corresponding repository inheriting `BaseRepository<T>` in `Shared/Data/`
5. Update `DatabaseInitializer.cs` seed data
6. The database will be dropped and recreated on next startup (due to `DropCreateDatabaseIfModelChanges`)

### Adding a New Endpoint to an Existing Controller
1. Add the action method with the appropriate HTTP verb attribute (`[HttpGet]`, `[HttpPost]`, etc.)
2. Default route is `api/{controller}/{id}` — add `[Route]` attribute for custom paths
3. Validate `ModelState.IsValid` on write operations before proceeding

---

## Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Database not created | LocalDB not installed | Install SQL Server Express LocalDB |
| 500 on first request | EF model/DB mismatch | The initializer will drop/recreate; restart app |
| Circular reference in JSON | Navigation properties | Add `[JsonIgnore]` to collection nav props |
| DI container error | Repository not registered | Register in `SimpleInjectorWebApiInitializer.cs` |
| 404 on API routes | Extensionless URL handler missing | Ensure Web.config has the `OPTIONSVerbHandler` and `ExtensionlessUrlHandler` entries |
