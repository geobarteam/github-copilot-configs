---
description: "Layer guidance for EF Core repositories, DbContext, and entity configurations. Covers BaseRepository pattern, AsNoTracking, IEntityTypeConfiguration, and DbSet registration. Activates when editing Persistence layer files."
applyTo: "src/Core/Persistence/**"
---
# Persistence Layer Conventions

## Repository

```csharp
namespace {{NamespaceRoot}}.Core.Persistence.Repositories;

public class <Entity>Repository({{DbContextName}} dbContext)
    : BaseRepository<<Entity>>(dbContext), I<Entity>Repository
{
    public async Task<IReadOnlyList<<Entity>>> GetBy<Criteria>Async(string value)
    {
        return await _dbContext.<Entity>s
            .Include(e => e.Doctor)
            .Where(e => e.<Property>.ToUpper() == value.ToUpper())
            .AsNoTracking()
            .ToListAsync();
    }
}
```

## Entity Configuration

```csharp
namespace {{NamespaceRoot}}.Core.Persistence.EntityTypeConfigurations;

public class <Entity>Configuration : IEntityTypeConfiguration<<Entity>>
{
    public void Configure(EntityTypeBuilder<<Entity>> builder)
    {
        builder.ToTable("<Entity>", "<Schema>");
        builder.HasKey(e => e.Id);
        builder.Property(e => e.Name).IsRequired().HasMaxLength(200);
        builder.HasOne(e => e.Doctor)
            .WithMany()
            .HasForeignKey(e => e.DoctorId)
            .OnDelete(DeleteBehavior.NoAction);
        builder.Navigation(e => e.Doctor).AutoInclude();
    }
}
```

## DbSet Registration

Add to `{{DbContextName}}`:

```csharp
public DbSet<<Entity>> <Entity>s { get; set; }
```

## Rules

- **`AsNoTracking()` on ALL read queries** — no exceptions.
- **No N+1 patterns** — use `.Include()` or `.AutoInclude()` for navigation properties.
- EF configuration via `IEntityTypeConfiguration<T>` only — never EF attributes on Domain entities.
- Repositories are registered via `PersistenceModule` + Scrutor suffix scanning (`*Repository` → Scoped).
- Every repository needs an `I<Name>Repository` interface (in Application layer) for Scrutor.
- File locations: repos in `Repositories/`, configs in `EntityTypeConfigurations/`.
