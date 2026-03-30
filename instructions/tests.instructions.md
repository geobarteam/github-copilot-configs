---
description: "Use when writing, reviewing, or generating unit tests or integration tests. Covers MSTest conventions, AAA structure, Moq usage, integration test factory pattern, and test folder layout for {{SolutionName}}."
applyTo: "src/Test/**"
---
# Test Conventions — {{SolutionName}}

## Frameworks
- **MSTest** + **Moq** for all tests. Test runner: `Microsoft.Testing.Platform`.
- No xUnit, NUnit, or NSubstitute.

## Naming
- File: `<ClassUnderTest>Tests.cs`
- Method: `<Method>_<Scenario>_<Expected>` — e.g. `Execute_WhenDoctorNameDuplicated_ReturnsError`

## Structure (AAA)
```csharp
// Arrange
var mock = new Mock<IDoctorRepository>();
mock.Setup(r => r.ListAllAsync()).ReturnsAsync(new List<Doctor> { ... });

// Act
var result = await _query.Execute();

// Assert
Assert.AreEqual(expected, result);
```

## Unit Tests — `src/Test/Unit/`

| Source layer | Test folder |
|---|---|
| `Core/Application/Functionalities/<Feature>/` | `Test/Unit/Application/` |
| `Core/Domain/` | `Test/Unit/Domain/` |

```csharp
[TestClass]
public sealed class GetDoctorsQueryTests
{
    private Mock<IDoctorRepository>? _mockRepository;
    private GetDoctorsQuery? _query;

    [TestInitialize]
    public void Setup() =>
        _query = new GetDoctorsQuery(_mockRepository = new Mock<IDoctorRepository>());

    [TestMethod]
    public async Task Execute_WhenCalled_ShouldReturnAllDoctorsFromRepository()
    {
        // Arrange
        _mockRepository!.Setup(r => r.ListAllAsync())
            .ReturnsAsync(new List<Doctor> { new() { Id = 1, Name = "Dr. Smith" } });

        // Act
        var result = await _query!.Execute();

        // Assert
        Assert.HasCount(1, result);
        _mockRepository.Verify(r => r.ListAllAsync(), Times.Once);
    }
}
```

- **Mock only at boundary (interfaces)**. No real DB or HTTP calls.
- `[TestInitialize]` for per-test setup; avoid `[ClassInitialize]` in unit tests.
- Every bugfix needs a regression test first (RED step).

## Integration Tests — `src/Test/Integration/`

```
Test/Integration/
  Infrastructure/
    CustomWebApplicationFactory.cs      # bootstraps SQLite + TestAuthenticationHandler
    {{DbContextName}}Extensions.cs   # SeedDatabase()
  Endpoints/
    <Feature>/                          # one folder per controller
  Basic/                                # infrastructure health tests
```

Reference pattern (`GetDoctorsTest.cs`):

```csharp
[TestClass]
public class GetDoctorsTest : IDisposable
{
    private readonly CustomWebApplicationFactory _factory;
    private readonly HttpClient _client;

    public GetDoctorsTest()
    {
        _factory = new CustomWebApplicationFactory();
        _client = _factory.CreateClient();
    }

    [TestMethod]
    public async Task Get_WithExistingDoctors_ReturnsOkWithDoctorList()
    {
        // Arrange
        await _factory.SeedTestDataAsync();

        // Act
        var response = await _client.GetAsync("/api/doctor");

        // Assert
        Assert.AreEqual(HttpStatusCode.OK, response.StatusCode);
        var doctors = await response.Content.ReadFromJsonAsync<IEnumerable<DoctorDto>>();
        Assert.IsNotNull(doctors);
    }

    public void Dispose()
    {
        _client?.Dispose();
        _factory?.Dispose();
    }
}
```

- `CustomWebApplicationFactory` uses SQLite in-memory and `TestAuthenticationHandler` (auth bypassed).
- Seed via `_factory.SeedTestDataAsync()` → calls `{{DbContextName}}Extensions.SeedDatabase()`.
- Auth disabled via `appsettings.UnitTest.json` (`"EnableAuthentication": false`) or environment `Testing`.
- Add service mocks (`IClockService`, `IMessagingService`, etc.) **only when the specific test scenario requires them**.

## Run Commands
```powershell
dotnet test src/Test/Unit/                          # all unit tests
dotnet test src/Test/Integration/                   # all integration tests
dotnet test src/Test/Unit/ --filter GetDoctors      # filtered by class/method
```
