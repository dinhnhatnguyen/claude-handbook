# CLAUDE.md Patterns — Template Collection

> Tập hợp templates CLAUDE.md cho 8 loại project phổ biến, kèm theo patterns cho global config, subdirectory config, và import files.
>
> Ghi chú 2026-04-30: đây là deep dive từ bản gốc. Dùng như thư viện ý tưởng, nhưng ưu tiên handbook mới và official docs khi có xung đột.

---

## Nguyên tắc chung

Trước khi xem templates, nhớ ba nguyên tắc:

1. **Ngắn gọn trên đầy đủ** — CLAUDE.md tốn token mỗi turn. 40 dòng focused tốt hơn 200 dòng comprehensive.
2. **WHY quan trọng hơn WHAT** — Claude đọc được code. Những gì Claude cần biết là context và constraints mà code không nói lên.
3. **Cụ thể, không vague** — "Dùng pydantic v2" tốt hơn "Follow best practices".

---

## 1. Global CLAUDE.md (áp dụng mọi project)

`~/.claude/CLAUDE.md`

```markdown
# Global Developer Preferences

## Identity
Full-stack developer, primary: Python và TypeScript.
Familiar with: FastAPI, Django, React, Next.js, PostgreSQL, Redis.

## Communication preferences
- Trả lời ngắn gọn, đúng trọng tâm
- Không giải thích những gì code đã nói rõ
- Nếu không chắc về yêu cầu, hỏi trước khi làm
- Không thêm feature, error handling, hoặc abstraction không được yêu cầu

## Code style (cross-project)
- Prefer functional style
- Type hints bắt buộc trong Python
- Không comment giải thích code hiển nhiên
- Test files: AAA pattern (Arrange, Act, Assert)

## Model preferences
- Architecture và security decisions: Opus
- Coding tasks: Sonnet (default)
- File search, simple edits: Haiku
```

---

## 2. Python Backend API (FastAPI)

```markdown
# Project: [Tên API]

## Context
REST API cho [mô tả ngắn]. Stack: FastAPI + PostgreSQL + Redis.
Deploy: Docker trên AWS ECS. ~[số] users, [QPS] req/s.

## Architecture decisions
- Async throughout: asyncpg + SQLAlchemy 2.0 async
- Caching: Redis với TTL 5 phút cho user data
- Auth: JWT (RS256), refresh token trong Redis
- Background tasks: Celery + SQS

## Code conventions
- Python 3.12+, type hints bắt buộc
- Pydantic v2 cho request/response schemas
- Custom exceptions thay vì generic HTTPException
- Dependency injection qua FastAPI Depends()
- Không dùng global state

## Error handling pattern
```python
# Đúng
raise UserNotFoundError(user_id=user_id)

# Sai
raise HTTPException(status_code=404, detail="Not found")
```

## Commands
- `make dev`      — uvicorn + postgres + redis via docker-compose
- `make test`     — pytest -xvs
- `make lint`     — ruff check + mypy
- `make migrate`  — alembic upgrade head

## Rules
- Không sửa files trong alembic/versions/ — tạo migration mới
- Không dùng sync SQLAlchemy trong async context
- Mọi endpoint cần có response_model
- Rate limiting đã cấu hình ở nginx, không implement thêm ở app
```

---

## 3. TypeScript/Node.js API (Express hoặc NestJS)

```markdown
# Project: [Tên API]

## Context
REST API. Stack: Node.js 20 + Express/NestJS + PostgreSQL (Prisma) + Redis.

## Architecture
- ORM: Prisma với generated types
- Auth: Passport.js + JWT
- Validation: class-validator + class-transformer
- Error handling: centralized error middleware

## Code conventions
- TypeScript strict mode: true
- Không dùng `any` — dùng `unknown` nếu cần
- Async/await, không callback
- Named exports, không default exports
- Interface cho DTOs, Type cho utility types

## Prisma patterns
```typescript
// Đúng: type-safe với Prisma types
const user = await prisma.user.findUniqueOrThrow({ where: { id } })

// Sai: bỏ qua null check
const user = await prisma.user.findUnique({ where: { id } })
user.email  // có thể null!
```

## Commands
- `npm run dev`       — nodemon với ts-node
- `npm test`          — jest với coverage
- `npm run lint`      — eslint + prettier check
- `npx prisma migrate dev` — create + apply migration

## Rules
- Không sửa files trong prisma/migrations/
- Không dùng `require()` — ES modules only
- Mọi controller method cần có Swagger decorator
```

---

## 4. React/Next.js Frontend

```markdown
# Project: [Tên App]

## Context
Web app. Stack: Next.js 14 (App Router) + TypeScript + Tailwind + shadcn/ui.
State: Zustand cho global, React Query cho server state.

## Architecture
- App Router (không phải Pages Router)
- Server Components mặc định, Client Components khi cần interactivity
- Data fetching: React Query với stale-while-revalidate
- Forms: React Hook Form + Zod validation

## Component conventions
- Mỗi component có file riêng: `ComponentName.tsx`
- Props interface đặt ngay trên component
- Tách UI component (dumb) và container (smart)
- Không inline style — Tailwind classes only

## State management
```typescript
// Server state: React Query
const { data, isLoading } = useQuery({ queryKey: ['users'], queryFn: fetchUsers })

// Global UI state: Zustand
const { isOpen, toggle } = useModalStore()

// Form state: React Hook Form
const { register, handleSubmit } = useForm<FormData>()
```

## Commands
- `npm run dev`    — Next.js dev server
- `npm test`       — jest + testing-library
- `npm run build`  — production build
- `npm run lint`   — eslint + tsc --noEmit

## Rules
- Không dùng `useEffect` cho data fetching — dùng React Query
- Không `any` type
- Images qua `next/image` component, không `<img>`
- API calls chỉ qua `src/lib/api/` directory
```

---

## 5. Python Data Science / ML

```markdown
# Project: [Tên ML Project]

## Context
[Mô tả problem]. Dataset: [kích thước, format]. Target metric: [metric].

## Environment
- Python 3.11 + conda environment: `conda activate ml-env`
- GPU: [có/không có, CUDA version]
- Framework: PyTorch / scikit-learn / transformers

## Project structure
- `data/raw/`        — không commit, không modify
- `data/processed/`  — generated by notebooks
- `notebooks/`       — exploration (numbered: 01_, 02_...)
- `src/`             — production code (importable)
- `models/`          — saved model artifacts

## Data conventions
- Không modify raw data — transform vào processed/
- Seed random: `SEED = 42` (defined in src/config.py)
- Train/val/test split: 70/15/15 (stratified)
- Normalize sau split, không trước (data leakage)

## Commands
- `make data`     — chạy data pipeline
- `make train`    — chạy training với default config
- `make evaluate` — evaluate trên test set
- `jupyter lab`   — mở Jupyter

## Rules
- Không commit model weights > 50MB — dùng DVC hoặc S3
- Experiments tracking qua MLflow (URL: [internal URL])
- Mọi experiment cần log: params, metrics, artifact path
- Không hardcode paths — dùng pathlib từ project root
```

---

## 6. Monorepo (nhiều packages)

```markdown
# Monorepo: [Tên Project]

## Structure
```
packages/
  api/        FastAPI backend
  web/        Next.js frontend  
  mobile/     React Native
  shared/     Shared types và utils
  infra/      Terraform
```

## Package-level CLAUDE.md
Mỗi package có CLAUDE.md riêng với specifics của package đó.
File này chứa cross-package conventions.

## Cross-package conventions
- Shared types: định nghĩa trong `packages/shared/types/`
- API contracts: OpenAPI spec trong `packages/api/openapi.yaml`
- Breaking changes cần update cả api/ và consumers

## Commands (từ root)
- `pnpm dev`          — start tất cả services
- `pnpm test`         — test tất cả packages
- `pnpm build`        — build tất cả
- `pnpm --filter api test`  — test chỉ api package

## Rules
- Không import trực tiếp giữa packages (chỉ qua shared/)
- Version bumping qua changesets: `pnpm changeset`
- Không commit lockfile conflicts — resolve bằng `pnpm install`

## Model assignment
- Cross-package architecture → Opus
- Single package tasks → Sonnet
- File search trong monorepo → Haiku (subagent)
```

---

## 7. DevOps / Infrastructure

```markdown
# Project: [Tên Infrastructure]

## Context
Infrastructure as Code cho [company/product].
Cloud: AWS (primary) + GCP (data).
IaC: Terraform + Ansible. Orchestration: Kubernetes (EKS).

## Environments
- `dev`     — playground, auto-deploy từ main
- `staging` — mirrors production, manual deploy
- `prod`    — production, requires approval

## Conventions
- Terraform modules trong `modules/`
- Không hardcode values — dùng variables và tfvars
- State backend: S3 + DynamoDB locking
- Secrets: AWS Secrets Manager, không trong terraform state

## Terraform patterns
```hcl
# Đúng: module với variables
module "rds" {
  source = "./modules/rds"
  instance_class = var.rds_instance_class
}

# Sai: hardcode
resource "aws_db_instance" "main" {
  instance_class = "db.t3.micro"  # hardcoded!
}
```

## Commands
- `make plan ENV=staging`   — terraform plan
- `make apply ENV=staging`  — terraform apply (hỏi confirm)
- `make destroy ENV=dev`    — terraform destroy (chỉ dev)

## Rules bất khả xâm phạm
- KHÔNG chạy apply trực tiếp vào prod — chỉ qua CI/CD
- KHÔNG hardcode secrets trong bất kỳ file nào
- KHÔNG xóa resources trong prod mà không có backup plan
- Mọi thay đổi prod cần PR + 2 approvals
```

---

## 8. Open Source Library

```markdown
# Library: [Tên Library]

## Overview
[Mô tả một dòng]. Language: Python. Target: Python 3.9+.
PyPI: [package-name]. Stars: ~[số].

## Design principles
- Zero dependencies bên ngoài stdlib khi có thể
- Backwards compatibility là priority 1
- Public API thay đổi chỉ qua deprecation cycle
- Performance: benchmark trước khi merge

## Public API rules
- Tất cả public functions cần docstring + type hints
- Không thay đổi function signatures mà không deprecate trước
- Thêm `DeprecationWarning` ít nhất 1 minor version trước khi remove

## Testing requirements
- Coverage > 95% cho public API
- Benchmark tests cho performance-critical paths
- Integration tests trên Python 3.9, 3.10, 3.11, 3.12

## Commands
- `make test`       — pytest với coverage
- `make bench`      — chạy benchmarks
- `make docs`       — build docs (Sphinx)
- `make release`    — bump version, build, publish

## Changelog
Mọi PR cần entry trong CHANGELOG.md (format: Keep a Changelog).
```

---

## 9. Subdirectory CLAUDE.md patterns

### 9.1 src/api/ — API module specifics

```markdown
# API Module

## Responsibility
HTTP handlers và routing only. Business logic thuộc về src/services/.

## Conventions
- Handler function: validate input, call service, format response
- Không có business logic trong handlers
- Không direct database access — chỉ qua services

## Pattern
```python
@router.post("/users", response_model=UserResponse, status_code=201)
async def create_user(data: CreateUserRequest, service: UserService = Depends()):
    user = await service.create_user(data)  # business logic ở service
    return UserResponse.from_orm(user)
```
```

### 9.2 tests/ — Test module specifics

```markdown
# Tests

## Structure
tests/
  unit/         — isolated, no I/O, no DB
  integration/  — requires running postgres + redis
  e2e/          — requires full stack running

## Conventions
- AAA pattern: Arrange, Act, Assert
- Fixtures trong conftest.py
- factory_boy cho model factories
- Không hardcode IDs trong tests

## Commands
- `pytest tests/unit/`         — unit only, fast
- `pytest tests/integration/`  — cần docker-compose up
- `pytest -k test_name`        — run specific test
```

### 9.3 scripts/ — Utility scripts

```markdown
# Scripts

Tất cả scripts ở đây là one-off utilities, không phải production code.
Không cần production-level error handling hay logging.

Scripts cần: idempotent (có thể chạy lại an toàn).
Scripts không cần: perfect code style.

Trước khi chạy script nào: `python script.py --dry-run` để preview.
```

---

## 10. Import files pattern

Khi CLAUDE.md chính cần giữ ngắn nhưng vẫn muốn accessible context:

**CLAUDE.md (30 dòng):**
```markdown
# Project: API Service

## Quick reference
Stack: FastAPI + PostgreSQL + Redis. See imports for details.

## Commands
- `make dev` / `make test` / `make lint` / `make migrate`

## Critical rules
- Không sửa migrations/
- Không commit .env

## References
@.claude/docs/architecture.md
@.claude/docs/api-conventions.md
@.claude/docs/testing-guide.md
```

**.claude/docs/architecture.md (loaded on demand):**
```markdown
# Architecture Details

## Service boundaries
[chi tiết dài về architecture...]

## Database schema decisions
[schema reasoning...]

## External dependencies
[integration details...]
```

Claude chỉ load import files khi thật sự cần, giữ main CLAUDE.md nhẹ trong mọi turn.

---

## 11. Checklist CLAUDE.md tốt

Đọc lại CLAUDE.md và kiểm tra:

- [ ] Dưới 60 dòng (không tính code blocks)
- [ ] Có WHY: tại sao project tồn tại, tại sao decision X được chọn
- [ ] Có commands với mô tả ngắn
- [ ] Có code style cụ thể (không phải vague như "clean code")
- [ ] Có model assignment rules
- [ ] Có constraints rõ ràng (không được sửa X, không dùng Y)
- [ ] Không chứa thông tin thay đổi thường xuyên
- [ ] Không chứa nội dung đã có trong README
- [ ] Được review và cập nhật sau mỗi architecture decision lớn
