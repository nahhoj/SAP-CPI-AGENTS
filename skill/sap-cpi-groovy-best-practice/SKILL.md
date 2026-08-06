# SAP CPI Groovy Best Practice Skill

**Sources this skill is built from:**
- https://pizug.com/cpi-groovy-examples (and individual examples)
- https://help.sap.com/docs/cloud-integration/sap-cloud-integration/use-scripting-appropriately
---

## 1. Purpose & Scope

Apply this skill before generating, reviewing, refactoring, or troubleshooting **any** Groovy script for SAP Cloud Integration (CPI / Integration Suite). It defines:

- The mandatory script structure every CPI Groovy script must follow.
- Proven patterns for the most common tasks (payload, headers, properties, XML, JSON, MPL logging, error handling, credentials).
- SAP official rules that the static analysis engine enforces on deployment.
- Performance, security, and compatibility rules.
- The output format the agent must use when responding.

Do **not** generate or modify Groovy code without reading this skill first.

---

## 2. Mandatory Script Template

Every CPI Groovy script must use this structure. Nothing executes without it.

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import java.util.HashMap

def Message processData(Message message) {
    // Your logic here
    return message
}
```

**Rules:**
- `processData` is the default entry point. It can be renamed in the iFlow Script step UI, but `processData` is the universal convention — never rename it without a documented reason.
- The method **must** receive `Message` and **must** return `Message`. Forgetting the return statement is the single most common beginner error.
- `messageLogFactory` is **injected automatically** by the runtime — do not import or instantiate it.
- `import java.util.HashMap` is included by convention; only include imports you actually use.

---

## 3. Message API Quick Reference

| Goal | Method |
|---|---|
| Read body as String | `message.getBody(java.lang.String) as String` |
| Read body as Reader (streaming) | `message.getBody(java.io.Reader)` |
| Read body as InputStream | `message.getBody(java.io.InputStream)` |
| Write body | `message.setBody(value)` |
| Read all headers (Map) | `message.getHeaders()` |
| Read one header | `message.getHeaders().get("headerName")` |
| Write / overwrite header | `message.setHeader("name", value)` |
| Read all properties (Map) | `message.getProperties()` |
| Read one property | `message.getProperties().get("propName")` |
| Write / overwrite property | `message.setProperty("name", value)` |
| Get MPL message log | `messageLogFactory.getMessageLog(message)` |
| Log payload attachment | `messageLog.addAttachmentAsString("label", body, "text/plain")` |
| Log custom MPL header | `messageLog.addCustomHeaderProperty("key", value)` |
| Set MPL string property | `messageLog.setStringProperty("key", value)` |

**Type casting rule:** Always cast the body result explicitly:
```groovy
// Correct — explicit cast avoids ClassCastException at runtime
def body = message.getBody(java.lang.String) as String

// Also correct — using the class reference directly
def body = message.getBody(String)
```

---

## 4. XML Handling

### Why no `import` for XmlSlurper / XmlParser?

Groovy automatically imports the entire `groovy.util.*` package. `XmlSlurper`, `XmlParser`, `XmlNodePrinter`, and `NodeBuilder` all live there — no explicit import needed.

`groovy.json.*` is **not** auto-imported, which is why `JsonSlurper`, `JsonBuilder`, and `JsonOutput` always require an explicit `import` statement (see section 5).

| Class | Package | Auto-imported? |
|---|---|---|
| `XmlSlurper` | `groovy.util` | ✅ no import needed |
| `XmlParser` | `groovy.util` | ✅ no import needed |
| `XmlNodePrinter` | `groovy.util` | ✅ no import needed |
| `NodeBuilder` | `groovy.util` | ✅ no import needed |
| `JsonSlurper` | `groovy.json` | ❌ must import explicitly |
| `JsonBuilder` | `groovy.json` | ❌ must import explicitly |
| `JsonOutput` | `groovy.json` | ❌ must import explicitly |

Other always-auto-imported packages: `java.lang.*`, `java.util.*`, `java.io.*`, `java.net.*`, `groovy.lang.*`.

---

### 4a. Read XML and extract values (XmlSlurper — recommended for reading)

```groovy
import com.sap.gateway.ip.core.customdev.util.Message

def Message processData(Message message) {
    def body = message.getBody(java.lang.String) as String
    def xml = new XmlSlurper().parseText(body)

    // GPath navigation — dot notation accesses child elements
    def orderId = xml.OrderID.text()
    def itemCount = xml.Items.Item.size()

    message.setProperty("OrderID", orderId)
    message.setProperty("ItemCount", itemCount.toString())
    return message
}
```

**XmlSlurper vs XmlParser:**
- `XmlSlurper` — lazy, returns `GPathResult`; best for reading and navigating. Preferred for large documents.
- `XmlParser` — eager, returns `Node` tree; required when you need to mutate the document and serialize it back.

### 4b. Modify XML and write back (XmlParser + XmlNodePrinter)

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import org.xml.sax.InputSource

def Message processData(Message message) {
    def body = message.getBody(java.io.Reader)
    def doc = new XmlParser().parse(body)

    // Navigate and modify
    doc.Order.Status.each { it.value = "PROCESSED" }

    // Serialize back to String
    def sw = new StringWriter()
    def printer = new XmlNodePrinter(new PrintWriter(sw))
    printer.preserveWhitespace = true
    printer.print(doc)

    message.setBody(sw.toString())
    return message
}
```

### 4c. Merge two XML documents

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import org.xml.sax.InputSource

def Message processData(Message message) {
    def body = message.getBody(java.io.Reader)
    def mainDoc = new XmlParser().parse(body)

    def externalXml = message.getProperties().get("external_xml_data") as String
    def externalDoc = new XmlParser().parse(new InputSource(new StringReader(externalXml)))

    // Wrap external data in a named extension element for clarity
    def nodeBuilder = new NodeBuilder()
    def extension = nodeBuilder.Extension1 {}
    extension.append(externalDoc)
    mainDoc.append(extension)

    def sw = new StringWriter()
    def printer = new XmlNodePrinter(new PrintWriter(sw))
    printer.preserveWhitespace = true
    printer.print(mainDoc)

    message.setBody(sw.toString())
    return message
}
```

### 4d. Namespace handling

Namespaces are a common source of bugs. Declare them explicitly:

```groovy
def xml = new XmlSlurper().parseText(body)
xml.declareNamespace(ns: "http://example.com/schema")

// Access namespaced element
def value = xml.'ns:OrderID'.text()
```

---

## 5. JSON Handling

### 5a. Parse JSON and extract values

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import groovy.json.JsonSlurper

def Message processData(Message message) {
    // Use Reader for better memory efficiency on larger payloads
    def json = new JsonSlurper().parse(message.getBody(java.io.Reader))

    def customerId = json.customer?.id?.toString()
    def orderTotal = json.order?.total?.toString()

    message.setProperty("CustomerID", customerId ?: "")
    message.setProperty("OrderTotal", orderTotal ?: "")
    return message
}
```

### 5b. Build and write JSON

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import groovy.json.JsonBuilder
import groovy.json.JsonOutput

def Message processData(Message message) {
    def headers = message.getHeaders()

    // Build a JSON object from headers
    def builder = new JsonBuilder()
    builder(headers)
    def prettyJson = JsonOutput.prettyPrint(builder.toString())

    message.setBody(prettyJson)
    message.setHeader("Content-Type", "application/json")
    return message
}
```

---

## 6. MPL Logging

### 6a. Basic payload logging

`messageLogFactory` is always available — do not import or declare it.

```groovy
import com.sap.gateway.ip.core.customdev.util.Message

def Message processData(Message message) {
    def body = message.getBody(java.lang.String) as String

    def messageLog = messageLogFactory.getMessageLog(message)
    if (messageLog != null) {
        messageLog.setStringProperty("Logging", "Payload snapshot")
        messageLog.addAttachmentAsString("Payload", body, "text/plain")
    }
    return message
}
```

**Always null-check `messageLog` before using it.** It returns null when the iFlow log level does not produce an MPL entry.

### 6b. Log-level-aware logging (production-safe)

Keep logging steps in the flow without paying the performance cost in production. The iFlow log level is available as a property.

```groovy
import com.sap.gateway.ip.core.customdev.util.Message

def Message processData(Message message) {
    def body = message.getBody(java.lang.String) as String

    def props = message.getProperties()
    def logConfig = props.get("SAP_MessageProcessingLogConfiguration")
    def logLevel = logConfig?.logLevel as String

    def messageLog = messageLogFactory.getMessageLog(message)
    if (messageLog != null) {
        if (logLevel == "DEBUG" || logLevel == "TRACE") {
            messageLog.setStringProperty("Logging", "Debug payload capture")
            messageLog.addAttachmentAsString("Payload", body, "text/plain")
        }
    }
    return message
}
```

### 6c. Custom MPL header property (searchable in monitoring)

Adds business identifiers (PO number, IDoc number, etc.) that are indexed and searchable in the SAP Integration Suite monitoring since January 2021.

```groovy
import com.sap.gateway.ip.core.customdev.util.Message

def Message processData(Message message) {
    def messageLog = messageLogFactory.getMessageLog(message)
    if (messageLog != null) {
        def poNumber = message.getHeaders().get("po_number")
        if (poNumber != null) {
            messageLog.addCustomHeaderProperty("po_number", poNumber)
        }
    }
    return message
}
```

### 6d. Log all HTTP headers (debugging)

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import groovy.json.JsonBuilder
import groovy.json.JsonOutput

def Message processData(Message message) {
    def builder = new JsonBuilder()
    builder(message.getHeaders())
    def prettyJson = JsonOutput.prettyPrint(builder.toString())

    def messageLog = messageLogFactory.getMessageLog(message)
    if (messageLog != null) {
        messageLog.addAttachmentAsString("HTTP Headers", prettyJson, "application/json")
    }
    return message
}
```

---

## 7. Error Handling

### 7a. Custom business exception

Throw a descriptive exception to trigger the iFlow Exception Subprocess with a clear, diagnosable error message.

```groovy
import com.sap.gateway.ip.core.customdev.util.Message

def Message processData(Message message) {
    def status = message.getProperties().get("ResponseStatus") as String
    if (status != "200") {
        throw new Exception("Upstream system returned unexpected status: ${status}")
    }
    return message
}
```

### 7b. Stop processing silently (IgnoreMessageException)

Terminates the iFlow with `Completed` status and no error. Use for conditional skip logic (e.g., duplicate detection, empty payload guard).

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import com.sap.it.api.exception.IgnoreMessageException

def Message processData(Message message) {
    def skip = message.getProperties().get("SkipProcessing") as String
    if (skip == "true") {
        throw new IgnoreMessageException()
    }
    return message
}
```

### 7c. Extract HTTP error body (in Exception Subprocess)

Place this script in an Exception Subprocess to capture the HTTP error response body for logging or routing.

```groovy
import com.sap.gateway.ip.core.customdev.util.Message

// Reference: https://help.sap.com/viewer/368c481cd6954bdfa5d0435479fd4eaf/Cloud/en-US/a443efe1d5d2403fb95ee9def1a672d4.html
def Message processData(Message message) {
    def props = message.getProperties()
    def ex = props.get("CamelExceptionCaught")

    if (ex != null) {
        def className = ex.getClass().getCanonicalName()
        if (className == "org.apache.camel.component.ahc.AhcOperationFailedException") {
            def messageLog = messageLogFactory.getMessageLog(message)
            if (messageLog != null) {
                messageLog.addAttachmentAsString("http.ResponseBody", ex.getResponseBody(), "text/plain")
            }
            message.setProperty("http.ResponseBody", ex.getResponseBody())
            message.setProperty("http.StatusCode", ex.getStatusCode())
            message.setProperty("http.StatusText", ex.getStatusText())
            message.setBody(ex.getResponseBody())
        }
    }
    return message
}
```

### 7d. try/catch template

Use try/catch when calling code that may throw (XML parsing, credential access, type conversions). Always rethrow with context.

```groovy
import com.sap.gateway.ip.core.customdev.util.Message

def Message processData(Message message) {
    try {
        def body = message.getBody(java.lang.String) as String
        // ... risky logic ...
        message.setBody(body)
    } catch (Exception e) {
        // Log the issue before rethrowing so monitoring has detail
        def messageLog = messageLogFactory.getMessageLog(message)
        if (messageLog != null) {
            messageLog.addAttachmentAsString("Error", e.getMessage(), "text/plain")
        }
        throw new Exception("processData failed: ${e.getMessage()}", e)
    }
    return message
}
```

---

## 8. Credentials & Secure Store

**Never hardcode credentials, tokens, API keys, or passwords in a script.** Always retrieve them from the SAP Secure Store.

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import com.sap.it.api.securestore.SecureStoreService
import com.sap.it.api.securestore.UserCredential
import com.sap.it.api.ITApiFactory

def Message processData(Message message) {
    // Externalize the credential name via a Content Modifier property
    // so it can be changed without editing the script
    def credentialName = message.getProperties().get("credential_name") as String

    SecureStoreService secureStoreService = ITApiFactory.getService(SecureStoreService.class, null)
    UserCredential userCredential = secureStoreService.getUserCredential(credentialName)

    def user = userCredential.getUsername().toString()
    def pass = userCredential.getPassword().toString()

    // Store in properties for downstream use — never store in headers
    // (headers can leak to receiver systems)
    message.setProperty("auth_user", user)
    message.setProperty("auth_pass", pass)

    return message
}
```

**Security rules from SAP official docs:**
- Do **not** store credentials in headers — headers can be forwarded to receiver systems accidentally.
- Do **not** log password values via MPL.
- Externalize the credential name using `{{credential_name}}` in a Content Modifier so the script does not need to change per environment.

---

## 9. SAP Official Rules (Static Analysis — Mandatory Compliance)

SAP Cloud Integration runs static analysis on every deployed Groovy script. Violating these rules blocks deployment or causes runtime failures.

### ⛔ Rule 1 — Never use Eval or GroovyShell

Using `Eval` or `GroovyShell` leads to **out-of-memory errors** and is flagged by static analysis.

```groovy
// ❌ FORBIDDEN — causes OOM and static analysis failure
def result = Eval.me("1 + 1")
def shell = new GroovyShell()
shell.evaluate("println 'hello'")

// ✅ Use native Groovy instead
def result = 1 + 1
```

### ⛔ Rule 2 — Never use unsupported external JAR files

Only SAP-certified and natively supported libraries may be used. Uploading unsupported external `.jar` files is not recommended.

```groovy
// ❌ FORBIDDEN
@Grab(group='commons-lang', module='commons-lang', version='2.6')
// @Grab is not supported in CPI at all — remove it

// ✅ Use only native CPI SDK and bundled Java/Groovy APIs
```

### ⛔ Rule 3 — Never access secure parameters and assign to headers

Accessing a credential and storing it as a header is a security violation caught by static analysis.

```groovy
// ❌ FORBIDDEN — credential in header leaks to receiver system
message.setHeader("Authorization", "Bearer " + token)

// ✅ Store in property, use in a downstream Content Modifier or adapter config
message.setProperty("auth_token", token)
```

### ⛔ Rule 4 — Scripts must pass all static analysis checks

Static analysis checks for: defects, bad practices, inconsistencies, style issues. Scripts that fail static analysis will not deploy. Common causes:
- Missing return statement
- Unused imports
- Unreachable code
- Deprecated API usage

---

## 10. Performance Rules

| Rule | Reason |
|---|---|
| Prefer `message.getBody(java.io.Reader)` over `String` for large payloads | Avoids loading entire payload into JVM heap |
| Use `XmlSlurper` over `XmlParser` when only reading | XmlSlurper is lazy and more memory-efficient |
| Never use blocking HTTP calls inside a script (`new URL(...).text`) | Blocks the thread, no timeout, no connection pool |
| Avoid loops over large XML node sets in a single script | Common bottleneck — use Message Mapping or XSLT for bulk transformations |
| Use standard CPI components (Value Mapping, Message Mapping, Router) when they fit | Script steps have higher overhead than native steps |
| Check payload size before processing when the input is unbounded | Use `message.getHeaders().get("Content-Length")` or body size check to guard against huge payloads |
| Avoid script-level (binding) variables | They persist across message executions and cause hard-to-find state leaks |

---

## 11. Groovy 2.4.21 / Groovy 4 Compatibility Cheatsheet

The primary target is Groovy 4 / Java 17. Prefer syntax that also runs on Groovy 2.4.21 / Java 8 so scripts work on classic-engine tenants.

| Feature | Groovy 2.4.21 compatible | Groovy 4 only | Recommendation |
|---|---|---|---|
| `def` for dynamic typing | ✅ | ✅ | Use `def` — works everywhere |
| Safe navigation `?.` | ✅ | ✅ | Safe to use |
| String GString `"${var}"` | ✅ | ✅ | Safe to use |
| `as String` cast | ✅ | ✅ | Prefer over `(String)` cast |
| `collect`, `inject`, `each`, `find` | ✅ | ✅ | Safe to use |
| `@TypeChecked` / `@CompileStatic` | ✅ (limited) | ✅ | Avoid — causes issues with CPI runtime binding |
| `var` keyword | ❌ | ✅ | **Use `def` instead** for compatibility |
| Records (`record`) | ❌ | ✅ | Not needed in CPI scripts |
| `switch` expressions (arrow syntax) | ❌ | ✅ | **Use classic `switch` or `if/else`** for compatibility |
| `java.util.stream` Stream API | ⚠️ (Java 8 only) | ✅ | Prefer Groovy GDK collections — works on both |
| `String::strip()` | ❌ (Java 8) | ✅ (Java 11+) | **Use `String::trim()` instead** for compatibility |
| Text blocks `"""..."""` | ✅ (Groovy GString) | ✅ | Safe — Groovy multi-line strings work on both |

**When you use a Groovy-4-only or Java-17-only feature, flag it explicitly in your response as a compatibility risk.**

---

## 12. Anti-Patterns — Never Do These

| Anti-pattern | Why it's wrong | What to do instead |
|---|---|---|
| `message.getBody()` without type argument | Returns `Object` — causes `ClassCastException` downstream | `message.getBody(java.lang.String) as String` |
| `System.out.println(...)` | Output is not visible in CPI monitoring | Use `messageLogFactory` MPL logging |
| Catch `Exception` and return `null` or silently discard | Hides errors; makes diagnosis in monitoring impossible | Rethrow with context or use `IgnoreMessageException` for intentional skips |
| `new URL("...").text` | Blocking call with no timeout; hangs the thread | Use a CPI HTTP adapter instead |
| `Eval.me(...)` or `new GroovyShell()` | Out-of-memory, static analysis failure, not supported | Native Groovy logic |
| `@Grab(...)` annotations | Not supported in CPI runtime | Use only bundled libraries |
| Hardcoded credentials (`"password123"`) | Security violation, static analysis failure | `SecureStoreService` via `ITApiFactory` |
| Credential stored as header | Leaks to receiver system | Store as property |
| Script-level (binding) variables outside the method | Persist across message executions; cause state leaks | Declare all variables inside `processData` |
| Modifying a collection while iterating it | `ConcurrentModificationException` | Collect to a new list, then modify |
| Not null-checking `messageLog` | NPE when log level doesn't generate an MPL entry | Always `if (messageLog != null)` |
| Not returning `message` | CPI receives `null`, iFlow crashes with a cryptic error | Always `return message` at the end |

---

## 13. Canonical Patterns Cheatsheet

Copy-paste ready patterns for the most frequent tasks.

### Read body + return unchanged
```groovy
import com.sap.gateway.ip.core.customdev.util.Message

def Message processData(Message message) {
    def body = message.getBody(java.lang.String) as String
    // ... use body ...
    return message
}
```

### Read property → set property
```groovy
def orderId = message.getProperties().get("OrderID") as String
message.setProperty("ProcessedOrderID", orderId + "-OK")
```

### Read header → set header
```groovy
def ct = message.getHeaders().get("Content-Type") as String
message.setHeader("X-Original-Content-Type", ct)
```

### Parse XML → extract field → set property
```groovy
def xml = new XmlSlurper().parseText(message.getBody(java.lang.String) as String)
message.setProperty("OrderID", xml.OrderID.text())
```

### Parse JSON → extract field → set property
```groovy
import groovy.json.JsonSlurper
def json = new JsonSlurper().parse(message.getBody(java.io.Reader))
message.setProperty("CustomerID", json.customer?.id?.toString() ?: "")
```

### MPL log + null guard
```groovy
def messageLog = messageLogFactory.getMessageLog(message)
if (messageLog != null) {
    messageLog.addAttachmentAsString("Label", body, "text/plain")
}
```

### Conditional skip (no error)
```groovy
import com.sap.it.api.exception.IgnoreMessageException
if (condition) throw new IgnoreMessageException()
```

### Read credential
```groovy
import com.sap.it.api.securestore.SecureStoreService
import com.sap.it.api.securestore.UserCredential
import com.sap.it.api.ITApiFactory
def credName = message.getProperties().get("credential_name") as String
SecureStoreService svc = ITApiFactory.getService(SecureStoreService.class, null)
UserCredential cred = svc.getUserCredential(credName)
def user = cred.getUsername().toString()
def pass = cred.getPassword().toString()
```

---

## 14. Agent Output Format (MANDATORY)

When generating or modifying a Groovy script, the agent response must include:

1. **Complete script** — never partial snippets. The output must be copy-paste ready.
2. **Import list justification** — one line per non-obvious import explaining why it's needed.
3. **Compatibility note** — explicitly flag any Groovy-4-only or Java-17-only construct with a warning like: `⚠️ Compatibility: This uses [feature] which requires Groovy 4 / Java 11+. Replace with [alternative] if your tenant is on the classic engine.`
4. **Input/output example** — show what the script expects as input and what it produces.
5. **MPL logging note** — if the script logs to MPL, confirm the null-guard is in place and explain what is logged.
6. **Anti-pattern check** — before finalizing, verify the script does not contain any item from section 12. State the check was done.

---

## 15. References

- Pizug CPI Groovy Examples: https://pizug.com/cpi-groovy-examples
- Pizug GitHub: https://github.com/pizug/cpi-groovy-examples
- SAP Use Scripting Appropriately: https://help.sap.com/docs/cloud-integration/sap-cloud-integration/use-scripting-appropriately
- SAP Script API Methods: https://help.sap.com/docs/cloud-integration/sap-cloud-integration/using-script-api-methods-in-groovy-scripts
- SAP Static Analysis of Groovy Scripts: https://help.sap.com/docs/cloud-integration/sap-cloud-integration/static-analysis-of-groovy-scripts
- SAP Avoid Using Eval Classes: https://help.sap.com/docs/cloud-integration/sap-cloud-integration/avoid-using-eval-classes
- SAP Setting Log Levels: https://help.sap.com/viewer/368c481cd6954bdfa5d0435479fd4eaf/Cloud/en-US/4e6d3fc3f34544f6ac5289268b653ad1.html
- SAP HTTP Error Body Pattern: https://help.sap.com/viewer/368c481cd6954bdfa5d0435479fd4eaf/Cloud/en-US/a443efe1d5d2403fb95ee9def1a672d4.html
- SecureStoreService Javadoc: https://help.sap.com/doc/471310fc71c94c2d913884e2ff1b4039/Cloud/en-US/com/sap/it/api/securestore/SecureStoreService.html
