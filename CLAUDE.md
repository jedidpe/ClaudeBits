
# PMDX – Claude Instructions

Private equity data exchange platform. Converting a frontend-only prototype into an
enterprise SaaS. React frontend + AWS serverless backend deployed via CDK.

---

# Project steering
@./.claude/steering/architecture-principles.md
@./.claude/steering/billing-and-payments.md
@./.claude/steering/implementation-patterns.md
@./.claude/steering/repository-structure.md
@./.claude/steering/branching-strategy.md

# project specs

## Instructions for Claude
- MUST use agent steering found in ./claude/steering/
- MUST refer to project files in specs/

## Commands

### Frontend
```bash
npm run dev          # Vite dev server (hot reload)
npm run build        # Production build → build/
```

### Backend unit tests (run from project root)
```bash
cd infra
python -m pytest test/unit -v --tb=short          # all 192 tests
python -m pytest test/unit/handlers -v --tb=short  # handler tests only
python -m pytest test/unit/test_auth_module.py -v  # single file
```

### CDK (run from infra/)
```bash
cd infra
npx cdk synth --all --quiet          # generate CloudFormation (no AWS calls needed)
npx cdk deploy PmdxDataStack         # deploy a single stack
npx cdk deploy --all                 # deploy all stacks in dependency order
npx cdk diff PmdxApiStack            # diff against deployed
npx cdk destroy PmdxDataStack        # tear down (careful in production)
```
CDK deploy requires AWS credentials. Use `AWS_PROFILE=pmdx-dev` or export
`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`.

---

## Project Structure

```
PMDX/
├── src/                        # React 19 + TypeScript + Vite frontend
│   ├── components/             # Page and UI components
│   │   ├── Page*.tsx           # Full-page views (Page04aDocumentLibrary, etc.)
│   │   ├── AuthContext.tsx     # Cognito auth state (useAuth hook)
│   │   └── SettingsContext.tsx
│   └── lib/
│       ├── api/                # API layer
│       │   ├── client.ts       # Base fetch wrapper (apiGet/apiPost)
│       │   ├── hooks.ts        # React Query hooks (useDocuments, useFunds, etc.)
│       │   ├── documents.ts    # Document types + getDocuments()
│       │   ├── funds.ts
│       │   ├── me.ts
│       │   └── settings.ts
│       └── mockData.ts         # Static mock data used by some views
├── infra/                      # CDK TypeScript + Lambda Python
│   ├── bin/infra.ts            # CDK app entry point — stack wiring
│   ├── lib/stacks/             # CDK stack definitions
│   │   ├── data-stack.ts       # DynamoDB tables
│   │   ├── auth-stack.ts       # Cognito User Pool
│   │   ├── api-stack.ts        # HTTP API + all Lambdas
│   │   ├── frontend-stack.ts   # S3 + CloudFront
│   │   ├── monitoring-stack.ts # CloudWatch alarms
│   │   └── github-actions-stack.ts  # OIDC role (deploy once)
│   ├── lambdas/                # Python 3.12 Lambda handlers
│   │   ├── shared/pmdx_common/ # Shared layer: auth, db, response, logger, metrics
│   │   ├── get_documents/      # GET /api/v1/documents
│   │   ├── get_funds/          # GET /api/v1/funds
│   │   ├── get_fund_detail/    # GET /api/v1/funds/{fundId}
│   │   ├── get_me/             # GET /api/v1/me
│   │   ├── get_settings/       # GET /api/v1/settings
│   │   ├── put_settings/       # PUT /api/v1/settings
│   │   ├── authorizer/         # JWT Lambda authorizer
│   │   └── admin/              # Admin-only endpoints
│   └── test/unit/              # pytest test suite (192 tests)
│       ├── conftest.py         # Shared fixtures (make_event, aws_infra)
│       ├── handlers/           # Handler-specific tests
│       │   └── conftest.py     # _load(), _reset_singletons(), USER_POOL_ID patch
│       ├── test_auth_module.py
│       ├── test_db.py
│       └── test_response.py
├── docs/                       # CICD_Setup.md, Developer_Setup.md, Environment_Config.md
├── .github/workflows/
│   └── deploy.yml              # CI/CD pipeline (security → test → build → deploy)
└── ToDo.md                     # Project progress tracker
```

---

## AWS Context

| Resource | Value |
|---|---|
| Account | 598508853322 |
| Region | eu-west-2 |
| API Gateway | https://qyrezak4re.execute-api.eu-west-2.amazonaws.com |
| Cognito Pool ID | eu-west-2_CVMy3y3jI |
| Cognito Client ID | 56g0pouurl516mril21ml3c6eg |
| GitHub Org/Repo | jedidpe/Pmdx |

CDK stack deploy order: `DataStack → AuthStack → ApiStack → MonitoringStack → FrontendStack`

---

## Key Gotchas (Hard-won lessons)

### Lambda / Python

**API field names must match exactly** — Lambda and TypeScript types must agree.
The Lambda returns `uploadedDate` (not `uploadDate`) and `fundId`/`fundName` (not `fund`).
Always cross-check `handler.py` response dict keys against the TypeScript interface.

**Handlers cache module-level constants at import time.**
`USER_POOL_ID = os.environ.get('USER_POOL_ID', '')` is set when the module loads,
not per-request. In moto tests, patch it after pool creation:
```python
for mod in list(sys.modules.values()):
    if mod is not None and hasattr(mod, 'USER_POOL_ID'):
        mod.USER_POOL_ID = pool_id
```

**HTTP API v2 Lambda authorizer — `identitySource` is a list, not a string:**
```python
token = event.get('identitySource', '')
if isinstance(token, list):
    token = token[0] if token else ''
```

**DynamoDB `begins_with` over-matches hierarchical SK patterns.**
`SK begins_with 'FUND#'` also matches `FUND#fund-1#PC#pc-1`. Always filter by
`entityType` after querying.

### Testing

**Handler import collision** — multiple test files importing `handler` via `import`
share the Python module cache. Use `importlib.util.spec_from_file_location` with a
unique module name instead. See `infra/test/unit/handlers/conftest.py` `_load()`.

**`from conftest import` in handler tests** — pytest resolves `conftest` to the
nearest `conftest.py`, which is `handlers/conftest.py`, not the root one. Root
conftest items (`make_event`, `_LambdaContext`) are re-exported from handlers conftest.

### Frontend

**`new Date(undefined)` → Invalid Date → `format()` throws `RangeError: Invalid time value`.**
Always guard date fields from the API: `d.uploadedDate ? new Date(d.uploadedDate) : new Date()`

**Tailwind classes must appear as literal strings** — dynamic construction
(`'text-' + color`) is purged at build time. Use full class names.

### CDK / Windows

**No Docker on Windows** — CDK Lambda bundling uses local pip with cross-platform
wheels. See `patterns.md` for the `tryBundle` snippet.

**Git Bash path expansion** — paths like `/aws/lambda/...` get mangled.
Prefix AWS CLI commands with `MSYS_NO_PATHCONV=1`.

**CDK tsconfig** — use `module: "commonjs"` and `moduleResolution: "node"`,
NOT `NodeNext` (breaks ts-node resolution).

### CI/CD (deploy.yml)

**Trivy releases** — Trivy publishes nearly all versions as GitHub pre-releases.
Only the latest stable (currently `v0.69.2`) is a full release.
`gh release download` and `install.sh` both fail on pre-release tags.
Always check `gh release list --repo aquasecurity/trivy` to find the current stable.

**trivy-action@0.34.1** uses `aquasecurity/setup-trivy` internally, which runs
`install.sh` from the trivy repo's live main branch. This is unreliable.
Current fix: install Trivy once via `gh release download`, then pass
`skip-setup-trivy: true` to all three trivy-action steps.

---

## Architecture Decisions

- **Single-table DynamoDB design** — `PK=TENANT#{id}`, `SK=FUND#...`, `DOC#...`, etc.
- **Lambda Powertools** — structured JSON logging, EMF metrics (`PMDX` namespace,
  `TenantId` dimension), module-level `Logger`/`Metrics` instances.
- **OIDC auth in CI** — no stored AWS credentials. GitHub Actions exchanges a JWT
  for temporary credentials by assuming a scoped IAM role.
- **Vite build modes** — `development`/`staging`/`production` map to `.env.*` files.
  Build with `vite build --mode staging` etc.

---

## Git Workflow

- Branch per feature, PR for all changes — never commit directly to `main`
- Commit format: `type(scope): description`
  - e.g. `fix(ci): pin Trivy binary`, `feat(api): add document upload endpoint`
- PR merges trigger the full CI pipeline (Trivy security → pytest → Vite build → CDK deploy)
- `main` branch → Production environment
- Dependabot PRs: merge low-risk ones directly; check `gh run list` after each merge
  to catch CI failures before stacking more merges
