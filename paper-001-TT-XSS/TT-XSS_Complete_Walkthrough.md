<div class="justify" style="text-align: justify;">

<style>
div.justify, div.justify h1, div.justify h2, div.justify h3, div.justify p,
div.justify li, div.justify td, div.justify th {
    text-align: justify;
}
</style>

# TT-XSS Complete Walkthrough: From Vulnerable Code to Detection
## The Vulnerable Web Page
Let's start with a realistic vulnerable page that we'll trace through the entire TT-XSS pipeline.
```html
<html>
<head><title>Product Search</title></head>
<body>
  <h2>Search Results</h2>
  <div id="output"></div>

  <script>
    // STEP 1: Read from URL hash (Controllable Source)
    var raw = window.location.hash;

    // STEP 2: Decode it (Transfer Function)
    var decoded = decodeURIComponent(raw);

    // STEP 3: Slice off the '#' character (Transfer Function)
    var query = decoded.slice(1);

    // STEP 4: Build a display string (Transfer Function)
    var message = "You searched for: " + query;

    // STEP 5: Inject into innerHTML (SINK — DANGER!)
    document.getElementById("output").innerHTML = message;
  </script>
</body>
</html>
```
The attacker visits:
```
https://shop.example.com/search.html#<img src=x onerror=alert(1)>
```

## Phase 1 — URL Collection & Crawling
The Crawler Module starts here.
```
Input URL Queue
┌─────────────────────────────────────────────────────────────┐
│  https://shop.example.com/search.html#test                  │
└─────────────────────────────────────────────────────────────┘
          │
          ▼  Static Parsing → scans raw HTML for href, src links
          ▼  Dynamic Parsing → renders JS, finds AJAX-loaded URLs
          ▼  Bloom Filter checks → "Have we seen this URL before? No."
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│  URL confirmed as new → sent to Task Scheduling Module      │
└─────────────────────────────────────────────────────────────┘
```
## Phase 2 — Taint Tracking (Detection Module)
Modified PhantomJS loads the page. JavaScriptCore executes the script. Every instrumented function is now watched.

### Step 2.1 — Source Detected: `window.location.hash`

`window.location.hash` is a Controllable Source (Table 1 in the paper).

**TT-XSS triggers the `initialize` function:**
```
Counter Singleton
├── Current count: 41
└── Generates new ID: 42   ← unique TaintTrace serial number
```

A new `TaintTrace` object is born:
```
TaintTrace #42
├── Trace id    : 42
├── Trace func  : ""          ← empty, nothing executed yet
├── Trace detail: []          ← empty list
└── ExecState   : tainted ✓  ← the string is now "tagged"
``` 

The actual string value at this point:
```
raw = "#<img src=x onerror=alert(1)>"
      ↑ tainted, carrying ID=42
```

### Step 2.2 — Transfer Function 1: `decodeURIComponent()`

`decodeURIComponent` is a Transfer Function (Table 2, function number `03`).

**The tainted string passes through it. TT-XSS records:**
```
TaintTrace #42 updated:
├── Trace func  : "03"
└── Trace detail: [
      "1",                                   ← parameter count
      "#%3Cimg src=x onerror=alert(1)%3E",  ← input parameter value
      "03",                                  ← function number
      "#<img src=x onerror=alert(1)>"        ← resulting string after decode
    ]
```

>The tag travels with the string. decoded is still tainted with ID=42.


### Step 2.3 — Transfer Function 2: `.slice(1)`

`.slice()` is a Transfer Function (Table 2, function number `07`).

```
TaintTrace #42 updated:
├── Trace func  : "03 07"
└── Trace detail: [
      ... previous entries ...,
      "2",                                   ← parameter count (start=1, string itself)
      "1",                                   ← slice argument
      "07",                                  ← function number
      "<img src=x onerror=alert(1)>"         ← resulting string (# is gone)
    ]
```
> `.slice(1)` cut off the `#` character. The `PaddingBlock` strategy accounts for this!


### Step 2.4 — Transfer Function 3: String Concatenation `+`

Concatenation is a Transfer Function (Table 2, function number `0a`).

```javascript
var message = "You searched for: " + query;
//             ← clean string      ← tainted string (ID=42)
```

If any part of a concatenation is tainted, the result is tainted.
```
TaintTrace #42 updated:
├── Trace func  : "03 07 0a"
└── Trace detail: [
      ... previous entries ...,
      "2",                                          ← parameter count
      "You searched for: ",                         ← left operand
      "0a",                                         ← function number
      "You searched for: <img src=x onerror=alert(1)>"  ← resulting string
    ]
```

### Step 2.5 — SINK Detected: innerHTML

`innerHTML` is a Sink Function (Table 3, function number `1c`).

**TT-XSS triggers the `report` function.**
```
TaintTrace #42 FINAL STATE:
├── Trace id    : 42
├── Trace func  : "03 07 0a ff"    ← "ff" appended = SINK REACHED
└── Trace detail: [
      "1",
      "#%3Cimg src=x onerror=alert(1)%3E",
      "03",
      "#<img src=x onerror=alert(1)>",
      ---
      "2",
      "1",
      "07",
      "<img src=x onerror=alert(1)>",
      ---
      "2",
      "You searched for: ",
      "0a",
      "You searched for: <img src=x onerror=alert(1)>",
      ---
      "1c",                                          ← sink function number
      "You searched for: <img src=x onerror=alert(1)>"  ← final value at sink
    ]
```

The complete taint trace is passed to the **Verification Module**.

**Visual Taint Flow Summary**
```
window.location.hash
"#<img src=x onerror=alert(1)>"
         │  TAG: ID=42
         │
         ▼
decodeURIComponent()  [func 03]
"#<img src=x onerror=alert(1)>"
         │  TAG: ID=42
         │
         ▼
.slice(1)  [func 07]
"<img src=x onerror=alert(1)>"    ← # removed
         │  TAG: ID=42
         │
         ▼
"You searched for: " + query  [func 0a]
"You searched for: <img src=x onerror=alert(1)>"
         │  TAG: ID=42
         │
         ▼
innerHTML  [SINK — func 1c] → "ff" appended
         │
         ▼
      🚨 ALERT: Source-to-Sink path confirmed!
      Trace func = "03 07 0a ff"
```
## Phase 3 — Automated Vulnerability Verification

The Verification Module receives `TaintTrace #42` and begins building an attack vector.

### Step 3.1 — Store Detection Report as XML
```xml
<DetectionReport id="42">
  <Source>
    <Function>window.location.hash</Function>
    <Position>anchor</Position>
  </Source>
  <TransferProcess>
    <Step func="03">decodeURIComponent</Step>
    <Step func="07">slice(1)</Step>
    <Step func="0a">concatenation</Step>
  </TransferProcess>
  <Sink>
    <Function>innerHTML</Function>
    <Context>HTML</Context>
  </Sink>
</DetectionReport>
```

### Step 3.2 — Determine Attack Vector Components <br>
The module reads the trace and makes these decisions:
| Question | Answer | Reasoning |
|-----------|---------|-----------|
| Which Payload type? | `<svg/onload=alert(1)>` | Sink is `innerHTML` → HTML context |
| Need PaddingBlock? | Yes, 1 char | `slice(1)` cuts the first character |
| Need ClosingBlock? | No | No open syntax before the injection point |
| Need ExtraBlock? | No | No trailing code after the sink assignment |

### Step 3.3 — Assemble the Attack Vector
```
Vector = [PaddingBlock] [ClosingBlock] Payload [ExtraBlock]

PaddingBlock = "A"                          ← 1 char to be consumed by slice(1)
ClosingBlock = (none)
Payload      = <svg/onload=alert(1)>
ExtraBlock   = (none)

Final Vector = "A<svg/onload=alert(1)>"
```

### Step 3.4 — Determine Injection Position (Table 6) <br>
Source is `window.location.hash` → inject in the URL anchor (`#`).
```
Test URL:
https://shop.example.com/search.html#A<svg/onload=alert(1)>
                                     ↑
                                     # anchor position
```

### Step 3.5 — PhantomJS Executes the Validation Script

```javascript
// Verification Module runs this inside PhantomJS

var page = require('webpage').create();

// Register the alert handler BEFORE loading the page
page.onAlert = function(msg) {
    console.log("[VULNERABILITY CONFIRMED] alert() fired: " + msg);
    // → Store verified report in database
};

page.open(
    "https://shop.example.com/search.html#A<svg/onload=alert(1)>",
    function(status) {
        // Page loaded and JS executed
        // If onAlert fired → vulnerability is real
        // If not → preserve trace for manual review
    }
);
```

### Step 3.6 — Trace the Execution with the Attack Vector <br>
Let's watch what happens when PhantomJS loads the test URL:

```javascript
// URL: search.html#A<svg/onload=alert(1)>

var raw     = window.location.hash;
// raw = "#A<svg/onload=alert(1)>"

var decoded = decodeURIComponent(raw);
// decoded = "#A<svg/onload=alert(1)>"    (already decoded)

var query   = decoded.slice(1);
// query = "A<svg/onload=alert(1)>"
//          ↑ PaddingBlock 'A' was consumed here, payload survives!

var message = "You searched for: " + query;
// message = "You searched for: A<svg/onload=alert(1)>"

document.getElementById("output").innerHTML = message;
// Browser parses the HTML string
// <svg/onload=alert(1)> is valid HTML → onload fires → alert(1) executes!
```
``` 
page.onAlert FIRES ✅

┌──────────────────────────────────────────────────┐
│          VULNERABILITY CONFIRMED                 │
│                                                  │
│  Source  : window.location.hash                  │
│  Sink    : innerHTML                             │
│  Vector  : #A<svg/onload=alert(1)>               │
│  Trace   : 03 07 0a ff                           │
│  Trace ID: 42                                    │
└──────────────────────────────────────────────────┘
```

**Vulnerability Report Stored in Database**
```xml
<VulnerabilityReport>
  <Confirmed>true</Confirmed>
  <TraceID>42</TraceID>
  <URL>https://shop.example.com/search.html</URL>
  <AttackVector>#A&lt;svg/onload=alert(1)&gt;</AttackVector>
  <Source>window.location.hash</Source>
  <Sink>innerHTML</Sink>
  <TraceFunc>03 07 0a ff</TraceFunc>
  <TraceDetail>
    <Step func="03" input="#..." output="#&lt;img...&gt;"/>
    <Step func="07" arg="1"   output="&lt;img...&gt;"/>
    <Step func="0a" left="You searched for: " output="You searched for: &lt;img...&gt;"/>
    <Step func="1c" context="HTML" final="You searched for: A&lt;svg/onload=alert(1)&gt;"/>
  </TraceDetail>
</VulnerabilityReport>
```

**What Does a SAFE Page Look Like?**<br>
Here's the same page with input sanitization:

```javascript
var raw     = window.location.hash;
var decoded = decodeURIComponent(raw);
var query   = decoded.slice(1);

// Sanitize: escape HTML special characters
var safe    = query.replace(/</g, "&lt;").replace(/>/g, "&gt;");

var message = "You searched for: " + safe;
document.getElementById("output").innerHTML = message;
```

**What TT-XSS sees:**
```
window.location.hash → tainted (ID=43)
         │
decodeURIComponent() [03] → still tainted
         │
.slice(1) [07] → still tainted
         │
.replace() [0b] → still tainted (tag follows through replace)
         │
concatenation [0a] → still tainted
         │
innerHTML [1c] → reached sink → "ff" appended
         │
Verification Module injects:
#A<svg/onload=alert(1)>
         │
innerHTML receives:
"You searched for: A&lt;svg/onload=alert(1)&gt;"
         │
Browser renders it as PLAIN TEXT — no HTML tags execute
         │
page.onAlert does NOT fire ✅
```
```
┌──────────────────────────────────────────────────┐
│             NO VULNERABILITY                     │
│                                                  │
│  Taint trace was detected (source → sink)        │
│  But attack vector did NOT execute               │
│  → Taint trace preserved for manual review       │
│  → Page is SAFE against this vector              │
└──────────────────────────────────────────────────┘
```

> Note: TT-XSS still detected a trace here (because tainted data did reach innerHTML), but verification failed — meaning it's either a false positive or requires a more sophisticated bypass. The trace is logged for manual analysis.


## The Two Limitations — Demonstrated Concretely

### Limitation 1 — Second-Order Inputs Cannot Be Automatically Verified<br>
Consider this vulnerable page:

```javascript
// Page A: stores attacker data into a cookie
document.cookie = "username=" + location.search;

// Page B (loaded later): reads from cookie and injects into DOM
var user = document.cookie.split("username=")[1];
document.getElementById("welcome").innerHTML = "Hello, " + user;
```

**Why TT-XSS fails here:**
```
Phase 1 — TT-XSS visits Page B:
├── Source: document.cookie  → tainted (ID=55)
├── Transfer: split(), concat
└── Sink: innerHTML → "ff" appended → trace detected ✅

Phase 2 — Verification Module tries to verify:
├── Source is document.cookie → NOT in the URL
├── Table 6 has no injection rule for cookies
├── Cannot inject attack vector directly into a cookie via URL
└── Cannot automatically set the cookie and then reload Page B
    in a coordinated two-phase interaction
         │
         ▼
❌  AUTOMATIC VERIFICATION FAILS
    Taint trace #55 is preserved for manual review only
```

**The core problem:** Verifying this needs two steps — write to cookie then read from page. The current framework only does one-shot URL injection.
**Future fix:** Multi-phase interaction support — the Verification Module would first submit a request that sets the cookie, then load Page B with the cookie attached.

### Limitation 2 — Complex Payloads Make Attack Vector Construction Slow<br>
Consider this deeply nested transfer chain:

```javascript
var raw = location.hash;                       // Source
var a   = raw.substr(5, 20);                   // slice: needs padding of 5+ chars
var b   = a.replace("search", "");             // replacement: payload must survive
var c   = b.split("&")[0];                     // split: payload must be in first segment
var d   = "Result: " + c + " (page 1 of 10)"; // concat: ExtraBlock must comment out suffix
document.getElementById("out").innerHTML = d; // Sink: HTML context
```

**What TT-XSS must figure out:**
```
Attack Vector must:
1. Survive substr(5, 20)    → PaddingBlock of exactly 5 chars before payload,
                               and payload ≤ 20 chars total
2. Survive replace("search")→ payload must not contain "search"
3. Survive split("&")[0]    → no "&" inside the payload
4. Neutralize " (page 1 of 10)" suffix → ExtraBlock = "//" won't work here
                               because it's inside innerHTML HTML context
                               → need ExtraBlock = "<" to break the HTML

Final Vector attempt:
#AAAAA<svg/onload=alert(1)>
       ↑↑↑↑↑               ↑
       5 padding chars    ExtraBlock breaks trailing HTML
```

**Why this is slow:**
```
The Verification Module must try combinations:

PaddingBlock length: try 1, 2, 3, 4, 5... → needs 5 specifically
ExtraBlock type: try "//", try "<\"", try "<" → needs "<"
Payload encoding: try raw, try encoded → raw works here

Each failed attempt = another PhantomJS page load + wait for JS execution
Many layers = exponential combinations to try
No intelligence in the search = brute force

For just 3 variables with 5 options each:
5 × 3 × 2 = 30 combinations to try
Each = ~2 seconds in PhantomJS
Total = ~60 seconds for one URL
```

**Future fix:** Use **heuristic search** — genetic algorithms or guided fuzzing that learns from previous successful vectors and skips unlikely combinations, reducing search time dramatically.

## Complete Pipeline at a Glance
```
https://shop.example.com/search.html#<img src=x onerror=alert(1)>
         │
         ▼
┌─────────────────────┐
│   Crawler Module    │  Bloom Filter: new URL → add to queue
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Task Scheduling    │  Assign to Detection Module
└────────┬────────────┘
         │
         ▼
┌────────────────────────────────────────────────────┐
│              Detection Module                      │
│                                                    │
│  Modified PhantomJS loads page                     │
│  JavaScriptCore executes script                    │
│                                                    │
│  Source: window.location.hash → TaintTrace #42     │
│  Transfer: decodeURIComponent [03]                 │
│  Transfer: .slice(1) [07]                          │
│  Transfer: concatenation [0a]                      │
│  Sink: innerHTML [1c] → "ff" → report()            │
└────────┬───────────────────────────────────────────┘
         │  TaintTrace #42 passed forward
         ▼
┌────────────────────────────────────────────────────┐
│            Verification Module                     │
│                                                    │
│  Parse XML report                                  │
│  Sink context: HTML → Payload: <svg/onload=...>    │
│  slice(1) detected → PaddingBlock: "A"             │
│  Source: location.hash → inject in anchor #        │
│                                                    │
│  Test URL: search.html#A<svg/onload=alert(1)>      │
│  PhantomJS loads URL + registers page.onAlert      │
│  alert() fires ✅                                   │
└────────┬───────────────────────────────────────────┘
         │
         ▼
┌─────────────────────┐
│  Data Persistence   │  Store verified vulnerability report
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│     UI Module       │  Display result to security analyst
└─────────────────────┘

RESULT: DOM-XSS CONFIRMED at innerHTML via window.location.hash
ATTACK VECTOR: #A<svg/onload=alert(1)>
```

</div>