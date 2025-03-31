# Hybrid Caching in .NET / ASP.NET Core

Hybrid caching in .NET combines two types of cache storage, introducing a two-level caching system:

-   **Local in-memory caching (L1)** as the primary storage
-   **Distributed caching (L2)** as the secondary storage
    
This approach ensures fast in-memory access while maintaining persistence and scalability through external storage.

----------

## Adding and Configuring Hybrid Caching in .NET

To integrate hybrid caching into a .NET application, install the `Microsoft.Extensions.Caching.Hybrid` NuGet package:

```sh
 dotnet add package Microsoft.Extensions.Caching.Hybrid
```

Next, add the HybridCache service using `AddHybridCache()` in the service collection:

```csharp
builder.Services.AddHybridCache();
```

For additional configuration, use the overload method:

```csharp
builder.Services.AddHybridCache(options =>
{
    options.MaximumPayloadBytes = 1024 * 10 * 10; // 10MB
    options.MaximumKeyLength = 256;
    options.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(30),
        LocalCacheExpiration = TimeSpan.FromMinutes(30)
    };
    options.ReportTagMetrics = true;
    options.DisableCompression = true;
});

```



HybridCache automatically detects and uses any configured distributed cache as L2 storage. It natively supports `string` and `byte[]` serialization via `System.Text.Json`, but developers can integrate custom serializers using `AddSerializer<T>()` or `AddSerializerFactory()`.

----------

## Using Hybrid Caching in ASP.NET Core

The HybridCache API simplifies cache operations such as retrieval, storage, and invalidation. Consider the following `Course` class:

```csharp
public class Course
{
    public int Id { get; set; }
    public required string Name { get; set; }
    public required string Category { get; set; }
}

```

### Service Interface

Define an interface for course retrieval, creation, and cache invalidation:

```csharp
public interface CourseService
{
    Task<Course?> GetCourseAsync(int id, CancellationToken cancellationToken = default);
    Task PostCourseAsync(Course course, CancellationToken cancellationToken = default);
    Task InvalidateByCourseIdAsync(int id, CancellationToken cancellationToken = default);
    Task InvalidateByCategoryAsync(string tag, CancellationToken cancellationToken = default);
}

```

### Implementing the Service

Inject HybridCache into the implementation class:

```csharp
public class CourseService(HybridCache cache) : ICourseService
{
    public static readonly List<Course> courseList = [
        new Course { Id = 1, Name = "WebAPI", Category = "Backend" },
        new Course { Id = 2, Name = "Microservices", Category = "Backend" },
        new Course { Id = 3, Name = "Blazor", Category = "Frontend" },
    ];
}

```

----------

## Cache Operations

### Retrieving Cached Data

The `GetOrCreateAsync()` method first checks the cache. If the data is missing, it fetches from the source and stores it:

```csharp
public async Task<Course?> GetCourseAsync(int id, CancellationToken cancellationToken = default)
{
    return await cache.GetOrCreateAsync(
        $"course-{id}", async token =>
        {
            await Task.Delay(1000, token);
            return courseList.FirstOrDefault(course => course.Id == id);
        },
        options: new HybridCacheEntryOptions
        {
            Expiration = TimeSpan.FromMinutes(30),
            LocalCacheExpiration = TimeSpan.FromMinutes(30)
        },
        tags: ["course"],
        cancellationToken: cancellationToken
    );
}

```

This prevents cache stampede by ensuring that concurrent requests wait for the first one to complete.

### Writing Data to Cache

To store an entry without reading it first, use `SetAsync()`:

```csharp
public async Task PostCourseAsync(Course course, CancellationToken cancellationToken = default)
{
    courseList.Add(course);
    await cache.SetAsync($"course-{course.Id}", course,
        options: new HybridCacheEntryOptions
        {
            Expiration = TimeSpan.FromMinutes(30),
            LocalCacheExpiration = TimeSpan.FromMinutes(30)
        },
        tags: [$"cat-{course.Category}"],
        cancellationToken: cancellationToken);
}

```

If a key already exists, its value is overwritten.

### Invalidating Cache Entries

#### By Key

```csharp
public async Task InvalidateByCourseIdAsync(int id, CancellationToken cancellationToken = default)
{
    await cache.RemoveAsync($"course-{id}", cancellationToken);
}

```

#### By Tag

To remove multiple entries sharing the same tag:

```csharp
public async Task InvalidateByCategoryAsync(string tag, CancellationToken cancellationToken = default)
{
    await cache.RemoveByTagAsync($"cat-{tag}", cancellationToken);
}

```

----------

