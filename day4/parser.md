# Cribl Stream Pipeline Lab - Detailed Slide-by-Slide Explanation

---

## Slide 1: Title Slide

**Title:** Cribl Stream Pipeline Lab

**Subtitle:** Hands-on walkthrough using live httpd access-log data

### Core Functions Covered:
- **Eval** - Derive and enrich fields with expressions
- **Parser** - Extract structured fields from raw text
- **Mask** - Protect sensitive data
- **Drop** - Filter out noise and low-value events
- **Sampling** - Thin out routine traffic
- **Aggregation** - Roll up events into metrics

### Presentation Type:
This is a **hands-on, practical lab** that will demonstrate how to build a complete pipeline using real Apache/httpd access log data. Each function will be implemented sequentially in a single pipeline, showing how they work together in practice.

---

## Slide 2: Objective & Pipeline Flow

### Overall Objective:
Build a single Cribl pipeline that:
1. Ingests raw Apache/httpd logs
2. Applies each core function in sequence
3. Transforms noisy raw text into clean, filtered, and summarized events
4. Produces actionable, structured data

### Complete Pipeline Architecture:
```
Source 
  ↓
Parser 
  ↓
Eval 
  ↓
Mask 
  ↓
Drop 
  ↓
Sampling 
  ↓
Aggregation 
  ↓
Destination
```

### Execution Model:
- **Functions execute top to bottom** in exactly the order shown above
- Each stage receives the **output of the one before it**
- Events flow sequentially through the pipeline
- Processing is deterministic - same input always produces same output

### Lab Setup Instructions:

**Navigate to:**
```
Cribl UI → Processing → Pipelines → Add Pipeline
```

**Pipeline Configuration:**
- **Name:** `httpd_access_lab`
- **Route match:** 
  ```
  source == "/var/log/httpd/access_log" OR "/var/log/httpd/error_log"
  ```
  This route matches events coming from either Apache access logs or error logs

### Key Concept:
The pipeline is configured to automatically process all httpd log entries that match the source path, demonstrating both access and error log handling in a single pipeline.

---

## Slide 3: Sample Data Used Throughout This Lab

### Understanding Event Structure:

#### Raw Event Data (_raw field)
```
::1 - - [01/Sep/2026:23:03:34 +0530] "GET / HTTP/1.1" 403 7620 "-" "curl/7.76.1"
```

**This is the complete, unstructured Apache access log line that will be parsed.**

#### Event Metadata (Already Present Before Parsing)

| Field | Value | Description |
|-------|-------|-------------|
| `_time` | 1788284014 | Unix timestamp; represents 2026-09-01 23:03:34 +0530 (India Standard Time) |
| `host` | localhost.localdomain | The machine where the log originated |
| `source` | /var/log/httpd/access_log | Path to the source log file; used in routing decisions |
| `cribl_breaker` | fallback | Indicates event break detection method used |

### Why This Matters:
- **_time:** Required for time-series analysis and correlation with other events
- **host:** Identifies the server in multi-server environments
- **source:** Enables source-based routing decisions in the pipeline
- **cribl_breaker:** Indicates whether Cribl automatically detected event boundaries

### Sample Event Context:
The sample line represents:
- **Client IP:** ::1 (localhost IPv6)
- **Request Method:** GET
- **Requested URL:** / (root)
- **HTTP Version:** HTTP/1.1
- **Status Code:** 403 (Forbidden)
- **Response Size:** 7620 bytes
- **User Agent:** curl/7.76.1 (command-line HTTP client, indicating automated traffic)

This single event will be used as the test case throughout the entire lab to validate each function's behavior.

---

## Slide 4: Step 1 — Parser: Extract Structured Fields

### Function Type: Regex Extract

### Configuration Details:

| Parameter | Value |
|-----------|-------|
| **Type** | Regex Extract |
| **Source field** | _raw |
| **Overwrite fields** | true |

### Regular Expression (Regex):
```regex
^(?<clientip>\S+) \S+ \S+ \[(?<timestamp>[^\]]+)\] "(?<method>\S+) (?<url>\S+) (?<httpversion>[^\"]+)" (?<status>\d+) (?<bytes>\d+) "(?<referrer>[^\"]*)" "(?<useragent>[^\"]*)\"$
```

### Regex Breakdown:

| Component | Regex | Captures | Purpose |
|-----------|-------|----------|---------|
| Start anchor | `^` | - | Match beginning of line |
| Client IP | `(?<clientip>\S+)` | `::1` | IPv4 or IPv6 address; `\S+` = one or more non-whitespace characters |
| Skip 2 fields | `\S+ \S+` | - | Skip username and ident (typically "-" in logs) |
| Timestamp | `(?<timestamp>[^\]]+)` | `01/Sep/2026:23:03:34 +0530` | Everything between `[` and `]`; `[^\]]+` = anything except `]` |
| HTTP Method | `(?<method>\S+)` | `GET` | HTTP verb (GET, POST, PUT, DELETE, etc.) |
| URL Path | `(?<url>\S+)` | `/` | Requested resource path |
| HTTP Version | `(?<httpversion>[^\"]+)` | `HTTP/1.1` | Protocol version; `[^\"]+` = anything except quotes |
| Status Code | `(?<status>\d+)` | `403` | HTTP response code; `\d+` = one or more digits |
| Bytes | `(?<bytes>\d+)` | `7620` | Response size in bytes |
| Referrer | `(?<referrer>[^\"]*)` | `-` | HTTP Referer header (blank/dash if none) |
| User Agent | `(?<useragent>[^\"]*)` | `curl/7.76.1` | Browser/client identification string |
| End anchor | `$` | - | Match end of line |

### Extracted Fields (8 total):

```
clientip:     ::1
timestamp:    01/Sep/2026:23:03:34 +0530
method:       GET
url:          /
httpversion:  HTTP/1.1
status:       403
bytes:        7620
referrer:     -
useragent:    curl/7.76.1
```

### Validation Test:
**Action:** Paste the sample _raw line into the Parser preview and confirm **all 8 fields populate correctly**.

**Expected Result:** Each named capture group should extract its corresponding value from the raw log line.

### Why This Step Matters:
- Transforms unstructured text into structured, queryable fields
- Enables downstream functions (Eval, Mask, Drop) to work with specific values
- Foundation for all subsequent processing steps

---

## Slide 5: Step 2 — Eval: Derive and Enrich Fields

### Purpose:
Use **JavaScript expressions** to create new fields, calculate values, and enrich events with derived data.

### Derived Fields Table:

| Field Name | Expression | Sample Result | Purpose |
|------------|-----------|----------------|---------|
| `status_num` | `Number(status)` | `403` | Convert status code string to numeric value for math operations |
| `is_error` | `status_num >= 400` | `true` | Boolean flag: true if response is 4xx or 5xx error |
| `status_class` | `'4xx' / '5xx' / '3xx' / '2xx' by range` | `'4xx'` | Categorize status code into HTTP status class |
| `bytes_kb` | `Math.round(Number(bytes)/1024)` | `7` | Convert bytes to kilobytes (7620 bytes ÷ 1024 ≈ 7.4 KB, rounded to 7) |
| `is_bot` | `/curl\|wget\|bot\|spider/i.test(useragent)` | `true` | Boolean flag: true if user agent contains bot/crawler indicators (curl, wget, bot, spider) |
| `log_type` | `source.includes('error_log') ? 'error':'access'` | `'access'` | Classify event as 'error' or 'access' based on source file path |

### Detailed Field Explanations:

#### 1. **status_num**
```javascript
Number(status)
```
- **Why:** The Parser extracts status as a string "403"
- **Function:** Converts to numeric type so math operations work
- **Used by:** is_error calculation and Drop conditions

#### 2. **is_error**
```javascript
status_num >= 400
```
- **Why:** Identify problematic requests (client errors: 4xx, server errors: 5xx)
- **Result:** Boolean true/false
- **Used for:** Error aggregation and sampling rules

#### 3. **status_class**
```javascript
'4xx' / '5xx' / '3xx' / '2xx' by range
```
- **Logic:** 
  - 400-499 → '4xx'
  - 500-599 → '5xx'
  - 300-399 → '3xx'
  - 200-299 → '2xx'
- **Sample:** 403 falls in 400-499 range → '4xx'
- **Used by:** Aggregation grouping

#### 4. **bytes_kb**
```javascript
Math.round(Number(bytes)/1024)
```
- **Calculation:** 7620 bytes ÷ 1024 = 7.41... KB → rounded to 7
- **Why:** More readable size metric for analysis
- **Used for:** Drop conditions (e.g., drop tiny responses < 1 KB)

#### 5. **is_bot**
```javascript
/curl|wget|bot|spider/i.test(useragent)
```
- **Pattern:** Case-insensitive regex matching
- **Matches:** curl, wget, bot, spider (or any substring containing these)
- **Sample:** curl/7.76.1 contains "curl" → true
- **Use:** Filter out automated health checks and crawlers

#### 6. **log_type**
```javascript
source.includes('error_log') ? 'error':'access'
```
- **Logic:** Ternary conditional
- **Sample:** source = "/var/log/httpd/access_log" (includes 'error_log'? NO) → 'access'
- **Purpose:** Distinguish between two log file types in a single pipeline

### Integration Point:
These derived fields enable intelligent downstream decisions:
- **Mask** can use `is_bot` and `is_error` to decide what to mask
- **Drop** uses `is_bot`, `status_num`, `status_class` to filter events
- **Aggregation** groups by `status_class` and sums/counts using `is_error`

### Validation:
**In Eval preview:** Confirm output shows:
- `status_class = '4xx'`
- `is_bot = true`
- `status_num = 403`
- `is_error = true`

---

## Slide 6: Step 3 — Mask: Protect Sensitive Data

### Purpose:
Redact or obscure sensitive information before forwarding data to storage or destinations.

### Mask Rules (Regex-based):

#### Rule 1: Mask IPv4 (Last Octet)
```
Field:   clientip
Regex:   (\d{1,3}\.\d{1,3}\.\d{1,3}\.)\d{1,3}
Replace: $1***
```

**Example:**
- Input:  `192.168.1.42`
- Regex captures: `192.168.1.` (group 1)
- Output: `192.168.1.***`

**Explanation:**
- `(\d{1,3}\.\d{1,3}\.\d{1,3}\.)` → Captures first 3 octets + dot (group $1)
- `\d{1,3}` → Matches 4th octet (not captured, will be replaced)
- `$1***` → Replace with captured groups + asterisks

---

#### Rule 2: Mask IPv6 Loopback
```
Field:   clientip
Regex:   ^::1$
Replace: ::MASKED
```

**Applied to sample event:**
- Input:  `::1` (IPv6 localhost)
- Output: `::MASKED`

**Explanation:**
- `^::1$` → Match exactly "::1" (full address match)
- Simple exact replacement with `::MASKED`

---

#### Rule 3: Redact Query Secrets
```
Field:   url
Regex:   ([?&](?:token|apikey|password)=)[^&\s]+
Replace: $1***REDACTED***
```

**Example:**
- Input:  `/api/search?token=abc123xyz&page=1`
- Output: `/api/search?token=***REDACTED***&page=1`

**Explanation:**
- `([?&](?:token|apikey|password)=)` → Captures parameter name up to `=` (group $1)
  - `[?&]` → Must be preceded by `?` or `&`
  - `(?:token|apikey|password)` → Match exactly these parameter names (non-capturing group)
  - `=` → Literal equals sign
- `[^&\s]+` → Match everything after `=` until `&` or whitespace (the secret value)
- `$1***REDACTED***` → Keep parameter name, replace value with `***REDACTED***`

---

### Applied to Sample Event:

**Before Masking:**
```
clientip: ::1
url:      /
```

**After Masking:**
```
clientip: ::MASKED
url:      /
```

**Why:** 
- IPv6 loopback (::1) is treated as sensitive
- Sample URL "/" has no secrets to redact

### Validation:
**In Mask preview:** Confirm that:
- `clientip` changed from `::1` to `::MASKED`
- `url` remains unchanged (no secrets to redact in this sample)

### Real-World Importance:
Masking ensures:
- **Compliance:** GDPR, HIPAA, PCI-DSS requirements for data protection
- **Security:** Prevents leakage of API keys, tokens, passwords in logs
- **Privacy:** Anonymizes user/client identifiers
- **Storage:** Reduces sensitive data footprint in centralized logging

---

## Slide 7: Step 4 — Drop: Filter Out Noise

### Purpose:
Discard low-value events **before** they consume downstream storage or licensing costs.

### Drop Condition (Primary):
```
is_bot === true && status_num === 403
```

**Meaning:** DROP the event if:
- The user agent is detected as a bot (curl, wget, bot, spider)
- **AND** the response status is 403 (Forbidden)

### Rationale:
Automated/bot health checks that consistently get blocked (403) add **no analytic value**:
- Bots making requests to protected endpoints
- Expected to fail (403 = Access Denied)
- Adds noise without useful insights
- Wastes storage/licensing costs

### Applied to Sample Event:
```
is_bot      = true (useragent contains "curl")
status_num  = 403 (parsed from status field)
Condition   = true && true = TRUE
Result:     EVENT DROPPED ✓
```

### Additional Conditions Worth Trying:

#### Condition 1: Filter Favicon Requests
```javascript
log_type === 'access' && url === '/favicon.ico'
```
- **Why:** Favicon.ico requests are ubiquitous, rarely useful for analysis
- **Impact:** Removes thousands of repetitive 404 errors
- **Applied to:** Access logs only (not error logs)

#### Condition 2: Filter Tiny Successful Responses
```javascript
status_class === '2xx' && bytes_kb < 1
```
- **Why:** Successful responses under 1 KB are often empty or health-check responses
- **Impact:** Removes low-value success logs
- **Use case:** When focus is on meaningful user activity or errors

### Drop Logic Flow:
```
Every event → Evaluate condition
              ↓
              TRUE? → DROP (event removed, never reaches destination)
              ↓
              FALSE? → PASS (event continues to next pipeline function)
```

### Important Notes:
- Drop should be **after Eval** (uses derived fields)
- Drop should be **before Sampling** (doesn't need to thin events you'll drop anyway)
- Multiple Drop functions can be chained for complex filter logic

### Validation:
**In Drop preview:** Confirm the sample event is marked:
```
"will be dropped"
```

### Cost Impact Example:
Without Drop:
- 1 million requests/day
- 70% are bot 403s (700k dropped events)
- Cost: Pay for 1M ingestion

With Drop:
- Same 1 million requests/day
- Drop the 700k bot 403s at the edge
- Cost: Pay for 300k ingestion (70% cost savings)

---

## Slide 8: Step 5 & 6 — Sampling and Aggregation

### Overview:
Two complementary functions that **thin and summarize** high-volume routine traffic while preserving all error events.

---

### Sampling — Thin Out Routine Traffic

**Purpose:** Reduce volume of routine/routine (2xx) traffic while keeping 100% of errors

**Configuration:**

**Rule 1: Error Preservation**
```
Filter:     [No filter]
Rate:       100% (1 in 1)
Condition:  Keep all events
Result:     status_class === '4xx' or '5xx' → 100% retained
```

**Rule 2: Routine Traffic Sampling**
```
Filter:     status_class === '2xx'
Rate:       10% (1 in 10)
Meaning:    Of every 10 successful (2xx) requests, keep 1; discard 9
```

**Logic:**
```
For each event:
  If status_class is '4xx' or '5xx'?
    → Keep (100% of errors)
  Else if status_class is '2xx'?
    → Keep 1 in 10 (drop 9 out of 10)
```

**Example Volume Reduction:**
```
Input:  1000 events in 30s window
        - 800 are 2xx (success)
        - 200 are 4xx/5xx (errors)

Output: ~280 events
        - 200 errors (100%)
        - ~80 of 800 successes (10%)
        
Reduction: 72% fewer events forwarded
```

### Aggregation — Roll Up Into Metrics

**Purpose:** Group similar events and compute aggregate statistics

**Configuration:**

| Setting | Value |
|---------|-------|
| **Group by** | status_class, host |
| **Window** | 30s (lab) / 5min (production) |

**Meaning:** Every 30 seconds, create a summary event for each unique combination of:
- status_class (2xx, 3xx, 4xx, 5xx)
- host (e.g., localhost.localdomain, app-server-01)

### Aggregation Metrics:

| Metric | Expression | Meaning |
|--------|-----------|---------|
| `request_count` | `count()` | Total number of events in this group during the window |
| `error_count` | `count(is_error === true)` | Number of events where is_error flag is true |
| `total_bytes` | `sum(Number(bytes))` | Sum of all response sizes in this group |
| `avg_bytes` | `avg(Number(bytes))` | Average response size in this group |

### Example Output (Per Window):
```json
{
  "status_class": "4xx",
  "host": "localhost.localdomain",
  "request_count": 12,
  "error_count": 12,
  "total_bytes": 91440,
  "avg_bytes": 7620
}
```

**Interpretation:**
- In this 30-second window:
  - 12 requests with 4xx status codes
  - All 12 were errors (is_error = true)
  - Total response size: 91,440 bytes (≈89 KB)
  - Average response per request: 7,620 bytes (≈7.4 KB)

---

### Critical Ordering: Sampling → Aggregation

**IMPORTANT:** Place **Sampling BEFORE Aggregation**

**Why?**
```
Without Sampling (Wrong):
  Input: 1000 2xx events → Aggregation counts all 1000

With Sampling (Correct):
  Input: 1000 2xx events → Sampling keeps 100 → Aggregation counts 100
  (Your metrics reflect the actual thinned volume you forward)
```

If Aggregation comes first, your metrics will be inflated because they count events before thinning.

---

### Data Flow Example:

```
Raw Apache Log
    ↓
[Parser] Extract fields from _raw
    ↓
[Eval] Add is_error, status_class, is_bot
    ↓
[Mask] Redact IPs and secrets
    ↓
[Drop] Remove bot 403s (is_bot && status===403)
    ↓
[Sampling] Keep 100% errors, 10% of 2xx
    ↓
[Aggregation] 
    Group: status_class, host
    Window: 30s
    Metrics: request_count, error_count, total_bytes, avg_bytes
    ↓
Summary Event to Destination
```

---

### Validation:
In preview, confirm:
- **Sampling:** ~90% of 2xx test events removed (only 10% pass through)
- **Aggregation:** One summary event per group per window
  - Example: One event for (4xx, localhost.localdomain)
  - Example: One event for (2xx, localhost.localdomain)

---

## Slide 9: Final Pipeline Order & Validation Checklist

### Complete Pipeline Architecture:

```
1. Parser
    ↓
2. Eval
    ↓
3. Mask
    ↓
4. Drop
    ↓
5. Sampling
    ↓
6. Aggregation
```

### Why This Specific Order Matters:

| Step | Why It's Here | Dependencies |
|------|---------------|--------------|
| **Parser** | First - must extract raw data into fields | None (needs raw data) |
| **Eval** | Second - creates derived fields needed downstream | Requires Parser results |
| **Mask** | Third - protect data before filtering decisions | Uses eval results |
| **Drop** | Fourth - filters using eval fields before expensive aggregation | Requires is_bot, status_num, status_class |
| **Sampling** | Fifth - thin volume before aggregation | Operates on filtered stream |
| **Aggregation** | Last - creates metrics from thinned data | Must be after Sampling for accurate counts |

---

### Comprehensive Validation Checklist

#### ✓ Check 1: Parser Preview
```
Test: Paste the sample _raw line into Parser preview
Expected: All 8 fields populate
- clientip: ::1
- timestamp: 01/Sep/2026:23:03:34 +0530
- method: GET
- url: /
- httpversion: HTTP/1.1
- status: 403
- bytes: 7620
- useragent: curl/7.76.1
```

**Pass/Fail:** If any field is empty or has wrong value, regex is incorrect.

---

#### ✓ Check 2: Eval Preview
```
Test: Run sample through Parser + Eval
Expected: Derived fields appear correctly
- status_class: '4xx' (status 403 falls in 400-499 range)
- is_bot: true (useragent contains "curl")
- is_error: true (status 403 >= 400)
- bytes_kb: 7 (7620 / 1024 ≈ 7)
- status_num: 403 (string "403" converted to number)
- log_type: 'access' (source path is access_log, not error_log)
```

**Pass/Fail:** Each expression must evaluate correctly.

---

#### ✓ Check 3: Mask Preview
```
Test: Run sample through Parser + Eval + Mask
Expected: Sensitive fields masked
- clientip: ::MASKED (was ::1)
- url: / (no secrets to redact)
- All other fields unchanged
```

**Pass/Fail:** IPv6 loopback must be replaced with ::MASKED.

---

#### ✓ Check 4: Drop Preview
```
Test: Run sample through Parser + Eval + Mask + Drop
Expected: Event marked as dropped
Status: "will be dropped"
Reason: is_bot === true && status_num === 403 (both conditions met)
```

**Pass/Fail:** Sample event must be marked for drop.

---

#### ✓ Check 5: Sampling Preview
```
Test: Run 100 similar 2xx test events through Sampling
Expected: ~10 events pass (10% rate)
         ~90 events dropped

Test: Run 10 4xx/5xx test events through Sampling
Expected: All 10 events pass (100% rate)
```

**Pass/Fail:** Sampling reduction ratio must be approximately correct.

---

#### ✓ Check 6: Aggregation Preview
```
Test: Run multiple events through full pipeline
Expected: One summary event per unique (status_class, host) pair per 30s window

Example output:
{
  "status_class": "4xx",
  "host": "localhost.localdomain",
  "request_count": 12,
  "error_count": 12,
  "total_bytes": 91440,
  "avg_bytes": 7620
}
```

**Pass/Fail:** Correct grouping and metric calculation.

---

### Troubleshooting Guide:

| Issue | Check |
|-------|-------|
| Parser fields missing | Regex doesn't match log format - test with actual log line |
| Eval shows null/undefined | Expression has wrong field name - verify Parser output |
| Mask not working | Regex pattern doesn't match - test regex separately |
| Drop not filtering | Condition logic is wrong - verify boolean operators |
| Sampling rate off | Filter condition doesn't match expected events |
| Aggregation shows no output | Group-by fields may be missing from earlier steps |

---

## Slide 10: Summary

### From Raw Text to Clean Signal

**Six functions, one pipeline:**
```
Parse → Enrich → Protect → Filter → Thin → Summarize
```

### What Each Function Accomplishes:

| Function | Input | Output | Benefit |
|----------|-------|--------|---------|
| **Parse** | Raw text log line | Structured fields | Queryable, analyzable data |
| **Eval** | Structured fields | Derived fields + classifications | Intelligence for downstream decisions |
| **Mask** | Original values | Redacted values | Security, compliance, privacy |
| **Drop** | All events | Filtered events | Cost reduction, noise elimination |
| **Sampling** | All remaining events | 90% reduction (2xx), 100% errors | Volume thinning for high-traffic scenarios |
| **Aggregation** | Individual events | Summary metrics | Efficient storage, actionable insights |

### Complete Transformation:

**Before Pipeline:**
```
::1 - - [01/Sep/2026:23:03:34 +0530] "GET / HTTP/1.1" 403 7620 "-" "curl/7.76.1"
```

**After Pipeline (Dropped):**
```
[Event removed - bot 403]
(or if this wasn't dropped)

{
  clientip: "::MASKED",
  timestamp: "01/Sep/2026:23:03:34 +0530",
  method: "GET",
  url: "/",
  status_class: "4xx",
  is_bot: true,
  is_error: true,
  ... (summarized in 30s window)
}
```

---

## Key Takeaways

### 1. **Order Matters**
   - Parser → Eval → Mask → Drop → Sampling → Aggregation
   - Each function depends on previous outputs
   - Wrong order breaks functionality or introduces bugs

### 2. **Incremental Testing**
   - Test each function independently using preview
   - Add one function at a time
   - Validate output before moving to next step

### 3. **Real-World Applications**
   - **Security:** Masking prevents credential leakage
   - **Cost:** Dropping and sampling reduce storage/license costs
   - **Intelligence:** Eval enriches data for better analysis
   - **Efficiency:** Aggregation produces actionable metrics

### 4. **Parse Once, Use Many Times**
   - Parser extracts fields once
   - All downstream functions reuse those fields
   - Efficiency: One parsing pass, multiple processing stages

### 5. **Trade-offs**
   - **Sampling:** Lose some data detail, gain volume reduction
   - **Masking:** Lose some field values, gain compliance/security
   - **Drop:** Lose some events, reduce noise/cost
   - **Aggregation:** Lose individual events, gain summary metrics

### 6. **Validation is Essential**
   - Preview each function step-by-step
   - Use the sample event throughout
   - Document expected output at each stage

---

## Lab Completion Checklist

- [ ] Pipeline created with name `httpd_access_lab`
- [ ] Route configured for `/var/log/httpd/access_log` and `/var/log/httpd/error_log`
- [ ] Parser function added with correct regex (8 fields extracted)
- [ ] Eval function added with 6 derived fields (status_num, is_error, status_class, bytes_kb, is_bot, log_type)
- [ ] Mask function added with 3 rules (IPv4, IPv6, URL secrets)
- [ ] Drop function added with condition: `is_bot === true && status_num === 403`
- [ ] Sampling function added with rules (100% errors, 10% 2xx)
- [ ] Aggregation function added with group-by (status_class, host) and 4 metrics
- [ ] All 6 validation checks passed
- [ ] Pipeline tested with real httpd access logs
- [ ] Events successfully flowing through all 6 functions

---

## Advanced Exploration

### Extend the Lab:

1. **Add more Drop conditions:**
   - Filter favicon.ico requests
   - Remove health check endpoints
   - Drop events under 1 KB

2. **Enhance Eval expressions:**
   - Calculate request processing time
   - Classify user agents (browser vs mobile vs bot)
   - Extract HTTP status descriptions

3. **Modify Sampling:**
   - Different rates for different status classes
   - Time-based sampling (more aggressive at peak hours)
   - Keep 100% of slow requests (high latency)

4. **Add post-processing:**
   - Connect to Aggregation output
   - Format metrics for specific destination system
   - Add timestamp and metadata

5. **Implement error-specific pipeline:**
   - Create separate route for error_log source
   - More aggressive enrichment for errors
   - Different aggregation window for error metrics

---

## Conclusion

This lab demonstrates how **Cribl Stream pipelines** transform raw, noisy log data into clean, secure, and actionable insights. By combining six core functions in the right order, you can:

- **Extract** structured data from unstructured text
- **Enrich** events with intelligence and derived fields
- **Secure** sensitive information through masking
- **Filter** low-value noise at the edge
- **Optimize** volume through sampling
- **Summarize** high-volume data into metrics

This pipeline pattern is applicable to any log source: Apache, Nginx, application logs, cloud services, security events, and more.
