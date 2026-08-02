---
name: sap-cpi-groovy-best-practice
description: Production-grade best practices for SAP Cloud Integration Groovy development.
---

# SAP CPI Groovy Best Practice

## Mandatory Workflow

1. Understand the user's functional requirement.
2. Apply SAP Cloud Integration best practices.
3. Consult the Pizug repositories and SAP Help documentation for relevant implementation patterns, and perform a deep dive into the following resources
   - https://pizug.com/cpi-groovy-examples
   - https://github.com/pizug/cpi-groovy-examples
   - https://help.sap.com/docs/cloud-integration/sap-cloud-integration/use-scripting-appropriately?q=groovy
4. Adapt examples to the user's scenario.
5. Never copy repository examples verbatim.
6. Generate production-ready code.

## Mandatory Rules

- Target SAP Cloud Integration.
- Primary runtime target: Groovy 4 / Java 17 APIs.
- Maintain backward compatibility with Groovy 2.4.21 / Java 8: whenever a Groovy 4 feature and a Groovy 2.4.21-compatible equivalent both solve the problem, prefer the one that works on both.
- Only use a Groovy-4-only or Java-17-only construct when there is no reasonable 2.4.21-compatible equivalent, and explicitly flag it in the "Best Practices Applied" section of the output as a compatibility risk for tenants still on the classic engine.
- If the user explicitly confirms their tenant is still on the classic engine only (Groovy 2.4.21 / Java 8, no Groovy 4), drop the Groovy 4 target for that session and generate strictly 2.4.21/Java 8 compatible code.
- Prefer standard Integration Flow steps over Groovy.
- Use SAP public APIs only.
- Avoid unsupported Java/Groovy features.

## XML

- Prefer XmlSlurper for read-only parsing.
- Prefer XmlParser only when modifying XML.
- Prefer Reader/InputStream.
- Avoid parseText() for large payloads.

## JSON

- Use JsonSlurper.
- Prefer Reader for large payloads.

## Performance

- Minimize memory.
- Avoid binding variables.
- Use StringBuilder.
- Stream large payloads.
- Remove dead code.

## Security

Never expose:
- Passwords
- OAuth tokens
- JWT
- Cookies
- Authorization headers

Never use:
- System.exit()
- Runtime.exec()
- ProcessBuilder
- Eval()
- Reflection

## Logging

- Log only when required.
- Never log credentials.
- Never log large payloads unless requested.

## Code Review Checklist

Verify:
- Compatibility
- Imports
- Memory usage
- Namespace handling
- Exception handling
- Logging
- Security
- Performance
- Readability

## Output Format

Return:

1. Analysis
2. Problems
3. Solution
4. Complete Groovy Script
5. Explanation
6. Best Practices Applied

## Reference Material

Primary guidance:
- sap-cpi-groovy-best-practice

Reference repositories:
- https://pizug.com/cpi-groovy-examples
- https://github.com/pizug/cpi-groovy-examples

Best Practice SAP
- https://help.sap.com/docs/cloud-integration/sap-cloud-integration/use-scripting-appropriately?q=groovy

Rules:
- Reuse patterns.
- Improve implementations.
- Adapt to the user's requirements.
- Follow SAP Cloud Integration best practices.

## Compatibility Matrix

Platform:
- SAP Cloud Integration

Script Runtime Target (primary):
- Groovy 4 / Java 17 APIs

Backward Compatibility Requirement:
- Groovy 2.4.21 / Java 8 — prefer equivalents that work here too whenever reasonable.

Local Runner JVM:
- Java 17 (matches the primary target, so local tests are representative)

Note: Only drop the Groovy 4 target entirely if the user explicitly confirms their SAP CPI tenant is still on the classic engine (Groovy 2.4.21 / Java 8 only).

## Final Validation

Before returning code verify:

✓ Syntax
✓ Imports
✓ SAP APIs
✓ Groovy compatibility
✓ Memory usage
✓ Security
✓ Logging
✓ Readability
✓ Production readiness
