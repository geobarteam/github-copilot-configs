---
name: add-dbup
description: "Use when adding a new DbUp migration script (CREATE TABLE, ALTER TABLE, seed data, stored procedure, index, etc.) to the Database project. Handles script naming, numbering, and embedded resource configuration. Use for: new tables, schema changes, data migrations, index creation, stored procedures."
argument-hint: "Describe the migration, e.g. 'Create Subscription table' or 'Add Email column to Patient'"
---

# Add DbUp Migration Script

Add a new sequentially numbered SQL migration script to the DbUp Database project. DbUp tracks which scripts have been run via a journal table (`SchemaVersions`) and executes only new scripts in order.

## Prerequisites

- The **Database project** exists at `src/Database/` as a console app referencing `DbUp-SqlServer`.
- Scripts are embedded resources in `src/Database/Scripts/`.
- The `.csproj` includes: `<EmbeddedResource Include="Scripts/**/*.sql" />`.

## Workflow

```
1. DISCOVER — find the highest-numbered existing script in src/Database/Scripts/
2. CREATE — create the new script with the next sequential number
3. VERIFY — dotnet build src/Database/ → confirm compiles
4. 🛑 STOP — present the script for review
```

---

## 1. Discover Next Script Number

Read the `src/Database/Scripts/` folder and find the highest existing number prefix:

```
src/Database/Scripts/
├── 0001_CreateDoctorTable.sql
├── 0002_CreatePatientTable.sql
├── 0003_AddEmailToDoctor.sql
└── ...
```

The next script number = highest existing + 1, zero-padded to 4 digits.

If no scripts exist yet, start at `0001`.

## 2. Script Naming Convention

```
<NNNN>_<DescriptiveName>.sql
```

| Part | Rule | Example |
|------|------|---------|
| `NNNN` | Zero-padded sequential number | `0004` |
| `DescriptiveName` | PascalCase, describes the change | `CreateSubscriptionTable` |

**Naming patterns by change type:**

| Change | Name pattern | Example |
|--------|-------------|---------|
| New table | `Create<Entity>Table` | `0004_CreateSubscriptionTable.sql` |
| Add column | `Add<Column>To<Entity>` | `0005_AddEmailToPatient.sql` |
| Drop column | `Remove<Column>From<Entity>` | `0006_RemovePhoneFromDoctor.sql` |
| Add index | `AddIndex<Entity><Column>` | `0007_AddIndexPatientLastName.sql` |
| Seed data | `Seed<Entity>Data` | `0008_SeedSpecialtyData.sql` |
| Stored procedure | `Create<ProcName>Procedure` | `0009_CreateSearchDoctorsProcedure.sql` |
| Alter table | `Alter<Entity><Description>` | `0010_AlterDoctorExtendNameLength.sql` |

## 3. Script Templates

### CREATE TABLE

```sql
CREATE TABLE [<Schema>].[<Entity>]
(
    [Id]        INT             IDENTITY(1, 1)  NOT NULL,
    [Name]      NVARCHAR(100)                   NOT NULL,
    [CreatedAt] DATETIME2(7)                    NOT NULL    DEFAULT GETUTCDATE(),
    [UpdatedAt] DATETIME2(7)                    NULL,

    CONSTRAINT [PK_<Entity>] PRIMARY KEY CLUSTERED ([Id] ASC)
);
GO
```

### ADD COLUMN

```sql
ALTER TABLE [<Schema>].[<Entity>]
    ADD [<ColumnName>] <DataType> <NULL|NOT NULL> <DEFAULT>;
GO
```

### ADD INDEX

```sql
CREATE NONCLUSTERED INDEX [IX_<Entity>_<Column>]
    ON [<Schema>].[<Entity>] ([<Column>] ASC);
GO
```

### ADD FOREIGN KEY

```sql
ALTER TABLE [<Schema>].[<ChildEntity>]
    ADD CONSTRAINT [FK_<ChildEntity>_<ParentEntity>]
    FOREIGN KEY ([<ParentEntity>Id])
    REFERENCES [<Schema>].[<ParentEntity>] ([Id]);
GO
```

### SEED DATA

```sql
SET IDENTITY_INSERT [<Schema>].[<Entity>] ON;

INSERT INTO [<Schema>].[<Entity>] ([Id], [Name])
VALUES
    (1, 'Value1'),
    (2, 'Value2'),
    (3, 'Value3');

SET IDENTITY_INSERT [<Schema>].[<Entity>] OFF;
GO
```

### STORED PROCEDURE

```sql
CREATE OR ALTER PROCEDURE [<Schema>].[usp_<Name>]
    @Param1 <DataType>,
    @Param2 <DataType>
AS
BEGIN
    SET NOCOUNT ON;

    -- Implementation
    SELECT [Id], [Name]
    FROM [<Schema>].[<Entity>]
    WHERE [Name] LIKE '%' + @Param1 + '%';
END;
GO
```

## 4. Script Rules

- **Idempotency**: Use `IF NOT EXISTS` guards for schema changes when possible:
  ```sql
  IF NOT EXISTS (SELECT 1 FROM sys.tables WHERE name = '<Entity>')
  BEGIN
      CREATE TABLE ...
  END;
  GO
  ```
- **GO statement**: End each logical batch with `GO`.
- **Schema prefix**: Always qualify table names with schema (`[dbo]`, `[<Schema>]`).
- **No data loss**: Never `DROP TABLE` or `DROP COLUMN` without a migration step that preserves data. If a destructive change is needed, split into two scripts: (1) migrate data, (2) drop.
- **Consistent types**: Use `NVARCHAR` for text, `DATETIME2(7)` for timestamps, `BIT` for booleans, `DECIMAL(18,2)` for money.
- **Always NULL-aware**: Explicitly specify `NULL` or `NOT NULL` on every column.

## 5. EF Core Configuration Alignment

After creating a migration script, ensure the EF Core `IEntityTypeConfiguration<T>` in `Core/Persistence/EntityTypeConfigurations/` matches exactly:

- Table name and schema match the SQL script.
- Column types, lengths, and nullability match.
- Primary key and index definitions match.

> See `persistence-layer.instructions.md` for EF configuration patterns.

## 6. DbUp Program.cs Pattern

For reference, the Database project's `Program.cs` follows this pattern:

```csharp
var connectionString = args.FirstOrDefault()
    ?? configuration.GetConnectionString("DefaultConnection");

EnsureDatabase.For.SqlDatabase(connectionString);

var upgrader = DeployChanges.To
    .SqlDatabase(connectionString)
    .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly())
    .WithTransactionPerScript()
    .LogToConsole()
    .Build();

var result = upgrader.PerformUpgrade();
```

Scripts are executed in **alphabetical order** — the `NNNN_` prefix ensures correct sequencing. DbUp records executed scripts in the `SchemaVersions` journal table and skips them on subsequent runs.

---

## Checklist

- [ ] Script number is sequential (no gaps, no duplicates)
- [ ] Script name is descriptive PascalCase
- [ ] Schema-qualified table names (`[dbo].[Entity]`)
- [ ] `GO` terminators after each batch
- [ ] Explicit `NULL`/`NOT NULL` on all columns
- [ ] EF configuration matches SQL schema
- [ ] `dotnet build src/Database/` compiles
- [ ] Script reviewed for data safety (no accidental drops)
