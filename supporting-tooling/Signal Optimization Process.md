**Signal Refinement Process**

This process was used to improve signal coverage, identify contextual variations, and enhance consistency within the Signal Interpretation Layer.

1. **Signal Identification & Sorting**  
   Initial signal indicators were identified and ranked by frequency of occurrence.

2. **Distribution Analysis**  
   Heat map visualization was used to assess signal concentration and distribution.

3. **Weight Distribution Assessment**  
   High-frequency indicators initially represented 70% to 80% of signal identification weight.

4. **Context Expansion**  
   Synonyms, related phrases, and contextual variations were introduced to improve signal coverage.

5. **Signal Balancing**  
   Dominant indicators were reduced to approximately 30% weighting and supported by secondary indicators to improve consistency.

6. **Validation Audit**  
   Standardized decision scenarios were re-audited to assess balance, consistency, and confidence levels.

**Visual Example: Signal Distribution Heat Map**

![](/architecture/keyword-heat-map.png)

**Visual Example: Signal Identification Worksheet**

![](/architecture/keyword-identification-sheet.png)

**Implementation Notes**

- Embedding sample-size calculations and audit-scoring formulas within the same worksheet can significantly impact Excel performance. Separating calculations into dedicated worksheets improves efficiency and maintainability.
- Using manual or partial calculation modes allows formulas to execute only when required, reducing processing overhead during large-scale analysis and auditing activities.

**Outcome**

The process improved:

- Signal coverage
- Context recognition
- Indicator balance
- Decision consistency
- Audit confidence
