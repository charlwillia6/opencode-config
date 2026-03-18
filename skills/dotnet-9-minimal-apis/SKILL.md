---
name: dotnet-9-minimal-apis
description: Modern .NET 9 development with minimal APIs, C# 13/14 features, and Windows/desktop integration patterns.
---

# .NET 9 Minimal APIs + C# 13/14 Patterns (2026)

## Project Structure

```
src/
├── MyApp.API/
│   ├── Program.cs
│   ├── Properties/
│   ├── appsettings.json
│   └── MyApp.API.csproj
├── MyApp.Application/
│   ├── Services/
│   ├── DTOs/
│   └── MyApp.Application.csproj
├── MyApp.Domain/
│   ├── Entities/
│   ├── Interfaces/
│   └── MyApp.Domain.csproj
├── MyApp.Infrastructure/
│   ├── Data/
│   ├── Repositories/
│   └── MyApp.Infrastructure.csproj
└── tests/
    └── MyApp.API.Tests/
```

## Minimal API Setup (.NET 9)

### Program.cs
```csharp
// MyApp.API/Program.cs

var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddOpenApi();
builder.Services.AddAuthenticationOptions();
builder.Services.AddDatabaseOptions(builder.Configuration);
builder.Services.AddApplicationServices();

var app = builder.Build();

// Configure the HTTP request pipeline
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();

app.UseAuthentication();
app.UseAuthorization();

// Feature modules
app.MapDevicesApi();
app.MapAuthApi();
app.MapAnalyticsApi();

app.Run();
```

### Feature Module Pattern
```csharp
// MyApp.API/Endpoints/DevicesEndpoint.cs
namespace MyApp.API.Endpoints;

public static class DevicesEndpoint
{
    public static RouteGroupBuilder MapDevicesApi(this WebApplication app)
    {
        var group = app.MapGroup("/api/devices");

        group.MapGet("/", async (IDeviceService deviceService) =>
        {
            var devices = await deviceService.GetAllAsync();
            return Results.Ok(devices);
        })
        .WithSummary("Get all devices")
        .WithDescription("Retrieves a list of all registered devices");

        group.MapGet("/{id}", async (Guid id, IDeviceService deviceService) =>
        {
            var device = await deviceService.GetByIdAsync(id);
            return device is not null ? Results.Ok(device) : Results.NotFound();
        })
        .WithName("GetDevice");

        group.MapPost("/", async (CreateDeviceRequest request, IDeviceService deviceService) =>
        {
            var device = await deviceService.CreateAsync(request);
            return Results.Created($"/api/devices/{device.Id}", device);
        })
        .ValidateInput();

        group.MapPut("/{id}", async (Guid id, UpdateDeviceRequest request, IDeviceService deviceService) =>
        {
            var updated = await deviceService.UpdateAsync(id, request);
            return updated ? Results.Ok() : Results.NotFound();
        });

        group.MapDelete("/{id}", async (Guid id, IDeviceService deviceService) =>
        {
            var deleted = await deviceService.DeleteAsync(id);
            return deleted ? Results.NoContent() : Results.NotFound();
        });

        return group;
    }
}
```

## C# 13/14 Features

### File-Local Types
```csharp
// C# 13 - File-Local Types
file class DeviceValidator
{
    public static bool IsValid(Device device)
    {
        return !string.IsNullOrWhiteSpace(device.Name) && device.Id != Guid.Empty;
    }
}
```

### Default Lambda Parameters
```csharp
// C# 14 - Default Lambda Parameters
Func<string, int, string> formatString = (str, length = 20) => str.PadRight(length);

var result1 = formatString("Hello");      // "Hello               "
var result2 = formatString("World", 10);  // "World     "
```

### Generic Attributes
```csharp
// C# 14 - Generic Attributes
[AttributeUsage(AttributeTargets.Property, AllowMultiple = false)]
public class MaxLengthAttribute<T> : Attribute where T : IComparable
{
    public T Value { get; }
    public MaxLengthAttribute(T value) => Value = value;
}

public class User
{
    [MaxLength<string>("50")]
    public string Name { get; set; } = string.Empty;
}
```

### Ref Readonly Parameters
```csharp
// C# 14 - Ref Readonly
public readonly struct ReadOnlyData
{
    private readonly int[] _data;
    
    public readonly ref readonly int[] GetData() => ref _data;
}
```

## Entity Framework Core 9

### DbContext Configuration
```csharp
// MyApp.Infrastructure/Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;
using MyApp.Domain.Entities;

namespace MyApp.Infrastructure.Data;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Device> Devices => Set<Device>();
    public DbSet<User> Users => Set<User>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<Device>()
            .HasIndex(d => d.SerialNumber)
            .IsUnique();

        modelBuilder.Entity<Device>()
            .Property(d => d.Status)
            .HasConversion<string>();

        modelBuilder.Entity<User>()
            .HasIndex(u => u.Email)
            .IsUnique();

        // Seed data
        modelBuilder.Entity<Device>().HasData(
            new Device { Id = Guid.NewGuid(), Name = "Sample Device", Status = DeviceStatus.Online, CreatedAt = DateTime.UtcNow }
        );
    }
}
```

### Repository Pattern
```csharp
// MyApp.Infrastructure/Repositories/DeviceRepository.cs
using Microsoft.EntityFrameworkCore;
using MyApp.Domain.Interfaces;
using MyApp.Domain.Entities;
using MyApp.Infrastructure.Data;

namespace MyApp.Infrastructure.Repositories;

public class DeviceRepository : IDeviceRepository
{
    private readonly AppDbContext _context;
    public DeviceRepository(AppDbContext context)
    {
        _context = context;
    }

    public async Task<IEnumerable<Device>> GetAllAsync()
    {
        return await _context.Devices.ToListAsync();
    }

    public async Task<Device?> GetByIdAsync(Guid id)
    {
        return await _context.Devices.FindAsync(id);
    }

    public async Task<Device> CreateAsync(Device device)
    {
        _context.Devices.Add(device);
        await _context.SaveChangesAsync();
        return device;
    }

    public async Task<bool> UpdateAsync(Guid id, Device device)
    {
        var existing = await _context.Devices.FindAsync(id);
        if (existing is null) return false;

        existing.Name = device.Name;
        existing.Status = device.Status;

        await _context.SaveChangesAsync();
        return true;
    }

    public async Task<bool> DeleteAsync(Guid id)
    {
        var device = await _context.Devices.FindAsync(id);
        if (device is null) return false;

        _context.Devices.Remove(device);
        await _context.SaveChangesAsync();
        return true;
    }
}
```

## Windows Desktop Integration

### Tray Icon (Windows)
```csharp
// Windows only - MyApp.API/Services/TrayService.cs
#if WINDOWS
using Windows.ApplicationModel;

public static class TrayService
{
    public static async Task InitializeAsync()
    {
        var tray = TrayManager.Current;
        
        var menu = new TrayMenu();
        menu.Items.Add(new TrayItem("Show Window", async () =>
        {
            MainWindow.Activate();
        }));
        
        menu.Items.Add(new TraySeparator());
        
        menu.Items.Add(new TrayItem("Exit", async () =>
        {
            await ExitAsync();
        }));
        
        tray.Menu = menu;
        tray.IsVisible = true;
    }

    private static async Task ExitAsync()
    {
        Environment.Exit(0);
    }
}
#endif
```

### Window Management (Windows)
```csharp
// Windows only
#if WINDOWS
using Microsoft.UI.Xaml.Controls;
using Windows.Storage;

public static class WindowService
{
    public static void InitializeMainWindow()
    {
        MainWindow = new MainWindow();
        MainWindow.Activate();
        
        MainWindow.Closed += async (sender, args) =>
        {
            args.Cancel = true;
            await HideAsync();
        };
    }

    private static async Task HideAsync()
    {
        MainWindow.Hide();
    }
}
#endif
```

## Configuration & Options

### AppSettings
```json
// appsettings.Development.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=myapp_dev;Trusted_Connection=True;MultipleActiveResultSets=true"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft": "Information"
    }
  },
  "DeviceSettings": {
    "MaxDevices": 100,
    "AutoConnectTimeout": 30000
  }
}
```

### Options Pattern
```csharp
// MyApp.Application/Options/DeviceOptions.cs
namespace MyApp.Application.Options;

public class DeviceOptions
{
    public int MaxDevices { get; set; }
    public int AutoConnectTimeout { get; set; }
}

// Program.cs
builder.Services.Configure<DeviceOptions>(
    builder.Configuration.GetSection("DeviceSettings"));

// Service using options
public class DeviceService : IDeviceService
{
    private readonly DeviceOptions _options;
    
    public DeviceService(IOptions<DeviceOptions> options)
    {
        _options = options.Value;
    }
    
    public async Task<IEnumerable<Device>> GetAllAsync()
    {
        // Use _options.MaxDevices for limiting results
        // Use _options.AutoConnectTimeout for connection logic
        return await _deviceRepository.GetAllAsync();
    }
}
```

## Validation

### FluentValidation
```csharp
// MyApp.Application/Validation/CreateDeviceValidator.cs
using FluentValidation;

public class CreateDeviceValidator : AbstractValidator<CreateDeviceRequest>
{
    public CreateDeviceValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .WithMessage("Device name is required.")
            .MaximumLength(100)
            .WithMessage("Device name must not exceed 100 characters.");

        RuleFor(x => x.SerialNumber)
            .NotEmpty()
            .WithMessage("Serial number is required.")
            .Matches(@"^[A-Z0-9]+$")
            .WithMessage("Serial number must contain only uppercase letters and numbers.");

        RuleFor(x => x.Location)
            .MaximumLength(255)
            .When(x => !string.IsNullOrEmpty(x.Location));
    }
}

// Validation attribute
public static class ValidatorExtensions
{
    public static IEndpointConventionBuilder ValidateInput(this IEndpointConventionBuilder builder)
    {
        return builder.Add.endpointFilter<ValidationFilter>();
    }
}

public class ValidationFilter : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var validators = context.HttpContext.RequestServices.GetServices<IValidator>();
        var argument = context.Arguments.FirstOrDefault();
        
        if (argument is not null)
        {
            foreach (var validator in validators)
            {
                var result = await validator.ValidateAsync(argument);
                if (!result.IsValid)
                {
                    return Results.BadRequest(result.Errors);
                }
            }
        }
        
        return await next(context);
    }
}
```

## Common Patterns & Anti-Patterns

### Do
✅ Use Minimal APIs for lightweight services  
✅ Leverage C# 13/14 features for cleaner code  
✅ Use IOptions for configuration  
✅ Employ FluentValidation for input validation  
✅ Use dependency injection throughout  

### Don't
❌ Mix concerns in endpoint handlers  
❌ Use static classes for services  
❌ Skip validation in API endpoints  
❌ Hardcode connection strings in code  
❌ Ignore proper error handling  

## References

* [Microsoft .NET 9 Documentation](https://docs.microsoft.com/dotnet)
* [Entity Framework Core 9](https://docs.microsoft.com/ef/core)
* [FluentValidation](https://fluentvalidation.net)
* [C# 13/14 Features](https://docs.microsoft.com/dotnet/csharp/whats-new)

## 2026 Best Practices

* **Minimal APIs**: Default for new .NET services
* **C# 13/14**: Use modern language features
* **EF Core 9**: Latest ORM with better performance
* **Source Generators**: For compile-time code generation
* **HTTP Client Factory**: For typed HTTP clients
