# Java Interview Topics Checklist
## For Lead/Principal/Architect Roles

---

## Java Core (Must Know)

### Language Fundamentals
- [ ] **Object vs Primitive Types**: Wrapper classes, autoboxing
- [ ] **String Immutability**: Why strings are immutable, `StringBuilder` vs `String`
- [ ] **`==` vs `equals()`**: Reference vs value comparison
- [ ] **`hashCode()` and `equals()`**: Contract, best practices
- [ ] **Generics**: Wildcards (`?`, `? extends T`, `? super T`), type erasure
- [ ] **Reflection**: Runtime type inspection (less common, but good to know)

### Collections Framework
- [ ] **List**: `ArrayList` vs `LinkedList` (when to use)
- [ ] **Set**: `HashSet` vs `TreeSet` vs `LinkedHashSet`
- [ ] **Map**: `HashMap` vs `TreeMap` vs `LinkedHashMap`
- [ ] **Thread-Safe Collections**: `ConcurrentHashMap`, `CopyOnWriteArrayList`
- [ ] **Streams API**: `filter`, `map`, `reduce`, `collect`, parallel streams

### Memory & JVM
- [ ] **Heap vs Stack**: Where objects live
- [ ] **Garbage Collection**: Generational GC, GC algorithms (G1, ZGC)
- [ ] **Memory Leaks**: Common causes, how to detect
- [ ] **JVM Tuning**: Heap size, GC tuning flags

### Concurrency
- [ ] **Thread Creation**: `Thread` class vs `Runnable` vs `Callable`
- [ ] **Synchronization**: `synchronized` keyword, `volatile`
- [ ] **Thread Pools**: `ExecutorService`, `ThreadPoolExecutor`
- [ ] **Concurrent Collections**: `ConcurrentHashMap`, `BlockingQueue`
- [ ] **`CompletableFuture`**: Async programming (Java 8+)
- [ ] **Deadlock**: How to prevent, detect

---

## Spring Framework (Critical)

### Spring Core
- [ ] **Dependency Injection**: Constructor vs setter injection
- [ ] **Bean Scopes**: Singleton, Prototype, Request, Session
- [ ] **Bean Lifecycle**: `@PostConstruct`, `@PreDestroy`
- [ ] **AOP (Aspect-Oriented Programming)**: `@Aspect`, `@Around`, `@Before`
- [ ] **Auto-configuration**: How Spring Boot auto-configures beans

### Spring Boot
- [ ] **Auto-configuration**: How it works, conditional beans
- [ ] **Starters**: What are Spring Boot starters
- [ ] **Actuator**: Health checks, metrics, endpoints
- [ ] **Profiles**: `application-dev.yml`, `@Profile`
- [ ] **Externalized Configuration**: `@Value`, `@ConfigurationProperties`

### Spring MVC / REST
- [ ] **`@RestController` vs `@Controller`**: Difference
- [ ] **Request Mapping**: `@RequestMapping`, `@GetMapping`, etc.
- [ ] **Request/Response**: `@RequestBody`, `@ResponseBody`, `@PathVariable`
- [ ] **Exception Handling**: `@ControllerAdvice`, `@ExceptionHandler`
- [ ] **Validation**: `@Valid`, `@NotNull`, `@Size`

### Spring Data JPA
- [ ] **Repository Pattern**: `CrudRepository`, `JpaRepository`
- [ ] **Query Methods**: Method naming conventions
- [ ] **Custom Queries**: `@Query`, JPQL, native queries
- [ ] **Transactions**: `@Transactional`, propagation, isolation
- [ ] **Lazy vs Eager Loading**: `FetchType.LAZY`, N+1 problem

### Spring Security
- [ ] **Authentication**: Basic, JWT, OAuth2
- [ ] **Authorization**: `@PreAuthorize`, `@Secured`
- [ ] **Filter Chain**: How Spring Security filters work
- [ ] **Password Encoding**: BCrypt, password hashing

---

## Architecture & Design Patterns

### Design Patterns
- [ ] **Singleton**: Spring's singleton scope
- [ ] **Factory**: `@Bean` methods, `FactoryBean`
- [ ] **Repository**: Data access abstraction
- [ ] **Strategy**: Polymorphism, dependency injection
- [ ] **Observer**: Spring events (`ApplicationEventPublisher`)
- [ ] **Proxy**: Spring AOP proxies

### Microservices
- [ ] **Service Discovery**: Eureka, Consul
- [ ] **API Gateway**: Spring Cloud Gateway
- [ ] **Circuit Breaker**: Resilience4j, Hystrix
- [ ] **Distributed Tracing**: Sleuth, Zipkin
- [ ] **Configuration Server**: Spring Cloud Config

### System Design
- [ ] **Scalability**: Horizontal vs vertical scaling
- [ ] **Caching**: Redis, Caffeine, `@Cacheable`
- [ ] **Message Queues**: RabbitMQ, Kafka
- [ ] **Database Design**: Normalization, indexing, sharding
- [ ] **Load Balancing**: Round-robin, least connections

---

## Database & Persistence

### JPA/Hibernate
- [ ] **Entity Relationships**: `@OneToMany`, `@ManyToOne`, `@ManyToMany`
- [ ] **Cascading**: `CascadeType`, when to use
- [ ] **Lazy Loading**: N+1 problem, `@EntityGraph`
- [ ] **Transactions**: ACID properties, isolation levels
- [ ] **Connection Pooling**: HikariCP (Spring Boot default)

### Database Concepts
- [ ] **ACID Properties**: Atomicity, Consistency, Isolation, Durability
- [ ] **Isolation Levels**: Read uncommitted, committed, repeatable, serializable
- [ ] **Indexing**: B-tree indexes, when to index
- [ ] **Normalization**: 1NF, 2NF, 3NF
- [ ] **SQL Optimization**: Query plans, EXPLAIN

---

## Testing

### Unit Testing
- [ ] **JUnit 5**: `@Test`, assertions
- [ ] **Mockito**: `@Mock`, `@Spy`, `when().thenReturn()`
- [ ] **Test Slices**: `@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest`

### Integration Testing
- [ ] **`@SpringBootTest`**: Full context testing
- [ ] **Test Containers**: Docker-based integration tests
- [ ] **MockMvc**: Testing REST controllers

---

## DevOps & Deployment

### Containerization
- [ ] **Docker**: Dockerfile, multi-stage builds
- [ ] **Docker Compose**: Multi-container apps
- [ ] **Kubernetes**: Pods, Services, Deployments (basics)

### CI/CD
- [ ] **Maven/Gradle**: Build lifecycle, plugins
- [ ] **Jenkins/GitHub Actions**: Pipeline basics
- [ ] **Artifact Repositories**: Maven Central, Nexus

---

## Common Interview Questions

### Java Core
1. **What's the difference between `==` and `equals()`?**
2. **Why are Strings immutable in Java?**
3. **Explain garbage collection in Java.**
4. **What is the difference between `ArrayList` and `LinkedList`?**
5. **How does `HashMap` work internally?**
6. **What is thread safety? How do you achieve it?**
7. **Explain `volatile` keyword.**
8. **What is the difference between `synchronized` and `ReentrantLock`?**

### Spring Boot
1. **How does Spring Boot auto-configuration work?**
2. **What is the difference between `@Component`, `@Service`, and `@Repository`?**
3. **Explain Spring's dependency injection.**
4. **How does `@Transactional` work?**
5. **What is the Spring Bean lifecycle?**
6. **How do you handle exceptions in Spring Boot?**
7. **Explain Spring Security filter chain.**
8. **What is the difference between `@Controller` and `@RestController`?**

### Architecture
1. **Design a URL shortener (like bit.ly).**
2. **Design a chat system.**
3. **How would you scale a Spring Boot application?**
4. **Explain microservices vs monolith.**
5. **How do you handle distributed transactions?**
6. **What is the circuit breaker pattern?**
7. **How do you implement caching in a Spring Boot app?**

---

## Interview Prep Strategy

### Week 1-2: Core Java
- Study Java fundamentals
- Practice coding problems (LeetCode Easy/Medium)
- Understand JVM basics

### Week 3-4: Spring Boot
- Build a REST API project
- Understand Spring annotations
- Practice Spring interview questions

### Week 5-6: Architecture
- Study design patterns
- Practice system design
- Learn microservices concepts

### Week 7-8: Mock Interviews
- Practice explaining concepts
- Whiteboard coding
- System design practice

---

## Resources for Each Topic

### Java Core
- **Baeldung**: Java tutorials
- **Oracle Java Docs**: Official documentation
- **Effective Java** (book): Best practices

### Spring Boot
- **Spring.io Guides**: https://spring.io/guides
- **Baeldung Spring**: Comprehensive tutorials
- **Spring Boot Reference**: Official docs

### System Design
- **Grokking the System Design Interview**: Book/course
- **High Scalability**: http://highscalability.com
- **System Design Primer**: GitHub repo

### Interview Practice
- **LeetCode**: Coding problems (Java)
- **Pramp**: Mock interviews
- **InterviewBit**: Java interview questions

---

## Red Flags (What NOT to Say)

❌ "I don't know multithreading" (critical for architect role)
❌ "I've never used Spring Boot" (must know for Java roles)
❌ "I don't understand dependency injection" (fundamental)
❌ "I've never deployed to production" (leadership role needs this)
❌ "I don't know microservices" (architect role requirement)

---

## Green Flags (What to Emphasize)

✅ "I've built scalable systems in .NET, applying same principles to Java"
✅ "I understand architecture patterns from my 12 years of experience"
✅ "I've led teams and made technical decisions"
✅ "I'm learning Java/Spring Boot through hands-on projects"
✅ "I can explain trade-offs between different approaches"

---

## Final Checklist Before Interview

- [ ] Can explain Java memory model
- [ ] Can explain Spring Boot auto-configuration
- [ ] Can design a REST API
- [ ] Can explain microservices architecture
- [ ] Can solve coding problems in Java
- [ ] Can design a system on whiteboard
- [ ] Can explain your projects clearly
- [ ] Can discuss architecture decisions
- [ ] Can answer behavioral questions about leadership

