

## 🔐 **Identity & Security in .NET Applications**

### Q1: What is "Identity" in the context of Windows-based .NET applications, and why is it important?

**Answer:**
In Windows-based systems, **every process runs as a specific Windows user account**. This identity determines:

- What **permissions** the process has
- What **network resources** it can access  
- How it **authenticates to remote systems** like SQL Server or MSDTC

**Key deployment scenarios:**

1. **ASP.NET/Web API (IIS-hosted)**: Uses Application Pool Identity
   - Configurable in IIS Manager → Application Pools → Advanced Settings → Identity
   - Options: ApplicationPoolIdentity (default), NetworkService, LocalSystem, Custom account

2. **Console Apps**: Run as the current Windows user (`whoami`)

3. **Windows Services**: Use assigned service account (LocalSystem, NetworkService, or custom domain account)

**Real-world impact**: When using `TransactionScope` with distributed transactions, the same identity is used for MSDTC coordination across systems.

---

### Q2: How do you handle authentication in modern .NET applications with cloud services?

**Answer:**
Modern .NET applications use multiple authentication patterns:

**Traditional:**
```csharp
// Windows Integrated Authentication
var connection = new SqlConnection("Data Source=DB;Integrated Security=SSPI");
```

**Modern Cloud (AWS/Azure):**
```csharp
// JWT Token-based authentication
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(key),
            ValidateIssuer = false,
            ValidateAudience = false
        };
    });

// AWS IAM roles for service-to-service
var credentials = new InstanceProfileAWSCredentials();
var s3Client = new AmazonS3Client(credentials);
```

**Best practices for 2025:**
- Use managed identities (Azure) or IAM roles (AWS) for service authentication
- Implement OAuth 2.0/OpenID Connect for user authentication
- Use certificate-based authentication for high-security scenarios

---

## 🏗️ **Modern .NET Architecture & Patterns**

### Q3: Explain the difference between .NET Framework, .NET Core, and .NET 5+. Which should you choose in 2025?

**Answer:**
- **.NET Framework (Legacy)**: Windows-only, full framework with extensive libraries
- **.NET Core**: Cross-platform, lightweight, cloud-first (discontinued, evolved into .NET 5+)
- **.NET 5+ (Current)**: Unified platform, cross-platform, high-performance

**For 2025, choose .NET 8+ because:**
- **Performance**: 20-30% faster than .NET Framework
- **Cloud-native**: Built for containers, microservices
- **Cross-platform**: Linux containers, AWS Lambda support
- **Modern features**: Native AOT, minimal APIs, improved JSON handling

```csharp
// Modern .NET 8 minimal API
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/health", () => new { Status = "Healthy", Timestamp = DateTime.UtcNow });
app.Run();
```

---

### Q4: How do you implement dependency injection in modern .NET applications?

**Answer:**
.NET has built-in DI container with three lifetimes:

```csharp
// Program.cs (.NET 8)
var builder = WebApplication.CreateBuilder(args);

// Transient - new instance every time
builder.Services.AddTransient<IEmailService, EmailService>();

// Scoped - one per HTTP request
builder.Services.AddScoped<IUserRepository, UserRepository>();

// Singleton - one instance for application lifetime
builder.Services.AddSingleton<IConfigurationService, ConfigurationService>();

// AWS services integration
builder.Services.AddAWSService<IAmazonS3>();
builder.Services.AddAWSService<IAmazonSQS>();

var app = builder.Build();
```

**Advanced patterns for mid-level:**
- **Factory Pattern**: `AddTransient<Func<string, IPaymentProcessor>>(provider => key => ...)`
- **Keyed Services (.NET 8)**: `AddKeyedScoped<IPaymentProcessor, StripePaymentProcessor>("stripe")`
- **Configuration binding**: `builder.Services.Configure<DatabaseOptions>(builder.Configuration.GetSection("Database"))`

---

## ☁️ **Cloud Integration & AWS Services**

### Q5: How do you deploy .NET applications to AWS in 2025?

**Answer:**
Multiple deployment options based on requirements:

**1. AWS Lambda (Serverless)**
```csharp
// Lambda function for API Gateway
public class Function
{
    public async Task<APIGatewayProxyResponse> FunctionHandler(
        APIGatewayProxyRequest request, ILambdaContext context)
    {
        return new APIGatewayProxyResponse
        {
            StatusCode = 200,
            Body = JsonSerializer.Serialize(new { message = "Hello from Lambda!" }),
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }
}
```

**2. ECS Fargate (Containerized)**
```dockerfile
# Dockerfile for .NET 8
FROM mcr.microsoft.com/dotnet/aspnet:8.0
COPY bin/Release/net8.0/publish/ App/
WORKDIR /App
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**3. Elastic Beanstalk (Traditional)**
- Upload deployment package
- Auto-scaling and load balancing
- Easy rollbacks and monitoring

**Key considerations:**
- Use **Application Load Balancer** for HTTP/HTTPS traffic
- Implement **health checks** for container orchestration
- Use **AWS Systems Manager** for configuration management

---

### Q6: How do you handle configuration and secrets in cloud-deployed .NET applications?

**Answer:**
Modern configuration hierarchy (most to least priority):

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 1. appsettings.json
// 2. appsettings.{Environment}.json  
// 3. Environment variables
// 4. Command line arguments

// AWS Secrets Manager integration
builder.Configuration.AddSecretsManager(configurator: options =>
{
    options.SecretFilter = entry => entry.Name.StartsWith("MyApp/");
    options.KeyGenerator = (entry, key) => key.Replace("MyApp/", "");
});

// AWS Systems Manager Parameter Store
builder.Configuration.AddSystemsManager("/myapp");
```

**Best practices:**
- **Never** store secrets in appsettings.json
- Use **AWS Secrets Manager** for database connections, API keys
- Use **Systems Manager Parameter Store** for non-secret configuration
- Implement configuration validation:

```csharp
builder.Services.AddOptions<DatabaseConfig>()
    .Bind(builder.Configuration.GetSection("Database"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

---

## 🔄 **Async Programming & Performance**

### Q7: Explain the difference between Task.Result, .Wait(), and await. What are the risks?

**Answer:**
**Task.Result and .Wait() are dangerous** - they can cause **deadlocks** in certain contexts:

```csharp
// ❌ DANGEROUS - Can deadlock in ASP.NET Framework
public ActionResult BadExample()
{
    var result = SomeAsyncMethod().Result; // Blocks thread
    return View(result);
}

// ❌ DANGEROUS - Same issue
public ActionResult BadExample2()
{
    SomeAsyncMethod().Wait(); // Blocks thread
    return View();
}

// ✅ GOOD - Proper async/await
public async Task<ActionResult> GoodExample()
{
    var result = await SomeAsyncMethod(); // Non-blocking
    return View(result);
}
```

**Why deadlocks happen:**
1. `Result`/`Wait()` **blocks** the calling thread
2. The async method tries to **resume** on the original context
3. Original context is **blocked** → deadlock

**Solutions:**
- Always use `await` in async methods
- Use `ConfigureAwait(false)` in libraries
- In .NET Core/5+, default synchronization context doesn't cause this issue

---

### Q8: How do you implement efficient database operations in .NET?

**Answer:**
**Entity Framework Core best practices:**

```csharp
// ✅ Async operations
public async Task<List<User>> GetActiveUsersAsync()
{
    return await _context.Users
        .Where(u => u.IsActive)
        .AsNoTracking() // Read-only operations
        .ToListAsync();
}

// ✅ Bulk operations (.NET 8)
public async Task DeleteInactiveUsersAsync()
{
    await _context.Users
        .Where(u => !u.IsActive)
        .ExecuteDeleteAsync(); // Bulk delete without loading entities
}

// ✅ Efficient pagination
public async Task<PagedResult<User>> GetUsersPagedAsync(int page, int size)
{
    var totalCount = await _context.Users.CountAsync();
    var users = await _context.Users
        .Skip((page - 1) * size)
        .Take(size)
        .AsNoTracking()
        .ToListAsync();
    
    return new PagedResult<User>(users, totalCount, page, size);
}
```

**Connection management with AWS RDS:**
```csharp
// Connection pooling configuration
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.EnableRetryOnFailure(3);
        sqlOptions.CommandTimeout(30);
    }));

// AWS RDS Proxy for connection pooling
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer("Server=myapp-rds-proxy.proxy-xyz.us-east-1.rds.amazonaws.com;..."));
```

---

## 🧪 **Testing Strategies**

### Q9: How do you structure unit tests for .NET applications in 2025?

**Answer:**
**Modern testing approach with xUnit, FluentAssertions, and Testcontainers:**

```csharp
// Unit test example
public class UserServiceTests
{
    private readonly Mock<IUserRepository> _mockRepository;
    private readonly UserService _userService;

    public UserServiceTests()
    {
        _mockRepository = new Mock<IUserRepository>();
        _userService = new UserService(_mockRepository.Object);
    }

    [Fact]
    public async Task GetUserAsync_WithValidId_ReturnsUser()
    {
        // Arrange
        var userId = Guid.NewGuid();
        var expectedUser = new User { Id = userId, Name = "John Doe" };
        _mockRepository.Setup(x => x.GetByIdAsync(userId))
                      .ReturnsAsync(expectedUser);

        // Act
        var result = await _userService.GetUserAsync(userId);

        // Assert
        result.Should().NotBeNull();
        result.Name.Should().Be("John Doe");
        _mockRepository.Verify(x => x.GetByIdAsync(userId), Times.Once);
    }
}
```

**Integration tests with Testcontainers:**
```csharp
public class UserControllerIntegrationTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _dbContainer = new PostgreSqlBuilder()
        .WithImage("postgres:15")
        .WithDatabase("testdb")
        .Build();

    public async Task InitializeAsync()
    {
        await _dbContainer.StartAsync();
    }

    [Fact]
    public async Task GetUsers_ReturnsListOfUsers()
    {
        // Test with real database running in container
        var connectionString = _dbContainer.GetConnectionString();
        // ... test implementation
    }

    public async Task DisposeAsync()
    {
        await _dbContainer.DisposeAsync();
    }
}
```

---

## 📊 **Monitoring & Observability**

### Q10: How do you implement monitoring and logging in .NET applications deployed to AWS?

**Answer:**
**Structured logging with Serilog and AWS CloudWatch:**

```csharp
// Program.cs
builder.Host.UseSerilog((context, services, configuration) =>
    configuration
        .ReadFrom.Configuration(context.Configuration)
        .Enrich.FromLogContext()
        .WriteTo.Console()
        .WriteTo.AmazonCloudWatch(
            logGroupName: "/aws/ecs/myapp",
            region: RegionEndpoint.USEast1));

// Usage in controllers/services
public class UserController : ControllerBase
{
    private readonly ILogger<UserController> _logger;

    public UserController(ILogger<UserController> logger)
    {
        _logger = logger;
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<User>> GetUser(Guid id)
    {
        using var scope = _logger.BeginScope(new { UserId = id });
        
        _logger.LogInformation("Retrieving user {UserId}", id);
        
        try
        {
            var user = await _userService.GetUserAsync(id);
            _logger.LogInformation("Successfully retrieved user {UserId}", id);
            return Ok(user);
        }
        catch (UserNotFoundException ex)
        {
            _logger.LogWarning(ex, "User {UserId} not found", id);
            return NotFound();
        }
    }
}
```

**Metrics with AWS X-Ray:**
```csharp
// Add X-Ray tracing
builder.Services.AddXRayTracing();

// Custom metrics
public class UserService
{
    public async Task<User> GetUserAsync(Guid id)
    {
        using var segment = AWSXRayRecorder.Instance.BeginSubsegment("GetUser");
        segment.AddMetadata("UserId", id.ToString());
        
        var user = await _repository.GetByIdAsync(id);
        
        segment.AddMetadata("UserFound", user != null);
        return user;
    }
}
```

---

## 🔒 **Security Best Practices**

### Q11: What are the key security considerations for .NET applications in 2025?

**Answer:**
**1. Input Validation & Sanitization:**
```csharp
public class CreateUserRequest
{
    [Required, StringLength(100, MinimumLength = 2)]
    [RegularExpression(@"^[a-zA-Z\s]+$")]
    public string Name { get; set; }

    [Required, EmailAddress]
    public string Email { get; set; }
}

// Use model validation
[HttpPost]
public async Task<ActionResult> CreateUser([FromBody] CreateUserRequest request)
{
    if (!ModelState.IsValid)
        return BadRequest(ModelState);
    
    // Process request...
}
```

**2. SQL Injection Prevention:**
```csharp
// ✅ GOOD - Parameterized queries
public async Task<User> GetUserByEmailAsync(string email)
{
    return await _context.Users
        .FirstOrDefaultAsync(u => u.Email == email); // EF Core handles parameterization
}

// ✅ GOOD - Raw SQL with parameters
var users = await _context.Users
    .FromSqlRaw("SELECT * FROM Users WHERE Email = {0}", email)
    .ToListAsync();
```

**3. HTTPS and Security Headers:**
```csharp
// Program.cs
app.UseHsts(); // HTTP Strict Transport Security
app.UseHttpsRedirection();

// Security headers middleware
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Add("X-Frame-Options", "DENY");
    context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
    await next();
});
```

**4. AWS Security Integration:**
```csharp
// Use AWS Secrets Manager for sensitive data
builder.Configuration.AddSecretsManager();

// Implement proper IAM roles - least privilege principle
// Use AWS WAF for application-level protection
// Enable AWS GuardDuty for threat detection
```

---

## 🚀 **Performance Optimization**

### Q12: How do you optimize .NET application performance for cloud deployment?

**Answer:**
**1. Memory Management:**
```csharp
// Use object pooling for frequently created objects
services.AddSingleton<ObjectPool<StringBuilder>>(provider =>
{
    var policy = new StringBuilderPooledObjectPolicy();
    return new DefaultObjectPool<StringBuilder>(policy);
});

// Use ArrayPool for temporary arrays
public void ProcessData(int[] data)
{
    var buffer = ArrayPool<int>.Shared.Rent(data.Length);
    try
    {
        // Process data using buffer
    }
    finally
    {
        ArrayPool<int>.Shared.Return(buffer);
    }
}
```

**2. Caching Strategies:**
```csharp
// In-memory caching
services.AddMemoryCache();

// Redis distributed caching for AWS
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "my-redis-cluster.cache.amazonaws.com:6379";
});

// Usage
public class UserService
{
    private readonly IMemoryCache _cache;
    
    public async Task<User> GetUserAsync(Guid id)
    {
        var cacheKey = $"user:{id}";
        if (_cache.TryGetValue(cacheKey, out User cachedUser))
            return cachedUser;
            
        var user = await _repository.GetByIdAsync(id);
        _cache.Set(cacheKey, user, TimeSpan.FromMinutes(15));
        return user;
    }
}
```

**3. Response Compression:**
```csharp
// Enable response compression
services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<GzipCompressionProvider>();
});
```

**4. Database Optimization:**
```csharp
// Use compiled queries for frequently executed operations
private static readonly Func<AppDbContext, Guid, Task<User>> GetUserQuery =
    EF.CompileAsyncQuery((AppDbContext context, Guid id) =>
        context.Users.FirstOrDefault(u => u.Id == id));

public async Task<User> GetUserAsync(Guid id)
{
    return await GetUserQuery(_context, id);
}
```

---

## 🛠️ **Modern Development Practices**

### Q13: How do you implement health checks in .NET applications for AWS deployment?

**Answer:**
```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddDbContext<AppDbContext>()
    .AddUrlGroup(new Uri("https://api.external-service.com/health"), "external-service")
    .AddAWSService<IAmazonS3>("s3-bucket-check", check =>
        check.BucketName = "my-app-bucket");

// Custom health check
public class CustomHealthCheck : IHealthCheck
{
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken cancellationToken)
    {
        try
        {
            // Perform health check logic
            return HealthCheckResult.Healthy("Service is running correctly");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Service is not responding", ex);
        }
    }
}

// Configure endpoints
app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

---

### Q14: Explain how to implement background services in .NET for cloud deployment.

**Answer:**
```csharp
// Background service implementation
public class OrderProcessingService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<OrderProcessingService> _logger;
    private readonly IAmazonSQS _sqsClient;

    public OrderProcessingService(
        IServiceProvider serviceProvider,
        ILogger<OrderProcessingService> logger,
        IAmazonSQS sqsClient)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
        _sqsClient = sqsClient;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var messages = await _sqsClient.ReceiveMessageAsync(new ReceiveMessageRequest
                {
                    QueueUrl = "https://sqs.us-east-1.amazonaws.com/123456789/order-queue",
                    MaxNumberOfMessages = 10,
                    WaitTimeSeconds = 20 // Long polling
                }, stoppingToken);

                foreach (var message in messages.Messages)
                {
                    using var scope = _serviceProvider.CreateScope();
                    var orderService = scope.ServiceProvider.GetRequiredService<IOrderService>();
                    
                    await ProcessOrderAsync(message, orderService);
                    await _sqsClient.DeleteMessageAsync("queue-url", message.ReceiptHandle);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing orders");
                await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
            }
        }
    }

    private async Task ProcessOrderAsync(Message message, IOrderService orderService)
    {
        var order = JsonSerializer.Deserialize<Order>(message.Body);
        await orderService.ProcessOrderAsync(order);
        _logger.LogInformation("Processed order {OrderId}", order.Id);
    }
}

// Registration
builder.Services.AddHostedService<OrderProcessingService>();
```

---

## 🎭 **Advanced Scenarios**

### Q15: How do you handle distributed transactions in modern .NET applications?

**Answer:**
**Traditional MSDTC approach (on-premises):**
```csharp
// Legacy approach - avoid in cloud
using (var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
{
    await _orderRepository.CreateAsync(order);
    await _inventoryRepository.UpdateAsync(inventory);
    scope.Complete(); // MSDTC coordinates across databases
}
```

**Modern cloud-native approach - Saga pattern:**
```csharp
public class OrderSaga
{
    private readonly IOrderService _orderService;
    private readonly IInventoryService _inventoryService;
    private readonly IPaymentService _paymentService;

    public async Task ProcessOrderAsync(CreateOrderCommand command)
    {
        var sagaId = Guid.NewGuid();
        
        try
        {
            // Step 1: Create order
            var orderId = await _orderService.CreateOrderAsync(command.Order, sagaId);
            
            // Step 2: Reserve inventory
            await _inventoryService.ReserveItemsAsync(command.Items, sagaId);
            
            // Step 3: Process payment
            await _paymentService.ProcessPaymentAsync(command.Payment, sagaId);
            
            // Success - confirm all steps
            await _orderService.ConfirmOrderAsync(orderId);
            await _inventoryService.ConfirmReservationAsync(sagaId);
        }
        catch (Exception ex)
        {
            // Compensate in reverse order
            await _paymentService.RefundAsync(sagaId);
            await _inventoryService.ReleaseReservationAsync(sagaId);
            await _orderService.CancelOrderAsync(sagaId);
            throw;
        }
    }
}
```

**Event-driven approach with AWS EventBridge:**
```csharp
public class OrderCreatedHandler : INotificationHandler<OrderCreatedEvent>
{
    public async Task Handle(OrderCreatedEvent notification, CancellationToken cancellationToken)
    {
        // Publish event to AWS EventBridge
        await _eventBridge.PutEventsAsync(new PutEventsRequest
        {
            Entries = new List<PutEventsRequestEntry>
            {
                new PutEventsRequestEntry
                {
                    Source = "order-service",
                    DetailType = "Order Created",
                    Detail = JsonSerializer.Serialize(notification)
                }
            }
        });
    }
}
```

---

## 📋 **System Design Questions**

### Q16: Design a scalable .NET microservices architecture on AWS.

**Answer:**
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   API Gateway   │────│  Application     │────│   Services      │
│   (AWS ALB)     │    │  Load Balancer   │    │   (ECS/EKS)     │
└─────────────────┘    └──────────────────┘    └─────────────────┘
         │                        │                       │
         │                        │                       │
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   CloudFront    │    │   Auto Scaling   │    │   RDS/DynamoDB  │
│   (CDN)         │    │   Groups         │    │   (Database)    │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

**Key components:**

**1. API Gateway Pattern:**
```csharp
// Ocelot API Gateway configuration
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/users/{everything}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [
        {
          "Host": "user-service.internal",
          "Port": 443
        }
      ],
      "UpstreamPathTemplate": "/users/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://api.mycompany.com"
  }
}
```

**2. Service Communication:**
```csharp
// HTTP client with Polly retry policies
services.AddHttpClient<IUserService, UserService>(client =>
{
    client.BaseAddress = new Uri("https://user-service.internal");
})
.AddPolicyHandler(GetRetryPolicy())
.AddPolicyHandler(GetCircuitBreakerPolicy());

static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: retryAttempt => 
                TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));
}
```

**3. Event-Driven Architecture:**
```csharp
// AWS SQS/SNS integration
public class OrderEventPublisher
{
    private readonly IAmazonSimpleNotificationService _snsClient;

    public async Task PublishOrderCreatedAsync(Order order)
    {
        var message = new
        {
            EventType = "OrderCreated",
            Data = order,
            Timestamp = DateTime.UtcNow
        };

        await _snsClient.PublishAsync(new PublishRequest
        {
            TopicArn = "arn:aws:sns:us-east-1:123456789:order-events",
            Message = JsonSerializer.Serialize(message)
        });
    }
}
```

---

## ⚡ **Quick Fire Round - Essential Concepts**

### Q17: What's new in .NET 8 that impacts mid-level developers?

**Answer:**
- **Native AOT**: Faster cold starts, smaller memory footprint
- **Keyed DI Services**: Multiple implementations with keys
- **Primary Constructors**: Simplified class syntax
- **Collection Expressions**: `List<int> numbers = [1, 2, 3];`
- **Frozen Collections**: Immutable, optimized for lookups
- **Time Abstraction**: `TimeProvider` for testable time operations

### Q18: Explain the Repository vs Unit of Work patterns.

**Answer:**
```csharp
// Repository Pattern
public interface IUserRepository
{
    Task<User> GetByIdAsync(Guid id);
    Task<List<User>> GetAllAsync();
    Task AddAsync(User user);
    Task UpdateAsync(User user);
    Task DeleteAsync(Guid id);
}

// Unit of Work Pattern
public interface IUnitOfWork : IDisposable
{
    IUserRepository Users { get; }
    IOrderRepository Orders { get; }
    Task<int> SaveChangesAsync();
    Task BeginTransactionAsync();
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
}
```

**Modern approach**: EF Core DbContext already implements Unit of Work pattern.

### Q19: How do you handle versioning in REST APIs?

**Answer:**
**URL Versioning:**
```csharp
[ApiController]
[Route("api/v{version:apiVersion}/users")]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
public class UsersController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public Task<List<UserV1>> GetUsersV1() { }

    [HttpGet]
    [MapToApiVersion("2.0")]
    public Task<List<UserV2>> GetUsersV2() { }
}
```

### Q20: What are the key differences between IEnumerable, ICollection, and IList?

**Answer:**
- **IEnumerable**: Forward-only iteration, deferred execution, most general
- **ICollection**: Adds Count, Add, Remove, Contains - collection operations
- **IList**: Adds indexing, Insert, RemoveAt - positional access

**Performance tip**: Use `IReadOnlyList<T>` when you don't need modification capabilities.

---

## 🎯 **Practical Coding Challenges**

### Q21: Write a method that safely executes multiple async operations with a timeout and cancellation.

**Answer:**
```csharp
public static async Task<T[]> ExecuteWithTimeoutAsync<T>(
    IEnumerable<Func<CancellationToken, Task<T>>> operations,
    TimeSpan timeout,
    CancellationToken cancellationToken = default)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
    cts.CancelAfter(timeout);

    var tasks = operations.Select(op => op(cts.Token)).ToArray();

    try
    {
        return await Task.WhenAll(tasks);
    }
    catch (Oper