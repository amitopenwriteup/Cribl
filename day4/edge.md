# Cribl Edge - Basic Concepts: Slide-by-Slide Explanation

---

## Slide 1: Title Slide
**Title:** Cribl Edge - Basic Concepts

### Overview:
This is the introductory slide that sets the stage for the presentation. It introduces the main topics that will be covered:
- **Sources** - Where data enters the system
- **Routes** - How data is directed through the system
- **Pipelines** - Processing chains for data
- **Functions** - Individual processing units

This slide establishes the foundational vocabulary and core concepts that form the architecture of Cribl Edge.

---

## Slide 2: What Cribl Edge Does

### Main Concept:
Cribl Edge collects, processes, and delivers observability data in near real-time.

### Key Points:

**Data Collection Locations:**
- Edge runs on Linux, Windows, and macOS machines
- Collects logs, metrics, and application data
- Delivers data to Cribl Stream or any supported destination

**The Data Flow Architecture:**
```
Machines & Apps → Sources → Routes → Pipelines → Destinations
```

### Process Breakdown:
1. **Sources** collect data from various machines and applications
2. **Routes** determine where the data should flow
3. **Pipelines** are built from Functions that process the data
4. **Destinations** receive the processed events

This slide provides the high-level view of how all the components work together in an integrated system.

---

## Slide 3: Sources

### Definition:
Sources are configurations that allow Edge to collect data from local machines or receive data from remote senders.

### Two Data Collection Methods:

#### 1. Local Collection
- Reads logs, metrics, and other data directly from the machine where Edge runs
- Monitors the local system for observability data

#### 2. Remote Receivers
- Accepts data pushed from other systems
- Supports protocols like TCP and Syslog
- Enables centralized data collection from distributed sources

### Key Insight:
Sources are the entry point for all data into Cribl Edge, whether collected locally or received remotely.

---

## Slide 4: QuickConnect

### Purpose:
QuickConnect is a graphical interface that provides a fast path from Sources directly to Destinations, with optional processing.

### How It Works:
- Uses a **drag-and-drop** interface
- Connect Sources directly to Destinations
- Optionally include or exclude Pipelines or Packs in the connection
- Accessible via: **Fleet menu → Collect**

### Key Constraint:
**QuickConnect completely bypasses Routes:**
- No routing table involved
- No conditional cloning or cascading
- Every connection is **parallel and independent**
- Each Source-to-Destination connection operates independently

### Use Case:
Ideal for simple, direct data flows where complex routing logic isn't needed. Provides the fastest configuration path for basic data collection and delivery scenarios.

---

## Slide 5: Routes

### Definition:
Routes evaluate incoming events against filter expressions to determine which Pipeline should handle each event.

### How Routes Work:
- Routes are **evaluated in order** (sequential processing)
- Each Route **pairs with exactly one Pipeline and one Destination**
- Routes use **filter expressions** to match events

### Two Processing Modes:

#### 1. Final: ON (Default)
- A matching event is **consumed** by that Route's Pipeline and Destination
- **Processing stops** - the event does not continue down the routing table
- Used when an event needs only one specific treatment

#### 2. Final: OFF
- A **clone** of the matching event is sent to the associated Pipeline
- The **original event continues** down the routing table to be checked against later Routes
- Useful when the same events need **multiple treatments and destinations**
- Enables event cloning and cascading workflows

### Key Advantage:
Routes provide intelligent, conditional routing of events to different processing pipelines based on event properties.

---

## Slide 6: Pipelines

### Definition:
A Pipeline is an ordered series of Functions that process events sequentially.

### How Pipelines Execute:
- Events enter at the start of a Pipeline
- Each Function processes the event and **passes its output to the next Function**
- Order matters - the sequence you specify determines execution order

### Three Pipeline Designation Types:
All three share the same editor but differ by placement in the data flow:

#### 1. Pre-processing
- Attached to a **Source**
- Executes **before** the event reaches a Route
- Early data preparation and transformation

#### 2. Processing
- Attached to a **Route** (the main Pipeline)
- Central processing stage in the data flow
- Primary transformation and enrichment

#### 3. Post-processing
- Attached to a **Destination**
- Executes **just before** data delivery
- Final formatting and preparation for the target system

### Pipeline Visualization:
```
Function 1 → Function 2 → Function 3 → Function 4
```

Events flow through each function in sequence, with each function transforming the event for the next stage.

---

## Slide 7: Functions

### Definition:
A Function is a piece of code that executes on an event - the **smallest unit of processing** that can happen to it.

### Function Capabilities:
- Processes individual events
- Can be **filtered** to only act on matching events
- Stackable in a Pipeline for complex processing workflows

### Example Functions:

#### 1. Replace
- Replace the term "foo" with "bar" on each event
- Simple text transformation at the field level

#### 2. Encrypt
- Hash or encrypt the value of a field (e.g., sensitive data)
- Security and data protection transformation

#### 3. Enrich
- Add a field to matching events (e.g., dc=jfk-42)
- Append contextual information to events

### Function Stacking:
```
Function A → Function B → Function C
```

Functions are stacked in a Pipeline and executed sequentially. Each function receives the output of the previous function as input, enabling complex, layered processing workflows.

### Key Point:
Functions are the building blocks of Cribl Edge processing. Complex data transformations are built by combining multiple functions in a Pipeline.

---

## Slide 8: Putting It Together - End-to-End Data Flow

### Complete Architecture Overview:

This final slide synthesizes all the concepts into a comprehensive workflow:

#### 1. **Sources**
- Collect or receive data on the local machine
- Entry point for all observability data

#### 2. **QuickConnect** (Optional)
- Fast path for simple connections
- Wires Sources directly to Destinations
- Bypasses Routes and Pipelines
- Use for straightforward scenarios

#### 3. **Routes**
- Match incoming events with filter expressions
- Send matched events to appropriate Pipeline and Destination
- Evaluated in order for intelligent event routing
- Enable event cloning with Final: OFF

#### 4. **Pipelines**
- Ordered chain of Functions
- Events pass through each Function sequentially
- Can be Pre-processing (at Source), Processing (at Route), or Post-processing (at Destination)

#### 5. **Functions**
- Smallest unit of processing
- Applied individually to events
- Can replace, encrypt, enrich, and perform other transformations
- Combined in Pipelines for complex workflows

### Data Flow Path:
```
Sources → [QuickConnect OR Routes] → Pipelines (Functions) → Destinations
```

### Summary:
Cribl Edge provides a flexible, modular architecture for collecting, processing, and routing observability data. Users can choose between simple QuickConnect configurations for direct flows or complex Route-based architectures with multi-stage Pipelines for sophisticated data processing needs.

---

## Key Takeaways

1. **Sources** are where all data enters the system
2. **QuickConnect** provides a simple drag-and-drop interface for basic connections
3. **Routes** intelligently direct events to appropriate processing paths
4. **Pipelines** organize Functions into processing chains
5. **Functions** are the individual processing units that transform events
6. The entire system is designed for **real-time** data collection, processing, and delivery
7. Architecture supports both **simple** (QuickConnect) and **complex** (Route-based) data flows

---
