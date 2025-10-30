# **COMPLETE INTERVIEWER QUESTIONS & ANSWERS**
## **For Java, Spring Boot, Microservices, System Design, IBM MQ, Oracle, .NET, Angular, DevOps**
### **Based on YOUR Projects: Recon Engine, CRTS, OPS Portal**

---

## **PART 1: JAVA & CORE BACKEND QUESTIONS**

### **Q1: Explain Java Streams and how you used them in the Reconciliation Engine**

**INTERVIEWER ASKS:**
"You mentioned processing 10 million transactions daily. Walk me through how you optimized the reconciliation engine using Java Streams."

**YOUR ANSWER:**

"Java Streams provide a declarative, functional approach to processing collections. They're lazy, immutable, and support both sequential and parallel processing.

In the Reconciliation Engine, I leveraged parallel streams to process 10M transactions daily. Here's the pattern I implemented:

**Problem:** Sequential processing took 12 hours. We needed results in 8 hours for morning reports.

**Solution:**
```java
public ReconciliationReport reconcileDaily(LocalDate date) {
    List<Transaction> sourceTransactions = loadSourceTransactions(date); // 10M records
    
    // Step 1: Partition by account using groupingBy
    Map<String, List<Transaction>> grouped = sourceTransactions
        .parallelStream()  // CPU-intensive matching = parallel
        .filter(txn -> txn.getAmount().compareTo(BigDecimal.ZERO) > 0)
        .filter(txn -> 'PENDING'.equals(txn.getStatus()))
        .collect(Collectors.groupingByConcurrent(
            Transaction::getAccountId,
            Collectors.toCollection(ConcurrentHashMap::new),
            Collectors.toList()
        ));
    
    // Step 2: Parallel matching per account
    List<ReconciliationResult> results = grouped.entrySet()
        .parallelStream()  // Process accounts in parallel
        .map(entry -> reconcileAccount(entry.getKey(), entry.getValue()))
        .flatMap(List::stream)
        .collect(Collectors.toList());
    
    return generateReport(results);
}

private List<ReconciliationResult> reconcileAccount(String accountId, List<Transaction> sourceTxns) {
    return sourceTxns.stream()  // Sequential here - each account processed independently
        .map(source -> findMatch(source)
            .map(target -> ReconciliationResult.matched(source, target))
            .orElse(ReconciliationResult.unmatched(source))
        )
        .collect(Collectors.toList());
}
```

**Results:**
- Processing time: 12 hours → 7 hours (42% improvement)
- Throughput: 230 TPS → 400 TPS (40% increase)
- Infrastructure: Same 5 servers (just better utilization)

**Key Optimization Insights:**
1. **Partitioned before parallel** - groupingBy by account splits work efficiently
2. **Used groupingByConcurrent** - thread-safe collection during parallel operations
3. **Kept sequential where needed** - I/O operations (DB calls) remained sequential to avoid overwhelming connection pool
4. **Measured before optimizing** - JProfiler showed bottleneck was matching logic, not I/O
5. **ForkJoinPool tuning** - Configured via -Djava.util.concurrent.ForkJoinPool.common.parallelism=10

This taught me when to use parallel streams: large datasets (>10K), CPU-intensive operations, independent tasks. Not suitable for I/O-bound work."

---

### **Q2: Explain Lambda Expressions and Functional Interfaces - How did you use them in CRTS?**

**INTERVIEWER ASKS:**
"Tell me about lambda expressions and functional interfaces. Give a real example from one of your projects."

**YOUR ANSWER:**

"Lambda expressions enable functional programming in Java. They work with Functional Interfaces (@FunctionalInterface) - interfaces with exactly one abstract method.

In the CRTS project (Charles River Trading System integration), I used lambdas + functional interfaces for flexible message routing:

**Problem:** We had 5+ message types (TradeAllocation, Settlement, CashMovement, etc.). Original code had massive if-else chains.

**Solution - Strategy Pattern with Lambdas:**
```java
// 1. Define a functional interface for message processing
@FunctionalInterface
public interface MessageHandler {
    ProcessingResult handle(Message message);
}

// 2. Create message processor factory
public class MessageProcessorFactory {
    private final Map<String, MessageHandler> handlers = Map.of(
        'TRADE_ALLOCATION', msg -> tradeService.allocate(msg),
        'SETTLEMENT', msg -> settlementService.settle(msg),
        'POSITION_UPDATE', msg -> positionService.update(msg),
        'CASH_MOVEMENT', msg -> cashService.process(msg),
        'CORPORATE_ACTION', msg -> corporateActionService.process(msg)
    );
    
    public ProcessingResult processMessage(Message message) {
        return Optional.ofNullable(handlers.get(message.getType()))
            .map(handler -> handler.handle(message))
            .orElseThrow(() -> new UnsupportedMessageTypeException(message.getType()));
    }
}

// 3. Usage with callback pattern
public void handleMessage(Message msg) {
    processMessage(
        msg,
        result -> {  // Success callback (lambda)
            log.info('Message processed: {}', result);
            auditService.log(result);
            notificationService.notify(result);
        },
        error -> {  // Error callback (lambda)
            log.error('Message processing failed', error);
            deadLetterQueue.send(msg, error);
            alertService.triggerAlert(error);
        }
    );
}
```

**Benefits:**
1. **Eliminated if-else chains** - Messages are added to map, not code changes
2. **Higher-order functions** - Passed behavior (lambdas) as parameters
3. **Testability** - Easy to mock handlers in unit tests
4. **Type safety** - Compiler ensures handler interface compliance
5. **Conciseness** - 1 line lambdas vs 10-line methods

**Other Lambda Patterns I Used:**
- **Retries with exponential backoff:**
```java
<T> T executeWithRetry(FailableSupplier<T> operation, int maxRetries) {
    int attempts = 0;
    while (attempts < maxRetries) {
        try {
            return operation.get();  // Lambda passed as supplier
        } catch (Exception e) {
            if (++attempts >= maxRetries) throw new RetryFailedException(e);
            Thread.sleep(1000 * attempts);  // Exponential backoff
        }
    }
}
```

- **Filtering with composed predicates:**
```java
Predicate<Trade> isHighValue = t -> t.getAmount() > 1_000_000;
Predicate<Trade> isPending = t -> 'PENDING'.equals(t.getStatus());
List<Trade> reviewed = trades.stream()
    .filter(isHighValue.and(isPending))
    .collect(Collectors.toList());
```

The key insight: Lambdas make code **declarative** (what to do, not how), which improves readability and maintainability."

---

### **Q3: What is Optional and how does it eliminate NullPointerException?**

**INTERVIEWER ASKS:**
"How do you handle null values in Java? Show me from your reconciliation engine."

**YOUR ANSWER:**

"Optional is a container for null values. It forces explicit null handling instead of defensive null checks.

**Traditional approach (error-prone):**
```java
Trade trade = repository.findById(id);
if (trade != null && trade.getAccount() != null && 
    trade.getAccount().getCustomer() != null &&
    trade.getAccount().getCustomer().getEmail() != null) {
    sendEmail(trade.getAccount().getCustomer().getEmail());
} else {
    log.warn('Cannot send email - null reference');
}
```

**Optional approach (clean, safe):**
```java
repository.findById(id)  // Returns Optional<Trade>
    .map(Trade::getAccount)        // Optional<Account>
    .map(Account::getCustomer)      // Optional<Customer>
    .map(Customer::getEmail)        // Optional<String>
    .ifPresentOrElse(
        email -> sendEmail(email),
        () -> log.warn('Customer email not found')
    );
```

**In Recon Engine - Finding Reconciliation Matches:**
```java
public Optional<Transaction> findMatch(Transaction source) {
    return Optional.ofNullable(targetSystem.findByKey(source.getKey()))
        .filter(target -> matchesAmount(source, target))
        .filter(target -> matchesDate(source, target))
        .filter(target -> matchesCurrency(source, target));
}

// Usage
findSourceTransaction(id)
    .flatMap(this::findMatch)
    .map(match -> ReconciliationResult.matched(match))
    .ifPresentOrElse(
        result -> {
            saveResult(result);
            log.info('Matched: {}', id);
        },
        () -> {
            createException(id);
            log.warn('No match: {}', id);
        }
    );
```

**Key Optional Methods:**
- **map()** - Transform value if present
- **flatMap()** - Chain operations returning Optional
- **filter()** - Keep value only if predicate true
- **orElse()** - Default value (always creates default)
- **orElseGet()** - Lazy default (only called if empty)
- **orElseThrow()** - Throw custom exception
- **ifPresent()** - Execute if present
- **ifPresentOrElse()** - Execute one of two branches

**In 2+ years of production, we had ZERO NullPointerExceptions in the core reconciliation logic because of this pattern.**"

---

### **Q4: Explain CompletableFuture and async programming in CRTS**

**INTERVIEWER ASKS:**
"Tell me about asynchronous programming. How did you implement it in CRTS for parallel API calls?"

**YOUR ANSWER:**

"CompletableFuture enables non-blocking async execution with functional composition. It returns immediately, allowing other tasks to run while waiting for results.

**Problem in CRTS:** 
Fetching enriched trade data from Charles River, Aladdin, and market data systems took 300ms sequentially:
- Charles River API call: 100ms
- Aladdin system call: 80ms
- Market data service: 120ms
- Total: 300ms

**Sequential approach (blocking):**
```java
Trade trade = charlesRiverClient.fetchTrade(id);           // Wait 100ms
MarketData market = marketDataService.fetch(trade);       // Wait 80ms
ReferenceData refData = referenceService.fetch(trade);    // Wait 120ms
return enrichTrade(trade, market, refData);               // Total: 300ms
```

**CompletableFuture approach (parallel):**
```java
public CompletableFuture<EnrichedTrade> enrichTradeAsync(String tradeId) {
    // Launch all 3 requests in parallel
    CompletableFuture<Trade> tradeFuture = 
        CompletableFuture.supplyAsync(
            () -> charlesRiverClient.fetchTrade(tradeId),
            asyncExecutor  // Custom thread pool
        );
    
    CompletableFuture<MarketData> marketFuture = 
        CompletableFuture.supplyAsync(
            () -> marketDataService.fetch(tradeId),
            asyncExecutor
        );
    
    CompletableFuture<ReferenceData> refDataFuture = 
        CompletableFuture.supplyAsync(
            () -> referenceService.fetch(tradeId),
            asyncExecutor
        );
    
    // Wait for all 3 to complete
    return CompletableFuture.allOf(tradeFuture, marketFuture, refDataFuture)
        .thenApply(v -> new EnrichedTrade(
            tradeFuture.join(),
            marketFuture.join(),
            refDataFuture.join()
        ))
        .exceptionally(ex -> {
            log.error('Failed to enrich trade', ex);
            throw new EnrichmentFailedException(tradeId, ex);
        });
}
```

**Results:**
- Sequential: 300ms
- Parallel: 120ms (slowest request)
- **60% latency reduction**

**Advanced CompletableFuture Patterns:**

1. **Timeout handling:**
```java
public CompletableFuture<Trade> fetchWithTimeout(String id) {
    return CompletableFuture.supplyAsync(() -> externalApi.fetch(id))
        .orTimeout(2, TimeUnit.SECONDS)
        .exceptionally(ex -> {
            log.warn('Timeout, using cache');
            return tradeCache.get(id);
        });
}
```

2. **Retry logic:**
```java
public <T> CompletableFuture<T> executeWithRetry(
    Supplier<CompletableFuture<T>> supplier,
    int maxRetries) {
    
    return supplier.get()
        .exceptionallyCompose(ex -> {
            if (maxRetries > 0) {
                log.warn('Retry {} remaining', maxRetries);
                return executeWithRetry(supplier, maxRetries - 1);
            }
            return CompletableFuture.failedFuture(ex);
        });
}
```

3. **First successful result (multiple sources):**
```java
public CompletableFuture<Trade> fetchFromMultipleSources(String id) {
    CompletableFuture<Trade> source1 = asyncFetch(database1, id);
    CompletableFuture<Trade> source2 = asyncFetch(database2, id);
    CompletableFuture<Trade> cache = asyncFetch(cacheLayer, id);
    
    return CompletableFuture.anyOf(source1, source2, cache)
        .thenApply(result -> (Trade) result);  // First successful wins
}
```

**Thread Pool Configuration:**
```java
@Bean
public Executor asyncExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(20);
    executor.setMaxPoolSize(50);
    executor.setQueueCapacity(500);
    executor.setThreadNamePrefix('crts-async-');
    executor.initialize();
    return executor;
}
```

**When NOT to use CompletableFuture:**
- CPU-intensive operations (use parallel streams instead)
- Operations requiring strict order
- Simple synchronous operations (overhead not worth it)

**Key insight:** CompletableFuture changed CRTS from blocking request/response to non-blocking async, enabling **60% latency reduction while using same infrastructure**."

---

### **Q5: Explain Java 17 Sealed Classes and Records - How did you use them?**

**INTERVIEWER ASKS:**
"Java 17 is an LTS release. Tell me about sealed classes and records. Did you use them in your projects?"

**YOUR ANSWER:**

"Java 17 introduces sealed classes and records for type safety and conciseness.

**Sealed Classes - Controlling Inheritance:**

In CRTS, we have fixed message types. Before sealed classes:
```java
// Old approach - no compiler guarantee of all types
public interface MessageType {
    void process();
}

// Anyone could create new implementations!
public class RandomMessage implements MessageType { }  // Dangerous!
```

**With sealed classes (Java 17):**
```java
public sealed interface MessageType 
    permits TradeAllocation, Settlement, PositionUpdate, CashMovement {
}

public final class TradeAllocation implements MessageType {
    private final String tradeId;
    private final BigDecimal quantity;
    
    public TradeAllocation(String tradeId, BigDecimal quantity) {
        this.tradeId = tradeId;
        this.quantity = quantity;
    }
}

// Similar for Settlement, PositionUpdate, CashMovement...

// Now exhaustive pattern matching (compiler enforces!)
public void routeMessage(MessageType message) {
    switch (message) {
        case TradeAllocation t -> {
            log.info('Allocating trade: {}', t.tradeId);
            tradeService.allocate(t);
        }
        case Settlement s -> settlementService.settle(s);
        case PositionUpdate p -> positionService.update(p);
        case CashMovement c -> cashService.process(c);
        // NO default needed! Compiler ensures all cases covered
    };
}
```

**Benefits:**
- Compiler forces handling all message types
- Adding new type → compilation error everywhere it's used
- **Prevents production bugs from forgotten cases**

**Records - Immutable Data Classes:**

Reconciliation results are DTOs. Before records (~50 lines):
```java
public class ReconciliationResult {
    private final String sourceId;
    private final String targetId;
    private final boolean matched;
    private final BigDecimal amountDifference;
    
    public ReconciliationResult(String sourceId, String targetId, ...) {
        this.sourceId = sourceId;
        this.targetId = targetId;
        // ... field assignments
    }
    
    // getters
    public String getSourceId() { return sourceId; }
    // ... 20+ more lines
    
    // equals, hashCode, toString
}
```

**With records (1 line):**
```java
public record ReconciliationResult(
    String sourceId,
    String targetId,
    boolean matched,
    BigDecimal amountDifference,
    LocalDateTime reconciledAt
) {
    // Compact constructor for validation
    public ReconciliationResult {
        if (sourceId == null) throw new IllegalArgumentException('Source ID required');
    }
    
    // Custom methods allowed
    public boolean hasDiscrepancies() {
        return amountDifference.compareTo(BigDecimal.ZERO) > 0;
    }
}
```

**Automatically generates:**
- Constructor
- Getters (no 'get' prefix: result.sourceId(), not result.getSourceId())
- equals(), hashCode(), toString()
- All fields final and private

**Real Usage in Reconciliation:**
```java
// Create result (automatic constructor)
ReconciliationResult result = new ReconciliationResult(
    'SRC123',
    'TGT123',
    true,
    BigDecimal.ZERO,
    LocalDateTime.now()
);

// Access fields directly (no getters needed)
if (result.matched()) {
    saveToDatabase(result.sourceId(), result.targetId());
}

// In streams
List<ReconciliationResult> results = // ...
results.stream()
    .filter(r -> r.matched())
    .map(r -> r.sourceId())
    .collect(Collectors.toList());
```

**Impact:**
- Eliminated ~500 lines of boilerplate DTO code
- Guaranteed immutability (prevents concurrent modification bugs)
- Pattern matching with sealed classes enables type-safe routing

**When I migrated Recon to Java 17:**
- Added 30+ sealed interfaces for different entity types
- Converted 50 DTOs to records
- Reduced boilerplate by 40%
- **Zero production issues from type safety improvements**"

---

### **Q6: Explain Virtual Threads (Java 21) and when to use them**

**INTERVIEWER ASKS:**
"Java 21 introduced virtual threads via Project Loom. Tell me about them."

**YOUR ANSWER:**

"Virtual threads are JVM-managed, lightweight threads that enable massive concurrency without the overhead of platform threads.

**Problem with Platform Threads:**
```java
// Traditional threads: 1-2 MB each, limited by OS
ExecutorService executor = Executors.newFixedThreadPool(1000);  // Only 1000 threads

for (int i = 0; i < 1_000_000; i++) {
    executor.submit(() -> {
        blockingCall();  // Thread blocked waiting
    });
}
// Only 1000 can run concurrently - rest queue up!
```

**Solution with Virtual Threads (Java 21):**
```java
// Virtual threads: few KB each, millions possible
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {
        executor.submit(() -> {
            blockingCall();  // Virtual thread suspends, not OS thread
        });
    }
}
// 1 million concurrent virtual threads on handful of OS threads!
```

**How It Works:**
```
Traditional Threads (1:1 mapping)
Virtual Thread 1 ← OS Thread 1
Virtual Thread 2 ← OS Thread 2
Virtual Thread 3 ← OS Thread 3
(Only 3 run concurrently)

Virtual Threads (many:1 mapping)
Virtual Thread 1 \
Virtual Thread 2 - ← OS Thread 1 (runs when Virtual Thread yields)
Virtual Thread 3 / 
Virtual Thread 4 \
Virtual Thread 5 - ← OS Thread 2
...
Virtual Thread 1M /
```

**Use Cases for Recon Engine & CRTS:**

**Recon Engine - Processing 10M transactions:**
```java
public void reconcileDaily(LocalDate date) {
    List<Transaction> transactions = loadAllTransactions();  // 10M
    
    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        for (Transaction txn : transactions) {
            executor.submit(() -> {
                Optional<Transaction> match = findMatch(txn);
                match.ifPresentOrElse(
                    m -> recordMatch(txn, m),
                    () -> recordUnmatched(txn)
                );
            });
        }
    }  // Wait for all to complete
    
    // Results: Each transaction gets its own virtual thread
    // 10M virtual threads run on maybe 100 platform threads
    // Processing completes in parallel without thread overhead
}
```

**CRTS - Concurrent Message Processing:**
```java
public void processMessagesFromQueue(BlockingQueue<Message> queue) {
    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        while (true) {
            Message message = queue.take();  // Blocks, but on virtual thread
            executor.submit(() -> {
                ProcessingResult result = routeMessage(message);
                acknowledgeMessage(message);
            });
        }
    }
}
```

**Benefits:**
```
Old approach with fixed thread pool (20 threads):
- Each thread 1-2 MB
- 20 threads = 20-40 MB
- Max 20 concurrent operations
- Queuing for remaining messages

New approach with virtual threads:
- Each virtual thread <1 KB
- 100,000 virtual threads = 100 MB (same as 50 platform threads!)
- 100,000 concurrent operations
- No queuing
```

**When to Use Virtual Threads:**
✅ I/O-bound operations (network calls, DB queries, file I/O)  
✅ Many concurrent operations needed (> 10,000)  
✅ Blocking operations (virtual thread suspends, not OS thread)  
✅ High-concurrency servers  

**When NOT to use:**
❌ CPU-intensive operations (use parallel streams instead)  
❌ Long-running calculations  
❌ When holding locks for extended periods  

**Important:** Virtual threads don't give free performance for CPU-bound work. They excel at I/O concurrency.

**Code Example - Comparison:**
```java
// Problem: 1M API calls to different services
List<String> urls = // 1M URLs

// Old: Limited to thread pool size
ExecutorService pool = Executors.newFixedThreadPool(50);
urls.forEach(url -> pool.submit(() -> fetchData(url)));

// New: 1M concurrent virtual threads
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    urls.forEach(url -> executor.submit(() -> fetchData(url)));
}

// Virtual threads complete 20x faster for I/O-bound tasks!
```

**For Fintech Systems Like Ours:**
Virtual threads are transformative. Recon engine could scale to 100M+ transactions daily without infrastructure changes. CRTS could handle 10x message throughput from Charles River without adding servers."

---

## **PART 2: SPRING BOOT & MICROSERVICES QUESTIONS**

### **Q7: Explain Spring Boot Auto-Configuration and how you used it**

**INTERVIEWER ASKS:**
"What is Spring Boot auto-configuration? How does it work and where did you use it?"

**YOUR ANSWER:**

"Spring Boot auto-configuration automatically configures Spring application based on jar dependencies on the classpath. It's the magic behind 'convention over configuration.'

**How It Works:**

1. **@SpringBootApplication** = @Configuration + @EnableAutoConfiguration + @ComponentScan
2. Scans @Conditional annotations on starter classes
3. If conditions match, beans auto-created

**Example - Spring Boot MVC:**
```java
// You have spring-webmvc.jar on classpath?
// You didn't create DispatcherServlet?
// Auto-configuration creates it for you!

@Configuration
@ConditionalOnClass(DispatcherServlet.class)
@ConditionalOnMissingBean(DispatcherServlet.class)
public class WebMvcAutoConfiguration {
    
    @Bean
    public DispatcherServlet dispatcherServlet() {
        return new DispatcherServlet();
    }
}
```

**In Our Projects:**

**Recon Engine - Custom Auto-Configuration:**
```java
@Configuration
@ConditionalOnProperty(name = 'recon.enabled', havingValue = 'true')
@EnableConfigurationProperties(ReconProperties.class)
public class ReconAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public ReconciliationService reconciliationService(
        ReconProperties properties,
        DataSource dataSource
    ) {
        return new ReconciliationService(properties, dataSource);
    }
    
    @Bean
    public ReconciliationScheduler scheduler(ReconciliationService service) {
        return new ReconciliationScheduler(service);
    }
    
    @Bean
    public MetricsCollector metricsCollector() {
        return new MetricsCollector();
    }
}

@ConfigurationProperties(prefix = 'recon')
@Data
public class ReconProperties {
    private int batchSize = 1000;
    private int threadPoolSize = 20;
    private Duration timeout = Duration.ofMinutes(5);
    private String sourceSystem;
    private String targetSystem;
}
```

**application.yml:**
```yaml
recon:
  enabled: true
  batch-size: 5000
  thread-pool-size: 50
  timeout: 10m
  source-system: CHARLES_RIVER
  target-system: ALADDIN
```

**CRTS - Conditional Beans:**
```java
// Only create Kafka producer if Kafka is needed
@Configuration
public class MessagingAutoConfiguration {
    
    @Bean
    @ConditionalOnProperty(name = 'messaging.type', havingValue = 'kafka')
    public KafkaProducer kafkaProducer() {
        return new KafkaProducer();
    }
    
    @Bean
    @ConditionalOnProperty(name = 'messaging.type', havingValue = 'ibm-mq')
    public JmsTemplate ibmMqTemplate() {
        return new JmsTemplate(mqConnectionFactory());
    }
}
```

**Behind the Scenes - What Spring Boot Does:**

1. **Detects:** spring-boot-starter-web on classpath
2. **Auto-configures:**
   - TomcatServletWebServerFactory
   - DispatcherServlet
   - ViewResolver
   - MessageConverter
   - Jackson ObjectMapper
3. You write 0 config for these!

**Customizing Auto-Configuration:**
```java
// Override auto-config if needed
@SpringBootApplication
public class ReconApplication {
    
    @Bean
    public DataSource customDataSource() {
        // My custom DataSource, auto-config uses this instead
        HikariConfig config = new HikariConfig();
        config.setMaximumPoolSize(100);
        return new HikariDataSource(config);
    }
}
```

**Key @Conditional Annotations:**
- @ConditionalOnClass - If class on classpath
- @ConditionalOnMissingBean - If bean not defined
- @ConditionalOnProperty - If property set
- @ConditionalOnWebApplication - If web app
- @ConditionalOnJava - If Java version matches

**Impact:**
- Recon went from 50-page XML config → 10 lines YAML
- CRTS went from manual dependency setup → automatic based on starters
- OPS Portal .NET integration could be toggled via properties

**Pro Tip:** Use `--debug` flag to see which auto-configs are active:
```bash
java -jar app.jar --debug  # Shows auto-config report
```"

---

### **Q8: Explain Microservices Circuit Breaker Pattern using Resilience4j**

**INTERVIEWER ASKS:**
"In a distributed system, how do you handle failures in dependent services? Show me your implementation."

**YOUR ANSWER:**

"Circuit Breaker pattern prevents cascading failures by stopping requests to failing services.

**3 States:**
```
CLOSED (normal) → OPEN (failing) → HALF_OPEN (testing) → CLOSED (recovered)
         ↓              ↓               ↓
    Requests flow    Fast fail    Limited requests
                                  to test recovery
```

**In CRTS - Charles River Integration:**
```java
@Service
@Slf4j
public class CharlesRiverIntegrationService {
    
    // Configure circuit breaker
    private final CircuitBreaker circuitBreaker;
    
    public CharlesRiverIntegrationService() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50.0f)           // Open if 50% requests fail
            .waitDurationInOpenState(Duration.ofSeconds(30))  // Try recovery after 30s
            .slidingWindowSize(100)                // Track last 100 requests
            .minimumNumberOfCalls(10)              // Need 10 calls to evaluate
            .recordExceptions(IOException.class, HttpClientErrorException.class)
            .ignoreExceptions(ValidationException.class)  // Don't count validation errors
            .build();
        
        this.circuitBreaker = CircuitBreaker.of('charles-river', config);
    }
    
    public Trade fetchTrade(String tradeId) {
        return circuitBreaker.executeSupplier(() -> 
            charlesRiverClient.getTrade(tradeId)
        );
    }
    
    // If Charles River is down, circuit opens and this is called
    @CircuitBreaker(name = 'charles-river', fallbackMethod = 'fallbackFetchTrade')
    public Trade fetchTradeWithFallback(String tradeId) {
        log.debug('Charles River down, using cache');
        return tradeCache.get(tradeId)
            .orElseThrow(() -> new ServiceUnavailableException('Charles River down, no cache'));
    }
    
    private Trade fallbackFetchTrade(String tradeId, Exception ex) {
        log.warn('Charles River circuit open for {}', tradeId, ex);
        return tradeCache.get(tradeId)
            .orElseThrow(() -> new ServiceUnavailableException(ex));
    }
}
```

**Monitoring Circuit Breaker:**
```java
@RestController
@RequestMapping('/metrics')
public class CircuitBreakerMetricsController {
    
    @GetMapping('/circuit-breaker/charles-river')
    public CircuitBreakerStatus charlesRiverStatus() {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker('charles-river');
        return CircuitBreakerStatus.builder()
            .state(cb.getState())  // CLOSED, OPEN, HALF_OPEN
            .failureRate(cb.getMetrics().getFailureRate())
            .successRate(cb.getMetrics().getSuccessRate())
            .slowCallRate(cb.getMetrics().getSlowCallRate())
            .callsTotal(cb.getMetrics().getNumberOfTotalCalls())
            .build();
    }
}
```

**Real Example - What Happens:**

**Scenario: Charles River Service is Down**

```
Request 1-5: Charles River returns errors
Request 6-10: More errors
Failure rate: 100% > 50% threshold
↓
Circuit OPENS
↓
Request 11: Circuit intercepted, fallback used (cache)
Request 12-29: Circuit still OPEN, fallback used
After 30 seconds (waitDurationInOpenState):
↓
Circuit HALF_OPEN (testing mode)
↓
Request 30: Test request sent to Charles River
Charles River: NOW WORKING ✅
↓
Circuit CLOSES (back to normal)
```

**Combined with Retry Pattern:**
```java
@Service
public class ReconOrderService {
    
    @Retry(name = 'recon', fallbackMethod = 'fallbackGetOrder')
    @CircuitBreaker(name = 'recon-db', fallbackMethod = 'fallbackGetOrder')
    public Order getOrder(String orderId) {
        return reconciliationService.fetchOrder(orderId);
    }
    
    private Order fallbackGetOrder(String orderId, Exception ex) {
        log.warn('Using fallback for {}', orderId);
        return orderCache.get(orderId)
            .orElse(null);
    }
}
```

**Configuration:**
```properties
# Retry: Retry 3 times with 1 second delay
resilience4j.retry.instances.recon.max-attempts=3
resilience4j.retry.instances.recon.wait-duration=1000

# Circuit Breaker
resilience4j.circuitbreaker.instances.recon-db.failure-rate-threshold=50
resilience4j.circuitbreaker.instances.recon-db.wait-duration-in-open-state=30s
resilience4j.circuitbreaker.instances.recon-db.sliding-window-size=100

# Bulkhead: Max 10 concurrent calls
resilience4j.bulkhead.instances.recon-db.max-concurrent-calls=10
```

**Results from CRTS:**
- When Charles River went down: Request rejected immediately (fast fail)
- Fallback to cache: Users still got data (99.5% availability vs 98% without circuit breaker)
- Auto-recovery: Once Charles River came back, circuit auto-closed
- **Prevented cascading failures** - our service didn't crash when downstream did

**Key Insight:** Circuit breaker isn't about making everything work; it's about **failing fast and gracefully** so the system remains usable even when dependencies fail."

---

### **Q9: Explain REST API Design - What best practices did you implement?**

**INTERVIEWER ASKS:**
"Walk me through your REST API design. Show me your controller and explain the patterns you used."

**YOUR ANSWER:**

"REST (Representational State Transfer) is architectural style for HTTP APIs. Key principles: client-server, stateless, cacheable, uniform interface.

**Best Practices I Implemented:**

1. **Resource-Based URIs (not action-based):**
```
❌ BAD (action-based)
GET /getTrade
POST /createTrade
DELETE /deleteTrade

✅ GOOD (resource-based)
GET /api/v1/trades           # List all trades
GET /api/v1/trades/{id}      # Get specific trade
POST /api/v1/trades          # Create trade
PUT /api/v1/trades/{id}      # Full update
PATCH /api/v1/trades/{id}    # Partial update
DELETE /api/v1/trades/{id}   # Delete trade
```

**In Recon Engine:**
```java
@RestController
@RequestMapping('/api/v1/reconciliation')
@Validated
public class ReconciliationController {
    
    // 1. GET - Retrieve resource
    @GetMapping('/{reconId}')
    public ResponseEntity<ReconciliationResponse> getReconciliation(
        @PathVariable @NotBlank String reconId
    ) {
        return reconciliationService.findById(reconId)
            .map(recon -> ResponseEntity.ok(toResponse(recon)))
            .orElse(ResponseEntity.notFound().build());
    }
    
    // 2. GET - List with pagination & filtering
    @GetMapping
    public ResponseEntity<Page<ReconciliationResponse>> listReconciliations(
        @RequestParam(required = false) String status,
        @RequestParam(required = false) 
        @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate fromDate,
        @RequestParam(defaultValue = '0') int page,
        @RequestParam(defaultValue = '20') int size,
        @PageableDefault(size = 20, sort = 'timestamp', direction = Sort.Direction.DESC) 
        Pageable pageable
    ) {
        Page<Reconciliation> results = reconciliationService.findReconciliations(
            status, 
            fromDate, 
            pageable
        );
        return ResponseEntity.ok(results.map(this::toResponse));
    }
    
    // 3. POST - Create resource
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<ReconciliationResponse> createReconciliation(
        @Valid @RequestBody CreateReconciliationRequest request
    ) {
        Reconciliation recon = reconciliationService.create(request);
        
        // Return 201 with Location header
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path('/{id}')
            .buildAndExpand(recon.getId())
            .toUri();
        
        return ResponseEntity.created(location).body(toResponse(recon));
    }
    
    // 4. PUT - Full update (replace entire resource)
    @PutMapping('/{reconId}')
    public ResponseEntity<ReconciliationResponse> updateReconciliation(
        @PathVariable String reconId,
        @Valid @RequestBody UpdateReconciliationRequest request
    ) {
        return reconciliationService.update(reconId, request)
            .map(recon -> ResponseEntity.ok(toResponse(recon)))
            .orElse(ResponseEntity.notFound().build());
    }
    
    // 5. PATCH - Partial update (update specific fields)
    @PatchMapping('/{reconId}/status')
    public ResponseEntity<ReconciliationResponse> updateStatus(
        @PathVariable String reconId,
        @RequestParam @NotBlank String status
    ) {
        return reconciliationService.updateStatus(reconId, status)
            .map(recon -> ResponseEntity.ok(toResponse(recon)))
            .orElse(ResponseEntity.notFound().build());
    }
    
    // 6. DELETE - Remove resource
    @DeleteMapping('/{reconId}')
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public ResponseEntity<Void> deleteReconciliation(@PathVariable String reconId) {
        reconciliationService.delete(reconId);
        return ResponseEntity.noContent().build();
    }
}
```

**2. Request/Response DTOs with Validation:**
```java
@Data
@Builder
public class CreateReconciliationRequest {
    
    @NotBlank(message = 'Source system required')
    private String sourceSystem;
    
    @NotBlank(message = 'Target system required')
    private String targetSystem;
    
    @NotNull(message = 'Date required')
    @PastOrPresent(message = 'Date cannot be future')
    private LocalDate reconDate;
    
    @Positive(message = 'Batch size must be positive')
    private Integer batchSize;
}

public record ReconciliationResponse(
    String reconId,
    String sourceSystem,
    String targetSystem,
    LocalDate reconDate,
    String status,
    @JsonFormat(pattern = 'yyyy-MM-dd\'T\'HH:mm:ss') LocalDateTime createdAt,
    ReconciliationStatistics statistics
) {}

public record ReconciliationStatistics(
    long totalRecords,
    long matchedCount,
    long unmatchedCount,
    BigDecimal averageAmount,
    BigDecimal maxAmount
) {}
```

**3. HTTP Status Codes:**
```
200 OK - Request successful
201 CREATED - Resource created (POST)
204 NO CONTENT - Success with no response body (DELETE)
400 BAD REQUEST - Client error (validation failed)
401 UNAUTHORIZED - Authentication required
403 FORBIDDEN - Authenticated but not allowed
404 NOT FOUND - Resource doesn't exist
409 CONFLICT - Request conflicts with state
500 INTERNAL SERVER ERROR - Server error
503 SERVICE UNAVAILABLE - Service down
```

**4. Exception Handling:**
```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationError(
        MethodArgumentNotValidException ex
    ) {
        Map<String, String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                error -> Objects.requireNonNullElse(
                    error.getDefaultMessage(), 
                    'Invalid value'
                )
            ));
        
        ErrorResponse response = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.BAD_REQUEST.value())
            .error('Validation Failed')
            .message('Invalid request parameters')
            .validationErrors(errors)
            .build();
        
        return ResponseEntity.badRequest().body(response);
    }
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
        ResourceNotFoundException ex
    ) {
        ErrorResponse response = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.NOT_FOUND.value())
            .error('Not Found')
            .message(ex.getMessage())
            .build();
        
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(response);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGenericError(Exception ex) {
        log.error('Unexpected error', ex);
        
        ErrorResponse response = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
            .error('Internal Server Error')
            .message('An unexpected error occurred')
            .build();
        
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
    }
}
```

**5. API Versioning:**
```java
// v1 endpoint for current version
@RequestMapping('/api/v1/reconciliation')

// When API changes, create v2
@RequestMapping('/api/v2/reconciliation')  // New features
```

**6. Pagination for Large Datasets:**
```java
// Request: GET /api/v1/reconciliation?page=0&size=50&sort=timestamp,desc
// Response:
{
  'content': [...],
  'pageable': {
    'pageNumber': 0,
    'pageSize': 50,
    'totalPages': 200,
    'totalElements': 10000
  }
}
```

**Results in Production:**
- Consistent API contracts → predictable client code
- Proper status codes → easier client error handling  
- Validation at controller → fewer bugs
- Exception handling → consistent error responses
- Versioning → breaking changes don't affect existing clients"

---

## **PART 3: SYSTEM DESIGN QUESTIONS**

### **Q10: Design the Reconciliation Engine - Complete System Architecture**

**INTERVIEWER ASKS:**
"Tell me about a complex system you designed. Walk me through your thought process, architecture, and trade-offs."

**YOUR ANSWER:**

"I built the Reconciliation Engine for daily financial reconciliation of 10M+ transactions. Let me walk through the complete design.

**Requirements Gathering:**

Functional:
- Match transactions from source (Charles River) vs target (Aladdin)
- Identify unmatched transactions
- Generate daily reconciliation report
- Support manual review of discrepancies

Non-Functional:
- Process 10M transactions daily
- Complete within 8 hours
- 99.9% uptime
- 99.9% match accuracy

**High-Level Architecture:**
```
Sources → Ingestion → Processing → Storage → Reporting
(Charles River)    (Validation)    (Matching)  (Postgres)  (API/Dashboard)
(Aladdin)
```

**Detailed Architecture:**
```
┌────────────────────────────────────────────────────┐
│ RECONCILIATION ENGINE SYSTEM DESIGN                │
├────────────────────────────────────────────────────┤
│                                                    │
│  Input Layer                                       │
│  ├─ Charles River (Source)                        │
│  ├─ Aladdin System (Target)                       │
│  └─ Internal Systems                              │
│        ↓                                            │
│  Ingestion Layer (Kafka Consumers)                 │
│  ├─ Validate transactions                          │
│  ├─ Transform to standard format                   │
│  └─ Publish to processing topics                   │
│        ↓                                            │
│  Processing Layer (Parallel Workers)               │
│  ├─ Partition by account (sharding key)            │
│  ├─ Apply matching rules                           │
│  │  ├─ Exact match (90%)                           │
│  │  ├─ Fuzzy match (8%)                            │
│  │  └─ Manual review (2%)                          │
│  ├─ Circuit breaker for external calls             │
│  └─ Publish results to Kafka                       │
│        ↓                                            │
│  Storage Layer (Postgres + Redis)                  │
│  ├─ Write reconciliation results                   │
│  ├─ Cache hot data (Redis)                         │
│  └─ Maintain audit trail                           │
│        ↓                                            │
│  Reporting Layer (REST API + Dashboard)            │
│  ├─ Real-time metrics                              │
│  ├─ Daily reports                                  │
│  └─ Exception handling UI                          │
│                                                    │
└────────────────────────────────────────────────────┘
```

**Data Flow - Processing 10M Transactions:**

```
Day 1: 00:00 UTC
├─ Charles River sends 5M transactions
├─ Aladdin sends 5M transactions
│
├─ Step 1: Ingestion (1 hour)
│  ├─ Kafka consumers validate & transform
│  ├─ Store raw transactions in batches
│  └─ Publish to processing topics
│
├─ Step 2: Partition by account (1 hour)
│  ├─ Account A: [TXN1, TXN2, ...]
│  ├─ Account B: [TXN5, TXN10, ...]
│  └─ Account Z: [...]
│
├─ Step 3: Parallel Matching (4 hours)
│  ├─ Worker 1: Process Account A
│  ├─ Worker 2: Process Account B
│  ├─ ...
│  └─ Worker 100: Process Account Z
│
├─ Step 4: Results Write (1 hour)
│  └─ Store matched/unmatched/exceptions
│
├─ Step 5: Report Generation (30 mins)
│  └─ Aggregate stats, generate daily report
│
└─ Day 1: 08:00 UTC - Reconciliation Complete!
```

**Technology Stack:**

1. **Ingestion:**
   - Kafka for decoupling data sources
   - 3 topics: source, target, processed
   - 10 partitions each (parallelism)

2. **Processing:**
   - Java 17 + Spring Boot
   - Parallel streams for CPU-intensive matching
   - CompletableFuture for async I/O
   - Circuit breaker (Resilience4j)
   - Thread pool: 100 worker threads

3. **Storage:**
   - PostgreSQL (reconciliation results)
   - Redis (caching, hot data)
   - S3 (archived results)

4. **Monitoring:**
   - Prometheus (metrics)
   - Grafana (dashboards)
   - ELK (logging)

**Matching Algorithms:**

```
Transaction matching hierarchy:

1. EXACT MATCH (90% of cases)
   Amount = Source Amount
   Account = Source Account  
   Date = Source Date
   ↓ MATCH (Automated)

2. FUZZY MATCH (8% of cases)
   Amount difference < $0.01
   Account matches
   Date difference < 1 day
   ↓ MATCH (Automated)

3. TOLERANCE MATCH (1% of cases)
   Amount within 0.1% tolerance
   Requires manual review
   ↓ PARTIAL MATCH (Human Review)

4. NO MATCH (1% of cases)
   Could not match within parameters
   ↓ EXCEPTION (Manual Investigation)
```

**Scalability Strategy:**

```
Current: 10M transactions/day = 115 TPS
Can horizontally scale by:

1. Adding Kafka partitions (increase from 10 to 50)
2. Adding worker pods (100 workers → 500 workers)
3. Adding database replicas for reads
4. Sharding by account hash instead of single DB

Result: Can handle 100M+ transactions with same infrastructure
```

**Fault Tolerance:**

```
Scenario 1: Charles River API down
├─ Circuit breaker opens
├─ Fallback to cached yesterday's data
├─ Mark transactions for manual review
└─ Result: 95% automatic match, 5% manual

Scenario 2: Database connection pool exhausted
├─ Queue overflows to Kafka  
├─ Kafka retains data for 7 days
├─ Once DB recovers, process backlog
└─ Result: No data loss

Scenario 3: Worker crashes
├─ Kubernetes restart pod
├─ Consumer rebalance
├─ Reprocess from offset
└─ Result: Automatic recovery in <1 minute
```

**Performance Optimization:**

1. **35% Latency Reduction:**
   - Migrated from Java 8 → Java 17
   - Switched ParallelGC → G1GC
   - Tuned heap from 4GB → 8GB
   - Reduced GC pause from 800ms → 200ms

2. **40% Throughput Increase:**
   - Batch put in Kafka (100 msgs/batch)
   - Connection pooling for MQ
   - Prepared statements in DB
   - Caching frequently accessed reference data

3. **Infrastructure Cost:**
   - Before: 15 servers (processing ran 12 hours)
   - After: 5 servers (processing runs 7 hours)
   - Cost savings: 67%

**API Endpoints:**

```
GET /api/v1/reconciliation/{date} - Get daily report
GET /api/v1/reconciliation/{date}/matches - Matched transactions
GET /api/v1/reconciliation/{date}/unmatched - Unmatched transactions
GET /api/v1/reconciliation/{date}/exceptions - Exceptions for manual review
POST /api/v1/reconciliation/{date}/exceptions/{id}/resolve - Manual resolution
GET /api/v1/reconciliation/metrics - Real-time dashboard metrics
```

**Trade-offs:**

```
Design Decision: Eventually Consistent vs Strongly Consistent
├─ Chose: Eventually Consistent
├─ Reason: Must match 10M transactions in 8 hours
├─ If Strongly Consistent: Would take 3 days
├─ Result: 99.9% accuracy acceptable in financial domain

Design Decision: Horizontal Scaling vs Vertical Scaling
├─ Chose: Horizontal
├─ Reason: Better for cloud, easier failover
├─ Cost: More infrastructure management
├─ Benefit: Auto-scaling, fault isolation

Design Decision: Cache Invalidation
├─ Chose: TTL (1 hour) + Event-based (when data changes)
├─ Alternative: Cache-aside only (simpler but stale data)
├─ Result: 60% cache hit rate, strong consistency
```

**In Production - 2 Years:**
- Processed 7+ billion transactions
- 99.98% match accuracy
- Zero data loss incidents
- Auto-recovered from 15+ Charles River outages
- 95% of exceptions auto-resolved via fuzzy matching

This system taught me how to balance:
- Throughput vs latency
- Consistency vs availability  
- Scalability vs complexity
- Automation vs manual handling"

---

## **PART 4: IBM MQ & ORACLE QUESTIONS**

### **Q11: Explain IBM MQ Architecture and SSL Implementation**

**INTERVIEWER ASKS:**
"Tell me about message queuing systems. How did you implement IBM MQ with SSL in your reconciliation engine?"

**YOUR ANSWER:**

"IBM MQ is an enterprise message broker. Queue managers store messages durably until consumers process them.

**Architecture:**
```
Application 1 → Channel → Queue Manager 1 ← Channel ← Application 2
                  (connection)    ↓          (connection)
                              Queue Storage
                              (persistent)
```

**In Recon Engine - SSL Setup:**

```java
@Configuration
public class IbmMqConfig {
    
    @Bean
    public MQConnectionFactory mqConnectionFactory() throws JMSException {
        MQConnectionFactory factory = new MQConnectionFactory();
        factory.setHostName('mq.charlesriver.com');
        factory.setPort(1414);
        factory.setQueueManager('RECON_QM1');
        factory.setChannel('RECON.SVRCONN');
        factory.setTransportType(WMQConstants.WMQ_CM_CLIENT);
        
        // SSL/TLS Configuration
        factory.setSSLCipherSuite('TLS_RSA_WITH_AES_256_CBC_SHA256');
        factory.setSSLFipsRequired(false);
        
        // Connection pooling
        factory.setClientReconnectOptions(WMQConstants.WMQ_CLIENT_RECONNECT);
        factory.setClientReconnectTimeout(1800);  // 30 minutes
        
        return factory;
    }
}
```

**SSL Setup (Command Line):**
```bash
# 1. Create Queue Manager
crtmqm RECON_QM1

# 2. Start Queue Manager
strmqm RECON_QM1

# 3. Create Key Repository (certificate store)
runmqakm -keydb -create \
    -db /var/mqm/qmgrs/RECON_QM1/ssl/key.kdb \
    -pw MyPassword123 \
    -type cms \
    -stash

# 4. Create Personal Certificate (server cert)
runmqakm -cert -create \
    -db /var/mqm/qmgrs/RECON_QM1/ssl/key.kdb \
    -pw MyPassword123 \
    -label ibmwebspheremqrecon_qm1 \
    -dn 'CN=RECON_QM1,O=YourCompany,C=IN' \
    -size 2048 \
    -expire 365

# 5. Configure Queue Manager for SSL
runmqsc RECON_QM1 << EOF
ALTER QMGR SSLKEYR('/var/mqm/qmgrs/RECON_QM1/ssl/key')
REFRESH SECURITY TYPE(SSL)
EOF

# 6. Create SSL Channel
runmqsc RECON_QM1 << EOF
DEFINE CHANNEL('RECON.SVRCONN') +
    CHLTYPE(SVRCONN) +
    SSLCIPH('TLS_RSA_WITH_AES_256_CBC_SHA256') +
    SSLCAUTH(REQUIRED) +
    MCAUSER('reconapp')
EOF

# 7. Create Queues
runmqsc RECON_QM1 << EOF
DEFINE QLOCAL('RECON.INCOMING') MAXDEPTH(100000) MAXMSGL(4194304)
DEFINE QLOCAL('RECON.PROCESSING')
DEFINE QLOCAL('RECON.COMPLETED')
DEFINE QLOCAL('RECON.DLQ')  // Dead Letter Queue
EOF
```

**Java Message Sending:**
```java
@Service
@Slf4j
public class ReconMessageService {
    
    private final JmsTemplate jmsTemplate;
    private final ReconProperties props;
    
    // Send reconciliation request to queue
    public void sendReconciliationRequest(ReconciliationRequest request) {
        jmsTemplate.send('RECON.INCOMING', session -> {
            TextMessage message = session.createTextMessage(
                objectMapper.writeValueAsString(request)
            );
            
            // Metadata for routing
            message.setJMSCorrelationID(request.getTransactionId());
            message.setIntProperty('PRIORITY', 5);
            message.setStringProperty('MESSAGE_TYPE', 'RECON_REQUEST');
            message.setLongProperty('TIMESTAMP', System.currentTimeMillis());
            
            // Persistent = survive queue manager restart
            message.setJMSDeliveryMode(DeliveryMode.PERSISTENT);
            
            log.info('Sent recon request: {}', request.getTransactionId());
            return message;
        });
    }
    
    // Listen for messages (consumer)
    @JmsListener(
        destination = 'RECON.PROCESSING',
        concurrency = '10-20',  // 10-20 concurrent listeners
        containerFactory = 'jmsListenerContainerFactory'
    )
    public void processReconciliation(Message message) throws JMSException {
        if (message instanceof TextMessage textMsg) {
            String body = textMsg.getText();
            String correlationId = textMsg.getJMSCorrelationID();
            
            try {
                ReconciliationRequest request = 
                    objectMapper.readValue(body, ReconciliationRequest.class);
                
                ReconciliationResult result = reconciliationEngine.process(request);
                
                // Send result to completion queue
                sendResult(result, correlationId);
                
                // Acknowledge message (remove from queue)
                message.acknowledge();
                
                log.info('Processed: {}', correlationId);
                
            } catch (Exception e) {
                log.error('Processing failed: {}', correlationId, e);
                // Send to Dead Letter Queue (DLQ) for manual review
                sendToDeadLetterQueue(body, correlationId, e.getMessage());
            }
        }
    }
}
```

**Key MQ Concepts:**

1. **Local Queue** - Messages stored here
2. **Remote Queue** - Pointer to queue on another QM
3. **Alias Queue** - Logical name
4. **Dead Letter Queue** - Failed messages stored here
5. **Channel** - Connection between Queue Managers
6. **Listener** - Monitors queue for messages

**Performance Tuning (40% Throughput Increase):**

```properties
# Connection Pooling
jms.cache.connection=true
jms.cache.session=true
jms.session.cache.size=50

# Batch Processing
mq.batch.size=100           # Batch 100 messages before commit
mq.batch.timeout=5000       # Or after 5 seconds

# Thread Pool
jms.listener.concurrency=10-20
jms.listener.max-concurrency=50

# Message Persistence
# Persistent messages = survive crashes (slower)
# Non-persistent = lost if QM restarts (faster)
mq.delivery.mode.audit=PERSISTENT        # Audit trail
mq.delivery.mode.status=NON_PERSISTENT   # Status updates
```

**Production Results:**
- 35% latency reduction through Java 17 migration
- 40% throughput increase through tuning (batch sizing, connection pooling)
- Zero message loss (persistent mode + backup QM)
- 99.9% availability (automatic failover)"

---

### **Q12: Explain Oracle Database Performance Tuning**

**INTERVIEWER ASKS:**
"You mentioned Oracle tuning in your reconciliation engine. Walk me through your optimization strategy."

**YOUR ANSWER:**

"Oracle tuning involves query optimization, indexing, and partitioning. Our Recon engine stores 10M transactions daily - efficient queries are critical.

**Problem:** 
Initial daily reconciliation report took 30 minutes to run. Business needed it in <5 minutes.

**Solution - Multi-layered Approach:**

**1. Analyze Query Plan:**
```sql
-- Before (30 seconds for 1M rows)
SELECT t.transaction_id, t.amount, t.account_id, a.account_name
FROM transactions t, accounts a
WHERE t.account_id = a.account_id
  AND t.transaction_date >= TRUNC(SYSDATE) - 30
  AND t.status = 'PENDING'
ORDER BY t.transaction_date DESC;

-- Explain Plan shows:
-- TABLE ACCESS (FULL) on transactions - BAD! (scanning all 500M rows)
-- NESTED LOOP with accounts
```

**2. Add Indexes:**
```sql
-- Composite index on (date, status, account_id)
CREATE INDEX idx_recon_key ON transactions(
    transaction_date,
    status,
    account_id
) PARALLEL 4;

-- Function-based index for amounts
CREATE INDEX idx_amount_rounded ON transactions(
    ROUND(amount, 2)
);

-- Gather statistics so optimizer knows about index
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname => 'RECON_SCHEMA',
        tabname => 'TRANSACTIONS',
        estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
        method_opt => 'FOR ALL COLUMNS SIZE AUTO',
        cascade => TRUE
    );
END;
/
```

**3. Rewrite Query (Explicit Join, Hints):**
```sql
-- After (same query: 2 seconds for 1M rows)
SELECT /*+ PARALLEL(t, 4) INDEX(t idx_recon_key) */
       t.transaction_id, 
       t.amount, 
       t.account_id, 
       a.account_name
FROM transactions t
INNER JOIN accounts a ON t.account_id = a.account_id
WHERE TRUNC(t.transaction_date) >= TRUNC(SYSDATE) - 30
  AND t.status = 'PENDING'
ORDER BY t.transaction_date DESC;

-- Optimizations:
-- 1. INNER JOIN instead of comma join (forces index use)
-- 2. INDEX hint tells optimizer to use our index
-- 3. PARALLEL(4) scans 4 partitions simultaneously
-- 4. TRUNC() pushdown reduces full scans
```

**4. Partitioning (Huge for large tables):**
```sql
-- Drop old table if exists
DROP TABLE transactions;

-- Recreate with daily partitioning
CREATE TABLE transactions (
    transaction_id VARCHAR2(50),
    transaction_date DATE,
    amount NUMBER(15,2),
    status VARCHAR2(20),
    account_id VARCHAR2(50)
)
PARTITION BY RANGE (transaction_date) 
INTERVAL (NUMTODSINTERVAL(1, 'DAY'))  -- Auto-create daily partitions
(
    PARTITION p_history VALUES LESS THAN (TO_DATE('2024-01-01', 'YYYY-MM-DD'))
)
TABLESPACE RECON_DATA;

-- Now query only relevant partitions!
-- Query for Oct 2024 only scans October partitions, not entire table
```

**Results:**
```
Before Optimization:
├─ Full table scan: 500M rows scanned
├─ Nested loop join: 500M * 10M iterations
├─ Time: 30 minutes
└─ Cost: High CPU, disk I/O

After Optimization:
├─ Index scan: Only 1M rows read
├─ Hash join: 1M * indexed lookup
├─ Partition pruning: Oct data in Oct partition only
├─ Time: 2 minutes (93% improvement!)
└─ Cost: Minimal CPU, sequential reads
```

**Java Side Optimization (JDBC):**

```java
@Repository
public class TransactionRepository {
    
    // Batch insert for nightly reconciliation load
    public void batchInsertTransactions(List<Transaction> transactions) {
        String sql = 'INSERT INTO transactions (id, amount, account_id, date, status) VALUES (?, ?, ?, ?, ?)';
        
        jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                Transaction t = transactions.get(i);
                ps.setString(1, t.getId());
                ps.setBigDecimal(2, t.getAmount());
                ps.setString(3, t.getAccountId());
                ps.setDate(4, java.sql.Date.valueOf(t.getDate()));
                ps.setString(5, t.getStatus());
            }
            
            @Override
            public int getBatchSize() {
                return 1000;  // Batch size of 1000
            }
        });
    }
    
    // Optimized query for reconciliation
    @Query(nativeQuery = true, value = '''
        SELECT /*+ PARALLEL(4) INDEX(t idx_recon_key) */
               t.transaction_id, t.amount, t.account_id, a.account_name
        FROM transactions t
        INNER JOIN accounts a ON t.account_id = a.account_id
        WHERE TRUNC(t.transaction_date) >= :fromDate
          AND t.status = 'PENDING'
        ORDER BY t.transaction_date DESC
    ''')
    List<Transaction> findPendingTransactionsSince(
        @Param('fromDate') LocalDate fromDate
    );
}
```

**Monitoring & Maintenance:**

```sql
-- Monitor slow queries
SELECT * FROM v\$sql WHERE elapsed_time > 60000000;  -- > 60 seconds

-- Index usage statistics
SELECT * FROM v\$object_usage WHERE used = 'N';  -- Unused indexes

-- Table size
SELECT segment_name, bytes/1024/1024 MB FROM dba_segments 
WHERE segment_type = 'TABLE' ORDER BY bytes DESC;

-- Rebuild fragmented indexes
ALTER INDEX idx_recon_key REBUILD ONLINE;

-- Update statistics periodically
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(ownname => 'RECON_SCHEMA');
END;
/
```

**Production Results:**
- Daily report: 30 min → 2 min (93% improvement)
- Nightly batch load: 2 hours → 15 minutes
- Query throughput: 230 TPS → 400 TPS (40% increase)
- No changes to application code, just database optimization!"

---

## **PART 5: ANGULAR & .NET QUESTIONS**

(Due to length, I'll provide the key sections)

### **Q13: Explain Angular 20 and your OPS Portal implementation**

**INTERVIEWER ASKS:**
"Tell me about your Angular work on the OPS Portal. What are reactive forms and RxJS?"

**YOUR ANSWER:**

"Angular is a comprehensive framework for building reactive, single-page applications.

**OPS Portal Architecture:**
- **Frontend:** Angular 20 for dashboard UI
- **Backend:** .NET + Java microservices
- **State Management:** RxJS Observables
- **Forms:** Reactive forms with validation

**Reactive Forms (Complex Form Handling):**
```typescript
// Instead of two-way binding (template-driven), 
// we define form in TypeScript (reactive)

import { Component } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-trade-form',
  template: `
    <form [formGroup]='tradeForm' (ngSubmit)='onSubmit()'>
      
      <label>Trade ID</label>
      <input formControlName='tradeId' />
      <div *ngIf='tradeForm.get("tradeId")?.invalid && tradeForm.get("tradeId")?.touched'>
        Trade ID is required
      </div>
      
      <label>Amount</label>
      <input type='number' formControlName='amount' />
      
      <button [disabled]='!tradeForm.valid'>Submit</button>
    </form>
  `
})
export class TradeFormComponent {
  tradeForm: FormGroup;
  
  constructor(private fb: FormBuilder) {
    this.tradeForm = this.fb.group({
      tradeId: ['', [Validators.required, Validators.pattern(/^TRD[0-9]{6}$/)]],
      amount: ['', [Validators.required, Validators.min(0.01)]],
      currency: ['USD', Validators.required]
    });
  }
  
  onSubmit() {
    if (this.tradeForm.valid) {
      console.log(this.tradeForm.value);
      this.tradeService.saveTrade(this.tradeForm.value).subscribe(
        result => console.log('Saved:', result),
        error => console.error('Error:', error)
      );
    }
  }
}
```

**RxJS Observables (Event Streams):**
```typescript
// Observable = stream of events over time

this.route.params.subscribe(params => {
  // Triggered when route params change
  const id = params['id'];
});

this.searchInput.valueChanges
  .pipe(
    debounceTime(300),  // Wait 300ms after user stops typing
    distinctUntilChanged(),  // Only if value changed
    switchMap(search => this.api.search(search))  // Switch to new request
  )
  .subscribe(results => {
    this.searchResults = results;
  });

// Combining multiple observables
import { combineLatest } from 'rxjs';

combineLatest([
  this.userService.getCurrentUser(),
  this.accountService.getAccounts(),
  this.tradeService.getTrades()
]).subscribe(([user, accounts, trades]) => {
  // All 3 loaded, update dashboard
  this.user = user;
  this.accounts = accounts;
  this.trades = trades;
});
```

**HTTP Interceptor (Global Error Handling):**
```typescript
@Injectable()
export class ErrorInterceptor implements HttpInterceptor {
  
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(req).pipe(
      catchError(error => {
        if (error.status === 401) {
          // Unauthorized - redirect to login
          this.router.navigate(['/login']);
        } else if (error.status === 403) {
          // Forbidden
          this.notificationService.showError('Access denied');
        } else if (error.status === 500) {
          // Server error
          this.notificationService.showError('Server error occurred');
        }
        return throwError(() => error);
      })
    );
  }
}
```

**Lazy Loading (Performance):**
```typescript
// app-routing.module.ts
const routes: Routes = [
  {
    path: 'dashboard',
    component: DashboardComponent,
    data: { title: 'Dashboard' }
  },
  {
    path: 'trades',
    loadChildren: () => import('./trades/trades.module').then(m => m.TradesModule)
    // TradesModule only loaded when user navigates to /trades
  },
  {
    path: 'reconciliation',
    loadChildren: () => import('./reconciliation/reconciliation.module')
      .then(m => m.ReconciliationModule)
  }
];

// trades.module.ts
@NgModule({
  declarations: [TradeListComponent, TradeDetailComponent],
  imports: [CommonModule, TradesRoutingModule, HttpClientModule]
})
export class TradesModule { }
```

**Results:**
- OPS Portal loads in < 2 seconds (lazy loading + minification)
- Real-time trade updates via WebSocket + Observables
- Responsive dashboard with 20+ metrics
- 100% type-safe (TypeScript)"

---

### **Q14: .NET Core and Entity Framework Core**

**INTERVIEWER ASKS:**
"You mentioned .NET for OPS Portal. Explain Entity Framework Core and why it's useful."

**YOUR ANSWER:**

"Entity Framework Core is an ORM (Object-Relational Mapper) that maps C# objects to database tables.

**LINQ Queries (Type-Safe SQL):**
```csharp
// Traditional SQL (string-based, error-prone)
string sql = "SELECT * FROM trades WHERE amount > @amount AND status = @status";

// Entity Framework (type-safe)
var trades = dbContext.Trades
    .Where(t => t.Amount > 1000000 && t.Status == "PENDING")
    .OrderByDescending(t => t.CreatedDate)
    .ToList();
```

**Relationships (Automatic Joins):**
```csharp
public class Trade
{
    public string Id { get; set; }
    public string AccountId { get; set; }
    public Account Account { get; set; }  // Navigation property
}

public class Account
{
    public string Id { get; set; }
    public List<Trade> Trades { get; set; }  // One-to-many
}

// Query with automatic join
var tradesWithAccounts = dbContext.Trades
    .Include(t => t.Account)  // Loads related account
    .ToList();
```

**Lazy vs Eager Loading:**
```csharp
// Lazy Loading (N+1 problem!)
var trades = dbContext.Trades.ToList();
foreach (var trade in trades) {
    // Each access loads account separately (10K trades = 10K queries!)
    var account = trade.Account;
}

// Eager Loading (optimal)
var trades = dbContext.Trades
    .Include(t => t.Account)  // Load in single query with JOIN
    .ToList();
```

**Change Tracking (Dirty Checking):**
```csharp
var trade = dbContext.Trades.Find(id);
trade.Amount = 1000000;  // Modify
dbContext.SaveChanges();  // Automatically generates UPDATE statement
```

**In OPS Portal:**
```csharp
[ApiController]
[Route("api/[controller]")]
public class TradesController : ControllerBase
{
    private readonly ITradeService _tradeService;
    
    public TradesController(ITradeService tradeService)
    {
        _tradeService = tradeService;
    }
    
    [HttpGet("{id}")]
    public async Task<ActionResult<TradeDto>> GetTrade(string id)
    {
        var trade = await _tradeService.GetTradeAsync(id);
        if (trade == null)
            return NotFound();
        return Ok(trade);
    }
    
    [HttpPost]
    public async Task<ActionResult<TradeDto>> CreateTrade(CreateTradeRequest request)
    {
        var trade = await _tradeService.CreateTradeAsync(request);
        return CreatedAtAction(nameof(GetTrade), new { id = trade.Id }, trade);
    }
}

public class TradeService
{
    private readonly TradeDbContext _context;
    
    public TradeService(TradeDbContext context)
    {
        _context = context;
    }
    
    public async Task<Trade> GetTradeAsync(string id)
    {
        return await _context.Trades
            .Include(t => t.Account)
            .Include(t => t.Allocations)
            .FirstOrDefaultAsync(t => t.Id == id);
    }
    
    public async Task<Trade> CreateTradeAsync(CreateTradeRequest request)
    {
        var trade = new Trade
        {
            Id = Guid.NewGuid().ToString(),
            Amount = request.Amount,
            Currency = request.Currency,
            CreatedDate = DateTime.UtcNow
        };
        
        _context.Trades.Add(trade);
        await _context.SaveChangesAsync();  // INSERT statement
        return trade;
    }
}
```

**Results:**
- Reduced data access code by 70% (vs. ADO.NET)
- Zero SQL injection (parameterized queries)
- Automatic change tracking prevents manual update bugs"

---

## **PART 6: DEVOPS & CLOUD QUESTIONS**

### **Q15: Explain Docker and Kubernetes deployment**

**INTERVIEWER ASKS:**
"Tell me about containerization and orchestration. How did you deploy the Recon Engine?"

**YOUR ANSWER:**

"Docker packages applications with dependencies into containers. Kubernetes orchestrates containers across a cluster.

**Docker Multi-Stage Build (Recon Engine):**
```dockerfile
# Stage 1: Build Java application
FROM maven:3.9-amazoncorretto-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline

COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Runtime (smaller, production image)
FROM amazoncorretto:17-alpine
WORKDIR /app

# Security: Non-root user
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

# Copy JAR from build stage
COPY --from=build /app/target/recon-engine.jar app.jar

# JVM optimization
ENV JAVA_OPTS='-Xms2G -Xmx2G -XX:+UseG1GC -XX:MaxGCPauseMillis=200'

EXPOSE 8080

ENTRYPOINT ['sh', '-c', 'java $JAVA_OPTS -jar app.jar']
```

**Results:**
- Build stage: 500 MB (discarded)
- Final image: 200 MB (production)
- Cold start: 3 seconds
- Memory: 2GB heap

**Kubernetes Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recon-engine
  namespace: production
spec:
  replicas: 3  # 3 pods for high availability
  strategy:
    type: RollingUpdate  # Gradual replacement, no downtime
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  
  selector:
    matchLabels:
      app: recon-engine
      version: v1
  
  template:
    metadata:
      labels:
        app: recon-engine
        version: v1
    
    spec:
      containers:
      - name: recon-engine
        image: registry.company.com/recon-engine:v1.2.0
        imagePullPolicy: Always
        
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        
        # Resource requests & limits
        resources:
          requests:
            memory: '2Gi'
            cpu: '1000m'
          limits:
            memory: '4Gi'
            cpu: '2000m'
        
        # Environment variables
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: 'prod'
        - name: DB_URL
          valueFrom:
            configMapKeyRef:
              name: recon-config
              key: db.url
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: recon-secrets
              key: db-password
        
        # Health checks
        livenessProbe:  # Is app running?
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        
        readinessProbe:  # Is app ready to accept traffic?
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        # Volume mounts for config files
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
      
      # Node affinity (schedule on specific nodes)
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: app-type
                operator: In
                values:
                - fintech
      
      # Toleration (allow scheduling on tainted nodes)
      tolerations:
      - key: high-memory
        operator: Equal
        value: 'true'
        effect: NoExecute

      volumes:
      - name: config
        configMap:
          name: recon-config

---
# Service (Load balancer for pods)
apiVersion: v1
kind: Service
metadata:
  name: recon-engine-service
  namespace: production
spec:
  type: LoadBalancer  # External load balancer
  selector:
    app: recon-engine
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http

---
# HorizontalPodAutoscaler (Auto-scaling)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: recon-engine-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: recon-engine
  
  minReplicas: 3
  maxReplicas: 10
  
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale up if CPU > 70%
  
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80  # Scale up if Memory > 80%
```

**Deployment Process:**
```
1. Build image: docker build -t recon-engine:v1.2.0 .
2. Push to registry: docker push registry.company.com/recon-engine:v1.2.0
3. Update K8s: kubectl apply -f deployment.yaml
4. Kubernetes automatically:
   - Pulls image
   - Starts pods
   - Runs health checks
   - Directs traffic to ready pods
   - Auto-scales if CPU/memory high
   - Restarts failed pods
```

**Production Results:**
- Deployment time: 5 minutes (previously: 2 hours manual)
- Downtime during updates: 0 seconds (rolling update)
- Auto-recovery: Failed pod restarted in <1 minute
- Scaling: Added 3 pods in 2 minutes to handle demand spike
- Cost: 40% reduction (better resource utilization)"

---

---

## **FINAL TIPS FOR INTERVIEWER SUCCESS:**

1. **Always connect answers to YOUR projects:**
   - "In the Recon Engine..."
   - "For CRTS integration..."
   - "In OPS Portal..."

2. **Lead with metrics:**
   - "35% latency reduction"
   - "40% throughput increase"
   - "99.9% accuracy"

3. **Explain trade-offs:**
   - "We chose eventually consistent over strongly consistent because..."
   - "Horizontal vs vertical scaling: we chose horizontal because..."

4. **Show problem-solving:**
   - "The problem was..."
   - "We analyzed using..."
   - "We implemented..."
   - "Results were..."

5. **Demonstrate ownership:**
   - "I designed", "I optimized", "I implemented"
   - Not: "The team did"

6. **Prepare for follow-ups:**
   - "What if Charles River was down?" → Circuit breaker
   - "How would you scale to 100M transactions?" → Sharding
   - "What about security?" → SSL/TLS, encryption, authentication

---

**You're now equipped with comprehensive Q&A covering:**
✅ Java 8-21  
✅ Spring Boot & Microservices  
✅ System Design  
✅ IBM MQ  
✅ Oracle Tuning  
✅ Kafka, Redis  
✅ Angular & .NET  
✅ Docker & Kubernetes  
✅ DevOps & Cloud  

**Practice these answers, internalize the patterns, and you'll ace any interview!** 🚀