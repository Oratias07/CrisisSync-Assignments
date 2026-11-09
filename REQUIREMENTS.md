# CrisisSync – Assignment 1: Requirements, Traceability Matrix and Acceptance Tests

**Group number:** 20

| # | Name | ID Number | College Email |
|---|------|-----------|---------------|
| 1 | Or Atias | as Request | as Request |
| 2 | Shada Dabbah | 326447000 | shada.dabah@e.braude.ac.il |
| 3 | Seren Amoun | 215081084 | Seren.Amoun@e.braude.ac.il |
| 4 | Leen Abu Salah | 212920672 | Leen.Abusalah@e.braude.ac.il |

---

## 1. Scope

This document defines requirements and acceptance tests for two operations of the CrisisSync system:

1. **Operation 1 – Reserve a resource package for a task, including reporting task completion.**
2. **Operation 2 – Produce the SLA (timeliness and service level) report.**

Use case names refer to the model in `G20_Assignment1.vpp`.
Requirement identifiers: `FR-R#` / `NFR-R#` (Operation 1) and `FR-S#` / `NFR-S#` (Operation 2).

---

## 2. Operation 1 – Reserve Resource Package and Report Task Completion

### 2.1 Functional Requirements (FR)

| ID | Requirement |
|----|-------------|
| FR-R1 | The system shall allow an authenticated Dispatcher to request reservation of the resource package of a task only when the task is in state *Ready*, and shall reject the request, without changing any resource status, when the task is in any other state. |
| FR-R2 | The system shall reserve all resources of a package within a single SQL transaction that locks the requested resource records, verifies that every requested resource is *Available*, sets every requested resource to *Reserved* and sets the task to state *Active*. |
| FR-R3 | The system shall roll back the entire reservation transaction, reserve no resource and keep the task in state *Ready* when at least one requested resource is not *Available*, including when two Dispatchers request the same resource concurrently, in which case exactly one request shall succeed; the system shall display to the failed Dispatcher a message naming each unavailable resource. |
| FR-R4 | The system shall accept a task completion report only from the Field Commander designated for the task, and only while the task is in state *Active*. |
| FR-R5 | The system shall, upon an accepted completion report, set the task to *Completed*, set every resource reserved to the task to *Available* immediately, and set to *Ready* every dependent task whose prerequisite tasks are all *Completed*. |

### 2.2 Non-Functional Requirements (NFR)

| ID | Category | Requirement |
|----|----------|-------------|
| NFR-R1 | Performance | The system shall return the result of a reservation request for a package of up to 10 resources to the Dispatcher within 2 seconds for at least 95% of requests, with 20 Dispatchers sending requests concurrently. |
| NFR-R2 | Security | The system shall verify the user role on the server for 100% of reservation and completion-report requests, and shall reject, with no change to task or resource state, every request from a user who is not an authenticated Dispatcher (reservation) or not the designated Field Commander (completion report). |
| NFR-R3 | Reliability | The system shall leave 0 partially reserved packages after a database failure or lost connection during a reservation transaction, verified in 100 consecutive fault-injection runs, and shall accept a new reservation request within 10 seconds after the failure is detected. |
| NFR-R4 | Usability | The system shall display the reservation-failure message within 1 second of the failure, and shall allow a Field Commander to report task completion within at most 2 clicks starting from the live event status screen. |

### 2.3 Acceptance Tests (BDD) – Operation 1

#### AT-R01 – Successful reservation (normal path)
```gherkin
Scenario: Dispatcher reserves a full package for a Ready task
  Given a Dispatcher is logged in
  And task "Debris Removal" is in state "Ready"
  And its package (1 Field Commander, 1 Heavy Equipment Operator, 1 Bulldozer, 1 Paramedic) is fully "Available"
  When the Dispatcher requests reservation of the package
  Then all 4 resources are "Reserved"
  And task "Debris Removal" is "Active"
```

#### AT-R02 – Reservation fails because a resource is unavailable (exception path)
```gherkin
Scenario: One resource is taken by a concurrent request
  Given two Dispatchers each have a "Ready" task whose package contains Bulldozer "BD-7" with status "Available"
  When both Dispatchers request reservation at the same moment
  Then exactly one request succeeds and "BD-7" is reserved to that task only
  And the other transaction is rolled back, none of its resources is "Reserved" and its task remains "Ready"
  And the failed Dispatcher sees, within 1 second, a message naming "BD-7"
```

#### AT-R03 – Task completion (normal path)
```gherkin
Scenario: Designated Field Commander reports completion
  Given task T1 is "Active" with 4 "Reserved" resources
  And task T2 is "Blocked" and depends only on T1
  And task T3 is "Blocked" and depends on T1 and on a task that is not "Completed"
  And the Field Commander designated for T1 is on the live event status screen
  When he reports completion of T1 using at most 2 clicks
  Then T1 is "Completed" and its 4 resources are "Available"
  And T2 is "Ready"
  And T3 remains "Blocked"
```

#### AT-R04 – Invalid or unauthorized requests are rejected (exception path)
```gherkin
Scenario Outline: Request that must be rejected
  Given <user> is logged in
  And <situation>
  When <user> <action>
  Then the system rejects the request
  And no task or resource changes state

  Examples:
    | user                           | situation                              | action                          |
    | a Dispatcher                   | the task is "Blocked"                  | requests reservation            |
    | a Field Crew Member            | the task is "Ready"                    | requests reservation            |
    | a Field Commander not assigned | the task is "Active" with another Field Commander designated | reports completion of the task |
```

#### AT-R05 – Recovery from database failure during reservation (error / timeout path)
```gherkin
Scenario: Connection to the database is lost during the transaction
  Given task "Debris Removal" is "Ready" with a 4-resource package, all "Available"
  And the connection is cut after 2 resources were updated inside the transaction
  When the failure is detected
  Then the transaction is rolled back and 0 resources are "Reserved"
  And within 10 seconds a new reservation request for the same task is accepted and succeeds
```

#### AT-R06 – Reservation response time (performance)
```gherkin
Scenario: 20 Dispatchers reserve concurrently
  Given 20 Dispatchers each have a "Ready" task with a package of 10 "Available" resources
  When all 20 send a reservation request at the same moment
  Then at least 95% of the results are returned to their Dispatchers within 2 seconds
```

---

## 3. Operation 2 – Produce the SLA (Timeliness and Service Level) Report

### 3.1 Functional Requirements (FR)

| ID | Requirement |
|----|-------------|
| FR-S1 | The system shall generate the SLA report automatically at the end of each calendar month, according to the system clock. |
| FR-S2 | The system shall allow the System Administrator to generate the SLA report on demand for a selected past month, and shall reject the selection of a month that has not ended. |
| FR-S3 | The system shall present in the SLA report, for each of the 5 event types, the average time from the opening of an event to the completion of its last task, and shall show an event type with no events in the month as 0 events with no average value. |
| FR-S4 | The system shall allow only the System Administrator to view the SLA report. |

### 3.2 Non-Functional Requirements (NFR)

| ID | Category | Requirement |
|----|----------|-------------|
| NFR-S1 | Performance | The system shall generate and display the SLA report for a month containing up to 1,000 events within 10 seconds. |
| NFR-S2 | Security | The system shall verify the System Administrator role on the server for 100% of SLA report requests, and shall return no report data to any other user. |
| NFR-S3 | Reliability | The system shall abort an SLA report generation that fails or exceeds 30 seconds, store 0 partial reports, display an error message within 2 seconds of the abort, and accept a new report request immediately afterwards without a server restart. |
| NFR-S4 | Usability | The system shall let the System Administrator obtain the report of a past month within at most 3 user actions from the main screen, and shall display every average duration in the format hh:mm. |

### 3.3 Acceptance Tests (BDD) – Operation 2

#### AT-S01 – Automatic monthly generation (normal path)
```gherkin
Scenario: End of month triggers the SLA report
  Given the system clock reaches the end of calendar month M
  When the scheduler fires
  Then the SLA report of month M is generated automatically
  And the System Administrator can view it
```

#### AT-S02 – On-demand report for a past month (normal path)
```gherkin
Scenario: Administrator requests the SLA report of a past month
  Given the System Administrator is on the main screen
  And in month M there are two "Fire" events whose last tasks were completed 2:00 and 4:00 hours after the events were opened
  And month M has no "Natural Disaster" events
  When he selects month M and requests the report using at most 3 actions
  Then the report shows for "Fire" 2 events with an average of 03:00
  And the report shows "Natural Disaster" with 0 events and no average
  And all 5 event types are listed
```

#### AT-S03 – Invalid month or unauthorized user is rejected (exception path)
```gherkin
Scenario Outline: Request that must be rejected
  Given <user> is logged in
  When <user> requests the SLA report for <month>
  Then the system rejects the request
  And no report data is returned

  Examples:
    | user                        | month                          |
    | the System Administrator    | the current month (not ended)  |
    | a Dispatcher                | a past month                   |
```

#### AT-S04 – Recovery from generation failure or timeout (error / timeout path)
```gherkin
Scenario: Report generation exceeds the time limit
  Given the System Administrator requests the SLA report
  And the generation fails or exceeds 30 seconds
  When the system aborts the generation
  Then no partial report is stored
  And an error message is displayed within 2 seconds of the abort
  And a new request issued immediately afterwards succeeds without a server restart
```

#### AT-S05 – Report generation time (performance)
```gherkin
Scenario: Month with 1,000 events
  Given month M contains 1,000 events
  When the System Administrator requests the SLA report of month M
  Then the report is displayed within 10 seconds
```

---

## 4. Requirements Traceability Matrix (RTM)

| Requirement | Type | Implementing Use Case(s) in the Model | Acceptance Test ID(s) |
|-------------|------|----------------------------------------|------------------------|
| FR-R1 | FR | Reserve Resource Package | AT-R01, AT-R04 |
| FR-R2 | FR | Reserve Package Atomically | AT-R01 |
| FR-R3 | FR | Reserve Package Atomically; Handle Reservation Conflict | AT-R02 |
| FR-R4 | FR | Report Task Completion | AT-R03, AT-R04 |
| FR-R5 | FR | Report Task Completion; Release Task Resources; Promote Dependent Tasks | AT-R03 |
| NFR-R1 | NFR (Performance) | Reserve Resource Package; Reserve Package Atomically | AT-R06 |
| NFR-R2 | NFR (Security) | Reserve Resource Package; Report Task Completion | AT-R04 |
| NFR-R3 | NFR (Reliability) | Reserve Package Atomically | AT-R05 |
| NFR-R4 | NFR (Usability) | Handle Reservation Conflict; Report Task Completion | AT-R02, AT-R03 |
| FR-S1 | FR | Generate SLA Report | AT-S01 |
| FR-S2 | FR | Generate SLA Report | AT-S02, AT-S03 |
| FR-S3 | FR | Generate SLA Report | AT-S02 |
| FR-S4 | FR | Generate SLA Report | AT-S03 |
| NFR-S1 | NFR (Performance) | Generate SLA Report | AT-S05 |
| NFR-S2 | NFR (Security) | Generate SLA Report | AT-S03 |
| NFR-S3 | NFR (Reliability) | Generate SLA Report | AT-S04 |
| NFR-S4 | NFR (Usability) | Generate SLA Report | AT-S02 |
