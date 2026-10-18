**SQL + Java/Spring Boot** is a powerful combination for building SAP interfaces: Java/Spring Boot acts as the integration service layer, SQL handles reliable data storage and reporting, and SAP exposes business data or operations through interfaces such as **OData**, **RFC/BAPI**, **IDoc**, or **SAP Integration Suite**. In airline terms, Spring Boot can be the “connectivity and orchestration layer” between an airline’s booking website, airport systems, cargo platform, and SAP ERP. [docs.spring](https://docs.spring.io/spring-boot/reference/data/sql.html)

## Core abbreviations

| Term | Full form | Plain-English meaning |
|---|---|---|
| **SQL** | Structured Query Language | Language used to store, retrieve, update, and analyze relational data |
| **JDBC** | Java Database Connectivity | Java’s standard API for connecting to databases |
| **JPA** | Java Persistence API | Java specification for mapping Java objects to database tables |
| **ORM** | Object-Relational Mapping | Automatically maps classes/objects to tables/rows |
| **Spring Boot** | Spring Boot framework | Java framework for quickly building production-ready backend services |
| **REST** | Representational State Transfer | HTTP-based API style using resources and JSON |
| **OData** | Open Data Protocol | SAP-friendly REST-like protocol for reading and changing business data |
| **RFC** | Remote Function Call | SAP’s standard mechanism to call a function in a remote SAP system |
| **BAPI** | Business Application Programming Interface | SAP’s standardized business-level API, usually invoked through RFC |
| **IDoc** | Intermediate Document | SAP’s structured document format for asynchronous business messages |
| **ALE** | Application Link Enabling | SAP technology for distributing business data across systems |
| **EDI** | Electronic Data Interchange | Standardized computer-to-computer exchange of business documents |
| **CPI** | Cloud Platform Integration | Older name for SAP Integration Suite’s Cloud Integration capability |
| **PI/PO** | Process Integration / Process Orchestration | SAP’s on-premise integration middleware |
| **SAP BTP** | SAP Business Technology Platform | SAP’s cloud platform for apps, integrations, APIs, and services |
| **CAP** | Cloud Application Programming Model | SAP’s framework for building enterprise services and apps |

SAP describes **BAPIs** as standard business interfaces for exchanging business data between SAP and non-SAP components, while **RFC** is the standard interface for communication between SAP systems, and **IDoc** is SAP’s standard format for EDI-style document exchange. [learning.sap](https://learning.sap.com/courses/developing-integration-scenarios-using-idoc-rfc-adapter-of-sap-process-orchestration/explaining-the-integration-technology-based-on-bapis-and-idocs-basics_db458c59-a70f-480d-b139-65065be1b9e9)

## Where this fits in an airline landscape

An airline typically has many systems:

| System | Example purpose |
|---|---|
| SAP S/4HANA | Finance, procurement, vendor invoices, material/parts management |
| Booking system | Passenger reservations and ticketing |
| Airport operations system | Gates, slots, baggage, ground handling |
| Cargo system | Air waybills, shipments, customs documents |
| Loyalty system | Frequent-flyer points and tiers |
| Data warehouse | Operational and financial reporting |

Spring Boot sits between these systems:

```text
Booking Website / Airport App / Cargo Portal
                 |
          Spring Boot API Layer
                 |
   SQL Database + SAP Integration Layer
                 |
       SAP S/4HANA / SAP BTP / SAP Integration Suite
```

For example, when a passenger buys an extra baggage allowance:

1. The booking website calls a Spring Boot REST API.
2. Spring Boot validates the request and stores an integration audit record in SQL.
3. Spring Boot calls SAP through OData, BAPI, or an SAP Integration Suite endpoint.
4. SAP creates or updates the billing document.
5. Spring Boot records success/failure and returns a response to the website.

## Why SQL matters

SQL is essential because integrations need durable, queryable data—not just temporary JSON in memory.

Use SQL for:

- **Integration audit logs** — what was sent, when, to which SAP system, and with what status.
- **Idempotency keys** — prevent duplicate SAP document creation if a message is retried.
- **Outbox/inbox tables** — reliably queue outbound and inbound messages.
- **Reference/master data** — airports, airlines, aircraft types, charge codes, tax codes.
- **Reconciliation reports** — compare bookings in the airline system with invoices in SAP.
- **Error analysis** — identify failed baggage-fee postings or missing vendor invoices.

Spring Boot supports SQL through `JdbcTemplate`, `JdbcClient`, Spring Data JPA/Hibernate, and Spring Data JDBC. [docs.spring](https://docs.spring.io/spring-boot/reference/data/sql.html)

## SQL basics for SAP integrations

### Create integration tables

```sql
CREATE TABLE integration_message (
    message_id        UUID PRIMARY KEY,
    idempotency_key   VARCHAR(100) NOT NULL UNIQUE,
    source_system     VARCHAR(50) NOT NULL,
    target_system     VARCHAR(50) NOT NULL,
    business_object   VARCHAR(50) NOT NULL,
    payload           JSONB NOT NULL,
    status            VARCHAR(20) NOT NULL,
    sap_document_no   VARCHAR(30),
    error_message     TEXT,
    retry_count       INT DEFAULT 0,
    created_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    processed_at      TIMESTAMP
);
```

Airline examples of `business_object`:

- `BAGGAGE_FEE`
- `FLIGHT_DELAY`
- `CARGO_INVOICE`
- `AIRPORT_SLOT`
- `VENDOR_INVOICE`

### Insert an outbound message

```sql
INSERT INTO integration_message (
    message_id,
    idempotency_key,
    source_system,
    target_system,
    business_object,
    payload,
    status
)
VALUES (
    gen_random_uuid(),
    'AI202-BAG-98765',
    'BOOKING_SYSTEM',
    'SAP_S4HANA',
    'BAGGAGE_FEE',
    '{"bookingId":"BK98765","flightNo":"AI202","extraKg":4,"feeInr":2000}',
    'PENDING'
);
```

### Query failed messages

```sql
SELECT
    message_id,
    idempotency_key,
    business_object,
    error_message,
    retry_count,
    created_at
FROM integration_message
WHERE target_system = 'SAP_S4HANA'
  AND status = 'FAILED'
  AND retry_count < 5
ORDER BY created_at;
```

### Reconcile bookings with SAP invoices

```sql
SELECT
    b.booking_id,
    b.passenger_id,
    b.total_amount,
    i.sap_invoice_number,
    i.invoice_amount
FROM booking b
LEFT JOIN sap_invoice i
       ON i.booking_id = b.booking_id
WHERE b.booking_date >= CURRENT_DATE - INTERVAL '7 days'
  AND (i.sap_invoice_number IS NULL
       OR i.invoice_amount <> b.total_amount);
```

This finds bookings with missing or mismatched SAP invoices.

## SQL advanced concepts

### Joins

Join flight, booking, and passenger data:

```sql
SELECT
    f.flight_no,
    f.origin_airport,
    f.destination_airport,
    b.booking_id,
    p.passenger_name,
    b.baggage_weight_kg
FROM flight f
JOIN booking b ON b.flight_id = f.flight_id
JOIN passenger p ON p.passenger_id = b.passenger_id
WHERE f.origin_airport = 'MAA'
  AND f.flight_date = CURRENT_DATE;
```

### Aggregations

Find total excess-baggage revenue by airport:

```sql
SELECT
    f.origin_airport,
    SUM(b.extra_baggage_fee) AS total_extra_fee,
    COUNT(*) AS baggage_count
FROM booking b
JOIN flight f ON f.flight_id = b.flight_id
WHERE b.extra_baggage_fee > 0
GROUP BY f.origin_airport
ORDER BY total_extra_fee DESC;
```

### Window functions

Rank delayed flights by delay duration:

```sql
SELECT
    flight_no,
    origin_airport,
    destination_airport,
    delay_minutes,
    RANK() OVER (
        PARTITION BY origin_airport
        ORDER BY delay_minutes DESC
    ) AS delay_rank
FROM flight_delay
WHERE delay_date = CURRENT_DATE;
```

### Transactions

A transaction ensures multiple database changes succeed or fail together.

```sql
BEGIN;

UPDATE integration_message
SET status = 'PROCESSING'
WHERE idempotency_key = 'AI202-BAG-98765';

INSERT INTO integration_audit (
    message_id,
    action,
    details
)
SELECT
    message_id,
    'STARTED',
    'Sending baggage fee to SAP'
FROM integration_message
WHERE idempotency_key = 'AI202-BAG-98765';

COMMIT;
```

In Spring Boot, use `@Transactional` so the same guarantee is handled by the framework.

## Java/Spring Boot architecture

A clean integration service should separate concerns:

```text
Controller  →  Service  →  SAP Client  →  SAP
     ↓              ↓             ↓
   DTOs       Domain logic   OData/RFC/IDoc
                    ↓
              Repository → SQL Database
```

| Layer | Responsibility | Airline example |
|---|---|---|
| Controller | HTTP request/response | `POST /api/v1/baggage-fees` |
| DTO | Data transfer object | `BaggageFeeRequest`, `SapPostingResponse` |
| Service | Business orchestration | Validate booking, calculate fee, call SAP |
| Repository | Database access | Save integration message and audit trail |
| SAP client | SAP communication | Call OData/BAPI/IDoc endpoint |
| Mapper | Convert formats | Convert booking JSON to SAP payload |

## Spring Boot project structure

```text
airline-sap-integration/
├── src/main/java/com/airline/integration/
│   ├── AirlineSapIntegrationApplication.java
│   ├── controller/
│   │   └── BaggageFeeController.java
│   ├── service/
│   │   └── BaggageFeeIntegrationService.java
│   ├── sap/
│   │   ├── SapODataClient.java
│   │   └── SapRfcClient.java
│   ├── repository/
│   │   └── IntegrationMessageRepository.java
│   ├── model/
│   │   ├── IntegrationMessage.java
│   │   └── Booking.java
│   ├── dto/
│   │   ├── BaggageFeeRequest.java
│   │   └── SapPostingResponse.java
│   └── config/
│       ├── SapProperties.java
│       └── RestClientConfig.java
└── src/main/resources/
    ├── application.yml
    └── db/migration/
        └── V1__create_integration_tables.sql
```

## Spring Boot dependencies

Typical Maven dependencies:

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>

  <dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
  </dependency>

  <dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
  </dependency>
</dependencies>
```

Spring Boot provides extensive SQL support, from direct JDBC access with `JdbcClient` or `JdbcTemplate` to ORM technologies such as Hibernate. [docs.spring](https://docs.spring.io/spring-boot/reference/data/sql.html)

## Configuration

`application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/airline_integration
    username: airline_user
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false

sap:
  base-url: https://sap-airline.example.com
  odata-path: /sap/opu/odata/sap/FLIGHT_BOOKING_SRV
  client-id: ${SAP_CLIENT_ID}
  client-secret: ${SAP_CLIENT_SECRET}
  token-url: https://sap-airline.example.com/oauth/token
```

Never hard-code SAP credentials. Use environment variables, Kubernetes secrets, AWS Secrets Manager, Azure Key Vault, or SAP BTP destination services.

## REST controller

```java
@RestController
@RequestMapping("/api/v1/baggage-fees")
public class BaggageFeeController {

    private final BaggageFeeIntegrationService service;

    public BaggageFeeController(BaggageFeeIntegrationService service) {
        this.service = service;
    }

    @PostMapping
    public ResponseEntity<SapPostingResponse> postBaggageFee(
            @Valid @RequestBody BaggageFeeRequest request) {

        SapPostingResponse response =
            service.sendBaggageFeeToSap(request);

        return ResponseEntity.accepted().body(response);
    }
}
```

Request DTO:

```java
public record BaggageFeeRequest(
        @NotBlank String bookingId,
        @NotBlank String flightNo,
        @NotBlank String passengerId,
        @NotNull @Positive BigDecimal extraWeightKg,
        @NotNull @Positive BigDecimal feeInr
) {}
```

Example request:

```json
{
  "bookingId": "BK98765",
  "flightNo": "AI202",
  "passengerId": "PAX45678",
  "extraWeightKg": 4,
  "feeInr": 2000
}
```

## Service layer

```java
@Service
public class BaggageFeeIntegrationService {

    private final IntegrationMessageRepository repository;
    private final SapODataClient sapClient;
    private final ObjectMapper objectMapper;

    public BaggageFeeIntegrationService(
            IntegrationMessageRepository repository,
            SapODataClient sapClient,
            ObjectMapper objectMapper) {
        this.repository = repository;
        this.sapClient = sapClient;
        this.objectMapper = objectMapper;
    }

    @Transactional
    public SapPostingResponse sendBaggageFeeToSap(
            BaggageFeeRequest request) {

        IntegrationMessage message = new IntegrationMessage();
        message.setIdempotencyKey(
            request.flightNo() + "-BAG-" + request.bookingId());
        message.setSourceSystem("BOOKING_SYSTEM");
        message.setTargetSystem("SAP_S4HANA");
        message.setBusinessObject("BAGGAGE_FEE");
        message.setStatus("PENDING");

        try {
            message.setPayload(objectMapper.writeValueAsString(request));
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("Unable to serialize request", e);
        }

        repository.save(message);

        try {
            SapPostingResponse response =
                sapClient.postBaggageFee(request);

            message.setStatus("SUCCESS");
            message.setSapDocumentNo(response.sapDocumentNumber());
            message.setProcessedAt(Instant.now());
            repository.save(message);

            return response;

        } catch (Exception e) {
            message.setStatus("FAILED");
            message.setErrorMessage(e.getMessage());
            message.setRetryCount(message.getRetryCount() + 1);
            repository.save(message);

            throw new SapIntegrationException(
                "SAP baggage-fee posting failed", e);
        }
    }
}
```

This design gives you:

- A persistent audit trail.
- An idempotency key to prevent duplicate SAP postings.
- Clear success/failure tracking.
- Retry support.
- A clean separation between HTTP, business logic, SAP communication, and SQL persistence.

## SAP integration options

| Interface | Direction | Best for | Airline example |
|---|---|---|---|
| **OData** | Both | Modern REST-style SAP APIs | Read flights, update gate, create baggage-fee posting |
| **RFC** | Usually inbound to SAP | Direct remote function call | Check material availability for aircraft spare parts |
| **BAPI** | Usually inbound to SAP | Standard business-object operations | Create purchase order for jet fuel or spare parts |
| **IDoc** | Both, asynchronous | Document exchange and EDI | Send cargo airway bill or vendor invoice |
| **SOAP** | Both | Legacy enterprise integrations | Older airline or airport partner systems |
| **SAP Integration Suite / CPI** | Both | Mapping, orchestration, connectivity | Connect booking system, SAP, cargo partner, and airport system |
| **JDBC to SAP HANA** | Read/write database | Reporting or approved data access | Operational dashboard over flight and invoice data |

SAP Cloud Integration is designed to integrate business processes across companies, organizations, and departments, and SAP Integration Suite connects SAP cloud applications, third-party applications, and on-premise systems. [help.sap](https://help.sap.com/docs/cloud-integration/sap-cloud-integration/sap-cloud-integration)

## OData integration from Spring Boot

OData is often the preferred modern integration option for SAP S/4HANA because it gives external applications a business-level, REST-like API. [eproxim](https://eproxim.com/sap-integration-options-idoc-bapi-odata-cpi/)

### SAP OData endpoints

```text
GET  /odata/v4/flight/Flights?$filter=Origin eq 'MAA'
GET  /odata/v4/flight/Flights('AI202')
POST /odata/v4/flight/BaggageFees
PATCH /odata/v4/flight/Flights('AI202')
```

### Spring RestClient client

```java
@Component
public class SapODataClient {

    private final RestClient restClient;

    public SapODataClient(RestClient.Builder builder,
                          SapProperties properties) {
        this.restClient = builder
            .baseUrl(properties.baseUrl() + properties.odataPath())
            .build();
    }

    public SapPostingResponse postBaggageFee(BaggageFeeRequest request) {
        Map<String, Object> payload = Map.of(
            "bookingId", request.bookingId(),
            "flightNo", request.flightNo(),
            "passengerId", request.passengerId(),
            "extraWeightKg", request.extraWeightKg(),
            "feeAmount", request.feeInr(),
            "currency", "INR"
        );

        return restClient.post()
            .uri("/BaggageFees")
            .contentType(MediaType.APPLICATION_JSON)
            .body(payload)
            .retrieve()
            .body(SapPostingResponse.class);
    }
}
```

### Read flights with filters

```java
public List<FlightDto> getDelayedFlights(String originAirport) {
    String uri = UriComponentsBuilder
        .fromPath("/Flights")
        .queryParam("$filter", "Origin eq '{origin}' and Status eq 'DELAYED'")
        .queryParam("$select", "FlightNo,Origin,Destination,Status,Gate")
        .buildAndExpand(originAirport)
        .toUriString();

    ODataResponse<FlightDto> response = restClient.get()
        .uri(uri)
        .retrieve()
        .body(new ParameterizedTypeReference<>() {});

    return response.value();
}
```

## RFC and BAPI integration

**RFC** lets an external program call a function in a remote SAP system. **BAPIs** are SAP’s standardized business interfaces and are implemented as RFC-enabled function modules. [learning.sap](https://learning.sap.com/courses/developing-integration-scenarios-using-idoc-rfc-adapter-of-sap-process-orchestration/explaining-the-integration-technology-based-on-bapis-and-idocs-basics_db458c59-a70f-480d-b139-65065be1b9e9)

Use RFC/BAPI when:

- SAP does not expose a modern OData API for the operation.
- You need a synchronous SAP transactional operation.
- You must reuse a stable, SAP-provided business API.
- The integration is tightly coupled to SAP business-object behavior.

Airline examples:

| BAPI/RFC use case | SAP business object |
|---|---|
| Create purchase order for aircraft spare parts | Purchase Order |
| Check stock of landing-gear components | Material / Inventory |
| Post vendor invoice for airport charges | Accounts Payable |
| Create customer invoice for cargo shipment | Billing Document |
| Check cost center budget for airport slot fees | Cost Center / Finance |

A Java application may connect through an SAP JCo (**Java Connector**) library, an SAP RFC connector, or—more commonly in modern landscapes—through middleware such as SAP Integration Suite.

## IDoc integration

An **IDoc** is a structured business document used for asynchronous exchange between SAP and other systems. SAP identifies IDocs as a standard format for transferring data between SAP and non-SAP systems. [learning.sap](https://learning.sap.com/courses/developing-integration-scenarios-using-idoc-rfc-adapter-of-sap-process-orchestration/explaining-the-integration-technology-based-on-bapis-and-idocs-basics_db458c59-a70f-480d-b139-65065be1b9e9)

Airline IDoc examples:

| IDoc scenario | Direction | Example |
|---|---|---|
| Cargo airway bill | Airline cargo system → SAP | Create billing document |
| Vendor invoice | Airport/fuel vendor → SAP | Accounts-payable posting |
| Purchase order confirmation | SAP → procurement portal | Confirm spare-part order |
| Flight-related cost posting | Airport system → SAP | Landing, parking, and handling charges |
| Master data distribution | SAP → airline systems | Vendor, material, or customer master data |

### IDoc processing pattern

```text
Cargo System
     |
     | JSON / XML
     v
Spring Boot API
     |
     | Validate + persist message
     v
SAP Integration Suite / PI PO
     |
     | Map JSON/XML to IDoc
     v
SAP S/4HANA
```

Spring Boot should not usually generate raw IDoc segments itself unless there is a strong reason. Let SAP Integration Suite or PI/PO handle protocol conversion, mapping, retries, and monitoring; keep Spring Boot focused on API design, validation, business orchestration, and persistence.

## SAP Integration Suite and middleware

**SAP Integration Suite**, formerly commonly called **SAP Cloud Platform Integration (CPI)**, provides managed integration flows and connectivity between cloud, on-premise, SAP, and non-SAP systems. [support.sap](https://support.sap.com/en/alm/solution-manager/expert-portal/public-cloud-operations/sap-cloud-platform-integration.html)

Use middleware when you need:

- Protocol conversion: JSON ↔ XML ↔ IDoc ↔ SOAP.
- Message mapping: airline booking format → SAP format.
- Orchestration: call multiple systems in sequence.
- Retries, dead-letter queues, and monitoring.
- Security policies, throttling, and partner onboarding.
- Decoupling: booking system should not need to know SAP’s internal format.

Example airline flow:

```text
Booking System
   → Spring Boot API
   → SAP Integration Suite
   → SAP S/4HANA
   → SAP Convergent Invoicing / Finance
```

Spring Boot can expose a stable airline API:

```http
POST /api/v1/baggage-fees
```

while SAP Integration Suite maps it to SAP’s expected payload and protocol.

## JPA and SQL in Spring Boot

### Entity

```java
@Entity
@Table(name = "integration_message")
public class IntegrationMessage {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID messageId;

    @Column(nullable = false, unique = true)
    private String idempotencyKey;

    private String sourceSystem;
    private String targetSystem;
    private String businessObject;

    @Column(columnDefinition = "jsonb")
    private String payload;

    private String status;
    private String sapDocumentNo;
    private String errorMessage;

    private int retryCount;

    private Instant createdAt;
    private Instant processedAt;
}
```

### Repository

```java
public interface IntegrationMessageRepository
        extends JpaRepository<IntegrationMessage, UUID> {

    List<IntegrationMessage> findByStatusAndRetryCountLessThan(
            String status, int maxRetryCount);

    Optional<IntegrationMessage> findByIdempotencyKey(String key);
}
```

### Custom query

```java
@Query("""
    SELECT m
    FROM IntegrationMessage m
    WHERE m.targetSystem = :targetSystem
      AND m.status = 'FAILED'
      AND m.retryCount < :maxRetryCount
    ORDER BY m.createdAt ASC
""")
List<IntegrationMessage> findRetryableFailures(
        String targetSystem, int maxRetryCount);
```

## Transaction and idempotency design

A robust SAP interface must handle retries safely.

### Problem

The booking system sends a baggage fee to SAP. The network times out. The booking system retries. Without protection, SAP may create two billing documents.

### Solution: idempotency

Before calling SAP:

```java
Optional<IntegrationMessage> existing =
    repository.findByIdempotencyKey(request.flightNo() + "-BAG-" + request.bookingId());

if (existing.isPresent()
        && existing.get().getStatus().equals("SUCCESS")) {
    return new SapPostingResponse(
        existing.get().getSapDocumentNo(),
        "ALREADY_PROCESSED"
    );
}
```

The idempotency key should be based on a stable business identity:

```text
AI202-BAG-BK98765
```

Not on a random UUID, because a retry would generate a new UUID and bypass duplicate protection.

## Error handling

### Custom exception

```java
public class SapIntegrationException extends RuntimeException {

    private final String errorCode;

    public SapIntegrationException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    public String getErrorCode() {
        return errorCode;
    }
}
```

### Global handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(SapIntegrationException.class)
    public ResponseEntity<ApiErrorResponse> handleSapError(
            SapIntegrationException ex) {

        return ResponseEntity.status(HttpStatus.BAD_GATEWAY).body(
            new ApiErrorResponse(
                ex.getErrorCode(),
                ex.getMessage(),
                Instant.now()
            )
        );
    }
}
```

Response:

```json
{
  "errorCode": "SAP_BAGGAGE_FEE_FAILED",
  "message": "SAP rejected baggage-fee posting: invalid tax code",
  "timestamp": "2026-10-18T15:00:00Z"
}
```

## Resilience patterns

| Pattern | Meaning | Airline example |
|---|---|---|
| **Retry** | Try again after a temporary failure | SAP gateway briefly unavailable |
| **Circuit breaker** | Stop calling an unhealthy system temporarily | SAP system is down; avoid flooding it |
| **Timeout** | Do not wait forever | SAP call must return within 10 seconds |
| **Outbox pattern** | Save message first, send asynchronously | Ensure booking event is never lost |
| **Dead-letter queue** | Store repeatedly failed messages | Failed cargo invoices for manual review |
| **Reconciliation job** | Compare source and SAP data nightly | Detect missing invoices |
| **Rate limiting** | Control request volume | Avoid overwhelming SAP during mass rebooking |

Use Spring Retry, Resilience4j, or a message broker such as Kafka/RabbitMQ for advanced patterns.

## Scheduled jobs

A Spring scheduler can retry failed SAP messages.

```java
@Component
public class SapRetryScheduler {

    private final IntegrationMessageRepository repository;
    private final BaggageFeeIntegrationService service;

    public SapRetryScheduler(IntegrationMessageRepository repository,
                             BaggageFeeIntegrationService service) {
        this.repository = repository;
        this.service = service;
    }

    @Scheduled(fixedDelay = 300000)
    public void retryFailedMessages() {
        List<IntegrationMessage> messages =
            repository.findByStatusAndRetryCountLessThan("FAILED", 5);

        messages.forEach(message -> {
            try {
                service.retry(message);
            } catch (Exception e) {
                // Log and continue with next message
            }
        });
    }
}
```

## Security

For SAP integrations, security is not optional.

- Use **OAuth 2.0** or **SAML** where supported.
- Store credentials in a secrets manager, not in Git.
- Use **mTLS** (**mutual Transport Layer Security**) for high-trust system-to-system communication.
- Apply least-privilege SAP authorization roles.
- Validate and sanitize every inbound payload.
- Log business identifiers, not sensitive passenger data such as full passport numbers.
- Encrypt data in transit and at rest.
- Use API gateways for rate limiting, authentication, and audit logging.

## Testing strategy

| Test type | Tool | Example |
|---|---|---|
| Unit test | JUnit + Mockito | Fee calculation and mapping logic |
| Repository test | Testcontainers + PostgreSQL | Save and query integration messages |
| API test | MockMvc / WebTestClient | `POST /api/v1/baggage-fees` |
| SAP client test | WireMock / MockWebServer | Simulate SAP OData success and failure |
| Integration test | Testcontainers | Full Spring Boot + database flow |
| Contract test | Pact / OpenAPI validation | Booking system ↔ Spring Boot contract |
| Reconciliation test | SQL test data | Detect missing SAP invoices |

Example unit test:

```java
@Test
void shouldCalculateExtraBaggageFee() {
    BigDecimal fee = BaggageFeeCalculator.calculate(
        new BigDecimal("27"),
        new BigDecimal("23"),
        new BigDecimal("500")
    );

    assertEquals(new BigDecimal("2000"), fee);
}
```

## Performance practices

- Push filtering and aggregation to the database using SQL.
- Use indexes on `idempotency_key`, `status`, `target_system`, and `created_at`.
- Use pagination for large reconciliation queries.
- Avoid N+1 queries in JPA.
- Use connection pooling, such as HikariCP.
- Set practical timeouts for SAP calls.
- Batch high-volume operations where SAP supports it.
- Cache stable reference data such as airport codes and tax codes.
- Monitor integration latency, failure rate, retry count, and SAP response time.

## Clean-core principle

SAP’s **clean core** philosophy favors standard, upgrade-stable APIs and extensions over heavy custom code inside the ERP core. Modern SAP guidance encourages moving away from tightly coupled legacy BAPI-based interfaces toward standardized, lifecycle-managed APIs where possible. [community.sap](https://community.sap.com/t5/technology-blog-posts-by-sap/modernizing-rfc-bapi-based-integrations-for-a-clean-core-with-sap/ba-p/14240582)

For an airline, this means:

- Prefer OData APIs and SAP Integration Suite for new integrations.
- Use BAPI/RFC only where no modern equivalent exists.
- Keep airline-specific rules in a separate integration or extension layer.
- Avoid direct database writes into SAP tables.
- Avoid copying SAP business logic into Spring Boot.
- Use SAP as the system of record for SAP-owned business objects.

## Learning path

1. **SQL fundamentals** — SELECT, JOIN, GROUP BY, transactions, indexes.
2. **Advanced SQL** — window functions, CTEs, query plans, reconciliation queries.
3. **Java fundamentals** — records, streams, exceptions, concurrency basics.
4. **Spring Boot** — REST controllers, dependency injection, configuration, validation.
5. **Spring Data** — JPA, Spring Data JDBC, transactions, migrations.
6. **API design** — REST, OpenAPI, DTOs, pagination, error contracts.
7. **SAP integration concepts** — OData, RFC, BAPI, IDoc, ALE, SAP Gateway.
8. **SAP Integration Suite** — integration flows, mappings, adapters, monitoring.
9. **Resilience** — retries, circuit breakers, outbox, idempotency.
10. **Security and observability** — OAuth, mTLS, logging, metrics, tracing.
11. **SAP BTP and CAP Java** — cloud-native SAP extension development.

## Java-to-SAP skill mapping

| Your Java/Spring skill | SAP integration use |
|---|---|
| Spring Boot REST API | Expose airline APIs to booking, airport, and cargo systems |
| Spring Data JPA | Persist integration messages, audit logs, and reconciliation data |
| SQL and PostgreSQL | Build reliable integration stores and reporting queries |
| Spring Security | Secure APIs with OAuth2/JWT and role-based access |
| Resilience4j | Handle SAP downtime and retry safely |
| Kafka/RabbitMQ | Build event-driven airline integration flows |
| Testcontainers | Test real database behavior |
| Docker/Kubernetes | Deploy integration services reliably |
| OpenAPI | Define stable contracts for airline and airport partners |

## Mini end-to-end example

**Scenario:** Passenger at Chennai Airport has 27 kg baggage. Free allowance is 23 kg. Extra fee is ₹2,000. The airline system must post this fee to SAP.

### 1. Booking system calls Spring Boot

```http
POST /api/v1/baggage-fees
Content-Type: application/json

{
  "bookingId": "BK98765",
  "flightNo": "AI202",
  "passengerId": "PAX45678",
  "extraWeightKg": 4,
  "feeInr": 2000
}
```

### 2. Spring Boot validates and stores message

```sql
INSERT INTO integration_message (
    idempotency_key,
    source_system,
    target_system,
    business_object,
    payload,
    status
)
VALUES (
    'AI202-BAG-BK98765',
    'BOOKING_SYSTEM',
    'SAP_S4HANA',
    'BAGGAGE_FEE',
    '{"bookingId":"BK98765","feeInr":2000}',
    'PENDING'
);
```

### 3. Spring Boot calls SAP OData

```http
POST /odata/v4/flight/BaggageFees
Content-Type: application/json

{
  "bookingId": "BK98765",
  "flightNo": "AI202",
  "feeAmount": 2000,
  "currency": "INR"
}
```

### 4. SAP returns document number

```json
{
  "sapDocumentNumber": "5100012345",
  "status": "POSTED"
}
```

### 5. Spring Boot updates SQL

```sql
UPDATE integration_message
SET status = 'SUCCESS',
    sap_document_no = '5100012345',
    processed_at = CURRENT_TIMESTAMP
WHERE idempotency_key = 'AI202-BAG-BK98765';
```

### 6. Booking system receives response

```json
{
  "sapDocumentNumber": "5100012345",
  "status": "POSTED"
}
```

## What this role actually does

An SAP interface developer using SQL and Java/Spring Boot typically:

- Designs APIs between airline, airport, cargo, finance, and SAP systems.
- Builds Spring Boot services for orchestration, validation, and transformation.
- Writes SQL for integration logs, reconciliation, reporting, and data quality checks.
- Integrates SAP through OData, RFC/BAPI, IDoc, SOAP, or SAP Integration Suite.
- Implements retries, idempotency, monitoring, and error recovery.
- Works with SAP functional and ABAP teams to define business-object ownership.
- Ensures integrations remain secure, observable, and upgrade-safe.

The best mental model is: **SQL is the system’s memory, Spring Boot is its coordinator, and SAP is the authoritative business system.** SAP’s BAPI/RFC/IDoc/OData options serve different integration needs, while SAP Integration Suite provides managed connectivity and process integration across SAP and non-SAP landscapes. [learning.sap](https://learning.sap.com/courses/developing-integration-scenarios-using-idoc-rfc-adapter-of-sap-process-orchestration/explaining-the-integration-technology-based-on-bapis-and-idocs-basics_db458c59-a70f-480d-b139-65065be1b9e9)
