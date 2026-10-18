**OData** (**Open Data Protocol**) is an open, REST-based standard for exposing and consuming business data through HTTP APIs. In SAP, it is the standard way Fiori apps, external Java/Spring Boot services, mobile apps, and partner systems read or change SAP business data—such as flights, bookings, passengers, airports, and baggage records. [docs.oasis-open](https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part1-protocol.html)

## What OData means

Think of OData as a **standardized, queryable REST API** for business data.

- Plain REST might expose `/flights` and require each developer to invent filtering, paging, and metadata rules.
- OData defines those rules consistently: how to filter, sort, paginate, select fields, navigate related data, and describe the data model.
- A client can discover what data and operations exist by reading the service’s **metadata document**.

OData is an application-level protocol for interacting with data through RESTful interfaces; it supports describing data models and querying/editing data according to those models. [docs.oasis-open](https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part1-protocol.html)

## Airline analogy

Imagine an airline exposes a **Flight Information API**.

Without OData:

```http
GET /api/flights?from=MAA&status=DELAYED&page=1
```

Every API may use different parameter names: `from`, `origin`, `departureAirport`, `page`, `limit`, and so on.

With OData:

```http
GET /odata/v4/flight/Flights
  ?$filter=Origin eq 'MAA' and Status eq 'DELAYED'
  &$select=FlightNo,Origin,Destination,Status
  &$top=20
```

Any OData-aware client—SAPUI5, Java, Python, Power BI, Postman—can understand the same query conventions. OData provides database-like access to server-side resources while following REST principles. [learning.sap](https://learning.sap.com/courses/building-odata-services-with-sap-gateway/explaining-open-data-protocol-odata-)

## Core abbreviations

| Term | Full form | Meaning |
|---|---|---|
| **OData** | Open Data Protocol | Open REST-based data API standard |
| **HTTP** | HyperText Transfer Protocol | Web communication protocol |
| **REST** | Representational State Transfer | API style based on resources and HTTP methods |
| **JSON** | JavaScript Object Notation | Common lightweight data format |
| **XML** | eXtensible Markup Language | Structured markup format, common in older SAP services |
| **CSDL** | Common Schema Definition Language | OData metadata/model definition language |
| **EDM** | Entity Data Model | OData’s data model: entities, properties, relationships |
| **CRUD** | Create, Read, Update, Delete | Basic data operations |
| **CSRF** | Cross-Site Request Forgery | Security attack OData services must protect against |
| **ETag** | Entity Tag | Version marker used for optimistic concurrency |
| **SAP Gateway** | SAP Gateway | SAP component that exposes ABAP backend data as OData |
| **RAP** | ABAP RESTful Application Programming Model | Modern ABAP framework for OData V4 services |
| **CDS** | Core Data Services | SAP data-modeling layer often exposed through OData |
| **BTP** | SAP Business Technology Platform | SAP cloud platform for services, APIs, and extensions |

## OData building blocks

### Entity

An **entity** is a business object with a unique key.

| Entity | Key | Airline example |
|---|---|---|
| `Flight` | `FlightNo` + `FlightDate` | AI202 on 18-Oct-2026 |
| `Airport` | `AirportCode` | MAA |
| `Passenger` | `PassengerId` | PAX45678 |
| `Booking` | `BookingId` | BK98765 |
| `Baggage` | `BaggageTagNo` | AI123456 |

### Entity set

An **entity set** is a collection of entities.

```text
/Flights
/Airports
/Passengers
/Bookings
/Baggage
```

### Entity type

An **entity type** describes the fields of an entity.

```text
Flight
├── FlightNo
├── FlightDate
├── Origin
├── Destination
├── Status
├── ScheduledDeparture
└── Gate
```

### Navigation property

A **navigation property** links related entities.

```text
Flight → Bookings
Booking → Passenger
Booking → Baggage
Airport → DepartingFlights
```

For example, one flight can have many bookings; one booking belongs to one passenger.

### Metadata document

The metadata describes the API’s entities, fields, relationships, and operations.

```http
GET /odata/v4/flight/$metadata
```

A client can inspect this to learn that `Flights` exists, that `FlightNo` is a key, and that `Flight` has a navigation property to `Bookings`. OData’s metadata document represents the service’s data model for clients. [docs.oasis-open](https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part1-protocol.html)

## HTTP methods in OData

| HTTP method | OData operation | SQL equivalent | Airline example |
|---|---|---|---|
| `GET` | Read | `SELECT` | Get delayed flights from MAA |
| `POST` | Create | `INSERT` | Create a baggage-fee record |
| `PUT` | Replace | `UPDATE` full record | Replace complete flight-delay record |
| `PATCH` | Partial update | `UPDATE` selected fields | Change only the gate |
| `DELETE` | Delete | `DELETE` | Cancel a ground-service request |

OData uses standard HTTP semantics, so it fits naturally with browsers, mobile apps, SAPUI5, and Java/Spring Boot clients. [docs.oasis-open](https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part1-protocol.html)

## OData URL anatomy

```text
https://sap-airline.example.com
  /odata/v4
  /flight
  /Flights('AI202')
  ?$select=FlightNo,Status,Gate
  &$expand=Bookings
```

| Part | Meaning |
|---|---|
| `https://sap-airline.example.com` | SAP system or API gateway host |
| `/odata/v4` | OData version |
| `/flight` | Service namespace or service root |
| `/Flights` | Entity set |
| `('AI202')` | Key of one flight |
| `?$select=...` | Return only selected fields |
| `&$expand=Bookings` | Include related bookings |

## Query options

OData query options begin with `$`. They let the client control what data is returned without creating a separate endpoint for every need. Standard options include `$filter`, `$select`, `$orderby`, `$top`, `$skip`, `$count`, and `$expand`. [learn.microsoft](https://learn.microsoft.com/en-us/odata/concepts/queryoptions-overview)

### `$filter` — filter rows

Get delayed flights from Chennai:

```http
GET /odata/v4/flight/Flights
  ?$filter=Origin eq 'MAA' and Status eq 'DELAYED'
```

Supported operators include:

| Operator | Meaning | Example |
|---|---|---|
| `eq` | Equals | `Status eq 'DELAYED'` |
| `ne` | Not equal | `Status ne 'CANCELLED'` |
| `gt` | Greater than | `DelayMinutes gt 30` |
| `ge` | Greater than or equal | `DelayMinutes ge 30` |
| `lt` | Less than | `DelayMinutes lt 60` |
| `le` | Less than or equal | `DelayMinutes le 60` |
| `and` | Logical AND | `Origin eq 'MAA' and Status eq 'DELAYED'` |
| `or` | Logical OR | `Origin eq 'MAA' or Origin eq 'BLR'` |
| `contains()` | Text contains | `contains(FlightNo,'AI')` |
| `startswith()` | Starts with | `startswith(AirportCode,'M')` |

### `$select` — choose fields

```http
GET /odata/v4/flight/Flights
  ?$select=FlightNo,Origin,Destination,Status
```

This avoids returning unnecessary fields such as aircraft configuration or internal cost data.

### `$orderby` — sort

```http
GET /odata/v4/flight/Flights
  ?$filter=Status eq 'DELAYED'
  &$orderby=DelayMinutes desc
```

### `$top` and `$skip` — pagination

Get the first 20 delayed flights:

```http
GET /odata/v4/flight/Flights
  ?$filter=Status eq 'DELAYED'
  &$orderby=DelayMinutes desc
  &$top=20
```

Get the next 20:

```http
GET /odata/v4/flight/Flights
  ?$filter=Status eq 'DELAYED'
  &$orderby=DelayMinutes desc
  &$top=20&$skip=20
```

### `$count` — count records

```http
GET /odata/v4/flight/Flights
  ?$filter=Origin eq 'MAA'
  &$count=true
```

### `$expand` — include related data

Get flight AI202 along with its bookings:

```http
GET /odata/v4/flight/Flights('AI202')
  ?$expand=Bookings
```

Get flight, bookings, and each booking’s passenger:

```http
GET /odata/v4/flight/Flights('AI202')
  ?$expand=Bookings($expand=Passenger)
```

`$expand` returns related resources inline with the requested resource. [learn.microsoft](https://learn.microsoft.com/en-us/odata/concepts/queryoptions-overview)

## JSON examples

### Read one flight

Request:

```http
GET /odata/v4/flight/Flights('AI202')
```

Response:

```json
{
  "@odata.context": "$metadata#Flights/$entity",
  "FlightNo": "AI202",
  "FlightDate": "2026-10-18",
  "Origin": "MAA",
  "Destination": "DEL",
  "Status": "DELAYED",
  "DelayMinutes": 35,
  "Gate": "B7"
}
```

### Read a list

Request:

```http
GET /odata/v4/flight/Flights
  ?$filter=Origin eq 'MAA' and Status eq 'DELAYED'
  &$select=FlightNo,Destination,DelayMinutes
```

Response:

```json
{
  "@odata.context": "$metadata#Flights(FlightNo,Destination,DelayMinutes)",
  "value": [
    {
      "FlightNo": "AI202",
      "Destination": "DEL",
      "DelayMinutes": 35
    },
    {
      "FlightNo": "AI674",
      "Destination": "HYD",
      "DelayMinutes": 15
    }
  ]
}
```

### Create a baggage-fee record

Request:

```http
POST /odata/v4/flight/BaggageFees
Content-Type: application/json
```

Body:

```json
{
  "bookingId": "BK98765",
  "flightNo": "AI202",
  "passengerId": "PAX45678",
  "extraWeightKg": 4,
  "feeAmount": 2000,
  "currency": "INR"
}
```

Response:

```json
{
  "baggageFeeId": "BF98765",
  "bookingId": "BK98765",
  "status": "POSTED",
  "sapDocumentNumber": "5100012345"
}
```

### Update only the gate

```http
PATCH /odata/v4/flight/Flights('AI202')
Content-Type: application/json
If-Match: W/"2026-10-18T15:00:00Z"
```

Body:

```json
{
  "Gate": "B7"
}
```

## OData V2 vs V4

| Feature | OData V2 | OData V4 |
|---|---|---|
| SAP prevalence | Very common in legacy SAP Gateway services | Preferred for new SAP S/4HANA, RAP, and Fiori development |
| Data format | Often XML-oriented, though JSON is supported | JSON-first, leaner payloads |
| Query capabilities | Standard filtering, paging, expand | More expressive query language, including `$apply` for aggregation |
| Metadata | CSDL/XML-based | Richer metadata and improved JSON protocol |
| Batch support | Supported | Supported with modern batch conventions |
| Typical SAP use | Existing SAP Gateway services | New RAP services and modern Fiori apps |

SAP Gateway has supported OData V2 since AS ABAP 7.00 and OData V4 since AS ABAP 7.50; OData V3 was skipped by SAP Gateway.  For new development, prefer **OData V4**, especially with **RAP** and modern Fiori apps. [learning.sap](https://learning.sap.com/courses/introducing-sap-abap-platform-fundamentals/discussing-the-sap-gateway-and-odata-services)

## SAP ways to expose OData

| Technology | Typical use | Airline example |
|---|---|---|
| **SAP Gateway / SEGW** | Classic ABAP-based OData V2 service | Expose flight-delay data to a legacy mobile app |
| **RAP** | Modern ABAP OData V4 service | Build a Fiori “Manage Flight Bookings” app |
| **CAP** | Cloud application services | Build a flight-disruption service on SAP BTP |
| **SAP API Management** | Publish, secure, and monitor APIs | Expose airline APIs to travel partners |
| **SAP Integration Suite** | Integration and mapping | Connect booking system, SAP, and airport platform |
| **S/4HANA Cloud APIs** | Standard SAP business APIs | Read purchase orders or post invoices |

SAP Gateway acts as a bridge between SAP systems and external applications, making SAP data and processes available to different devices and platforms. [learning.sap](https://learning.sap.com/courses/introducing-sap-abap-platform-fundamentals/discussing-the-sap-gateway-and-odata-services)

## RAP and OData V4

In modern SAP, **RAP** is the main way to build OData V4 services on ABAP.

Typical flow:

```text
Database table
   → CDS view / data model
   → RAP behavior definition
   → RAP behavior implementation
   → Service definition
   → Service binding
   → OData V4 API
   → Fiori Elements / SAPUI5 / external client
```

Airline example:

| Layer | Airline artifact |
|---|---|
| Table | `ZFLIGHT_BOOKING` |
| CDS view | `ZC_FlightBooking` |
| Behavior | `create`, `update`, `delete`, `checkInPassenger` |
| Service definition | `ZSD_FlightBooking` |
| Service binding | OData V4 UI / Web API |
| Fiori app | “Manage Flight Bookings” |

The ABAP backend remains responsible for authorization, validations, determinations, and business rules. OData is the standardized contract through which the UI or external service communicates with that backend.

## OData actions and functions

Besides CRUD, OData supports custom operations.

### Function

A **function** must not change data. It returns information.

```http
GET /odata/v4/flight/GetAvailableSeats(FlightNo='AI202',FlightDate=2026-10-18)
```

Response:

```json
{
  "availableSeats": 12
}
```

### Action

An **action** can change data.

```http
POST /odata/v4/flight/Flights('AI202')/CancelFlight
```

Body:

```json
{
  "reason": "Severe weather at destination"
}
```

Airline action examples:

- `CancelFlight`
- `DelayFlight`
- `AssignGate`
- `CheckInPassenger`
- `IssueBoardingPass`
- `CancelBooking`
- `ReleaseAirportSlot`

## Concurrency with ETag

Two airport staff members might try to change the same flight’s gate at the same time. An **ETag** helps prevent one user from silently overwriting another user’s change.

1. Client reads flight AI202 and receives an ETag.
2. Client sends `PATCH` with `If-Match: <etag>`.
3. If another user changed the flight first, SAP returns HTTP `412 Precondition Failed`.
4. The client reloads the latest data and retries.

This is called **optimistic concurrency control**.

## Error handling

OData uses standard HTTP status codes.

| HTTP status | Meaning | Airline example |
|---|---|---|
| `200 OK` | Success | Flights retrieved |
| `201 Created` | Resource created | Baggage-fee record created |
| `204 No Content` | Update/delete succeeded with no response body | Gate updated |
| `400 Bad Request` | Invalid request | Missing booking ID |
| `401 Unauthorized` | Not authenticated | Missing or expired token |
| `403 Forbidden` | Not authorized | Gate agent cannot approve refunds |
| `404 Not Found` | Resource not found | Flight AI999 does not exist |
| `409 Conflict` | Business conflict | Seat already booked |
| `412 Precondition Failed` | ETag mismatch | Another user changed the gate |
| `500 Internal Server Error` | Server failure | SAP backend error |

Example OData V4 error:

```json
{
  "error": {
    "code": "SEAT_UNAVAILABLE",
    "message": "No seats are available on flight AI202 for the requested class."
  }
}
```

## Security

An OData service must protect business data.

- Use HTTPS.
- Authenticate users or systems using OAuth 2.0, SAML, or SAP-issued tokens.
- Authorize at the backend: a gate agent should not access payroll data.
- Apply **CSRF** protection for state-changing requests from browser clients.
- Validate all input on the server.
- Use field-level authorization where needed—for example, hide passenger passport data from ground staff.
- Log access and changes for audit purposes.
- Rate-limit public or partner-facing APIs.

## Performance practices

- Use `$select` to return only required fields.
- Use `$filter` so SAP returns only relevant records.
- Use `$top` and `$skip` for pagination.
- Avoid `$expand` on large collections unless necessary.
- Use server-driven paging for large entity sets.
- Cache stable reference data, such as airport master data.
- Push calculations to the database or backend, not the client.
- Monitor expensive OData requests and SQL statements.

Efficient request:

```http
GET /odata/v4/flight/Flights
  ?$filter=Origin eq 'MAA' and Status eq 'DELAYED'
  &$select=FlightNo,Destination,DelayMinutes,Gate
  &$orderby=DelayMinutes desc
  &$top=50
```

This returns only delayed Chennai-origin flights, only four fields, sorted by delay, limited to 50 records.

## Consuming OData from Spring Boot

A Java/Spring Boot service can consume an SAP OData API like any HTTP JSON API, while benefiting from OData’s standard query syntax.

```java
@Component
public class FlightODataClient {

    private final RestClient restClient;

    public FlightODataClient(RestClient.Builder builder) {
        this.restClient = builder
            .baseUrl("https://sap-airline.example.com/odata/v4/flight")
            .build();
    }

    public List<FlightDto> getDelayedFlights(String originAirport) {
        String uri = UriComponentsBuilder
            .fromPath("/Flights")
            .queryParam("$filter",
                "Origin eq '" + originAirport + "' and Status eq 'DELAYED'")
            .queryParam("$select", "FlightNo,Origin,Destination,DelayMinutes")
            .queryParam("$orderby", "DelayMinutes desc")
            .queryParam("$top", 50)
            .toUriString();

        ODataResponse<FlightDto> response = restClient.get()
            .uri(uri)
            .retrieve()
            .body(new ParameterizedTypeReference<>() {});

        return response.value();
    }
}
```

For production, use a proper OData client or SDK where possible, URL-encode query values, and avoid string-concatenating user input directly into filters.

## OData vs plain REST

| Aspect | Plain REST | OData |
|---|---|---|
| Query convention | API-specific | Standard `$filter`, `$select`, `$expand` |
| Metadata | Often separate documentation | Standard `$metadata` document |
| Relationships | Custom endpoints or embedded objects | Navigation properties and `$expand` |
| Paging | Custom `page`/`size` | Standard `$top`/`$skip` or server-driven paging |
| CRUD semantics | Usually REST conventions | Standardized HTTP and payload conventions |
| Best for | Simple custom APIs | Enterprise data services, Fiori, SAP integrations |
| Trade-off | More design freedom | More standardization and slightly more complex URLs |

Use plain REST when you need a very simple, custom API. Use OData when multiple clients need consistent, queryable access to structured SAP business data.

## Learning path

1. **HTTP and REST basics** — methods, status codes, headers, JSON.
2. **OData fundamentals** — entity, entity set, key, navigation property.
3. **Metadata** — read `$metadata`, understand entity types and associations.
4. **Query options** — `$filter`, `$select`, `$orderby`, `$top`, `$skip`, `$count`.
5. **Relationships** — `$expand`, nested expands, navigation restrictions.
6. **CRUD** — GET, POST, PATCH, PUT, DELETE.
7. **Actions and functions** — custom business operations.
8. **Concurrency** — ETags and `If-Match`.
9. **SAP exposure options** — SAP Gateway, RAP, CAP, API Management.
10. **Security and performance** — OAuth, CSRF, authorization, pagination, caching.
11. **Client development** — SAPUI5, Java/Spring Boot, Postman, OData clients.

## Mini end-to-end example

**Scenario:** A gate agent needs delayed flights departing from Chennai.

### 1. Fiori app calls OData

```http
GET /odata/v4/flight/Flights
  ?$filter=Origin eq 'MAA' and Status eq 'DELAYED'
  &$select=FlightNo,Destination,DelayMinutes,Gate
  &$orderby=DelayMinutes desc
```

### 2. SAP backend translates to SQL-like logic

```sql
SELECT flight_no, destination_airport, delay_minutes, gate
FROM flight
WHERE origin_airport = 'MAA'
  AND status = 'DELAYED'
ORDER BY delay_minutes DESC;
```

### 3. OData returns JSON

```json
{
  "value": [
    {
      "FlightNo": "AI202",
      "Destination": "DEL",
      "DelayMinutes": 35,
      "Gate": "B7"
    }
  ]
}
```

### 4. Agent changes the gate

```http
PATCH /odata/v4/flight/Flights('AI202')
Content-Type: application/json
If-Match: W/"flight-ai202-v7"
```

```json
{
  "Gate": "C3"
}
```

### 5. SAP validates and updates

The ABAP/RAP backend checks whether the agent is authorized, whether gate C3 is available, and whether the flight is still editable. If valid, it updates the record and returns HTTP `204 No Content`.

## What an OData developer does

An SAP OData developer:

- Designs entity models for business objects such as flights, bookings, and passengers.
- Exposes CDS/RAP or SAP Gateway services as OData APIs.
- Defines keys, associations, navigation properties, and access controls.
- Implements CRUD operations and custom actions/functions.
- Optimizes queries using OData system query options.
- Secures services with authentication, authorization, and CSRF protection.
- Supports Fiori apps, external Java services, mobile apps, and partner integrations.

In short: **OData is SAP’s standardized API language for business data.** It gives Fiori and external systems a consistent way to query, navigate, create, update, and delete SAP data while leaving business rules and authorization in the SAP backend. [docs.oasis-open](https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part1-protocol.html)
