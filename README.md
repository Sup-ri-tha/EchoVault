# EchoVault

> Some things are meant to be opened later.

EchoVault is a digital time capsule platform that allows users to preserve memories, letters, photos, and personal reflections for their future selves or the people they care about.

Create a capsule today. Seal it. Choose when it unlocks. Revisit your memories when the time comes.

EchoVault transforms ordinary digital memories into meaningful experiences by connecting the present with the future.

---

## Overview

People capture countless memories through photos, notes, messages, and journals, but these memories often become scattered across different platforms.

EchoVault provides a dedicated space to preserve these moments and revisit them at a meaningful point in the future.

Users can create personal capsules, write letters to their future selves, share capsules with friends, and schedule messages for important dates.

Unlike a conventional notes application, EchoVault introduces time-based access control: a sealed capsule remains inaccessible until its scheduled unlock time.

## Features

### Authentication and User Profiles

- User registration and login
- Secure password hashing
- Email verification
- Password reset
- Profile management
- User-specific capsule dashboard

### Digital Time Capsules

- Create and manage capsules
- Add titles, descriptions, and cover images
- Choose a capsule unlock date
- Save capsules as drafts
- Seal capsules for future access
- View capsule countdowns
- Automatically unlock capsules when eligible

### Memory Management

- Add multiple memories to a capsule
- Write letters and journal entries
- Upload photographs and attachments
- Add captions and descriptions
- Organize memories chronologically
- Preview memories before sealing

### Future Self Mode

Create letters and reflections for your future self.

Answer prompts about your current goals, experiences, and expectations. When the capsule unlocks, revisit your original thoughts and reflect on how your life has changed.

### Shared Capsules

- Invite other users to contribute
- Create collaborative time capsules
- Add memories from multiple contributors
- Manage capsule participants
- Seal shared capsules for a future date

### Future Delivery

Create capsules for friends and family.

- Select a recipient
- Write a personal message
- Schedule delivery
- Notify recipients when capsules unlock
- Track whether a recipient has opened a capsule

### Dashboard and Archive

- View all created capsules
- Track locked and unlocked capsules
- Search capsules
- Filter by status
- Sort by unlock date
- View upcoming unlocks
- Archive opened capsules

### Notifications

- Capsule unlock notifications
- Upcoming unlock reminders
- Invitation notifications
- Email delivery status

---

## Technology Stack

### Backend

- Python
- Django
- Django REST Framework
- PostgreSQL
- Django ORM

### Authentication

- Django Authentication
- JWT authentication using Simple JWT

### Background Processing

- Celery
- Redis

### Testing and Code Quality

- pytest
- pytest-django
- Ruff

### Development and Deployment

- Git
- GitHub
- Docker
- Docker Compose
- Environment variables

### Frontend

The frontend can be developed using React and connected to the Django REST API.

---

## Architecture

EchoVault follows a modular Django application structure.

```text
echovault/
├── config/
│   ├── settings/
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
│
├── apps/
│   ├── accounts/
│   ├── capsules/
│   ├── memories/
│   ├── invitations/
│   └── notifications/
│
├── tests/
├── requirements.txt
├── manage.py
├── Dockerfile
├── docker-compose.yml
├── .env.example
└── README.md
```

### Core Models

| Model | Responsibility |
|---|---|
| User | Account and authentication |
| Capsule | Capsule metadata and unlock schedule |
| Memory | Text, photos, and attachments |
| CapsuleParticipant | Shared capsule membership |
| Invitation | Capsule invitations |
| CapsuleEvent | Capsule activity history |
| Notification | Notification delivery tracking |

---

## Capsule Lifecycle

A capsule follows this lifecycle:

```text
DRAFT
  |
  v
SEALED
  |
  v
LOCKED
  |
  v
UNLOCKED
  |
  v
ARCHIVED
```

### Capsule Access Rules

1. A capsule can be edited while it is in draft state.
2. A sealed capsule cannot be modified.
3. A capsule remains locked until its unlock time.
4. The backend validates access using the current server time.
5. Only authorized users can access capsule contents.
6. Once unlocked, the capsule can be opened and archived.

The frontend countdown is informational. The backend is responsible for enforcing the unlock condition.

---

## API Overview

The backend exposes a REST API.

### Authentication

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/auth/register/` | Register a user |
| POST | `/api/auth/login/` | Authenticate a user |
| POST | `/api/auth/refresh/` | Refresh access token |
| POST | `/api/auth/logout/` | Log out |
| GET | `/api/auth/profile/` | Retrieve profile |
| PATCH | `/api/auth/profile/` | Update profile |

### Capsules

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/capsules/` | List accessible capsules |
| POST | `/api/capsules/` | Create a capsule |
| GET | `/api/capsules/{id}/` | Retrieve capsule |
| PATCH | `/api/capsules/{id}/` | Update a draft capsule |
| DELETE | `/api/capsules/{id}/` | Delete an eligible capsule |
| POST | `/api/capsules/{id}/seal/` | Seal a capsule |
| GET | `/api/capsules/{id}/memories/` | List capsule memories |

### Memories

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/memories/` | Add a memory |
| GET | `/api/memories/{id}/` | Retrieve a memory |
| PATCH | `/api/memories/{id}/` | Update an eligible memory |
| DELETE | `/api/memories/{id}/` | Delete an eligible memory |

### Invitations

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/invitations/` | Create an invitation |
| GET | `/api/invitations/` | List invitations |
| POST | `/api/invitations/{id}/accept/` | Accept invitation |
| POST | `/api/invitations/{id}/reject/` | Reject invitation |

### Notifications

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/notifications/` | List notifications |
| POST | `/api/notifications/{id}/read/` | Mark as read |

These endpoints describe the planned API and may change during implementation.

---

## Getting Started

### Prerequisites

- Python 3.13+
- PostgreSQL
- Redis
- Git

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/echovault.git

cd echovault
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

Example configuration:

```env
SECRET_KEY=your-secret-key
DEBUG=True

DATABASE_URL=postgresql://postgres:password@localhost:5432/echovault

REDIS_URL=redis://localhost:6379/0

EMAIL_HOST=smtp.example.com
EMAIL_PORT=587
EMAIL_HOST_USER=your-email
EMAIL_HOST_PASSWORD=your-password
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

### 8. Start the Celery worker

```bash
celery -A config worker -l info
```

Celery Beat can be used to schedule periodic notification tasks.

---

## Testing

Run the test suite:

```bash
pytest
```

The test suite should cover:

- Authentication and authorization
- Capsule creation and validation
- Capsule sealing
- Time-based access control
- Memory uploads
- Invitation workflows
- Notification processing
- Database constraints

---

## Security Considerations

EchoVault stores personal memories and private messages. Security is a core requirement.

- Enforce authentication on private endpoints.
- Validate capsule ownership and participant permissions.
- Enforce unlock dates on the backend.
- Validate uploaded file types and sizes.
- Keep secrets in environment variables.
- Use HTTPS in production.
- Prevent unauthorized access to media files.
- Avoid exposing capsule contents in notification emails.
- Use database transactions for critical operations.

Capsule time locking is an application-level access control mechanism. It does not imply end-to-end encryption or protection against privileged database access.

---

## Roadmap

### Phase 1 — Core MVP

- [ ] Authentication
- [ ] Capsule creation
- [ ] Memory management
- [ ] Capsule sealing
- [ ] Time-based access control
- [ ] Countdown
- [ ] Dashboard

### Phase 2 — Advanced Features

- [ ] Future Self Mode
- [ ] Email notifications
- [ ] Capsule invitations
- [ ] Shared capsules
- [ ] Activity history
- [ ] Archive and search

### Phase 3 — Production

- [ ] Automated tests
- [ ] Docker setup
- [ ] Production configuration
- [ ] API documentation
- [ ] Deployment
- [ ] Monitoring and logging

---

## Learning Objectives

EchoVault is designed as a learning project for exploring Python and Django through a real application.

The project provides practical experience with:

- Python fundamentals and object-oriented programming
- Django project architecture
- Django REST Framework
- ORM relationships and database migrations
- Authentication and authorization
- REST API design
- Date and time handling
- File processing
- Background tasks with Celery
- Redis
- Automated testing
- Database transactions
- Docker and deployment

---

## Project Status

EchoVault is being developed as a four-week learning project.

The initial goal is to build a secure, functional MVP with capsule creation, memory management, and time-based access control before expanding into shared capsules and future delivery.

## License

This project is intended for educational and portfolio purposes.

A license can be added before public distribution.
