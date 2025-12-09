---
trigger: always_on
---

#ABL Syntax
- Always use abl-syntax rules for information on ABL syntax.
- Prefer to define datasets statically.
- Pass datasets by reference to methods when possible.
- Always use the use-widget-pool option when defining a class.
- Consider using tracking-changes when updating data in a temp-table.
- Always make sure that public methods don't need class context. All context should be applied via parameters.
- Always reference database tables using a locally defined named buffer.
- Use the VAR statement instead of DEFINE VARIABLE
- Always use lowercase keywords
