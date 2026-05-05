# DDD-Aware SDLC · V3 — Production-Ready Pipeline

> Quy trình phát triển phần mềm cho dự án **Domain-Driven Design**, vận hành bằng **Claude Code CLI** kết hợp **OpenSpec** (tài liệu hoá & quản lý spec) + **Superpowers** (coding workflow). Tích hợp 13 nâng cấp DDD (A–M) so với pipeline AI-driven generic.

---

## 1. Tổng quan

### 1.1. Triết lý

Pipeline đặt **domain model** vào trung tâm chứ không phải code. Mọi giai đoạn đều phải:
- Bảo toàn **Ubiquitous Language** (glossary là source of truth, không tự sinh từ mới)
- Tôn trọng **Bounded Context** boundary (linter chặn import chéo)
- Phát triển model theo nguyên tắc **Refactoring toward Deeper Insight** (sai bản chất → quay về Plan, không patch code)

### 1.2. Hai bộ tool chính

| Tool | Vai trò | Lệnh tiêu biểu |
|---|---|---|
| **OpenSpec** | Tài liệu hoá & quản lý spec dạng version-controlled change proposal | `/openspec:propose` · `/openspec:apply` · `/openspec:archive` |
| **Superpowers** | Coding workflow có kỷ luật (brainstorm → plan → review-gate → TDD → execute → review) | `brainstorming` · `writing-plans` · `plan-review-gate` · `test-driven-development` · `executing-plans` · `subagent-driven-development` · `code-reviewer` · `commit-commands` |

### 1.3. Project layout

```
ddd-platform/                          # monorepo
├── services/
│   ├── billing/                       # Bounded Context · Billing
│   │   └── infra/                     # service-level CDK stacks
│   ├── identity/                      # Bounded Context · Identity
│   └── notification/                  # Bounded Context · Notification
├── shared/
│   ├── glossary.md                    # Ubiquitous Language
│   ├── context-map.yaml               # quan hệ giữa contexts
│   ├── adr/                           # Architectural Decision Records
│   └── schemas/                       # AsyncAPI / JSON-Schema
├── infra-cdk/                         # AWS CDK app · TypeScript
│   ├── bin/
│   │   └── app.ts                     # CDK entry · stage prod/staging
│   ├── lib/
│   │   ├── platform/                  # cross-service stacks
│   │   │   ├── networking-stack.ts    # VPC · subnets · SGs
│   │   │   ├── event-bus-stack.ts     # EventBridge bus + rules
│   │   │   ├── api-gateway-stack.ts   # shared HTTP API + Cognito
│   │   │   └── schema-registry-stack.ts  # Glue Schema Registry
│   │   └── shared/                    # reusable constructs
│   │       ├── service.construct.ts
│   │       └── observability.construct.ts
│   ├── cdk.json
│   └── package.json
├── openspec/
│   ├── specs/                         # spec hiện tại
│   ├── changes/                       # proposals đang làm
│   └── archive/                       # changes đã merge
├── .github/workflows/                 # CI · arch-lint · contract-test · cdk diff
└── .claude/
    ├── settings.json                  # hooks, permissions
    ├── hooks/                         # pre-commit lint:arch · lint:glossary
    └── skills/                        # custom domain skills
```

Mỗi service (per bounded context) layer theo DDD và có CDK stacks riêng:

```
services/billing/
├── src/billing/
│   ├── domain/                        # thuần nghiệp vụ — KHÔNG import infra
│   │   ├── refund_request.py          # Aggregate root
│   │   ├── value_objects/
│   │   ├── events/
│   │   └── services/                  # Domain Service (vd. RefundEligibilityChecker)
│   ├── application/                   # orchestration, transactions
│   ├── infrastructure/                # adapters cụ thể (code-level)
│   │   ├── repositories/
│   │   ├── adapters/                  # ACL ra context khác
│   │   └── events/                    # EventBridge publisher
│   └── interface/                     # entry points
│       ├── api/                       # REST handlers
│       └── consumers/                 # SQS event consumers
├── tests/
├── features/                          # BDD scenarios (.feature)
├── pacts/                             # consumer-driven contracts
├── infra/                             # SERVICE-LEVEL CDK STACKS
│   ├── billing-stack.ts               # composite root stack
│   ├── billing-database-stack.ts      # Aurora Postgres ServerlessV2
│   ├── billing-service-stack.ts       # ECS Fargate + ALB target
│   ├── billing-pipeline-stack.ts      # CodePipeline + CodeBuild
│   └── billing-observability-stack.ts # CloudWatch · X-Ray · Datadog
├── Dockerfile
└── pyproject.toml
```

### 1.4. AWS deploy target — biểu diễn dưới dạng CDK stacks

Toàn bộ hạ tầng AWS được code-as-infrastructure bằng **AWS CDK (TypeScript)**, tách thành stack hierarchy theo bounded context:

```
CDK App (infra-cdk/bin/app.ts)
│
├── Stage: ddd-platform-prod
│   │
│   ├── PlatformStack                       # shared · deploy 1 lần per env
│   │   ├── NetworkingStack                 # VPC · 3 AZ · NAT · SGs
│   │   ├── EventBusStack                   # EventBridge "app-events"
│   │   ├── ApiGatewayStack                 # shared HTTP API · Cognito
│   │   └── SchemaRegistryStack             # Glue Schema Registry · BACKWARD
│   │
│   ├── BillingStack                        # composite per bounded context
│   │   ├── BillingDatabaseStack            # Aurora Postgres v15 · ACU 4-8
│   │   ├── BillingServiceStack             # ECS Fargate · ALB · auto-scale 2-10
│   │   ├── BillingSecretsStack             # Secrets Manager · rotate 30d
│   │   ├── BillingObservabilityStack       # CW logs · X-Ray · Datadog · alarms
│   │   └── BillingPipelineStack            # CodePipeline · CodeBuild · ECR
│   │
│   ├── IdentityStack                       # mirror BillingStack composition
│   └── NotificationStack
│
└── Stage: ddd-platform-staging             # clone prod · smaller capacity
    └── (mirror prod stacks)
```

**Dependency order** (CDK auto-resolve):
- `PlatformStack` → `BillingStack` (Billing import VPC, EventBus, ApiGateway từ Platform)
- `BillingDatabaseStack` → `BillingServiceStack` (service inject DB endpoint)
- `BillingPipelineStack` independent — deploy/destroy không phụ thuộc service

**Deploy commands**:
```bash
cd infra-cdk
npx cdk diff   ddd-platform-staging/BillingStack/*
npx cdk deploy ddd-platform-staging/BillingStack/* --require-approval=never

# After staging green + manual approval:
npx cdk deploy ddd-platform-prod/BillingStack/*    --require-approval=any-change
```

**Mapping CDK stacks → AWS services**:

| CDK Stack | AWS Resources |
|---|---|
| `NetworkingStack` (Platform) | VPC · 3 AZ · public/private/isolated subnet · NAT · SGs |
| `EventBusStack` (Platform) | EventBridge bus `app-events` + rules → SQS Notification |
| `ApiGatewayStack` (Platform) | HTTP API + Cognito JWT authorizer |
| `SchemaRegistryStack` (Platform) | Glue Schema Registry · BACKWARD compatibility |
| `BillingDatabaseStack` | Aurora Postgres ServerlessV2 · IAM auth · point-in-time recovery |
| `BillingServiceStack` | ECS Fargate Task + ALB Target Group · auto-scale 2-10 |
| `BillingSecretsStack` | Secrets Manager · DB password · Stripe key · rotate 30d |
| `BillingObservabilityStack` | CloudWatch Log Group + Metric Filters · X-Ray · Datadog forwarder · SLO alarms |
| `BillingPipelineStack` | CodePipeline + CodeBuild + ECR repository · image scan |
| `NotificationServiceStack` | SQS + DLQ · redrive 3 attempts · alarm DLQ > 0 |

**Tooling đi kèm**:

| Tool | Vai trò |
|---|---|
| LaunchDarkly | Feature flag cho gradual rollout (Phase 5.5) |
| Datadog + PagerDuty | SLO monitoring · oncall (Phase 6) |

---

## 2. Pipeline tổng thể

```
                    ┌─────────────────────────────────────────────────────────────┐
                    │                  Per-Context Pipeline Runner (H)            │
                    │                                                              │
USER                │   ┌──────┐    ┌──────┐  spike   ┌──────┐    ┌──────┐         │
 │  yêu cầu         │   │ DISC │───▶│  PS  │─ ─ ─ ─ ─▶│ SPIKE│ ─ ▶│  PT  │         │
 │                  │   └──┬───┘    └──┬───┘ optional └──────┘    └──┬───┘         │
 ▼                  │      │           │                              │              │
┌──────┐  Yêu cầu   │   Phase 0    Phase 1                       Plan & Specs       │
│ USER │───────────▶│   Discovery  Strategic + Tactical                              │
└──────┘            │      ▲         ▲                                              │
                    │      │         │                                              │
                    │      │  knowl. │ deeper                                       │
                    │      │  sync   │ insight                                      │
                    │      │         │                                              │
                    │   ┌──┴───┐  ┌──┴───┐    ┌──────┐    ┌──────┐    ┌──────┐    │
                    │   │  SY  │  │  PR  │◀───│ DMR  │◀───│ CMP  │◀───│  CR  │    │
                    │   └──┬───┘  └──┬───┘    └──┬───┘    └──┬───┘    └──┬───┘    │
                    │      │         │           │           │           │          │
                    │      ▲         ▲           │           │           │          │
                    │      │         │           ▼ refactor  ▼ block     ▼          │
                    │   ┌──┴───┐  ┌──┴───┐    ┌──────┐    ┌──────┐    ┌──────┐    │
                    │   │ ROLL │◀─│ PR   │    │ DEV  │◀───│ DEV  │    │ DEV  │◀── │
                    │   └──┬───┘  └──────┘    └──┬───┘    └──────┘    └──────┘    │
                    │      │ SLO              ┌──┴───┐                              │
                    │      ▼ breach           │ LINT │                              │
                    │   ┌──────┐              └──┬───┘                              │
                    │   │ OPS  │                 │                                  │
                    │   └──┬───┘              ┌──┴───┐  ┌──────┐  ┌──────┐         │
                    │      │ incident         │ SBE  │─▶│ UIT  │─▶│  CT  │         │
                    │      │ → discovery      └──────┘  └──┬───┘  └──┬───┘         │
                    │      │                              tests fail  │             │
                    │      └──────────────────────────────▶ DEV       │             │
                    │                                                  │             │
                    │                                                  ▼             │
                    │                                              test results      │
                    └──────────────────────────────────────────────────────────────┘

                              ╭ FAST LANE (M) — hotfix · typo · config-only ╮
                              │  USER → DEV → LINT → UIT → PR → ROLL          │
                              │  (skip Discovery + Plan + Compliance heavy)   │
                              ╰─────────────────────────────────────────────╯
```

### 2.1. 7 phase chính

| Phase | Tên | Nodes | Output chính |
|---|---|---|---|
| 0 | Domain Discovery | DISC | events.md · glossary delta · context-map delta |
| 1 | Strategic + Tactical Plan | PS · (SPIKE) · PT | strategic-plan.md · tactical plan.md · ADR · review-checklist |
| 2 | Implementation | DEV · LINT | source code (layered) · arch-lint-report · glossary-drift |
| 3 | Testing | SBE · UIT · CT | .feature · test results · pact contracts |
| 4 | Review | CR · CMP · DMR | review-report · security-report · domain-model-health |
| 5 | Delivery | PR · ROLL · SY | git commit · PR · CDK stack · rollout-report · archive |
| 6 | Operate | OPS | SLO snapshot · incident report · discovery feedback |

### 2.2. Vòng lặp (loops)

| Loop | Khi nào kích hoạt | Hành động |
|---|---|---|
| **3 → 2** Tests Fail | UIT/CT có test rớt | DEV fix code · re-run test |
| **4 → 2** Issues Found | CR phát hiện vấn đề kỹ thuật | DEV refactor · re-test |
| **4 → 2** Compliance Block | CMP phát hiện CVE/PII/secret | DEV fix · re-scan |
| **(F) 4 → 1** Deeper Insight | DMR thấy mô hình sai bản chất | Quay về Tactical Plan, KHÔNG patch code |
| **5.5 → 5** SLO Breach | ROLL canary breach SLO | Auto-rollback CodeDeploy + revert flag |
| **6 → 0** Incident → Discovery | OPS có incident sinh insight nghiệp vụ | Feed insight về Discovery sprint sau |
| **5 → 0** Knowledge Sync | SY archive xong | Glossary/ContextMap mới feed về Discovery |

---

## 3. 13 nâng cấp DDD

| Mã | Nâng cấp | Vào đâu |
|---|---|---|
| **A** | Domain Discovery (Phase 0 mới) | Phase 0 |
| **B** | Strategic vs Tactical Plan (split) | Phase 1 |
| **C** | Architecture + Glossary Linter | Phase 2 (LINT node) |
| **D** | Specification by Example + Contract Test | Phase 3 |
| **E** | Domain Model Reviewer | Phase 4 (DMR node) |
| **F** | Refactor toward Deeper Insight (loop) | Loop 4 → 1 |
| **G** | Glossary / Context-Map / ADR Sync | Phase 5 (SY node) |
| **H** | Per-Context Pipeline Runner (vỏ chạy độc lập) | Toàn pipeline |
| **I** | Spike (optional throwaway PoC) | Phase 1.5 |
| **J** | Compliance & Security Gate | Phase 4 (CMP node) |
| **K** | Gradual Rollout (canary + SLO gate) | Phase 5.5 (ROLL node) |
| **L** | Operate phase (SLO + Incident loop) | Phase 6 |
| **M** | Fast Lane (bypass cho hotfix) | Song song |

---

## 4. Chi tiết từng node

### 4.0. USER · Product Owner — Define Requirement

**Node làm gì?** PO viết yêu cầu thô (pain point, compliance) thành ticket có acceptance criteria rõ — đầu vào của cả pipeline.

| | |
|---|---|
| **Input** | Pain point từ Customer Service · ràng buộc compliance · success metric |
| **AI Activity** | Không (thủ công) |
| **Output** | Jira ticket với story/AC/stakeholders/risk |

---

### 4.1. DISC · Domain Discovery (Phase 0 · upgrade A)

**Node làm gì?** Một buổi Discovery duy nhất với Domain Expert: chạy Event Storming tìm events/commands/actors/policies, validate ngôn ngữ (Refund vs Reversal), chốt Ubiquitous Language trong `glossary.md`, vẽ Context Map (ACL · Conformist · Customer/Supplier).

| | |
|---|---|
| **Input** | Ticket + AC · Domain Expert (Finance + CS) · glossary v3.2 hiện tại |
| **AI Activity** | `superpowers:brainstorming` orchestrate session · `openspec-propose` cho glossary delta + context map delta |
| **Lệnh CLI** | `> /superpowers:brainstorm Domain discovery cho FT-204` |
| **Output** | `docs/events.md` · `glossary.md` delta · `context-map.yaml` delta · quyết định ngôn ngữ + edge case |

**Note cho người demo**: gộp Event Storming + language validation + glossary + context map vào 1 buổi để giữ tính liền mạch. Đừng tách thành 4 step riêng.

---

### 4.2. PS · Strategic Plan (Phase 1 · upgrade B)

**Node làm gì?** Strategic design — chọn integration pattern giữa các context (ACL · Conformist · Customer/Supplier · Shared Kernel), SLA, fallback khi downstream service down. Mọi quyết định lớn ghi thành ADR.

| | |
|---|---|
| **Input** | Glossary v0 + Context Map từ DISC |
| **AI Activity** | `openspec-propose` |
| **Lệnh CLI** | `> /openspec:propose FT-204 strategic integration plan` |
| **Output** | `changes/ft-204-refund-strategic/{proposal,design,tasks}.md` · `adr/0042-*.md` |

---

### 4.3. SPIKE · Throwaway PoC (Phase 1.5 · upgrade I · OPTIONAL)

**Node làm gì?** Khi tactical plan có giả định kỹ thuật chưa chắc chắn (vd. throughput Aurora chịu được không), tách 1 nhánh **throwaway PoC** trong vài giờ để validate trước. Code spike **không merge** — chỉ giữ `findings.md`.

| | |
|---|---|
| **Input** | Strategic plan + giả định rủi ro |
| **AI Activity** | `superpowers:brainstorming` để framing hypothesis · viết script load test |
| **Lệnh CLI** | `> git checkout -b spike/refund-throughput && /superpowers:brainstorm Validate refund throughput` |
| **Output** | `spike/findings.md` (hypothesis VALIDATED/INVALIDATED + caveat) · spike branch bị xoá |
| **Kỷ luật** | Time-box ≤ 1 ngày · branch không merge |

---

### 4.4. PT · Tactical Plan (Phase 1 · upgrade B)

**Node làm gì?** Tactical design — Aggregate nào mới, invariants ra sao, Repository, Domain Service, event handler. Sinh plan ~12 task. **Cần human OK qua plan-review-gate** mới được execute.

| | |
|---|---|
| **Input** | Strategic plan đã chốt + spike findings (nếu có) |
| **AI Activity** | `superpowers:writing-plans` · `plan-review-gate` |
| **Lệnh CLI** | `> /superpowers:writing-plans FT-204 tactical implementation` |
| **Output** | `plan.md` (12 task) · aggregate design · `review-checklist.md` (status PENDING) |
| **Gate** | Developer phải tick checklist + đặt `Overall Decision: OK` mới được execute |

---

### 4.5. DEV · Developer Agent (Phase 2 · upgrade C)

**Node làm gì?** Execute plan từng task, code theo layering DDD (`domain/application/infrastructure/interface`), bám TDD (RED → GREEN → REFACTOR). Mỗi task xong tick trong `tasks.md`.

| | |
|---|---|
| **Input** | `plan.md` đã APPROVED |
| **AI Activity** | `superpowers:executing-plans` + `test-driven-development` + `openspec-apply` (mark task done) |
| **Lệnh CLI** | `> /superpowers:executing-plans plan.md` |
| **Output** | Source code layered · `tests/` · tasks 1→12 marked done |
| **Tuỳ chọn** | `subagent-driven-development` cho task song song không phụ thuộc |

---

### 4.6. LINT · Architecture + Glossary Linter (Phase 2 · upgrade C)

**Node làm gì?** Pre-commit hook tự chạy. **Block commit** nếu code phá boundary (domain import infrastructure) hoặc dùng từ ngoài glossary. Bắt fix tại chỗ, không bypass `--no-verify`.

| | |
|---|---|
| **Input** | Source code mới + `arch-rules.yaml` + `glossary.md` |
| **AI Activity** | Hook bash · không AI |
| **Lệnh CLI** | `> git commit` (hook auto trigger `npm run lint:arch && lint:glossary`) |
| **Output** | `arch-lint-report.json` · `glossary-drift.json` · build pass/block |

---

### 4.7. SBE · Specification by Example (Phase 3 · upgrade D)

**Node làm gì?** Viết file `.feature` Given-When-Then bằng đúng glossary terms — Domain Expert + QA không biết code vẫn đọc và review được scenario.

| | |
|---|---|
| **Input** | Glossary + AC từ ticket |
| **AI Activity** | `superpowers:test-driven-development` |
| **Lệnh CLI** | `> /superpowers:test-driven-development Refund BDD specs` |
| **Output** | `features/billing/refund.feature` (3+ scenario) · step definitions skeleton |

---

### 4.8. UIT · Unit · Integration · E2E (Phase 3 · upgrade D)

**Node làm gì?** Chạy unit + integration + E2E. Fail → pipeline auto loop ngược về Developer (3 → 2).

| | |
|---|---|
| **Input** | Source code + feature files |
| **AI Activity** | Bash test runners (jest · behave · playwright) |
| **Lệnh CLI** | `> npm run test:all` |
| **Output** | `test-results.junit.xml` · `coverage-by-aggregate.json` |
| **Loop** | Fail → DEV (3 → 2) |

---

### 4.9. CT · Contract Test (Phase 3 · upgrade D)

**Node làm gì?** Consumer-driven contract test bằng Pact: đảm bảo event/API schema không phá Notification, Identity đang consume. Pass = safe to deploy.

| | |
|---|---|
| **Input** | Event schemas + consumer expectations từ broker |
| **AI Activity** | Pact CLI |
| **Lệnh CLI** | `> npm run test:contract` |
| **Output** | `pacts/billing-{notification,identity}.json` · verdict compatible |

---

### 4.10. CR · Code Reviewer (Phase 4 · upgrade E)

**Node làm gì?** Reviewer kiểm chất lượng kỹ thuật + security: race condition, secret leak, SQL injection, performance — như code review truyền thống.

| | |
|---|---|
| **Input** | PR diff đầy đủ + test results |
| **AI Activity** | `superpowers:requesting-code-review` spawn `code-reviewer` subagent |
| **Lệnh CLI** | `> /superpowers:requesting-code-review PR #1287` |
| **Output** | `review-report.md` · GitHub PR comments line-level |

---

### 4.11. CMP · Compliance & Security Gate (Phase 4 · upgrade J)

**Node làm gì?** Compliance & Security Gate: SAST (Snyk/Semgrep) · DAST · SBOM (Syft CycloneDX) · license scan · PII data classification · GDPR/PCI check. Block PR nếu critical CVE hoặc PII leak.

| | |
|---|---|
| **Input** | PR diff + dependency manifest + compliance ruleset |
| **AI Activity** | `snyk · semgrep · syft · gitleaks · license-checker` + `superpowers:requesting-code-review --focus pii-pci` |
| **Lệnh CLI** | `> npm run security:all` |
| **Output** | `security-report.json` · `sbom.cdx.json` · audit trail S3 (object-lock 7 năm) |
| **Block rule** | Critical CVE / PII finding / secret leak → BLOCK PR |

---

### 4.12. DMR · Domain Model Reviewer (Phase 4 · upgrade E)

**Node làm gì?** Reviewer thứ 2 chỉ kiểm domain model health: Aggregate có anemic không, ACL leak, language drift. Sai bản chất → kích Loop F quay về Plan, không patch code lấy lệ.

| | |
|---|---|
| **Input** | PR diff + glossary + context-map + tactical plan |
| **AI Activity** | `superpowers:requesting-code-review --focus domain-model` |
| **Lệnh CLI** | `> /superpowers:requesting-code-review --focus domain-model` |
| **Output** | `domain-model-health.json` · `refactor-proposals/` |
| **Quyết định** | (a) fix tại chỗ HOẶC (b) kích Loop F — tuỳ mức độ |

---

### 4.13. PR · Commit + PR + Deploy Staging (Phase 5 · upgrade G)

**Node làm gì?** AI sinh conventional commit, push branch, mở PR đính kèm test plan + link OpenSpec change + ADR. CI build image → push ECR. Chạy `cdk diff` để review thay đổi hạ tầng. Merge → **CDK deploy** các stack (BillingDatabaseStack/ServiceStack/ObservabilityStack/PipelineStack) tự động lên staging. Prod cần manual approval + `cdk deploy ddd-platform-prod/...`.

| | |
|---|---|
| **Input** | Code đã APPROVED bởi cả CR + CMP + DMR |
| **AI Activity** | `commit-commands:commit-push-pr` → `gh pr create` → CodeBuild → `cdk diff` → `cdk deploy` |
| **Lệnh CLI** | `> /commit-commands:commit-push-pr` rồi `npx cdk deploy ddd-platform-staging/BillingStack/*` |
| **Output** | Conventional commit · PR #1287 · ECR image · CDK CloudFormation stacks · ECS task definition |
| **CDK artifacts** | `infra-cdk/bin/app.ts` · `services/billing/infra/billing-stack.ts` (composite) · `billing-service-stack.ts` (NestedStack) |

---

### 4.14. ROLL · Gradual Rollout (Phase 5.5 · upgrade K)

**Node làm gì?** Bật feature flag (LaunchDarkly), route canary 5% → 25% → 50% → 100% theo SLO check (P95 latency, error rate, refund completion rate). SLO breach → **auto-rollback** qua CodeDeploy + revert flag, không cần human.

| | |
|---|---|
| **Input** | Image deploy staging + smoke test pass + SLO definition |
| **AI Activity** | CodeDeploy + LaunchDarkly orchestration |
| **Lệnh CLI** | `> /superpowers:executing-plans rollout.md` |
| **Output** | `rollout-report.json` · 4 stage SLO snapshot · feature flag @ 100% · auto-rollback armed 24h |
| **SLO targets** | P95 < 500ms · error rate < 0.1% · refund completion > 99% |

---

### 4.15. SY · Glossary / Context-Map / ADR Sync (Phase 5 · upgrade G)

**Node làm gì?** Post-merge: move change từ `changes/` sang `archive/`, apply specs delta, bump glossary version, register event schema lên Glue Schema Registry với enforce BACKWARD compatibility. Knowledge sync cho sprint sau.

| | |
|---|---|
| **Input** | OpenSpec change đã merged + glossary delta + ADR mới |
| **AI Activity** | `openspec-archive-change` · schema registry CLI |
| **Lệnh CLI** | `> /openspec:archive ft-204-refund-strategic` |
| **Output** | `archive/.../MIGRATION.md` · glossary v3.2 → v3.3 · context-map updated · `schemas/events/billing.refund.v1.json` registered |

---

### 4.16. OPS · Operate (Phase 6 · upgrade L)

**Node làm gì?** SLO monitoring 24/7, error budget tracking, oncall alerting (PagerDuty), runbook tự động. Khi có incident → root cause → tạo incident report + feed back vào Discovery loop để tinh chỉnh domain model.

| | |
|---|---|
| **Input** | Service đã 100% traffic + SLO definition + runbook + oncall rotation |
| **AI Activity** | Datadog SLO tracker · PagerDuty alerts · `superpowers:brainstorming` cho post-mortem |
| **Lệnh CLI** | (passive monitoring) · post-mortem qua brainstorming |
| **Output** | `incidents/INC-NNNN.md` (5-why) · `slo-snapshot-week.json` · Discovery ticket mới · runbook update |
| **Loop quan trọng nhất** | Incident → Discovery: production phát hiện edge case nghiệp vụ chưa biết → nuôi domain model trưởng thành |

---

## 5. Sample task walkthrough — FT-204

### 5.1. Đề bài

**Title**: Cho phép Customer yêu cầu Refund cho đơn đã thanh toán

**Background**: 18% support ticket/tháng là về refund, average resolution 36h. Finance team cần audit trail rõ + control ngưỡng phê duyệt.

**User stories**:
- Là Customer, tôi muốn yêu cầu refund online cho đơn đã settle
- Là Manager, tôi muốn approve/reject refund > 5M VND
- Là Finance Lead, tôi muốn audit trail mọi refund
- Là Customer, tôi muốn nhận email xác nhận khi refund completed/failed

**Acceptance criteria**:
- `EligibilityWindow` = 30 ngày từ `order.settledAt`; quá → reject `OutsideEligibilityWindow`
- `amount > 5,000,000 VND` → state `PendingApproval`, đợi Manager approve trong 24h
- `amount ≤ payment.amount` (cho phép partial); vi phạm → 400 `InvalidRefundAmount`
- Refund success → publish `RefundCompleted` topic `billing.refund.v1`; gửi email + Slack #finance
- Refund failed → retry 3 lần exp backoff; alert oncall
- Identity unreachable → state `PendingVerification`, không block

**Out of scope**: FX · subscription · charge-back

**Đụng 3 bounded context**: Billing (primary) · Identity · Notification

**Est**: 13 SP · 5 ngày

### 5.2. Walkthrough qua từng phase

| Phase | Node | Input mở | AI activity tóm tắt | Output ngắn |
|---|---|---|---|---|
| 0 | DISC | Ticket + Domain Expert (An, Linh) | `/superpowers:brainstorm` event storming + language validation + glossary delta | events.md · 4 term mới · 3 relationship · tách Refund vs Reversal |
| 1 | PS | DISC artifacts | `/openspec:propose strategic plan` | proposal/design/tasks · ADR-0042 (Notification = Conformist) |
| 1.5 | SPIKE | Hypothesis về Aurora throughput | locust load test 500 RPS | findings.md: VALIDATED · caveat ACU 4→8 |
| 1 | PT | Strategic + spike findings | `/superpowers:writing-plans` + `plan-review-gate` | plan.md (12 task) · review-checklist PENDING |
| — | (gate) | Developer tick checklist | Đặt `Overall Decision: OK` | Gate APPROVED |
| 2 | DEV | plan.md APPROVED | `executing-plans` + TDD + `openspec-apply` | RefundRequest aggregate · 12 task done |
| 2 | LINT | Source code | Pre-commit hook bash | 2 violations → FAIL · dev fix |
| 3 | SBE | Glossary | `test-driven-development` BDD | refund.feature 3 scenario |
| 3 | UIT | Source + feature | `npm run test:all` | 73 pass · 1 FAIL e2e (loop về DEV) |
| 3 | CT | Schemas + consumer | `pact verify` | 3 contracts compatible |
| 4 | CR | PR diff | `requesting-code-review` | 2 issue · 0 critical |
| 4 | CMP | PR + dep manifest | snyk + semgrep + syft + PII scan | 1 PII finding (GDPR Art.32) → BLOCK · dev fix |
| 4 | DMR | PR + glossary + context-map | `requesting-code-review --focus domain-model` | 2 model concern · fix in place (không Loop F) |
| 5 | PR | Approved code | `commit-push-pr` + CodeBuild + CodePipeline | PR #1287 · ECR image · ECS task v47 staging |
| 5.5 | ROLL | Image staging + SLO def | CodeDeploy canary 5/25/50/100% + LaunchDarkly | rollout-report: PROMOTED · 0 SLO breach |
| 5 | SY | Merged change | `openspec:archive` + Glue Schema Registry | glossary v3.3 · schema registered BACKWARD |
| 6 | OPS | Service prod 100% | Datadog SLO tracker · PagerDuty | INC-8842 (AmEx 3DS timeout) · post-mortem · feed Discovery |

### 5.3. Một số file artifact cụ thể

#### `docs/events.md` (output DISC)
```markdown
# Refund Domain Events — Billing context

## Events
RefundRequested   { orderId, customerId, amount, reason, requestedAt }
RefundApproved    { refundId, approvedBy, approvedAt }
RefundRejected    { refundId, rejectedBy, reason }
RefundCompleted   { refundId, transactionId, completedAt }
RefundFailed      { refundId, errorCode, retryable }

## Commands
RequestRefund     actor: Customer
ApproveRefund     actor: Manager  guard: amount > 5M

## Policies
- EligibilityWindow: order.settledAt within 30 days
- ManagerApproval: amount > 5,000,000 VND
- RetryOnFailure: max 3 attempts, exponential backoff

## Language decisions
→ Tách Refund (sau settle) vs Reversal (trước settle) — KHÔNG dùng lẫn
→ Edge cases: PartialRefund · CrossCurrencyRefund (FX tại lúc settle)
```

#### `src/billing/domain/refund_request.py` (output DEV)
```python
class RefundRequest:
    @classmethod
    def request(cls, order, payment, amount, reason):
        # Invariant 1
        if amount <= Money.zero():
            raise InvalidRefundAmount("Must be positive")
        # Invariant 2
        if amount > payment.amount:
            raise InvalidRefundAmount("Exceeds paid amount")
        # Invariant 3
        if order.settled_at < datetime.utcnow() - timedelta(days=30):
            raise OutsideEligibilityWindow()

        instance = cls(id=RefundId.new(), state=State.PENDING, ...)
        instance._record(RefundRequested(...))
        return instance
```

#### `incidents/INC-8842.md` (output OPS — feed back về Discovery)
```markdown
# INC-8842 · Refund failure spike (AmEx 3DS)

## Root cause (5-why)
1. Refund fail vì gateway timeout
2. Timeout vì AmEx 3DS step lâu hơn Visa
3. Domain model giả định mọi card cùng latency
4. Discovery chưa surface card-type-specific timing
5. Không có domain event RefundDelayed để tách "delay" khác "failed"

## Action items → Discovery loop
- [ ] Add domain event RefundDelayed vào glossary
- [ ] Aggregate RefundRequest tách state Delayed ≠ Failed
- [ ] Update event storming: card-type-specific timing
- [ ] Runbook: timeout config per card scheme

## Loop
→ Feed insight vào sprint Discovery kế tiếp (OPS → DISC)
```

---

## 6. Khi nào dùng / KHÔNG dùng pipeline này

### 6.1. Phù hợp khi
- Greenfield hoặc brownfield vừa, domain tương đối ổn định
- Team đã buy-in DDD và AI-augmented dev
- Có Domain Expert sẵn sàng (1-2 buổi/sprint)
- Tính năng vừa-lớn (5-13 SP), đụng 2-3 bounded context
- Single-team hoặc 2-3 team coordination

### 6.2. KHÔNG dùng khi
- Dự án CRUD nhỏ, single context — overhead lớn hơn giá trị
- Hotfix khẩn cấp — dùng **Fast Lane** thay thế
- Domain còn rất mơ hồ (R&D phase) — DDD đòi hỏi domain đã đủ rõ
- Team < 3 người — overhead governance không bù được
- Không có Domain Expert — DDD mất linh hồn
- Hệ thống legacy lớn không thể refactor — strangler fig pattern phù hợp hơn

### 6.3. Fast Lane — khi nào kích hoạt

Bypass pipeline đầy đủ, đi tắt: `USER → DEV → LINT → UIT → PR → ROLL`

| Trường hợp | Có dùng Fast Lane? |
|---|---|
| Sửa typo trong UI text | ✓ |
| Bump version dependency (patch) | ✓ |
| Config-only change (env var, feature flag default) | ✓ |
| Hotfix prod incident (P0/P1) | ✓ với manual override + post-mortem bắt buộc |
| Fix bug trong domain logic | ✗ — đi đầy đủ pipeline |
| Thêm field vào domain event | ✗ — phá schema, đi qua DISC + CT |
| Sửa aggregate invariant | ✗ — đi đầy đủ |

**Kỷ luật Fast Lane**:
- Cần manual override (không tự động)
- Audit log lưu vào S3
- Post-mortem bắt buộc nếu **> 1 lần/sprint** (chỉ ra workflow chính có vấn đề)
- Không skip LINT (vẫn phải qua pre-commit hook)

---

## 7. Tham khảo nhanh — lệnh CLI theo phase

| Phase | Skill / Command | Vai trò |
|---|---|---|
| 0 DISC | `/superpowers:brainstorm` + `/openspec:propose` | Event storming + glossary/context-map delta |
| 1 PS | `/openspec:propose <name> strategic plan` | Strategic design + ADR |
| 1.5 SPIKE | `git checkout -b spike/...` + `/superpowers:brainstorm` | Throwaway PoC |
| 1 PT | `/superpowers:writing-plans` + `plan-review-gate` | Tactical plan + human gate |
| 2 DEV | `/superpowers:executing-plans` + `test-driven-development` + `/openspec:apply` | Code execution + TDD |
| 2 LINT | (auto pre-commit hook) | `npm run lint:arch && lint:glossary` |
| 3 SBE | `/superpowers:test-driven-development` | BDD `.feature` |
| 3 UIT | `npm run test:all` | jest · behave · playwright |
| 3 CT | `npm run test:contract` | Pact verification |
| 4 CR | `/superpowers:requesting-code-review` | Quality + security |
| 4 CMP | `npm run security:all` | SAST · DAST · SBOM · PII |
| 4 DMR | `/superpowers:requesting-code-review --focus domain-model` | Aggregate · ACL · drift |
| 5 PR | `/commit-commands:commit-push-pr` + CodeBuild + CodePipeline | Commit + PR + deploy staging |
| 5.5 ROLL | CodeDeploy canary + LaunchDarkly | Gradual rollout với SLO gate |
| 5 SY | `/openspec:archive` + Glue Schema Registry CLI | Knowledge sync |
| 6 OPS | Datadog · PagerDuty + `/superpowers:brainstorm` | SLO monitoring + post-mortem |

---

## 8. Hooks & guardrails (`.claude/settings.json`)

```json
{
  "hooks": {
    "preToolUse": [
      {
        "match": "Bash.*git commit",
        "command": "npm run lint:arch && npm run lint:glossary"
      }
    ]
  },
  "skills": {
    "plan-review-gate": "enforced",
    "openspec-archive": "auto-on-merge"
  },
  "permissions": {
    "allow": ["Bash(npm run *)", "Bash(git *)", "Bash(gh *)", "Bash(aws *)"],
    "deny": ["Bash(rm -rf *)", "Bash(git push --force *)"]
  }
}
```

---

## 9. Trạng thái production-readiness

| Yêu cầu thực tế | V1 (gốc gist) | V2 (DDD-aware) | V3 (production-ready) |
|---|---|---|---|
| Pipeline có DDD discipline | ❌ | ✓ | ✓ |
| Spike / PoC validation | ❌ | ❌ | ✓ (I) |
| Compliance/security gate | ❌ | ❌ | ✓ (J) |
| Gradual rollout + SLO gate | ❌ | ❌ | ✓ (K) |
| Operate phase + incident loop | ❌ | ❌ | ✓ (L) |
| Fast lane cho thay đổi nhỏ | ❌ | ❌ | ✓ (M) |
| Domain Expert involvement | partial | ✓ | ✓ |
| Per-context independence | ❌ | ✓ (H) | ✓ |
| AWS deploy concrete | ❌ | partial | ✓ |

V3 đủ để đưa vào dự án DDD thực tế cho team đã buy-in DDD + Claude Code + OpenSpec/Superpowers.

---

*Tài liệu này là bản text companion của `sdlc-ddd-v3.html` — diagram tương tác có click-to-detail cho mỗi node.*
