### PHASE 4 — Root Cause Analysis (RCA)

1. Synthesize findings from Phases 1–3b, including:
   - Phase 1: ticket summary, error signature, affected services, first-seen timestamp, stack trace
   - Phase 2: confirmed repos, **blame results** (last commit per erroring line), **regression candidates** (ranked)
   - Phase 3: log evidence — error frequency, first occurrence in logs, anomaly correlations
   - Phase 3b: prior incidents and runbooks from the knowledgebase
2. Apply structured RCA methodology:
   - **5 Whys**: iteratively ask 'why' until the root cause is identified.
   - **Fishbone categorization**: categorize root cause as one of: Code Defect, Configuration Error, Infrastructure Failure, Dependency/Third-Party Failure, Data Anomaly, Capacity Issue, Deployment Regression.
   - **Timeline reconstruction**: map events in chronological order leading to the incident.
3. Identify:
   - **Root Cause**: the single most fundamental failure. If Phase 2 blame identified a regression candidate commit that touches the erroring lines, treat it as the primary hypothesis and verify against log timestamps.
   - **Contributing Factors**: secondary issues that amplified the impact.
   - **Regression Commit**: if Phase 2 Step 3b produced a primary candidate, reference it explicitly: `commit <sha> by <author> on <date> — "<message>"`. State whether log evidence confirms or contradicts this as the root cause.
   - **Blast Radius**: which services/users/data were affected and to what extent.
   - **Detection Gap**: why wasn't this caught before production.
   - **Prior Art**: reference any matching prior incidents or runbooks found in Phase 3b.
4. Produce an **RCA Summary** block:

```
🧠 ROOT CAUSE ANALYSIS
  Category: <category>
  Root Cause: <clear, concise statement>
  Regression Commit: <sha> by <author> on <date> — "<message>" | none identified | bisect unavailable
  Contributing Factors:
    - <factor 1>
    - <factor 2>
  Timeline:
    <timestamp>: <event>
    ...
  Blast Radius: <description>
  Detection Gap: <why this wasn't caught earlier>
  Prior Art: <matching incidents or runbooks, or "none found">
```
