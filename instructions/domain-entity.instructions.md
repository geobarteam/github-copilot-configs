---
description: "Layer guidance for Domain entities. Covers entity conventions, IEntity interface, navigation properties, and forbidden patterns. Activates when editing Domain layer files."
applyTo: "src/Core/Domain/**"
---
# Domain Entity Conventions

## Entity Pattern

```csharp
namespace {{NamespaceRoot}}.Core.Domain.Functionalities.<Feature>;

public class <Entity> : IEntity
{
    public int Id { get; set; }

    // Required scalar properties
    public string Name { get; set; }

    // Foreign keys
    public int DoctorId { get; set; }

    // Navigation properties (configured in Persistence, not here)
    public Doctor Doctor { get; set; }
}
```

## Rules

- Implement `IEntity` (provides `int Id`).
- Plain C# — **no EF attributes**, no `[Required]`, no `[MaxLength]`. All EF configuration goes in `IEntityTypeConfiguration<T>` in Persistence.
- **No dependencies** — Domain references nothing. No NuGet packages, no project references.
- Navigation properties are declared here but configured (FK, AutoInclude) in Persistence.
- One entity per file. File location: `src/Core/Domain/Functionalities/<Feature>/<Entity>.cs`.
- Use `_camelCase` for private fields (SA1309 is disabled).

## Forbidden

- EF attributes on entities.
- Business logic that depends on infrastructure.
- References to Application, Persistence, Infrastructure, or any Host project.
