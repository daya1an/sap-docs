ABAP (**Advanced Business Application Programming**) is SAP’s proprietary programming language for building and customizing business software such as SAP ERP and SAP S/4HANA. Think of SAP as the airline’s central operating system; ABAP is the language you use to add airline-specific rules, screens, reports, and integrations—such as baggage-fee calculations, flight-delay alerts, or airport slot management. [help.sap](https://help.sap.com/doc/abapdocu_latest_index_htm/latest/en-US/ABENABAP_OVERVIEW.html)

## What SAP and ABAP mean

- **SAP** — a German enterprise-software company; its products help companies run finance, procurement, sales, HR, logistics, and production.
- **ERP** — **Enterprise Resource Planning**: one integrated system that connects business departments and shares a common set of data.
- **ABAP** — **Advanced Business Application Programming**: SAP’s main server-side language for customizing and extending SAP applications. [learning.sap-press](https://learning.sap-press.com/abap)
- **S/4HANA** — SAP’s modern ERP suite, typically running on the **SAP HANA** in-memory database.
- **AS ABAP** — **Application Server ABAP**: the SAP runtime environment where ABAP programs execute. It handles business logic, database access, user-interface communication, and security. [help.sap](https://help.sap.com/doc/abapdocu_latest_index_htm/latest/en-US/ABENABAP_OVERVIEW.html)

For an airline, SAP might manage:

- Purchasing aircraft parts.
- Paying airport and fuel vendors.
- Managing employee travel and payroll.
- Tracking cargo bookings.
- Handling customer orders, invoices, and refunds.

ABAP lets the airline customize those standard processes without rewriting the entire SAP product.

## How SAP is structured

SAP traditionally separates an application into three layers:

| Layer | Plain-English meaning | Airline example |
|---|---|---|
| Presentation layer | What users see and click | Airport check-in screen or Fiori web app |
| Application layer | Business rules and processing | “If a passenger’s baggage exceeds 23 kg, charge excess-baggage fee” |
| Database layer | Where business data is stored | Flight, passenger, booking, invoice, and airport tables |

ABAP mainly lives in the application layer. It reads and writes business data, applies rules, and sends results to screens, APIs, or other systems. SAP describes ABAP programs as processing and formatting database data and exchanging input/output with the presentation layer. [help.sap](https://help.sap.com/doc/abapdocu_latest_index_htm/latest/en-US/ABENABAP_OVERVIEW.html)

## Basic ABAP concepts

### Data types and variables

ABAP stores values in variables with declared types.

```abap
DATA lv_flight_no      TYPE string.
DATA lv_departure_time TYPE t.
DATA lv_ticket_price   TYPE p DECIMALS 2.
DATA lv_passenger_count TYPE i.

lv_flight_no        = 'AI202'.
lv_departure_time   = '063000'.   " 06:30:00
lv_ticket_price     = '12500.50'.
lv_passenger_count  = 180.
```

Naming conventions commonly used:

- `lv_` — local variable.
- `ls_` — local structure.
- `lt_` — local internal table.
- `iv_` — importing parameter.
- `ev_` — exporting parameter.
- `cv_` — changing parameter.

For an airline:

```abap
DATA lv_baggage_weight TYPE p DECIMALS 2.
DATA lv_extra_fee      TYPE p DECIMALS 2.

lv_baggage_weight = 27.

IF lv_baggage_weight > 23.
  lv_extra_fee = ( lv_baggage_weight - 23 ) * 500.
ENDIF.
```

This means: if baggage is above 23 kg, charge ₹500 for every extra kilogram.

### Structures

A **structure** groups related fields—like one row in a form.

```abap
TYPES: BEGIN OF ty_flight,
         flight_no       TYPE string,
         origin_airport  TYPE string,
         destination_airport TYPE string,
         departure_time  TYPE t,
         arrival_time    TYPE t,
       END OF ty_flight.

DATA ls_flight TYPE ty_flight.

ls_flight-flight_no            = 'AI202'.
ls_flight-origin_airport       = 'MAA'.   " Chennai
ls_flight-destination_airport  = 'DEL'.   " Delhi
ls_flight-departure_time       = '063000'.
ls_flight-arrival_time         = '091500'.
```

### Internal tables

An **internal table** is an in-memory list of rows, similar to a `List<Flight>` in Java.

```abap
DATA lt_flights TYPE STANDARD TABLE OF ty_flight.
DATA ls_flight  TYPE ty_flight.

ls_flight-flight_no           = 'AI202'.
ls_flight-origin_airport      = 'MAA'.
ls_flight-destination_airport = 'DEL'.
APPEND ls_flight TO lt_flights.

ls_flight-flight_no           = '6E123'.
ls_flight-origin_airport      = 'BLR'.
ls_flight-destination_airport = 'HYD'.
APPEND ls_flight TO lt_flights.
```

`APPEND` adds a row. ABAP internal tables are central to business processing because ERP operations often involve many records—passengers, bookings, invoices, cargo items, or flight legs. [help.sap](https://help.sap.com/doc/abapdocu_latest_index_htm/latest/en-US/ABENABAP_OVERVIEW.html)

### Control statements

```abap
IF lv_passenger_count > 180.
  WRITE 'Flight is overbooked'.
ELSEIF lv_passenger_count = 180.
  WRITE 'Flight is full'.
ELSE.
  WRITE 'Seats available'.
ENDIF.

CASE lv_flight_no.
  WHEN 'AI202'.
    WRITE 'Chennai to Delhi'.
  WHEN '6E123'.
    WRITE 'Bengaluru to Hyderabad'.
  WHEN OTHERS.
    WRITE 'Unknown flight'.
ENDCASE.

LOOP AT lt_flights INTO ls_flight.
  WRITE: / ls_flight-flight_no, ls_flight-origin_airport.
ENDLOOP.
```

## Database access in ABAP

ABAP has database access built into the language, which is one reason it is well suited to ERP applications. [help.sap](https://help.sap.com/doc/abapdocu_latest_index_htm/latest/en-US/ABENABAP_OVERVIEW.html)

### Database tables

In an airline system, you might have tables such as:

| Table | Purpose |
|---|---|
| `ZFLIGHT` | Custom flight master data |
| `ZBOOKING` | Passenger bookings |
| `ZPASSENGER` | Passenger details |
| `ZAIRPORT` | Airport master data |
| `ZBAGGAGE` | Checked baggage records |

The `Z` prefix is commonly used for customer-created objects, helping distinguish them from SAP standard objects.

### SELECT

```abap
SELECT flight_no,
       origin_airport,
       destination_airport,
       departure_time
  FROM zflight
  WHERE origin_airport = 'MAA'
  INTO TABLE @DATA(lt_flights).
```

This reads all flights departing from Chennai into an internal table.

### INSERT, UPDATE, DELETE

```abap
INSERT zbooking FROM @ls_booking.

UPDATE zbooking
  SET booking_status = 'CONFIRMED'
  WHERE booking_id = @lv_booking_id.

DELETE FROM zbooking
  WHERE booking_id = @lv_booking_id.
```

In real systems, direct database changes must be carefully controlled because booking, payment, and inventory data must remain consistent.

## Modularization

Instead of writing one giant program, ABAP encourages reusable blocks.

### Subroutines

Older procedural style:

```abap
PERFORM calculate_baggage_fee USING lv_weight
                             CHANGING lv_fee.

FORM calculate_baggage_fee USING iv_weight TYPE p
                           CHANGING cv_fee TYPE p.
  IF iv_weight > 23.
    cv_fee = ( iv_weight - 23 ) * 500.
  ELSE.
    cv_fee = 0.
  ENDIF.
ENDFORM.
```

### Function modules

A **function module** is a reusable server-side function stored in a **function group**. It can be called from other ABAP programs and sometimes exposed to external systems through **RFC** (**Remote Function Call**).

Example: calculate the total fare for a booking.

```abap
CALL FUNCTION 'Z_CALCULATE_BOOKING_FARE'
  EXPORTING
    iv_flight_no   = lv_flight_no
    iv_passengers  = lv_passenger_count
  IMPORTING
    ev_total_fare  = lv_total_fare
  EXCEPTIONS
    flight_not_found = 1.
```

### Includes

An **include** is a reusable piece of source code inserted into another program. It is useful for shared constants, type definitions, or form routines.

## Object-oriented ABAP

Modern ABAP supports **OOP** (**Object-Oriented Programming**): classes, objects, methods, inheritance, interfaces, and events. SAP added object-oriented ABAP, known as **ABAP Objects**, in 2000. [learning.sap-press](https://learning.sap-press.com/abap)

For an airline, model real-world things as classes:

```abap
CLASS zcl_flight DEFINITION.
  PUBLIC SECTION.
    METHODS:
      constructor
        IMPORTING iv_flight_no TYPE string
                  iv_origin TYPE string
                  iv_destination TYPE string,
      get_duration_minutes
        RETURNING VALUE(rv_minutes) TYPE i,
      is_delayed
        RETURNING VALUE(rv_delayed) TYPE abap_bool.
ENDCLASS.
```

A class is like a blueprint for a flight. Each actual flight—AI202, 6E123, UK850—is an object created from that blueprint.

### Class implementation

```abap
CLASS zcl_flight IMPLEMENTATION.
  METHOD constructor.
    me->flight_no = iv_flight_no.
    me->origin = iv_origin.
    me->destination = iv_destination.
  ENDMETHOD.

  METHOD get_duration_minutes.
    rv_minutes = 165. " Simplified example
  ENDMETHOD.

  METHOD is_delayed.
    rv_delayed = xsdbool( actual_departure > scheduled_departure ).
  ENDMETHOD.
ENDCLASS.
```

### Interfaces

An **interface** defines a contract without implementation.

```abap
INTERFACE zif_airport_service.
  METHODS:
    get_slot
      IMPORTING iv_airport_code TYPE string
                iv_flight_no TYPE string
      RETURNING VALUE(rv_slot) TYPE string.
ENDINTERFACE.
```

Different airports can implement this differently:

- Chennai Airport may allocate slots by runway capacity.
- Delhi Airport may prioritize international connections.
- A smaller airport may allow walk-in slot requests.

## ABAP Dictionary: DDIC

**DDIC** means **ABAP Dictionary**. It is SAP’s central place to define reusable data structures, tables, data elements, domains, and search helps.

| DDIC object | Meaning | Airline example |
|---|---|---|
| Domain | Technical value range and formatting | Airport code must be 3 uppercase letters |
| Data element | Business meaning plus labels | `Airport Code` with label “Airport” |
| Table | Persisted database structure | `ZFLIGHT` |
| Structure | Reusable field grouping | Flight route structure |
| Search help | User-friendly value lookup | Search airports by city or code |
| Lock object | Prevents conflicting updates | Prevents two agents booking the last seat simultaneously |

Example: define an airport code domain that accepts only three characters, then reuse it in `ZFLIGHT`, `ZAIRPORT`, and `ZBOOKING`.

## SAP transactions and programs

A **transaction code**, or **T-code**, is a shortcut to an SAP screen or task.

Common development-related T-codes:

| T-code | Full meaning | Use |
|---|---|---|
| `SE80` | Object Navigator | Main ABAP development workbench |
| `SE38` | ABAP Editor | Write and run reports/programs |
| `SE11` | ABAP Dictionary | Create tables, structures, domains |
| `SE37` | Function Builder | Create function modules |
| `SE24` | Class Builder | Create ABAP classes |
| `SE16` / `SE16N` | Data Browser | View table contents |
| `ST22` | ABAP Dump Analysis | Investigate runtime errors |
| `SM30` | Table Maintenance | Maintain custom table entries |
| `SFP` | Form Builder | Design Adobe Forms |
| `SMARTFORMS` | Smart Forms | Design printable business forms |

For an airline, `SMARTFORMS` or Adobe Forms could generate:

- Boarding passes.
- Baggage tags.
- Cargo airway bills.
- Refund receipts.
- Airport slot confirmation letters.

## Reports and ALV

A **report** is a program that reads data and displays it. **ALV** means **ABAP List Viewer**, SAP’s standard grid/table display technology.

Example: show all delayed flights.

```abap
SELECT flight_no,
       origin_airport,
       destination_airport,
       scheduled_departure,
       actual_departure
  FROM zflight
  WHERE actual_departure > scheduled_departure
  INTO TABLE @DATA(lt_delayed_flights).

cl_salv_table=>factory(
  IMPORTING
    r_salv_table = DATA(lo_alv)
  CHANGING
    t_table      = lt_delayed_flights ).

lo_alv->display( ).
```

An operations manager could use this report to see delayed departures by airport, airline, or route.

## Forms and user interfaces

ABAP backend logic can support several UI technologies:

- **SAP GUI** — the classic desktop client.
- **Dynpro** — classic SAP screen programming.
- **Web Dynpro ABAP** — older browser-based SAP UI framework.
- **SAP Fiori / SAPUI5** — modern web user experience.
- **ABAP RESTful Application Programming Model (RAP)** — modern way to build OData services and Fiori apps from ABAP.

For example, a modern airport ground-staff app could use Fiori for the screen and ABAP/RAP for the backend logic that validates boarding passes, assigns gates, and updates baggage status.

## Enhancements and customization

SAP software is standard software. Customers should avoid modifying SAP’s core code directly because upgrades become difficult. Instead, SAP provides extension mechanisms.

### User exits

Older hooks in standard programs where customer code can be added.

Example: after a booking is saved, add a custom airline rule to apply a frequent-flyer discount.

### Customer exits

More structured enhancement points, often in function-module form.

### BAdIs

**BAdI** means **Business Add-In**. It is SAP’s object-oriented enhancement technique. You implement an interface that SAP calls at a defined point.

Example: SAP calls a BAdI before saving a flight booking. Your airline adds a rule:

```abap
IF ls_booking-passenger_count > ls_flight-available_seats.
  RAISE EXCEPTION TYPE zcx_booking_overbooked.
ENDIF.
```

### Enhancement spots

Modern, explicitly defined places in standard SAP code where custom logic can be inserted.

### BAdI vs user exit

| Mechanism | Style | Typical use |
|---|---|---|
| User exit | Older, procedural | Legacy custom logic |
| Customer exit | Function-module based | Older enhancements |
| BAdI | Object-oriented | Preferred modern extension point |
| Enhancement spot | Modern explicit hook | Insert custom logic at defined points |

## Bad practice: modification

A **modification** changes SAP standard code directly. It may work initially but creates upgrade and support problems.

Bad approach:

```abap
" Directly editing SAP standard booking program
IF airline_code = 'AI'.
  " Add Air India-specific logic here
ENDIF.
```

Better approach:

```abap
" Use a BAdI implementation
IF airline_code = 'AI'.
  apply_air_india_booking_rules( ls_booking ).
ENDIF.
```

## Debugging and error handling

### Debugging

ABAP debugging lets you pause a program, inspect variables, and step through logic.

Use cases:

- Why was a passenger charged the wrong baggage fee?
- Why did a cargo invoice fail?
- Why is an airport slot not being released?

### Exceptions

Use **exception classes** for predictable business errors.

```abap
TRY.
    lo_booking->confirm( ).
  CATCH zcx_seat_unavailable INTO DATA(lx_error).
    MESSAGE lx_error->get_text( ) TYPE 'E'.
ENDTRY.
```

For example, if a customer tries to confirm a booking after all seats are sold, raise `zcx_seat_unavailable` rather than allowing corrupt data.

### Runtime errors

A **short dump** is an ABAP runtime crash, visible in transaction `ST22` (**ABAP Dump Analysis**). Common causes include dividing by zero, accessing an unassigned object reference, or reading a nonexistent table row.

## Performance and HANA

Modern SAP systems often use **SAP HANA**, an in-memory database. Instead of bringing millions of rows into ABAP and filtering them there, push filtering and aggregation to the database. SAP explicitly recommends **code pushdown** using ABAP SQL. [learning.sap](https://learning.sap.com/courses/deepening-your-abap-programming-knowledge)

### Poor approach

```abap
SELECT * FROM zbooking INTO TABLE lt_bookings.

LOOP AT lt_bookings INTO ls_booking.
  IF ls_booking-origin_airport = 'MAA'.
    ADD 1 TO lv_count.
  ENDIF.
ENDLOOP.
```

### Better approach

```abap
SELECT COUNT(*)
  FROM zbooking
  WHERE origin_airport = 'MAA'
  INTO @DATA(lv_count).
```

The second approach asks the database to count only Chennai-origin bookings, instead of loading every booking into application memory.

Other performance practices:

- Select only required fields, not `SELECT *`.
- Use proper indexes for frequent search fields.
- Avoid database access inside nested loops.
- Use internal-table keys and sorted/hashed tables where appropriate.
- Use **ATC** (**ABAP Test Cockpit**) to find quality and performance issues.
- Use **SQL Monitor** to identify expensive SQL statements.

## Transports and change management

ABAP development happens in systems that usually follow a landscape:

| System | Purpose |
|---|---|
| **DEV** | Development system |
| **QAS** | Quality Assurance/System testing |
| **PRD** | Production system |

A **transport request** packages changes—programs, tables, classes, forms—so they can move from DEV to QAS and then PRD.

Airline example:

1. Developer creates baggage-fee logic in DEV.
2. Tester verifies 23 kg, 30 kg, and invalid-weight cases in QAS.
3. After approval, the transport moves the logic to PRD.
4. Airport check-in staff begin using the updated fee rule.

## Testing and quality

ABAP supports unit testing through **ABAP Unit**.

```abap
CLASS ltcl_baggage_fee DEFINITION
  FOR TESTING
  RISK LEVEL HARMLESS
  DURATION SHORT.

  PRIVATE SECTION.
    METHODS test_normal_baggage FOR TESTING.
    METHODS test_extra_baggage FOR TESTING.
ENDCLASS.

CLASS ltcl_baggage_fee IMPLEMENTATION.
  METHOD test_normal_baggage.
    DATA(lv_fee) = zcl_baggage=>calculate_fee( 20 ).
    cl_abap_unit_assert=>assert_equals(
      exp = 0
      act = lv_fee ).
  ENDMETHOD.

  METHOD test_extra_baggage.
    DATA(lv_fee) = zcl_baggage=>calculate_fee( 25 ).
    cl_abap_unit_assert=>assert_equals(
      exp = 1000
      act = lv_fee ).
  ENDMETHOD.
ENDCLASS.
```

This ensures the baggage-fee rule remains correct when developers change the code later.

## Integration with other systems

Airlines rarely run only SAP. ABAP can integrate with:

- **IDoc** — **Intermediate Document**: SAP’s standard asynchronous document format, often used for EDI-style business messages.
- **RFC** — **Remote Function Call**: SAP-to-SAP or external synchronous calls.
- **BAPI** — **Business Application Programming Interface**: standardized SAP business-object APIs.
- **OData** — Open Data Protocol, commonly used by Fiori apps.
- **REST APIs** — HTTP-based integrations.
- **SAP PI/PO or CPI** — integration middleware; **PI/PO** means Process Integration/Process Orchestration, while **CPI** means Cloud Platform Integration.

Airline examples:

- Send flight-delay messages to a passenger-notification system.
- Receive airport slot allocations from an external airport-management platform.
- Send cargo airway bills to a logistics partner.
- Sync crew rosters with an HR or workforce-management system.
- Publish booking data to a revenue-management system.

## Modern ABAP: RAP and ABAP Cloud

**RAP** means **ABAP RESTful Application Programming Model**. It is SAP’s modern approach for building enterprise services and Fiori apps on ABAP.

A RAP application typically has:

| RAP layer | Meaning | Airline example |
|---|---|---|
| Data model | CDS views over tables | Flight and booking data |
| Behavior definition | Business operations | Create booking, cancel booking, check in |
| Behavior implementation | ABAP logic | Validate seats, calculate fare, issue boarding pass |
| Service definition/binding | Exposes OData UI/API | Fiori “Manage Flight Bookings” app |

**CDS** means **Core Data Services**—SAP’s semantically rich data-modeling layer. It lets you define business views with associations, annotations, and calculations.

Example concept:

```abap
@AccessControl.authorizationCheck: #CHECK
define view entity ZC_FlightBooking
  as select from zbooking
{
  key booking_id,
      passenger_id,
      flight_no,
      booking_status,
      total_fare
}
```

**ABAP Cloud** is SAP’s cloud-ready development model. It restricts certain obsolete or system-specific APIs and encourages clean, upgrade-stable, cloud-compatible extensions. ABAP remains central to SAP’s portfolio, including S/4HANA and SAP BTP (**Business Technology Platform**) development. [learning.sap-press](https://learning.sap-press.com/abap)

## Learning path

1. **ABAP syntax basics** — variables, structures, internal tables, loops, conditions.
2. **ABAP Dictionary** — tables, data elements, domains, search helps.
3. **Open SQL / ABAP SQL** — SELECT, INSERT, UPDATE, DELETE, joins, aggregations.
4. **Modularization** — function modules, includes, classes, methods.
5. **Object-oriented ABAP** — classes, interfaces, inheritance, exceptions.
6. **Reports and ALV** — operational airline reports.
7. **Debugging and testing** — debugger, ABAP Unit, ATC.
8. **Enhancements** — BAdIs, enhancement spots, user exits.
9. **Forms and UI** — Smart Forms, Adobe Forms, Fiori basics.
10. **Modern stack** — CDS views, RAP, OData, ABAP Cloud, SAP BTP.

## How your Java background maps

| Java concept | ABAP equivalent |
|---|---|
| Class | ABAP class |
| Interface | ABAP interface |
| Method | Method |
| `List<T>` | Internal table |
| POJO / DTO | Structure |
| JPA entity / table | DDIC table |
| Spring service | ABAP class or function module |
| REST controller | RAP service definition/binding or OData service |
| Exception class | ABAP exception class |
| Maven/Gradle dependency | SAP package / transport |
| JUnit | ABAP Unit |
| Spring Data query | ABAP SQL / CDS view |
| Application server | AS ABAP |

Your strongest transferable advantage is object-oriented design: model `Flight`, `Airport`, `Passenger`, `Booking`, `Baggage`, and `Crew` as clean classes, keep business rules in services, and avoid scattering logic across reports and forms.

## A complete mini example

Below is a simplified ABAP class that calculates baggage fees for an airline.

```abap
CLASS zcl_baggage_fee DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC.

  PUBLIC SECTION.
    TYPES:
      BEGIN OF ty_baggage,
        booking_id       TYPE string,
        passenger_id     TYPE string,
        weight_kg        TYPE p DECIMALS 2,
        extra_fee_inr    TYPE p DECIMALS 2,
      END OF ty_baggage.

    CONSTANTS gc_free_allowance_kg TYPE p DECIMALS 2 VALUE 23.
    CONSTANTS gc_fee_per_kg_inr    TYPE p DECIMALS 2 VALUE 500.

    CLASS-METHODS calculate_fee
      IMPORTING iv_weight_kg TYPE p DECIMALS 2
      RETURNING VALUE(rv_fee) TYPE p DECIMALS 2.

ENDCLASS.

CLASS zcl_baggage_fee IMPLEMENTATION.

  METHOD calculate_fee.
    IF iv_weight_kg <= gc_free_allowance_kg.
      rv_fee = 0.
    ELSE.
      rv_fee = ( iv_weight_kg - gc_free_allowance_kg )
               * gc_fee_per_kg_inr.
    ENDIF.
  ENDMETHOD.

ENDCLASS.
```

Usage:

```abap
DATA(lv_fee) = zcl_baggage_fee=>calculate_fee( 27 ).
" Result: 2000 INR for 4 extra kg
```

## What ABAP developers actually do

A typical ABAP developer’s work includes:

- Analyzing functional requirements from SAP consultants or business users.
- Creating custom tables, CDS views, classes, and programs.
- Building reports, interfaces, forms, workflows, and Fiori backend services.
- Enhancing standard SAP processes through BAdIs and enhancement spots.
- Debugging production issues and performance problems.
- Writing unit tests and documenting custom objects.
- Moving changes through DEV, QAS, and PRD using transport requests.

In airline terms, an ABAP developer might build the backend that ensures a passenger’s booking, baggage fee, tax calculation, refund, and frequent-flyer points all remain consistent across the airline’s SAP landscape. ABAP supports both procedural and object-oriented styles, has integrated database access, and remains a foundation for SAP’s mission-critical business applications. [help.sap](https://help.sap.com/doc/abapdocu_latest_index_htm/latest/en-US/ABENABAP_OVERVIEW.html)
