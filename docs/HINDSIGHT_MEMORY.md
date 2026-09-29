# How Hindsight Memory Is Used in Patch Drift

## 1. Introduction

Patch Drift identifies API changes that cause previously working requests to fail.

When a failure occurs, the system needs to determine:

* Is the failure actually caused by API drift?
* Have similar failures occurred before?
* What repairs worked or failed previously?
* Is the previous experience relevant to the current failure?
* Should a repair be attempted?
* Can the repair be safely verified?

To support this process, Patch Drift uses **Hindsight as a persistent experience-memory layer**.

Hindsight stores previous API incidents, detected drift patterns, repair attempts, and their observed outcomes. Relevant experiences can then be retrieved when a new failure occurs and used as **additional historical evidence**.

> **Hindsight provides experience. It does not make or apply the repair decision by itself.**

---

## 2. Role of Hindsight

Hindsight has two primary responsibilities:

1. **Store structured experiences from previous API incidents.**
2. **Retrieve relevant experiences during future incidents.**

The overall process is:

```text
API Failure
     ↓
Current Analysis
     ↓
Retrieve Historical Evidence
     ↓
Combine Current + Historical Evidence
     ↓
Generate Repair Candidate
     ↓
Safety / Decision Evaluation
     ↓
Retry & Verify
     ↓
Store New Experience
```

Hindsight therefore acts as a **memory and retrieval component**, while other components remain responsible for analysis, repair, safety, and verification.

---

## 3. Responsibilities of the Components

| Component                   | Responsibility                                    |
| --------------------------- | ------------------------------------------------- |
| **Failure Detector**        | Detects a failed API request                      |
| **Drift Analyzer**          | Analyzes whether API behavior may have changed    |
| **Hindsight**               | Stores and retrieves previous API experiences     |
| **Repair Engine**           | Generates possible repairs                        |
| **Safety / Decision Layer** | Evaluates evidence and decides whether to proceed |
| **Retry Engine**            | Tests the proposed repair                         |
| **Verification**            | Determines whether the repair actually worked     |

The distinction is important:

```text
Hindsight
    ↓
Provides historical evidence

Repair Engine
    ↓
Generates repair candidate

Safety / Decision Layer
    ↓
Determines whether to attempt it

Retry / Verification
    ↓
Tests the result
```

---

## 4. What Hindsight Stores

A Hindsight experience can contain:

```text
API / Service
HTTP Method
Endpoint Pattern
Request Structure
HTTP Status
Error Message
Observed Response
Detected Drift Type
Original Request
Repair Attempted
Retry Result
Verification Result
Relevant Context / Metadata
```

For example:

```text
API:
Event API

Failure:
404 Not Found

Original:
GET /api/events?event_id=101

Detected Drift:
Query parameter → Path parameter

Repair:
GET /api/events/101

Result:
SUCCESS

Verification:
200 OK
```

Hindsight is therefore more than a raw error log. It stores **structured, retrieval-oriented experiences** that can be useful during future API incidents.

---

## 5. Example: Endpoint Structure Changed

Suppose CampusConnect originally uses:

```text
GET /api/events?event_id=101
```

The API provider later changes the endpoint to:

```text
GET /api/events/101
```

CampusConnect continues using the old request and receives:

```text
404 Not Found
```

### Current Analysis

The Drift Analyzer identifies a possible:

```text
Query parameter → Path parameter
```

change.

Patch Drift then queries Hindsight for relevant previous experiences.

Suppose it finds:

```text
Previous Incident

Original:
GET /students?id=25

Detected Drift:
Query parameter → Path parameter

Repair:
GET /students/25

Result:
SUCCESS
```

The URLs are different, but the **structural drift pattern is similar**.

This historical experience provides evidence that moving the parameter into the path may be worth considering.

The Repair Engine can generate:

```text
GET /api/events/101
```

The Safety / Decision Layer evaluates the candidate before it is attempted.

If the retry returns:

```text
200 OK
```

the repair is verified and the new experience is stored in Hindsight.

---

## 6. How Relevant Experiences Are Retrieved

Hindsight should not require an identical URL to find a useful experience.

Retrieval can consider factors such as:

* API or service
* HTTP method
* Status code
* Error pattern
* Endpoint structure
* Request structure
* Detected drift type
* Previous repair pattern
* Relevant context

For example:

```text
Current:
404 + endpoint structure changed

Previous:
404 + similar endpoint structure change
```

may be more relevant than simply searching for an identical URL.

However:

> **Previous experiences are evidence, not proof.**

A highly similar incident can still have a different underlying cause. The current API must still be analyzed and the proposed repair must be verified.

---

## 7. Handling Memory Uncertainty

### No relevant experience

Hindsight may have no matching experience.

```text
Failure
   ↓
Query Hindsight
   ↓
No useful experience
   ↓
Continue using current evidence
```

The system must still work for completely new types of API drift.

### Conflicting experiences

Hindsight may return different outcomes:

```text
Experience A → Repair succeeded
Experience B → Repair failed
```

The system should not blindly choose one.

It should consider:

* Relevance
* Similarity
* Recency
* Previous outcomes
* Current API evidence

### Failed repairs

Failed repairs should also be stored.

A failed experience can provide evidence against repeatedly attempting the same unsuccessful repair pattern in a similar situation.

---

## 8. Role in False-Alarm Reduction

Not every API failure is API drift.

For example:

```text
401 Unauthorized
      ↓
Previous authentication failures
      +
No evidence of endpoint change
      ↓
Weak support for endpoint drift
```

In contrast:

```text
404 Not Found
      ↓
Endpoint structure changed
      +
Similar successful historical repair
      ↓
Additional evidence supporting drift
```

Hindsight therefore contributes historical evidence to the decision process.

It does **not** independently determine whether a failure is drift, nor does it guarantee false-alarm reduction.

---

## 9. Complete Architecture

```text
                 API Failure
                      ↓
              Current Analysis
                      │
                      │
             ┌────────┴────────┐
             ↓                 ↓
      Current Evidence     Hindsight
                           Historical
                            Evidence
             └────────┬────────┘
                      ↓
              Decision Layer
                      ↓
              Repair Candidate
                      ↓
                Safety Check
                      ↓
                    Retry
                      ↓
                Verification
                      ↓
             Store Experience
                      ↓
                  Hindsight
                      │
                      └────→ Future Incidents
```

This creates an experience loop:

```text
Observe → Analyze → Retrieve → Repair → Verify → Remember
```

---

## 10. Hindsight vs. Other Components

The simplest way to explain the architecture is:

| Component                   | Main Question                               |
| --------------------------- | ------------------------------------------- |
| **Drift Analyzer**          | Does this failure look like API drift?      |
| **Hindsight**               | What happened in similar situations before? |
| **Repair Engine**           | What could potentially fix this?            |
| **Safety / Decision Layer** | Should we attempt this repair?              |
| **Retry / Verification**    | Did the repair actually work?               |

> **Hindsight is the case archive, not the detective.**

---

## 11. Technical Design Principle

The central principle is:

> **Memory should influence decisions, not replace decisions.**

Patch Drift combines:

```text
Current Evidence
      +
Historical Evidence
      +
Safety Constraints
      ↓
Repair Decision
      ↓
Actual API Verification
```

This prevents the system from blindly copying previous repairs.

A previous successful repair can support a new repair candidate, but the **current API response remains the final verification authority**.

---

## 12. One-Minute Evaluator Explanation

If asked **“How are you using Hindsight in Patch Drift?”**, the answer is:

> **“Hindsight is our persistent experience-memory layer. When an API failure occurs, Patch Drift analyzes the current request and retrieves relevant previous incidents from Hindsight. These experiences contain detected drift patterns, previous repair attempts, and their observed outcomes. We use this historical information as additional evidence when generating and evaluating a new repair candidate. Hindsight does not directly apply the repair. The Repair Engine generates the candidate, the Safety and Decision Layer evaluates it, and the Retry mechanism verifies the result. The new outcome is then stored back in Hindsight for future incidents.”**

---

## 13. Final Takeaway

Without Hindsight:

```text
Failure
   ↓
New Investigation
```

With Hindsight:

```text
Failure
   ↓
Current Investigation
   +
Relevant Previous Experiences
   ↓
Repair Decision
   ↓
Verification
   ↓
New Experience
```

Hindsight gives Patch Drift a persistent memory of previous API incidents.

The system can therefore ask not only:

> **“What is happening now?”**

but also:

> **“What relevant experiences do we already have, and what did their outcomes tell us?”**