# Interview[java-backend-complete-prep.md](https://github.com/user-attachments/files/23225504/java-backend-complete-prep.md)
# Complete Java Backend Developer Interview Preparation Guide
## For Remote Backend Developer Roles | 3+ Years Experience

**Prepared for:** Aman Raj  
**Target Companies:** JP Morgan, Goldman Sachs, BlackRock, PayPal, Razorpay, Groww, Zerodha, CRED  
**Target Compensation:** 22-25 LPA  
**Focus:** Fintech, BFSI, High-Performance Systems

---

## TABLE OF CONTENTS

1. [Core Java (8/11/17/21) - Deep Dive](#core-java)
2. [Spring Boot & Microservices Architecture](#spring-boot-microservices)
3. [System Design - Fintech Focused](#system-design)
4. [IBM MQ - Complete Guide](#ibm-mq)
5. [Oracle Database Performance Tuning](#oracle-tuning)
6. [Cloud & DevOps (AWS, Azure, Jenkins, Docker, K8s)](#cloud-devops)
7. [Angular & .NET Basics](#angular-dotnet)
8. [Data Structures & Algorithms - 50 Problems](#dsa-problems)
9. [Behavioral Interview - STAR Stories](#behavioral)
10. [Salary Negotiation Scripts](#salary-negotiation)
11. [Study Plans (7/21/60 Days)](#study-plans)

---

## 1. CORE JAVA (8/11/17/21) - DEEP DIVE {#core-java}

### 1.1 Java 8 Features - The Foundation

#### **Streams API - Complete Mastery**

**What are Streams?**
Streams represent a sequence of elements supporting sequential and parallel aggregate operations. They are NOT data structures but take input from Collections, Arrays, or I/O channels.

**Key Characteristics:**
- **Lazy evaluation**: Intermediate operations are not executed until a terminal operation is invoked
- **Functional in nature**: Operations don't modify the source
- **Possibly unbounded**: Can work with infinite sequences
- **Consumable**: Can only be traversed once

**Stream Pipeline Structure:**
```java
// Source → Intermediate Operations → Terminal Operation
List<String> result = list.stream()           // Source
    .filter(s -> s.length() > 5)              // Intermediate
    .map(String::toUpperCase)                  // Intermediate
    .collect(Collectors.toList());            // Terminal
```

**Critical Stream Operations - With Real-World Examples:**

```java
// 1. FILTER - Selecting elements based on predicate
// Use Case: Filter trades above threshold in Recon
List<Trade> highValueTrades = trades.stream()
    .filter(trade -> trade.getAmount() > 1_000_000)
    .filter(trade -> trade.getStatus().equals("PENDING"))
    .collect(Collectors.toList());

// 2. MAP - Transform each element
// Use Case: Extract account IDs from trade objects
List<String> accountIds = trades.stream()
    .map(Trade::getAccountId)
    .distinct()
    .collect(Collectors.toList());

// 3. FLATMAP - Flatten nested structures
// Use Case: Get all securities from multiple portfolios
List<Security> allSecurities = portfolios.stream()
    .flatMap(portfolio -> portfolio.getSecurities().stream())
    .collect(Collectors.toList());

// 4. REDUCE - Aggregate to single value
// Use Case: Calculate total transaction amount
Double totalAmount = transactions.stream()
    .map(Transaction::getAmount)
    .reduce(0.0, Double::sum);
    
// Alternative using method reference
Double total = transactions.stream()
    .map(Transaction::getAmount)
    .reduce(0.0, (a, b) -> a + b);

// 5. COLLECT - Powerful terminal operation
// Group trades by status
Map<String, List<Trade>> tradesByStatus = trades.stream()
    .collect(Collectors.groupingBy(Trade::getStatus));

// Partition into two groups
Map<Boolean, List<Trade>> partitioned = trades.stream()
    .collect(Collectors.partitioningBy(t -> t.getAmount() > 100000));

// Custom collector - count trades per account
Map<String, Long> tradesPerAccount = trades.stream()
    .collect(Collectors.groupingBy(
        Trade::getAccountId,
        Collectors.counting()
    ));

// 6. SORTED - Sort elements
List<Trade> sortedTrades = trades.stream()
    .sorted(Comparator.comparing(Trade::getTimestamp).reversed())
    .limit(10)
    .collect(Collectors.toList());

// 7. PEEK - Debug intermediate operations
List<Trade> processed = trades.stream()
    .peek(t -> log.debug("Before filter: {}", t))
    .filter(t -> t.getAmount() > 1000)
    .peek(t -> log.debug("After filter: {}", t))
    .collect(Collectors.toList());

// 8. PARALLEL STREAMS - For large datasets
// Use Case: Process 10M reconciliation records
long matchCount = reconciliationRecords.parallelStream()
    .filter(this::matchesWithSourceSystem)
    .count();

// WARNING: Only use parallel streams when:
// - Dataset is large (10,000+ elements)
// - Operations are CPU-intensive
// - Order doesn't matter
// - No shared mutable state
```

**Advanced Stream Patterns for Fintech:**

```java
// Pattern 1: Complex Aggregation
// Calculate sum of matched trades grouped by currency
Map<String, Double> totalByCurrency = trades.stream()
    .filter(t -> t.isMatched())
    .collect(Collectors.groupingBy(
        Trade::getCurrency,
        Collectors.summingDouble(Trade::getAmount)
    ));

// Pattern 2: Multi-level Grouping
// Group by date, then by status
Map<LocalDate, Map<String, List<Trade>>> grouped = trades.stream()
    .collect(Collectors.groupingBy(
        t -> t.getTimestamp().toLocalDate(),
        Collectors.groupingBy(Trade::getStatus)
    ));

// Pattern 3: Custom Collector for Statistics
DoubleSummaryStatistics stats = trades.stream()
    .collect(Collectors.summarizingDouble(Trade::getAmount));
System.out.println("Average: " + stats.getAverage());
System.out.println("Max: " + stats.getMax());

// Pattern 4: Finding elements efficiently
Optional<Trade> firstLargeTrade = trades.stream()
    .filter(t -> t.getAmount() > 10_000_000)
    .findFirst(); // Short-circuits

// Pattern 5: Combining Predicates
Predicate<Trade> isHighValue = t -> t.getAmount() > 1_000_000;
Predicate<Trade> isPending = t -> "PENDING".equals(t.getStatus());
Predicate<Trade> needsReview = isHighValue.and(isPending);

List<Trade> reviewTrades = trades.stream()
    .filter(needsReview)
    .collect(Collectors.toList());
```

**Stream Performance Considerations:**

```java
// GOOD - Single pass through data
long count = list.stream()
    .filter(x -> x > 10)
    .map(x -> x * 2)
    .count();

// BAD - Multiple passes (avoid this)
long filteredCount = list.stream().filter(x -> x > 10).count();
long mappedCount = list.stream().map(x -> x * 2).count();

// GOOD - Using takeWhile for sorted data (Java 9+)
List<Trade> recentTrades = sortedTrades.stream()
    .takeWhile(t -> t.getTimestamp().isAfter(cutoffTime))
    .collect(Collectors.toList());

// Use parallel streams wisely
// GOOD - Large dataset, CPU-intensive operation
parallelStream().map(this::expensiveCalculation)

// BAD - Small dataset, I/O bound
parallelStream().map(this::callExternalAPI) // Don't do this!
```

**Connection to Your Recon Project:**
In your reconciliation engine, you likely used streams to:
1. Filter unmatched transactions: `transactions.stream().filter(t -> !t.isMatched())`
2. Group by account: `groupingBy(Transaction::getAccountId)`
3. Aggregate amounts: `summingDouble(Transaction::getAmount)`
4. Bulk operations: Processing 41 server environments efficiently

---

#### **Lambda Expressions - Functional Programming in Java**

**Syntax and Evolution:**

```java
// Old way (Anonymous Inner Class)
Comparator<Trade> comparator = new Comparator<Trade>() {
    @Override
    public int compare(Trade t1, Trade t2) {
        return t1.getAmount().compareTo(t2.getAmount());
    }
};

// Lambda - Full form
Comparator<Trade> lambdaComparator = (Trade t1, Trade t2) -> {
    return t1.getAmount().compareTo(t2.getAmount());
};

// Lambda - Type inference
Comparator<Trade> shorter = (t1, t2) -> t1.getAmount().compareTo(t2.getAmount());

// Lambda - Method reference (most concise)
Comparator<Trade> shortest = Comparator.comparing(Trade::getAmount);
```

**Functional Interfaces - The Foundation:**

```java
// Built-in Functional Interfaces

// 1. Function<T, R> - Takes input, returns output
Function<Trade, String> getAccountId = Trade::getAccountId;
Function<String, Integer> stringLength = String::length;
Function<Integer, Integer> square = x -> x * x;

// Chaining functions
Function<Trade, Integer> accountIdLength = getAccountId.andThen(stringLength);

// 2. Predicate<T> - Boolean test
Predicate<Trade> isHighValue = t -> t.getAmount() > 1_000_000;
Predicate<Trade> isPending = t -> "PENDING".equals(t.getStatus());

// Combining predicates
Predicate<Trade> needsAttention = isHighValue.and(isPending).or(isFailed);

// 3. Consumer<T> - Takes input, returns nothing (side effect)
Consumer<Trade> logTrade = t -> log.info("Processing trade: {}", t);
Consumer<Trade> saveTrade = t -> repository.save(t);

// Chaining consumers
Consumer<Trade> processAndLog = saveTrade.andThen(logTrade);

// 4. Supplier<T> - No input, returns output
Supplier<LocalDateTime> currentTime = LocalDateTime::now;
Supplier<String> randomId = () -> UUID.randomUUID().toString();

// 5. BiFunction<T, U, R> - Two inputs, one output
BiFunction<Double, Double, Double> add = (a, b) -> a + b;
BiFunction<Trade, Trade, Boolean> compareTrades = (t1, t2) -> 
    t1.getAccountId().equals(t2.getAccountId());
```

**Advanced Lambda Patterns in Your Projects:**

```java
// Pattern 1: Strategy Pattern with Lambdas
// In Recon - Different matching strategies
@FunctionalInterface
interface MatchingStrategy {
    boolean matches(Transaction source, Transaction target);
}

MatchingStrategy exactMatch = (s, t) -> 
    s.getAmount().equals(t.getAmount()) && 
    s.getAccountId().equals(t.getAccountId());

MatchingStrategy fuzzyMatch = (s, t) -> {
    double diff = Math.abs(s.getAmount() - t.getAmount());
    return diff < 0.01 && s.getAccountId().equals(t.getAccountId());
};

// Pattern 2: Callback Pattern
// Processing messages from IBM MQ
void processMessage(Message msg, Consumer<String> successCallback, 
                   Consumer<Exception> errorCallback) {
    try {
        String processed = process(msg);
        successCallback.accept(processed);
    } catch (Exception e) {
        errorCallback.accept(e);
    }
}

// Usage
processMessage(
    message,
    result -> log.info("Success: {}", result),
    error -> log.error("Failed", error)
);

// Pattern 3: Factory Pattern with Suppliers
Map<String, Supplier<MessageProcessor>> processors = Map.of(
    "ADD_CASH", () -> new AddCashProcessor(),
    "DELETE_CASH", () -> new DeleteCashProcessor(),
    "ADD_SECURITY", () -> new AddSecurityProcessor()
);

MessageProcessor processor = processors.get(messageType).get();

// Pattern 4: Retry Logic with Functional Approach
<T> T retryOperation(Supplier<T> operation, int maxRetries) {
    int attempts = 0;
    while (attempts < maxRetries) {
        try {
            return operation.get();
        } catch (Exception e) {
            attempts++;
            if (attempts >= maxRetries) throw e;
            Thread.sleep(1000 * attempts);
        }
    }
    throw new RuntimeException("Max retries exceeded");
}

// Usage in your Recon project
Transaction result = retryOperation(
    () -> externalSystemClient.fetchTransaction(id),
    3
);
```

**Method References - Four Types:**

```java
// 1. Static Method Reference
Function<String, Integer> parser1 = Integer::parseInt;
// Equivalent to: s -> Integer.parseInt(s)

// 2. Instance Method Reference (specific object)
Trade trade = new Trade();
Supplier<String> supplier = trade::getAccountId;
// Equivalent to: () -> trade.getAccountId()

// 3. Instance Method Reference (arbitrary object of particular type)
Function<String, String> uppercase = String::toUpperCase;
// Equivalent to: s -> s.toUpperCase()

// 4. Constructor Reference
Supplier<Trade> tradeSupplier = Trade::new;
Function<String, Trade> tradeWithId = Trade::new; // assuming constructor exists
// Equivalent to: () -> new Trade()
```

**Real-World Usage in Your CRTS Project:**

```java
// Processing Charles River messages
public class CRTSMessageProcessor {
    
    private final Map<String, Function<Message, ProcessingResult>> handlers = Map.of(
        "TRADE_ALLOCATION", this::handleTradeAllocation,
        "SETTLEMENT", this::handleSettlement,
        "POSITION_UPDATE", this::handlePositionUpdate
    );
    
    public ProcessingResult processMessage(Message message) {
        return Optional.ofNullable(handlers.get(message.getType()))
            .map(handler -> handler.apply(message))
            .orElseThrow(() -> new UnsupportedOperationException(
                "Unknown message type: " + message.getType()));
    }
    
    private ProcessingResult handleTradeAllocation(Message msg) {
        // Implementation
        return new ProcessingResult("SUCCESS");
    }
}
```

---

#### **Optional - Null Safety Done Right**

**Why Optional? The Billion Dollar Mistake**
Tony Hoare, inventor of null references, called it his "billion-dollar mistake". NPE is the #1 cause of production bugs. Optional makes null handling explicit.

**Creating Optionals:**

```java
// 1. Empty Optional
Optional<Trade> emptyTrade = Optional.empty();

// 2. Optional with non-null value
Trade trade = new Trade();
Optional<Trade> optionalTrade = Optional.of(trade);
// WARNING: Optional.of() throws NPE if trade is null!

// 3. Optional that might be null (MOST COMMON)
Trade nullableTrade = repository.findById(id);
Optional<Trade> safeTrade = Optional.ofNullable(nullableTrade);
```

**Optional Operations - Comprehensive Guide:**

```java
// 1. isPresent() and get() - AVOID THIS PATTERN!
// BAD - defeats the purpose of Optional
if (optional.isPresent()) {
    Trade trade = optional.get();
    // do something
}

// 2. ifPresent() - Execute if value exists
optional.ifPresent(trade -> log.info("Found trade: {}", trade));

// 3. orElse() - Provide default value
Trade trade = optional.orElse(new Trade()); // Always creates default

// 4. orElseGet() - Provide supplier (lazy evaluation)
Trade trade = optional.orElseGet(() -> createDefaultTrade()); // Only called if empty

// 5. orElseThrow() - Throw exception if empty
Trade trade = optional.orElseThrow(() -> 
    new TradeNotFoundException("Trade not found"));

// 6. map() - Transform the value if present
Optional<String> accountId = optionalTrade
    .map(Trade::getAccountId);

// 7. flatMap() - For nested Optionals (IMPORTANT!)
Optional<String> currency = optionalTrade
    .flatMap(Trade::getCurrency); // Assuming getCurrency returns Optional<String>

// 8. filter() - Keep value only if predicate matches
Optional<Trade> highValueTrade = optionalTrade
    .filter(t -> t.getAmount() > 1_000_000);
```

**Advanced Optional Patterns for Production Code:**

```java
// Pattern 1: Chaining Operations
String result = Optional.ofNullable(trade)
    .map(Trade::getAccount)
    .map(Account::getCustomer)
    .map(Customer::getEmail)
    .orElse("no-email@example.com");

// Pattern 2: Combining with Streams
List<Trade> trades = repository.findAll();
List<String> highValueAccounts = trades.stream()
    .map(Trade::getAccountId)
    .filter(Objects::nonNull)
    .distinct()
    .collect(Collectors.toList());

// Better with Optional
List<String> accounts = trades.stream()
    .map(trade -> Optional.ofNullable(trade.getAccountId()))
    .filter(Optional::isPresent)
    .map(Optional::get)
    .distinct()
    .collect(Collectors.toList());

// Pattern 3: Optional in Method Returns
// GOOD - Repository pattern
public Optional<Trade> findTradeById(String id) {
    Trade trade = database.query(id);
    return Optional.ofNullable(trade);
}

// Usage
repository.findTradeById(id)
    .map(this::processTrade)
    .map(this::enrichTrade)
    .ifPresent(this::sendToDownstream);

// Pattern 4: Avoiding Nested Optionals
// BAD
Optional<Optional<String>> nested = optional.map(Trade::getCurrency);

// GOOD - Use flatMap
Optional<String> currency = optional.flatMap(Trade::getCurrency);

// Pattern 5: Optional with Validation
public Optional<Trade> validateAndProcess(Trade trade) {
    return Optional.of(trade)
        .filter(t -> t.getAmount() > 0)
        .filter(t -> t.getAccountId() != null)
        .map(this::enrichWithMarketData)
        .map(this::applyBusinessRules);
}
```

**Optional in Your Recon Project:**

```java
public class ReconciliationService {
    
    // Find matching transaction
    public Optional<Transaction> findMatch(Transaction source) {
        return Optional.ofNullable(targetSystem.findByKey(source.getKey()))
            .filter(target -> matchesAmount(source, target))
            .filter(target -> matchesDate(source, target));
    }
    
    // Process reconciliation
    public ReconciliationResult reconcile(String transactionId) {
        return findSourceTransaction(transactionId)
            .flatMap(this::findMatch)
            .map(match -> ReconciliationResult.matched(match))
            .orElseGet(() -> ReconciliationResult.unmatched(transactionId));
    }
    
    private Optional<Transaction> findSourceTransaction(String id) {
        return Optional.ofNullable(sourceSystem.findById(id));
    }
}
```

**When NOT to Use Optional:**

```java
// DON'T use Optional for:
// 1. Class fields
class BadPractice {
    private Optional<String> name; // NO! Use nullable field instead
}

// 2. Method parameters
public void process(Optional<Trade> trade) {} // NO!

// 3. Collections
Optional<List<Trade>> trades; // NO! Return empty list instead

// 4. Primitive wrappers - Use OptionalInt, OptionalLong, OptionalDouble
Optional<Integer> count = Optional.of(42); // NO!
OptionalInt count = OptionalInt.of(42);    // YES!
```

---

### 1.2 Java 11 New Features

#### **HTTP Client (Replacement for HttpURLConnection)**

```java
// Creating HTTP Client (reusable)
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)
    .connectTimeout(Duration.ofSeconds(10))
    .build();

// Synchronous GET Request
HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/trades"))
    .header("Authorization", "Bearer " + token)
    .GET()
    .build();

HttpResponse<String> response = client.send(request, 
    HttpResponse.BodyHandlers.ofString());

// Asynchronous Request (Non-blocking)
client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
    .thenApply(HttpResponse::body)
    .thenApply(this::parseTradeData)
    .thenAccept(this::processeTrades)
    .exceptionally(e -> {
        log.error("Failed to fetch trades", e);
        return null;
    });

// POST Request with JSON Body
String json = """
    {
        "tradeId": "TRD123",
        "amount": 1000000,
        "currency": "USD"
    }
    """;

HttpRequest postRequest = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/trades"))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(json))
    .build();

// Use in your CRTS integration
HttpResponse<String> charlesRiverResponse = client.send(
    postRequest, 
    HttpResponse.BodyHandlers.ofString()
);
```

#### **Local Variable Type Inference (var) - Enhanced**

```java
// Java 11 allows var in lambda parameters
// Useful for adding annotations
BiFunction<String, String, String> concat = 
    (@NonNull var s1, @NonNull var s2) -> s1 + s2;

// In your code
trades.stream()
    .map((@NonNull var trade) -> trade.getAccountId())
    .collect(Collectors.toList());
```

#### **String Methods Enhancement**

```java
String multiline = """
    SELECT t.trade_id, t.amount, t.currency
    FROM trades t
    WHERE t.status = 'PENDING'
    AND t.amount > 1000000
    """;

// New String methods
" ".isBlank();           // true
"  text  ".strip();      // "text" (Unicode-aware)
"Java".repeat(3);        // "JavaJavaJava"

"line1\nline2\nline3".lines()
    .forEach(System.out::println);
```

#### **Collection.toArray() Enhancement**

```java
// Old way
List<Trade> trades = getTradeList();
Trade[] array = trades.toArray(new Trade[trades.size()]);

// Java 11 - simpler
Trade[] array = trades.toArray(Trade[]::new);
```

---

### 1.3 Java 17 LTS Features (YOUR VERSION!)

#### **Sealed Classes - Controlled Inheritance**

**What Problem Do They Solve?**
In fintech systems, you often have a fixed set of types (e.g., trade types, message types). Sealed classes let you restrict inheritance to known subtypes, enabling exhaustive pattern matching.

**Basic Sealed Class:**

```java
// Define a sealed interface/class
public sealed interface PaymentMethod 
    permits CreditCard, DebitCard, UPI, NetBanking {
}

public final class CreditCard implements PaymentMethod {
    private String cardNumber;
    private String cvv;
}

public final class DebitCard implements PaymentMethod {
    private String cardNumber;
    private String pin;
}

public final class UPI implements PaymentMethod {
    private String upiId;
}

public non-sealed class NetBanking implements PaymentMethod {
    private String bankName;
    // non-sealed allows further extension
}
```

**Use in Your Recon Project:**

```java
public sealed interface ReconciliationStatus 
    permits Matched, Unmatched, PartialMatch, Exception {
}

public final class Matched implements ReconciliationStatus {
    private Transaction source;
    private Transaction target;
    private LocalDateTime matchedAt;
}

public final class Unmatched implements ReconciliationStatus {
    private Transaction source;
    private String reason;
}

public final class PartialMatch implements ReconciliationStatus {
    private Transaction source;
    private Transaction target;
    private List<String> discrepancies;
}

public final class Exception implements ReconciliationStatus {
    private String errorMessage;
    private Throwable cause;
}
```

**Why This is Powerful:**

```java
// The compiler KNOWS all possible subtypes
public String processReconciliation(ReconciliationStatus status) {
    return switch (status) {
        case Matched m -> "Matched: " + m.getSource().getId();
        case Unmatched u -> "Unmatched: " + u.getReason();
        case PartialMatch p -> "Partial: " + p.getDiscrepancies();
        case Exception e -> "Error: " + e.getErrorMessage();
        // No default needed! Compiler ensures exhaustiveness
    };
}
```

**Sealed Classes in Message Processing (CRTS):**

```java
public sealed interface MessageType 
    permits TradeAllocation, Settlement, PositionUpdate, CashMovement {
}

public final class TradeAllocation implements MessageType {
    private String tradeId;
    private BigDecimal quantity;
}

public final class Settlement implements MessageType {
    private String settlementId;
    private LocalDate settlementDate;
}

// etc.

// Type-safe message routing
public void routeMessage(MessageType message) {
    switch (message) {
        case TradeAllocation t -> tradeService.allocate(t);
        case Settlement s -> settlementService.settle(s);
        case PositionUpdate p -> positionService.update(p);
        case CashMovement c -> cashService.process(c);
    };
}
```

#### **Records - Immutable Data Carriers**

**What Are Records?**
Records are concise syntax for declaring data-carrying classes. Perfect for DTOs, value objects, and message payloads.

**Traditional Class vs Record:**

```java
// Traditional way - VERBOSE
public class TradeDTO {
    private final String tradeId;
    private final BigDecimal amount;
    private final String currency;
    
    public TradeDTO(String tradeId, BigDecimal amount, String currency) {
        this.tradeId = tradeId;
        this.amount = amount;
        this.currency = currency;
    }
    
    public String getTradeId() { return tradeId; }
    public BigDecimal getAmount() { return amount; }
    public String getCurrency() { return currency; }
    
    @Override
    public boolean equals(Object o) { /* boilerplate */ }
    
    @Override
    public int hashCode() { /* boilerplate */ }
    
    @Override
    public String toString() { /* boilerplate */ }
}

// Record - ONE LINE!
public record TradeDTO(String tradeId, BigDecimal amount, String currency) {}

// Automatically generates:
// - Constructor
// - Getters (no "get" prefix: trade.tradeId())
// - equals(), hashCode(), toString()
// - All fields are private final
```

**Using Records in Your Projects:**

```java
// Recon project - Reconciliation result
public record ReconciliationResult(
    String sourceId,
    String targetId,
    boolean matched,
    BigDecimal amountDifference,
    List<String> discrepancies
) {
    // Compact constructor for validation
    public ReconciliationResult {
        if (sourceId == null) {
            throw new IllegalArgumentException("Source ID cannot be null");
        }
        // Auto-assignment happens after this block
    }
    
    // You can add custom methods
    public boolean hasDiscrepancies() {
        return !discrepancies.isEmpty();
    }
}

// CRTS project - Message payload
public record TradeMessage(
    String messageId,
    String tradeId,
    TradeType type,
    BigDecimal quantity,
    String securityId,
    LocalDateTime timestamp
) {
    // Static factory method
    public static TradeMessage fromJson(String json) {
        // Parse and return
    }
}

// OPS Portal - API response
public record ApiResponse<T>(
    int statusCode,
    String message,
    T data,
    LocalDateTime timestamp
) {
    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(200, "Success", data, LocalDateTime.now());
    }
    
    public static <T> ApiResponse<T> error(String message) {
        return new ApiResponse<>(500, message, null, LocalDateTime.now());
    }
}
```

**Records with Validation:**

```java
public record Trade(
    String id,
    BigDecimal amount,
    String currency
) {
    // Compact constructor with validation
    public Trade {
        Objects.requireNonNull(id, "ID cannot be null");
        Objects.requireNonNull(currency, "Currency cannot be null");
        
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        
        if (id.isBlank()) {
            throw new IllegalArgumentException("ID cannot be blank");
        }
    }
}
```

**Records vs Lombok:**

```java
// Lombok
@Data
@AllArgsConstructor
public class TradeDTO {
    private final String id;
    private final BigDecimal amount;
}

// Record - No dependency needed, built into Java
public record TradeDTO(String id, BigDecimal amount) {}
```

#### **Pattern Matching for Switch**

**Evolution from Traditional Switch:**

```java
// Old switch (Java 8)
String result;
switch (status) {
    case "PENDING":
        result = "Processing";
        break;
    case "COMPLETED":
        result = "Done";
        break;
    case "FAILED":
        result = "Error";
        break;
    default:
        result = "Unknown";
}

// Java 17 - Switch expressions
String result = switch (status) {
    case "PENDING" -> "Processing";
    case "COMPLETED" -> "Done";
    case "FAILED" -> "Error";
    default -> "Unknown";
};
```

**Pattern Matching with instanceof:**

```java
// Old way - Manual casting
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// Java 17 - Pattern matching
if (obj instanceof String s) {
    System.out.println(s.length()); // 's' is auto-cast
}

// In condition chains
if (obj instanceof String s && s.length() > 10) {
    // s is available here
}
```

**Advanced Pattern Matching in Switch:**

```java
// Type patterns with switch
Object obj = getMessage();

String formatted = switch (obj) {
    case Integer i -> String.format("Int: %d", i);
    case String s -> String.format("String: %s", s);
    case Trade t -> String.format("Trade: %s", t.getId());
    case null -> "Null value";
    default -> "Unknown type";
};

// Guarded patterns
String classification = switch (trade) {
    case Trade t when t.getAmount().compareTo(new BigDecimal("1000000")) > 0 
        -> "High value";
    case Trade t when t.getAmount().compareTo(new BigDecimal("100000")) > 0 
        -> "Medium value";
    case Trade t -> "Normal value";
};

// With sealed classes (exhaustive!)
String status = switch (reconciliationResult) {
    case Matched m -> "Matched at " + m.getMatchedAt();
    case Unmatched u -> "Unmatched: " + u.getReason();
    case PartialMatch p -> "Partial: " + p.getDiscrepancies().size() + " issues";
    case Exception e -> "Error: " + e.getErrorMessage();
    // No default needed - compiler knows all cases
};
```

**Real-World Usage in Your Projects:**

```java
// Recon - Processing different reconciliation outcomes
public void handleReconciliation(ReconciliationStatus status) {
    switch (status) {
        case Matched m -> {
            log.info("Matched transaction: {}", m.getSource().getId());
            updateDatabase(m);
            sendNotification(m);
        }
        case Unmatched u -> {
            log.warn("Unmatched: {}", u.getReason());
            createException(u);
            alertTeam(u);
        }
        case PartialMatch p -> {
            log.warn("Partial match with {} discrepancies", 
                p.getDiscrepancies().size());
            requireManualReview(p);
        }
        case Exception e -> {
            log.error("Reconciliation failed", e.getCause());
            triggerAlert(e);
        }
    }
}

// CRTS - Message routing
public ProcessingResult routeMessage(Message message) {
    return switch (message.getType()) {
        case "TRADE_ALLOCATION" -> processTradeAllocation(message);
        case "SETTLEMENT" -> processSettlement(message);
        case "POSITION_UPDATE" -> processPositionUpdate(message);
        case "CASH_MOVEMENT" -> processCashMovement(message);
        default -> throw new UnsupportedOperationException(
            "Unknown message type: " + message.getType());
    };
}
```

#### **Text Blocks - Multi-line Strings**

```java
// Perfect for SQL queries in your Recon project
String query = """
    SELECT 
        t.transaction_id,
        t.amount,
        t.currency,
        t.account_id,
        t.timestamp
    FROM transactions t
    WHERE t.status = 'PENDING'
        AND t.amount > 1000000
        AND t.processed_date >= SYSDATE - 1
    ORDER BY t.timestamp DESC
    """;

// JSON templates for API calls
String requestBody = """
    {
        "tradeId": "%s",
        "action": "%s",
        "amount": %f,
        "currency": "USD",
        "metadata": {
            "source": "RECON_ENGINE",
            "timestamp": "%s"
        }
    }
    """.formatted(tradeId, action, amount, timestamp);

// IBM MQ message template
String mqMessage = """
    <Message>
        <Header>
            <MessageType>TRADE_UPDATE</MessageType>
            <Timestamp>%s</Timestamp>
        </Header>
        <Body>
            <TradeId>%s</TradeId>
            <Status>%s</Status>
        </Body>
    </Message>
    """.formatted(timestamp, tradeId, status);
```

---

### 1.4 Java 21 Latest Features

#### **Virtual Threads (Project Loom) - GAME CHANGER**

**What's the Problem Virtual Threads Solve?**

Traditional threads (Platform threads):
- OS-level threads
- Heavy (1-2 MB each)
- Limited by OS (typically 1000-10000)
- Context switching overhead

Virtual threads:
- JVM-managed
- Lightweight (few KB)
- Millions possible
- Scheduled on platform threads

**Traditional vs Virtual Thread Example:**

```java
// OLD WAY - Platform Threads (Limited scalability)
ExecutorService executor = Executors.newFixedThreadPool(100);
for (int i = 0; i < 10000; i++) {
    executor.submit(() -> {
        // Handle request
        callExternalAPI();
    });
}

// NEW WAY - Virtual Threads (Scale to millions!)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10000; i++) {
        executor.submit(() -> {
            // Handle request
            callExternalAPI();
        });
    }
}
```

**Virtual Threads in Your Fintech Projects:**

```java
// Recon - Processing millions of transactions
public class ReconciliationEngine {
    
    public void reconcileDaily() {
        List<Transaction> transactions = loadAllTransactions(); // 10M records
        
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (Transaction txn : transactions) {
                executor.submit(() -> {
                    // Each transaction gets its own virtual thread
                    Optional<Transaction> match = findMatch(txn);
                    match.ifPresentOrElse(
                        m -> recordMatch(txn, m),
                        () -> recordUnmatched(txn)
                    );
                });
            }
        } // Auto-shutdown and wait
    }
}

// CRTS - Handling concurrent message processing
public class MessageProcessor {
    
    public void processMessages(List<Message> messages) {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            messages.forEach(msg -> 
                executor.submit(() -> {
                    // Each message in its own virtual thread
                    processMessage(msg);
                    acknowledgeMessage(msg);
                })
            );
        }
    }
}

// OPS Portal - Handling multiple backend service calls
public CompletableFuture<DashboardData> loadDashboard(String userId) {
    return CompletableFuture.supplyAsync(() -> {
        // All these run in virtual threads
        var userData = Thread.startVirtualThread(() -> fetchUserData(userId));
        var accountData = Thread.startVirtualThread(() -> fetchAccounts(userId));
        var tradeData = Thread.startVirtualThread(() -> fetchTrades(userId));
        
        // Wait for all
        userData.join();
        accountData.join();
        tradeData.join();
        
        return new DashboardData(/*...*/);
    }, virtualThreadExecutor());
}
```

**When to Use Virtual Threads:**
✅ I/O-bound operations (API calls, DB queries, file I/O)
✅ High concurrency scenarios (millions of requests)
✅ Blocking operations (waiting for network response)

❌ CPU-bound operations (use parallel streams)
❌ Operations holding locks for long time

#### **Sequenced Collections**

**The Problem:**
Getting first/last element was inconsistent across collections.

```java
// OLD - Inconsistent APIs
List<Trade> list = getTrades();
Trade first = list.get(0);
Trade last = list.get(list.size() - 1);

Deque<Trade> deque = getTradeDeque();
Trade first = deque.getFirst();
Trade last = deque.getLast();

// NEW - Unified API (Java 21)
List<Trade> list = getTrades();
Trade first = list.getFirst();  // Clean!
Trade last = list.getLast();

// Reverse order
List<Trade> reversed = list.reversed();

// Use in your code
public Trade getMostRecentTrade(List<Trade> trades) {
    return trades.getLast(); // Much cleaner than get(size()-1)
}
```

#### **String Templates (Preview)**

```java
// OLD
String message = "Trade " + tradeId + " with amount " + amount + 
                " in currency " + currency + " was processed";

// NEW (Java 21 Preview)
String message = STR."""
    Trade \{tradeId} with amount \{amount} 
    in currency \{currency} was processed
    """;

// In your Recon logs
String logMessage = STR."""
    Reconciliation Result:
    - Source ID: \{source.getId()}
    - Target ID: \{target.getId()}
    - Amount Match: \{amountsMatch}
    - Timestamp: \{LocalDateTime.now()}
    """;
```

#### **Pattern Matching for Switch - Enhanced**

```java
// Null handling in switch
String result = switch (obj) {
    case null -> "Null value";
    case String s -> "String: " + s;
    case Integer i -> "Integer: " + i;
    default -> "Other type";
};

// Record patterns
record Trade(String id, BigDecimal amount) {}

String classify = switch (obj) {
    case Trade(var id, var amount) when amount.compareTo(BigDecimal.valueOf(1000000)) > 0
        -> "High value trade: " + id;
    case Trade(var id, var amount) 
        -> "Normal trade: " + id;
    default -> "Not a trade";
};
```

---

### 1.5 Concurrency & CompletableFuture - Production Patterns

#### **CompletableFuture - Async Programming**

**Basic Operations:**

```java
// 1. Creating CompletableFutures
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
    // Runs in ForkJoinPool.commonPool()
    return fetchDataFromDB();
});

CompletableFuture<Void> future2 = CompletableFuture.runAsync(() -> {
    // No return value
    logEvent();
});

// 2. With custom executor
ExecutorService executor = Executors.newFixedThreadPool(10);
CompletableFuture<String> future3 = CompletableFuture.supplyAsync(
    () -> callExternalAPI(),
    executor
);
```

**Chaining Operations:**

```java
// Transform result
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> "Hello")
    .thenApply(String::length)        // Transform: String -> Integer
    .thenApply(len -> len * 2);       // Transform: Integer -> Integer

// No return value
future.thenAccept(result -> System.out.println(result));

// Combine two independent futures
CompletableFuture<Trade> trade = CompletableFuture.supplyAsync(() -> fetchTrade());
CompletableFuture<Price> price = CompletableFuture.supplyAsync(() -> fetchPrice());

CompletableFuture<String> combined = trade.thenCombine(price, 
    (t, p) -> "Trade " + t.getId() + " at price " + p.getValue());

// Chain dependent operations
CompletableFuture<String> result = CompletableFuture.supplyAsync(() -> "TradeID123")
    .thenCompose(id -> fetchTradeDetails(id))  // Returns CompletableFuture
    .thenCompose(trade -> enrichWithMarketData(trade))
    .thenApply(enrichedTrade -> enrichedTrade.toString());
```

**Real-World Patterns from Your Projects:**

```java
// Pattern 1: Parallel API Calls (OPS Portal)
public CompletableFuture<DashboardData> loadDashboard(String userId) {
    CompletableFuture<User> userFuture = 
        CompletableFuture.supplyAsync(() -> userService.getUser(userId));
    
    CompletableFuture<List<Account>> accountsFuture = 
        CompletableFuture.supplyAsync(() -> accountService.getAccounts(userId));
    
    CompletableFuture<List<Trade>> tradesFuture = 
        CompletableFuture.supplyAsync(() -> tradeService.getTrades(userId));
    
    // Wait for all and combine
    return CompletableFuture.allOf(userFuture, accountsFuture, tradesFuture)
        .thenApply(v -> new DashboardData(
            userFuture.join(),
            accountsFuture.join(),
            tradesFuture.join()
        ));
}

// Pattern 2: Timeout and Fallback (CRTS)
public CompletableFuture<MarketData> getMarketDataWithTimeout(String symbol) {
    return CompletableFuture.supplyAsync(() -> marketDataService.fetch(symbol))
        .orTimeout(2, TimeUnit.SECONDS)
        .exceptionally(ex -> {
            log.warn("Market data fetch failed, using cache", ex);
            return marketDataCache.get(symbol);
        });
}

// Pattern 3: Retry Logic
public CompletableFuture<Transaction> fetchWithRetry(String id, int maxRetries) {
    return CompletableFuture.supplyAsync(() -> externalSystem.fetch(id))
        .exceptionallyCompose(ex -> {
            if (maxRetries > 0) {
                log.warn("Retry {} remaining for {}", maxRetries, id);
                return fetchWithRetry(id, maxRetries - 1);
            }
            throw new RuntimeException("Max retries exceeded", ex);
        });
}

// Pattern 4: First Successful (Multiple data sources)
public CompletableFuture<Trade> fetchFromMultipleSources(String tradeId) {
    CompletableFuture<Trade> source1 = 
        CompletableFuture.supplyAsync(() -> database1.fetch(tradeId));
    
    CompletableFuture<Trade> source2 = 
        CompletableFuture.supplyAsync(() -> database2.fetch(tradeId));
    
    CompletableFuture<Trade> source3 = 
        CompletableFuture.supplyAsync(() -> cache.fetch(tradeId));
    
    // Return first successful
    return CompletableFuture.anyOf(source1, source2, source3)
        .thenApply(result -> (Trade) result);
}

// Pattern 5: Reconciliation Engine - Bulk Processing
public CompletableFuture<ReconciliationReport> reconcileBatch(
    List<Transaction> transactions) {
    
    List<CompletableFuture<ReconciliationResult>> futures = 
        transactions.stream()
            .map(txn -> CompletableFuture.supplyAsync(() -> reconcile(txn)))
            .collect(Collectors.toList());
    
    return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
        .thenApply(v -> {
            List<ReconciliationResult> results = futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList());
            return generateReport(results);
        });
}
```

**Exception Handling:**

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    if (Math.random() > 0.5) {
        throw new RuntimeException("Random failure");
    }
    return "Success";
})
.exceptionally(ex -> {
    log.error("Operation failed", ex);
    return "Default value";
})
.handle((result, ex) -> {
    if (ex != null) {
        log.error("Error occurred", ex);
        return "Error: " + ex.getMessage();
    }
    return result;
});
```

#### **Thread Pools and Executors**

```java
// 1. Fixed Thread Pool - For bounded parallelism
ExecutorService executor = Executors.newFixedThreadPool(10);
// Use case: Processing fixed number of worker threads

// 2. Cached Thread Pool - Creates threads as needed
ExecutorService cached = Executors.newCachedThreadPool();
// Use case: Many short-lived tasks

// 3. Single Thread Executor - Sequential processing
ExecutorService single = Executors.newSingleThreadExecutor();
// Use case: Maintaining order

// 4. Scheduled Executor - Periodic tasks
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(5);
scheduled.scheduleAtFixedRate(() -> {
    // Run every minute
    reconciliationService.checkPendingReconciliations();
}, 0, 1, TimeUnit.MINUTES);

// 5. Work Stealing Pool - Java 8+
ExecutorService workStealing = Executors.newWorkStealingPool();
// Use case: CPU-intensive tasks with varying duration

// Always shutdown!
try {
    executor.shutdown();
    if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
        executor.shutdownNow();
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
}
```

#### **Synchronization and Thread Safety**

```java
// 1. Synchronized method
public synchronized void updateBalance(double amount) {
    this.balance += amount;
}

// 2. Synchronized block - More granular
public void processTransaction(Transaction txn) {
    // Non-critical code here
    synchronized (this) {
        // Critical section
        this.balance += txn.getAmount();
    }
}

// 3. ReentrantLock - More flexible
private final ReentrantLock lock = new ReentrantLock();

public void processWithLock(Transaction txn) {
    lock.lock();
    try {
        // Critical section
        updateDatabase(txn);
    } finally {
        lock.unlock(); // Always unlock in finally
    }
}

// 4. ReadWriteLock - Optimize read-heavy scenarios
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();

public double getBalance() {
    rwLock.readLock().lock();
    try {
        return this.balance;
    } finally {
        rwLock.readLock().unlock();
    }
}

public void updateBalance(double amount) {
    rwLock.writeLock().lock();
    try {
        this.balance += amount;
    } finally {
        rwLock.writeLock().unlock();
    }
}

// 5. Concurrent Collections
ConcurrentHashMap<String, Trade> tradeCache = new ConcurrentHashMap<>();
CopyOnWriteArrayList<String> subscribers = new CopyOnWriteArrayList<>();
BlockingQueue<Message> messageQueue = new LinkedBlockingQueue<>();

// 6. Atomic Variables - Lock-free thread safety
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();
counter.compareAndSet(10, 20);

AtomicReference<Trade> currentTrade = new AtomicReference<>();
currentTrade.set(new Trade());
```

**Connection to Your Projects:**

In your Recon engine with 41 servers and high throughput:
- CompletableFuture for parallel reconciliation
- Thread pools for bounded parallelism
- Concurrent collections for caching
- Atomic counters for statistics

---

### 1.6 JVM Internals & Garbage Collection

#### **JVM Architecture**

```
┌─────────────────────────────────────────────────┐
│              JVM (Java Virtual Machine)          │
├─────────────────────────────────────────────────┤
│  ┌───────────────────────────────────────────┐  │
│  │         Class Loader Subsystem            │  │
│  │  - Bootstrap CL, Extension CL, System CL  │  │
│  └───────────────────────────────────────────┘  │
│                                                   │
│  ┌───────────────────────────────────────────┐  │
│  │         Runtime Data Areas                │  │
│  │                                            │  │
│  │  1. Method Area (Metaspace in Java 8+)   │  │
│  │     - Class metadata                      │  │
│  │     - Static variables                    │  │
│  │     - Constant pool                       │  │
│  │                                            │  │
│  │  2. Heap (Garbage Collected)             │  │
│  │     ┌─────────────────────────────────┐  │  │
│  │     │    Young Generation              │  │  │
│  │     │  - Eden Space                    │  │  │
│  │     │  - Survivor 0, Survivor 1        │  │  │
│  │     └─────────────────────────────────┘  │  │
│  │     ┌─────────────────────────────────┐  │  │
│  │     │    Old Generation (Tenured)      │  │  │
│  │     │  - Long-lived objects            │  │  │
│  │     └─────────────────────────────────┘  │  │
│  │                                            │  │
│  │  3. Stack (per thread)                   │  │
│  │     - Local variables                     │  │
│  │     - Method call frames                  │  │
│  │                                            │  │
│  │  4. PC Register (per thread)             │  │
│  │  5. Native Method Stack                  │  │
│  └───────────────────────────────────────────┘  │
│                                                   │
│  ┌───────────────────────────────────────────┐  │
│  │        Execution Engine                   │  │
│  │  - Interpreter                            │  │
│  │  - JIT Compiler (C1, C2)                  │  │
│  │  - Garbage Collector                      │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

#### **Garbage Collectors - Deep Dive**

**1. G1GC (Your Current GC - Java 17 Default)**

```bash
# G1GC tuning parameters
-XX:+UseG1GC
-Xms4G -Xmx8G                           # Heap size
-XX:MaxGCPauseMillis=200                # Target pause time
-XX:G1HeapRegionSize=16M                # Region size
-XX:InitiatingHeapOccupancyPercent=45   # When to start concurrent GC
-XX:G1NewSizePercent=30                 # Min young gen size
-XX:G1MaxNewSizePercent=60              # Max young gen size
-XX:ConcGCThreads=4                     # Concurrent GC threads
-XX:ParallelGCThreads=8                 # Parallel GC threads
```

**How G1GC Works:**
- Divides heap into ~2000 equal-sized regions
- Each region can be Eden, Survivor, or Old
- Performs concurrent marking to find reclaimable regions
- Mixed GC collects Young + some Old regions
- Predictable pause times by collecting incrementally

**When to use:**
- ✅ Heap size > 4GB
- ✅ Need predictable pause times
- ✅ Balanced throughput and latency
- ✅ Your Recon engine (moderate latency requirements)

**2. ZGC (Ultra-Low Latency - Java 21)**

```bash
# ZGC configuration
-XX:+UseZGC
-XX:+ZGenerational                      # Java 21+
-Xms16G -Xmx16G                        # Set min = max
-XX:ZUncommitDelay=300                 # Unused memory uncommit delay
-XX:ZCollectionInterval=5              # Force GC interval (seconds)
-XX:ZAllocationSpikeTolerance=2        # Handle allocation spikes
-XX:ConcGCThreads=4
```

**ZGC Characteristics:**
- Pause times < 10ms regardless of heap size
- Colored pointers for concurrent operations
- Load barriers for concurrent compaction
- Works with heaps up to 16TB
- Higher CPU usage (10-15% overhead)

**When to use:**
- ✅ Extremely low latency requirements (< 10ms)
- ✅ Large heaps (> 100GB)
- ✅ Real-time trading systems
- ✅ High-frequency reconciliation
- ❌ Small heaps or tight CPU budgets

**3. Parallel GC (Throughput-focused)**

```bash
-XX:+UseParallelGC
-XX:ParallelGCThreads=8
-XX:MaxGCPauseMillis=500
```

**When to use:**
- ✅ Batch processing
- ✅ Throughput over latency
- ✅ Your nightly reconciliation jobs

**4. Serial GC**
- Single-threaded
- Use only for very small applications

**Garbage Collection Tuning Example for Recon Engine:**

```bash
# Production JVM args for Recon Engine (8GB heap)
java -server \
  -Xms8G -Xmx8G \                      # Fixed heap prevents resizing overhead
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \           # 200ms max pause
  -XX:+ParallelRefProcEnabled \        # Parallel reference processing
  -XX:+UseStringDeduplication \        # Save memory on duplicate strings
  -XX:InitiatingHeapOccupancyPercent=45 \
  -XX:G1HeapRegionSize=16M \
  \
  # GC Logging for monitoring
  -Xlog:gc*:file=/var/log/recon-gc.log:time,uptime,level,tags \
  -Xlog:gc+heap=debug \
  \
  # Heap dump on OOM
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/dumps/recon-heap.hprof \
  \
  # JMX monitoring
  -Dcom.sun.management.jmxremote \
  -Dcom.sun.management.jmxremote.port=9010 \
  -Dcom.sun.management.jmxremote.authenticate=false \
  -Dcom.sun.management.jmxremote.ssl=false \
  \
  # Application
  -jar recon-engine.jar
```

#### **Memory Leaks and Analysis**

**Common Memory Leak Patterns:**

```java
// 1. Static collection holding references
public class CacheService {
    private static Map<String, Trade> cache = new HashMap<>(); // Memory leak!
    
    public void addTrade(Trade trade) {
        cache.put(trade.getId(), trade); // Never removed
    }
}

// Fix: Use bounded cache
private static final int MAX_SIZE = 10000;
private static Map<String, Trade> cache = new LinkedHashMap<>() {
    @Override
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > MAX_SIZE;
    }
};

// 2. Unclosed resources
public void processFile(String path) {
    InputStream is = new FileInputStream(path); // Leak if exception occurs
    // process file
} // 'is' never closed

// Fix: Try-with-resources
public void processFile(String path) throws IOException {
    try (InputStream is = new FileInputStream(path)) {
        // process file
    } // Automatically closed
}

// 3. ThreadLocal not cleaned up
public class UserContext {
    private static ThreadLocal<User> currentUser = new ThreadLocal<>();
    
    public static void setUser(User user) {
        currentUser.set(user); // Leak in thread pools
    }
    
    // Fix: Always remove
    public static void clearUser() {
        currentUser.remove();
    }
}

// 4. Listeners not unregistered
public class EventService {
    private List<EventListener> listeners = new ArrayList<>();
    
    public void addListener(EventListener listener) {
        listeners.add(listener);
    }
    
    // Missing: removeListener() - Leak!
}
```

**Analyzing Memory Issues:**

```bash
# 1. Generate heap dump
jmap -dump:live,format=b,file=heap.bin <pid>

# 2. Analyze with Eclipse MAT, VisualVM, or JProfiler

# 3. Monitor live heap
jconsole <pid>
jvisualvm <pid>

# 4. GC analysis
jstat -gc <pid> 1000  # Every 1 second

# 5. Flight Recorder (Production-safe profiling)
jcmd <pid> JFR.start duration=60s filename=recording.jfr
```

---

This completes Part 1 (Core Java). This is EXTREMELY detailed with real examples from your projects. 

Interview talking points ready:
- "In my Recon project, I used streams for bulk processing 10M transactions"
- "I optimized G1GC for 200ms pause times on 8GB heap"
- "For CRTS, sealed classes + pattern matching ensured type-safe message routing"
- "Virtual threads in Java 21 let us scale to millions of concurrent reconciliations"

---

## 2. SPRING BOOT & MICROSERVICES {#spring-boot-microservices}

### 2.1 Spring Boot Fundamentals

#### **Auto-Configuration - The Magic Behind Spring Boot**

**How It Works:**

```java
// Spring Boot scans @Conditional annotations
@Configuration
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean(DataSource.class)
public class DataSourceAutoConfiguration {
    
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}

// Triggered when:
// 1. DataSource.class is on classpath
// 2. No DataSource bean already defined
// 3. spring.datasource.* properties present
```

**Creating Your Own Auto-Configuration:**

```java
// For your Recon Engine - Auto-configure reconciliation service
@Configuration
@ConditionalOnProperty(name = "recon.enabled", havingValue = "true")
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
}

@ConfigurationProperties(prefix = "recon")
@Data
public class ReconProperties {
    private int batchSize = 1000;
    private int threadPoolSize = 10;
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
  thread-pool-size: 20
  timeout: 10m
  source-system: CHARLES_RIVER
  target-system: ALADDIN
```

#### **Component Scanning and Bean Lifecycle**

```java
@SpringBootApplication  // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class ReconApplication {
    public static void main(String[] args) {
        SpringApplication.run(ReconApplication.class, args);
    }
}

// Bean lifecycle hooks
@Component
public class ReconciliationEngine {
    
    @PostConstruct  // Called after dependency injection
    public void init() {
        log.info("Recon Engine starting...");
        loadConfigurations();
        connectToSystems();
    }
    
    @PreDestroy  // Called before bean destruction
    public void cleanup() {
        log.info("Recon Engine shutting down...");
        closeConnections();
        flushPendingRecords();
    }
}

// Bean scopes
@Bean
@Scope("singleton")  // Default - one instance per container
public CacheService cacheService() { }

@Bean
@Scope("prototype")  // New instance each time
public TradeProcessor tradeProcessor() { }

@Bean
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public UserContext userContext() { } // New per HTTP request
```

#### **Dependency Injection - Advanced Patterns**

```java
// 1. Constructor Injection (PREFERRED - immutable, testable)
@Service
public class ReconciliationService {
    private final TransactionRepository repository;
    private final MatchingEngine matchingEngine;
    private final NotificationService notificationService;
    
    // @Autowired optional with single constructor
    public ReconciliationService(
        TransactionRepository repository,
        MatchingEngine matchingEngine,
        NotificationService notificationService
    ) {
        this.repository = repository;
        this.matchingEngine = matchingEngine;
        this.notificationService = notificationService;
    }
}

// 2. Optional Dependencies
@Service
public class ReportService {
    private final ReportRepository repository;
    private final Optional<EmailService> emailService; // Optional
    
    public ReportService(
        ReportRepository repository,
        @Autowired(required = false) EmailService emailService
    ) {
        this.repository = repository;
        this.emailService = Optional.ofNullable(emailService);
    }
    
    public void generateReport(Report report) {
        repository.save(report);
        emailService.ifPresent(service -> service.send(report));
    }
}

// 3. Qualifying beans when multiple implementations exist
@Service
public class MessageProcessor {
    private final MessageQueue primaryQueue;
    private final MessageQueue backupQueue;
    
    public MessageProcessor(
        @Qualifier("ibmMq") MessageQueue primaryQueue,
        @Qualifier("oracleAq") MessageQueue backupQueue
    ) {
        this.primaryQueue = primaryQueue;
        this.backupQueue = backupQueue;
    }
}

@Configuration
public class QueueConfig {
    @Bean
    @Qualifier("ibmMq")
    public MessageQueue ibmMqQueue() {
        return new IbmMqQueue();
    }
    
    @Bean
    @Qualifier("oracleAq")
    public MessageQueue oracleAqQueue() {
        return new OracleAqQueue();
    }
}

// 4. Collection injection - All beans of type
@Service
public class CompositeValidator {
    private final List<Validator> validators;
    
    public CompositeValidator(List<Validator> validators) {
        this.validators = validators; // All Validator beans injected
    }
    
    public boolean validate(Trade trade) {
        return validators.stream()
            .allMatch(validator -> validator.isValid(trade));
    }
}

// 5. @Primary for default bean
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Primary  // This will be injected by default
    public DataSource primaryDataSource() {
        // Production Oracle DB
    }
    
    @Bean
    public DataSource reportingDataSource() {
        // Read replica
    }
}
```

#### **Profiles - Environment Management**

```java
// Profile-specific configuration
@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public DataSource dataSource() {
        // H2 in-memory DB for dev
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}

@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public DataSource dataSource() {
        // Oracle production DB
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:oracle:thin:@prod-db:1521:RECON");
        config.setMaximumPoolSize(50);
        config.setConnectionTimeout(30000);
        return new HikariDataSource(config);
    }
}

// Profile-specific properties
// application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
  jpa:
    show-sql: true

// application-prod.yml
spring:
  datasource:
    url: jdbc:oracle:thin:@prod-db:1521:RECON
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10

// Component with profile
@Service
@Profile("!prod")  // Active in all profiles except prod
public class DebugService {
    @EventListener(ApplicationReadyEvent.class)
    public void onStartup() {
        log.info("=== DEBUG MODE ACTIVE ===");
    }
}

// Activate profiles
// 1. application.properties
spring.profiles.active=dev

// 2. Command line
java -jar app.jar --spring.profiles.active=prod

// 3. Environment variable
export SPRING_PROFILES_ACTIVE=prod

// 4. Programmatically
SpringApplication app = new SpringApplication(ReconApplication.class);
app.setAdditionalProfiles("prod");
app.run(args);
```

---

### 2.2 REST API Design - Best Practices

#### **Controller Design**

```java
@RestController
@RequestMapping("/api/v1/trades")
@Validated
public class TradeController {
    
    private final TradeService tradeService;
    
    public TradeController(TradeService tradeService) {
        this.tradeService = tradeService;
    }
    
    // GET - Retrieve single resource
    @GetMapping("/{tradeId}")
    public ResponseEntity<TradeResponse> getTrade(
        @PathVariable String tradeId
    ) {
        return tradeService.findById(tradeId)
            .map(trade -> ResponseEntity.ok(toResponse(trade)))
            .orElse(ResponseEntity.notFound().build());
    }
    
    // GET - List with pagination and filtering
    @GetMapping
    public ResponseEntity<Page<TradeResponse>> getTrades(
        @RequestParam(required = false) String accountId,
        @RequestParam(required = false) String status,
        @RequestParam(required = false) 
        @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate fromDate,
        @PageableDefault(size = 20, sort = "timestamp", direction = Sort.Direction.DESC) 
        Pageable pageable
    ) {
        Page<Trade> trades = tradeService.findTrades(accountId, status, fromDate, pageable);
        Page<TradeResponse> response = trades.map(this::toResponse);
        return ResponseEntity.ok(response);
    }
    
    // POST - Create resource
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<TradeResponse> createTrade(
        @Valid @RequestBody CreateTradeRequest request
    ) {
        Trade trade = tradeService.create(request);
        
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(trade.getId())
            .toUri();
        
        return ResponseEntity
            .created(location)
            .body(toResponse(trade));
    }
    
    // PUT - Full update
    @PutMapping("/{tradeId}")
    public ResponseEntity<TradeResponse> updateTrade(
        @PathVariable String tradeId,
        @Valid @RequestBody UpdateTradeRequest request
    ) {
        return tradeService.update(tradeId, request)
            .map(trade -> ResponseEntity.ok(toResponse(trade)))
            .orElse(ResponseEntity.notFound().build());
    }
    
    // PATCH - Partial update
    @PatchMapping("/{tradeId}/status")
    public ResponseEntity<TradeResponse> updateStatus(
        @PathVariable String tradeId,
        @RequestParam String status
    ) {
        return tradeService.updateStatus(tradeId, status)
            .map(trade -> ResponseEntity.ok(toResponse(trade)))
            .orElse(ResponseEntity.notFound().build());
    }
    
    // DELETE
    @DeleteMapping("/{tradeId}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public ResponseEntity<Void> deleteTrade(@PathVariable String tradeId) {
        tradeService.delete(tradeId);
        return ResponseEntity.noContent().build();
    }
    
    private TradeResponse toResponse(Trade trade) {
        return new TradeResponse(/* map fields */);
    }
}
```

#### **Request/Response DTOs**

```java
// Request DTO with validation
@Data
public class CreateTradeRequest {
    
    @NotBlank(message = "Account ID is required")
    @Pattern(regexp = "^ACC[0-9]{8}$", message = "Invalid account ID format")
    private String accountId;
    
    @NotNull(message = "Amount is required")
    @DecimalMin(value = "0.01", message = "Amount must be positive")
    @Digits(integer = 15, fraction = 2, message = "Invalid amount format")
    private BigDecimal amount;
    
    @NotBlank(message = "Currency is required")
    @Size(min = 3, max = 3, message = "Currency must be 3 characters")
    private String currency;
    
    @NotNull(message = "Trade type is required")
    @EnumValue(enumClass = TradeType.class)
    private String tradeType;
    
    @Size(max = 500, message = "Description too long")
    private String description;
    
    @Valid  // Nested validation
    private List<@Valid SecurityAllocation> allocations;
}

// Response DTO (Java Records for immutability)
public record TradeResponse(
    String tradeId,
    String accountId,
    BigDecimal amount,
    String currency,
    String status,
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss") LocalDateTime createdAt,
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss") LocalDateTime updatedAt,
    List<SecurityAllocationResponse> allocations
) {}

// Custom validator
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = EnumValueValidator.class)
public @interface EnumValue {
    String message() default "Invalid enum value";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    Class<? extends Enum<?>> enumClass();
}

public class EnumValueValidator implements ConstraintValidator<EnumValue, String> {
    private List<String> acceptedValues;
    
    @Override
    public void initialize(EnumValue annotation) {
        acceptedValues = Arrays.stream(annotation.enumClass().getEnumConstants())
            .map(Enum::name)
            .collect(Collectors.toList());
    }
    
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return value == null || acceptedValues.contains(value);
    }
}
```

#### **Exception Handling - Centralized**

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    // Handle validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationError(
        MethodArgumentNotValidException ex
    ) {
        Map<String, String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                error -> Objects.requireNonNullElse(error.getDefaultMessage(), "Invalid value")
            ));
        
        ErrorResponse response = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Validation Failed")
            .message("Invalid request parameters")
            .validationErrors(errors)
            .build();
        
        return ResponseEntity.badRequest().body(response);
    }
    
    // Handle resource not found
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
        ResourceNotFoundException ex
    ) {
        ErrorResponse response = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.NOT_FOUND.value())
            .error("Not Found")
            .message(ex.getMessage())
            .build();
        
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(response);
    }
    
    // Handle business exceptions
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(
        BusinessException ex
    ) {
        ErrorResponse response = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.UNPROCESSABLE_ENTITY.value())
            .error("Business Rule Violation")
            .message(ex.getMessage())
            .errorCode(ex.getErrorCode())
            .build();
        
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(response);
    }
    
    // Handle database exceptions
    @ExceptionHandler(DataAccessException.class)
    public ResponseEntity<ErrorResponse> handleDatabaseError(
        DataAccessException ex
    ) {
        log.error("Database error occurred", ex);
        
        ErrorResponse response = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
            .error("Database Error")
            .message("A database error occurred. Please try again.")
            .build();
        
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
    }
    
    // Catch-all for unexpected errors
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGenericError(
        Exception ex,
        HttpServletRequest request
    ) {
        log.error("Unexpected error: {} for request: {}", 
            ex.getMessage(), request.getRequestURI(), ex);
        
        ErrorResponse response = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
            .error("Internal Server Error")
            .message("An unexpected error occurred")
            .path(request.getRequestURI())
            .build();
        
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
    }
}

@Data
@Builder
public class ErrorResponse {
    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String errorCode;
    private String path;
    private Map<String, String> validationErrors;
}
```

#### **Filters and Interceptors**

```java
// 1. Servlet Filter - Lowest level, before Spring Security
@Component
@Order(1)
public class RequestLoggingFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain
    ) throws ServletException, IOException {
        
        String requestId = UUID.randomUUID().toString();
        MDC.put("requestId", requestId);
        
        long startTime = System.currentTimeMillis();
        
        try {
            filterChain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            log.info("Request: {} {} - Status: {} - Duration: {}ms",
                request.getMethod(),
                request.getRequestURI(),
                response.getStatus(),
                duration);
            
            MDC.clear();
        }
    }
}

// 2. HandlerInterceptor - After controller mapping, before controller
@Component
public class AuthenticationInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler
    ) throws Exception {
        
        String authToken = request.getHeader("Authorization");
        
        if (authToken == null || !isValidToken(authToken)) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("Unauthorized");
            return false; // Stop the chain
        }
        
        // Add user to request context
        User user = extractUser(authToken);
        request.setAttribute("currentUser", user);
        
        return true; // Continue to controller
    }
    
    @Override
    public void postHandle(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler,
        ModelAndView modelAndView
    ) throws Exception {
        // After controller, before view rendering
    }
    
    @Override
    public void afterCompletion(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler,
        Exception ex
    ) throws Exception {
        // After everything, even if exception occurred
    }
}

@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    
    @Autowired
    private AuthenticationInterceptor authInterceptor;
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
            .addPathPatterns("/api/**")
            .excludePathPatterns("/api/public/**", "/api/health");
    }
}

// 3. Argument Resolver - Custom parameter resolution
@Component
public class CurrentUserArgumentResolver implements HandlerMethodArgumentResolver {
    
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(CurrentUser.class);
    }
    
    @Override
    public Object resolveArgument(
        MethodParameter parameter,
        ModelAndViewContainer mavContainer,
        NativeWebRequest webRequest,
        WebDataBinderFactory binderFactory
    ) throws Exception {
        
        HttpServletRequest request = 
            (HttpServletRequest) webRequest.getNativeRequest();
        return request.getAttribute("currentUser");
    }
}

// Usage in controller
@GetMapping("/me")
public ResponseEntity<UserResponse> getCurrentUser(@CurrentUser User user) {
    return ResponseEntity.ok(toResponse(user));
}
```

---

This is Part 2 Section 1-2. The document is getting extensive. Should I continue with the remaining sections (2.3 onwards for Spring Boot, then all other sections)? 

The complete document will be around 20,000+ lines covering all 11 sections comprehensively as you requested. Let me know if you want me to continue generating the rest or if you'd like any modifications to what's been created so far.
