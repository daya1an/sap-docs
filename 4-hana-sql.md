**SAP HANA SQLScript** is SAP HANA’s extended SQL language for writing database-level procedures, functions, and data-intensive logic. In airline terms, it lets the HANA database itself calculate things such as delayed-flight statistics, excess-baggage revenue, airport-slot utilization, or cargo reconciliation—close to the data, instead of pulling millions of rows into Java or ABAP. [help.sap](https://help.sap.com/docs/hana-cloud-database/sap-hana-cloud-sap-hana-database-performance-guide-for-developers/sqlscript-performance)

## What SQLScript means

- **SQL** — **Structured Query Language**: standard language for reading and changing relational data.
- **SAP HANA** — SAP’s high-performance, in-memory database platform.
- **SQLScript** — SAP HANA’s SQL extension language for embedding application logic inside the database.
- **Procedure** — a stored, reusable database program that can accept inputs, perform multiple steps, and return outputs.
- **Function** — a reusable calculation; a **scalar function** returns one value, while a **table function** returns a result set.
- **UDF** — **User-Defined Function**: a custom function created by a developer.
- **DDL** — **Data Definition Language**: creates or changes database objects such as tables and views.
- **DML** — **Data Manipulation Language**: inserts, updates, deletes, or selects data.

SQLScript extends SQL with table types, side-effect-free functions, and procedural constructs such as variables, branches, loops, and exception handlers. [help.sap](https://help.sap.com/doc/6254b3bb439c4f409a979dc407b49c9b/2.0.08/en-US/SAP_HANA_SQL_Script_Reference_en.pdf)

## Airline analogy

Imagine an airline wants a daily report:

- Total delayed flights by origin airport.
- Average delay in minutes.
- Total excess-baggage revenue.
- Top five delayed routes.
- Flights delayed more than 30 minutes.

A normal application might:

1. Read all flights from HANA into Java.
2. Loop through them.
3. Calculate totals in Java.
4. Return the report.

With SQLScript, HANA does the heavy processing internally:

```text
Flight + Booking + Baggage tables
          |
   SQLScript procedure
          |
Daily Airline Operations Report
```

This is usually faster for large datasets because SQLScript is designed to embed data-intensive logic in the database and exploit HANA’s in-memory processing and parallel execution. [help.sap](https://help.sap.com/docs/hana-cloud-database/sap-hana-cloud-sap-hana-database-performance-guide-for-developers/sqlscript-performance)

## SQL vs SQLScript

| Aspect | Standard SQL | SAP HANA SQLScript |
|---|---|---|
| Main purpose | Query and modify data | Data-intensive application logic |
| Variables | Limited, usually query-scoped | Declared local scalar and table variables |
| Control flow | Mostly not procedural | `IF`, `CASE`, `FOR`, `WHILE`, exception handling |
| Reusable logic | Views, queries | Procedures, scalar UDFs, table UDFs |
| Multi-step processing | Often multiple separate statements | One database-side program |
| Best use | Straightforward reads/writes | Complex calculations, orchestration, batch processing |

SQLScript supports both **declarative logic**—SQL statements that describe *what* data you want—and **imperative logic**—procedural steps such as loops and conditionals that describe *how* to process it. SAP recommends favoring declarative SQL because it gives HANA more optimization opportunities. [help.sap](https://help.sap.com/docs/hana-cloud-database/sap-hana-cloud-sap-hana-database-performance-guide-for-developers/sqlscript-performance)

## Basic SQLScript syntax

SQLScript code usually lives inside a procedure or function.

```sql
CREATE PROCEDURE GET_DELAYED_FLIGHTS(
    IN IV_ORIGIN_AIRPORT NVARCHAR(3),
    OUT ET_DELAYED_FLIGHTS TABLE (
        FLIGHT_NO NVARCHAR(10),
        DESTINATION_AIRPORT NVARCHAR(3),
        DELAY_MINUTES INTEGER,
        GATE NVARCHAR(5)
    )
)
LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
AS
BEGIN
    ET_DELAYED_FLIGHTS =
        SELECT
            FLIGHT_NO,
            DESTINATION_AIRPORT,
            DELAY_MINUTES,
            GATE
        FROM FLIGHT
        WHERE ORIGIN_AIRPORT = :IV_ORIGIN_AIRPORT
          AND STATUS = 'DELAYED'
        ORDER BY DELAY_MINUTES DESC;
END;
```

Call it:

```sql
CALL GET_DELAYED_FLIGHTS('MAA', ?);
```

This returns all delayed flights departing from Chennai Airport (**MAA**).

## Variables

Use `DECLARE` to define local variables.

```sql
DECLARE LV_TOTAL_DELAY INTEGER;
DECLARE LV_AVERAGE_DELAY DECIMAL(10,2);
DECLARE LV_AIRPORT_CODE NVARCHAR(3);
DECLARE LT_FLIGHTS TABLE (
    FLIGHT_NO NVARCHAR(10),
    DELAY_MINUTES INTEGER
);
```

Assign values using `:` before a variable:

```sql
LV_AIRPORT_CODE = 'MAA';

SELECT SUM(DELAY_MINUTES)
INTO LV_TOTAL_DELAY
FROM FLIGHT_DELAY
WHERE ORIGIN_AIRPORT = :LV_AIRPORT_CODE;
```

## Conditional logic

```sql
IF :LV_TOTAL_DELAY > 500 THEN
    LV_ALERT_LEVEL = 'HIGH';
ELSEIF :LV_TOTAL_DELAY > 200 THEN
    LV_ALERT_LEVEL = 'MEDIUM';
ELSE
    LV_ALERT_LEVEL = 'LOW';
END IF;
```

Airline example:

- More than 500 total delay minutes at an airport: high disruption.
- Between 201 and 500: medium disruption.
- 200 or fewer: low disruption.

## Loops

SQLScript supports `FOR` and `WHILE` loops, but use them carefully. Loops are imperative logic and can prevent HANA from optimizing the work as a single data flow. [help.sap](https://help.sap.com/docs/hana-cloud-database/sap-hana-cloud-sap-hana-database-performance-guide-for-developers/sqlscript-performance)

### FOR loop

```sql
DECLARE LV_INDEX INTEGER;

FOR LV_INDEX IN 1..5 DO
    INSERT INTO AIRLINE_ALERT_LOG (
        ALERT_ID,
        MESSAGE
    )
    VALUES (
        :LV_INDEX,
        'Processing airport disruption batch ' || :LV_INDEX
    );
END FOR;
```

### WHILE loop

```sql
DECLARE LV_RETRY_COUNT INTEGER = 0;

WHILE :LV_RETRY_COUNT < 3 DO
    -- Retry logic, such as checking a failed integration batch
    LV_RETRY_COUNT = :LV_RETRY_COUNT + 1;
END WHILE;
```

### Prefer set-based SQL

Instead of looping through flights one by one:

```sql
-- Avoid this pattern for large datasets
FOR each_flight IN ... DO
    UPDATE FLIGHT
    SET DELAY_CATEGORY = 'HIGH'
    WHERE FLIGHT_NO = :each_flight.FLIGHT_NO;
END FOR;
```

Use one set-based statement:

```sql
UPDATE FLIGHT
SET DELAY_CATEGORY = CASE
    WHEN DELAY_MINUTES >= 60 THEN 'HIGH'
    WHEN DELAY_MINUTES >= 15 THEN 'MEDIUM'
    ELSE 'LOW'
END
WHERE FLIGHT_DATE = CURRENT_DATE;
```

The second version lets HANA process all rows together and is generally preferable.

## Procedures

A **procedure** is a reusable sequence of data transformations. SAP describes procedures as a way to describe transformations on input data and database tables. [help.sap](https://help.sap.com/docs/hana-cloud-database/sap-hana-cloud-sap-hana-sqlscript-reference/d43d91578c3b42b3bacfd89aacf0d62f.html)

### Example: calculate airport disruption summary

```sql
CREATE OR REPLACE PROCEDURE CALCULATE_AIRPORT_DISRUPTION(
    IN IV_REPORT_DATE DATE,
    OUT ET_AIRPORT_SUMMARY TABLE (
        ORIGIN_AIRPORT NVARCHAR(3),
        TOTAL_FLIGHTS INTEGER,
        DELAYED_FLIGHTS INTEGER,
        TOTAL_DELAY_MINUTES INTEGER,
        AVERAGE_DELAY_MINUTES DECIMAL(10,2),
        DISRUPTION_LEVEL NVARCHAR(10)
    )
)
LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
AS
BEGIN
    ET_AIRPORT_SUMMARY =
        SELECT
            ORIGIN_AIRPORT,
            COUNT(*) AS TOTAL_FLIGHTS,
            SUM(CASE WHEN STATUS = 'DELAYED' THEN 1 ELSE 0 END)
                AS DELAYED_FLIGHTS,
            SUM(DELAY_MINUTES) AS TOTAL_DELAY_MINUTES,
            AVG(DELAY_MINUTES) AS AVERAGE_DELAY_MINUTES,
            CASE
                WHEN SUM(DELAY_MINUTES) > 500 THEN 'HIGH'
                WHEN SUM(DELAY_MINUTES) > 200 THEN 'MEDIUM'
                ELSE 'LOW'
            END AS DISRUPTION_LEVEL
        FROM FLIGHT
        WHERE FLIGHT_DATE = :IV_REPORT_DATE
        GROUP BY ORIGIN_AIRPORT;
END;
```

Call:

```sql
CALL CALCULATE_AIRPORT_DISRUPTION('2026-10-18', ?);
```

## Input and output parameters

| Parameter type | Meaning | Airline example |
|---|---|---|
| `IN` | Value passed into procedure | Airport code, report date |
| `OUT` | Value returned by procedure | Summary table, calculated fee |
| `INOUT` | Input that can also be modified and returned | Running balance or status |

```sql
CREATE PROCEDURE CALCULATE_BAGGAGE_FEE(
    IN IV_WEIGHT_KG DECIMAL(10,2),
    IN IV_FREE_ALLOWANCE_KG DECIMAL(10,2),
    IN IV_FEE_PER_KG DECIMAL(10,2),
    OUT EV_EXTRA_FEE DECIMAL(10,2)
)
LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
AS
BEGIN
    IF :IV_WEIGHT_KG > :IV_FREE_ALLOWANCE_KG THEN
        EV_EXTRA_FEE =
            (:IV_WEIGHT_KG - :IV_FREE_ALLOWANCE_KG)
            * :IV_FEE_PER_KG;
    ELSE
        EV_EXTRA_FEE = 0;
    END IF;
END;
```

Call:

```sql
CALL CALCULATE_BAGGAGE_FEE(27, 23, 500, ?);
```

Result:

```text
2000.00
```

## Scalar functions

A **scalar UDF** returns a single value.

```sql
CREATE OR REPLACE FUNCTION GET_FLIGHT_STATUS_LABEL(
    IV_STATUS NVARCHAR(20)
)
RETURNS RV_STATUS_LABEL NVARCHAR(30)
LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
AS
BEGIN
    RV_STATUS_LABEL = CASE :IV_STATUS
        WHEN 'ON_TIME'  THEN 'On Time'
        WHEN 'DELAYED'  THEN 'Delayed'
        WHEN 'CANCELLED' THEN 'Cancelled'
        ELSE 'Unknown'
    END;
END;
```

Use it in SQL:

```sql
SELECT
    FLIGHT_NO,
    GET_FLIGHT_STATUS_LABEL(STATUS) AS STATUS_LABEL
FROM FLIGHT
WHERE FLIGHT_DATE = CURRENT_DATE;
```

## Table functions

A **table function** returns a table and can be used directly inside a `SELECT`, like a view or subquery. SAP notes that table functions are useful when you need direct filtering, sorting, or grouping on their output. [developers.sap](https://developers.sap.com/tutorials/hana-cloud-sqlscript-table-user/)

```sql
CREATE OR REPLACE FUNCTION GET_DELAYED_FLIGHTS_BY_AIRPORT(
    IV_ORIGIN_AIRPORT NVARCHAR(3)
)
RETURNS TABLE (
    FLIGHT_NO NVARCHAR(10),
    DESTINATION_AIRPORT NVARCHAR(3),
    DELAY_MINUTES INTEGER,
    GATE NVARCHAR(5)
)
LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
AS
BEGIN
    RETURN
        SELECT
            FLIGHT_NO,
            DESTINATION_AIRPORT,
            DELAY_MINUTES,
            GATE
        FROM FLIGHT
        WHERE ORIGIN_AIRPORT = :IV_ORIGIN_AIRPORT
          AND STATUS = 'DELAYED';
END;
```

Use it like a table:

```sql
SELECT *
FROM GET_DELAYED_FLIGHTS_BY_AIRPORT('MAA')
ORDER BY DELAY_MINUTES DESC;
```

You can also filter its output:

```sql
SELECT *
FROM GET_DELAYED_FLIGHTS_BY_AIRPORT('MAA')
WHERE DELAY_MINUTES > 30;
```

## Table types

A **table type** defines the structure of a table variable or output parameter without creating a physical database table.

```sql
CREATE TYPE TT_FLIGHT_SUMMARY AS TABLE (
    ORIGIN_AIRPORT NVARCHAR(3),
    TOTAL_FLIGHTS INTEGER,
    DELAYED_FLIGHTS INTEGER,
    TOTAL_DELAY_MINUTES INTEGER
);
```

Use it in a procedure:

```sql
CREATE PROCEDURE GET_FLIGHT_SUMMARY(
    IN IV_REPORT_DATE DATE,
    OUT ET_SUMMARY TT_FLIGHT_SUMMARY
)
LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
AS
BEGIN
    ET_SUMMARY =
        SELECT
            ORIGIN_AIRPORT,
            COUNT(*) AS TOTAL_FLIGHTS,
            SUM(CASE WHEN STATUS = 'DELAYED' THEN 1 ELSE 0 END)
                AS DELAYED_FLIGHTS,
            SUM(DELAY_MINUTES) AS TOTAL_DELAY_MINUTES
        FROM FLIGHT
        WHERE FLIGHT_DATE = :IV_REPORT_DATE
        GROUP BY ORIGIN_AIRPORT;
END;
```

## Exception handling

Use exception handling to prevent a failure from leaving incomplete results.

```sql
DECLARE EXIT HANDLER FOR SQLEXCEPTION
BEGIN
    INSERT INTO SQLSCRIPT_ERROR_LOG (
        ERROR_TIME,
        PROCEDURE_NAME,
        ERROR_MESSAGE
    )
    VALUES (
        CURRENT_TIMESTAMP,
        'CALCULATE_AIRPORT_DISRUPTION',
        ::SQL_ERROR_MESSAGE
    );
END;
```

Airline example: if the daily disruption calculation fails, record the error for the operations-support team instead of silently failing.

## Dynamic SQL

Dynamic SQL builds SQL text at runtime. It is powerful but should be used sparingly because it can reduce plan reuse and create SQL-injection risks.

```sql
EXECUTE IMMEDIATE
    'SELECT COUNT(*) FROM FLIGHT WHERE ORIGIN_AIRPORT = '''
    || :IV_AIRPORT || '''';
```

Prefer parameterized or static SQL:

```sql
SELECT COUNT(*)
INTO LV_FLIGHT_COUNT
FROM FLIGHT
WHERE ORIGIN_AIRPORT = :IV_AIRPORT;
```

## SAP HANA-specific SQL features

SAP HANA SQL/SQLScript includes advanced features useful for airline analytics.

### Window functions

Rank delayed flights within each airport:

```sql
SELECT
    ORIGIN_AIRPORT,
    FLIGHT_NO,
    DELAY_MINUTES,
    RANK() OVER (
        PARTITION BY ORIGIN_AIRPORT
        ORDER BY DELAY_MINUTES DESC
    ) AS DELAY_RANK
FROM FLIGHT
WHERE FLIGHT_DATE = CURRENT_DATE
  AND STATUS = 'DELAYED';
```

### Common Table Expressions: CTE

**CTE** means **Common Table Expression**—a named temporary result set.

```sql
WITH DELAYED_FLIGHTS AS (
    SELECT
        ORIGIN_AIRPORT,
        FLIGHT_NO,
        DELAY_MINUTES
    FROM FLIGHT
    WHERE FLIGHT_DATE = CURRENT_DATE
      AND STATUS = 'DELAYED'
)
SELECT
    ORIGIN_AIRPORT,
    COUNT(*) AS DELAYED_COUNT,
    AVG(DELAY_MINUTES) AS AVG_DELAY
FROM DELAYED_FLIGHTS
GROUP BY ORIGIN_AIRPORT;
```

### Hierarchies

Airports and airline route networks can have hierarchical relationships—for example, hub airports, regional airports, and feeder routes.

```sql
SELECT
    AIRPORT_CODE,
    PARENT_AIRPORT_CODE,
    HIERARCHY_LEVEL
FROM AIRPORT_HIERARCHY;
```

### Full-text search

Search incident notes or maintenance remarks:

```sql
SELECT *
FROM MAINTENANCE_INCIDENT
WHERE CONTAINS(INCIDENT_NOTES, 'engine OR hydraulic', FUZZY(0.8));
```

## SQLScript in SAP HANA models

SQLScript is often used with:

| Artifact | Purpose | Airline example |
|---|---|---|
| **Calculation view** | Graphical analytical model | Flight-delay dashboard |
| **Table function** | Script-based reusable data source | Complex delayed-flight calculation |
| **Stored procedure** | Multi-step batch processing | Nightly airport disruption load |
| **Scalar UDF** | Reusable single-value calculation | Baggage-fee calculation |
| **HANA Cloud HDI container** | Deployment container for database objects | Airline reporting schema |
| **DWC / SAP Datasphere** | Data warehousing and modeling | Enterprise airline analytics |

For calculation views, SAP recommends using SQL expressions rather than Column Engine language where possible, and avoiding unnecessary columns or calculated columns in filters, because these choices affect query optimization. [learning.sap](https://learning.sap.com/courses/developing-data-models-with-sap-hana-cloud/implement-best-practices-in-calculation-view-nodes_ffd2bc0b-895f-479f-93ab-4f39152dfbe4)

## Performance best practices

| Practice | Why it matters | Airline example |
|---|---|---|
| Prefer declarative SQL | Enables optimizer parallelization | Calculate all airport summaries in one query |
| Avoid row-by-row loops | Loops reduce data-flow optimization | Do not update each delayed flight separately |
| Filter early | Reduces data volume | Filter by `FLIGHT_DATE` before joining bookings |
| Select only needed columns | Reduces memory and transfer | Select flight number, delay, and gate—not every column |
| Use appropriate indexes | Speeds frequent lookups | Index `ORIGIN_AIRPORT`, `FLIGHT_DATE`, `STATUS` |
| Avoid dynamic SQL | Improves plan reuse | Use parameters instead of concatenated SQL |
| Use table functions for reusable logic | Allows direct filtering and composition | Reusable delayed-flight data source |
| Monitor with HANA tools | Finds expensive statements | Identify slow cargo-reconciliation procedures |

SAP’s SQLScript performance guidance specifically highlights imperative constructs, optimization opportunities, and recommended patterns/anti-patterns for stored procedures. [help.sap](https://help.sap.com/docs/hana-cloud-database/sap-hana-cloud-sap-hana-database-performance-guide-for-developers/sqlscript-performance)

## Security practices

- Use `SQL SECURITY INVOKER` or `SQL SECURITY DEFINER` deliberately.
- Grant only the minimum required table and procedure privileges.
- Avoid dynamic SQL with unvalidated user input.
- Validate airport codes, dates, and numeric values before use.
- Do not expose sensitive passenger data in analytical outputs.
- Log errors without storing passwords, tokens, or full personal data.
- Use roles such as `AirlineOpsReader`, `FinanceUser`, and `CargoIntegrationUser`.

## Testing SQLScript

Test procedures with representative airline data:

| Test case | Input | Expected result |
|---|---|---|
| Normal baggage | 20 kg | Fee = 0 |
| Extra baggage | 27 kg | Fee = 2,000 |
| No delayed flights | Empty flight data | Empty summary |
| Multiple airports | MAA, DEL, BLR | One summary row per airport |
| Null delay | `DELAY_MINUTES IS NULL` | Treated correctly, not as zero unless business rule says so |
| Invalid airport | `XX` | Clear error or empty result |

Example test query:

```sql
CALL CALCULATE_BAGGAGE_FEE(27, 23, 500, ?);
```

Expected output:

```text
2000.00
```

## Where SQLScript fits with your stack

| Technology | Role | Airline example |
|---|---|---|
| **SAP HANA SQLScript** | Database-side calculations and procedures | Airport disruption summary |
| **ABAP** | SAP business logic and ERP customization | Validate booking and create billing document |
| **OData/RAP** | Expose SAP data and operations as APIs | Fiori flight-disruption app |
| **Java/Spring Boot** | Integration orchestration and external APIs | Connect booking, cargo, airport, and SAP systems |
| **SQL** | Query and manage relational data | Integration audit and reconciliation |
| **SAPUI5/Fiori** | User interface | Airport operations dashboard |

A good division of responsibility is:

```text
SAPUI5/Fiori  → user interaction
Spring Boot   → integration and orchestration
ABAP/RAP      → SAP business rules
SQLScript     → heavy data calculations inside HANA
```

## Learning path

1. **Advanced SQL** — joins, aggregation, subqueries, CTEs, window functions.
2. **HANA data types** — `NVARCHAR`, `DECIMAL`, `DATE`, `TIMESTAMP`, `INTEGER`.
3. **SQLScript basics** — procedures, variables, `IF`, loops, output parameters.
4. **Table types and table variables** — pass structured result sets.
5. **Scalar UDFs** — reusable single-value calculations.
6. **Table UDFs** — reusable, queryable data sets.
7. **Exception handling** — controlled failure and logging.
8. **Performance tuning** — set-based logic, filtering, indexes, HANA tools.
9. **Calculation views and SAP Datasphere** — analytical modeling.
10. **Security and deployment** — privileges, HDI containers, transport, version control.

## Mini end-to-end example

**Scenario:** Build a nightly procedure that calculates excess-baggage revenue by origin airport.

### 1. Source tables

```text
FLIGHT
├── FLIGHT_NO
├── FLIGHT_DATE
├── ORIGIN_AIRPORT
└── DESTINATION_AIRPORT

BOOKING
├── BOOKING_ID
├── FLIGHT_NO
├── PASSENGER_ID
└── EXTRA_BAGGAGE_FEE
```

### 2. SQLScript procedure

```sql
CREATE OR REPLACE PROCEDURE CALCULATE_BAGGAGE_REVENUE(
    IN IV_REPORT_DATE DATE,
    OUT ET_BAGGAGE_REVENUE TABLE (
        ORIGIN_AIRPORT NVARCHAR(3),
        TOTAL_BOOKINGS INTEGER,
        EXTRA_BAGGAGE_BOOKINGS INTEGER,
        TOTAL_REVENUE DECIMAL(15,2)
    )
)
LANGUAGE SQLSCRIPT
SQL SECURITY INVOKER
AS
BEGIN
    ET_BAGGAGE_REVENUE =
        SELECT
            F.ORIGIN_AIRPORT,
            COUNT(B.BOOKING_ID) AS TOTAL_BOOKINGS,
            SUM(
                CASE
                    WHEN B.EXTRA_BAGGAGE_FEE > 0 THEN 1
                    ELSE 0
                END
            ) AS EXTRA_BAGGAGE_BOOKINGS,
            SUM(B.EXTRA_BAGGAGE_FEE) AS TOTAL_REVENUE
        FROM FLIGHT F
        INNER JOIN BOOKING B
            ON B.FLIGHT_NO = F.FLIGHT_NO
           AND B.FLIGHT_DATE = F.FLIGHT_DATE
        WHERE F.FLIGHT_DATE = :IV_REPORT_DATE
        GROUP BY F.ORIGIN_AIRPORT;
END;
```

### 3. Execute

```sql
CALL CALCULATE_BAGGAGE_REVENUE('2026-10-18', ?);
```

### 4. Sample output

| ORIGIN_AIRPORT | TOTAL_BOOKINGS | EXTRA_BAGGAGE_BOOKINGS | TOTAL_REVENUE |
|---|---:|---:|---:|
| MAA | 180 | 24 | 48,000.00 |
| DEL | 210 | 31 | 65,500.00 |
| BLR | 150 | 18 | 36,000.00 |

## What a SQLScript developer does

A SAP HANA SQLScript developer:

- Designs high-performance procedures, table functions, and scalar functions.
- Writes complex analytical SQL for airline operations, finance, cargo, and maintenance data.
- Builds reusable database logic for reporting and data processing.
- Optimizes queries for HANA’s in-memory, column-store, and parallel-processing capabilities.
- Supports calculation views, SAP Datasphere models, and OData/RAP services.
- Debugs slow SQL and large-volume batch jobs.
- Ensures database logic is secure, testable, and maintainable.

In short: **SQLScript is SQL plus controlled procedural power inside SAP HANA.** Use it when standard SQL is not enough for complex, data-heavy airline calculations, but keep business workflow and SAP-specific rules in ABAP/RAP or the appropriate integration layer. [help.sap](https://help.sap.com/docs/hana-cloud-database/sap-hana-cloud-sap-hana-database-performance-guide-for-developers/sqlscript-performance)
