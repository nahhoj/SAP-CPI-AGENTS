---
name: sap-cpi-groovy-best-practice
description: Best practices for writing, reviewing, or fixing Groovy scripts in SAP Cloud Integration (CPI). Covers mandatory template, Message/XML/JSON APIs, logging, error handling, credentials, SAP static rules, performance, and Groovy 2.4.21/4 compatibility.
---

# SAP CPI Groovy Best Practice

**Sources**: https://pizug.com/cpi-groovy-examples | https://help.sap.com/docs/cloud-integration/ | SAP Official Scripting Guidelines

## 1. Mandatory Script Template

Every CPI Groovy script MUST follow this structure:

```groovy
import com.sap.gateway.ip.core.customdev.util.Message

def Message processData(Message message) {
    // Your logic here
    return message
}
```

**Critical rules**:
- `processData` is the entry point — never rename it
- Always `return message` (forgetting this is the #1 beginner error)
- `messageLogFactory` is auto-injected; don't import/instantiate
- No `static void main(...)` wrapper

---

## 2. Message API Quick Reference

| Task | Method |
|------|--------|
| Read body (String) | `message.getBody(java.lang.String)` |
| Read body (streaming) | `message.getBody(java.io.Reader)` |
| Write body | `message.setBody(value)` |
| Read headers | `message.getHeaders().get("name")` |
| Write header | `message.setHeader("name", value)` |
| Read properties | `message.getProperties().get("name")` |
| Write property | `message.setProperty("name", value)` |
| MPL log | `messageLogFactory.getMessageLog(message)` |
| Log attachment | `messageLog.addAttachmentAsString("label", body, "type")` |
| MPL header property | `messageLog.addCustomHeaderProperty("key", value)` |

**Type casting rule**: Do NOT add `as String` after `getBody(java.lang.String)` — it already returns String. Redundant casting is flagged by SAP.

```groovy
// ✅ Correct
def body = message.getBody(java.lang.String)

// ❌ Avoid — redundant
def body = message.getBody(java.lang.String) as String
```

---

## 3. XML Handling

**⚠️ Import requirements differ by Groovy version**:
- **Groovy 2.4.21**: `XmlSlurper`, `XmlParser` auto-imported via `groovy.util.*`
- **Groovy 4**: Require explicit `import groovy.xml.*` (moved in Groovy 4.x)

### Read XML (XmlSlurper — preferred for reading)

```groovy
import groovy.xml.XmlSlurper

def reader = message.getBody(java.io.Reader)
def xml = new XmlSlurper().parse(reader)
def value = xml.OrderID.text()
```

Use `parse(Reader)`, not `parseText(String)` — avoids extra String allocation.

### Modify XML (XmlParser + XmlNodePrinter)

```groovy
import groovy.xml.XmlParser
import groovy.xml.XmlNodePrinter

def doc = new XmlParser().parse(message.getBody(java.io.Reader))
doc.Order.Status.each { it.value = "PROCESSED" }

def sw = new StringWriter()
new XmlNodePrinter(new PrintWriter(sw)).print(doc)
message.setBody(sw.toString())
```

### Namespace handling

```groovy
import groovy.xml.XmlSlurper

def xml = new XmlSlurper().parse(reader)
xml.declareNamespace(ns: "http://example.com/schema")
def value = xml.'ns:OrderID'.text()
```

---

## 4. JSON Handling

**JSON classes require explicit imports** (unlike XML): `groovy.json.JsonSlurper`, `groovy.json.JsonOutput`.

### Parse JSON

```groovy
import groovy.json.JsonSlurper

def json = new JsonSlurper().parse(message.getBody(java.io.Reader))
def customerId = json.customer?.id?.toString() ?: ""
```

### Build JSON

```groovy
import groovy.json.JsonBuilder

def builder = new JsonBuilder()
builder(data)
message.setBody(builder.toString())
```

**Avoid pretty-printing** (`JsonOutput.prettyPrint()`) for message bodies — use for debug attachments only. Compact output is smaller and SAP-recommended.

---

## 5. MPL Logging (Always null-check first)

### Basic logging

```groovy
def messageLog = messageLogFactory.getMessageLog(message)
if (messageLog != null) {
    messageLog.setStringProperty("Key", "value")
    messageLog.addAttachmentAsString("Label", body, "text/plain")
}
```

### Log-level aware (production-safe)

```groovy
def logLevel = message.getProperties().get("SAP_MessageProcessingLogConfiguration")?.logLevel as String

if (messageLog != null && (logLevel == "DEBUG" || logLevel == "TRACE")) {
    messageLog.addAttachmentAsString("Payload", body, "text/plain")
}
```

### Custom MPL header (searchable in monitoring)

```groovy
if (messageLog != null) {
    def poNumber = message.getHeaders().get("po_number")
    if (poNumber != null) {
        messageLog.addCustomHeaderProperty("po_number", poNumber)
    }
}
```

---

## 6. Error Handling

### Throw descriptive exception

```groovy
if (status != "200") {
    throw new Exception("Upstream returned: ${status}")
}
```

### Silent skip (no error)

```groovy
import com.sap.it.api.exception.IgnoreMessageException

if (condition) throw new IgnoreMessageException()
```

### Extract HTTP error body (Exception Subprocess)

```groovy
def ex = message.getProperties().get("CamelExceptionCaught")
if (ex?.class?.canonicalName == "org.apache.camel.component.ahc.AhcOperationFailedException") {
    message.setProperty("http.ResponseBody", ex.getResponseBody())
    message.setProperty("http.StatusCode", ex.getStatusCode())
}
```

### try/catch template

```groovy
try {
    def body = message.getBody(java.lang.String)
    // ... risky logic ...
} catch (Exception e) {
    if (messageLog != null) {
        messageLog.addAttachmentAsString("Error", e.getMessage(), "text/plain")
    }
    throw new Exception("Failed: ${e.message}", e)
}
```

---

## 7. Secure Parameters (Length Limits)

**Secure Parameter max length varies by platform**:
- **Cloud Foundry**: 4096 characters (including spaces)
- **Neo**: 255 characters (including spaces)

Validate/truncate upstream if source data could exceed limits.

---

## 7a. Credentials (NEVER hardcode)

```groovy
import com.sap.it.api.securestore.SecureStoreService
import com.sap.it.api.ITApiFactory

def credName = message.getProperties().get("credential_name") as String
def svc = ITApiFactory.getService(SecureStoreService.class, null)
def cred = svc.getUserCredential(credName)
def user = cred.getUsername().toString()
def pass = cred.getPassword().toString()

message.setProperty("auth_user", user)  // Store in PROPERTY, not HEADER
message.setProperty("auth_pass", pass)
```

**Security rules**: Never hardcode secrets. Never store credentials in headers (leak to receivers). Externalize via Content Modifier properties.

---

## 8. Logging (Use SLF4J, NOT println)

**Logging best practice**:
- ✅ Use **SLF4J** (Simple Logging Facade for Java)
- ✅ Use `messageLogFactory` for MPL attachments & properties
- ❌ Never use `System.out.println()` / `println()` — not visible in CPI monitoring

**SLF4J recommended** over custom logging frameworks (SAP official guidance).

---

## 8a. Binding Variables (Anti-pattern)

❌ **Avoid script-level binding variables** (variables outside `processData`):

```groovy
// ❌ WRONG — persists across executions, causes state leaks
body = "data"

def Message processData(Message message) { ... }
```

✅ **Always declare inside `processData`**:

```groovy
def Message processData(Message message) {
    def body = "data"  // Scoped, cleaned up after execution
    ...
}
```

**Why**: Binding variables stay in memory until iFlow redeploy → OutOfMemoryError. Also shared between executions, not thread-safe.

---

## 8b. SAP Static Analysis Rules (Mandatory Compliance)

| Rule | ❌ Forbidden | ✅ Use instead |
|------|-------------|----------------|
| **Eval/GroovyShell** | `Eval.me(...)` / `new GroovyShell()` | Native Groovy logic (causes OOM) |
| **External JARs** | `@Grab(...)` / unsupported libraries | Only bundled SAP/Java APIs |
| **Cred in header** | `message.setHeader("Auth", token)` | `message.setProperty(...)` |
| **Return statement** | Missing `return message` | Always return at end |
| **Unused imports** | Duplicate/unused imports | Clean imports list |
| **TimeZone.setDefault** | Changes JVM default TZ tenant-wide | Causes DB connectivity issues |

---

## 9. Performance Rules

| Rule | Why | Alternative |
|------|-----|-------------|
| Large payloads | Use `Reader`, not `String` | `message.getBody(java.io.Reader)` |
| XML reading | `XmlParser` allocates DOM | Use `XmlSlurper` (lazy) |
| Blocking HTTP | `new URL().text` hangs thread | Use CPI HTTP adapter |
| Large loops | Iterating huge XML sets | Use Message Mapping / XSLT |
| String concat in loops | `text += "..."` allocates many objects | Use `StringBuilder.append()` |
| Script variables | Persist across executions, leak state | Declare inside `processData` only |
| Message body pretty-print | Inflates transmitted payload | Compact JSON/XML; pretty-print only for debug |

---

## 10a. Groovy 2.4.21 / Groovy 4 Compatibility

**Primary target**: Groovy 4 / Java 17. Use compatible syntax when reasonable equivalents exist.

| Feature | 2.4.21 | 4 | Recommendation |
|---------|--------|---|-----------------|
| `def` for typing | ✅ | ✅ | Safe everywhere |
| `?.` (safe nav) | ✅ | ✅ | Safe everywhere |
| String GString `"${x}"` | ✅ | ✅ | Safe everywhere |
| `collect`, `each`, `find` | ✅ | ✅ | Safe everywhere |
| `XmlSlurper`, `XmlParser` location | `groovy.util.*` (auto) | `groovy.xml.*` (explicit import) | Import explicitly for Groovy 4 |
| `var` keyword | ❌ | ✅ | **Use `def` instead** |
| `switch` arrow syntax | ❌ | ✅ | **Use classic switch** |
| Records | ❌ | ✅ | Not needed in CPI |
| `String::strip()` | ❌ | ✅ | **Use `trim()` instead** |

**Flag Groovy-4-only features explicitly** as compatibility risks.

---

## 10b. Anti-Patterns (Never Do These)

| Anti-pattern | Problem | Fix |
|---|---|---|
| `message.getBody()` (no type) | Returns `Object` → ClassCastException | `message.getBody(java.lang.String)` |
| `System.out.println()` | Not visible in CPI monitoring | Use `messageLogFactory` MPL logging |
| Catch & discard silently | Hides errors from monitoring | Rethrow with context or `IgnoreMessageException` |
| `new URL("...").text` | Blocking, no timeout, hangs | Use CPI HTTP adapter |
| `Eval.me()` / `GroovyShell` | OOM, static analysis failure | Native Groovy |
| `@Grab(...)` | Not supported in CPI | Use bundled libraries only |
| Hardcoded credentials | Security violation, fails static analysis | `SecureStoreService` |
| Credential in header | Leaks to receiver | Store as property |
| Script-level binding variables | Persist across messages, leak state | Declare inside `processData` |
| Not null-checking `messageLog` | NPE when logging is off | `if (messageLog != null)` |
| Not returning `message` | iFlow crashes | Always `return message` |
| Redundant `as String` | Noise, flagged by SAP | Remove it |
| `parseText(String)` large XML | Extra String allocation | Use `.parse(Reader)` |

---

## 11. Value Mapping API

Access SAP Value Mappings from scripts using the ValueMappingApi:

```groovy
import com.sap.it.api.ITApiFactory
import com.sap.it.api.mapping.ValueMappingApi

def valueMapApi = ITApiFactory.getService(ValueMappingApi.class, null)
def result = valueMapApi.getMappedValue(sourceAgency, sourceId, sourceValue, targetAgency, targetId)
message.setProperty("mappedValue", result)
```

Deploy the value mapping first in the integration package before referencing it.

---

## 12. Custom Header Properties (MPL Search)

Write custom header properties to MPL for business data searchability:

```groovy
def messageLog = messageLogFactory.getMessageLog(message)
if (messageLog != null) {
    def poNumber = message.getHeaders().get("po_number")
    if (poNumber != null) {
        messageLog.addCustomHeaderProperty("po_number", poNumber)
    }
}
```

**Usage**: Search messages in Monitor by custom header (e.g., `po_number = 12345`).

**Standard searchable headers**:
- `SAP_Sender` / `SAP_Receiver` (set via Content Modifier)
- `SAP_ApplicationID` (custom business data)
- `SAP_MessageType` (message type identifier)

---

## 13. Canonical Patterns (Copy-Paste Ready)

**Read body**:
```groovy
def body = message.getBody(java.lang.String)
```

**Parse XML → extract → set property**:
```groovy
import groovy.xml.XmlSlurper

def xml = new XmlSlurper().parse(message.getBody(java.io.Reader))
message.setProperty("OrderID", xml.OrderID.text())
```

**Parse JSON → extract → set property**:
```groovy
import groovy.json.JsonSlurper
def json = new JsonSlurper().parse(message.getBody(java.io.Reader))
message.setProperty("CustomerID", json.customer?.id?.toString() ?: "")
```

**MPL log with null guard**:
```groovy
def messageLog = messageLogFactory.getMessageLog(message)
if (messageLog != null) {
    messageLog.addAttachmentAsString("Label", body, "text/plain")
}
```

**Conditional skip (no error)**:
```groovy
import com.sap.it.api.exception.IgnoreMessageException
if (skip) throw new IgnoreMessageException()
```

**Read credential**:
```groovy
import com.sap.it.api.securestore.SecureStoreService
import com.sap.it.api.ITApiFactory
def cred = ITApiFactory.getService(SecureStoreService.class, null)
    .getUserCredential(message.getProperties().get("cred_name"))
```

**Read URL query params → set properties**:
```groovy
def query = message.getHeaders().get('CamelHttpQuery') as String
query?.tokenize('&')?.collectEntries { it.tokenize('=') }?.each { k, v ->
    message.setProperty(k, URLDecoder.decode(v, 'UTF-8'))
}
```

---

## 14. Agent Output Format (MANDATORY)

When generating/modifying a Groovy script, respond with:

1. **Complete script** — copy-paste ready, no fragments
2. **Import justification** — why each non-obvious import is needed
3. **Compatibility note** — flag any Groovy-4-only or Java-17-only features
4. **Input/output example** — show what script expects & produces
5. **MPL logging note** — confirm null-guard in place, explain what's logged
6. **Anti-pattern check** — verify script avoids all anti-patterns (Section 12)
7. **SAP compliance** — confirm script passes static analysis (no Eval, no unsupported APIs, clean imports)

---

## 15. References

- **Pizug**: https://pizug.com/cpi-groovy-examples | https://github.com/pizug/cpi-groovy-examples
- **SAP Scripting**: https://help.sap.com/docs/cloud-integration/sap-cloud-integration/use-scripting-appropriately
