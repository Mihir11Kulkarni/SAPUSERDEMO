# GitHub Copilot + SAP Technical Integration Guide

Goal: Practical, implementation-focused steps to adopt GitHub Copilot across SAP-centric development (CAP, SAPUI5/Fiori, ABAP side-by-side, DevOps automation, security, extensibility).

## 1. Scope
Covers: VS Code integration, Knowledge Base setup, CAP modeling & handlers, ABAP modernization workflow, UI5 scaffolding, Chat Extension microservice, CI/CD (GitHub Actions + security), telemetry & metrics.

## 2. Component Map
- Developer Tooling: VS Code + GitHub Copilot (Inline + Chat).
- Knowledge Delivery: Copilot Enterprise Knowledge Bases (governance + domain repos).
- Runtime Targets: SAP BTP (Cloud Foundry / Kyma) for CAP deployment; optional on-prem ABAP artifacts (read-only extraction).
- Extension Surface: Copilot Chat Extension (custom commands → internal microservice).
- Automation: GitHub Actions (build/test/lint/deploy), CodeQL, Dependabot, secret scanning.
- Observability Inputs: Test coverage, PR cycle time, command usage, pipeline timings.

## 3. Integration Architecture (Logical)
Developer -> Copilot (Inline/Chat)
  |-- Pulls org Knowledge Base embeddings
  |-- Executes agentic commands (/fetchVendorRisk, /analyzeLogs) via extension
  |-- Generates/refactors: CDS, handlers, UI5 views, tests
CI/CD -> Validates (lint, test, security) -> Deploys CAP -> Provides feedback loop via metrics.

## 4. Prerequisites
- Node.js 18+
- @sap/cds, Jest (or mocha), SAP Fiori Tools extensions
- GitHub Copilot (Enterprise for KB), repo access policies
- Service credentials for BTP deployment stored as GitHub Actions secrets:
  - CF_API, CF_ORG, CF_SPACE, CF_USERNAME, CF_PASSWORD (or SSO token)
- Optional: ABAP source exported via abapGit for modernization prompts.

## 5. Knowledge Base Implementation
Repository: governance-docs/
Recommended folders:
  naming-conventions.md
  logging-guidelines.md
  security-guidelines.md
  architecture-decisions/
  domain-glossary.md

Registration Steps:
1. Curate concise, declarative rules (avoid prose ambiguity).
2. Register repo as Knowledge Base in Copilot Enterprise.
3. Validate with control prompts:
   Before: "Generate Purchase Requisition CDS entity."
   After:  "Using org naming standard (PR_ prefix & PascalCase entities) generate Purchase Requisition CDS entity."

## 6. CAP Modeling & Handlers (Prompt Flow)
Prompt 1 (Model): "From this table spec (paste CSV), produce CDS entity with default approvalStatus 'PENDING', amount > 0."
Prompt 2 (Handler): "Create service handler that sets HIGH_VALUE when amount >= 50000 and assigns approver role."
Prompt 3 (Tests): "Generate Jest tests: normal, boundary (50000), negative amount invalid."

### Example CDS (Generated/Refined)
```cds
namespace pr;
entity PR_Requisition {
  key ID : UUID;
  description : String(255);
  amount : Decimal(15,2);
  approvalStatus : String(20) default 'PENDING';
  highValueFlag : Boolean;
  createdAt : Timestamp;
}
```

### Example Handler (Node.js)
```javascript
// srv/pr-service.js
const cds = require('@sap/cds');
module.exports = cds.service.impl(function() {
  const { PR_Requisition } = this.entities;
  this.before('CREATE', PR_Requisition, req => {
    const { amount } = req.data;
    if (amount == null || amount <= 0) req.error(400, 'Amount must be > 0');
    if (amount >= 50000) {
      req.data.highValueFlag = true;
      req.data.approvalStatus = 'REVIEW_REQUIRED';
    }
  });
});
```

### Jest Test Sketch
```javascript
// test/pr-service.test.js
describe('PR_Requisition rules', () => {
  test('sets highValueFlag at boundary 50000', async () => {/* ... */});
  test('rejects negative amount', async () => {/* ... */});
});
```

## 7. ABAP Modernization Workflow
1. Export legacy ABAP snippet (SELECT + LOOP).
2. Prompt: "Explain this ABAP logic; generate equivalent CAP handler using efficient projection & filtering; add input validation."
3. Follow-up: "Refactor for bulk operations and parameterize vendor ID."

Result: Side-by-side CAP logic enabling gradual migration; commit transformation notes as ADR.

## 8. SAPUI5 / Fiori Integration
Prompt: "Generate List Report view for PR_Requisition with columns ID, description, amount, approvalStatus and custom RequestApproval action."
Follow-up: "Add action handler calling /risk enrichment service; show busy indicator."

## 9. Chat Extension Microservice
Purpose: Provide real-time domain signals (Vendor risk, log latency summary) to Copilot Chat.

### Express Service Skeleton
```javascript
// filepath: backend/microservices/copilot-extension/vendorRisk.js
const express = require('express');
const router = express.Router();
router.get('/vendor/:id', (req,res) => {
  const { id } = req.params;
  // Mock scoring logic
  const riskScore = id.endsWith('1') ? 82 : 34;
  res.json({ vendorId: id, riskScore, level: riskScore > 70 ? 'HIGH' : 'NORMAL' });
});
module.exports = router;
```

```javascript
// filepath: backend/microservices/copilot-extension/logAnalysis.js
const express = require('express');
const router = express.Router();
router.get('/logs/analysis', async (req,res) => {
  // Placeholder aggregation
  res.json({
    window: req.query.range || 'latest',
    topLatencyEndpoints: [
      { path: '/api/pr/create', p95: 420 },
      { path: '/api/pr/approve', p95: 310 }
    ],
    recommendations: ['Add index on amount', 'Batch approval updates']
  });
});
module.exports = router;
```

```javascript
// filepath: backend/microservices/copilot-extension/server.js
const express = require('express');
const app = express();
app.use(require('./vendorRisk'));
app.use(require('./logAnalysis'));
app.listen(process.env.PORT || 3001, () => console.log('Extension API up'));
```

### JSON Schemas (Aid Reasoning)
```json
{
  "$id": "vendorRisk.schema.json",
  "type": "object",
  "properties": {
    "vendorId": {"type":"string"},
    "riskScore": {"type":"integer","minimum":0,"maximum":100},
    "level": {"type":"string","enum":["HIGH","NORMAL","LOW"]}
  },
  "required": ["vendorId","riskScore","level"]
}
```

```json
{
  "$id": "logAnalysis.schema.json",
  "type": "object",
  "properties": {
    "window": {"type":"string"},
    "topLatencyEndpoints": {
      "type":"array",
      "items": {
        "type":"object",
        "properties": {
          "path":{"type":"string"},
          "p95":{"type":"number"}
        },
        "required":["path","p95"]
      }
    },
    "recommendations":{"type":"array","items":{"type":"string"}}
  },
  "required":["window","topLatencyEndpoints"]
}
```

### (Concept) Chat Extension Manifest Snippet
```json
{
  "name": "sap-copilot-extension",
  "version": "0.1.0",
  "commands": [
    {
      "name": "/fetchVendorRisk",
      "description": "Retrieve vendor risk score",
      "http": { "method": "GET", "url": "https://<host>/vendor/{vendorId}" },
      "schema": "vendorRisk.schema.json"
    },
    {
      "name": "/analyzeLogs",
      "description": "Summarize recent API latency",
      "http": { "method": "GET", "url": "https://<host>/logs/analysis?range={range}" },
      "schema": "logAnalysis.schema.json"
    }
  ]
}
```

## 10. Integration into Business Logic
Prompt after command: "/fetchVendorRisk VEND-4471. Update handler to reject HIGH risk vendors with 403."
Resulting delta example:
```javascript
if (risk.level === 'HIGH') req.error(403, 'Vendor risk too high for PR creation');
```

## 11. CI/CD (GitHub Actions)
```yaml
// filepath: .github/workflows/cap-ci.yml
name: cap-ci
on:
  push:
    branches: [ main ]
  pull_request:
permissions:
  contents: read
  security-events: write
jobs:
  build_test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '18' }
      - run: npm ci
      - run: npm run lint --if-present
      - run: npm test --if-present -- --ci --reporters=default
      - name: Archive coverage
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage
  codeql:
    needs: build_test
    uses: github/codeql-action/.github/workflows/codeql.yml@main
  deploy:
    needs: build_test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '18' }
      - run: npm ci
      - name: Deploy to CF
        env:
          CF_API: ${{ secrets.CF_API }}
          CF_ORG: ${{ secrets.CF_ORG }}
          CF_SPACE: ${{ secrets.CF_SPACE }}
          CF_USERNAME: ${{ secrets.CF_USERNAME }}
          CF_PASSWORD: ${{ secrets.CF_PASSWORD }}
        run: |
          npm install -g cf-cli
          cf login -a $CF_API -u $CF_USERNAME -p $CF_PASSWORD -o $CF_ORG -s $CF_SPACE
          cf push
```

## 12. Security & Compliance Enhancements
Prompts:
- "Review pr-service.js for input validation gaps and propose sanitized logging."
- "Add CodeQL configuration for JavaScript; exclude generated files."

Add secret scanning & Dependabot by enabling in repo settings.
Optional: Add .github/dependabot.yml for npm ecosystem.

## 13. Metrics Collection
Sources:
- GitHub Insights API: PR lead/cycle time.
- Test coverage artifact parsed → stored in dashboard.
- Copilot Enterprise usage dashboard: active users, acceptance rate.
- Custom: Count occurrences of risk rejection (instrument handler with log -> aggregator).

Prompt: "Generate a Node script to parse coverage summary JSON and push metric to InfluxDB."

## 14. Operational Runbook
Startup:
1. `npm ci`
2. `cds watch` (local dev)
3. `node backend/microservices/copilot-extension/server.js`
Health:
- /vendor/:id returns JSON in <100ms typical (mock).
- /logs/analysis used on-demand; ensure schema consistency or update manifest.
Failure Modes:
- Extension timeout → fallback: paste mock JSON into Chat and prompt manually.
- KB unresponsive → use explicit prompt injection (paste naming + security rules).

## 15. Core Prompt Reference (Condensed)
- Model: "Generate CDS entity with rules (amount>0, default status)."
- Handler augmentation: "Enforce HIGH risk vendor block using fetched JSON response."
- Test strengthening: "Increase branch coverage to >90%; add negative path for missing description."
- UI5: "Add table column with conditional formatting when highValueFlag true."

## 16. Expansion Path
Phase 1: Core modeling + tests + pipeline
Phase 2: ABAP modernization + KB refinement
Phase 3: Chat Extension + metrics instrumentation
Phase 4: Policy-based guardrails (pre-commit hooks, secret scanning baselines)

---
Minimal, technical-first; excludes strategic narrative & workshop storytelling.
