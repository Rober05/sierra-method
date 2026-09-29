---
ontology: https://fireforce6.github.io/mission-control/bundle
---

# Fire Force Mission Control — Analysis & Systems Audit Notebook

## Part 1 & 2: Architectural Queries and Views

### 1. Mass Threshold Conformance Query
Identifies components that exceed the methodological mass threshold (5.0 kg).

```table
PREFIX component: <https://www.modelware.io/sierra/component#>
PREFIX oml: <http://opencaesar.io/oml#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>

SELECT ?part ?massInKg ?maxAllowedKg
WHERE {
  ?part a component:PhysicalPart .
  {
    ?part component:mass ?q .
    ?q oml:value ?v ;
       oml:unit ?u .
    ?u oml:multiplier ?m .
    BIND(xsd:decimal(?v) * xsd:decimal(?m) AS ?massInKg)
  } UNION {
    ?part component:mass ?raw .
    FILTER(isLiteral(?raw))
    BIND(xsd:decimal(STR(?raw)) AS ?val)
    BIND(DATATYPE(?raw) AS ?dt)
    BIND(IF(?dt = <http://opencaesar.io/si/g>, ?val / 1000.0, ?val) AS ?massInKg)
  }
  BIND(5.0 AS ?maxAllowedKg)
  FILTER(?massInKg > ?maxAllowedKg)
}
ORDER BY DESC(?massInKg)
```

### 2. Mass Capacity Near-Miss Analysis
Identifies physical parts operating at or near their operational limits (>= 25% of 15.0 kg threshold).

```table
PREFIX component: <https://www.modelware.io/sierra/component#>
PREFIX oml: <http://opencaesar.io/oml#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>

SELECT ?part ?massInKg ?percentageOfLimit
WHERE {
  ?part a component:PhysicalPart .
  {
    ?part component:mass ?q .
    ?q oml:value ?v ;
       oml:unit ?u .
    ?u oml:multiplier ?m .
    BIND(xsd:decimal(?v) * xsd:decimal(?m) AS ?massInKg)
  } UNION {
    ?part component:mass ?raw .
    FILTER(isLiteral(?raw))
    BIND(xsd:decimal(STR(?raw)) AS ?val)
    BIND(DATATYPE(?raw) AS ?dt)
    BIND(IF(?dt = <http://opencaesar.io/si/g>, ?val / 1000.0, ?val) AS ?massInKg)
  }
  BIND(15.0 AS ?maxAllowedKg)
  BIND((?massInKg / ?maxAllowedKg) * 100.0 AS ?percentageOfLimit)
  FILTER(?percentageOfLimit >= 25.0 && ?percentageOfLimit <= 100.0)
}
ORDER BY DESC(?percentageOfLimit)
```

### 3. Orphan Requirements Audit
Detects missing structural relationships where a Requirement or Concern has no declared stating Stakeholder.

```table
PREFIX stakeholder: <https://www.modelware.io/sierra/stakeholder#>
PREFIX requirement: <https://www.modelware.io/sierra/requirement#>
PREFIX base: <https://www.modelware.io/sierra/base#>

SELECT ?req ?description ?priority
WHERE {
  { ?req a stakeholder:Concern }
  UNION
  { ?req a stakeholder:Requirement }
  UNION
  { ?req a requirement:Requirement }

  OPTIONAL { ?req base:description ?description }
  OPTIONAL { ?req base:priority ?priority }
  OPTIONAL { ?req requirement:description ?description }

  FILTER NOT EXISTS {
    { ?req stakeholder:isStatedBy ?stater }
    UNION { ?stater stakeholder:expresses ?req }
    UNION { ?req stakeholder:isExpressedBy ?stater }
    UNION { ?stater stakeholder:states ?req }
    UNION { ?req requirement:isStatedBy ?stater }
    UNION { ?req requirement:isSpecifiedBy ?stater }
    UNION { ?stater requirement:specifies ?req }
    UNION {
      ?req requirement:refines ?concern .
      ?concern stakeholder:isExpressedBy ?stater .
    }
  }
}
ORDER BY DESC(?priority)
```

### 4. Mission-to-Stakeholder Coverage Matrix
Generates a full cross-product matrix rendering unmapped coverage relationships as explicit zero values `0`.

```matrix
---
rowColumnLabel: Mission / Stakeholder
stylesheet:
  - selector: cell [Number(value) > 1]
    style:
      background-color: lightgreen
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX stakeholder: <https://www.modelware.io/sierra/stakeholder#>

SELECT ?row ?column (COALESCE(?n, 0) AS ?value)
WHERE {
  ?row a mission:Mission .
  ?column a stakeholder:Stakeholder .

  OPTIONAL {
    SELECT ?row ?column (COUNT(*) AS ?n)
    WHERE {
      ?row a mission:Mission .
      ?row mission:pursues ?o .
      ?o mission:isDerivedFrom ?c .
      ?c stakeholder:isExpressedBy ?column .
    }
    GROUP BY ?row ?column
  }
}
ORDER BY ?row ?column
```

### 5. Derived Analysis Graph (Shared Objective Stakeholder Topology)
Constructs a derived graph linking stakeholder concerns that share derived mission objectives.

```graph
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX stakeholder: <https://www.modelware.io/sierra/stakeholder#>
PREFIX analysis: <https://www.modelware.io/sierra/analysis#>

CONSTRUCT {
  ?c1 analysis:sharesObjectiveWith ?c2 .
}
WHERE {
  ?o a mission:Objective ;
     mission:isDerivedFrom ?c1 ;
     mission:isDerivedFrom ?c2 .
  FILTER(STR(?c1) < STR(?c2))
}
```

---

## Part 3: Scripted Mass Rollup Pipeline

```javascript
const sparqlQuery = `
PREFIX component: <https://www.modelware.io/sierra/component#>
PREFIX oml: <http://opencaesar.io/oml#>

SELECT ?part ?val ?unit
WHERE {
  ?part a component:PhysicalPart ;
        component:mass ?qty .
  ?qty oml:value ?val .
  OPTIONAL { ?qty oml:unit ?unit }
}
`;

try {
  const rawResults = await query(sparqlQuery);
  
  // Extract rows array from custom notebook response format
  let rows = [];
  if (rawResults && Array.isArray(rawResults.rows)) {
    rows = rawResults.rows;
  } else if (Array.isArray(rawResults)) {
    rows = rawResults;
  } else if (rawResults && rawResults.results && Array.isArray(rawResults.results.bindings)) {
    rows = rawResults.results.bindings;
  }

  const masses = [];
  for (const row of rows) {
    const valStr = typeof row.val === 'object' ? row.val.value : row.val;
    const unitStr = typeof row.unit === 'object' ? row.unit.value : (row.unit || '');
    
    let num = parseFloat(valStr);
    if (!isNaN(num)) {
      // Convert kilograms to grams if specified in unit URI
      if (unitStr.toLowerCase().includes('kg')) {
        num *= 1000.0;
      }
      masses.push(num);
    }
  }

  // Calculate rollup statistics
  const count = masses.length;
  const total = count > 0 ? masses.reduce((a, b) => a + b, 0) : 0;
  const mean = count > 0 ? total / count : 0;
  const max = count > 0 ? Math.max(...masses) : 0;

  // Render summary UI card
  display(`
    <div style="padding: 16px; border: 1px solid #ccc; border-radius: 6px; background-color: #f9f9f9; font-family: sans-serif;">
      <h4 style="margin-top: 0; color: #444; margin-bottom: 12px;">Physical Parts Mass Rollup Summary</h4>
      <ul style="list-style-type: disc; padding-left: 20px; color: #333; line-height: 1.6; margin: 0;">
        <li><b>Total Identified Parts:</b> ${count}</li>
        <li><b>Total Mass:</b> ${total.toFixed(2)} g (${(total / 1000).toFixed(2)} kg)</li>
        <li><b>Average Part Mass:</b> ${mean.toFixed(2)} g</li>
        <li><b>Heaviest Component:</b> ${max.toFixed(2)} g</li>
      </ul>
    </div>
  `);
} catch (err) {
  display(`<div style="color: red; padding: 10px;">Pipeline Error: ${err.message}</div>`);
}
```

---

## Part 4: Method Template Instantiation

```compose-template
template: <https://www.modelware.io/sierra/operational-analysis/summary-template>
```

---

## Part 5: Operational Dashboard Analysis

### Engineering Question
> **Which Stakeholders lack full traceability coverage to operational Missions in the Fire Force architecture?**

### Live Evidence
```matrix
---
rowColumnLabel: Mission / Stakeholder
stylesheet:
  - selector: cell [Number(value) > 1]
    style:
      background-color: lightgreen
---
PREFIX mission: <https://www.modelware.io/sierra/mission#>
PREFIX stakeholder: <https://www.modelware.io/sierra/stakeholder#>

SELECT ?row ?column (COALESCE(?n, 0) AS ?value)
WHERE {
  ?row a mission:Mission .
  ?column a stakeholder:Stakeholder .

  OPTIONAL {
    SELECT ?row ?column (COUNT(*) AS ?n)
    WHERE {
      ?row a mission:Mission .
      ?row mission:pursues ?o .
      ?o mission:isDerivedFrom ?c .
      ?c stakeholder:isExpressedBy ?column .
    }
    GROUP BY ?row ?column
  }
}
ORDER BY ?row ?column
```

### Interpretation
> The matrix above exposes gaps in traceability between operational missions and key project stakeholders. Red cells highlighted with explicit `0` values indicate stakeholder concerns that have no pursuing mission defined in the ABox. For example, if a stakeholder shows a `0` under a mission column, it signifies that their expressed concerns are currently unaddressed by that mission's scope, requiring either a new objective assertion or a scope refinement.

---

## Part 6: Reflection — Ontological Domain Truths vs. Closed-World Query Analysis

1. **Ontology (OML) vs. Validation (SHACL) vs. Querying (SPARQL):**
   - **OML Vocabulary** defines invariant domain concepts, relationships, and taxonomies (e.g., establishing that a `PhysicalPart` is a `Component`).
   - **SHACL Shapes** validate graph constraints and structural completeness during model authoring (e.g., asserting that every `Requirement` must explicitly reference a stating `Stakeholder`).
   - **SPARQL Analysis** operates globally over the instance population, performing calculations, gap detection, mass rollups, and generating analytical views across valid model instances.

2. **Open World Assumption (OWA) vs. Closed World Assumption (CWA):**
   - OWL/OML reasoning functions under the **Open World Assumption**: missing assertions indicate unknown information rather than facts deemed to be false.
   - SPARQL querying applies **Closed World** semantics over the bounded dataset loaded in the ABox. Constructive operators such as `FILTER NOT EXISTS` and `COALESCE(?n, 0)` convert absent assertions into explicit zero values and orphan flags, turning missing model links into concrete engineering findings.