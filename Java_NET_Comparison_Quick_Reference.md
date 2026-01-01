# .NET to Java Quick Reference
## For Experienced .NET Developers

---

## Language Syntax Comparison

### Basic Types
| .NET (C#) | Java | Notes |
|-----------|------|-------|
| `int` | `int` | Same |
| `string` | `String` | Java: reference type, immutable |
| `bool` | `boolean` | Java: lowercase |
| `DateTime` | `LocalDateTime`, `ZonedDateTime` | Java 8+ time API |
| `decimal` | `BigDecimal` | Different precision model |
| `var` | `var` (Java 10+) | Type inference |

### Collections
| .NET | Java | Notes |
|------|------|-------|
| `List<T>` | `ArrayList<T>`, `LinkedList<T>` | Java: interface-based |
| `Dictionary<K,V>` | `HashMap<K,V>`, `TreeMap<K,V>` | Java: no indexer, use `get()` |
| `HashSet<T>` | `HashSet<T>` | Similar |
| `Queue<T>` | `Queue<T>` | Similar |
| LINQ | Streams API | Java 8+ functional style |

### OOP Keywords
| .NET (C#) | Java | Notes |
|-----------|------|-------|
| `class` | `class` | Same |
| `interface` | `interface` | Java 8+: default methods |
| `abstract` | `abstract` | Same |
| `sealed` | `final` | Final class/method |
| `virtual` | (default) | Java: all methods virtual by default |
| `override` | `@Override` | Java: annotation |
| `namespace` | `package` | Similar concept |

### Access Modifiers
| .NET | Java | Notes |
|------|------|-------|
| `public` | `public` | Same |
| `private` | `private` | Same |
| `protected` | `protected` | Similar |
| `internal` | `package-private` (default) | No keyword needed |
| `protected internal` | `protected` | Closest equivalent |

---

## Framework Comparison

### Dependency Injection
| .NET | Java (Spring) | Notes |
|------|---------------|-------|
| `IServiceCollection` | `ApplicationContext` | Container |
| `services.AddScoped<T>()` | `@Service` (singleton by default) | Scope differs |
| `services.AddTransient<T>()` | `@Scope("prototype")` | Java: explicit scope |
| `services.AddSingleton<T>()` | `@Service` (default) | Java: singleton default |
| Constructor injection | Constructor injection | Same pattern |
| `[FromServices]` | `@Autowired` | Annotation-based |

### Web Framework
| ASP.NET Core | Spring Boot | Notes |
|--------------|-------------|-------|
| `[ApiController]` | `@RestController` | REST controller |
| `[Route("api/users")]` | `@RequestMapping("/api/users")` | Routing |
| `[HttpGet]` | `@GetMapping` | HTTP methods |
| `[FromBody]` | `@RequestBody` | Request body |
| `[FromQuery]` | `@RequestParam` | Query parameters |
| `app.UseCors()` | `@CrossOrigin` | CORS |
| Middleware | Interceptors/Filters | Similar concept |

### ORM
| Entity Framework | JPA/Hibernate | Notes |
|------------------|---------------|-------|
| `DbContext` | `EntityManager` | Context |
| `DbSet<T>` | `Repository<T>` | Data access |
| `[Key]` | `@Id` | Primary key |
| `[Required]` | `@NotNull` | Validation |
| `[Column("name")]` | `@Column(name = "name")` | Column mapping |
| LINQ queries | JPQL/Criteria API | Query language |
| Migrations | Flyway/Liquibase | Database versioning |

### Configuration
| .NET | Java (Spring) | Notes |
|------|---------------|-------|
| `appsettings.json` | `application.properties` / `application.yml` | Config files |
| `IConfiguration` | `@Value`, `@ConfigurationProperties` | Config injection |
| Environment variables | Environment variables | Same |
| `appsettings.Development.json` | `application-dev.yml` | Profiles |

---

## Common Patterns

### Dependency Injection Example

**.NET (C#):**
```csharp
public class UserService
{
    private readonly IUserRepository _repository;
    
    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }
}
```

**Java (Spring):**
```java
@Service
public class UserService {
    private final UserRepository repository;
    
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}
```

### REST Controller Example

**.NET (C#):**
```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    private readonly IUserService _service;
    
    [HttpGet("{id}")]
    public ActionResult<User> GetUser(int id)
    {
        return Ok(_service.GetById(id));
    }
}
```

**Java (Spring):**
```java
@RestController
@RequestMapping("/api/users")
public class UsersController {
    private final UserService service;
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable int id) {
        return ResponseEntity.ok(service.getById(id));
    }
}
```

### LINQ vs Streams

**.NET (C#):**
```csharp
var adults = users
    .Where(u => u.Age >= 18)
    .Select(u => u.Name)
    .ToList();
```

**Java:**
```java
List<String> adults = users.stream()
    .filter(u -> u.getAge() >= 18)
    .map(User::getName)
    .collect(Collectors.toList());
```

---

## Key Differences to Remember

### 1. Null Safety
- **.NET**: Nullable reference types (C# 8+)
- **Java**: No built-in null safety (use `Optional<T>`)

### 2. Properties vs Getters/Setters
- **.NET**: Properties (`public string Name { get; set; }`)
- **Java**: Explicit getters/setters (`getName()`, `setName()`)

### 3. Exception Handling
- **.NET**: `try-catch-finally`
- **Java**: `try-catch-finally` (similar, but checked exceptions)

### 4. Async/Await
- **.NET**: `async/await` (built-in)
- **Java**: `CompletableFuture` or reactive (Spring WebFlux)

### 5. Package Management
- **.NET**: NuGet packages
- **Java**: Maven (pom.xml) or Gradle (build.gradle)

---

## Interview Talking Points

### "Why Java over .NET?"
- **Ecosystem**: Larger open-source community
- **Cross-platform**: JVM runs everywhere
- **Enterprise**: More enterprise adoption
- **Microservices**: Spring Cloud is mature

### "How does your .NET experience help?"
- **Architecture**: Same design patterns
- **REST APIs**: Same concepts
- **ORM**: Similar principles (EF vs JPA)
- **DI**: Same patterns, different syntax
- **Leadership**: Experience translates directly

---

## Quick Conversion Cheat Sheet

### Common Tasks

**Create a list:**
- .NET: `var list = new List<string>();`
- Java: `List<String> list = new ArrayList<>();`

**Dictionary/Map:**
- .NET: `var dict = new Dictionary<string, int>(); dict["key"] = 1;`
- Java: `Map<String, Integer> map = new HashMap<>(); map.put("key", 1);`

**String interpolation:**
- .NET: `$"Hello {name}"`
- Java: `String.format("Hello %s", name)` or `"Hello " + name`

**Null check:**
- .NET: `if (obj != null)`
- Java: `if (obj != null)` (same, but consider `Optional`)

**Exception:**
- .NET: `throw new Exception("message");`
- Java: `throw new Exception("message");` (same)

---

## Spring Boot vs ASP.NET Core Mental Model

| Concept | ASP.NET Core | Spring Boot |
|---------|--------------|-------------|
| **Startup** | `Program.cs` | `@SpringBootApplication` main class |
| **Configuration** | `Startup.cs` / `Program.cs` | `@Configuration` classes |
| **Middleware** | `app.UseXxx()` | `@Component` with `@Order` |
| **Routing** | `[Route]` attributes | `@RequestMapping` |
| **Dependency Injection** | `services.AddXxx()` | `@Component`, `@Service`, `@Repository` |
| **Auto-configuration** | Convention-based | `@EnableAutoConfiguration` |
| **Actuator** | Health checks | Spring Boot Actuator |

---

## Resources for Quick Lookup
- **Baeldung**: https://www.baeldung.com (Best for Spring)
- **Spring.io Docs**: https://docs.spring.io/spring-boot/
- **Java API Docs**: https://docs.oracle.com/javase/8/docs/api/

