---
template:
  id: https://www.modelware.io/sierra/system-analysis/hazards
  name: "Hazards"
  rank: 0
  expose:
    - kind: compose
  params:
    - id: ontology
      type: iri
      defaultValue: ${context.ontology}
      required: true
---

# System Hazards

Identify system hazards, affected components, and safety mitigations:

```table-editor
---
columns: { this: { label: "Hazard" } }
stylesheet:
  - selector: cell[col === "Priority"  && value]
    target: value
    style:
      padding: 4px 12px
      border-radius: 999px
      font-size: 12px
      font-weight: 600
      color: "#ffffff"
  - selector: cell[value === "High"]
    target: value
    style:
      background-color: "#DC2626"
  - selector: cell[value === "Medium"]
    target: value
    style:
      background-color: "#F59E0B"
  - selector: cell[value === "Low"]
    target: value
    style:
      background-color: "#10B981"
---

# SHACL Namespace Prefix Definitions
@prefix sh: <http://www.w3.org/ns/shacl#> .
@prefix dash: <http://datashapes.org/dash#> .
@prefix base: <https://www.modelware.io/sierra/base#> .
@prefix component: <https://www.modelware.io/sierra/component#> .
@prefix hazards_vocab: <https://fireforce6.github.io/mission-control/system-analysis/hazards_vocab#> .

# SHACL Shape defining property columns displayed in the UI table
hazards_vocab:HazardShape
    a sh:NodeShape ;
    sh:targetClass hazards_vocab:Hazard ;
    

    # Priority property column
    sh:property [
        sh:path base:priority ;
        sh:name "Priority" ;
        sh:maxCount 1 ;
    ] ;
    

    # Affected Components relation column
    sh:property [
        sh:path hazards_vocab:affects ;
        sh:name "Affected Components" ;
        sh:class component:Component ;
    ] ;
    

    # Mitigations relation column
    sh:property [
        sh:path hazards_vocab:mitigatedBy ;
        sh:name "Mitigations" ;
        sh:class component:Component ;
    ] ;
    
    
    # Description property column with text area editor input
    sh:property [
        sh:path base:description ;
        sh:name "Description" ;
        dash:editor dash:TextAreaEditor ;
        sh:maxCount 1 ;
    ] ;
    .
```