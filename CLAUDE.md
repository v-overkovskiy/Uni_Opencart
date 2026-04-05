# CLAUDE.md — Полный план реализации VisualCard

> Этот файл — мастер-план для Claude Code. Выполняй последовательно, фаза за фазой.
> После каждой фазы — коммить с тегом фазы, проверяй `docker-compose up`, переходи дальше.
> Продукт — **VisualCard** — отдельный сервис на поддомене **visual.uplify.agency**.
> Является частью экосистемы Uplify наряду с prom.uplify.agency (AI Optimizer фидов).

---

## АРХИТЕКТУРНЫЕ ИНВАРИАНТЫ

> Этот раздел — долгоживущие принципы и ограничения. Не меняются между фазами.
> Все фазы ниже должны следовать этим правилам.

### Продуктовая рамка
- Отдельный сервис на `visual.uplify.agency`, НЕ режим внутри Prom
- Reuse от Prom: только платформенная обвязка (auth-паттерн, billing-паттерн, landing-структура, legal pages)
- Домен, UX и generation pipeline — полностью отдельные
- Core UX: **guided generator**, не editor-first. Продукт = "умная студия", не canvas

### Доменная модель (инварианты)
- `Project/Workspace → Assets → Jobs → Outputs`
- Один проект может включать несколько исходных фото (до 4)
- Jobs могут потреблять несколько assets (M2M через `job_assets`)
- Outputs отделены от Jobs (1 job → N output-вариантов)
- Retries, варианты, provider attempts НЕ сплющиваются в одну гигантскую строку

### Три output mode (первого класса)
- **PHOTO** — studio/catalog, in environment, in use, premium hero shot
- **BANNER** — promo, feature highlight, price offer, seasonal, retargeting. НЕ подтип инфографики
- **VIDEO** — motion from image, zoom/reveal, product showcase. НЕ блокирует core photo/banner flow

### AI/Provider архитектура
- Provider abstraction для image и video генерации (Protocol-based interfaces)
- НЕ хардкодить provider-specific логику в бизнес-сервисах
- Адаптеры: `ImageGenerationProvider`, `VideoGenerationProvider`, `BackgroundRemovalProvider`
- Fallback + circuit breaker между провайдерами

### Финмодель (инварианты)
- Валюта: **токены** (не "искры", не "кредиты")
- Source of truth для баланса: `TokenTransaction` ledger
- `user.token_balance` — денормализованный кэш, НЕ источник истины
- Автоопределение товара — бесплатно
- One-time пакеты + позже monthly subscriptions
- Ledger поддерживает: top-ups, списания, bonus, refunds, promo grants, rollover

### Deployment (инварианты)
- Отдельный docker compose проект
- Отдельная БД, Redis namespace, volumes, env/secrets, worker queues
- Отдельный субдомен + nginx routing
- `prom.uplify.agency` остаётся как припаркованный low-load сервис, НЕ удаляется
- НЕ мержить visual и Prom в один runtime/application
- Storage: S3/R2 compatible object storage, НЕ локальный диск

### Что НЕ делать (hard constraints)
- ❌ Canvas/editor-first продукт в V1
- ❌ Смешивать feed/XLSX domain с visual domain
- ❌ Двойной source of truth для баланса
- ❌ Provider-specific бизнес-логику в domain core
- ❌ Giant SPA без обоснования
- ❌ Overbuilt abstractions — pragmatic > perfect

---

## ГЛОБАЛЬНЫЙ КОНТЕКСТ

### Что строим

SaaS-сервис для продавцов на маркетплейсах Украины и Европы.
Продавец загружает фото товара со смартфона → AI автоматически распознаёт тип товара →
убирает фон → генерирует профессиональный фон/сцену → добавляет инфографику (тексты, плашки, преимущества) → на выходе готовая продающая карточка товара.

Дополнительные модули (post-MVP): генерация баннеров для маркетплейсов и соцсетей, AI-видео через xAI Grok Aurora.

### Референс

**aidentika.com** — аналог для российского рынка (WB, Ozon). Ключевой UX:
1. Upload фото → AI детектит тип товара (можно поправить вручную)
2. Выбор концепции: "В использовании" / "В окружении" / "Каталог (студийно)"
3. Пожелания (текстовое поле) + расширенные настройки
4. Тип контента: Фото / Карточка (инфографика) / Видео
5. Кнопка "Сгенерировать" → 4 кредита → результат за ~60 сек

**Важно:** Aidentika продаёт **guided generator** ("3 шага — и готово"), а НЕ canva-подобный редактор. Мы копируем этот подход.

### Что переиспользуем из prom.uplify.agency

| Компонент | Переиспользуем | Заметки |
|-----------|---------------|---------|
| Auth (Google OAuth + email/password) | Да, 80% | Тот же паттерн, та же БД users |
| Кредитная/балансовая модель | Да, 70% | Адаптировать под "искры" |
| Landing page структура | Да, 60% | Визуальный тон, hero, FAQ, pricing карточки |
| Юридические страницы | Да, 90% | Оферта, политика конфиденциальности, возврат |
| Job/batch infrastructure | Частично, 40% | Celery/Redis pipeline, но UX другой |
| Доменная модель (XLSX/Prom) | НЕТ | Совершенно другой домен |
| UX "загрузил → подождал → скачал файл" | НЕТ | Нужен real-time visual creation |
| Стиль бренда Uplify | Да | Но светлая тема (Aidentika-style) |

### Что НЕ делаем в V1

- ❌ Fabric.js / canvas editor (слишком сложно, не нужно для guided generator)
- ❌ Virtual try-on (отдельный продукт, не MVP)
- ❌ Полноценный multi-marketplace export с первого дня (generic + Prom/Rozetka presets)
- ❌ NextAuth + собственный JWT одновременно (выбираем ОДИН auth-контур)
- ❌ 5 языков с первого дня (только UK + EN, остальные позже)

---

## СТЕК ТЕХНОЛОГИЙ

```
Frontend:     Next.js 14+ (App Router), TypeScript, TailwindCSS, shadcn/ui, Zustand
Backend API:  Python 3.12+, FastAPI, SQLAlchemy 2.0, Alembic
AI Images:    Google Nano Banana 2 (основной) + Nano Banana Pro (premium/инфографика)
AI Video:     xAI Grok Aurora API ($0.15/видео)
AI Fallback:  Replicate API (bg removal, fallback для изображений)
DB:           PostgreSQL 16, Redis 7
Storage:      AWS S3 (MinIO для dev)
Queue:        Celery + Redis (broker)
Auth:         JWT (python-jose) — единый контур, Google OAuth через backend
Payments:     WayForPay (UA), Stripe (EU)
Infra:        Docker Compose (dev), Docker (prod)
CDN:          Cloudflare
```

### Архитектура провайдеров (Provider Layer)

**Критично:** AI-провайдеры НЕ вызываются напрямую из бизнес-логики.
Все вызовы идут через абстрактные интерфейсы:

```python
# Каждый провайдер — отдельный адаптер
class BackgroundRemovalProvider(Protocol):
    async def remove(self, image_url: str) -> RemovalResult: ...

class SceneGenerationProvider(Protocol):
    async def generate(self, prompt: str, params: SceneParams) -> str: ...

class ImageEditProvider(Protocol):
    async def edit(self, image_url: str, prompt: str, params: EditParams) -> str: ...

class VideoGenerationProvider(Protocol):
    async def generate(self, image_url: str, params: VideoParams) -> str: ...

# ОСНОВНЫЕ реализации:
class NanoBanana2SceneGeneration(SceneGenerationProvider): ...    # Gemini 3.1 Flash Image — быстрый, дешёвый ($0.08)
class NanoBananaProImageEdit(ImageEditProvider): ...              # Gemini 3 Pro Image — лучший текст, инфографика ($0.13)
class NanoBanana2BackgroundRemoval(BackgroundRemovalProvider): ... # Через edit prompt: "remove background"
class GrokVideoGeneration(VideoGenerationProvider): ...           # xAI Grok Aurora — видео ($0.15/6сек)

# FALLBACK реализации:
class ReplicateBackgroundRemoval(BackgroundRemovalProvider): ...   # rembg/remove-bg если NB2 не справляется
class ReplicateSceneGeneration(SceneGenerationProvider): ...       # SDXL/Flux как запасной

# Позже можно добавить:
class RunPodSelfHosted(SceneGenerationProvider): ...
```

**Стратегия провайдеров:**
- **Изображения:** Nano Banana 2 (основной — $0.08, 4-6 сек) → Nano Banana Pro (premium/инфографика — $0.13) → Replicate (fallback)
- **Background removal:** Nano Banana 2 edit mode → Replicate rembg (fallback)
- **Видео:** xAI Grok Aurora (единственный — $0.15/6сек видео)
- **Детекция товара:** Nano Banana 2 (встроенный reasoning Gemini, бесплатно в рамках генерации)

Это позволяет менять провайдеров без перелома ядра.

---

## СТРУКТУРА МОНОРЕПО

```
visualcard/
├── CLAUDE.md
├── docker-compose.yml
├── docker-compose.prod.yml
├── .env.example
├── .gitignore
├── frontend/
│   ├── package.json
│   ├── next.config.ts
│   ├── tailwind.config.ts
│   ├── tsconfig.json
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx              # Landing
│   │   │   ├── (auth)/
│   │   │   │   ├── login/page.tsx
│   │   │   │   └── register/page.tsx
│   │   │   ├── (dashboard)/
│   │   │   │   ├── layout.tsx
│   │   │   │   ├── create/page.tsx       # Главный — генерация карточек
│   │   │   │   ├── banners/page.tsx      # Генерация баннеров
│   │   │   │   ├── video/page.tsx        # AI Видео (Grok)
│   │   │   │   ├── gallery/page.tsx
│   │   │   │   ├── billing/page.tsx
│   │   │   │   └── settings/page.tsx
│   │   │   └── api/
│   │   │       └── webhooks/
│   │   ├── components/
│   │   │   ├── ui/                  # shadcn/ui
│   │   │   ├── layout/             # Header, Sidebar, Footer
│   │   │   ├── create/             # Upload, ConceptPicker, ResultView
│   │   │   ├── banners/            # BannerPresetStudio, FormatPicker
│   │   │   └── billing/            # PlanCards, TransactionHistory
│   │   ├── lib/
│   │   │   ├── api.ts
│   │   │   ├── auth.ts
│   │   │   ├── utils.ts
│   │   │   └── constants.ts
│   │   ├── stores/
│   │   │   ├── auth.ts
│   │   │   ├── create.ts           # main generation flow
│   │   │   ├── banners.ts
│   │   │   └── video.ts
│   │   ├── types/
│   │   │   └── index.ts
│   │   └── i18n/
│   │       ├── uk.json
│   │       └── en.json
│   └── public/
│       └── concepts/               # Превью концепций
├── backend/
│   ├── pyproject.toml
│   ├── alembic.ini
│   ├── alembic/
│   │   └── versions/
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── database.py
│   │   ├── dependencies.py
│   │   ├── models/
│   │   │   ├── __init__.py
│   │   │   ├── user.py
│   │   │   ├── project.py           # Product workspace (groups assets + jobs)
│   │   │   ├── asset.py             # Source images (uploaded)
│   │   │   ├── job.py               # Processing job (attempt)
│   │   │   ├── job_asset.py         # M2M: job ↔ multiple assets
│   │   │   ├── output.py            # Generated output variants
│   │   │   ├── template_preset.py   # Infographic presets
│   │   │   ├── token.py             # Token transactions ledger
│   │   │   └── payment.py
│   │   ├── schemas/
│   │   │   ├── __init__.py
│   │   │   ├── auth.py
│   │   │   ├── asset.py
│   │   │   ├── job.py
│   │   │   ├── output.py
│   │   │   └── billing.py
│   │   ├── api/
│   │   │   ├── __init__.py
│   │   │   ├── router.py
│   │   │   ├── auth.py
│   │   │   ├── assets.py
│   │   │   ├── jobs.py
│   │   │   ├── outputs.py
│   │   │   ├── banners.py
│   │   │   ├── video.py
│   │   │   ├── billing.py
│   │   │   ├── export.py
│   │   │   └── webhooks.py
│   │   ├── services/
│   │   │   ├── __init__.py
│   │   │   ├── auth_service.py
│   │   │   ├── token_service.py
│   │   │   ├── job_orchestrator.py   # Оркестрация pipeline
│   │   │   ├── storage_service.py
│   │   │   ├── payment_service.py
│   │   │   ├── product_detector.py   # Auto-detect product type
│   │   │   ├── banner_service.py
│   │   │   └── export_service.py
│   │   ├── workers/
│   │   │   ├── __init__.py
│   │   │   ├── celery_app.py
│   │   │   ├── card_generation.py     # Background removal + scene + composite
│   │   │   ├── infographic_overlay.py
│   │   │   ├── banner_generation.py
│   │   │   └── video_generation.py
│   │   ├── providers/                 # Provider abstraction layer
│   │   │   ├── __init__.py
│   │   │   ├── base.py               # Protocol/ABC interfaces
│   │   │   ├── nano_banana.py         # Google Nano Banana 2 + Pro (основной)
│   │   │   ├── replicate_fallback.py  # Replicate bg removal + scene (fallback)
│   │   │   ├── grok_video.py          # xAI Grok Aurora video
│   │   │   └── provider_registry.py   # Factory + fallback logic
│   │   ├── ai/
│   │   │   ├── __init__.py
│   │   │   ├── compositor.py          # Pillow compositing
│   │   │   ├── infographic_engine.py  # Pillow text/badge overlay
│   │   │   ├── banner_engine.py       # Banner layout + rendering
│   │   │   └── prompts/
│   │   │       ├── __init__.py
│   │   │       ├── scene_prompts.py
│   │   │       ├── banner_prompts.py
│   │   │       └── video_prompts.py
│   │   └── middleware/
│   │       ├── rate_limiter.py        # Per-user rate limiting
│   │       ├── idempotency.py         # Webhook idempotency
│   │       └── moderation.py          # Content moderation
│   ├── tests/
│   │   ├── conftest.py
│   │   ├── test_auth.py
│   │   ├── test_jobs.py
│   │   ├── test_billing.py
│   │   └── test_providers.py
│   └── Dockerfile
└── infra/
    ├── nginx/nginx.conf
    └── scripts/
        ├── init-db.sh
        └── seed-presets.py
```

---

## ФАЗА 0: ИНИЦИАЛИЗАЦИЯ ПРОЕКТА (неделя 1)

### Шаг 0.1 — Монорепо + Git

```
Создай директорию visualcard/ со всей структурой из раздела выше.
Инициализируй git. Создай .gitignore для Python + Node.js + Docker.
```

### Шаг 0.2 — Docker Compose (dev)

```yaml
# docker-compose.yml — сервисы:
# 1. postgres:16-alpine     (порт 5432, volume pgdata)
# 2. redis:7-alpine          (порт 6379)
# 3. minio                   (порт 9000/9001, S3 для dev)
# 4. backend                 (FastAPI, порт 8000, hot-reload, depends postgres+redis)
# 5. celery-worker           (тот же образ backend, command: celery -A app.workers.celery_app worker)
# 6. frontend                (Next.js, порт 3000, hot-reload)
```

### Шаг 0.3 — .env.example

```env
# Database
DATABASE_URL=postgresql+asyncpg://visualcard:visualcard@postgres:5432/visualcard
REDIS_URL=redis://redis:6379/0

# S3
S3_ENDPOINT=http://minio:9000
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=visualcard

# AI Providers — Images (Google Nano Banana)
GOOGLE_AI_API_KEY=AIzaxxxxxxxxxxxx
NANO_BANANA_MODEL_FAST=gemini-2.0-flash-exp       # Nano Banana 2
NANO_BANANA_MODEL_PRO=gemini-3.0-pro-exp           # Nano Banana Pro

# AI Providers — Video (xAI Grok Aurora)
XAI_API_KEY=xai-xxxxxxxxxxxx
XAI_BASE_URL=https://api.x.ai/v1

# AI Providers — Fallback (Replicate)
REPLICATE_API_TOKEN=r8_xxxxxxxxxxxx

# Auth (единый JWT контур)
JWT_SECRET=change-me-in-production
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=1440
JWT_REFRESH_TOKEN_EXPIRE_DAYS=30
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# Payments
WAYFORPAY_MERCHANT_LOGIN=
WAYFORPAY_MERCHANT_SECRET=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=

# App
FRONTEND_URL=http://localhost:3000
BACKEND_URL=http://localhost:8000
```

### Шаг 0.4 — Инициализация Frontend

```
cd frontend/
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir

Зависимости:
- shadcn/ui (init + компоненты: button, card, dialog, dropdown-menu, input,
  label, select, separator, sheet, skeleton, tabs, toast, tooltip, avatar,
  badge, progress, slider, accordion)
- zustand
- axios
- react-dropzone
- lucide-react
- react-hot-toast
- framer-motion
- zod + react-hook-form + @hookform/resolvers
- @tanstack/react-query
```

### Шаг 0.5 — Инициализация Backend

```
cd backend/

pyproject.toml:
- fastapi[standard]>=0.115
- uvicorn[standard]
- sqlalchemy[asyncio]>=2.0
- asyncpg
- alembic
- pydantic-settings
- python-jose[cryptography]
- passlib[bcrypt]
- celery[redis]
- redis
- boto3
- httpx
- google-generativeai        # Nano Banana 2 + Pro (Gemini API)
- replicate                  # fallback provider
- Pillow
- python-multipart
- wayforpay                      # UA payment provider (or raw httpx calls)
- stripe

Создай:
- app/config.py → Pydantic Settings (все env)
- app/main.py → FastAPI app + CORS (localhost:3000) + exception handlers
- app/database.py → async engine + async sessionmaker
```

**Коммит:** `git commit -m "Phase 0: Project init — monorepo, Docker, Next.js 14, FastAPI"`

---

## ФАЗА 1: БАЗА ДАННЫХ — НОРМАЛИЗОВАННАЯ СХЕМА (неделя 1-2)

> **Ключевое отличие от v1:** модель Generation разбита на 3 сущности:
> Asset (загруженное фото), Job (попытка обработки), Output (результат).
> Это позволит легко добавлять retry, варианты, и не раздувать одну таблицу.

### Шаг 1.1 — SQLAlchemy модели

**Project (рабочее пространство для одного товара, группирует assets и jobs):**
```python
class Project(Base):
    __tablename__ = "projects"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"), index=True)
    name: Mapped[str] = mapped_column(String(200))                    # auto-filled from detected product name
    category: Mapped[Optional[str]] = mapped_column(String(50))       # user-confirmed category
    status: Mapped[str] = mapped_column(String(20), default="active") # active, archived
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())
```

**User:**
```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    hashed_password: Mapped[Optional[str]] = mapped_column(String(255))  # null для Google OAuth
    name: Mapped[str] = mapped_column(String(100))
    avatar_url: Mapped[Optional[str]] = mapped_column(String(500))
    google_id: Mapped[Optional[str]] = mapped_column(String(100), unique=True)
    locale: Mapped[str] = mapped_column(String(5), default="uk")
    token_balance: Mapped[int] = mapped_column(default=20)  # ⚠️ DENORMALIZED CACHE — source of truth = TokenTransaction ledger. Обновлять ТОЛЬКО через TokenService с SELECT FOR UPDATE. Периодическая reconciliation: SUM(token_transactions.amount) WHERE user_id = X. Welcome bonus: 20 токенов (≈2 фото)
    region: Mapped[str] = mapped_column(String(2), default="ua")  # ua, eu → определяет валюту
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())
```

**Asset (загруженное исходное фото):**
```python
class Asset(Base):
    __tablename__ = "assets"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"), index=True)
    project_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("projects.id"), index=True)
    original_url: Mapped[str] = mapped_column(String(500))         # S3 URL оригинала
    no_bg_url: Mapped[Optional[str]] = mapped_column(String(500))  # PNG без фона
    mask_url: Mapped[Optional[str]] = mapped_column(String(500))   # маска сегментации
    detected_category: Mapped[Optional[str]] = mapped_column(String(50))  # auto-detected
    user_category: Mapped[Optional[str]] = mapped_column(String(50))      # user override
    detected_product_name: Mapped[Optional[str]] = mapped_column(String(200))
    width: Mapped[Optional[int]] = mapped_column()
    height: Mapped[Optional[int]] = mapped_column()
    file_size_bytes: Mapped[Optional[int]] = mapped_column()
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

**Job (попытка обработки — 1 asset может иметь N jobs):**
```python
class JobStatus(str, enum.Enum):
    PENDING = "pending"
    REMOVING_BG = "removing_bg"
    GENERATING_SCENE = "generating_scene"
    COMPOSITING = "compositing"
    ADDING_INFOGRAPHIC = "adding_infographic"
    GENERATING_BANNER = "generating_banner"
    GENERATING_VIDEO = "generating_video"
    COMPLETED = "completed"
    FAILED = "failed"

class JobType(str, enum.Enum):
    CARD = "card"               # стандартное фото (8 токенов)
    CARD_PREMIUM = "card_premium"  # premium/lifestyle фото (10 токенов)
    CARD_INFOGRAPHIC = "card_infographic"  # фото + инфографика (12 токенов)
    BANNER = "banner"           # рекламный баннер (12 токенов)
    VIDEO = "video"             # AI видео (60 токенов)
    VARIATION = "variation"     # regenerate / вариация (5 токенов)
    UPSCALE = "upscale"         # upscale 2x (3 токена)

class Job(Base):
    __tablename__ = "jobs"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"), index=True)
    project_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("projects.id"), index=True)
    # Assets linked via M2M table job_assets (поддержка до 4 фото)
    type: Mapped[JobType] = mapped_column()
    status: Mapped[JobStatus] = mapped_column(default=JobStatus.PENDING)

    # Настройки генерации
    concept: Mapped[str] = mapped_column(String(100))         # in_use, in_scene, catalog
    scene_preset: Mapped[Optional[str]] = mapped_column(String(100))  # studio_white, lifestyle_interior, etc.
    user_prompt: Mapped[Optional[str]] = mapped_column(Text)  # пожелания пользователя
    infographic_preset_id: Mapped[Optional[uuid.UUID]] = mapped_column(ForeignKey("infographic_presets.id"))

    # Banner-specific
    banner_format: Mapped[Optional[str]] = mapped_column(String(50))  # marketplace_promo, instagram_story, etc.
    banner_text: Mapped[Optional[dict]] = mapped_column(JSON)

    # Video-specific
    video_scenario: Mapped[Optional[str]] = mapped_column(String(100))

    # Provider tracking
    provider_name: Mapped[Optional[str]] = mapped_column(String(50))  # replicate, grok, etc.
    provider_job_id: Mapped[Optional[str]] = mapped_column(String(200))
    provider_cost_usd: Mapped[Optional[Decimal]] = mapped_column(Numeric(8, 4))

    # Credits & timing
    tokens_spent: Mapped[int] = mapped_column(default=0)
    processing_time_ms: Mapped[Optional[int]] = mapped_column()
    error_message: Mapped[Optional[str]] = mapped_column(Text)
    retry_count: Mapped[int] = mapped_column(default=0)

    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    completed_at: Mapped[Optional[datetime]] = mapped_column()
```

**Output (результат — 1 job может создать N output-вариантов):**
```python
class Output(Base):
    __tablename__ = "outputs"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    job_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("jobs.id"), index=True)
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"), index=True)
    variant_index: Mapped[int] = mapped_column(default=0)  # если генерим 2-4 варианта

    image_url: Mapped[Optional[str]] = mapped_column(String(500))
    video_url: Mapped[Optional[str]] = mapped_column(String(500))
    thumbnail_url: Mapped[Optional[str]] = mapped_column(String(500))
    width: Mapped[Optional[int]] = mapped_column()
    height: Mapped[Optional[int]] = mapped_column()

    is_favorite: Mapped[bool] = mapped_column(default=False)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

**JobAsset (M2M: job ↔ multiple assets, до 4 фото товара с разных сторон):**
```python
class JobAsset(Base):
    __tablename__ = "job_assets"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    job_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("jobs.id"), index=True)
    asset_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("assets.id"), index=True)
    role: Mapped[str] = mapped_column(String(20), default="primary")  # primary, angle_2, angle_3, angle_4
    sort_order: Mapped[int] = mapped_column(default=0)

    __table_args__ = (UniqueConstraint("job_id", "asset_id"),)
```

**InfographicPreset (шаблоны инфографики — preset-based, НЕ canvas editor):**
```python
class InfographicPreset(Base):
    __tablename__ = "infographic_presets"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(200))
    category: Mapped[str] = mapped_column(String(50))
    preview_url: Mapped[str] = mapped_column(String(500))
    layout_data: Mapped[dict] = mapped_column(JSON)
    # { "slots": [{ "type": "title", "x": ..., "y": ..., "maxChars": 40 }, ...], "colors": {...} }
    is_active: Mapped[bool] = mapped_column(default=True)
    sort_order: Mapped[int] = mapped_column(default=0)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

**TokenTransaction (леджер — source of truth для баланса):**
```python
class TransactionType(str, enum.Enum):
    PURCHASE = "purchase"       # покупка пакета
    SUBSCRIPTION = "subscription"  # ежемесячное начисление по подписке
    SPEND = "spend"             # списание за генерацию
    BONUS = "bonus"             # welcome bonus, промо
    REFUND = "refund"           # возврат за failed job
    ROLLOVER = "rollover"       # перенос неиспользованных токенов подписки

class TokenTransaction(Base):
    __tablename__ = "token_transactions"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"), index=True)
    type: Mapped[TransactionType] = mapped_column()
    amount: Mapped[int] = mapped_column()  # + начисление, - списание
    balance_after: Mapped[int] = mapped_column()
    description: Mapped[str] = mapped_column(String(500))
    job_id: Mapped[Optional[uuid.UUID]] = mapped_column(ForeignKey("jobs.id"))
    payment_id: Mapped[Optional[uuid.UUID]] = mapped_column(ForeignKey("payments.id"))
    idempotency_key: Mapped[Optional[str]] = mapped_column(String(100), unique=True)  # защита от дублей
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

**Payment:**
```python
class PaymentStatus(str, enum.Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    FAILED = "failed"
    REFUNDED = "refunded"

class PaymentProvider(str, enum.Enum):
    WAYFORPAY = "wayforpay"
    STRIPE = "stripe"

class Payment(Base):
    __tablename__ = "payments"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"), index=True)
    provider: Mapped[PaymentProvider] = mapped_column()
    external_id: Mapped[Optional[str]] = mapped_column(String(200), unique=True)
    idempotency_key: Mapped[str] = mapped_column(String(100), unique=True)
    amount: Mapped[Decimal] = mapped_column(Numeric(10, 2))
    currency: Mapped[str] = mapped_column(String(3))
    tokens_amount: Mapped[int] = mapped_column()
    package_name: Mapped[str] = mapped_column(String(50))
    status: Mapped[PaymentStatus] = mapped_column(default=PaymentStatus.PENDING)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    completed_at: Mapped[Optional[datetime]] = mapped_column()
```

### Шаг 1.2 — Alembic

```
alembic init --template async alembic
Настрой env.py → импортировать все модели → target_metadata = Base.metadata
alembic revision --autogenerate -m "initial schema"
```

### Шаг 1.3 — Pydantic Schemas

Для каждой модели — отдельные Request/Response schemas в `backend/app/schemas/`.
Используй `model_config = ConfigDict(from_attributes=True)`.

**Коммит:** `git commit -m "Phase 1: Normalized DB schema — assets, jobs, outputs, credits ledger"`

---

## ФАЗА 2: АУТЕНТИФИКАЦИЯ — ЕДИНЫЙ JWT КОНТУР (неделя 2)

> **Без NextAuth.** Один auth через backend (JWT). Google OAuth тоже через backend.
> Frontend хранит token в httpOnly cookie или в памяти + refresh.

### Шаг 2.1 — Auth Service

Создай `backend/app/services/auth_service.py`:

```python
class AuthService:
    async def register(self, email: str, password: str, name: str) -> tuple[User, TokenPair]:
        """Создать юзера + 20 бонусных токенов (≈2 фото) + вернуть JWT"""

    async def login(self, email: str, password: str) -> TokenPair:
        """Проверить пароль → вернуть access (24h) + refresh (30d) токены"""

    async def google_auth(self, google_token: str) -> tuple[User, TokenPair]:
        """
        Верифицировать Google ID token → найти или создать юзера →
        вернуть JWT. Для новых — 20 бонусных токенов.
        """

    async def refresh(self, refresh_token: str) -> TokenPair:
        """Новый access token по refresh token"""

    async def get_current_user(self, token: str) -> User:
        """FastAPI Dependency — из Authorization: Bearer header"""
```

### Шаг 2.2 — Auth API Routes

```
POST /api/auth/register          — email + password + name
POST /api/auth/login             — email + password → tokens
POST /api/auth/google            — google id_token → tokens
POST /api/auth/refresh           — refresh_token → new access_token
GET  /api/auth/me                — текущий юзер (protected)
```

### Шаг 2.3 — Frontend Auth

```
1. Zustand store: auth.ts — token, user, login(), register(), googleAuth(), logout()
2. axios interceptor: прикрепляет Bearer token, при 401 → refresh → retry
3. Страница /login — email/password + "Увійти через Google" (Google Identity Services)
4. Страница /register — name + email + password + "Зареєструватися через Google"
5. middleware.ts — редирект неавторизованных с /dashboard/* на /login
```

Дизайн: как у prom.uplify.agency — тёмная форма регистрации, но для VisualCard — **светлая тема**, белый фон, зелёный акцент #4ADE80, чёрные кнопки.

**Коммит:** `git commit -m "Phase 2: Auth — single JWT, Google OAuth, login/register UI"`

---

## ФАЗА 3: ЗАГРУЗКА + АВТО-ДЕТЕКЦИЯ ТОВАРА (неделя 3)

### Шаг 3.1 — Storage Service

```python
class StorageService:
    async def upload_image(self, file: UploadFile, user_id: str) -> Asset:
        """
        1. Валидация: JPEG/PNG/WebP/HEIC, max 20MB
        2. HEIC → JPEG через Pillow
        3. Создать thumbnail (400px по длинной стороне)
        4. Upload в S3: {user_id}/originals/{uuid}.{ext}
        5. Создать запись Asset в БД
        6. Вернуть Asset с URLs
        """

    async def get_presigned_url(self, key: str, expires: int = 3600) -> str
    async def delete_file(self, key: str) -> None
```

### Шаг 3.2 — Product Detector (как у Aidentika)

Создай `backend/app/services/product_detector.py`:

```python
class ProductDetector:
    """
    Автоматически определяет тип товара по фото.
    Использует Nano Banana 2 (Gemini reasoning) — бесплатно в рамках API вызова.
    Не нужен отдельный BLIP-2/LLaVA — Gemini понимает изображения нативно.
    """

    CATEGORIES = {
        "clothing": ["платье", "футболка", "куртка", "брюки", "пальто", ...],
        "shoes": ["кроссовки", "туфли", "ботинки", "сапоги", ...],
        "jewelry": ["кольцо", "серьги", "ожерелье", "браслет", ...],
        "electronics": ["телефон", "ноутбук", "наушники", "планшет", ...],
        "home": ["подушка", "ваза", "лампа", "полка", ...],
        "food": ["чай", "кофе", "шоколад", "специи", ...],
        "cosmetics": ["крем", "помада", "тушь", "парфюм", ...],
        "pets": ["корм", "миска", "ошейник", "игрушка", ...],
        "tools": ["ключ", "отвёртка", "дрель", "молоток", ...],
        "other": [],
    }

    async def detect(self, image_url: str) -> DetectionResult:
        """
        1. Вызвать Nano Banana 2 (Gemini) с image + structured prompt:
           "Analyze this product photo. Return JSON:
            {category, product_name_uk, product_name_en, confidence}"
        2. Gemini reasoning определяет тип товара без отдельной модели
        3. Вернуть: { category: "tools", product_name: "Ручний тріщотковий ключ", confidence: 0.85 }
        """
```

### Шаг 3.3 — Asset API

```
POST /api/assets/upload          — загрузить фото (multipart)
                                   → auto-detect category + product name
                                   → вернуть Asset с detected_category

PATCH /api/assets/{id}           — юзер поправил category / product_name вручную
GET   /api/assets                — список загруженных фото юзера
DELETE /api/assets/{id}          — удалить
```

### Шаг 3.4 — Frontend: Upload + Detection

```
Компонент UploadFlow:
1. Большая dropzone "Перетягніть фото товару або натисніть для завантаження"
2. После загрузки — показать превью + spinner "Визначаємо товар..."
3. Когда API вернул результат:
   - Поле "Це:" с автозаполненным названием (editable input)
   - Dropdown "Категорія:" с detected_category (editable)
   - Как у Aidentika: юзер может поправить, если AI ошибся
4. Кнопка "Далі →"
```

**Коммит:** `git commit -m "Phase 3: Upload, auto-detect product type, S3 storage"`

---

## ФАЗА 4: CORE — ГЕНЕРАЦИЯ КАРТОЧЕК (неделя 3-5)

### Шаг 4.1 — Provider Layer

Создай `backend/app/providers/`:

**base.py — интерфейсы:**
```python
from typing import Protocol

class BackgroundRemovalProvider(Protocol):
    async def remove(self, image_url: str) -> RemovalResult: ...

class SceneGenerationProvider(Protocol):
    async def generate(self, prompt: str, negative_prompt: str,
                       width: int, height: int) -> str: ...

class ImageEditProvider(Protocol):
    async def edit(self, image_url: str, prompt: str) -> str: ...
```

**nano_banana.py — ОСНОВНОЙ провайдер:**
```python
import google.generativeai as genai  # pip install google-generativeai

class NanoBananaProvider:
    """
    Google Nano Banana — основной провайдер для всех операций с изображениями.

    Модели:
    - Nano Banana 2 (gemini-2.0-flash-exp) — быстрый, $0.08/img, 4-6 сек
    - Nano Banana Pro (gemini-3.0-pro-exp) — premium, $0.13/img, текст/инфографика

    API: Google AI Studio (Gemini API)
    Документация: https://ai.google.dev/gemini-api/docs/image-generation
    """

    def __init__(self, api_key: str):
        genai.configure(api_key=api_key)
        self.model_fast = genai.GenerativeModel("gemini-2.0-flash-exp")    # NB2
        self.model_pro = genai.GenerativeModel("gemini-3.0-pro-exp")       # NB Pro

    async def remove_background(self, image_url: str) -> RemovalResult:
        """
        Nano Banana 2 edit mode:
        Prompt: "Remove the background completely, make it transparent PNG"
        Быстро ($0.08), хорошее качество для большинства товаров.
        """

    async def generate_scene(self, product_no_bg: bytes, prompt: str,
                             width: int, height: int, use_pro: bool = False) -> str:
        """
        Генерация сцены с товаром:
        1. Передать product image + prompt
        2. Nano Banana 2 (default) или Pro (для premium)
        3. Промпт: "Place this product in {scene_description}, professional photography"
        4. Вернуть URL результата

        use_pro=True для сложных сцен и инфографики с текстом.
        """

    async def generate_infographic(self, image: bytes, text_data: dict,
                                   preset_style: str) -> str:
        """
        Nano Banana Pro — ЛУЧШИЙ для текста на изображениях.
        Промпт включает: layout description + text content + style
        Рендерит текст, бейджи, цены прямо в генерации.
        Мультиязычный: украинский, русский, английский текст.
        """

    async def detect_product(self, image_url: str) -> DetectionResult:
        """
        Nano Banana 2 reasoning (бесплатно в рамках вызова):
        Prompt: "What product is in this image? Return: category, product_name, confidence"
        Используем встроенный reasoning Gemini — не нужен отдельный BLIP-2 вызов.
        """
```

**replicate_fallback.py — FALLBACK провайдер:**
```python
class ReplicateFallbackProvider:
    """
    Fallback если Nano Banana недоступен или для специфических задач.
    - Background removal: cjwbw/rembg (~$0.01)
    - Scene generation: black-forest-labs/flux-1.1-pro (~$0.05)
    """
    async def remove_background(self, image_url: str) -> RemovalResult: ...
    async def generate_scene(self, prompt: str, width: int, height: int) -> str: ...
```

**provider_registry.py:**
```python
class ProviderRegistry:
    """
    Factory + fallback. Порядок:
    1. Nano Banana 2 (primary — быстрый, дешёвый)
    2. Nano Banana Pro (premium — для текста/инфографики)
    3. Replicate (fallback — если Google API упал)

    Circuit breaker: если provider fails >50% за 5 min → переключить на fallback
    """
    def get_bg_removal(self) -> BackgroundRemovalProvider
    def get_scene_gen(self, premium: bool = False) -> SceneGenerationProvider
    def get_image_edit(self) -> ImageEditProvider
    def get_video_gen(self) -> VideoGenerationProvider  # всегда Grok
```

### Шаг 4.2 — Промпты по концепциям

Создай `backend/app/ai/prompts/scene_prompts.py`:

```python
# 3 основных концепции (как у Aidentika):
CONCEPTS = {
    "in_use": {
        # "В использовании" — товар в контексте применения
        "base_prompt": "Product being used in its natural context, lifestyle photography, "
                       "natural lighting, realistic setting, commercial quality, 8k",
        "negative": "text, watermark, logo, blurry, low quality, distorted, cartoon",
    },
    "in_scene": {
        # "В окружении" — реалистичная сцена
        "base_prompt": "Product placed in a beautiful realistic scene, professional product "
                       "photography, natural environment, premium commercial quality, 8k",
        "negative": "text, watermark, logo, blurry, low quality, distorted",
    },
    "catalog": {
        # "Каталог (студийно)" — чистый нейтральный фон
        "base_prompt": "Professional studio product photography, clean neutral background, "
                       "soft studio lighting, commercial catalog style, 8k, high detail",
        "negative": "text, watermark, logo, blurry, low quality, cluttered background",
    },
}

# Scene presets (подвыбор внутри каждой концепции)
SCENE_PRESETS = {
    "studio_white": "clean white background, soft shadows",
    "studio_gradient": "smooth gradient background from {color1} to {color2}",
    "lifestyle_interior": "modern minimalist interior, natural window light, wooden table, plants",
    "lifestyle_outdoor": "outdoor setting, golden hour lighting, bokeh background",
    "flat_lay": "top-down flat lay, marble surface, decorative props, Instagram aesthetic",
    "seasonal_spring": "spring flowers, cherry blossom, pastel colors, fresh atmosphere",
    "seasonal_winter": "winter holiday, snowflakes, pine branches, cozy atmosphere",
    "luxury": "dark marble surface, dramatic lighting, gold accents, premium feel",
    "tech_modern": "sleek dark surface, blue accent lighting, futuristic, minimal",
}

# Модификаторы для категорий товаров
CATEGORY_MODIFIERS = {
    "clothing": "fashion editorial style, fabric texture visible, ",
    "jewelry": "macro photography, sparkle and reflections, velvet surface, ",
    "electronics": "tech product photography, clean lines, ",
    "shoes": "shoe photography, showing design details, ",
    "food": "food photography, appetizing, fresh ingredients, ",
    "cosmetics": "beauty product photography, elegant, skincare aesthetic, ",
    "home": "interior design photography, cozy atmosphere, ",
    "pets": "pet product photography, warm friendly atmosphere, ",
    "tools": "industrial photography, workshop context, professional tools, ",
    "other": "professional product photography, ",
}
```

### Шаг 4.3 — Compositor (Pillow)

```python
class Compositor:
    async def composite(
        self,
        product_no_bg: bytes,    # PNG RGBA
        background: bytes,        # сгенерированный фон
        position: str = "center", # center, bottom-center
        scale: float = 0.7,
        shadow: bool = True,
    ) -> bytes:
        """
        1. Pillow: открыть product (RGBA) и background (RGB)
        2. Scale product
        3. Если shadow: Gaussian blur alpha, сдвиг вниз, opacity 30%
        4. Paste product на background
        5. Return JPEG bytes
        """
```

### Шаг 4.4 — Infographic Engine (preset-based, НЕ canvas)

```python
class InfographicEngine:
    """
    Preset-based инфографика. Юзер выбирает пресет и заполняет поля —
    движок рендерит через Pillow. Никакого Fabric.js.
    """

    async def render_preset(
        self,
        image: bytes,
        preset_id: str,
        data: dict,
    ) -> bytes:
        """
        data пример:
        {
            "title": "Платье в горошек",
            "price": "1 299 ₴",
            "old_price": "1 899 ₴",
            "badges": ["Хіт продажів", "-30%"],
            "features": [
                {"icon": "fabric", "text": "100% бавовна"},
                {"icon": "size", "text": "S-XL"},
                {"icon": "delivery", "text": "1-2 дні"},
            ],
        }

        1. Загрузить preset layout (JSON из БД)
        2. Render каждый slot: текст → Pillow ImageDraw + скачанные шрифты
        3. Render бейджи (цветные плашки с текстом)
        4. Render иконки + features
        5. Composite всё на изображение
        6. Return bytes
        """
```

### Шаг 4.5 — Job Orchestrator

```python
class JobOrchestrator:
    async def create_card_job(
        self,
        user_id: str,
        asset_id: str,
        concept: str,         # in_use, in_scene, catalog
        scene_preset: str,    # studio_white, lifestyle_interior, etc.
        user_prompt: str = "",
        infographic_preset_id: str = None,
        infographic_data: dict = None,
    ) -> Job:
        """
        1. Проверить token_balance >= cost (card=8, premium=10, infographic=12)
        2. Создать Job (status=PENDING)
        3. Атомарно списать токены (с idempotency_key)
        4. Enqueue Celery task
        5. Вернуть Job для polling
        """
```

### Шаг 4.6 — Celery Worker: Card Generation

```python
@celery_app.task(bind=True, max_retries=3, default_retry_delay=10)
def process_card_generation(self, job_id: str):
    """
    ДВА РЕЖИМА PIPELINE:

    === РЕЖИМ A: ATOMIC (PRIMARY — Nano Banana 2/Pro) ===
    Nano Banana генерирует финальное изображение за ОДИН вызов:
    product image + prompt → готовая карточка с фоном/сценой.
    Быстро (4-6 сек), дёшево ($0.08-0.13), один API call.

    1. Status → GENERATING_SCENE
       prompt = build_prompt(concept, scene_preset, category, user_prompt)
       nb_provider.generate_scene(product_image, prompt, width, height) → result

    2. Если есть infographic_preset_id:
       Status → ADDING_INFOGRAPHIC
       # Вариант A: Nano Banana Pro рендерит текст прямо в генерации
       # Вариант B: Pillow overlay поверх результата (для точного контроля layout)
       infographic_engine.render_preset(result, preset, data) → final

    3. Upload result в S3
    4. Создать Output запись
    5. Status → COMPLETED + processing_time_ms

    === РЕЖИМ B: STAGED (FALLBACK — Replicate или если NB не справился) ===
    Классический pipeline: bg removal → scene gen → composite.
    Медленнее (15-30 сек), но надёжнее для сложных случаев.

    1. Status → REMOVING_BG
       bg_provider.remove(asset.original_url) → save no_bg_url, mask_url

    2. Status → GENERATING_SCENE
       scene_provider.generate(prompt, negative, width, height) → background_url

    3. Status → COMPOSITING
       compositor.composite(no_bg_bytes, background_bytes) → composited

    4. Если есть infographic_preset_id:
       Status → ADDING_INFOGRAPHIC
       infographic_engine.render_preset(composited, preset, data) → final

    5. Upload → S3 → Output → COMPLETED

    === ВЫБОР РЕЖИМА ===
    provider_registry.get_pipeline_mode() определяет:
    - Если NB2/Pro доступен и circuit breaker OK → Режим A (atomic)
    - Если NB API упал / rate limit / circuit breaker open → Режим B (staged fallback)
    - Если user выбрал premium quality → Режим A с use_pro=True

    ON ERROR:
    - Режим A failed → автоматический retry через Режим B (staged)
    - Режим B failed → Status → FAILED + error_message
    - token_service.refund(user_id, job_id)
    - Если retries < max → self.retry()
    """
```

### Шаг 4.7 — API Routes

```
POST /api/jobs                   — создать job (card, card_infographic)
GET  /api/jobs/{id}              — статус job (для polling)
GET  /api/jobs/{id}/outputs      — результаты (output variants)
POST /api/jobs/{id}/retry        — повторить failed job

GET  /api/outputs                — все outputs юзера (gallery)
GET  /api/outputs/{id}/download  — скачать в оригинальном размере

GET  /api/presets                — список infographic presets
GET  /api/presets/{id}           — детали preset
```

**Коммит:** `git commit -m "Phase 4: Core pipeline — providers, compositor, infographics, Celery workers"`

---

## ФАЗА 5: ФРОНТЕНД — GUIDED GENERATOR (неделя 5-7)

> **Ключевой принцип:** guided generator, НЕ editor. Как Aidentika:
> Upload → Detect → Choose concept → Add wishes → Generate → Download.

### Шаг 5.1 — Dashboard Layout

```
/dashboard layout:
- Sidebar: Створити картку (primary), Банери, Відео, Галерея, Тарифи, Налаштування
- Header: лого "VisualCard by Uplify", token badge (🔥 420), user avatar dropdown
- Mobile: sidebar = Sheet (slide left)
- Стиль: белый фон, зелёный акцент #4ADE80, шрифт Inter
```

### Шаг 5.2 — Create Page (/dashboard/create)

```
Это ГЛАВНАЯ страница. Два столбца:

LEFT COLUMN — "Ваш товар" (01):
- Upload dropzone (если фото ещё нет)
- После загрузки: превью + "Ще" (добавить до 4 фото)
- "До 4 фото товару з різних сторін"
- Поле "Це:" — auto-detected product name (editable)
- Dropdown "Категорія:" — auto-detected (editable)
- ↓ Зелёная стрелка ↓
- "Настройте генерацію" (02):
  - Тип контенту: [Фото] [Карточка] [Відео] (tabs)
  - "Як показати товар?":
    - "В використанні" — контекст застосування
    - "В оточенні" — реалістична сцена
    - "Каталог (студійно)" — чистий об'єкт на нейтральному фоні
  - "Побажання" (textarea, max 2000 chars):
    - placeholder: "Наприклад: м'яке світло, мінімалізм, нейтральний фон"
    - Кнопка "✨ AI ідея" — автогенерация prompt-подсказки
  - "▸ Розширені налаштування" (expandable):
    - Scene preset selector (если нужен конкретный)
    - Для "Карточка": выбор infographic preset + заполнение полей
  - Кнопка "Згенерувати 🔥8 × 1 ▾"
    - 🔥8 = стоимость в токенах (10 для premium, 12 для инфографики)
    - × 1 = количество вариантов (dropdown: 1, 2, 4)

RIGHT COLUMN — "Результати" (03):
- Пока нет результатов: "Тут з'являться результати після генерації"
- Во время генерации: progress с шагами (Видаляємо фон → Генеруємо сцену → ...)
- После завершения: grid результатов
  - Клик → full-size
  - Кнопки: "⬇ Завантажити", "♡ В обране", "↻ Ще раз"
  - Dropdown "Завантажити для...": Оригінал, Rozetka, Prom.ua
```

### Шаг 5.3 — Zustand Store

```typescript
interface CreateStore {
  // Step state
  asset: Asset | null;
  detectedCategory: string | null;
  detectedName: string | null;
  userCategory: string | null;
  userName: string | null;

  // Generation settings
  contentType: "photo" | "card" | "video";
  concept: "in_use" | "in_scene" | "catalog";
  userPrompt: string;
  scenePreset: string | null;
  infographicPresetId: string | null;
  infographicData: Record<string, any> | null;
  variantCount: 1 | 2 | 4;

  // Job state
  currentJobId: string | null;
  jobStatus: JobStatus | null;
  outputs: Output[];

  // Actions
  uploadImage: (file: File) => Promise<void>;
  updateProductInfo: (category: string, name: string) => void;
  startGeneration: () => Promise<void>;
  pollJobStatus: () => Promise<void>;
  downloadOutput: (outputId: string, marketplace?: string) => void;
  reset: () => void;
}
```

### Шаг 5.4 — Gallery Page

```
/dashboard/gallery:
- Grid всех outputs юзера (masonry или regular grid)
- Фильтры: тип (фото/карточка/баннер/видео), дата
- Клик → modal с full-size + actions
- Multi-select → "Завантажити обрані" (ZIP)
```

**Коммит:** `git commit -m "Phase 5: Frontend — guided generator UI, gallery, status polling"`

---

## ФАЗА 6: БАННЕРЫ (неделя 7-8)

### Шаг 6.1 — Banner Engine

Создай `backend/app/ai/banner_engine.py`:

```python
BANNER_FORMATS = {
    # Маркетплейсы
    "rozetka_promo":      {"width": 1200, "height": 400, "name": "Rozetka промо-банер"},
    "prom_category":      {"width": 1920, "height": 600, "name": "Prom.ua категорія"},
    "amazon_brand_store": {"width": 3000, "height": 600, "name": "Amazon Brand Store"},

    # Соцсети
    "instagram_post":     {"width": 1080, "height": 1080, "name": "Instagram пост"},
    "instagram_story":    {"width": 1080, "height": 1920, "name": "Instagram сторіс"},
    "facebook_cover":     {"width": 1200, "height": 628, "name": "Facebook обкладинка"},
    "facebook_ad":        {"width": 1200, "height": 628, "name": "Facebook реклама"},

    # Google Ads
    "google_display_landscape": {"width": 1200, "height": 628, "name": "Google Display (landscape)"},
    "google_display_square":    {"width": 1200, "height": 1200, "name": "Google Display (square)"},
}

class BannerEngine:
    async def generate_banner(
        self,
        product_no_bg: bytes,
        format_key: str,
        text_data: dict,
        style: str = "modern",  # modern, minimal, bold, elegant
    ) -> bytes:
        """
        1. Сгенерировать фоновое изображение через AI (prompt по стилю + формату)
        2. Разместить product (no bg) по layout rules
        3. Добавить текст: headline, subheadline, CTA, price, badge
        4. Добавить лого/watermark если есть
        5. Return bytes в нужном формате/размере

        text_data пример:
        {
            "headline": "Весняний розпродаж -30%",
            "subheadline": "Нова колекція вже у продажу",
            "cta": "Купити зараз",
            "price": "від 999 ₴",
            "badge": "ХІТ",
        }
        """
```

### Шаг 6.2 — Banner API + Frontend

```
POST /api/jobs (type="banner")   — создать banner job
  Body: { asset_id, banner_format, text_data, style }

Frontend: /dashboard/banners
- Шаг 1: выбрать загруженное фото (из gallery assets)
- Шаг 2: выбрать формат (grid карточек с превью размеров)
- Шаг 3: заполнить текст (headline, subheadline, CTA, цена)
- Шаг 4: выбрать стиль (modern / minimal / bold / elegant)
- Кнопка "Згенерувати банер 🔥12"
- Результат: превью + download
```

**Коммит:** `git commit -m "Phase 6: Banner generation — marketplace + social media formats"`

---

## ФАЗА 7: AI ВИДЕО через xAI Grok Aurora (V1.1 — после запуска V1.0)

> **⚠️ V1.1 RELEASE:** Видео НЕ блокирует запуск V1.0.
> V1.0 = карточки + баннеры + биллинг + галерея + ops.
> Видео добавляется как отдельный релиз после стабилизации V1.0.
> На landing page видео показывается как "Скоро" / "Coming soon".

### Шаг 7.1 — Grok Video Provider

Создай `backend/app/providers/grok_video_gen.py`:

```python
class GrokVideoGeneration:
    """
    xAI Grok Imagine API — генерация видео из изображения.
    Endpoint: https://api.x.ai/v1/images/generations (для изображений)
    и /v1/video/generations (для видео)
    Стоимость: ~$0.03/image, ~$0.15/6sec video

    Документация: https://docs.x.ai/docs
    """

    async def generate_video(
        self,
        image_url: str,
        prompt: str,
        duration: int = 6,  # секунд
    ) -> str:
        """
        1. POST к xAI API с image + prompt
        2. Poll для получения результата (может занять 1-3 мин)
        3. Скачать видео, upload в S3
        4. Вернуть URL
        """

    async def generate_image(self, prompt: str, size: str = "1024x1024") -> str:
        """Опционально: генерация изображений через Grok Aurora (как альтернатива SDXL)"""
```

### Шаг 7.2 — Video Prompts

```python
VIDEO_SCENARIOS = {
    "zoom_in": {
        "name": "Плавний зум",
        "prompt": "Smooth slow camera zoom in on product, professional studio lighting, "
                  "product stays centered, subtle light changes, 6 seconds",
    },
    "rotation_360": {
        "name": "Обертання 360°",
        "prompt": "Product slowly rotating 360 degrees on turntable, clean background, "
                  "studio lighting revealing all sides, 6 seconds",
    },
    "cinematic_reveal": {
        "name": "Кінематографічний показ",
        "prompt": "Cinematic product reveal, dramatic lighting transition from dark to bright, "
                  "slow elegant camera movement, premium commercial feel, 6 seconds",
    },
    "lifestyle_context": {
        "name": "В контексті",
        "prompt": "Product in use in natural setting, gentle ambient movement, "
                  "lifestyle commercial feel, natural lighting, 6 seconds",
    },
}
```

### Шаг 7.3 — Video Celery Worker

```python
@celery_app.task(bind=True, max_retries=2, default_retry_delay=30)
def process_video_generation(self, job_id: str):
    """
    1. Получить result image из предыдущей генерации (или original)
    2. Собрать prompt из scenario + user_prompt
    3. Вызвать grok_video_provider.generate_video(image_url, prompt, duration=6)
    4. Poll статус (timeout 5 min)
    5. Download video → S3
    6. Create Output (video_url)
    7. Status → COMPLETED
    """
```

### Шаг 7.4 — Frontend: Video Page

```
/dashboard/video:
- Шаг 1: выбрать фото (из assets или из готовых outputs)
- Шаг 2: выбрать сценарій (4 карточки с превью)
- Шаг 3: (опционально) свій опис
- Кнопка "Створити відео 🔥60"
- Прогресс: "Генерація відео ~2 хвилини"
- Результат: video player (HTML5) + "⬇ Завантажити MP4"
```

**Коммит:** `git commit -m "Phase 7: AI Video — xAI Grok Aurora, video scenarios, player UI"`

---

## ФАЗА 8: БИЛЛИНГ И ПЛАТЕЖИ (неделя 9-10)

### Шаг 8.1 — Token Service (с idempotency)

> **Валюта сервиса — токены.** Более гранулярная система чем "кредиты".
> На UI всегда показывать эквивалент: "≈ 12 фото" / "або 2 відео".

```python
# === СПИСАНИЕ ТОКЕНОВ ===
OPERATION_COSTS = {
    # Основные операции
    JobType.CARD: 8,                  # стандартное фото (NB2)
    JobType.CARD_PREMIUM: 10,         # premium/lifestyle фото (NB Pro)
    JobType.CARD_INFOGRAPHIC: 12,     # фото + инфографика (NB Pro + Pillow)
    JobType.BANNER: 12,               # баннер
    JobType.VIDEO: 60,                # short video 6-8 сек (консервативно, пока не проверен xAI)

    # Дополнительные операции
    JobType.VARIATION: 5,             # regenerate / вариация существующего
    JobType.UPSCALE: 3,               # upscale до 2x
}
# Автоопределение товара — БЕСПЛАТНО (не тратит токены)

# === ONE-TIME ПАКЕТЫ (токены не сгорают) ===
PACKAGES = {
    "ua": {
        "start":    {"tokens": 100,   "price": 290,   "currency": "UAH"},  # ≈12 фото
        "creator":  {"tokens": 300,   "price": 690,   "currency": "UAH"},  # ≈37 фото
        "studio":   {"tokens": 800,   "price": 1490,  "currency": "UAH", "popular": True},  # ≈100 фото
        "business": {"tokens": 2200,  "price": 3490,  "currency": "UAH"},  # ≈275 фото
        "agency":   {"tokens": 6500,  "price": 8990,  "currency": "UAH"},  # ≈812 фото
    },
    "eu": {
        "start":    {"tokens": 100,   "price": 6.99,  "currency": "EUR"},
        "creator":  {"tokens": 300,   "price": 16.99, "currency": "EUR"},
        "studio":   {"tokens": 800,   "price": 36.99, "currency": "EUR", "popular": True},
        "business": {"tokens": 2200,  "price": 84.99, "currency": "EUR"},
        "agency":   {"tokens": 6500,  "price": 219.99,"currency": "EUR"},
    },
}

# === MONTHLY SUBSCRIPTIONS (для MRR — агентства, контент-команды) ===
SUBSCRIPTIONS = {
    "ua": {
        "studio_monthly":   {"tokens_per_month": 900,   "price": 1290,  "currency": "UAH"},
        "business_monthly": {"tokens_per_month": 2400,  "price": 2990,  "currency": "UAH"},
        "agency_monthly":   {"tokens_per_month": 6000,  "price": 6990,  "currency": "UAH"},
    },
    "eu": {
        "studio_monthly":   {"tokens_per_month": 900,   "price": 31.99, "currency": "EUR"},
        "business_monthly": {"tokens_per_month": 2400,  "price": 72.99, "currency": "EUR"},
        "agency_monthly":   {"tokens_per_month": 6000,  "price": 169.99,"currency": "EUR"},
    },
}
# Подписка: rollover до 50% неиспользованных токенов на следующий месяц.
# Top-up пакеты покупаются отдельно поверх подписки.
# При подписке per-token чуть выгоднее (~10%) чем разовый пакет.

class TokenService:
    async def spend(self, user_id, amount, job_id, idempotency_key) -> CreditTransaction:
        """
        Атомарное списание с SELECT FOR UPDATE на users.token_balance.
        idempotency_key = f"spend:{job_id}" — защита от двойного списания.
        """

    async def refund(self, user_id, job_id) -> CreditTransaction:
        """Возврат при failed job. idempotency_key = f"refund:{job_id}" """

    async def add(self, user_id, amount, payment_id, idempotency_key) -> CreditTransaction:
        """Начисление: покупка пакета, подписка, бонус, реферал."""

    async def check_subscription_renewal(self, user_id) -> None:
        """Ежемесячное начисление токенов по подписке + rollover логика."""
```

### Шаг 8.2 — WayForPay (Украина)

```python
class WayForPayService:
    """
    WayForPay — украинский платёжный провайдер.
    Документация: https://wiki.wayforpay.com/
    Поддерживает: Visa/MC, Apple Pay, Google Pay, ПриватБанк.
    """
    def create_payment(self, user_id, package_key, region="ua") -> dict:
        """
        WayForPay Purchase API:
        - merchantDomainName: visual.uplify.agency
        - returnUrl: {FRONTEND_URL}/billing?status=success
        - serviceUrl: {BACKEND_URL}/api/webhooks/wayforpay
        - Вернуть { payment_url, payment_id }
        """

    def verify_callback(self, data: dict, signature: str) -> dict:
        """Верификация HMAC_MD5 подписи + idempotency по orderReference"""
```

### Шаг 8.3 — Stripe (Европа)

```python
class StripeService:
    async def create_checkout_session(self, user_id, package_key) -> str:
        """Stripe Checkout → вернуть URL"""

    async def handle_webhook(self, payload, sig_header) -> None:
        """checkout.session.completed → начислить кредиты (idempotent)"""
```

### Шаг 8.4 — Webhook Routes (с idempotency)

```python
# Middleware: каждый webhook проверяет idempotency_key в БД
# Если уже обработан — return 200 OK без повторного начисления

POST /api/webhooks/wayforpay
POST /api/webhooks/stripe
```

### Шаг 8.5 — Frontend: Billing Page

```
/dashboard/billing (на украинском):

СЕКЦИЯ 1: Баланс
"У вас 🔥 420 токенів"
"Це приблизно 52 фото, 35 баннерів або 7 відео"

СЕКЦІЯ 2: Табы "Пакети" | "Підписки"

Таб "Пакети" — 5 карточек one-time:
- Start (100 / 290₴) — "≈12 фото"
- Creator (300 / 690₴) — "≈37 фото"
- Студія (800 / 1490₴) — "Популярний" badge — "≈100 фото"
- Бізнес (2200 / 3490₴) — "≈275 фото"
- Агенція (6500 / 8990₴) — "≈812 фото"

Таб "Підписки" — 3 monthly plans:
- Studio Monthly (900 токенів/міс / 1290₴) — "≈112 фото/міс"
- Business Monthly (2400 токенів/міс / 2990₴) — "≈300 фото/міс"
- Agency Monthly (6000 токенів/міс / 6990₴) — "≈750 фото/міс"
- Примітка: "Невикористані токени (до 50%) переносяться на наступний місяць"

Кнопка "Купити" / "Підписатися" → detect region → WayForPay (UA) / Stripe (EU)

СЕКЦІЯ 3: Розцінки
Таблица: операція, токени, приблизна вартість
- Стандартне фото: 8 токенів
- Premium фото: 10 токенів
- Фото + інфографіка: 12 токенів
- Баннер: 12 токенів
- Відео (6-8 сек): 60 токенів
- Варіація: 5 токенів
- Upscale: 3 токени

СЕКЦІЯ 4: Історія транзакцій
Таблица (дата, тип, операція, токени, баланс після)
```

**Коммит:** `git commit -m "Phase 8: Billing — tokens with idempotency, WayForPay, Stripe, subscriptions"`

---

## ФАЗА 9: ЭКСПОРТ + MARKETPLACE PRESETS (неделя 10-11)

### Шаг 9.1 — Export Service

```python
MARKETPLACE_SPECS = {
    "rozetka": {"main": {"w": 1000, "h": 1000}, "gallery": {"w": 1000, "h": 1500}},
    "prom_ua": {"main": {"w": 1200, "h": 1200}, "gallery": {"w": 1200, "h": 1600}},
    "amazon":  {"main": {"w": 2000, "h": 2000}, "gallery": {"w": 2000, "h": 2000}},
    "generic": {"main": {"w": 1500, "h": 1500}, "gallery": {"w": 1500, "h": 2000}},
}

class ExportService:
    async def export_for_marketplace(self, output_id: str, marketplace: str, type: str) -> bytes:
        """Resize + optimize + correct format. Бесплатно (не тратит кредиты)."""

    async def export_batch(self, output_ids: list[str], marketplace: str) -> bytes:
        """ZIP с несколькими изображениями"""
```

### Шаг 9.2 — Frontend: Download Dropdown

```
В результатах генерации и в Gallery — dropdown "Завантажити для...":
- Оригінал (full size)
- Rozetka (1000×1000)
- Prom.ua (1200×1200)
- Amazon (2000×2000)
- Інше (generic 1500×1500)
```

**Коммит:** `git commit -m "Phase 9: Marketplace export — Rozetka, Prom, Amazon presets"`

---

## ФАЗА 10: LANDING PAGE (неделя 11)

### Шаг 10.1 — Landing

```
frontend/src/app/page.tsx — главная страница visual.uplify.agency:

1. HERO
   - "Створіть продаючу картку товару за 60 секунд"
   - "AI-сервіс для продавців на маркетплейсах"
   - CTA: "Спробувати безкоштовно →" (зелёная кнопка)
   - Бегущая строка: AI ГЕНЕРАЦІЯ ★ ЛЕГКО РОЗІБРАТИСЯ ★ БЕЗ СТУДІЇ ★ В ПАРУ КЛІКІВ
   - Лайв-статистика: "Середній час генерації: 56 секунд"

2. ЯК ЦЕ ПРАЦЮЄ
   - "3 кроки — і готово"
   - 01: Завантажте фото (AI визначить тип товару)
   - 02: Оберіть концепцію + побажання
   - 03: Отримайте готову картку + інфографіку

3. МОЖЛИВОСТІ
   - Картки товарів (до/після)
   - Банери для маркетплейсів та соцмереж
   - AI Відео (превью)

4. ПРИКЛАДИ ПО КАТЕГОРІЯХ
   - Скроллер: Одяг, Взуття, Ювелірка, Електроніка, Дім, Зоотовари

5. ПОРІВНЯННЯ
   - "Традиційна зйомка" vs "VisualCard"
   - Час: 3-7 днів vs 5 хвилин
   - Вартість: від 2000₴ vs від 249₴

6. ТАРИФИ (4 карточки)

7. FAQ (accordion, 8-10 вопросов)

8. FOOTER: CTA + "by Uplify" + Telegram підтримка + legal links

Стиль: белый минимализм, зелёный #4ADE80, чёрные кнопки, Inter font.
Анимации: framer-motion (fade-in при скролле).
Язык: украинский.
```

**Коммит:** `git commit -m "Phase 10: Landing page — hero, examples, pricing, FAQ"`

---

## ФАЗА 7B: ОПЕРАЦИОННЫЙ СЛОЙ (параллельно с Фазой 7-8, неделя 8-9)

> **КРИТИЧНО:** Этот слой реализуется ПАРАЛЛЕЛЬНО с биллингом, а НЕ в конце.
> Idempotency нужен ДО первого webhook. Rate limiting нужен ДО продакшена.
> Provider retry/fallback нужен ДО первых реальных юзеров.

### Шаг 7B.1 — Rate Limiting

```python
# backend/app/middleware/rate_limiter.py
# Redis-based per-user rate limiting:
# - API: 100 req/min per user
# - Generations: 10 concurrent jobs per user
# - Uploads: 50/hour per user
```

### Шаг 7B.2 — Idempotency Middleware

```python
# backend/app/middleware/idempotency.py
# Для webhook handlers и payment callbacks:
# - Проверить idempotency_key в Redis/DB
# - Если уже обработан → return cached response
# - Иначе → обработать + сохранить ключ
```

### Шаг 7B.3 — Provider Retry + Fallback

```python
# backend/app/providers/provider_registry.py
# Retry policy: 3 attempts с exponential backoff
# Fallback: если primary provider 5xx → попробовать fallback provider
# Circuit breaker: если provider fails >50% за 5 min → временно отключить
```

### Шаг 7B.4 — Content Moderation

```python
# backend/app/middleware/moderation.py
# Базовая проверка:
# - NSFW detection на загруженных фото (через Replicate NSFW classifier)
# - Промпт-валидация: блокировать offensive/harmful prompts
```

### Шаг 7B.5 — Admin/Support Tools

```
Простая admin-страница (protected route, is_admin=True):
- Список юзеров (email, credits, registrations)
- Список jobs (status, duration, provider, cost)
- Мануальное начисление кредитов
- Статистика: генерации/день, revenue, provider costs
```

**Коммит:** `git commit -m "Phase 7B: Operational layer — rate limits, idempotency, moderation, admin"`

---

## ФАЗА 11: i18n + ТЕСТИРОВАНИЕ + PRODUCTION (неделя 10-11)

### Шаг 11.1 — i18n (UK + EN)

```
next-intl для фронтенда:
- messages/uk.json (украинский — default)
- messages/en.json (английский)
- Переключатель языка в header
- URL: /uk/dashboard, /en/dashboard
- Backend: ошибки API на языке юзера (Accept-Language)
```

### Шаг 11.2 — Backend Tests

```
pytest + httpx AsyncClient:
- test_auth.py: register, login, google_auth, refresh, protected routes
- test_jobs.py: create job, poll status, mock providers, retry on failure
- test_billing.py: credit spend/refund, idempotency, webhook verification
- test_providers.py: mock Replicate/Grok responses, fallback logic
- test_export.py: correct dimensions per marketplace
```

### Шаг 11.3 — Frontend Tests

```
Vitest + React Testing Library:
- Upload component
- Create flow (upload → detect → concept → generate)
- Billing page
```

### Шаг 11.4 — Docker Production

```
docker-compose.prod.yml:
- frontend: multi-stage build → nginx
- backend: gunicorn + uvicorn workers (4)
- celery: 2 workers (default, ai)
- postgres: volume + backup script
- redis: persistence
- nginx: reverse proxy, SSL, rate limiting
```

### Шаг 11.5 — CI/CD

```
.github/workflows/ci.yml:
1. lint (eslint + ruff)
2. test-backend (pytest)
3. test-frontend (vitest)
4. build docker images
5. deploy (ssh + docker-compose pull + up)
```

**Коммит:** `git commit -m "Phase 11: i18n, tests, Docker production, CI/CD"`

---

## СТОИМОСТЬ ОПЕРАЦИЙ И ЮНИТ-ЭКОНОМИКА

| Операция | Provider | Cost (USD) | Токенов | Цена юзеру (Start) | Цена юзеру (Agency) | Маржа |
|----------|----------|-----------|---------|--------------------|--------------------|-------|
| Стандартное фото | NB2 (atomic) | ~$0.08-0.16 | 8 | 23.2₴ | 11.1₴ | ~70-85% |
| Premium фото | NB Pro (atomic) | ~$0.13-0.21 | 10 | 29.0₴ | 13.8₴ | ~65-80% |
| Фото + инфографика | NB Pro + Pillow | ~$0.13-0.25 | 12 | 34.8₴ | 16.6₴ | ~65-80% |
| Баннер | NB2 + Pillow | ~$0.08-0.16 | 12 | 34.8₴ | 16.6₴ | ~70-85% |
| Video (6-8 сек) | xAI Grok Aurora | ~$0.15-0.80 | 60 | 174₴ | 83₴ | ~50-90%* |
| Вариация | NB2 | ~$0.08 | 5 | 14.5₴ | 6.9₴ | ~75-85% |
| Upscale | CPU (Pillow/Real-ESRGAN) | ~$0.01-0.03 | 3 | 8.7₴ | 4.1₴ | ~90%+ |

*Маржа видео сильно зависит от реального провайдера. $0.15 — оптимистичный xAI, $0.80 — Veo 2 уровень.

**Эффективная цена за токен по пакетам:**

| Пакет | Цена | Токены | ₴/токен | ₴/фото (8т) | ₴/баннер (12т) | ₴/видео (60т) |
|-------|------|--------|---------|-------------|---------------|--------------|
| Start | 290₴ | 100 | 2.90 | 23.2 | 34.8 | 174.0 |
| Creator | 690₴ | 300 | 2.30 | 18.4 | 27.6 | 138.0 |
| Studio | 1490₴ | 800 | 1.86 | 14.9 | 22.4 | 111.8 |
| Business | 3490₴ | 2200 | 1.59 | 12.7 | 19.0 | 95.2 |
| Agency | 8990₴ | 6500 | 1.38 | 11.1 | 16.6 | 83.1 |

**Ключевые метрики:**
- От Start к Agency цена за токен падает в ~2.1 раза (2.90 → 1.38)
- Strongest anchor: Studio (1490₴) — "Популярний" badge
- Welcome bonus: 20 токенов (≈2 фото) — достаточно для пробы, мотивирует купить пакет
- Image operations самые маржинальные (~70-85%), видео — зависит от провайдера

---

## ROADMAP

### V1.0 — Запуск (карточки + баннеры + биллинг)

| Фаза | Что | Недели | Статус |
|------|-----|--------|--------|
| 0 | Init: monorepo, Docker, Next.js, FastAPI | 1 | |
| 1 | DB: normalized schema (projects, assets, jobs, outputs, job_assets M2M) | 1-2 | |
| 2 | Auth: единый JWT, Google OAuth | 2 | |
| 3 | Upload + auto-detect product type | 3 | |
| 4 | Core: AI pipeline (NB atomic primary + staged fallback) | 3-5 | |
| 5 | Frontend: guided generator UI | 5-7 | |
| 6 | Banners: marketplace + social media | 7-8 | |
| 7B | Ops: rate limits, idempotency, provider retry, moderation, admin | 8-9 | |
| 8 | Billing: WayForPay + Stripe + tokens + subscriptions | 8-9 | |
| 9 | Export: marketplace presets (Rozetka, Prom, Amazon) | 9-10 | |
| 10 | Landing page | 10 | |
| 11 | i18n (UK+EN), tests, Docker prod, CI/CD | 10-11 | |

### V1.1 — Видео (после стабилизации V1.0)

| Фаза | Что | Недели | Статус |
|------|-----|--------|--------|
| 7 | Video: xAI Grok Aurora | +2-3 после V1.0 | |

---

## КАК РАБОТАТЬ С ЭТИМ ФАЙЛОМ

1. Положи `CLAUDE.md` в корень проекта `visualcard/`
2. Открой терминал: `cd visualcard`
3. Запусти Claude Code: `claude`
4. Скажи: **"Прочитай CLAUDE.md и начни с Фазы 0"**
5. Claude Code будет создавать файлы, настраивать проект, писать код
6. После каждой фазы проверяй: `docker-compose up -d && docker-compose logs -f`
7. Коммить: Claude сделает это автоматически
8. Для продолжения: **"Продолжи с Фазы X"**
9. При ошибке: **"Мы на Фазе X, шаг Y. Вот ошибка: ..."**
10. Для конкретного модуля: **"Реализуй только Шаг 4.3 — Compositor"**
