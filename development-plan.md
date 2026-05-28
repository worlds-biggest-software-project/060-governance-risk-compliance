# Development Plan: Governance, Risk & Compliance (GRC)

> Project: Candidate #60 · Created: 2026-05-25

---

## Technology Decisions & Rationale

### Language & Framework: Python 3.12+ / Django 5.x + Django REST Framework

**Rationale:** Django is the dominant framework in the open-source GRC space (CISO Assistant uses Django + DRF, Eramba's architecture patterns map cleanly to Django's ORM). Django's mature ORM handles the ~25-table schema with migrations, the admin interface accelerates early development, and DRF provides OpenAPI-documented REST APIs from day one. Python is the strongest language for LLM integration (Anthropic SDK, LangChain, LlamaIndex), which is critical for the AI-native differentiators (regulatory change monitoring, risk narrative drafting, policy-to-control gap analysis). Type hints via mypy enforce safety without a language switch.

**Alternatives considered:**
- TypeScript/Next.js: Stronger for frontend but weaker ORM ecosystem for complex relational models; Python's AI/ML library ecosystem is unmatched.
- Go: Performant but lacks mature ORM and AI integration libraries; higher development velocity cost for a domain-heavy application.

### Database: PostgreSQL 16+

**Rationale:** Every data model suggestion uses PostgreSQL. It provides Row-Level Security for tenant isolation, JSONB with GIN indexes for flexible metadata, partitioned tables for the audit log, generated columns for computed risk scores, and recursive CTEs for hierarchy traversal. PostgreSQL is battle-tested in compliance-critical applications and is the database behind CISO Assistant, LogicGate, and Hyperproof's architectures.

### Data Model: Hybrid Relational + JSONB (Suggestion 3) with Event Audit Trail from Suggestion 2

**Rationale:** Suggestion 3 (Hybrid Relational + JSONB) is selected as the primary schema because it reduces table count to ~23 tables (40% fewer than fully normalised), accelerates MVP delivery, and handles the fundamental GRC tension where core fields are universal but detail fields vary by industry, jurisdiction, and framework. The dividing line is clear: fields used in WHERE/ORDER BY/JOIN are relational columns; fields that vary by context go in JSONB with GIN indexes.

From Suggestion 2 (Event-Sourced), we adopt the immutable append-only audit_log with JSONB change payloads, partitioned by month. Full CQRS is deferred but the event vocabulary and audit trail completeness principles are incorporated from Phase 1. This provides regulatory examination readiness without the implementation complexity of full event sourcing.

From Suggestion 4 (Graph-Relational), we adopt the graph_node/graph_edge pattern as a Phase 8 addition for blast-radius analysis, cross-framework overlap detection, and AI context retrieval. The graph layer is derived from relational data via triggers — it is an index, not a source of truth.

### Frontend: React 18+ / Next.js 15 with TypeScript

**Rationale:** React is the dominant frontend framework for B2B SaaS applications. Next.js provides server-side rendering for dashboard performance, API routes for lightweight BFF patterns, and App Router for modern data fetching. TypeScript provides type safety. The frontend communicates exclusively via the DRF REST API, enabling third-party integrations and CLI tools.

### UI Component Library: shadcn/ui + Tailwind CSS

**Rationale:** shadcn/ui provides accessible, composable components that can be customised without fighting a component library's opinions. Tailwind CSS enables rapid UI development. This combination is used by leading modern B2B SaaS tools and produces polished interfaces that compete visually with LogicGate and Hyperproof.

### Authentication: SAML 2.0 + OIDC via django-allauth + python-social-auth

**Rationale:** Enterprise buyers require SAML 2.0 (Active Directory Federation Services, Okta); modern cloud buyers require OIDC. Both are table-stakes per the standards analysis. django-allauth provides local auth; python-social-auth adds SAML/OIDC. JWT bearer tokens for API access (RFC 7519, RFC 6749).

### AI Integration: Anthropic Claude API (primary) + pluggable LLM provider

**Rationale:** Claude excels at document analysis, regulatory text interpretation, and structured output — the core AI use cases (regulatory change monitoring, risk narrative drafting, policy-to-control gap analysis). A provider-agnostic abstraction layer enables OpenAI, local models (Ollama), and future providers. Prompt caching reduces cost for repeated framework analysis.

### Deployment: Docker Compose (self-hosted) + optional managed SaaS

**Rationale:** Docker-based deployment matches CISO Assistant's easiest-in-class deployment model. docker-compose.yml with PostgreSQL, Redis (task queue), and the application container enables single-command deployment. Kubernetes Helm chart for production scaling. SaaS tier deferred to post-MVP.

### Task Queue: Celery + Redis

**Rationale:** AI-powered features (regulatory scanning, control testing, evidence collection) are long-running background tasks. Celery is the standard Django async task framework. Redis provides both the message broker and caching layer.

### Object Storage: S3-compatible (MinIO for self-hosted, AWS S3 for SaaS)

**Rationale:** Evidence documents, policy PDFs, and audit reports require object storage. S3 API compatibility ensures portability between self-hosted (MinIO) and cloud deployments.

### Testing: pytest + factory_boy + Playwright

**Rationale:** pytest is the Python testing standard. factory_boy generates realistic GRC test data (risks, controls, frameworks). Playwright provides end-to-end browser testing for workflow-heavy features (policy approval, audit engagement lifecycle).

---

## Project Structure

```
grc-platform/
├── docker-compose.yml
├── Dockerfile
├── pyproject.toml
├── README.md
├── .env.example
├── backend/
│   ├── manage.py
│   ├── config/
│   │   ├── settings/
│   │   │   ├── base.py
│   │   │   ├── development.py
│   │   │   ├── production.py
│   │   │   └── test.py
│   │   ├── urls.py
│   │   ├── wsgi.py
│   │   └── asgi.py
│   ├── core/                       # Shared utilities, base models, tenant context
│   │   ├── models.py               # BaseModel with org_id, timestamps, audit mixin
│   │   ├── middleware.py           # Tenant context middleware (sets RLS org_id)
│   │   ├── permissions.py          # RBAC permission classes
│   │   ├── pagination.py
│   │   └── audit.py                # Audit log mixin (writes to audit_log)
│   ├── organizations/              # Tenant, user, role management
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   └── urls.py
│   ├── risks/                      # Risk register, risk categories, risk matrix
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── services.py             # Risk scoring logic, review scheduling
│   │   └── urls.py
│   ├── controls/                   # Control library, control testing
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   └── urls.py
│   ├── frameworks/                 # Compliance frameworks, requirements, crosswalks
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── importers/              # OSCAL importer, CSV importer
│   │   └── urls.py
│   ├── policies/                   # Policy lifecycle, attestations
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   └── urls.py
│   ├── audits/                     # Audit plans, engagements, findings, CAPs
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   └── urls.py
│   ├── vendors/                    # Third-party risk management
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   └── urls.py
│   ├── incidents/                  # Incident management
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   └── urls.py
│   ├── evidence/                   # Evidence repository, file management
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   └── urls.py
│   ├── regulatory/                 # Regulatory change tracking
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   └── urls.py
│   ├── ai/                         # AI service layer
│   │   ├── providers/              # LLM provider abstraction
│   │   ├── services/               # Risk analysis, regulatory monitoring, gap analysis
│   │   └── tasks.py                # Celery tasks for AI background jobs
│   ├── integrations/               # External system connectors
│   │   ├── connectors/             # AWS, GCP, GitHub, Okta, Jira connectors
│   │   └── tasks.py                # Evidence collection Celery tasks
│   └── tests/
│       ├── conftest.py
│       ├── factories.py            # factory_boy factories
│       ├── test_risks/
│       ├── test_controls/
│       ├── test_frameworks/
│       ├── test_policies/
│       ├── test_audits/
│       ├── test_vendors/
│       ├── test_incidents/
│       └── test_ai/
├── frontend/
│   ├── package.json
│   ├── tsconfig.json
│   ├── next.config.js
│   ├── src/
│   │   ├── app/                    # Next.js App Router
│   │   │   ├── layout.tsx
│   │   │   ├── (auth)/             # Login, SSO callback
│   │   │   ├── (dashboard)/        # Authenticated pages
│   │   │   │   ├── risks/
│   │   │   │   ├── controls/
│   │   │   │   ├── frameworks/
│   │   │   │   ├── policies/
│   │   │   │   ├── audits/
│   │   │   │   ├── vendors/
│   │   │   │   ├── incidents/
│   │   │   │   ├── regulatory/
│   │   │   │   └── settings/
│   │   │   └── api/                # BFF routes if needed
│   │   ├── components/
│   │   │   ├── ui/                 # shadcn/ui components
│   │   │   ├── forms/              # Risk form, control form, etc.
│   │   │   ├── tables/             # Data tables with sorting/filtering
│   │   │   ├── charts/             # Risk heat maps, compliance gauges
│   │   │   └── layout/             # Sidebar, header, breadcrumbs
│   │   ├── lib/
│   │   │   ├── api/                # API client (fetch wrappers, types)
│   │   │   ├── auth/               # Auth context, session management
│   │   │   └── utils/
│   │   └── types/                  # TypeScript interfaces matching API schemas
│   └── e2e/                        # Playwright end-to-end tests
│       ├── risks.spec.ts
│       ├── controls.spec.ts
│       ├── policies.spec.ts
│       └── audits.spec.ts
├── framework-packs/                # Built-in compliance framework data
│   ├── iso-27001-2022.json
│   ├── soc2-2022.json
│   ├── nist-csf-2.0.json
│   ├── gdpr.json
│   ├── eu-ai-act.json
│   ├── dora.json
│   ├── pci-dss-4.0.json
│   └── nist-800-53-r5.json
├── docs/
│   ├── api/                        # Generated OpenAPI docs
│   ├── deployment/
│   └── user-guide/
└── helm/                           # Kubernetes Helm chart
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
```

---

## Phase Dependency Graph

```
Phase 1: Foundation & Core Infrastructure
    │
    ├──> Phase 2: Risk Register & Control Library
    │       │
    │       ├──> Phase 3: Compliance Framework Engine
    │       │       │
    │       │       └──> Phase 5: Assessments & Evidence
    │       │               │
    │       │               └──> Phase 7: Automated Evidence Collection
    │       │                       │
    │       │                       └──> Phase 9: Continuous Control Monitoring
    │       │
    │       └──> Phase 4: Policy & Audit Management
    │               │
    │               └──> Phase 6: Third-Party Risk & Incident Management
    │
    ├──> Phase 8: AI-Powered Features
    │       │
    │       └──> Phase 10: Graph Analytics & Advanced Reporting
    │
    └──> Phase 11: OSCAL Interoperability & MCP Server
              │
              └──> Phase 12: SaaS Packaging & Enterprise Features
```

**Critical path:** Phases 1 → 2 → 3 → 5 → 7 → 9 (infrastructure through continuous monitoring)

**Parallelisable:** Phases 4 and 3 can run in parallel after Phase 2. Phase 8 can begin after Phase 2 for basic AI features, with deeper integration as later phases deliver data. Phase 11 can begin after Phase 3.

---

## Phase 1: Foundation & Core Infrastructure

**Goal:** Establish the project skeleton, database schema, authentication, multi-tenancy, RBAC, audit logging, and deployment infrastructure. No domain features yet — this phase delivers the platform on which all GRC modules are built.

**Duration:** 3-4 weeks

### Task 1.1: Project Scaffolding & Docker Environment

**What:** Initialise the Django backend project, Next.js frontend project, Docker Compose configuration, and CI pipeline. Set up pyproject.toml with dependencies, linting (ruff), formatting (black), and type checking (mypy).

**Design:**

```python
# backend/config/settings/base.py
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "rest_framework",
    "django_filters",
    "corsheaders",
    "core",
    "organizations",
]

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "core.permissions.IsTenantMember",
    ],
    "DEFAULT_PAGINATION_CLASS": "core.pagination.StandardPagination",
    "PAGE_SIZE": 25,
    "DEFAULT_FILTER_BACKENDS": [
        "django_filters.rest_framework.DjangoFilterBackend",
        "rest_framework.filters.SearchFilter",
        "rest_framework.filters.OrderingFilter",
    ],
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
}

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.environ.get("DB_NAME", "grc"),
        "USER": os.environ.get("DB_USER", "grc"),
        "PASSWORD": os.environ.get("DB_PASSWORD", ""),
        "HOST": os.environ.get("DB_HOST", "localhost"),
        "PORT": os.environ.get("DB_PORT", "5432"),
    }
}
```

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: grc
      POSTGRES_USER: grc
      POSTGRES_PASSWORD: ${DB_PASSWORD:-devpassword}
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  backend:
    build:
      context: .
      dockerfile: Dockerfile
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./backend:/app
    ports:
      - "8000:8000"
    depends_on:
      - db
      - redis
    environment:
      - DB_HOST=db
      - REDIS_URL=redis://redis:6379/0

  frontend:
    build:
      context: ./frontend
    command: npm run dev
    volumes:
      - ./frontend:/app
      - /app/node_modules
    ports:
      - "3000:3000"

  worker:
    build:
      context: .
      dockerfile: Dockerfile
    command: celery -A config worker -l info
    volumes:
      - ./backend:/app
    depends_on:
      - db
      - redis

volumes:
  pgdata:
```

**Testing:**
- `docker compose up` starts all services without errors
- `python manage.py check` passes with no warnings
- `python manage.py migrate` runs zero migrations (no models yet) without errors
- `npm run dev` serves the Next.js app on port 3000
- `ruff check .` and `mypy .` pass with zero issues
- CI pipeline runs lint, type check, and test stages to completion

### Task 1.2: Multi-Tenancy & Organisation Model

**What:** Create the `organization` and `app_user` models with PostgreSQL Row-Level Security. Implement the tenant context middleware that sets `app.current_org_id` on every request.

**Design:**

```python
# backend/core/models.py
import uuid
from django.db import models

class BaseModel(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True

class TenantModel(BaseModel):
    organization = models.ForeignKey(
        "organizations.Organization",
        on_delete=models.CASCADE,
        related_name="%(class)ss",
    )

    class Meta:
        abstract = True

# backend/organizations/models.py
from core.models import BaseModel
from django.db import models

class Organization(BaseModel):
    name = models.CharField(max_length=255)
    slug = models.SlugField(max_length=100, unique=True)
    industry = models.CharField(max_length=100, blank=True)
    jurisdiction = models.CharField(max_length=2, blank=True)  # ISO 3166-1 alpha-2
    settings = models.JSONField(default=dict)

class AppUser(BaseModel):
    organization = models.ForeignKey(Organization, on_delete=models.CASCADE)
    email = models.EmailField(max_length=320)
    display_name = models.CharField(max_length=255)
    status = models.CharField(max_length=20, default="active")
    roles = models.JSONField(default=list)
    auth_provider = models.CharField(max_length=50, default="local")
    auth_subject = models.CharField(max_length=500, blank=True)

    class Meta:
        unique_together = [("organization", "email")]

# backend/core/middleware.py
from django.db import connection

class TenantMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if hasattr(request, "user") and hasattr(request.user, "organization_id"):
            with connection.cursor() as cursor:
                cursor.execute(
                    "SET LOCAL app.current_org_id = %s",
                    [str(request.user.organization_id)],
                )
        return self.get_response(request)
```

```sql
-- Migration: enable RLS on tenant-scoped tables
ALTER TABLE organizations_appuser ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON organizations_appuser
    USING (organization_id = current_setting('app.current_org_id')::UUID);
```

**Testing:**
- Unit test: creating an AppUser with valid Organization succeeds
- Unit test: creating an AppUser with duplicate (org, email) raises IntegrityError
- Unit test: TenantMiddleware sets `app.current_org_id` in database session
- Unit test: querying AppUser without setting org context returns empty results (RLS enforcement)
- Unit test: querying AppUser with correct org context returns only that tenant's users
- Integration test: API request with valid JWT token resolves to correct organization context

### Task 1.3: Role-Based Access Control (RBAC)

**What:** Implement a permission system with predefined roles (admin, risk_owner, compliance_manager, auditor, cro, cco, ciso, viewer) and granular permissions. Roles are stored as JSONB on the user record. DRF permission classes enforce access per endpoint.

**Design:**

```python
# backend/core/permissions.py
from rest_framework.permissions import BasePermission

PERMISSION_CATALOG = {
    "risk.read", "risk.write", "risk.delete", "risk.approve",
    "control.read", "control.write", "control.delete",
    "framework.read", "framework.write",
    "policy.read", "policy.write", "policy.approve",
    "audit.read", "audit.write", "audit.manage",
    "vendor.read", "vendor.write",
    "incident.read", "incident.write",
    "evidence.read", "evidence.write",
    "admin.users", "admin.settings",
}

DEFAULT_ROLES = {
    "admin": list(PERMISSION_CATALOG),
    "cro": ["risk.*", "control.read", "framework.read", "audit.read", "vendor.read", "incident.read"],
    "compliance_manager": ["risk.read", "control.*", "framework.*", "policy.*", "audit.read", "vendor.read"],
    "auditor": ["risk.read", "control.read", "framework.read", "audit.*", "evidence.*"],
    "risk_owner": ["risk.read", "risk.write", "control.read"],
    "viewer": ["risk.read", "control.read", "framework.read", "policy.read", "audit.read"],
}

class IsTenantMember(BasePermission):
    def has_permission(self, request, view):
        return (
            request.user
            and request.user.is_authenticated
            and hasattr(request.user, "organization_id")
        )

class HasPermission(BasePermission):
    def __init__(self, required_permission):
        self.required_permission = required_permission

    def has_permission(self, request, view):
        if not request.user or not request.user.is_authenticated:
            return False
        user_permissions = self._resolve_permissions(request.user.roles)
        return self.required_permission in user_permissions

    def _resolve_permissions(self, roles):
        permissions = set()
        for role_entry in roles:
            role_name = role_entry.get("role", "")
            role_perms = DEFAULT_ROLES.get(role_name, [])
            for perm in role_perms:
                if perm.endswith(".*"):
                    prefix = perm[:-2]
                    permissions.update(p for p in PERMISSION_CATALOG if p.startswith(prefix))
                else:
                    permissions.add(perm)
        return permissions
```

**Testing:**
- Unit test: admin role resolves to all permissions in PERMISSION_CATALOG
- Unit test: viewer role resolves to read-only permissions only
- Unit test: wildcard expansion (`risk.*`) correctly expands to all risk permissions
- Unit test: HasPermission denies access when user lacks required permission
- Unit test: HasPermission allows access when user has required permission
- Unit test: scoped roles (role with `scope.type = "business_unit"`) restrict to that scope
- Integration test: API endpoint decorated with `@permission_required("risk.write")` returns 403 for viewer, 200 for risk_owner

### Task 1.4: Audit Trail & Logging

**What:** Implement an append-only audit log that captures every state-changing operation (create, update, delete, approve, attest) with actor, timestamp, entity reference, and JSONB change diff. Partitioned by month for performance and retention management.

**Design:**

```python
# backend/core/audit.py
from django.db import models
import uuid

class AuditLog(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    organization_id = models.UUIDField()
    actor_id = models.UUIDField(null=True)
    actor_email = models.EmailField(max_length=320, blank=True)
    action = models.CharField(max_length=50)  # create, update, delete, approve, attest
    entity_type = models.CharField(max_length=50)  # risk, control, policy, etc.
    entity_id = models.UUIDField()
    entity_ref_id = models.CharField(max_length=100, blank=True)
    changes = models.JSONField(null=True)  # {"field": {"old": "...", "new": "..."}}
    ip_address = models.GenericIPAddressField(null=True)
    timestamp = models.DateTimeField(auto_now_add=True)

    class Meta:
        managed = False  # partitioned table created via raw SQL migration
        indexes = [
            models.Index(fields=["organization_id", "-timestamp"]),
            models.Index(fields=["entity_type", "entity_id"]),
            models.Index(fields=["actor_id"]),
        ]

class AuditMixin:
    """Mixin for DRF views to automatically log state changes."""

    def perform_create(self, serializer):
        instance = serializer.save()
        self._write_audit_log("create", instance)

    def perform_update(self, serializer):
        old_data = serializer.instance.__class__.objects.get(pk=serializer.instance.pk)
        instance = serializer.save()
        changes = self._compute_diff(old_data, instance)
        self._write_audit_log("update", instance, changes=changes)

    def perform_destroy(self, instance):
        self._write_audit_log("delete", instance)
        instance.delete()

    def _write_audit_log(self, action, instance, changes=None):
        AuditLog.objects.create(
            organization_id=instance.organization_id,
            actor_id=getattr(self.request.user, "id", None),
            actor_email=getattr(self.request.user, "email", ""),
            action=action,
            entity_type=instance.__class__.__name__.lower(),
            entity_id=instance.id,
            entity_ref_id=getattr(instance, "ref_id", ""),
            changes=changes,
            ip_address=self._get_client_ip(),
        )
```

```sql
-- Raw SQL migration: create partitioned audit_log table
CREATE TABLE core_auditlog (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    actor_id        UUID,
    actor_email     VARCHAR(320),
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    entity_ref_id   VARCHAR(100),
    changes         JSONB,
    ip_address      INET,
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (timestamp);

-- Create partitions for current year
CREATE TABLE core_auditlog_2026_01 PARTITION OF core_auditlog
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... (generate monthly partitions programmatically)
```

**Testing:**
- Unit test: creating a risk via API produces an AuditLog entry with action="create"
- Unit test: updating a risk captures old and new values in changes JSONB
- Unit test: deleting a risk produces an AuditLog entry with action="delete"
- Unit test: audit log entries include correct actor_id and ip_address from request
- Unit test: audit log entries are immutable — UPDATE on audit_log raises error (via DB trigger or application enforcement)
- Unit test: querying audit trail for a specific entity returns events in chronological order
- Performance test: inserting 10,000 audit entries completes within 2 seconds (partition performance)

### Task 1.5: Authentication & JWT

**What:** Implement local authentication (email/password), JWT token issuance, and the authentication middleware. Set up django-allauth for local auth and prepare SAML/OIDC hooks (actual SSO integration in Phase 12).

**Design:**

```python
# backend/config/settings/base.py (additions)
from datetime import timedelta

SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=30),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
    "ROTATE_REFRESH_TOKENS": True,
    "SIGNING_KEY": os.environ.get("JWT_SECRET_KEY"),
    "AUTH_HEADER_TYPES": ("Bearer",),
}

# backend/organizations/views.py
from rest_framework_simplejwt.views import TokenObtainPairView
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer

class GRCTokenObtainSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        token["org_id"] = str(user.organization_id)
        token["email"] = user.email
        token["roles"] = user.roles
        return token
```

**Testing:**
- Unit test: login with valid credentials returns access and refresh tokens
- Unit test: login with invalid credentials returns 401
- Unit test: JWT access token contains org_id, email, and roles claims
- Unit test: expired access token returns 401 on subsequent API call
- Unit test: refresh token returns new access token
- Unit test: rotated refresh tokens invalidate old refresh token
- Integration test: full login flow via API client returns valid token, subsequent authenticated request succeeds

### Task 1.6: Frontend Shell & Layout

**What:** Set up the Next.js project with App Router, authentication context, sidebar navigation, header, and base layout. Implement the API client layer with typed fetch wrappers. No domain pages yet — just the authenticated shell.

**Design:**

```typescript
// frontend/src/lib/api/client.ts
const API_BASE = process.env.NEXT_PUBLIC_API_URL || "http://localhost:8000/api/v1";

interface ApiOptions {
  method?: string;
  body?: unknown;
  headers?: Record<string, string>;
}

export async function apiClient<T>(path: string, options: ApiOptions = {}): Promise<T> {
  const token = getAccessToken();
  const response = await fetch(`${API_BASE}${path}`, {
    method: options.method || "GET",
    headers: {
      "Content-Type": "application/json",
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options.headers,
    },
    body: options.body ? JSON.stringify(options.body) : undefined,
  });

  if (response.status === 401) {
    await refreshToken();
    return apiClient<T>(path, options);
  }

  if (!response.ok) {
    throw new ApiError(response.status, await response.json());
  }

  return response.json();
}

// frontend/src/components/layout/sidebar.tsx
const NAVIGATION = [
  { name: "Dashboard", href: "/", icon: LayoutDashboard },
  { name: "Risks", href: "/risks", icon: AlertTriangle },
  { name: "Controls", href: "/controls", icon: Shield },
  { name: "Frameworks", href: "/frameworks", icon: BookOpen },
  { name: "Policies", href: "/policies", icon: FileText },
  { name: "Audits", href: "/audits", icon: ClipboardCheck },
  { name: "Vendors", href: "/vendors", icon: Building2 },
  { name: "Incidents", href: "/incidents", icon: AlertCircle },
  { name: "Regulatory", href: "/regulatory", icon: Scale },
  { name: "Settings", href: "/settings", icon: Settings },
];
```

**Testing:**
- Visual test: authenticated shell renders sidebar, header, and content area
- Unit test: unauthenticated access redirects to login page
- Unit test: API client attaches Bearer token to requests
- Unit test: API client handles 401 by refreshing token and retrying
- Unit test: sidebar navigation links render for all modules
- E2E test (Playwright): login flow through UI → dashboard loads with sidebar visible

### Definition of Done — Phase 1
- [ ] `docker compose up` starts all services (PostgreSQL, Redis, backend, frontend, worker)
- [ ] Database migrations create organization, app_user, and audit_log tables
- [ ] PostgreSQL RLS enforces tenant isolation (verified by test)
- [ ] JWT authentication flow works end-to-end (login, access, refresh, expiry)
- [ ] RBAC permission system enforces role-based access (verified by tests)
- [ ] Audit log captures all state changes with actor/timestamp/changes
- [ ] Frontend shell with sidebar navigation renders for authenticated users
- [ ] OpenAPI schema generated and browsable at `/api/schema/swagger-ui/`
- [ ] CI pipeline passes: lint, type check, unit tests, integration tests
- [ ] README with setup instructions for local development

---

## Phase 2: Risk Register & Control Library

**Goal:** Implement the two foundational GRC domain models — the risk register and the control library — with full CRUD, many-to-many linking between risks and controls, risk scoring, and control testing. These are the entities that every subsequent phase builds upon.

**Duration:** 3-4 weeks

### Task 2.1: Risk Register Model & API

**What:** Create the `risk` model with configurable inherent/residual/target scoring, treatment types (ISO 31000: accept, mitigate, transfer, avoid, escalate), status lifecycle, review scheduling, and JSONB metadata for variable fields. Full CRUD API with filtering, sorting, and search.

**Design:**

```python
# backend/risks/models.py
from core.models import TenantModel
from django.db import models

class Risk(TenantModel):
    TREATMENT_CHOICES = [
        ("accept", "Accept"), ("mitigate", "Mitigate"),
        ("transfer", "Transfer"), ("avoid", "Avoid"), ("escalate", "Escalate"),
    ]
    STATUS_CHOICES = [
        ("identified", "Identified"), ("assessed", "Assessed"),
        ("treating", "Treating"), ("monitoring", "Monitoring"),
        ("accepted", "Accepted"), ("closed", "Closed"), ("escalated", "Escalated"),
    ]

    ref_id = models.CharField(max_length=50)
    title = models.CharField(max_length=500)
    description = models.TextField(blank=True)
    category = models.CharField(max_length=100, blank=True)
    business_unit = models.ForeignKey("organizations.BusinessUnit", null=True, blank=True, on_delete=models.SET_NULL)

    inherent_likelihood = models.IntegerField()
    inherent_impact = models.IntegerField()
    residual_likelihood = models.IntegerField(null=True, blank=True)
    residual_impact = models.IntegerField(null=True, blank=True)
    target_likelihood = models.IntegerField(null=True, blank=True)
    target_impact = models.IntegerField(null=True, blank=True)

    treatment_type = models.CharField(max_length=20, choices=TREATMENT_CHOICES, default="mitigate")
    risk_owner = models.ForeignKey("organizations.AppUser", null=True, blank=True, on_delete=models.SET_NULL)
    status = models.CharField(max_length=30, choices=STATUS_CHOICES, default="identified")
    next_review_date = models.DateField(null=True, blank=True)
    metadata = models.JSONField(default=dict)

    @property
    def inherent_score(self):
        return self.inherent_likelihood * self.inherent_impact

    @property
    def residual_score(self):
        if self.residual_likelihood and self.residual_impact:
            return self.residual_likelihood * self.residual_impact
        return None

    class Meta:
        unique_together = [("organization", "ref_id")]
        indexes = [
            models.Index(fields=["organization", "status"]),
            models.Index(fields=["organization", "-updated_at"]),
        ]

# backend/risks/views.py
from rest_framework import viewsets, filters
from django_filters.rest_framework import DjangoFilterBackend
from core.audit import AuditMixin

class RiskViewSet(AuditMixin, viewsets.ModelViewSet):
    serializer_class = RiskSerializer
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ["status", "treatment_type", "category", "risk_owner"]
    search_fields = ["ref_id", "title", "description"]
    ordering_fields = ["ref_id", "inherent_score", "residual_score", "status", "next_review_date"]
    ordering = ["-updated_at"]

    def get_queryset(self):
        return Risk.objects.filter(organization=self.request.user.organization)
```

**Testing:**
- Unit test: creating a risk with valid data returns 201 with all fields populated
- Unit test: inherent_score computed correctly (likelihood * impact)
- Unit test: residual_score is null when residual fields are empty
- Unit test: ref_id uniqueness enforced per organization (duplicate returns 400)
- Unit test: filtering by status returns only matching risks
- Unit test: search by title returns partial matches
- Unit test: ordering by residual_score descending shows highest risks first
- Unit test: risk metadata JSONB stores and retrieves arbitrary key-value pairs
- Unit test: treatment_type validation rejects invalid values
- Unit test: status transition validation (e.g., cannot go from "closed" to "identified")
- Integration test: creating a risk generates audit log entry
- Factory: `RiskFactory` generates realistic risk data with random scores and categories

### Task 2.2: Control Library Model & API

**What:** Create the `control` model with control types (preventive, detective, corrective, directive, compensating), automation levels, testing frequency, effectiveness ratings, and JSONB metadata. Full CRUD API with filtering.

**Design:**

```python
# backend/controls/models.py
from core.models import TenantModel
from django.db import models

class Control(TenantModel):
    TYPE_CHOICES = [
        ("preventive", "Preventive"), ("detective", "Detective"),
        ("corrective", "Corrective"), ("directive", "Directive"),
        ("compensating", "Compensating"),
    ]
    AUTOMATION_CHOICES = [
        ("manual", "Manual"), ("semi_automated", "Semi-Automated"),
        ("automated", "Automated"),
    ]
    EFFECTIVENESS_CHOICES = [
        ("effective", "Effective"), ("partially_effective", "Partially Effective"),
        ("ineffective", "Ineffective"), ("not_assessed", "Not Assessed"),
    ]

    ref_id = models.CharField(max_length=50)
    title = models.CharField(max_length=500)
    description = models.TextField(blank=True)
    control_type = models.CharField(max_length=30, choices=TYPE_CHOICES, default="preventive")
    automation_level = models.CharField(max_length=20, choices=AUTOMATION_CHOICES, default="manual")
    frequency = models.CharField(max_length=20, default="continuous")
    owner = models.ForeignKey("organizations.AppUser", null=True, blank=True, on_delete=models.SET_NULL)
    status = models.CharField(max_length=20, default="active")
    effectiveness = models.CharField(max_length=20, choices=EFFECTIVENESS_CHOICES, default="not_assessed")
    last_test_date = models.DateField(null=True, blank=True)
    next_test_date = models.DateField(null=True, blank=True)
    metadata = models.JSONField(default=dict)

    class Meta:
        unique_together = [("organization", "ref_id")]
```

**Testing:**
- Unit test: creating a control with valid data returns 201
- Unit test: ref_id uniqueness enforced per organization
- Unit test: control_type validation rejects invalid values
- Unit test: effectiveness defaults to "not_assessed" on creation
- Unit test: filtering by control_type, status, and effectiveness works correctly
- Unit test: JSONB metadata stores implementation evidence and testing procedures
- Integration test: audit log captures control creation and updates

### Task 2.3: Risk-Control Many-to-Many Linking

**What:** Implement the `risk_control` junction table and API endpoints for linking/unlinking controls to risks with relationship type (mitigates, monitors, compensates). Provide nested API views to show a risk's linked controls and a control's linked risks.

**Design:**

```python
# backend/risks/models.py (addition)
class RiskControl(models.Model):
    risk = models.ForeignKey(Risk, on_delete=models.CASCADE, related_name="risk_controls")
    control = models.ForeignKey("controls.Control", on_delete=models.CASCADE, related_name="control_risks")
    relationship = models.CharField(max_length=20, default="mitigates")
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = [("risk", "control")]

# API endpoint: POST /api/v1/risks/{risk_id}/controls/
# Body: {"control_id": "uuid", "relationship": "mitigates"}
# Returns: 201 with link data

# API endpoint: GET /api/v1/risks/{risk_id}/controls/
# Returns: list of linked controls with relationship type

# API endpoint: DELETE /api/v1/risks/{risk_id}/controls/{control_id}/
# Returns: 204
```

**Testing:**
- Unit test: linking a control to a risk creates a RiskControl record
- Unit test: duplicate link (same risk + control) returns 400
- Unit test: GET /risks/{id}/controls/ returns all linked controls with relationship type
- Unit test: GET /controls/{id}/risks/ returns all linked risks
- Unit test: deleting a link removes the RiskControl record but not the risk or control
- Unit test: deleting a risk cascades to delete its RiskControl records
- Unit test: linking a control from a different organization returns 404 (tenant isolation)

### Task 2.4: Control Testing & Evidence Recording

**What:** Implement the `control_test` model for recording control test results (pass/fail/partial/not_applicable) with test methodology, evidence notes, automated flag, and JSONB details. Update control effectiveness based on test results.

**Design:**

```python
# backend/controls/models.py (addition)
class ControlTest(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    control = models.ForeignKey(Control, on_delete=models.CASCADE, related_name="tests")
    tested_by = models.ForeignKey("organizations.AppUser", null=True, on_delete=models.SET_NULL)
    test_date = models.DateTimeField(auto_now_add=True)
    result = models.CharField(max_length=20)  # pass, fail, partial, not_applicable
    automated = models.BooleanField(default=False)
    details = models.JSONField(default=dict)
    # details: {methodology, evidence_notes, sample_size, population_size, exceptions_found, source_system}
    created_at = models.DateTimeField(auto_now_add=True)

# backend/controls/services.py
def update_control_effectiveness(control):
    """Recalculate effectiveness based on last N test results."""
    recent_tests = control.tests.order_by("-test_date")[:5]
    if not recent_tests:
        control.effectiveness = "not_assessed"
    else:
        pass_count = sum(1 for t in recent_tests if t.result == "pass")
        ratio = pass_count / len(recent_tests)
        if ratio >= 0.8:
            control.effectiveness = "effective"
        elif ratio >= 0.5:
            control.effectiveness = "partially_effective"
        else:
            control.effectiveness = "ineffective"
    control.last_test_date = recent_tests[0].test_date if recent_tests else None
    control.save()
```

**Testing:**
- Unit test: recording a test result creates a ControlTest record
- Unit test: recording a "pass" result for a previously untested control sets effectiveness to "effective"
- Unit test: recording 3 failures out of 5 tests sets effectiveness to "partially_effective"
- Unit test: control's last_test_date updates to most recent test date
- Unit test: control_test.details JSONB stores methodology and sample size data
- Unit test: test results are filterable by result type and date range
- Unit test: test history for a control returns results in reverse chronological order

### Task 2.5: Risk Register & Control Library Frontend

**What:** Build the frontend pages for risk register (list, detail, create, edit) and control library (list, detail, create, edit) with data tables, forms, risk score display (colour-coded), and the control linking UI.

**Design:**

```typescript
// frontend/src/app/(dashboard)/risks/page.tsx
// Data table with columns: Ref ID, Title, Category, Inherent Score, Residual Score,
//   Treatment, Owner, Status, Next Review
// Colour-coded risk scores: red (>15), orange (10-15), yellow (5-9), green (<5)
// Filters: status, category, treatment_type, owner
// Actions: Create Risk, Export CSV

// frontend/src/app/(dashboard)/risks/[id]/page.tsx
// Detail view with tabs:
//   - Overview: risk fields, score display, treatment plan
//   - Controls: linked controls with relationship type, add/remove controls
//   - History: audit trail for this risk
//   - Metadata: JSONB metadata viewer/editor

// frontend/src/components/charts/risk-heatmap.tsx
// 5x5 risk heat map showing risks positioned by likelihood (y) x impact (x)
// Risks displayed as dots; clicking a cell shows risks at that position
```

**Testing:**
- E2E test: navigate to /risks → data table loads with risk list
- E2E test: click "Create Risk" → form appears → fill fields → submit → risk appears in list
- E2E test: click a risk → detail page loads → tabs are navigable
- E2E test: link a control to a risk → control appears in risk's Controls tab
- E2E test: risk scores display with correct colour coding
- Visual regression test: risk heat map renders with correct axis labels and colours
- Accessibility test: forms have proper labels, tables have proper headers

### Definition of Done — Phase 2
- [ ] Risk register CRUD API with filtering, sorting, search, and pagination
- [ ] Control library CRUD API with filtering, sorting, search, and pagination
- [ ] Risk-Control many-to-many linking with relationship types
- [ ] Control testing with effectiveness calculation
- [ ] Audit log captures all risk and control state changes
- [ ] Frontend: risk list, risk detail (with tabs), risk creation/edit form
- [ ] Frontend: control list, control detail, control creation/edit form
- [ ] Frontend: risk heat map component
- [ ] 90%+ test coverage on risk and control models and API views
- [ ] factory_boy factories for Risk, Control, RiskControl, ControlTest

---

## Phase 3: Compliance Framework Engine

**Goal:** Build the compliance framework model with hierarchical requirements, the control-to-requirement mapping system, cross-framework crosswalks, and the framework import pipeline. Ship with ISO 27001, SOC 2, NIST CSF 2.0, GDPR, and EU AI Act as built-in framework packs.

**Duration:** 3-4 weeks

### Task 3.1: Framework & Requirement Models

**What:** Create the `framework` and `framework_requirement` models with hierarchical requirement trees (parent/child with materialised path), OSCAL identifiers for interoperability, and JSONB extended_fields for framework-specific metadata.

**Design:**

```python
# backend/frameworks/models.py
class Framework(BaseModel):
    organization = models.ForeignKey("organizations.Organization", null=True, blank=True, on_delete=models.CASCADE)
    name = models.CharField(max_length=255)
    version = models.CharField(max_length=50, blank=True)
    provider = models.CharField(max_length=255, blank=True)
    framework_type = models.CharField(max_length=30, default="standard")
    jurisdiction = models.CharField(max_length=2, blank=True)
    status = models.CharField(max_length=20, default="active")
    oscal_catalog_id = models.CharField(max_length=255, blank=True)
    metadata = models.JSONField(default=dict)

class FrameworkRequirement(BaseModel):
    framework = models.ForeignKey(Framework, on_delete=models.CASCADE, related_name="requirements")
    parent = models.ForeignKey("self", null=True, blank=True, on_delete=models.CASCADE, related_name="children")
    ref_id = models.CharField(max_length=100)
    title = models.CharField(max_length=500)
    description = models.TextField(blank=True)
    path = models.TextField(blank=True)  # materialised path: /A/A.5/A.5.1
    depth = models.IntegerField(default=0)
    sort_order = models.IntegerField(default=0)
    oscal_control_id = models.CharField(max_length=255, blank=True)
    extended_fields = models.JSONField(default=dict)

    class Meta:
        unique_together = [("framework", "ref_id")]
```

**Testing:**
- Unit test: creating a framework with requirements returns full hierarchy
- Unit test: materialised path is correctly computed on save (e.g., `/A/A.5/A.5.1`)
- Unit test: querying requirements by path prefix returns subtree efficiently
- Unit test: deleting a parent requirement cascades to children
- Unit test: framework with organization=NULL is a global/system framework visible to all tenants
- Unit test: OSCAL identifiers are stored and retrievable

### Task 3.2: Control-Requirement Mapping

**What:** Implement the `control_requirement` junction table linking controls to framework requirements with coverage status (mapped, partial, planned, not_applicable). Build API endpoints for mapping/unmapping and querying coverage.

**Design:**

```python
# backend/frameworks/models.py (addition)
class ControlRequirement(models.Model):
    control = models.ForeignKey("controls.Control", on_delete=models.CASCADE)
    requirement = models.ForeignKey(FrameworkRequirement, on_delete=models.CASCADE)
    coverage_status = models.CharField(max_length=20, default="mapped")
    notes = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = [("control", "requirement")]

# API endpoint: GET /api/v1/frameworks/{id}/coverage/
# Returns: {total_requirements, mapped, partial, planned, not_applicable, unmapped, coverage_pct}
```

**Testing:**
- Unit test: mapping a control to a requirement creates ControlRequirement record
- Unit test: coverage statistics correctly count mapped/partial/planned/unmapped requirements
- Unit test: a single control can satisfy requirements across multiple frameworks (multi-framework overlap)
- Unit test: GET /frameworks/{id}/requirements/ returns requirements with their mapped controls
- Unit test: GET /controls/{id}/requirements/ returns all requirements this control satisfies
- Unit test: coverage_pct calculation is accurate (mapped + partial) / total * 100

### Task 3.3: Cross-Framework Crosswalks

**What:** Implement the `requirement_crosswalk` table for mapping equivalent/related requirements across frameworks (e.g., ISO 27001 A.8.1 maps to NIST CSF PR.DS-1). Support mapping types (equivalent, superset, subset, related) and confidence levels.

**Design:**

```python
# backend/frameworks/models.py (addition)
class RequirementCrosswalk(models.Model):
    source_requirement = models.ForeignKey(FrameworkRequirement, on_delete=models.CASCADE, related_name="crosswalks_from")
    target_requirement = models.ForeignKey(FrameworkRequirement, on_delete=models.CASCADE, related_name="crosswalks_to")
    mapping_type = models.CharField(max_length=20, default="equivalent")
    confidence = models.CharField(max_length=10, default="high")
    source = models.CharField(max_length=100, blank=True)  # nist_cprt, manual, ai_suggested

    class Meta:
        unique_together = [("source_requirement", "target_requirement")]

# API endpoint: GET /api/v1/frameworks/{id}/crosswalks/?target_framework={id2}
# Returns: list of crosswalk mappings between two frameworks
```

**Testing:**
- Unit test: creating a crosswalk between two requirements from different frameworks succeeds
- Unit test: duplicate crosswalk returns 400
- Unit test: querying crosswalks between ISO 27001 and NIST CSF returns mapped pairs
- Unit test: crosswalk source attribution tracks whether mapping was manual, from NIST CPRT, or AI-suggested
- Unit test: crosswalks with confidence="low" are distinguishable from high-confidence mappings

### Task 3.4: Framework Import Pipeline

**What:** Build a JSON-based framework importer that reads framework pack files (ISO 27001, SOC 2, NIST CSF 2.0, GDPR, EU AI Act) and creates framework + requirement records. Support OSCAL Catalog JSON import for NIST frameworks.

**Design:**

```python
# backend/frameworks/importers/json_importer.py
def import_framework_pack(file_path: str, organization_id: str | None = None) -> Framework:
    """Import a framework from a JSON pack file.

    Pack file format:
    {
        "name": "ISO/IEC 27001:2022",
        "version": "2022",
        "provider": "ISO/IEC",
        "framework_type": "standard",
        "jurisdiction": null,
        "oscal_catalog_id": "...",
        "requirements": [
            {
                "ref_id": "A.5",
                "title": "Organizational controls",
                "children": [
                    {
                        "ref_id": "A.5.1",
                        "title": "Policies for information security",
                        "description": "...",
                        "extended_fields": {"implementation_guidance": "..."}
                    }
                ]
            }
        ]
    }
    """
    with open(file_path) as f:
        data = json.load(f)
    framework = Framework.objects.create(
        organization_id=organization_id,
        name=data["name"],
        version=data.get("version", ""),
        provider=data.get("provider", ""),
        framework_type=data.get("framework_type", "standard"),
    )
    _import_requirements(framework, data["requirements"], parent=None, depth=0, path="")
    return framework

# backend/frameworks/importers/oscal_importer.py
def import_oscal_catalog(catalog_json: dict, organization_id: str | None = None) -> Framework:
    """Import a framework from NIST OSCAL Catalog JSON."""
    # Maps OSCAL catalog.groups[].controls[] to Framework + FrameworkRequirement
    pass

# Management command:
# python manage.py import_framework framework-packs/iso-27001-2022.json
```

**Testing:**
- Unit test: importing ISO 27001 pack creates 1 framework + 93 Annex A requirements with correct hierarchy
- Unit test: importing NIST CSF 2.0 creates framework with 6 functions, 22 categories, 106 subcategories
- Unit test: imported requirements have correct parent-child relationships and materialised paths
- Unit test: re-importing the same framework is idempotent (updates existing, does not duplicate)
- Unit test: OSCAL catalog import correctly maps OSCAL control IDs to requirement oscal_control_id
- Integration test: `python manage.py import_framework` command succeeds for all 5 built-in packs
- Data integrity test: all imported requirements have valid parent references (no orphans)

### Task 3.5: Framework Management Frontend

**What:** Build the compliance framework UI: framework list, requirement tree browser, control-requirement mapping interface, coverage dashboard with gap analysis visualisation, and cross-framework overlap view.

**Design:**

```typescript
// frontend/src/app/(dashboard)/frameworks/page.tsx
// Card grid showing active frameworks with:
//   - Framework name, version, provider
//   - Coverage gauge (percentage of requirements mapped to controls)
//   - Requirement count
//   - Last assessment date

// frontend/src/app/(dashboard)/frameworks/[id]/page.tsx
// Tabs:
//   - Requirements: expandable tree of requirements with status badges
//   - Coverage: gap analysis view showing unmapped requirements
//   - Controls: controls mapped to this framework
//   - Crosswalks: cross-framework mappings to other active frameworks

// frontend/src/components/charts/coverage-gauge.tsx
// Circular gauge showing compliance coverage percentage
// Colour: green (>80%), yellow (60-80%), red (<60%)

// frontend/src/components/tables/requirement-tree.tsx
// Expandable tree table showing requirements with:
//   - Ref ID, Title
//   - Mapped controls (count + expand to see list)
//   - Coverage status badge (mapped/partial/gap)
//   - Action: map control to requirement
```

**Testing:**
- E2E test: navigate to /frameworks → framework cards load with coverage gauges
- E2E test: click a framework → requirement tree loads with expandable hierarchy
- E2E test: map a control to a requirement → coverage percentage updates
- E2E test: cross-framework overlap view shows controls satisfying multiple frameworks
- Visual test: coverage gauge renders correctly at 0%, 50%, and 100%

### Definition of Done — Phase 3
- [ ] Framework and requirement models with hierarchical tree structure
- [ ] Control-requirement mapping with coverage statistics API
- [ ] Cross-framework crosswalk model and API
- [ ] JSON framework importer with management command
- [ ] 5 built-in framework packs: ISO 27001, SOC 2, NIST CSF 2.0, GDPR, EU AI Act
- [ ] OSCAL Catalog import support
- [ ] Frontend: framework list with coverage gauges
- [ ] Frontend: requirement tree browser with control mapping
- [ ] Frontend: cross-framework overlap view
- [ ] 90%+ test coverage on framework models and import pipeline

---

## Phase 4: Policy & Audit Management

**Goal:** Implement the policy lifecycle (draft, review, approve, publish, attest, retire) and internal audit management (plan, engage, find, remediate) aligned to IIA Global Internal Audit Standards 2024.

**Duration:** 3-4 weeks

### Task 4.1: Policy Lifecycle Management

**What:** Create the `policy` model with version control, approval workflow, publication, and JSONB metadata for version history and linked controls. Build the policy attestation system for tracking user acknowledgement.

**Design:**

```python
# backend/policies/models.py
class Policy(TenantModel):
    STATUS_CHOICES = [
        ("draft", "Draft"), ("pending_review", "Pending Review"),
        ("pending_approval", "Pending Approval"), ("approved", "Approved"),
        ("published", "Published"), ("retired", "Retired"),
    ]
    TYPE_CHOICES = [
        ("policy", "Policy"), ("standard", "Standard"),
        ("procedure", "Procedure"), ("guideline", "Guideline"),
    ]

    ref_id = models.CharField(max_length=50)
    title = models.CharField(max_length=500)
    content = models.TextField(blank=True)  # Markdown content
    version = models.IntegerField(default=1)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default="draft")
    policy_type = models.CharField(max_length=30, choices=TYPE_CHOICES, default="policy")
    owner = models.ForeignKey("organizations.AppUser", null=True, on_delete=models.SET_NULL, related_name="owned_policies")
    metadata = models.JSONField(default=dict)
    # metadata: {approver_id, approved_at, effective_date, review_due_date, review_frequency,
    #            version_history: [{version, created_at, change_summary}], linked_control_ids}

class PolicyAttestation(BaseModel):
    policy = models.ForeignKey(Policy, on_delete=models.CASCADE, related_name="attestations")
    user = models.ForeignKey("organizations.AppUser", on_delete=models.CASCADE)
    policy_version = models.IntegerField()
    status = models.CharField(max_length=20, default="pending")  # pending, acknowledged, declined, expired

    class Meta:
        unique_together = [("policy", "user", "policy_version")]
```

```python
# backend/policies/services.py
def submit_for_approval(policy, approver):
    """Transition policy from draft to pending_approval."""
    policy.status = "pending_approval"
    policy.metadata["approver_id"] = str(approver.id)
    policy.save()

def approve_policy(policy, approver):
    """Approve and optionally publish a policy."""
    policy.status = "approved"
    policy.metadata["approved_at"] = timezone.now().isoformat()
    policy.save()

def publish_policy(policy):
    """Publish policy and create attestation requests for target audience."""
    policy.status = "published"
    policy.metadata["published_at"] = timezone.now().isoformat()
    policy.save()
    _create_attestation_requests(policy)

def _create_attestation_requests(policy):
    """Create pending attestation records for all users in target audience."""
    users = AppUser.objects.filter(organization=policy.organization, status="active")
    attestations = [
        PolicyAttestation(policy=policy, user=user, policy_version=policy.version, status="pending")
        for user in users
    ]
    PolicyAttestation.objects.bulk_create(attestations, ignore_conflicts=True)
```

**Testing:**
- Unit test: policy status transitions follow valid workflow (draft → pending_review → pending_approval → approved → published)
- Unit test: invalid transition (e.g., draft → published) raises ValidationError
- Unit test: approving a policy records approver and timestamp in metadata
- Unit test: publishing a policy creates attestation records for all active users
- Unit test: user attestation updates status from "pending" to "acknowledged"
- Unit test: attestation summary counts (total, acknowledged, pending) are correct
- Unit test: policy version increments on content update
- Unit test: version history in metadata tracks previous versions

### Task 4.2: Audit Engagement Management

**What:** Create the `audit_engagement` and `audit_finding` models aligned to IIA Standards 2024, with engagement lifecycle (planned, fieldwork, reporting, review, issued, closed) and finding structure (condition, criteria, cause, effect).

**Design:**

```python
# backend/audits/models.py
class AuditEngagement(TenantModel):
    STATUS_CHOICES = [
        ("planned", "Planned"), ("fieldwork", "Fieldwork"),
        ("reporting", "Reporting"), ("review", "Review"),
        ("issued", "Issued"), ("closed", "Closed"),
    ]

    ref_id = models.CharField(max_length=50)
    title = models.CharField(max_length=500)
    engagement_type = models.CharField(max_length=30, default="assurance")
    status = models.CharField(max_length=30, choices=STATUS_CHOICES, default="planned")
    lead_auditor = models.ForeignKey("organizations.AppUser", null=True, on_delete=models.SET_NULL)
    business_unit = models.ForeignKey("organizations.BusinessUnit", null=True, on_delete=models.SET_NULL)
    metadata = models.JSONField(default=dict)
    # metadata: {audit_plan_year, planned_start, planned_end, actual_start, actual_end,
    #            scope, objectives, team_members, budget_hours, actual_hours}

class AuditFinding(BaseModel):
    engagement = models.ForeignKey(AuditEngagement, on_delete=models.CASCADE, related_name="findings")
    ref_id = models.CharField(max_length=50)
    title = models.CharField(max_length=500)
    severity = models.CharField(max_length=20, default="medium")
    status = models.CharField(max_length=30, default="draft")
    finding_owner = models.ForeignKey("organizations.AppUser", null=True, on_delete=models.SET_NULL)
    due_date = models.DateField(null=True, blank=True)
    details = models.JSONField(default=dict)
    # details (IIA-aligned): {condition, criteria, cause, effect, recommendation,
    #   management_response, corrective_actions: [{description, assigned_to, due_date, status}],
    #   linked_risks, linked_controls}
```

**Testing:**
- Unit test: engagement lifecycle transitions are valid (planned → fieldwork → reporting → review → issued → closed)
- Unit test: creating a finding under an engagement associates correctly
- Unit test: finding details JSONB stores IIA condition/criteria/cause/effect structure
- Unit test: corrective actions embedded in finding details are queryable via JSONB operators
- Unit test: engagement finding count is accurate
- Unit test: filtering findings by severity and status works correctly
- Unit test: overdue findings (due_date < today, status not closed) are queryable

### Task 4.3: Policy & Audit Frontend

**What:** Build the policy management UI (list, detail, approval workflow, attestation tracking) and audit management UI (engagement list, engagement detail with findings, finding detail with corrective actions).

**Testing:**
- E2E test: create a policy → submit for review → approve → publish → attestation requests appear
- E2E test: user acknowledges a policy → attestation status updates
- E2E test: create audit engagement → add findings → track corrective actions
- E2E test: finding detail shows IIA condition/criteria/cause/effect fields
- E2E test: overdue findings are highlighted in red

### Definition of Done — Phase 4
- [ ] Policy CRUD with lifecycle workflow (draft through retired)
- [ ] Policy attestation system with user acknowledgement tracking
- [ ] Audit engagement CRUD with IIA-aligned lifecycle
- [ ] Audit finding CRUD with condition/criteria/cause/effect structure
- [ ] Corrective action tracking within findings
- [ ] Frontend: policy list, detail, approval workflow, attestation dashboard
- [ ] Frontend: audit engagement list, detail with findings, finding detail
- [ ] Audit log captures all policy and audit state changes
- [ ] 90%+ test coverage on policy and audit models

---

## Phase 5: Assessments & Evidence Management

**Goal:** Build the compliance assessment engine for evaluating requirements against frameworks and the evidence repository for storing, linking, and managing compliance evidence.

**Duration:** 2-3 weeks

### Task 5.1: Compliance Assessment Engine

**What:** Implement the `compliance_assessment` model for point-in-time assessments of an organisation against a framework. Each assessment contains per-requirement results (compliant, partially_compliant, non_compliant, not_applicable, not_assessed) stored as JSONB.

**Design:**

```python
# backend/frameworks/models.py (addition)
class ComplianceAssessment(TenantModel):
    framework = models.ForeignKey(Framework, on_delete=models.CASCADE, related_name="assessments")
    name = models.CharField(max_length=255)
    assessment_date = models.DateField()
    assessor = models.ForeignKey("organizations.AppUser", null=True, on_delete=models.SET_NULL)
    status = models.CharField(max_length=20, default="in_progress")
    overall_score = models.DecimalField(max_digits=5, decimal_places=2, null=True)
    results = models.JSONField(default=list)
    # results: [{requirement_id, requirement_ref, status, evidence_notes, assessed_by, assessed_at}]
    notes = models.TextField(blank=True)
```

**Testing:**
- Unit test: creating an assessment pre-populates results array with all framework requirements as "not_assessed"
- Unit test: updating a requirement result recalculates overall_score
- Unit test: overall_score = (compliant + partial * 0.5) / (total - not_applicable) * 100
- Unit test: assessment status transitions (in_progress → under_review → completed → archived)
- Unit test: historical assessments are queryable (show compliance trend over time)

### Task 5.2: Evidence Repository

**What:** Create the `evidence` model for storing compliance evidence documents with SHA-256 integrity hashes, collection metadata, and polymorphic linking to controls, assessments, findings, and vendors.

**Design:**

```python
# backend/evidence/models.py
class Evidence(TenantModel):
    name = models.CharField(max_length=500)
    evidence_type = models.CharField(max_length=30, default="document")
    file_path = models.TextField(blank=True)  # S3/MinIO object key
    file_size = models.BigIntegerField(null=True)
    mime_type = models.CharField(max_length=255, blank=True)
    hash_sha256 = models.CharField(max_length=64, blank=True)
    collection_method = models.CharField(max_length=20, default="manual")
    metadata = models.JSONField(default=dict)
    # metadata: {collected_by, source_system, valid_from, valid_to, linked_entities}

# File upload via presigned URL (S3-compatible):
# POST /api/v1/evidence/upload-url/ → returns presigned PUT URL
# Client uploads file directly to S3/MinIO
# POST /api/v1/evidence/ with metadata creates the Evidence record
```

**Testing:**
- Unit test: creating evidence with file upload generates SHA-256 hash
- Unit test: evidence metadata stores linked entity references (control_test, assessment, finding)
- Unit test: querying evidence by linked entity returns correct evidence items
- Unit test: evidence file integrity is verifiable by recomputing SHA-256
- Unit test: evidence valid_from/valid_to filters return only current evidence

### Task 5.3: Assessment & Evidence Frontend

**What:** Build the assessment workflow UI (start assessment, review requirements, mark compliance status, attach evidence) and evidence repository UI (upload, browse, link to entities).

**Testing:**
- E2E test: start an assessment → mark requirements as compliant/non-compliant → overall score updates
- E2E test: upload evidence document → link to a control test → evidence appears in control detail
- E2E test: evidence list is searchable by name and filterable by type

### Definition of Done — Phase 5
- [ ] Compliance assessment CRUD with per-requirement results
- [ ] Overall compliance score calculation
- [ ] Evidence repository with file upload (S3-compatible)
- [ ] SHA-256 integrity verification for evidence files
- [ ] Evidence linking to controls, assessments, findings
- [ ] Frontend: assessment workflow, evidence upload, evidence browser
- [ ] Assessment history shows compliance trend over time

---

## Phase 6: Third-Party Risk & Incident Management

**Goal:** Implement vendor/third-party risk management with assessment workflows and the incident management module with lifecycle tracking and regulatory reporting support.

**Duration:** 2-3 weeks

### Task 6.1: Vendor Management

**What:** Create the `vendor` model with criticality tiers, risk scoring, assessment history in JSONB metadata, and linking to the risk register. Build questionnaire-based assessment workflows.

**Design:**

```python
# backend/vendors/models.py
class Vendor(TenantModel):
    name = models.CharField(max_length=255)
    criticality = models.CharField(max_length=20, default="medium")
    status = models.CharField(max_length=20, default="active")
    owner = models.ForeignKey("organizations.AppUser", null=True, on_delete=models.SET_NULL)
    risk_score = models.DecimalField(max_digits=5, decimal_places=2, null=True)
    metadata = models.JSONField(default=dict)
    # metadata: {vendor_type, jurisdiction, website, contract_start, contract_end,
    #   certifications, dora_classification, assessment_history: [{date, result, assessor}]}
```

**Testing:**
- Unit test: creating a vendor with valid criticality succeeds
- Unit test: vendor risk score is recalculated on assessment completion
- Unit test: vendor-to-risk linking creates association in risk register
- Unit test: assessment history in metadata is append-only (preserves history)
- Unit test: DORA classification metadata is queryable (`metadata @> '{"dora_classification": "critical_ict_provider"}'`)

### Task 6.2: Incident Management

**What:** Create the `incident` model with lifecycle tracking (reported, investigating, contained, remediated, closed, post_mortem), regulatory reporting flags, financial impact, and linking to risks and controls.

**Design:**

```python
# backend/incidents/models.py
class Incident(TenantModel):
    ref_id = models.CharField(max_length=50)
    title = models.CharField(max_length=500)
    incident_type = models.CharField(max_length=30)
    severity = models.CharField(max_length=20, default="medium")
    status = models.CharField(max_length=30, default="reported")
    reported_by = models.ForeignKey("organizations.AppUser", null=True, on_delete=models.SET_NULL, related_name="+")
    assigned_to = models.ForeignKey("organizations.AppUser", null=True, on_delete=models.SET_NULL, related_name="+")
    metadata = models.JSONField(default=dict)
    # metadata: {description, reported_at, root_cause, remediation, financial_impact,
    #   records_affected, regulatory_reportable, regulatory_reported_at,
    #   gdpr_article_33_deadline, timeline: [{event, timestamp, by}],
    #   linked_risks, linked_controls}
```

**Testing:**
- Unit test: incident lifecycle transitions are valid
- Unit test: incident metadata stores timeline events in chronological order
- Unit test: regulatory_reportable flag triggers deadline calculation (e.g., GDPR 72-hour)
- Unit test: incident-to-risk linking works correctly
- Unit test: incident-to-control linking identifies control failures
- E2E test: report incident → investigate → contain → remediate → close with post-mortem

### Definition of Done — Phase 6
- [ ] Vendor CRUD with criticality, risk scoring, assessment tracking
- [ ] Vendor-to-risk linking
- [ ] Incident CRUD with lifecycle, regulatory reporting, timeline tracking
- [ ] Incident-to-risk and incident-to-control linking
- [ ] Frontend: vendor list, detail, assessment workflow
- [ ] Frontend: incident list, detail, timeline view
- [ ] Audit log captures all vendor and incident state changes

---

## Phase 7: Automated Evidence Collection

**Goal:** Build the integration framework for automatically collecting compliance evidence from external systems (AWS, GitHub, Okta, Jira) via API connectors, replacing manual evidence gathering.

**Duration:** 3-4 weeks

### Task 7.1: Integration Connector Framework

**What:** Design the pluggable connector architecture with a base connector class, credential management (encrypted storage), scheduling via Celery Beat, and result normalisation into control test records.

**Design:**

```python
# backend/integrations/connectors/base.py
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class EvidenceResult:
    source_system: str
    check_name: str
    result: str  # pass, fail, partial
    details: dict
    raw_response: dict
    collected_at: datetime

class BaseConnector(ABC):
    name: str
    description: str

    @abstractmethod
    def authenticate(self, credentials: dict) -> bool:
        """Verify credentials are valid."""

    @abstractmethod
    def collect_evidence(self, checks: list[str]) -> list[EvidenceResult]:
        """Run specified checks and return results."""

    @abstractmethod
    def available_checks(self) -> list[dict]:
        """List available checks this connector can perform."""

# backend/integrations/connectors/aws_connector.py
class AWSConnector(BaseConnector):
    name = "aws"
    description = "Amazon Web Services evidence collection"

    def collect_evidence(self, checks):
        results = []
        if "s3_encryption" in checks:
            # Query AWS Config for S3 bucket encryption compliance
            response = self.config_client.get_compliance_details_by_config_rule(
                ConfigRuleName="s3-bucket-server-side-encryption-enabled"
            )
            for eval_result in response["EvaluationResults"]:
                results.append(EvidenceResult(
                    source_system="aws_config",
                    check_name="s3_encryption",
                    result="pass" if eval_result["ComplianceType"] == "COMPLIANT" else "fail",
                    details={"resource": eval_result["EvaluationResultIdentifier"]["EvaluationResultQualifier"]},
                    raw_response=eval_result,
                    collected_at=datetime.now(),
                ))
        return results

# backend/integrations/tasks.py (Celery tasks)
@shared_task
def run_evidence_collection(connector_name, organization_id, checks):
    """Background task to collect evidence from an external system."""
    connector = get_connector(connector_name)
    results = connector.collect_evidence(checks)
    for result in results:
        ControlTest.objects.create(
            control_id=mapping.control_id,
            result=result.result,
            automated=True,
            details={
                "methodology": "automated_collection",
                "source_system": result.source_system,
                "check_name": result.check_name,
                **result.details,
            },
        )
```

**Testing:**
- Unit test: BaseConnector subclass must implement authenticate, collect_evidence, available_checks
- Unit test: EvidenceResult dataclass serialises correctly
- Unit test: connector registry returns all registered connectors
- Unit test: credential storage encrypts sensitive fields at rest
- Unit test: Celery task creates ControlTest records from connector results
- Integration test (mocked): AWS connector processes Config rule compliance results correctly
- Integration test (mocked): GitHub connector checks branch protection settings

### Task 7.2: Built-in Connectors (AWS, GitHub, Okta, Jira)

**What:** Implement connectors for the four most common evidence sources: AWS (Config rules, CloudTrail), GitHub (branch protection, dependabot, code scanning), Okta (MFA enforcement, access reviews), Jira (issue tracking for remediation).

**Testing:**
- Unit test (mocked): each connector returns valid EvidenceResult objects
- Unit test (mocked): AWS connector handles pagination for large Config rule results
- Unit test (mocked): GitHub connector checks branch protection, secret scanning, dependabot status
- Unit test (mocked): Okta connector verifies MFA enforcement policy
- Unit test (mocked): Jira connector tracks remediation issue status

### Task 7.3: Evidence Collection Scheduling & UI

**What:** Build the scheduling UI for configuring evidence collection frequency, mapping connectors to controls, and viewing collection history and results.

**Testing:**
- E2E test: configure AWS connector with credentials → schedule daily collection → results appear in control test history
- E2E test: evidence collection failure is logged and surfaced in UI
- E2E test: connector health dashboard shows last run time, status, and next scheduled run

### Definition of Done — Phase 7
- [ ] Pluggable connector framework with base class and registry
- [ ] Encrypted credential storage for connector authentication
- [ ] 4 built-in connectors: AWS, GitHub, Okta, Jira (with mocked integration tests)
- [ ] Celery Beat scheduling for periodic evidence collection
- [ ] Automated control test creation from connector results
- [ ] Frontend: connector configuration, scheduling, collection history
- [ ] Evidence collection failures logged and surfaced in UI

---

## Phase 8: AI-Powered Features

**Goal:** Implement the three core AI differentiators: regulatory change monitoring with control impact mapping, risk narrative and scoring assistance, and policy-to-control gap analysis. This is the primary competitive differentiation versus existing OSS GRC tools.

**Duration:** 4-5 weeks

### Task 8.1: LLM Provider Abstraction Layer

**What:** Build a provider-agnostic AI service layer that abstracts Anthropic Claude, OpenAI, and local model providers. Implement prompt management, response parsing, cost tracking, and caching.

**Design:**

```python
# backend/ai/providers/base.py
from abc import ABC, abstractmethod

class LLMProvider(ABC):
    @abstractmethod
    async def complete(self, system_prompt: str, user_prompt: str,
                       temperature: float = 0.3, max_tokens: int = 4096) -> str:
        pass

    @abstractmethod
    async def complete_structured(self, system_prompt: str, user_prompt: str,
                                   output_schema: dict) -> dict:
        """Return structured output conforming to the provided JSON schema."""
        pass

# backend/ai/providers/anthropic_provider.py
import anthropic

class AnthropicProvider(LLMProvider):
    def __init__(self):
        self.client = anthropic.Anthropic()
        self.model = "claude-sonnet-4-20250514"

    async def complete_structured(self, system_prompt, user_prompt, output_schema):
        response = self.client.messages.create(
            model=self.model,
            max_tokens=4096,
            system=[
                {"type": "text", "text": system_prompt, "cache_control": {"type": "ephemeral"}}
            ],
            messages=[{"role": "user", "content": user_prompt}],
        )
        return json.loads(response.content[0].text)

# backend/ai/services/base.py
class AIService:
    def __init__(self, provider: LLMProvider | None = None):
        self.provider = provider or get_default_provider()
```

**Testing:**
- Unit test: provider abstraction correctly routes to Anthropic/OpenAI/local
- Unit test: structured output parsing handles valid and malformed responses
- Unit test: prompt caching reduces API calls for repeated framework analysis
- Unit test: cost tracking records token usage per request
- Unit test: provider fallback works when primary provider returns error

### Task 8.2: AI-Powered Regulatory Change Monitoring

**What:** Build the regulatory change monitoring pipeline: ingest regulatory sources (RSS/API), classify applicability using LLM, assess impact on existing controls and frameworks, and create actionable regulatory_change records.

**Design:**

```python
# backend/ai/services/regulatory_monitor.py
class RegulatoryMonitorService(AIService):
    SYSTEM_PROMPT = """You are a regulatory compliance analyst for a {industry} company
    operating in {jurisdictions}. Analyze the following regulatory publication and determine:
    1. Is it applicable to this organization? (yes/no/maybe)
    2. What is the change type? (new_regulation/amendment/guidance/enforcement/sunset)
    3. What compliance frameworks are affected?
    4. What existing controls may need updating?
    5. What is the effective/enforcement date?

    Respond in JSON format conforming to the provided schema."""

    async def analyze_regulatory_change(self, text: str, org_context: dict) -> dict:
        result = await self.provider.complete_structured(
            system_prompt=self.SYSTEM_PROMPT.format(**org_context),
            user_prompt=f"Analyze this regulatory publication:\n\n{text}",
            output_schema=REGULATORY_CHANGE_SCHEMA,
        )
        return result

    async def map_impact_to_controls(self, change: dict, controls: list[dict]) -> list[dict]:
        """Given a regulatory change, identify which controls are affected."""
        prompt = f"""Given this regulatory change:
        {json.dumps(change)}

        And these existing controls:
        {json.dumps(controls)}

        Identify which controls are affected and describe the required changes."""
        return await self.provider.complete_structured(
            system_prompt="You are a compliance mapping expert.",
            user_prompt=prompt,
            output_schema=IMPACT_MAPPING_SCHEMA,
        )

# backend/regulatory/tasks.py
@shared_task
def scan_regulatory_sources(organization_id):
    """Periodic Celery task to scan regulatory sources for changes."""
    org = Organization.objects.get(id=organization_id)
    sources = get_regulatory_sources(org.industry, org.jurisdiction)
    for source in sources:
        new_publications = fetch_new_publications(source)
        for pub in new_publications:
            result = RegulatoryMonitorService().analyze_regulatory_change(
                text=pub.text,
                org_context={"industry": org.industry, "jurisdictions": org.jurisdiction},
            )
            if result["applicable"] in ("yes", "maybe"):
                RegulatoryChange.objects.create(
                    organization=org,
                    title=result["title"],
                    status="identified",
                    jurisdiction=result.get("jurisdiction"),
                    effective_date=result.get("effective_date"),
                    ai_confidence=result["confidence"],
                    metadata=result,
                )
```

**Testing:**
- Unit test (mocked LLM): regulatory text classified as applicable returns structured change record
- Unit test (mocked LLM): non-applicable regulation correctly filtered out
- Unit test (mocked LLM): impact mapping identifies affected controls with change descriptions
- Unit test: ai_confidence score is stored and queryable
- Unit test: regulatory change status workflow (identified → under_review → impact_assessed → implemented)
- Unit test: Celery task processes multiple sources and creates RegulatoryChange records
- Accuracy test: run 20 known regulatory changes through the pipeline, verify >= 90% correct classification (PwC benchmark)

### Task 8.3: AI-Assisted Risk Narrative & Scoring

**What:** Build LLM-powered features for risk register management: auto-generate risk descriptions from structured data, suggest likelihood/impact scores based on industry benchmarks, and draft treatment plans.

**Design:**

```python
# backend/ai/services/risk_assistant.py
class RiskAssistantService(AIService):
    async def suggest_risk_description(self, title: str, category: str,
                                        business_context: str) -> dict:
        """Generate a structured risk description with source, consequence, and treatment suggestions."""
        prompt = f"""Draft a risk description for:
        Title: {title}
        Category: {category}
        Business Context: {business_context}

        Provide: description, risk_source (ISO 31000), potential consequences,
        suggested_likelihood (1-5 with rationale), suggested_impact (1-5 with rationale),
        recommended_treatment_type, and treatment_plan_outline."""
        return await self.provider.complete_structured(
            system_prompt="You are a GRC risk analyst. Use ISO 31000 terminology.",
            user_prompt=prompt,
            output_schema=RISK_SUGGESTION_SCHEMA,
        )

    async def analyze_policy_control_gaps(self, policies: list[dict],
                                           controls: list[dict]) -> list[dict]:
        """Identify gaps, duplicates, and conflicts between policy library and control library."""
        prompt = f"""Analyze these policies and controls for alignment:
        Policies: {json.dumps(policies)}
        Controls: {json.dumps(controls)}

        Identify: 1) Policies referencing non-existent controls
        2) Controls with no governing policy
        3) Duplicate/overlapping policies or controls
        4) Conflicts between policy requirements and control implementations"""
        return await self.provider.complete_structured(
            system_prompt="You are a GRC policy-control alignment analyst.",
            user_prompt=prompt,
            output_schema=GAP_ANALYSIS_SCHEMA,
        )
```

**Testing:**
- Unit test (mocked LLM): risk description generation returns ISO 31000-aligned output
- Unit test (mocked LLM): scoring suggestions include rationale text
- Unit test (mocked LLM): policy-to-control gap analysis identifies known gaps in test data
- Unit test: AI suggestions are clearly marked as AI-generated (not auto-applied)
- E2E test: user creates a risk → clicks "AI Assist" → suggestions populate form fields (user reviews before saving)

### Task 8.4: Regulatory Change & AI Frontend

**What:** Build the regulatory change tracking UI with AI classification confidence indicators and the AI assistance UI for risk and policy workflows.

**Testing:**
- E2E test: regulatory changes list shows AI confidence badge (green >0.9, yellow >0.7, red <0.7)
- E2E test: clicking a regulatory change shows impact analysis with affected controls
- E2E test: risk creation form has "AI Assist" button that populates suggestions
- E2E test: AI suggestions show clear "AI-generated" label and require user confirmation

### Definition of Done — Phase 8
- [ ] LLM provider abstraction layer with Anthropic and OpenAI implementations
- [ ] Regulatory change monitoring pipeline with source scanning and LLM classification
- [ ] Impact mapping from regulatory changes to controls/frameworks
- [ ] AI-assisted risk description and scoring suggestions
- [ ] Policy-to-control gap analysis
- [ ] All AI outputs clearly marked as AI-generated
- [ ] Prompt caching for repeated framework analysis
- [ ] Frontend: regulatory change tracker with AI confidence indicators
- [ ] Frontend: AI assist buttons in risk and policy workflows
- [ ] >= 90% regulatory classification accuracy on test dataset (20+ samples)

---

## Phase 9: Continuous Control Monitoring

**Goal:** Evolve the evidence collection framework from periodic scheduled collection to continuous automated control testing. Controls are tested automatically at defined intervals, results are evaluated, and failing controls trigger alerts — shifting GRC from periodic attestation to continuous assurance.

**Duration:** 3-4 weeks

### Task 9.1: Continuous Monitoring Engine

**What:** Build the continuous monitoring scheduler that runs control tests at their defined frequency (continuous, daily, weekly, monthly), evaluates results against pass/fail thresholds, and triggers notifications for failing controls.

**Design:**

```python
# backend/integrations/monitoring.py
class ContinuousMonitor:
    def evaluate_control(self, control, test_results):
        """Evaluate control health based on recent test results."""
        if not test_results:
            return ControlHealth.UNKNOWN

        recent = test_results[:control.monitoring_config.get("window_size", 5)]
        fail_count = sum(1 for r in recent if r.result == "fail")
        threshold = control.monitoring_config.get("fail_threshold", 0.2)

        if fail_count / len(recent) > threshold:
            self.trigger_alert(control, fail_count, len(recent))
            return ControlHealth.FAILING
        elif fail_count > 0:
            return ControlHealth.DEGRADED
        return ControlHealth.HEALTHY

    def trigger_alert(self, control, fail_count, total):
        """Notify control owner and compliance manager of failing control."""
        Notification.objects.create(
            organization=control.organization,
            recipient=control.owner,
            notification_type="control_failing",
            entity_type="control",
            entity_id=control.id,
            message=f"Control {control.ref_id} failing: {fail_count}/{total} recent tests failed",
        )
```

**Testing:**
- Unit test: control with 0/5 failures returns HEALTHY
- Unit test: control with 1/5 failures returns DEGRADED
- Unit test: control with 3/5 failures (>20% threshold) returns FAILING and triggers alert
- Unit test: configurable fail_threshold per control overrides default
- Unit test: continuous monitoring respects control test frequency scheduling
- Integration test: end-to-end flow from connector collection → test creation → evaluation → alert

### Task 9.2: Monitoring Dashboard

**What:** Build the real-time control health dashboard showing control status across the organisation, with drill-down to individual test results and trend visualisation.

**Testing:**
- E2E test: dashboard shows red/yellow/green indicators for each monitored control
- E2E test: clicking a failing control shows recent test history and failure details
- E2E test: trend chart shows control health over time (last 30/90/365 days)

### Definition of Done — Phase 9
- [ ] Continuous monitoring engine with configurable thresholds
- [ ] Alert system for failing controls (email + in-app notification)
- [ ] Control health status (healthy, degraded, failing, unknown)
- [ ] Frontend: real-time control health dashboard
- [ ] Frontend: control health trend visualisation
- [ ] Monitoring configuration per control (frequency, window, threshold)

---

## Phase 10: Graph Analytics & Advanced Reporting

**Goal:** Add the graph layer from Data Model Suggestion 4 for relationship-heavy analytics: blast-radius analysis, cross-framework overlap detection, vendor dependency mapping, conflict-of-interest detection, and board-level reporting.

**Duration:** 3-4 weeks

### Task 10.1: Graph Layer Implementation

**What:** Create the `graph_node` and `graph_edge` tables as a derived graph index of relational data. Implement triggers to sync relational changes to the graph. Build graph query endpoints for traversal.

**Design:**

```python
# backend/core/graph.py
class GraphNode(models.Model):
    id = models.UUIDField(primary_key=True)  # same as relational entity ID
    organization = models.ForeignKey("organizations.Organization", on_delete=models.CASCADE)
    node_type = models.CharField(max_length=50)
    ref_id = models.CharField(max_length=100, blank=True)
    label = models.CharField(max_length=500)
    properties = models.JSONField(default=dict)

class GraphEdge(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    organization = models.ForeignKey("organizations.Organization", on_delete=models.CASCADE)
    source = models.ForeignKey(GraphNode, on_delete=models.CASCADE, related_name="outgoing_edges")
    target = models.ForeignKey(GraphNode, on_delete=models.CASCADE, related_name="incoming_edges")
    edge_type = models.CharField(max_length=50)
    properties = models.JSONField(default=dict)
    weight = models.DecimalField(max_digits=5, decimal_places=2, default=1.0)

    class Meta:
        unique_together = [("source", "target", "edge_type")]
```

```python
# backend/core/graph_queries.py
def blast_radius(node_id, max_depth=3):
    """Traverse the graph from a node to find all connected entities."""
    query = """
    WITH RECURSIVE blast AS (
        SELECT id, node_type, ref_id, label, 0 AS depth, ARRAY[id] AS path
        FROM core_graphnode WHERE id = %s
        UNION ALL
        SELECT gn.id, gn.node_type, gn.ref_id, gn.label, b.depth + 1, b.path || gn.id
        FROM blast b
        JOIN core_graphedge ge ON ge.source_id = b.id
        JOIN core_graphnode gn ON gn.id = ge.target_id
        WHERE b.depth < %s AND NOT gn.id = ANY(b.path)
    )
    SELECT DISTINCT node_type, ref_id, label, depth FROM blast WHERE id != %s ORDER BY depth;
    """
    with connection.cursor() as cursor:
        cursor.execute(query, [node_id, max_depth, node_id])
        return cursor.fetchall()
```

**Testing:**
- Unit test: creating a risk triggers graph_node creation via signal
- Unit test: linking a control to a risk creates a MITIGATES graph_edge
- Unit test: blast_radius from a control returns linked risks, requirements, and frameworks
- Unit test: cross-framework overlap query finds controls satisfying 2+ frameworks
- Unit test: conflict-of-interest query finds users owning both a risk and its mitigating control
- Unit test: orphaned control detection finds controls with no risk or requirement links
- Performance test: graph traversal with 10,000 nodes completes in < 500ms

### Task 10.2: Board-Level Reporting

**What:** Build reporting endpoints and dashboard for executive/board consumption: risk heat map, compliance posture summary, top risks, overdue findings, vendor risk summary.

**Testing:**
- E2E test: executive dashboard shows risk heat map, compliance gauges, and key metrics
- E2E test: report export to PDF generates formatted board report
- E2E test: drill-down from board summary to detailed risk/control views

### Definition of Done — Phase 10
- [ ] Graph layer (graph_node, graph_edge) with trigger-based sync
- [ ] Blast-radius analysis API
- [ ] Cross-framework overlap detection
- [ ] Conflict-of-interest detection
- [ ] Vendor dependency graph traversal
- [ ] Board-level reporting dashboard
- [ ] PDF/PowerPoint report export
- [ ] Materialised views for common analytics queries

---

## Phase 11: OSCAL Interoperability & MCP Server

**Goal:** Build native OSCAL import/export for framework interoperability with the US federal compliance ecosystem (FedRAMP, FISMA) and ship an MCP server that exposes GRC data to LLM agents.

**Duration:** 2-3 weeks

### Task 11.1: OSCAL Import/Export

**What:** Implement full OSCAL Catalog import (JSON/YAML), OSCAL Profile import for control baselines, and OSCAL Assessment Results export for compliance assessments.

**Design:**

```python
# backend/frameworks/importers/oscal_importer.py
def import_oscal_catalog(catalog_path: str) -> Framework:
    """Import NIST OSCAL Catalog JSON into Framework + FrameworkRequirement."""
    with open(catalog_path) as f:
        catalog = json.load(f)

    framework = Framework.objects.create(
        name=catalog["catalog"]["metadata"]["title"],
        version=catalog["catalog"]["metadata"]["version"],
        oscal_catalog_id=catalog["catalog"]["uuid"],
        framework_type="standard",
    )

    for group in catalog["catalog"].get("groups", []):
        _import_oscal_group(framework, group, parent=None, depth=0)
    return framework

# backend/frameworks/exporters/oscal_exporter.py
def export_assessment_results(assessment) -> dict:
    """Export a ComplianceAssessment as OSCAL Assessment Results JSON."""
    return {
        "assessment-results": {
            "uuid": str(assessment.id),
            "metadata": {
                "title": assessment.name,
                "last-modified": assessment.updated_at.isoformat(),
            },
            "results": [
                {
                    "uuid": str(uuid.uuid4()),
                    "title": assessment.name,
                    "start": assessment.assessment_date.isoformat(),
                    "findings": [
                        _requirement_to_oscal_finding(r)
                        for r in assessment.results
                    ],
                }
            ],
        }
    }
```

**Testing:**
- Unit test: OSCAL NIST 800-53 catalog imports all 1,196 controls with correct hierarchy
- Unit test: OSCAL profile import selects correct control baseline (FedRAMP Moderate)
- Unit test: OSCAL assessment results export produces valid JSON against OSCAL schema
- Unit test: round-trip: import catalog → create assessment → export results → validate schema

### Task 11.2: MCP Server

**What:** Build a Model Context Protocol server that exposes GRC data (risks, controls, frameworks, assessments) to LLM agents, enabling Claude and other agents to query and interact with the GRC platform.

**Design:**

```python
# backend/mcp_server/server.py
from mcp import Server
from mcp.types import Tool, Resource

server = Server("grc-platform")

@server.list_tools()
async def list_tools():
    return [
        Tool(name="query_risks", description="Query risk register with filters",
             inputSchema=RISK_QUERY_SCHEMA),
        Tool(name="query_controls", description="Query control library",
             inputSchema=CONTROL_QUERY_SCHEMA),
        Tool(name="assess_compliance", description="Get compliance posture for a framework",
             inputSchema=COMPLIANCE_QUERY_SCHEMA),
        Tool(name="analyze_regulatory_change", description="Analyze regulatory change impact",
             inputSchema=REGULATORY_ANALYSIS_SCHEMA),
    ]

@server.call_tool()
async def call_tool(name, arguments):
    if name == "query_risks":
        risks = Risk.objects.filter(**arguments.get("filters", {}))
        return [risk_to_dict(r) for r in risks[:50]]
    # ... other tool handlers
```

**Testing:**
- Unit test: MCP server lists available tools correctly
- Unit test: query_risks tool returns filtered risk data
- Unit test: assess_compliance tool returns framework coverage statistics
- Integration test: Claude agent queries risk register via MCP server and receives valid response

### Definition of Done — Phase 11
- [ ] OSCAL Catalog JSON import for NIST frameworks
- [ ] OSCAL Profile import for control baselines
- [ ] OSCAL Assessment Results JSON export
- [ ] OSCAL schema validation on import and export
- [ ] MCP server with tools for risk, control, compliance, and regulatory queries
- [ ] MCP server authentication and tenant isolation

---

## Phase 12: SaaS Packaging & Enterprise Features

**Goal:** Package the platform for production SaaS deployment with SAML/OIDC SSO, multi-tenant billing, Kubernetes Helm chart, and enterprise features (audit export, data retention, API rate limiting).

**Duration:** 3-4 weeks

### Task 12.1: SAML/OIDC SSO Integration

**What:** Implement full SAML 2.0 and OIDC authentication flows using python-social-auth, enabling enterprise SSO with Okta, Azure AD, and Google Workspace.

**Testing:**
- Integration test: SAML 2.0 login flow with Okta IdP (test instance)
- Integration test: OIDC login flow with Google Workspace
- Unit test: SSO user auto-provisioning creates AppUser on first login
- Unit test: SSO user role mapping from IdP group claims

### Task 12.2: Kubernetes Helm Chart

**What:** Build a production Helm chart with PostgreSQL (operator or managed), Redis, the application, Celery workers, and optional MinIO.

**Testing:**
- Deployment test: `helm install grc ./helm` on a test cluster succeeds
- Health check test: all pods report ready within 120 seconds
- Upgrade test: `helm upgrade grc ./helm` performs zero-downtime rolling update

### Task 12.3: API Rate Limiting & Webhook Support

**What:** Implement per-tenant API rate limiting (10 requests/second default, configurable) and webhook support for external integrations (notify on risk score change, control failure, incident creation).

**Testing:**
- Unit test: API returns 429 when rate limit exceeded
- Unit test: webhook fires on risk score change with correct payload
- Unit test: webhook retry logic handles transient failures (3 retries with exponential backoff)

### Task 12.4: Data Retention & Audit Export

**What:** Implement configurable data retention policies for audit logs (default 7 years, configurable per tenant), bulk audit trail export (JSON, CSV), and partition management for the audit_log table.

**Testing:**
- Unit test: audit log partitions older than retention period are archived
- Unit test: audit trail export produces valid JSON/CSV for a date range
- Unit test: exported audit trail includes all required fields for regulatory examination

### Definition of Done — Phase 12
- [ ] SAML 2.0 SSO with Okta and Azure AD
- [ ] OIDC SSO with Google Workspace
- [ ] Production Kubernetes Helm chart
- [ ] API rate limiting per tenant
- [ ] Webhook support for state change notifications
- [ ] Audit log data retention management
- [ ] Bulk audit trail export (JSON/CSV)
- [ ] Production deployment documentation
- [ ] Load testing: 1,000 concurrent users, < 200ms p95 latency on dashboard endpoints

---

## Summary

| Phase | Name | Duration | Dependencies |
|-------|------|----------|-------------|
| 1 | Foundation & Core Infrastructure | 3-4 weeks | None |
| 2 | Risk Register & Control Library | 3-4 weeks | Phase 1 |
| 3 | Compliance Framework Engine | 3-4 weeks | Phase 2 |
| 4 | Policy & Audit Management | 3-4 weeks | Phase 2 |
| 5 | Assessments & Evidence Management | 2-3 weeks | Phase 3 |
| 6 | Third-Party Risk & Incident Management | 2-3 weeks | Phase 4 |
| 7 | Automated Evidence Collection | 3-4 weeks | Phase 5 |
| 8 | AI-Powered Features | 4-5 weeks | Phase 2 |
| 9 | Continuous Control Monitoring | 3-4 weeks | Phase 7 |
| 10 | Graph Analytics & Advanced Reporting | 3-4 weeks | Phase 8 |
| 11 | OSCAL Interoperability & MCP Server | 2-3 weeks | Phase 3 |
| 12 | SaaS Packaging & Enterprise Features | 3-4 weeks | Phase 1 |
| **Total** | | **~36-48 weeks** | |

**MVP (Phases 1-5):** ~15-19 weeks. Delivers risk register, control library, compliance frameworks, policy management, audit management, assessments, and evidence repository. Sufficient for a usable open-source GRC platform competing with Eramba and SimpleRisk.

**AI Differentiation (Phase 8):** Can begin in parallel after Phase 2, delivering the core competitive advantages (regulatory change monitoring, risk scoring assistance, policy-to-control gap analysis) that differentiate from all existing OSS GRC tools.

**Full Platform (Phases 1-12):** ~36-48 weeks. Delivers a complete AI-native GRC platform with continuous monitoring, graph analytics, OSCAL interoperability, MCP server, and SaaS-ready packaging.
