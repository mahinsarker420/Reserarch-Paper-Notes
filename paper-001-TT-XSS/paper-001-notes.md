<div style="text-align: justify;">

# TT-XSS: A Novel Taint-Tracking Dynamic Detection Framework for DOM XSS

- **Authors:** Ran Wang, Guangquan Xu, Xianjiao Zeng, Xiaohong Li, Zhiyong Feng
- **DOI / Link:** https://doi.org/10.1016/j.jpdc.2017.07.006

## How I prepare these notes and how to read them
I first record unfamiliar terminology to reference later. Then I summarize each section and provide clear explanations of the methodology and other key parts of the paper. If you are already familiar with the terminology, you can skip the "Terminology Notes" section and start at "Paper Notes".



# Terminology Notes

## 1. What is DOM-Based XSS?
See: [DOM-Based XSS](../../Cybersecurity-Notes/DOM-Based-XSS.md)

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

## Absract (Summery):

DOM-Based XSS detection method can be divided into three types : black-box fuzzing, static analysis, and dynamic analysis.
- **Black-box fuzzing**: A testing method where random, malformed, or unexpected inputs are sent to an application without knowing its internal code, to discover crashes or vulnerabilities.
- **Static analysis**: A technique that examines source code or program files without running the application, looking for coding errors and potential security weaknesses.
- **Dynamic analysis**: A method that analyzes an application while it is actually running, monitoring its behavior and data flow to detect vulnerabilities during execution.

Black-box fuzzing and static analysis have problems because fuzzing can miss many vulnerabilities (high false negatives), while static analysis may incorrectly report safe code as vulnerable (high false positives). Dynamic analysis usually gives better results because it analyzes the application while it is running, but it is often difficult and expensive to implement. To solve these issues, the authors created a new framework called TT-XSS that uses taint tracking on the client side (inside the browser). Their system modifies JavaScript features and browser DOM APIs so it can track how untrusted data moves through the web page. It marks input sources, follows how the data is transferred, and checks whether it reaches dangerous places (sinks). Using this information, the framework can automatically create attack inputs to verify whether a real vulnerability exists. When compared with another security tool, AWVS 10.0, their framework found 1.8% more vulnerabilities and automatically generated attack payloads for 9.1% of vulnerabilities.

## 1. Introduction (Summary):
This paragraph explains that Cross-Site Scripting (XSS) is one of the major security problems in modern web applications because it can cause serious issues such as stealing user information, violating privacy, and even spreading malicious worms. XSS is considered an important security risk and is often listed among the top web security threats. XSS vulnerabilities are generally divided into three types: Reflected XSS, Stored XSS, and DOM-XSS. **DOM-XSS is different from the other two because its attack process happens inside the browser using JavaScript and the Document Object Model (DOM), so methods used for detecting Reflected and Stored XSS do not work well for it**. As web applications continue to grow and use more client-side JavaScript, DOM-XSS attacks are becoming more common, creating a need for better detection methods. Existing approaches mainly use black-box fuzzing and static analysis, but both have limitations: fuzzing may miss vulnerabilities because it cannot test every possible case, while static analysis struggles with problems like unclear code structure and obfuscated code. Dynamic analysis gives more accurate results but is difficult and expensive to implement. To solve this issue, the authors propose a new dynamic detection framework based on taint tracking, where they introduce new data types and methods to track data flow in the browser, automatically generate attack payloads to verify vulnerabilities, and build a prototype by modifying browser components such as JavaScriptCore and WebKit. The rest of the paper explains related work, the framework design, implementation details, experimental results, and future work.

</div>
</div>