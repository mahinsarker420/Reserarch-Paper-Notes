<div class="justify" style="text-align: justify;">

<style>
div.justify, div.justify h1, div.justify h2, div.justify h3, div.justify p,
div.justify li, div.justify td, div.justify th {
    text-align: justify;
}
</style>

# TT-XSS: A Novel Taint-Tracking Dynamic Detection Framework for DOM XSS

- **Authors:** Ran Wang, Guangquan Xu, Xianjiao Zeng, Xiaohong Li, Zhiyong Feng
- **DOI / Link:** https://doi.org/10.1016/j.jpdc.2017.07.006

## How I prepare these notes and how to read them
I first record unfamiliar terminology to reference later. Then I summarize each section and provide clear explanations of the methodology and other key parts of the paper. If you are already familiar with the terminology, you can skip the "Terminology Notes" section and start at "Paper Notes".

## Contents

- [Terminology Notes](#terminology-notes)
    - [What is DOM-Based XSS?](#1-what-is-dom-based-xss)
    - [What is Taint Tracking detection?](#2-what-is-taint-tracking-detection)
    - [Why the term Dynamic?](#3-why-the-term-dynamic)
    - [What is Obfuscated code?](#4-what-is-obfuscated-code)
- [Paper Notes](#paper-notes)
    - [Abstract (Summary)](#0-absract-summery)
    - [Introduction (Summary)](#1-introduction-summary)
    - [Related Work](#2-related-work)
    - [Detection framework for DOM-XSS](#3-detection-framework-for-dom-xss)
        - [URL Information Collection and Analysis Module](#31-url-information-collection-and-analysis-module)
        - [Taint tracking analysis](#32-taint-tracking-analysis)
            - [Analysis of related DOM APIs](#321-analysis-of-related-dom-apis)




# Terminology Notes

## 1. What is DOM-Based XSS?
See: [DOM-Based XSS](https://github.com/mahinsarker420/Cybersecurity-Notes/blob/75d2c5872991ef71b63553e1e838653db61af712/Cross-Site%20Scripting%20(XSS)/DOM-Based-XSS.md)

## 2. What is Taint Tracking detection?
Taint Tracking Based Dynamic Detection is a runtime security technique used to detect whether untrusted input (called tainted data) reaches sensitive parts of a program in a dangerous way. Think of it like placing invisible colored ink on user-controlled data and then watching where that ink spreads through the application while it runs.

### Example:

Suppose a website has a login form.
```
User input:
username = ' OR 1=1 --

↓ (mark as tainted)

Application processes data

↓

SQL Query:
SELECT * FROM users WHERE username=' ' OR 1=1 -- '

↓

Sensitive sink reached
(Database query execution)

↓

Alert: Possible SQL Injection
```

The flow works like this:
```
Source → Propagation → Sink → Detection
```
- Source = where external data enters the application (User input, URL parameters, Cookies, Network packets, File input)
- Propagation = how the data moves inside the program (Assignments, Variables, Function calls, String operations)
Sink = dangerous operations (SQL execution, System command execution, HTML rendering, File access)

If tainted data reaches a sensitive sink without proper sanitization, the system raises a warning.

### Example with XSS:
```JavaScript
var input = location.hash;      // Source (tainted)
document.innerHTML = input;     // Sink
```
Tracking process:
```
location.hash
      ↓
marked tainted
      ↓
stored in variable input
      ↓
passed into innerHTML
      ↓
possible XSS detected
```
## 3. Why the term Dynamic?
Because the monitoring happens while the program is actually executing, not by only reading source code.

### Limitations:
- Performance overhead can be high because every data movement may be tracked
- Only sees executed code paths
- Complex applications may create taint explosion (too many tracked variables)
- Attackers sometimes bypass simplistic taint rules

## 4. What is Obfuscated code?
Obfuscated code is code that has been intentionally modified to make it difficult for humans (and sometimes security tools) to understand, while still keeping the same functionality. Developers may use it to protect intellectual property, but attackers also use it to hide malicious behavior.

### Simple example:

Normal JavaScript:
```JavaScript
alert("Hello");
```
Obfuscated version:
```JavaScript
eval(String.fromCharCode(97,108,101,114,116,40,34,72,101,108,108,111,34,41))
```
What happens:
```
97 → a
108 → l
101 → e
114 → r
116 → t
```
After conversion, it becomes:
```JavaScript
alert("Hello")
```
Another XSS-related example:

Normal code:
```JavaScript
document.write("<script>alert('XSS')</script>");
```
Obfuscated code:
```JavaScript
document.write("\x3Cscript\x3Ealert('XSS')\x3C/script\x3E");
```
Here:
```
\x3C = <
\x3E = >
```
So the browser still executes:
```JavaScript
<script>alert('XSS')</script>
```
This is why obfuscated code creates problems for static analysis tools: the tool may struggle to understand the real behavior because the code is hidden or transformed.


# Paper Notes:

## 0 Absract (Summery):

DOM-Based XSS detection method can be divided into three types : black-box fuzzing, static analysis, and dynamic analysis.
- **Black-box fuzzing**: A testing method where random, malformed, or unexpected inputs are sent to an application without knowing its internal code, to discover crashes or vulnerabilities.
- **Static analysis**: A technique that examines source code or program files without running the application, looking for coding errors and potential security weaknesses.
- **Dynamic analysis**: A method that analyzes an application while it is actually running, monitoring its behavior and data flow to detect vulnerabilities during execution.

Black-box fuzzing and static analysis have problems because fuzzing can miss many vulnerabilities (high false negatives), while static analysis may incorrectly report safe code as vulnerable (high false positives). Dynamic analysis usually gives better results because it analyzes the application while it is running, but it is often difficult and expensive to implement. To solve these issues, the authors created a new framework called TT-XSS that uses taint tracking on the client side (inside the browser). Their system modifies JavaScript features and browser DOM APIs so it can track how untrusted data moves through the web page. It marks input sources, follows how the data is transferred, and checks whether it reaches dangerous places (sinks). Using this information, the framework can automatically create attack inputs to verify whether a real vulnerability exists. When compared with another security tool, AWVS 10.0, their framework found 1.8% more vulnerabilities and automatically generated attack payloads for 9.1% of vulnerabilities.

## 1 Introduction (Summary):
This paragraph explains that Cross-Site Scripting (XSS) is one of the major security problems in
modern web applications because it can cause serious issues such as stealing user information, violating privacy, and even spreading malicious worms. XSS is considered an important security risk and is often listed among the top web security threats. XSS vulnerabilities are generally divided into three types: Reflected XSS, Stored XSS, and DOM-XSS. **DOM-XSS is different from the other two because its attack process happens inside the browser using JavaScript and the Document Object Model (DOM), so methods used for detecting Reflected and Stored XSS do not work well for it**. As web applications continue to grow and use more client-side JavaScript, DOM-XSS attacks are becoming more common, creating a need for better detection methods. Existing approaches mainly use black-box fuzzing and static analysis, but both have limitations: fuzzing may miss vulnerabilities because it cannot test every possible case, while static analysis struggles with problems like unclear code structure and obfuscated code. Dynamic analysis gives more accurate results but is difficult and expensive to implement. To solve this issue, the authors propose a new dynamic detection framework based on taint tracking, where they introduce new data types and methods to track data flow in the browser, automatically generate attack payloads to verify vulnerabilities, and build a prototype by modifying browser components such as JavaScriptCore and WebKit. The rest of the paper explains related work, the framework design, implementation details, experimental results, and future work.

## 2 Related Work:
This section explains previous research related to DOM-XSS detection. As modern web applications have evolved, many functions that were previously handled by servers are now processed on the client side using JavaScript. With the growth of JavaScript and support for HTML5 features in browsers, web applications have become more powerful but also introduced more security risks. DOM-XSS is a purely client-side vulnerability that happens inside the browser during DOM processing and does not require direct server involvement. Detecting DOM-XSS is challenging because client-side code runs on users’ devices rather than under server control, JavaScript can dynamically execute strings as code using functions like eval(), and many applications use third-party JavaScript code that developers cannot fully monitor. The paragraph also reviews earlier research and tools. Some methods used static analysis, black-box fuzzing, or taint tracking to detect XSS, but each had limitations. Some tools focused on tracking sensitive information instead of malicious input, some required manual testing, and others could not automatically verify vulnerabilities. The authors explain that black-box testing often misses vulnerabilities because it cannot test every possible attack input, while static analysis may generate many false positives in complex situations. Therefore, the paper proposes using dynamic taint tracking because it can follow the actual flow of untrusted data during program execution and improve the detection and verification of DOM-XSS vulnerabilities.

## 3 Detection framework for DOM-XSS
TT-XSS dynamic DOM-XSS detection framework is mainly composed of three modules: URL information collection and analysis, taint tracking analysis and automatic vulnerability verification.

<p align="center"><img src="Figures/Detection_Process_of_TT-XSS.png" alt="Detection Process of TT-XSS" width="600" /></p>

### Understandable process flow of TT-XSS Detection:
```
Raw URLs
    ↓
Input into Framework
    ↓
Preprocessing
    ↓
Dynamic + Static Parsing
    ↓
Taint Tracking Analysis
    ↓
Obtain Taint Traces
    ↓
Analyze data flow inside web pages
    ↓
Automatic Vulnerability Verification
    ↓
Generate Vulnerability Report
```

### 3.1 URL Information Collection and Analysis Module 
This module functions as an intelligent web crawler that systematically discovers and processes web pages. Let me break down how it works:

#### 1. URL Queue System
- Starts with a queue (waiting list) containing the initial input URLs
- Continuously processes URLs from this queue
- As new URLs are discovered, they're added back to the queue
- Continues until the queue is empty (all URLs processed)

#### 2. Browser-Based Processing
When processing each URL, a browser:
- Parses the URL structure
- Renders the actual webpage (loads it visually)
- Gathers information from the rendered page
- Extracts new URLs found on that page

#### Two Key Features
**Feature 1:** Dual Parsing Methods

**Static Parsing:**
- Analyzes only the raw HTTP response (the initial server response)
- Extracts URLs directly visible in the HTML code
- Fast but limited - misses dynamically loaded content

**Dynamic Parsing:**
- Actually renders the webpage in a browser
- Waits for JavaScript to execute
- Captures URLs that appear after page interactions
- More comprehensive - finds URLs loaded via AJAX, JavaScript events, etc.

**Feature 2:** Duplicate Removal
The system prevents processing the same URL multiple times using two comparison methods:
- URL Comparison - Checks if the exact URL was already visited
- Script Comparison - Checks if pages have identical JavaScript code (might indicate same functionality despite different URLs)

**Bloom Filter Technology:**
- A space-efficient probabilistic data structure
- Quickly checks "Have we seen this URL before?" without storing every URL
- Very fast and memory-efficient for large-scale crawling

#### Why This Design Matters
- Comprehensive coverage - Finds both static and dynamic URLs
- mEfficiency - Avoids processing duplicates
- Thoroughness - Discovers the complete web structure, including hidden/JavaScript-loaded pages
- Scalability - Bloom Filter handles millions of URLs efficiently

### 3.2 Taint tracking analysis
Think of taint tracking like putting a colored dye in water to trace where it flows through a pipe system. Here, we're tracking how user input flows through a website to find security vulnerabilities.
#### What Problem Does This Solve?
DOM-XSS (Cross-Site Scripting) attacks happen when malicious code runs entirely inside your browser. Imagine someone injecting harmful code through a search box, and that code executes without the website even realizing it. This method catches those vulnerabilities.
#### How It Works?? 
When you type something into a website (like a search query or form), this system "tags" that input with a special marker, like putting a tracking chip on it. Then it watches where that tagged data goes and what happens to it.
**Example:**
- You type `<script>alert('hacked')</script>` in a search box
- The system tags this input: "This came from a user"
- It follows this tagged data as it moves through the website's code
- If the tagged data ends up somewhere dangerous (like being executed as code), it raises an alert

#### The Four Steps (Explained Simply)
**Step 1:** Make a List of Risky Functions
They identify all the dangerous "doors" where user input could cause trouble. These are called DOM APIs - basically, the functions websites use to manipulate web pages.
Examples of risky functions:
- `innerHTML` - inserts content into a page
- `document.write` - writes directly to the page
- `eval()` - executes code from text

Each one gets a number, like creating an index of "potential danger zones."

**Step 2**: Design the Tagging System
They figure out:
- How to tag the data - like putting a virtual sticker saying "User Input - Handle With Care"
- How tags transfer - if tagged data gets copied or modified, the tag follows it
- Where to store the tags - keeping a record of what's tagged and where it's going

Think of it like a package delivery system tracking a parcel through every checkpoint.

**Step 3:** Modify the Browser Engine
This is the technical heavy-lifting. They actually change how the browser works (specifically JavaScriptCore and WebKit engines) so that:
- Every risky function is modified to recognize and pass along tags
- When user input flows through these functions, the tags stay attached
- The system can see the complete journey of the data

**Analogy:** It's like installing cameras at every intersection in a city to track where a specific car goes.

**Step 4:** Analyze the Results
Once data flows are tracked, they examine:
- Which user inputs reached dangerous destinations?
- Did any input get executed as code?
- Where are the vulnerabilities?

This creates a taint trace - a complete map showing "User typed X → it went through functions A, B, C → it ended up being executed as code in location Z."

#### Special Advantages
**Works in Real-Time**

This isn't analyzing code on paper - it runs while you're actually browsing, catching issues as they happen.

**Beats Code Obfuscation**

Some hackers scramble their code to hide what it does. This method doesn't care because it watches what the code actually does in your browser, not what it looks like on paper.
Example: Even if malicious code looks like gibberish:
```javascript
var _0x4df=['alert','hacked'];eval(_0x4df[0]+'("'+_0x4df[1]+'")');
```
The system sees it's still using eval() and tracks it.

**Works With Third-Party Libraries**

Websites often use popular libraries like jQuery. The tracking still works accurately because it monitors the browser's underlying functions, not the high-level library code.

#### Real-World Example
**Vulnerable Website:**
```javascript
// User searches for something
var userSearch = getInputFromSearchBox(); 
document.getElementById('results').innerHTML = "You searched for: " + userSearch;
```
**What Taint Tracking Sees:**
- Tag: userSearch is marked as "user-controlled"
- Track: Tagged data flows into innerHTML
- Alert: "User input is being inserted directly into HTML - XSS vulnerability detected!"

The Fix: Sanitize the input first before displaying it.

#### Bottom Line
This method is like having a security guard with a tracking device who:
- Tags anything that comes from users
- Follows it everywhere through the website
- Sounds an alarm if it reaches somewhere it could cause harm

### 3.2.1 Analysis of related DOM APIs
Think of a DOM-XSS attack like water flowing through a house's plumbing. To prevent flooding (attacks), you need to monitor three things:
- Where water enters (Sources)
- How it travels through pipes (Transfer Functions)
- Where it comes out (Sinks)

If you miss monitoring even one pipe, you'll lose track of the water and miss potential leaks.

#### **Part 1:** Controllable Sources (Where Attacks Enter)
<p align="center"><img src="Figures/controllable_source.png" alt="Controllable sources" width="600" /></p>

These are the **entry points** hackers can manipulate to inject malicious code.

**What Are They?**<br>
Controllable sources are parts of a website that users can directly influence, like:
- The URL in your browser's address bar
- Cookies stored on your computer
- The name of your browser window

**The Main Categories:**

Location-Based Sources (Most Common) These come from the URL itself:


| Source                | What It Is            | Example                                 |
| --------------------  | -----------------     | ------------------                      |
| `location.href`       | The complete URL      | `https://example.com/search?q=hello#top`|
| `location.pathname`   | The page path         | `/search`                               |
| `location.search`     | Query parameters      | `?q=hello`                              |
| `location.hash`       | The anchor/fragment   | `#top`                                  |

Real Attack Example:
```
https://vulnerable-site.com/search?q=<script>alert('hacked')</script>
```
The `location.search` contains malicious code that could be executed.

**Document-Based Sources**
| Source                | What It Does            |
| --------------------  | -----------------     |
|`document.referrer`  | Where you came from (previous page)| 
|`document.cookie`    | Stored cookies                     |
|`document.URL`       | Current page URL                   |

**Attack Scenario:** A hacker creates a malicious website that redirects you to a vulnerable site. The `document.referrer` now contains the hacker's injected code.

**Window-Based Sources**
| Source                | What It is            |
| --------------------  | -----------------     |
|`window.name`  | A storage area that persists across page loads| 

**Why It's Dangerous:** Hackers can set `window.name` to malicious code, and even when you navigate to a new page, that value stays in your browser.

#### Part 2: Transfer Functions (How Data Transforms)
<p align="center"><img src="Figures/transfer_function.png" alt="Transfer functions" width="600" /></p>

Once malicious data enters, it often gets modified as it flows through the code. These are the "pipes" that change the data but must keep the tracking tag attached.

**Why This Matters:**

If you tag input as "dangerous" but it gets decoded, sliced, or concatenated, you need to ensure the tag follows the modified data.

**Types of Transfer Functions:**

**Decoding Functions** (Unwrap Hidden Code)
```javascript
decodeURI("%3Cscript%3Ealert('xss')%3C/script%3E")
// Becomes: <script>alert('xss')</script>
```
Hackers encode malicious scripts to bypass filters. These functions decode them back.

**String Manipulation**
```javascript
var input = "<script>alert('xss')</script>";
var partial = input.slice(0, 8); // "<script>"
```
Even a piece of the malicious code is still dangerous!

**Concatenation** (Joining Strings)
```javascript
var part1 = "<scr";
var part2 = "ipt>alert(1)</script>";
var dangerous = part1 + part2; // Full attack code assembled
```
Tags must follow through joins so you know the complete string is tainted.

**Pattern Matching & Replacement**
```javascript
var input = "HELLO<script>alert(1)</script>";
var cleaned = input.replace("HELLO", ""); 
// Still contains: <script>alert(1)</script>
```

Even after replacement, the dangerous part remains.

**Getting/Setting Values**
```javascript
document.getElementById('search').value = userInput;
```
If userInput is tagged as dangerous, the tag must transfer to the element's value.

#### Part 3: Sink Functions (Where Attacks Execute)
<p align="center"><img src="Figures/sink_function.png" alt="Sink functions" width="600" /></p>

These are the danger zones - functions that can actually execute code or modify the page in harmful ways.

**Why They're Called "Sinks":**

This is where the "tainted water" finally flows out and causes damage.

**Types of Dangerous Sinks:**

**Code Execution (MOST DANGEROUS)**
| Function              | What It Does          | Danger Level                            |
| --------------------  | -----------------     | ------------------                      |
|eval()         | Executes any string as JavaScript code  | 🔴 Critical|
|setTimeout()   | Runs code after a delay                 | 🔴 Critical|
|setInterval()  | Runs code repeatedly                    | 🔴 Critical|


Example:
```javascript
var userInput = location.search; // "?code=alert('hacked')"
eval(userInput); // EXECUTES THE ATTACK!
```
**Page Redirection**
| Function              | What It Does          |
| --------------------  | -----------------     | 
|`window.location`       | Redirects to a new URL|
|`location.href`          | Changes the current page|
|`location.replace()`     | Replaces the current page|

**Attack:**
```javascript
location.href = "javascript:alert('xss')"; // Executes code!
```
**DOM Modification**
| Function              | What It Does          |
| --------------------  | -----------------     | 
|innerHTML              | Inserts HTML (including scripts)|
|document.write()       | Writes directly to the document|

**Example:**
```javascript
document.getElementById('result').innerHTML = userInput;
// If userInput = "<img src=x onerror='alert(1)'>", attack executes!
```

#### Putting It All Together: A Complete Attack Path
**Step-by-Step Attack Flow:**
```javascript
// 1. SOURCE: Attacker crafts malicious URL
// URL: example.com/page?search=<script>alert('hacked')</script>

// 2. TRANSFER: Website processes the input
var userSearch = location.search;           // Source (tagged)
userSearch = decodeURI(userSearch);         // Transfer (tag follows)
userSearch = userSearch.replace("?search=", ""); // Transfer (tag follows)

// 3. SINK: Website displays the "search" result
document.getElementById('output').innerHTML = userSearch; // SINK - ATTACK!
```
**What Taint Tracking Sees:**
- Tag applied at: `location.search`
- Tag transferred through: `decodeURI → replace`
- Tag reached dangerous sink: `innerHTML`
- ALERT: `DOM-XSS Vulnerability Detected!`

#### Why All Three Must Be Tracked
**Missing Source Tracking:** You won't know malicious data entered the system.
**Missing Transfer Tracking:** You lose track of the data as it transforms.
**Missing Sink Tracking:** You won't know when/where the attack executes.

**The Complete System:** By numbering and modifying ALL these functions (Tables 1, 2, and 3), the system creates an unbreakable tracking chain from the moment data enters to when it potentially causes harm.


### 3.2.2 Design of Data Types and Methods 
This section describes how **taint tracking** is implemented inside the **WebKit engine** by introducing custom data types and methods. The goal is to monitor how user-controllable data (taint) flows through a web page's JavaScript execution, ultimately detecting potential **DOM-based Cross-Site Scripting (DOM-XSS)** vulnerabilities.

The design is divided into **three functional parts**:
1. Data types and methods for **marking taint strings**
2. Data types and methods for **storing taint information**
3. Methods for **outputting taint information**

**Table 4: Key Parameters of Data Types**

| Data Type   | Parameter    | Description                                              |
|-------------|--------------|----------------------------------------------------------|
| `ExecState` | `Taint`      | JavaScript string with taint strings                     |
| `TaintInfo` | `TaintTrace` | `TaintTrace` instances associated with a sink API        |
| `TaintTrace` | `Trace id`  | Current `TaintTrace` serial number                       |
| `TaintTrace` | `Trace func`| JavaScript function execution sequence                   |
| `TaintTrace` | `Trace detail` | Function execution detail information                 |
| `TaintTrace` | `ExecState` | `ExecState` instance for the `TaintTrace`                |
| `Counter`   | *(singleton)*| Singleton type used as `TaintTrace` counter              |


**Part 1 — Marking Taint Strings**

**What is a Taint String?**
A **taint string** is a JavaScript string that carries data from a **controllable source** (e.g., `location.hash`, `document.URL`). These strings are the vehicles through which potentially malicious data travels in the application.

**How Taint Strings are Tracked**

- **JavaScript strings** are the primary carriers of taint data.
- The value of a JavaScript string can be accessed from an **`ExecState` instance** while JavaScriptCore (the JS engine in WebKit) executes functions.
- To enable taint tracking, a new attribute called **`Taint`** is added to the JavaScript string class.
- **Getter and setter methods** are added for this `Taint` attribute so it can be read and written during execution.
- The `Taint` attribute stores the **trace number** of the current taint string — this number links the string back to its original taint trace entry.<br>

**Why This Matters**

Without modifying the string class to carry a taint marker, there would be no way to follow a piece of tainted data as it gets passed between functions, concatenated, or transformed during JavaScript execution.



**Part 2 — Storing Taint Information**

Three classes are designed to store and manage all taint-related information:


**2.1 `TaintTrace` Class**

This is the **core data structure** that records everything about a single taint flow. It has three attributes:

**`Trace id`**

- A **serial number** that uniquely identifies each taint trace.
- Generated by the **`Counter`** class every time a new controllable source operation is encountered.
- Serves a dual purpose:
  1. Acts as a unique label for the taint trace record.
  2. Is assigned to the taint string to link the string to its trace during taint transfer.

**`Trace func`**

- Stores the **sequence of functions executed** during the taint flow, in string format.
- Initially empty.
- Each time a function is executed in the taint flow, that function's **number** (from a predefined function table) is appended to `Trace func`.
- Functions are numbered using **two-digit hexadecimal** (e.g., `01`, `2a`, `ff`).
- **Special marker:** When `"ff"` is appended to `Trace func`, it signals that the taint has successfully reached a **sink** (a dangerous output point like `innerHTML` or `eval`).
- By reading `Trace func`, analysts can reconstruct the exact execution path the tainted data followed.

**Example:**

Trace func: **"01 03 0a ff"**

This means functions `01`, `03`, `0a` were executed in sequence, and `ff` means the taint reached a sink.

**`Trace detail`**

- Stores **detailed information** about each function execution, in a **string vector** (a list of strings).
- Initially empty.
- For each function executed during the taint flow, the following details are stored in order:
  1. **Parameters** — the arguments passed to the function, along with the parameter count (only for transfer operations).
  2. **Function number** — the number from the function table, linking the detail entry to `Trace func`.
  3. **String content** — the actual string value after the operation has been applied.

This structured format allows analysts to look up any function number in `Trace detail` and retrieve exactly what happened at that step.

**Summary of `Trace detail` storage format per function call:**
```
[parameter count] [parameter values] → [function number] → [resulting string content]
```


**2.2 `TaintInfo` Class**

- Each **`Document`** object in the WebKit engine is associated with one `TaintInfo` instance.
- `TaintInfo` stores a **collection of `TaintTrace` instances**, each corresponding to a different controllable source operation that occurred in that document.
- This allows all taint traces for a given page to be grouped together under one `TaintInfo` object.



**2.3 `Counter` Class**

- Implemented as a **singleton** (only one instance exists throughout the entire execution).
- Its sole responsibility is to **generate a new, unique serial number** every time a controllable source function is called.
- This number is assigned to a new `TaintTrace` as its `Trace id`, and also stamped onto the taint string so the two can be linked together.

**Why singleton?** Because all taint traces across the entire page must share one consistent numbering sequence. A singleton guarantees no two traces ever get the same ID.



**Part 3 — Outputting Taint Information**

**`showTaint` Interface**

- A method called **`showTaint`** is added to the **`Document`** object in WebCore (the rendering engine of WebKit).
- This interface allows taint information to be **outputted during the HTML parsing process** on the client side.
- It provides a convenient way to inspect and extract all taint traces that were recorded for a given document.



**How Vulnerability Detection Works**

The design connects all three parts to detect **DOM-XSS vulnerabilities**:

1. When a **controllable source** (e.g., `location.hash`) is accessed, `Counter` generates a new ID and a new `TaintTrace` is created.
2. The taint string is marked with this ID via the `Taint` attribute on `ExecState`.
3. As the tainted string flows through JavaScript functions, each function appends its number to `Trace func` and logs details into `Trace detail`.
4. If the tainted string reaches a **sink** (e.g., `document.write`, `innerHTML`), `"ff"` is appended to `Trace func`.
5. After parsing, `showTaint` outputs all recorded `TaintTrace` instances from `TaintInfo`.
6. Any trace whose `Trace func` **ends with `"ff"`** represents a **confirmed source-to-sink path** — a potential DOM-XSS vulnerability.

**Summary Diagram**

```
Controllable Source
        │
        ▼
Counter generates ID ──► New TaintTrace created
        │
        ▼
Taint string marked with ID (via ExecState.Taint)
        │
        ▼
Functions execute → Trace func grows (01, 03, 0a...) → Trace detail stores parameters, function numbers, string values
        │
        ▼
Sink reached? → Append "ff" to Trace func
        │
        ▼
showTaint outputs TaintInfo → Traces ending in "ff" = Potential DOM-XSS
```


**Key Takeaways**

| Concept | Purpose |
|---|---|
| `Taint` attribute on JS strings | Links a string to its trace ID as it flows through execution |
| `Counter` singleton | Ensures every taint trace has a unique, consistent serial number |
| `Trace func` | Records the sequence of functions; `"ff"` at the end = sink reached |
| `Trace detail` | Provides granular per-function details for forensic analysis |
| `TaintInfo` per `Document` | Groups all taint traces belonging to one web page |
| `showTaint` on `Document` | Entry point to extract all taint results after parsing |
| Trace ending in `"ff"` | Indicates a **real source-to-sink flow** = potential DOM-XSS vulnerability |


### 3.3 Automated Vulnerability Verification

**3.3.1 Attack Vectors**

**What is an Attack Vector?**

An **attack vector** is a specially crafted input string that is injected into a web application to test whether a **DOM-XSS vulnerability** actually exists and can be exploited. It is not just a random payload — it is a structured string designed to survive string operations, fix broken syntax, execute malicious code, and avoid interference from surrounding code.


**Structure of an Attack Vector**

The formal structure is defined as:
```
Vector ::= [PaddingBlock] [ClosingBlock] Payload [ExtraBlock]
```

The vector has **four components**, each serving a distinct role:

**Component 1 — `PaddingBlock` (Optional)**

**Definition:**
```
PaddingBlock ::= { a | b | c | ... | x | y | z }
```

A long string made up of alphabetic characters (e.g., `"aaaaaaaaaa..."`).

**Purpose:**

During taint transfer, the tainted string may pass through **string manipulation functions** such as:
- `substr()`
- `substring()`
- `slice()`

These functions can **truncate** the string, cutting off the important parts of the attack vector. By prepending a long padding string, these operations consume the padding characters first, leaving the actual payload intact and uncorrupted.

> **Analogy:** Think of it as a sacrificial buffer — the string operation "eats" the padding, so the real payload survives.



**Component 2 — `ClosingBlock` (Optional)**

**Definition:**
```
ClosingBlock ::= { ); | ); | > | > | > | </a> }
```

**Purpose:**

When the attack vector is injected into existing code or HTML, there is often **already-open syntax** before the injection point — for example:
- An open function call: `someFunction("`
- An open HTML tag: `<div id="`

The `ClosingBlock` contains closing characters that **complete and fix** the surrounding syntax before the payload begins. This ensures the browser or JavaScript engine does not throw a syntax error and instead proceeds to execute the malicious payload.

> **Example:** If the injection point is inside `document.write("` then a `ClosingBlock` of `");` closes that call cleanly before the payload runs.


**Component 3 — `Payload` (Required — Core of the Attack Vector)**

**Definition:**
```
Payload ::= { alert(1); | <svg/onload=alert(1)> | javascript:alert(1) }
```
**Purpose:**

The payload is the **actual malicious code** that proves the vulnerability exists. If the payload successfully executes (i.e., triggers `alert(1)`), it confirms a real DOM-XSS vulnerability.

There are **three types of payload**, chosen based on the **output/sink position** of the tainted data:

| Payload Type | Used When Sink Is... | Example |
|---|---|---|
| `alert(1);` | Inside a **JavaScript context** (e.g., `eval()`, `setTimeout()`) | `eval("alert(1);")`  |
| `<svg/onload=alert(1)>` | Inside an **HTML context** (e.g., `innerHTML`, `document.write`) | `div.innerHTML = "<svg/onload=alert(1)>"` |
| `javascript:alert(1)` | Inside a **URL context** (e.g., `location.href`, `<a href=...>`) | `location.href = "javascript:alert(1)"` |

This context-awareness is critical — a JavaScript payload injected into an HTML sink would not execute, and vice versa.



**Component 4 — `ExtraBlock` (Optional)**

**Definition:**
```
ExtraBlock ::= { // | <" | < }
```

**Purpose:**

After the injection point, there is often **remaining code or characters** that were originally part of the source string. If left as-is, this trailing content could:
- Cause a **syntax error** that prevents the payload from executing
- Interfere with the payload's logic

The `ExtraBlock` neutralizes this by either:
- **Commenting out** the remaining code (`//` in JavaScript)
- **Breaking the remaining HTML** structure (`<"` or `<`)

> **Example:** If the original string is `"userInput + rest_of_code"`, the `ExtraBlock` `//` comments out `rest_of_code` so it does not disrupt execution.


**Summary Table of Attack Vector Components**

| Component | Required? | Role | Example Values |
|---|---|---|---|
| `PaddingBlock` | Optional | Prevents truncation by string operations | `"aaaaaaaaa..."` |
| `ClosingBlock` | Optional | Closes existing open syntax before payload | `);`, `>`, `</a>` |
| `Payload` | **Required** | Executes to confirm vulnerability | `alert(1);`, `<svg/onload=alert(1)>` |
| `ExtraBlock` | Optional | Neutralizes interfering trailing code | `//`, `<"`, `<` |


**What Determines the Attack Vector's Content and Position?**

Three factors control how the final attack vector is assembled and where it is placed:

| Factor | Controls |
|---|---|
| **Transfer operations** | Which `ClosingBlock` and `ExtraBlock` are needed |
| **Sink position** | Which `Payload` type to use (JS / HTML / URL context) |
| **Source position** | Where in the request the attack vector is injected |



**Handling Second-Order Inputs**

Some sources are **second-order inputs** — meaning the data does not come directly from the URL but from stored or indirect sources such as:
- `document.referrer`
- Cookies

Because these values are **not directly controllable** in a single request, they **cannot be automatically verified** and require **manual analysis**.



**Table 6: Attack Vector Placement Rules by JS Source Function**

When the **entire URL** is used as input, attack vectors are placed in the **URL anchor (fragment — the `#` part)** for two key reasons:

1. **Anchor content is not sent to the server** — it stays entirely on the client side, so the server cannot interfere with, modify, or block the attack vector.
2. **Less URL encoding is applied** to anchor strings, reducing the chance the attack vector gets mangled by encoding transformations.

| JS Function | URL Placement Strategy | Example URL Pattern |
|---|---|---|
| `location` / `location.href` / `document.documentURI` / `document.URL` / `document.baseURI` / `document.URLUnencoded` | Vector placed in **anchor** (`#`) | `protocol://hostname[:port]/path/[;parameters][?query]#Vector` |
| `location.pathname` | Vector placed in **path** | `protocol://hostname[:port]/Vector/[;parameters][?query]#hash` |
| `location.search` | Vector placed in **query string** | `protocol://hostname[:port]/path/[;parameters][?Vector]#hash` |
| `location.hash` | Vector placed in **anchor** (`#`) | `protocol://hostname[:port]/path/[;parameters][?query]#Vector` |


**3.3.2 Automated Verification Process**

Once attack vectors are generated, the system follows a **step-by-step automated pipeline** to verify whether a real DOM-XSS vulnerability exists.



**Step-by-Step Process**

```
┌─────────────────────────────────────┐
│  1. Analyze Taint Trace Information │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  2. Generate Attack Vectors         │
│     (based on trace analysis)       │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  3. Construct Effective HTTP        │
│     Requests & Send to Target       │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  4. Receive HTTP Response           │
│     on Client Side                  │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  5. Render JavaScript & Simulate    │
│     Browser Behavior                │
└────────────────┬────────────────────┘
                 │
      ┌──────────┴────────┐
      ▼                   ▼
┌──────────────┐   ┌──────────────────────┐
│  SUCCESS     │   │  FAILURE             │
│              │   │                      │
│ Generate     │   │ Preserve taint info  │
│ vulnerability│   │ for manual analysis  │
│ report with  │   └──────────────────────┘
│ attack       │
│ vectors      │
└──────────────┘

```

**Detailed Explanation of Each Step**

#### Step 1 — Analyze Taint Trace Information
- The taint trace data collected during the taint tracking phase (Section 3.2) is examined.
- Traces that **end with `"ff"`** in their `Trace func` field are identified — these represent confirmed **source-to-sink flows**.
- The trace's details reveal which source function was used, what operations were applied, and which sink received the tainted data.

#### Step 2 — Generate Attack Vectors
- Based on the analysis:
  - The **sink type** (JS / HTML / URL context) determines which `Payload` to use.
  - The **transfer operations** along the trace determine whether a `PaddingBlock`, `ClosingBlock`, or `ExtraBlock` is needed and what their content should be.
  - The **source type** determines where in the HTTP request the vector is injected.

#### Step 3 — Construct and Send HTTP Requests
- A valid HTTP request is built with the attack vector embedded in the correct position (path, query string, or anchor, per Table 6).
- The request is sent to the **target server**.

#### Step 4 — Receive HTTP Response
- The client receives the server's HTTP response containing the rendered web page.

#### Step 5 — Render JavaScript and Simulate Browser
- The client **simulates a real browser environment**, rendering the page's JavaScript.
- This step determines whether the injected payload actually **executes** (i.e., `alert(1)` fires).


**Outcomes**

| Outcome | Result |
|---|---|
| **Payload executes successfully** | A **vulnerability report** is generated, including the confirmed attack vector and full taint trace details. |
| **Payload does not execute** | The taint trace information is **preserved** for further **manual analysis** by a security researcher. |



**Key Takeaways**

| Concept | Significance |
|---|---|
| Structured attack vector | Ensures the payload survives string operations, fixes surrounding syntax, and neutralizes trailing code |
| Three payload types | Context-aware exploitation — JS, HTML, and URL sinks each need a different payload |
| Anchor placement preference | Avoids server interference and reduces encoding issues |
| Automated browser simulation | Allows real end-to-end confirmation without manual browser testing |
| Fallback to manual analysis | Handles edge cases (e.g., second-order inputs) that automation cannot reliably verify |


## 4. TTXSS Prototype Framework — Architecture and Module Design

Based on the dynamic taint tracking detection method described in this paper, a prototype framework called **TTXSS** was designed and implemented. It automates the entire process of discovering, detecting, and verifying DOM-XSS vulnerabilities in web applications.



### Framework Architecture


<p align="center"><img src="Figures/tt-based-detection framework for DOM-XSS.png" alt="Detection Process of TT-XSS" width="600" /></p>

TTXSS is composed of **six core modules**, each with a distinct responsibility:

```
┌─────────────────────────────────────────────────────────────────┐
│                        UI Module                                │
│         (GUI interface — task distribution & monitoring)        │
└───────┬─────────────────────────────────────┬───────────────────┘
        │ distributes tasks                   │ view/modify detections
        ▼                                     ▼
┌───────────────────┐                ┌──────────────────────────┐
│  Crawler Module   │                │  Data Persistence Module │
│  (URL discovery   │                │  (stores all results     │
│   & parsing)      │                │   and reports)           │
└───────┬───────────┘                └──────────┬───────────────┘
        │ URL list                              ▲
        ▼                                       │ stores info
┌───────────────────────┐                       │
│  Task Scheduling      │                       │
│  Module               │                       │
│  (distributes URLs    │                       │
│   to Detection)       │                       │
└───────┬───────────────┘                       │
        │ calls                                 │
        ▼                                       │
┌───────────────────────┐   taint traces   ┌────┴──────────────────┐
│   Detection Module    │ ───────────────► │  Verification Module  │
│   (renders pages,     │                  │  (generates vectors,  │
│    gets taint traces) │ ◄─── connected   │   simulates attacks,  │
└───────────────────────┘     in series    │   confirms vulns)     │
                                           └───────────────────────┘
```



### Module Descriptions

**1. Crawler Module**
- **Role:** Discovers the attack surface of the target web application.
- **How:** Crawls and parses URLs to find all places where user-controlled input can be submitted (potential source injection points).
- **Output:** A list of URLs passed to the Task Scheduling Module.

**2. Task Scheduling Module**
- **Role:** Acts as the coordinator between modules.
- **How:** Receives the URL list from the Crawler Module, distributes them as tasks, and calls the Detection Module to analyze each URL.
- **Why it matters:** Ensures the framework can handle multiple URLs concurrently and in an organized manner.

**3. Detection Module**
- **Role:** Identifies **potential DOM-XSS vulnerabilities** by rendering pages and extracting taint traces.
- **How:** Renders each page in a modified headless browser, runs the taint tracking system, and collects all source-to-sink taint traces.
- **Output:** Taint trace data for pages with potential vulnerabilities, passed to the Verification Module.

**4. Verification Module**
- **Role:** Confirms whether identified potential vulnerabilities are **real and exploitable**.
- **How:** Analyzes taint traces, generates targeted attack vectors, and simulates real attacks.
- **Connection:** Runs **in series** with the Detection Module — detection feeds directly into verification.
- **Output:** Verified vulnerability reports stored by the Data Persistence Module.

**5. Data Persistence Module**
- **Role:** Stores all collected information — taint traces, detection reports, and verified vulnerability results.
- **Interaction:** Connected to both the Verification Module (receives results) and the UI Module (allows viewing and modifying detections).

**6. UI Module**
- **Role:** Provides a **graphical user interface (GUI)** for human operators.
- **Functions:**
  - Distributes tasks to the Crawler Module.
  - Interacts with the Data Persistence Module to view, monitor, and modify the current detection state.

### 4.1 Detection Module — Detailed Design

### Core Tool: Modified PhantomJS

The Detection Module is built on a **modified version of PhantomJS**, an open-source headless browser. PhantomJS was chosen because:

| Feature | Relevance to TTXSS |
|---|---|
| No UI required | Allows automated, server-side execution without a display |
| Full DOM rendering | Accurately simulates how a real browser processes a page |
| Full JavaScript execution | JavaScriptCore runs all JS, including taint-relevant operations |
| Network access | Can make real HTTP requests to target servers |
| Canvas/SVG support | Handles modern web content accurately |
| Wide application in automation | Already proven in crawling, testing, and page output tasks |

**PhantomJS's rendering stack:**
```PhantomJS
└── QtWebKit (rendering engine)
└── JavaScriptCore (JavaScript parser/executor)     
```

When PhantomJS loads a web page, it behaves like a full browser — JavaScript is executed just as it would be in Chrome or Firefox, making it a valid environment for taint tracking.

---

**What Was Modified in PhantomJS**

Two core components were modified:

- **WebCore** — handles DOM parsing, HTML rendering, and document-level operations
- **JavaScriptCore** — handles JavaScript parsing and execution

Modifications were made at three levels (corresponding to Table 1 in the paper):

| Level | What Was Modified | Purpose |
|---|---|---|
| **Source functions** | Functions like `location.hash`, `document.URL`, etc. | Initialize a new `TaintTrace` instance and mark `ExecState` as tainted |
| **Transfer functions** | String operations like `concat`, `substr`, `split`, etc. | Record taint propagation — append function numbers to `Trace func`, store details in `Trace detail` |
| **Sink functions** | Functions like `innerHTML`, `eval`, `document.write`, etc. | Call the `report` function and pass the `TaintTrace` instance to the Verification Module |

---

**How the Detection Module Works — Step by Step**

#### Step 1 — Page Loading
PhantomJS loads the target web page and begins executing JavaScript via JavaScriptCore.

#### Step 2 — Source Initialization
When a **controllable source function** is encountered (e.g., `location.hash` is read):
- The `initialize` function is called.
- A new `TaintTrace` instance is created with a unique ID from the `Counter` singleton.
- The `ExecState` instance is marked as **tainted**.

#### Step 3 — Taint Transfer Tracking
As the tainted string flows through **transfer functions** (string operations):
- Each function's number is appended to `Trace func`.
- Parameters, function number, and resulting string content are stored in `Trace detail`.
- The taint marking is propagated to any new strings produced by the operation.

#### Step 4 — Sink Detection and Reporting
When the tainted string reaches a **sink function**:
- `"ff"` is appended to `Trace func`, marking the trace as having reached a sink.
- The `report` function is called.
- The complete `TaintTrace` instance is passed to the **Verification Module**.

#### Step 5 — All Traces Reported
Every possible source-to-sink trace is reported and queued for verification. No potential vulnerability is discarded at this stage — all are preserved for the verification step to confirm.


```
Source Function Called
          │
          ▼
Initialize TaintTrace + Mark ExecState tainted
          │
          ▼
Tainted string passes through Transfer Functions
→ Append function numbers to Trace func
→ Store details in Trace detail
          │
          ▼
Tainted string reaches Sink Function
→ Append "ff" to Trace func
→ Call report()
→ Pass TaintTrace to Verification Module
```


### 4.2 Verification Module — Detailed Design

**Purpose**
After the Detection Module identifies one or more **dangerous traces** (traces with `"ff"` in `Trace func`), the Verification Module takes over to determine whether those traces represent **real, exploitable DOM-XSS vulnerabilities**.


**Step-by-Step Verification Process**

#### Step 1 — Store Detection Reports as XML
Dangerous trace information from the Detection Module is stored in **XML format**. Each XML detection report contains:
- **Controllable source operations** — which source function triggered the trace
- **Transfer process** — the sequence of string operations the tainted data passed through
- **Sink operations** — which sink function received the tainted data

#### Step 2 — Extract Source and Sink Operations
From the XML report:
- The **controllable source operations** are extracted to determine where in the URL or request the attack vector should be injected.
- The **sink operations** are extracted to determine which type of payload (`alert(1);`, `<svg/onload=alert(1)>`, or `javascript:alert(1)`) is appropriate.

#### Step 3 — Analyze Transfer Process with Regular Expressions
- The transfer process is analyzed using **regular expressions**.
- The goal is to identify any **string truncation or concatenation operations** in the trace that could modify the attack vector.
- Variables produced after these operations are recorded to ensure the attack vector is structured correctly (with the appropriate `PaddingBlock` to survive truncation).

#### Step 4 — Determine Attack Vector and Injection Position
Using the rules from **Table 5** (attack vector structure) and **Table 6** (injection position rules):
- The appropriate **attack vector** is assembled (`PaddingBlock` + `ClosingBlock` + `Payload` + `ExtraBlock`).
- The correct **injection position** in the URL is determined (path, query string, or anchor).

#### Step 5 — Generate Test URLs
- The attack vector is embedded into the URL at the correct position.
- A complete **test URL** is produced, ready for submission.

#### Step 6 — Execute Validation Script via PhantomJS API
- The Verification Module calls the **PhantomJS API** to run a validation script.
- The script:
  1. Submits the test URL along with the relevant **cookies** to the target server.
  2. Resolves the URL in a simulated browser environment.
  3. Registers a **`page.onAlert`** handler — a PhantomJS callback that fires when `alert()` is triggered in the page.

#### Step 7 — Confirm Vulnerability
- If the **`alert` event is caught** by `page.onAlert`, the vulnerability is **confirmed as real and exploitable**.
- If no alert fires, the test URL did not successfully exploit the vulnerability (the trace may be a false positive or require manual review).

#### Step 8 — Store Final Report
- The verified vulnerability report — including the confirmed attack vector, taint trace, and URL — is stored in the **database** via the Data Persistence Module.



**Verification Flow Diagram**
```
Detection Module Output
(TaintTrace with "ff" in Trace func)
            │
            ▼
Store as XML Detection Report
(source ops + transfer process + sink ops)
            │
            ▼
Extract source & sink operations
            │
            ▼
Analyze transfer process via RegEx
(record truncation/concatenation variables)
            │
            ▼
Determine attack vector (Table 5)

injection position (Table 6)
            │
            ▼
Generate Test URL
            │
            ▼
PhantomJS runs validation script
(submits URL + cookies, registers page.onAlert)
            │
    ┌───────┴──────────┐
    ▼                  ▼
alert() fires       No alert
    │                  │
    ▼                  ▼
Vulnerability      Preserve for
Confirmed         manual analysis
    │
    ▼
Store report in database
```



## Key Takeaways

| Module | Core Responsibility | Key Technology |
|---|---|---|
| Crawler | Discover injectable URLs | Web crawling & URL parsing |
| Task Scheduling | Coordinate and distribute work | Task queue management |
| Detection | Find potential vulnerabilities via taint tracking | Modified PhantomJS + WebCore + JavaScriptCore |
| Verification | Confirm real exploitability | Attack vector generation + PhantomJS `page.onAlert` |
| Data Persistence | Store all results | XML reports + database |
| UI | Human interface for control and monitoring | GUI over the full framework |

The **Detection and Verification modules are the heart of TT-XSS**. Detection uses a modified headless browser to passively observe all taint flows, while Verification actively attacks the identified flows to separate real vulnerabilities from false positives — all without requiring manual browser interaction.

## 5. Experiments — Detailed Explanation

To validate the feasibility and reliability of the TTXSS framework, experiments were conducted using **Firing Range** — a purpose-built vulnerable web application designed specifically for testing web security tools. The results were also compared against three existing tools: **Ra.2**, **JSPwn**, and **AWVS 10.0**.



### What is Firing Range?

- Firing Range is a **special testing web application** that covers classic web application vulnerabilities.
- It is **representative** of real-world DOM-XSS scenarios, making it an ideal benchmark.
- It is run on **Google App Engine**.

### 5.1 Experiment Setup and Process

**Hardware and Infrastructure**

| Component | Role |
|---|---|
| PC 1 | Crawled URLs and added tasks to the framework |
| PC 2 | Ran the detection engine against Firing Range, stored results in the database |
| Google App Engine | Hosted the Firing Range vulnerable web application |
| Results Database | Deployed on the second server to store all detection output |

Custom scripts were written to make the automated detection pipeline run smoothly across different software components.


**Types of DOM-XSS Vulnerabilities Tested**

**Table 7** shows the vulnerability types and counts in Firing Range used to simulate real-world conditions:

| Vulnerability Type | Count |
|---|---|
| LocationHash | 14 |
| Location | 8 |
| Cookies | 4 |
| Referrer | 3 |
| WindowName | 3 |
| LocalStorage | 10 |
| PostMessage | 3 |
| EventTriggering | 3 |
| Others | 7 |

**Sample Vulnerability Walkthrough**

The following sample code from Firing Range illustrates a typical DOM-XSS vulnerability:

```html
<html>
<head></head>
<body>
<script>
  var payload = window.location.hash.substr(1);       // Step 1: Read from URL anchor
  var div = document.createElement('div');             // Step 2: Create a new div element
  div.id = 'divEl';
  document.documentElement.appendChild(div);          // Step 3: Add div to page
  var divEl = document.getElementById('divEl');
  divEl.innerHTML = payload;                           // Step 4: Inject tainted string into innerHTML (SINK)
</script>
</body>
</html>
```

#### Step-by-Step Explanation of How TTXSS Handles This

**Step 1 — Source Detection:**
`window.location.hash` is a controllable source. When `.substr(1)` is called on it, TTXSS:
- Creates a new `TaintTrace` instance via `Counter`.
- Marks the resulting string (`payload`) as tainted.
- Records `substr` in `Trace func` and logs parameters in `Trace detail`.

**Step 2 & 3 — Transfer Tracking:**
The tainted `payload` variable is stored and passed around but not further transformed. The trace is maintained.

**Step 4 — Sink Detection:**
`divEl.innerHTML = payload` is a **sink** (HTML context). TTXSS:
- Appends `"ff"` to `Trace func`.
- Calls `report()` and passes the `TaintTrace` to the Verification Module.

**Attack Vector Generated:**
```
http://192.168.211.1/address/location.hash/innerHtml.html#<svg/onload=alert()>
```

**Why this vector?**
- **Source:** `location.hash` → vector is placed in the **anchor (`#`)** per Table 6.
- **Sink:** `innerHTML` → HTML context → payload is `<svg/onload=alert()>` per Table 5.
- **PaddingBlock:** Needed because `substr(1)` truncates the first character, so a padding of one character (`#`) is naturally handled by placing the vector after `#`.
- The `alert()` fires when the page loads, confirming the vulnerability.


### 5.2 Results and Comparison

**Tools Compared**

| Tool | Type | Description |
|---|---|---|
| **Ra.2** | Dynamic (black-box) | Firefox plugin using fuzzing datasets |
| **JSPwn** | Static analysis | Open-source static analyzer |
| **AWVS 10.0** | Dynamic (DeepScan) | Industry-standard web vulnerability scanner |
| **This paper (TTXSS)** | Dynamic (taint tracking) | The proposed framework |


**Weaknesses of Ra.2 and JSPwn**

#### Ra.2
- Relies **heavily on fuzzing datasets** — constructing a good dataset is expensive and time-consuming.
- Suffers from **high false negative rates** when application logic is complex.
- **Cannot detect** some source APIs like `document.referrer` and `window.name` because it is constrained by the Firefox plugin architecture.

#### JSPwn
- Requires **extensive manual parameter configuration** for detection rules.
- Needs **additional manual testing** due to the inherent limitations of static analysis.
- Affected by **code ambiguity and obfuscation** — if the JavaScript is minified or obfuscated, JSPwn's rule-based analysis breaks down.

> Because of these significant weaknesses, Ra.2 and JSPwn were not included in the final quantitative comparison. Only **AWVS 10.0** and **TTXSS** are compared numerically.


**Detection Results — Table 8**

| Vulnerability Type | This Paper (TTXSS) | AWVS 10.0 |
|---|---|---|
| LocationHash | 4 | 8 |
| Location | 3 | 0 |
| Cookies | 2 | 0 |
| Referrer | 2 | 3 |
| WindowName | 0 | 3 |
| LocalStorage | 0 | 0 |
| PostMessage | 0 | 0 |
| EventTriggering | 0 | 0 |
| Others | 5 | 1 |
| **Total** | **17** | **16** |

#### Analysis of Detection Results

**TTXSS detected 17 vulnerabilities vs AWVS 10.0's 16 — approximately 1.8% more.**

Key observations:

| Observation | Explanation |
|---|---|
| AWVS 10.0 detects more `LocationHash`, `Referrer`, and `WindowName` | AWVS is a mature, long-term updated commercial product with comprehensive scanning rules for well-known vulnerability patterns |
| TTXSS detects more `Others` (5 vs 1) | Uncommon or atypical vulnerabilities are harder for rule-based tools to catch; TTXSS's taint tracking is less reliant on predefined rules |
| TTXSS detects 0 for `WindowName`, `LocalStorage`, `PostMessage`, `EventTriggering` | These involve **second-order inputs** or **event-driven flows** not yet covered by TTXSS's current API set |
| All vulnerabilities with APIs covered by TTXSS were detected | TTXSS has 100% recall within its covered API scope — no false negatives for supported sources |
| Modifying APIs costs less than modifying logical expressions | TTXSS's approach (instrumenting source/transfer/sink APIs) is simpler and more maintainable than approaches that rewrite logical expressions |

**Verification Results — Table 9**

| | This Paper (TTXSS) | AWVS 10.0 |
|---|---|---|
| Detected | 17 | 16 |
| Automatically Verified | **5 (29.4%)** | **0 (0%)** |

#### Why AWVS 10.0 Verified 0

AWVS 10.0 identifies potential vulnerabilities by **tracking the source and sink positions**, but:
- It only provides the **location of input and output** — it does not capture intermediate **string variable information**.
- Without knowing how the string was transformed between source and sink, AWVS cannot construct a precise, context-aware attack vector.
- As a result, **all 16 of AWVS's detections require manual confirmation**.

#### Why TTXSS Verified 5 Automatically

TTXSS captures full **taint trace information** — including every string operation, parameter, and resulting value between source and sink. This allows it to:
- Precisely reconstruct the transformation path.
- Build an accurate attack vector (with correct `PaddingBlock`, `ClosingBlock`, `Payload`, and `ExtraBlock`).
- Automatically inject the vector and confirm execution via `page.onAlert`.


**Why Only 5 of 17 Were Automatically Verified**

The 12 vulnerabilities that were **detected but not automatically verified** come from sources that require **two input phases**, such as:
- `document.cookie`
- `document.referrer`
- `window.name`

These are **second-order inputs** — data that is first stored somewhere (e.g., in a cookie or HTTP header) and then read back later. Automating this requires controlling two separate interactions with the server, which the current framework does not yet support.

However, TTXSS still provides **full taint trace information** for these cases, making **manual verification significantly easier** compared to AWVS 10.0.


### Summary of Findings

| Metric | TTXSS | AWVS 10.0 |
|---|---|---|
| Total vulnerabilities detected | **17** | 16 |
| Automatically verified | **5** | 0 |
| Requires manual verification | 12 | **16** |
| False positive rate | **Low** | — |
| False negative rate | **Low** | — |
| Handles uncommon vulnerability types | **Yes** | Limited |
| Provides string-level trace detail | **Yes** | No |


### Key Takeaways

| Finding | Significance |
|---|---|
| TTXSS detects 1.8% more vulnerabilities than AWVS 10.0 | Competitive detection rate against a mature commercial tool |
| TTXSS is the only framework that automatically verifies any vulnerabilities | AWVS 10.0 and all other tools require 100% manual confirmation |
| Full taint trace information is the key differentiator | String variable tracking between source and sink enables precise attack vector generation |
| Second-order inputs remain a limitation | Cookies, referrer, and window.name require multi-phase interactions not yet automated |
| 100% recall within covered API scope | For all DOM-XSS types whose source APIs are instrumented, TTXSS misses none |
| The framework is easy to implement and extend | Adding more source/sink APIs directly improves coverage without architectural changes |


## 6. Conclusion and Future Work — Detailed Explanation

### 6.1 Conclusion

**What Was Proposed**
This paper proposed and implemented **TT-XSS** — a **dynamic DOM-XSS detection framework** built on the principle of **taint tracking**. It is not merely a theoretical contribution; a working **prototype** was implemented and experimentally validated.


**What TT-XSS Does**

| Contribution | Description |
|---|---|
| **Dynamic taint tracking** | Monitors how user-controllable data flows through JavaScript execution in real-time |
| **Taint trace collection** | Analyzes web pages to extract complete source-to-sink taint traces |
| **Automatic attack vector generation** | Uses taint trace information to construct precise, context-aware attack vectors without manual effort |
| **Automated vulnerability verification** | Injects attack vectors and confirms exploitability automatically via `page.onAlert` |
| **Practical developer utility** | Gives developers both detection results and detailed trace information, enabling faster and more targeted vulnerability repair |


**Why Automatic Attack Vector Generation Matters**

> *"According to these traces, our method can generate attack vectors automatically, which is quite significant for developers to detect and repair vulnerabilities effectively."*

Manual construction of attack vectors is:
- **Time-consuming** — analysts must understand the full taint flow before crafting a working vector.
- **Error-prone** — without string-level trace data, it is easy to miss truncation or encoding effects.
- **Unscalable** — manual analysis cannot keep pace with large or complex web applications.

TTXSS automates this entirely by using `Trace func` and `Trace detail` to reconstruct exactly what happened to the tainted string and build a vector that accounts for every transformation.


**Experimental Results Summary**

| Metric | TT-XSS | AWVS 10.0 |
|---|---|---|
| Vulnerabilities Detected | 17 | 16 |
| Improvement over AWVS | **+1.8%** | — |
| Automatically Verified | 5 **(9.1%)** | 0 **(0%)** |
| Manual Work Required | Partial | All |

**Two headline achievements:**
1. **Detects 1.8% more vulnerabilities** than AWVS 10.0 — a well-established, commercially maintained security tool.
2. **Automatically verifies 9.1% of vulnerabilities** — AWVS 10.0 verifies zero automatically, requiring full manual confirmation for every finding.

In the authors' own words, TT-XSS can find:
- **More comprehensive vulnerabilities** — covering a wider range of source APIs and vulnerability types (especially uncommon ones in the "Others" category).
- **More abundant vulnerability information** — full taint traces with string-level detail, not just input/output positions.

### 6.2 Limitations

The authors honestly acknowledge two current limitations of TT-XSS:

**Limitation 1 — Cannot Handle Second-Order (Two-Order) Inputs**

**What the problem is:**

The verification module currently selects attack vectors using a **regular fuzzing strategy**. This works well when the tainted input comes directly from the URL in a single request (e.g., `location.hash`). However, it **fails for second-order inputs** — sources where data is:
1. First **submitted and stored** (e.g., written into a cookie, referrer header, or `window.name`).
2. Later **read back** in a separate interaction and used in a DOM-XSS sink.

Examples of affected source types:
- `document.cookie`
- `document.referrer`
- `window.name`

Because these require **two separate phases of interaction** with the server, the current single-request attack vector injection cannot verify them automatically.

**Impact:** 12 of the 17 detected vulnerabilities fell into this category and could not be automatically verified — only their taint traces were provided for manual review.

**Limitation 2 — Slow Attack Vector Construction for Complex Payloads**

**What the problem is:**

When the taint trace involves:
- **Complex padding requirements** (e.g., many layers of string operations requiring a large or precisely sized `PaddingBlock`)
- **Complex payload structures** (e.g., deeply nested `ClosingBlock` + `ExtraBlock` combinations)

...the process of **constructing a valid attack vector** becomes computationally expensive and slow. The current approach tries combinations in a relatively straightforward manner, which does not scale well for intricate traces.


### 6.3 Future Work

The authors outline two concrete directions for future improvement:


**Future Work 1 — Extend Verification Module for Two-Order Inputs**

**Goal:** Modify the verification module to handle second-order inputs automatically.

**What this means in practice:**
- The framework would need to simulate **multi-phase interactions** — first injecting data into the storage mechanism (cookie, referrer, etc.), then triggering the page that reads it back.
- This would likely require more sophisticated session management, HTTP header manipulation, and multi-request coordination within PhantomJS.

**Expected impact:** The automatic verification rate would increase significantly beyond the current 9.1%, as many DOM-XSS vulnerabilities in real applications involve stored or second-order sources.

**Future Work 2 — Use Heuristic Search to Speed Up Attack Vector Construction**

**Goal:** Replace or augment the current attack vector construction process with **heuristic search algorithms** or other optimization techniques.

**Why heuristic search?**
- Heuristic search (e.g., genetic algorithms, simulated annealing, or guided fuzzing) can intelligently explore the space of possible attack vectors without exhaustively trying every combination.
- By learning from previous successful vectors, the algorithm can prioritize promising candidates and skip unlikely ones.

**Expected impact:** Faster construction of attack vectors for complex taint traces, making the framework more practical for large-scale automated scanning.



### Summary Table

| Aspect | Detail |
|---|---|
| **Framework name** | TT-XSS |
| **Core technique** | Dynamic taint tracking in modified PhantomJS |
| **Key achievement 1** | +1.8% more detections than AWVS 10.0 |
| **Key achievement 2** | 9.1% automatic verification vs 0% for AWVS 10.0 |
| **Limitation 1** | Cannot handle second-order (two-phase) inputs automatically |
| **Limitation 2** | Complex payloads slow down attack vector construction |
| **Future work 1** | Extend verification module for two-order inputs |
| **Future work 2** | Apply heuristic search to speed up vector construction |


## 7. Acknowledgment

This research was partially funded by:

- **National Science Foundation of China** — Grant Numbers 61572355 and 61572349
- **985 Funds of Tianjin University**
- **Tianjin Research Program of Application Foundation and Advanced Technology** — Grant Numbers 15JCYBJC15700 and 14JCTPJC00517

These funding bodies supported the research infrastructure, development, and experimental validation of the TT-XSS framework.


## Overall Paper Summary

The paper presents TT-XSS as a meaningful step forward in automated DOM-XSS detection. Its core insight — that **full taint trace information** (not just source/sink positions) is the key to both accurate detection and automatic verification — distinguishes it from all compared tools. While limitations around second-order inputs and performance remain, the proposed future directions are well-targeted and practically achievable, suggesting a clear path toward a more complete and scalable solution.
</div>
