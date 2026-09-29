---
template:
  id: https://www.modelware.io/sierra/operational-analysis/summary-template
  expose:
    - kind: compose
  params:
    - id: ontology
      defaultValue: "${context.ontology}"
---

### Stakeholder Concerns Overview

```table
PREFIX stakeholder: <https://www.modelware.io/sierra/stakeholder#>

SELECT ?Stakeholder ?Concern
WHERE {
  ?Stakeholder a stakeholder:Stakeholder ;
               stakeholder:expresses ?Concern .
}
ORDER BY ASC(?Stakeholder)
```