# EchoVault

> Write your why. Seal it. Let it find you every day.

EchoVault is a commitment-and-reflection web app. You write **what** you want to achieve and **why** it matters to you. The *what* stays visible but frozen. The *why* is sealed until the capsule ends. While the capsule runs, your *why* reaches you only through one AI-generated question each day, delivered as a notification. When the capsule ends, the *why* is revealed and you review how it went.

---

## Overview

Most goal trackers show you a checklist. They rarely remind you *why* you started, and that reason is usually what keeps people going.

A traditional time capsule has the opposite problem: it hides everything until the end, so it does nothing for you along the way.

EchoVault keeps a goal in front of you while protecting the reason behind it:

- **What**: visible for the whole capsule, so you always know what you committed to.
- **Why**: sealed. You cannot reread it, so you cannot edit it or wear it out by rereading it every day.
- **A daily question**: generated from your *why*, so the reason returns to you every day in a new form without being quoted.
- **A review**: at the end, you read your original *why* and mark the outcome.

---

## MVP user flow

1. Register or log in.
2. Create a capsule: enter a title and choose a duration preset (**14, 30, 60, or 90 days**).
3. Write **what** you want to do. This stays visible, and is frozen after activation.
4. Before writing the *why*, a notice is shown: *"Your 'why' will be sent to an AI to create your daily questions."*
5. Write **why**. This is sealed until the end.
6. Give or decline **AI consent**. If you decline, a built-in bank of generic questions is used.
7. Choose a **tone**: gentle, direct, or tough love.
8. Choose a **daily reminder time**, then grant browser notification permission. iPhone users see an "Add to Home Screen" guide.
9. **Activate.** A Celery task generates one question per day of the capsule in a single AI call and stores them. If the AI call fails, the generic bank is used instead.
10. You receive **one notification per day** at your chosen local time. In the last 10 days, the notification includes a countdown, for example *"8 days left: …"*.
11. **At the end**, the *why* is revealed, you mark the outcome (achieved / partial / missed), and you are prompted to start your next capsule.

---

## Capsule lifecycle

### Stored status

Only three statuses are saved in the database:

```text
DRAFT  ──activate──▶  ACTIVE  ──review──▶  COMPLETED
```

| Status | Meaning |
|---|---|
| `DRAFT` | Being written. Editable and deletable. |
| `ACTIVE` | Activated. Content is frozen; daily questions are delivered. |
| `COMPLETED` | The capsule has ended and the user has submitted a review. |

### Derived states

These are calculated from the current time and are never stored:

| Derived state | Condition |
|---|---|
| **running** | `status == ACTIVE` and `now < ends_at` |
| **ended** | `now >= ends_at` (the *why* can be revealed and the capsule reviewed) |
| **countdown phase** | running and `days_left <= 10` |

**Why derive them?** If "ended" were a stored status, a scheduled job would have to switch it at exactly the right moment, and any delay or failure would leave the data wrong. A value computed from `ends_at` and the current time is always correct.

---

## Rules

1. Only the owner can access a capsule. Other users get **404** (not 403), so they cannot tell whether the capsule exists.
2. Only `DRAFT` capsules can be edited or deleted. After activation, title, what, why, duration, and tone are frozen; only the reminder time may change.
3. Activation requires: title, what, why, duration preset, tone, an explicit AI consent choice, a reminder time, and **no other ACTIVE capsule** for that user. This is enforced by a database constraint, not only in code.
4. The API never returns the *why* while `now < ends_at`, not even to the owner.
5. All times are stored in UTC. The end time and reminder time are interpreted in the user's timezone, which is stored on the user profile.
6. Review is only allowed after the capsule has ended.
7. At most one notification per user per day. Notifications never quote the *why* word for word.
8. The *why* is sent to the AI only with consent and is never logged. API keys and VAPID keys live in environment variables.
9. If the user has no push subscription, the daily question is sent by email instead.
10. Each daily question has an empty answer field, reserved for the future "respond to each question" feature.

---

## Notifications

### How Web Push works

1. The React app registers a **service worker**, a small script the browser keeps running in the background.
2. After the user grants permission, the browser creates a **push subscription**: an endpoint URL on the browser vendor's push service, plus encryption keys. The app sends it to the backend, which stores it as a `PushSubscription`.
3. At the user's reminder time, a Celery Beat schedule triggers a task that sends the day's question to that endpoint using **pywebpush**. The message is encrypted and signed with the server's **VAPID** key pair, which proves the message really comes from EchoVault.
4. The push service delivers the message to the device, and the service worker displays the notification, even if the app is not open.

If the user has no valid subscription, or a push fails permanently (for example, the subscription has expired), the question is sent **by email** instead.

### iPhone limitation

On iOS, Web Push works only in **iOS 16.4 or later**, and only after the user adds EchoVault to the home screen (Share → *Add to Home Screen*) and opens it from there. In normal Safari tabs it does not work. The app detects iPhone users and shows a short guide. Laptops and Android support Web Push directly in the browser.

---

## AI and privacy

- Before the user writes the *why*, the app states clearly that it will be sent to an AI.
- AI consent is an explicit choice. If it is declined, the *why* never leaves the server, and questions come from a built-in generic bank.
- With consent, all questions are generated **once, on activation**, in a single AI call (one question per day), using the chosen tone. They are stored as `DailyQuestion` rows.
- If the AI call fails, the generic bank is used, so activation never depends on the AI provider being available.
- Generated questions are reflective prompts, and the *why* is never quoted word for word in them or in notifications.
- The *why* is **never logged**: not in application logs, Celery task arguments, or error reports. Tasks receive the capsule ID and load the text from the database themselves.
- The LLM API key and VAPID keys are read from environment variables and are never committed.

---

## Tech stack

| Area | Tools |
|---|---|
| Language | Python 3.13 |
| Backend | Django, Django REST Framework |
| Database | PostgreSQL |
| Authentication | Simple JWT |
| Background jobs | Celery, Celery Beat, Redis |
| AI | An LLM API for question generation |
| Notifications | Web Push via pywebpush (VAPID keys), email fallback |
| API docs | Swagger / OpenAPI |
| Testing | pytest, pytest-django |
| Code quality | Ruff, pre-commit |
| DevOps | Docker Compose, GitHub Actions |
| Frontend | React (Vite) as a basic PWA (manifest + service worker) |

---

## Architecture

EchoVault is split into small Django apps, each with a single responsibility.

```text
echovault/
├── config/
│   ├── settings/
│   ├── celery.py
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
│
├── apps/
│   ├── accounts/        # Custom User (with timezone), registration, JWT, profile
│   ├── capsules/        # Capsule model, drafts, activation, reveal, review
│   ├── questions/       # DailyQuestion, AI generation task, generic question bank
│   └── notifications/   # PushSubscription, scheduled delivery, email fallback
│
├── frontend/            # React (Vite) PWA
├── tests/
├── requirements.txt
├── manage.py
├── Dockerfile
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## Core models

### User

A custom user model, so fields can be added later without a painful migration.

| Field | Notes |
|---|---|
| email / username, password | Standard authentication fields |
| `timezone` | IANA name (e.g. `Asia/Kolkata`), validated, defaults to `UTC` |

### Capsule

| Field | Notes |
|---|---|
| `owner` | FK to User |
| `title`, `what`, `why` | Text. `why` is never returned before the capsule ends |
| `duration_days` | Choices: 14 / 30 / 60 / 90 |
| `status` | `draft` / `active` / `completed` |
| `tone` | `gentle` / `direct` / `tough_love` |
| `ai_consent` | Nullable boolean: `null` until the user chooses |
| `reminder_time` | Local time of day, in the owner's timezone |
| `starts_at`, `ends_at` | UTC datetimes, set on activation |
| `outcome` | `achieved` / `partial` / `missed`, nullable until review |
| `created_at`, `updated_at` | Timestamps |

Computed properties: `has_ended`, `days_left`, `in_countdown` (10 days or fewer left).
A partial unique constraint allows only **one `active` capsule per user**.

### DailyQuestion

| Field | Notes |
|---|---|
| `capsule` | FK to Capsule |
| `day_number` | 1 … `duration_days`, unique per capsule |
| `text` | The question |
| `source` | `ai` or `generic` |
| `sent_at` | When it was delivered (nullable); prevents duplicate sends |
| `answer` | Empty, reserved for the future "respond to each question" feature |

### PushSubscription

| Field | Notes |
|---|---|
| `user` | FK to User |
| `endpoint` | Push service URL (unique) |
| `p256dh`, `auth` | Encryption keys from the browser |
| `created_at` | Timestamp |

---

## Planned API endpoints

> These endpoints are **subject to change** during implementation. Swagger docs will be available at `/api/docs/`.

### Authentication and profile

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/auth/register/` | Register a user |
| POST | `/api/auth/login/` | Obtain JWT access and refresh tokens |
| POST | `/api/auth/refresh/` | Refresh the access token |
| GET | `/api/auth/profile/` | Retrieve profile (including timezone) |
| PATCH | `/api/auth/profile/` | Update profile (including timezone) |

### Capsules

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/capsules/` | List own capsules (title, what, status, days left) |
| POST | `/api/capsules/` | Create a draft capsule |
| GET | `/api/capsules/{id}/` | Retrieve a capsule (`why` only after it ends) |
| PATCH | `/api/capsules/{id}/` | Update a draft, or the reminder time of an active capsule |
| DELETE | `/api/capsules/{id}/` | Delete a draft |
| POST | `/api/capsules/{id}/activate/` | Validate and activate a draft |
| GET | `/api/capsules/{id}/today/` | Today's question and countdown |
| POST | `/api/capsules/{id}/review/` | Submit the outcome after the capsule ends |

### Notifications

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/notifications/vapid-public-key/` | Public VAPID key for the browser |
| POST | `/api/notifications/subscriptions/` | Save a push subscription |
| DELETE | `/api/notifications/subscriptions/` | Remove a push subscription |

---

## Getting started

### Prerequisites

- Python 3.13+
- PostgreSQL
- Redis
- Git

### 1. Clone the repository

```bash
git clone https://github.com/Sup-ri-tha/EchoVault.git

cd EchoVault
```

### 2. Create a virtual environment

```bash
python -m venv .venv
```

Activate it.

Linux / macOS:

```bash
source .venv/bin/activate
```

Windows:

```bash
.venv\Scripts\activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure environment variables

Create a `.env` file using the provided example:

```bash
cp .env.example .env
```

Example configuration (placeholder values only):

```env
SECRET_KEY=your-secret-key
DEBUG=True

DATABASE_URL=postgresql://postgres:password@localhost:5432/echovault

REDIS_URL=redis://localhost:6379/0

EMAIL_HOST=smtp.example.com
EMAIL_PORT=587
EMAIL_HOST_USER=your-email
EMAIL_HOST_PASSWORD=your-password

# Web Push (generate your own key pair; never commit real keys)
VAPID_PUBLIC_KEY=your-vapid-public-key
VAPID_PRIVATE_KEY=your-vapid-private-key
VAPID_CLAIM_EMAIL=mailto:you@example.com

# AI question generation
LLM_API_KEY=your-llm-api-key
LLM_MODEL=your-model-name
```

Do not commit `.env` or production credentials to version control.

### 5. Run database migrations

```bash
python manage.py migrate
```

### 6. Create a superuser

```bash
python manage.py createsuperuser
```

### 7. Start the development server

```bash
python manage.py runserver
```

The API will be available at:

```text
http://127.0.0.1:8000/
```

### 8. Start the Celery worker and Celery Beat

In two separate terminals:

```bash
celery -A config worker -l info
```

```bash
celery -A config beat -l info
```

The worker runs tasks such as question generation. Beat is the scheduler that triggers daily notification delivery.

---

## Testing

Run the test suite:

```bash
pytest
```

Run the linter:

```bash
ruff check .
ruff format --check .
```

The test suite covers:

- Registration, login, and timezone validation
- Ownership (other users get 404)
- Draft editing rules and frozen fields after activation
- Activation validation and the one-active-capsule database constraint
- The *why* being hidden until the capsule ends
- `has_ended`, `days_left`, and `in_countdown`, tested with **frozen time** so results don't depend on when the tests run
- Question generation, including the fallback to the generic bank (the AI is mocked)
- Delivery: one notification per day, countdown text, email fallback
- Review only after the end

---

## Development workflow

- **Branch per issue**, named `type/issue-number-short-name`, for example `feat/11-capsule-model` or `docs/15-readme-redesign`.
- **Conventional Commits**: `feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `chore:`, `ci:`. For example: `feat(capsules): add activation endpoint`.
- **Pull requests** include `Closes #N` so the issue closes automatically when merged.
- **Squash merge** into `main`, so each PR becomes one clean commit.
- pre-commit runs Ruff before every commit. GitHub Actions runs lint and tests on every PR.

---

## Definition of Done

An issue is done when:

- [ ] Acceptance criteria are met
- [ ] Tests are written and passing
- [ ] Ruff is clean
- [ ] CI is green
- [ ] Docs are updated if behaviour changed
- [ ] It is merged via a PR
- [ ] I can explain every line in the PR

---

## Roadmap

### Sprint 1 (Nov 5–11): Foundations

- [ ] Repository, CI, pre-commit
- [ ] Authentication with user timezone
- [ ] Capsule model
- [ ] Create, list, and edit drafts
- [ ] Activation

### Sprint 2 (Nov 12–18): Questions and delivery

- [ ] AI question generation (Celery) with generic fallback
- [ ] Daily question and countdown logic
- [ ] Web Push subscriptions and VAPID
- [ ] Scheduled delivery with email fallback
- [ ] End-of-capsule reveal and review
- [ ] Swagger docs

### Sprint 3 (Nov 19–25): Frontend and release

- [ ] React PWA: landing, register, login, home, create flow, today's question card, review screen
- [ ] Notification permission flow
- [ ] iPhone "Add to Home Screen" guide
- [ ] Full Docker Compose setup
- [ ] Deployment
- [ ] Demo account with seeded data
- [ ] Release v1.0.0

---

## Stretch goals (not in MVP)

- Responding to daily questions, and an AI end-of-capsule summary (the data model is already prepared)
- Adaptive questions based on progress
- Multiple active capsules
- Email verification and password reset
- Mobile app with a home-screen widget (Expo)
- Archive and search

---

## Security considerations

- All capsule endpoints require authentication. Querysets are filtered by owner, so other users get 404.
- The sealed *why* is enforced in the serializer/API layer based on server time, never only in the frontend.
- The *why* is never written to logs, Celery task arguments, error reports, or notifications.
- The *why* is sent to the AI provider only with explicit consent.
- Secrets (Django secret key, database credentials, LLM API key, VAPID private key, email password) are kept in environment variables.
- The one-active-capsule rule is enforced by a database constraint, and activation runs in a transaction to avoid race conditions.
- Push subscriptions belong to a single user and are removed when the push service reports they have expired.
- HTTPS is required in production (Web Push and service workers only work over HTTPS).

Sealing is an application-level access rule. It is **not** end-to-end encryption, and it does not protect against someone with direct database access.

---

## Learning objectives

EchoVault is a learning project for practising Python, Django, and professional engineering habits through a real application:

- Django project structure and app boundaries
- Custom user models and JWT authentication
- DRF serializers, viewsets, permissions, and object-level access
- Model design, choices, computed properties, and migrations
- Database constraints vs application checks, and transactions
- Timezones: storing UTC and interpreting local time
- Background jobs and scheduling with Celery, Celery Beat, and Redis
- Calling an external AI API safely, with consent, fallbacks, and no logging of private data
- Web Push, service workers, and PWAs
- Testing with pytest, mocks, and frozen time
- Ruff, pre-commit, CI with GitHub Actions, Docker Compose
- Working from issues, branches, PRs, and ADRs

---

## License

This project is licensed under the MIT License.
