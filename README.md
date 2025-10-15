# GitHub Copilot + SAP Engineering Integration (Senior Engineer Playbook)

Audience: SAP extension engineers, platform leads, DevOps/security owners.  
Objective: Embed GitHub Copilot (Inline, Chat, Enterprise KB, Extensions) as an accelerator in SAP side‑by‑side and modernization workflows (CAP, UI5/Fiori, ABAP transition, CI/CD, security hardening, runtime insights).

---

## 0. Core Principle
Treat Copilot as a programmable acceleration layer—not a code generator to copy blindly. Every interaction: (Context + Constraint + Intent) ⇒ Deterministic refinement loop (Review → Adjust → Commit).

---

## 1. Domain Surface Areas & Copilot Leverage Points

| Domain Layer | Tech Focus | Copilot Output Types | Guardrails |
|--------------|-----------|----------------------|------------|
| Data Model | CDS (entities, projections, annotations) | Entity scaffolding, validation annotations, draft enablement | Verify cardinality & naming conventions |
| Service Logic | CAP Handlers (Node/TypeScript) | CRUD augmentations, actions/functions, transactional patterns | Enforce input validation, error mapping |
| UI Layer | SAPUI5 / Fiori Elements | XML views, controller stubs, action handlers, OPA tests | Review accessibility & i18n usage |
| Integration | OData / REST / Event Mesh | Typed client wrappers, error retry/backoff | Avoid leaking credentials in prompts |
| Modernization | ABAP → CAP | Explanations, refactor proposals, side‑by‑side function mapping | Benchmark logic equivalence |
| DevOps | GitHub Actions | Multi-stage pipelines, CF/Kyma deploy jobs, artifact caching | Pin action versions |
| Security & Quality | CodeQL, secret scanning, tests | Test suite expansion, CodeQL query tuning, lint rule fillers | Reject insecure patterns (eval, raw SQL) |
| Knowledge Injection | KB + Architectural ADRs | Domain-aware suggestions (naming, logging, roles) | Keep KB docs atomic, non-ambiguous |
| Agentic Augmentation | Chat Extension | Live telemetry summarization, risk scoring enrichment | Keep commands idempotent & fast (<800ms) |

---

## 2. Reference Architecture (Execution View)

Component responsibilities:
1. Repo Grouping:
   - cap-pr-service (core business logic)
   - ui5-pr-app (UI channel)
   - governance-docs (KB seed: naming, logging, security, performance)
   - agent-microservices (Chat extension APIs)
2. Cross-Cutting:
   - Shared ESLint + Prettier config
   - Reusable CodeQL config
   - GitHub environment secrets: CF_* / KYMA_* / SBSS_* (segmented by environment)
3. Observability:
   - Optional: Add lightweight telemetry middleware (response time histogram) for Copilot-driven performance prompts.

---

## 3. Knowledge Base Engineering

Structure (flat, rule-first):
```
governance-docs/
  naming.md                # Prefixes, entity casing, action verbs
  logging.md               # Log levels, correlation id pattern
  security.md              # Input validation, PII redaction table
  performance.md           # CAP caching, index hints, batch recommendations
  modernization-map.md     # ABAP object class → CAP construct mapping
  glossary.md              # Domain terms canonical spellings
```
Guideline Authoring Rules:
- One requirement per bullet (immutable phrasing).
- Provide both “Anti-pattern” and “Preferred” snippet pairs.
- Avoid soft verbs (“should”); use “MUST”, “MUST NOT”.

Validation Loop:
1. Baseline prompt (no KB) → capture output.
2. Enable KB → identical prompt → diff for rule adherence.
3. Iterate ambiguous rules until deterministic adoption is ≥90%.

---

## 4. Prompt Engineering Recipes (Deterministic Patterns)

| Pattern | Template |
|---------|----------|
| Model Scaffold | Task: Generate CDS entity for X. Context: <table spec>. Constraints: Must use prefix PR_, default status=PENDING, amount>0 validation annotation or handler. Output: Only CDS code block. |
| Refactor | Task: Refactor handler to isolate validation. Context: (paste file). Constraints: Keep API signature, add JSDoc, no change to DB schema. Output: Unified diff. |
| Test Expansion | Task: Extend tests to cover boundary + error path. Context: current test file + target coverage (90% branches). Output: Additional Jest cases only. |
| ABAP Translation | Task: Explain ABAP snippet, produce equivalent CAP action handler. Context: (ABAP). Constraints: Parameterized queries, reject negative amount. Output: Explanation + handler code. |
| Security Review | Task: Perform security review. Context: handler code. Constraints: List issues + patched code. Output: Table + diff. |
| Performance Hypothesis | Task: Suggest top 3 optimizations. Context: Logs (RT, DB counts). Constraints: Only actionable CAP config/SQL/index proposals. Output: Bulleted list. |

---

## 5. CAP Implementation Blueprint

Minimal CDS with domain naming:
```cds
namespace pr;

@Capabilities.Insertable : true
@Capabilities.Updateable : true
entity PR_Requisition {
  key ID               : UUID;
  description          : String(255);
  amount               : Decimal(15,2);
  currency             : String(3);
  vendorId             : String(20);
  approvalStatus       : String(15) default 'PENDING'; // PENDING|REVIEW_REQUIRED|APPROVED|REJECTED
  highValueFlag        : Boolean default false;
  riskScore            : Integer default 0;
  createdAt            : Timestamp;
  updatedAt            : Timestamp;
}
```

Service with action:
```cds
service PRService {
  entity PR_Requisition as projection on pr.PR_Requisition;
  action Approve(id: UUID) returns Boolean;
  action BulkReview(threshold: Decimal(15,2)) returns Integer;
}
```

Handler patterns (focus on separation of concerns):
```javascript
// srv/pr-service.js
const cds = require('@sap/cds');

module.exports = cds.service.impl(function() {
  const { PR_Requisition } = this.entities;

  this.before(['CREATE','UPDATE'], PR_Requisition, enforceInput);
  this.before('CREATE', PR_Requisition, deriveHighValue);
  this.before('CREATE', PR_Requisition, setTimestamps);
  this.before('UPDATE', PR_Requisition, setUpdated);
  this.on('Approve', approveSingle);
  this.on('BulkReview', bulkReview);
});

function enforceInput(req){
  const { amount, description } = req.data;
  if (amount == null || amount <= 0) req.error(400, 'Amount must be > 0');
  if (!description) req.error(400, 'Description required');
}

function deriveHighValue(req){
  if (req.data.amount >= 50000) {
    req.data.highValueFlag = true;
    req.data.approvalStatus = 'REVIEW_REQUIRED';
  }
}

function setTimestamps(req){
  const now = new Date().toISOString();
  req.data.createdAt = now;
  req.data.updatedAt = now;
}

function setUpdated(req){
  req.data.updatedAt = new Date().toISOString();
}

async function approveSingle(req){
  const tx = cds.transaction(req);
  const { id } = req.data;
  const row = await tx.run(SELECT.one.from('pr.PR_Requisition').where({ ID: id }));
  if (!row) return req.error(404, 'Not found');
  if (row.approvalStatus === 'APPROVED') return true;
  await tx.run(UPDATE('pr.PR_Requisition').set({ approvalStatus: 'APPROVED', updatedAt: new Date().toISOString() }).where({ ID: id }));
  return true;
}

async function bulkReview(req){
  const { threshold } = req.data;
  const tx = cds.transaction(req);
  const count = await tx.run(
    UPDATE('pr.PR_Requisition')
      .set({ approvalStatus: 'REVIEW_REQUIRED', updatedAt: new Date().toISOString() })
      .where('amount >=', threshold, 'and approvalStatus =', 'PENDING')
  );
  return count;
}
```

---

## 6. ABAP Modernization Mapping

| ABAP Construct | CAP Equivalent | Copilot Prompt Cue |
|----------------|----------------|--------------------|
| SELECT SINGLE | SELECT.one.from | “Use SELECT.one, not full list iteration” |
| LOOP AT ... MODIFY | Bulk UPDATE with WHERE | “Replace row-by-row with set-based update” |
| FORM routines | Reusable handler functions / utility module | “Extract pure function for business rule” |
| SY-SUBRC checks | Try/if presence checks | “Use conditional on result, no subrc” |

Modernization Enforcement Prompt:
“Translate ABAP logic to CAP. Replace iterative DB writes with a single set-based UPDATE. Provide complexity reduction explanation.”

---

## 7. SAPUI5 / Fiori Elements Integration

List Report Hint:
- For quick win: Use CAP OData + Fiori Elements (metadata-driven) instead of handcrafting heavy XML.
- Copilot assists generating annotation blocks.

Annotation augmentation prompt:
“Add UI annotations for PR_Requisition to enable list report: ID (ID), description (HeaderInfo), amount (numeric formatting), approvalStatus (ValueList?). Output: CDS diff only.”

---

## 8. Chat Extension Engineering (Agentic Layer)

Performance Targets:
- Command latency < 800ms
- JSON schema stable (version in $id)
- Pure GET operations idempotent

Extension API Example (Enrichment + Diagnostics):
```javascript
// backend/agent/index.js
const express = require('express'), app = express();
app.get('/vendor/:id', vendorRisk);
app.get('/diagnostics/perf', perfSummary);
function vendorRisk(req,res){
  const { id } = req.params;
  const score = (id.charCodeAt(id.length-1) * 7) % 100;
  res.json({ vendorId:id, riskScore:score, level: score > 70 ? 'HIGH':'NORMAL', version:'1.0.0' });
}
function perfSummary(_req,res){
  res.json({
    timestamp: Date.now(),
    topEndpoints:[
      { path:'/odata/v4/PRService/PR_Requisition', p95:390, count:1420 },
      { path:'/odata/v4/PRService/Approve', p95:260, count:210 }
    ],
    recommendations:['Add composite index (amount, approvalStatus)','Batch approval calls','Enable ETag caching']
  });
}
app.listen(3001);
```

Follow-up Copilot prompt:
“Given this perfSummary JSON (paste), suggest code-level and index optimizations for CAP service; classify each as low/medium/high impact.”

---

## 9. GitHub Actions Pipeline (Optimized)

Key Points:
- Cache node_modules
- Separate security job
- Conditional deploy
- Fail-fast strategy
- Upload coverage

```yaml
# .github/workflows/cap-pipeline.yml
name: cap-pipeline
on:
  pull_request:
  push:
    branches: [ main ]
permissions:
  contents: read
  security-events: write
jobs:
  build_test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 18, cache: 'npm' }
      - run: npm ci
      - run: npm run lint --if-present
      - run: npm test --if-present -- --ci --reporters=default
      - name: Coverage artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage
  codeql:
    uses: github/codeql-action/.github/workflows/codeql.yml@main
    needs: build_test
  deploy_cf:
    if: github.ref == 'refs/heads/main'
    needs: build_test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 18 }
      - run: npm ci
      - name: Push to Cloud Foundry
        env:
          CF_API: ${{ secrets.CF_API }}
          CF_ORG: ${{ secrets.CF_ORG }}
          CF_SPACE: ${{ secrets.CF_SPACE }}
          CF_USERNAME: ${{ secrets.CF_USERNAME }}
          CF_PASSWORD: ${{ secrets.CF_PASSWORD }}
        run: |
          npm i -g cf-cli
          cf login -a $CF_API -u $CF_USERNAME -p $CF_PASSWORD -o $CF_ORG -s $CF_SPACE
          cf push
```

---

## 10. Security Hardening Checklist (Copilot-Assisted)

| Category | Action | Copilot Usage |
|----------|--------|---------------|
| Input Validation | Amount > 0, required fields | “List missing validation paths” |
| Output Consistency | Remove internal fields | “Identify fields to redact from response” |
| Logging Hygiene | No PII / tokens | “Scan code for possible PII logging” |
| Dependency Audit | Dependabot auto PRs | “Summarize risk of updated dependency” |
| Secrets | Use Actions encrypted secrets | “Generate a secret scanning policy explanation” |
| CodeQL Coverage | Tune queries | “Add custom CodeQL query to detect raw SQL template strings” |

---

## 11. Performance Optimization Patterns

Common CAP hot spots & prompts:
| Symptom | Root Cause | Prompt |
|---------|------------|--------|
| High p95 read latency | Repeated N+1 expansions | “Suggest batching strategy for these entity READ handlers” |
| Elevated CPU | Synchronous loops + per-row validation hitting DB | “Refactor this code to single set-based operation” |
| Slow approval action | Missing index | “Given entity definition & filter pattern, propose index DDL (SQLLite/Postgres flavor)” |

---

## 12. Metrics & Success Instrumentation

Instrumentation Script Prompt:
“Generate Node middleware capturing (method, path, duration, rows) and log JSON lines prefixed PERF:. Provide Jest test for middleware.”

Metrics Table:
| Metric | Capture Point | Target Delta |
|--------|---------------|-------------|
| Lines Authored vs Accepted | Copilot acceptance metrics | >45% after 4 weeks |
| PR Cycle Time | GitHub API (opened→merged) | -25% |
| Test Coverage (Branches) | Coverage report | +10% |
| High-Risk Vendor Blocks | Handler logs | Visibility only (baseline) |
| ABAP Refactor Throughput | Modernization epics | 3 objects / sprint |

---

## 13. Operational Runbook (Condensed)

| Task | Command |
|------|--------|
| Local dev | `npx cds watch` |
| Run agent microservice | `node backend/agent/index.js` |
| Execute tests | `npm test` |
| Lint fix | `npm run lint -- --fix` |
| Simulate high-value create | POST /odata/v4/PRService/PR_Requisition (amount=60000) |
| Approve PR | POST /odata/v4/PRService/Approve { id: ... } |

Incident Quick Checks:
- Validation errors? Review before hooks.
- Slow endpoint? Query cds run logs + perf middleware output.
- Deploy failure? Inspect Actions logs; re-run with debug flag `ACTIONS_STEP_DEBUG=true`.

---

## 14. Controlled Demo Script (Time-Boxed ~15 min)

1. Show raw table spec → prompt model generation (2 min)
2. Add high-value + approval logic via prompt (2 min)
3. Generate tests & run (2 min)
4. UI5 list view scaffold prompt (optional quick peek) (2 min)
5. ABAP snippet → CAP translation (2 min)
6. Pipeline YAML generation & commit diff (2 min)
7. Knowledge Base before/after naming enforcement (2 min)
8. Agent command invocation + logic adaptation (1 min)

Keep fallback branches: step-01-model, step-02-logic, step-03-tests, etc.

---

## 15. Failure Modes & Resilience

| Failure | Mitigation |
|---------|-----------|
| Copilot latency | Switch to Chat; pre-stage partial code |
| Knowledge Base not loading | Inline paste of critical rules |
| CF deploy failure | Run local only; show pipeline artifact success |
| Agent API down | Paste cached JSON response into prompt |
| Over-generated boilerplate | Prompt: “Minimize solution; remove unnecessary abstractions” |

---

## 16. Expansion Roadmap

Phase | Focus | Deliverables
------|-------|-------------
1 | Baseline acceleration | Model, handler, tests, pipeline
2 | Modernization | ABAP object migrations, ADRs
3 | Governance | KB refinement, metrics dashboards
4 | Agentic | Extension commands, performance insights
5 | Advanced Security | Custom CodeQL queries, SBOM attestation

---

## 17. High-Confidence Prompt Set (Memorize)

1. “Generate CDS entity PR_Requisition (fields X) with validation (amount>0) and default approvalStatus=PENDING. Output code only.”
2. “Create CAP handler logic to flag amount>=50000 as highValueFlag and route approvalStatus to REVIEW_REQUIRED.”
3. “Expand test suite to reach 90% branch coverage for pr-service.js; output only new Jest tests.”
4. “Translate this ABAP loop (paste) into a single set-based CAP update; explain complexity reduction.”
5. “Author a GitHub Actions workflow: build→lint→test→CodeQL→conditional CF deploy; cache dependencies.”
6. “Refactor handler for security: sanitize inputs, consistent error structure {code,message}.”
7. “Given this performance JSON (paste), list top 3 CAP optimizations with estimated impact (ms).”

---

## 18. Minimal ADR Template (For Copilot Generation)

```
# ADR: Move Approval Logic from ABAP to CAP Service

Context:
Decision:
Status: Proposed
Drivers:
- Centralize validation
- Enable test automation
- Reduce transport dependencies

Considered Options:
1. Keep ABAP
2. Hybrid (ABAP validation + CAP enrichment)
3. Full CAP externalization (Chosen)

Consequences (Positive):
- Faster iteration
- Unified pipeline
Consequences (Negative):
- Initial replatform effort
- Requires new performance baselines
```

Prompt:
“Fill this ADR with details given (paste ABAP snippet + CAP handler) emphasizing maintainability & deployment decoupling.”

---

## 19. Governance Guardrails (Enforce via Review)

| Rule | Enforcement |
|------|-------------|
| No business logic inside UI5 controllers | Code review + KB statement |
| All DB writes via transactional context | Handlers pattern tests |
| Validation logically centralized | before hooks only |
| Secrets never logged | ESLint custom rule / scanning |
| ABAP migration diffs documented | ADR per object |

Copilot Prompt:
“Scan pr-service.js for violations of governance rules: (list). Report compliance matrix.”

---

## 20. Immediate Starter Actions

1. Create governance-docs skeleton (rule-first).
2. Scaffold CAP repo + commit baseline model.
3. Generate initial handler + tests; establish coverage baseline.
4. Implement pipeline (no deploy) → extend with CodeQL.
5. Add one ABAP modernization example + ADR.
6. Optional: Add agent microservice skeleton; stub command usage transcript.

---

Keep this file authoritative; append deltas as capabilities (e.g. additional agent commands) mature.
