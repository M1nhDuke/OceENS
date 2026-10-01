# OcéEns II

Course evaluation platform designed for the EPF engineering school.

## Overview

The **OcéEns II** application enables program managers, facilitators, campus management, and administrators to create and manage evaluation surveys for various EPF study tracks, and allows students to complete them. Responses can be exported, viewed, and summarized using an LLM. The interface adheres to EPF's official visual identity guidelines.

### Technical Stack

| Component | Technology |
|-----------|-------------|
| **Framework** | FastAPI (Python 3.12) |
| **Authentication** | Microsoft Entra ID (Azure AD) via OAuth2.0 / MSAL, Microsoft Graph |
| **Database** | SQLite (via SQLAlchemy + SQLModel) |
| **Templating** | Jinja2 (server-side rendering) |
| **Frontend** | HTML / CSS / JavaScript (vanilla) |
| **Server** | Uvicorn |
| **Logging** | Standard Python `logging` module, via Uvicorn handlers |
| **Exports** | Pandas (CSV) |
| **Verbatim summaries** | Separate daemon, LLM call (`requests-cache`, `markdown-it-py`) |

---

## Roles

- `student`: responds to surveys for which they are registered.
- `program_manager:<code>`: manages surveys for their specific study track(s).
- `facilitator:<code>`: facilitates surveys for their specific study track(s).
- `campus_manager:<campus>`: campus-wide scope.
- `admin`: general administration.

A user can hold multiple roles, each with its own scope (study track codes or campus names separated by `;`). ---

## Main Pages and Routes

| Route | Description |
|-------|-------------|
| `/` | Home page, authentication hub. |
| `/login`, `/auth/callback`, `/logout` | Microsoft Entra ID authentication flow. |
| `/dev/login` | Development login: user selection page via `GET`, login via `POST` (only with `AUTH_MODE=dev`; see [Development Mode Authentication](#development-mode-authentication)). |
| `/dashboard/student` | Student dashboard. |
| `/dashboard/program-manager` | Program manager dashboard. |
| `/dashboard/facilitator` | Facilitator dashboard. |
| `/dashboard/campus-manager` | Campus management dashboard. |
| `/dashboard/teachers/analytics` | Satisfaction score by teacher, filterable by year/semester/program. Accessible to `campus_manager` and `program_manager` roles, scoped to their respective areas. |
| `/dashboard/admin` | Administrator dashboard. |
| `/dashboard/survey-create` | Survey creation/configuration. |
| `/api/surveys/{survey_id}` | Questionnaire (survey response). |
| `/api/surveys/{survey_id}/status` | Change survey status. |
| `/api/surveys/{survey_id}/students` | Manage students registered for a survey. |
| `/api/surveys/{survey_id}/export` | CSV export of responses. |
| `/api/surveys/{survey_id}/visualisation` | Response visualization. Accepts `?teacher=<name>` to load with a pre-applied teacher filter. |
| `/api/surveys/{survey_id}/generate-summaries` | Trigger LLM summary generation. |
| `/api/surveys/{survey_id}/destroy-summaries` | Delete generated summaries. |
| `/api/users/{user_id}/role` | Modify a user's role. |
| `/backend/prompts` | List of LLM prompts (admin only). |
| `/backend/prompts/new` | Prompt creation form. |
| `/backend/prompts/{id}/edit` | Prompt editing form. |
| `/api/prompts` | Create a prompt (POST, form). |
| `/api/prompts/{id}` | Modify a prompt (PUT, fetch). Blocked if the prompt is referenced in `summaries`. |
| `/api/prompts/{id}/delete` | Delete a prompt (POST, form). Blocked if the prompt is referenced in `summaries`. |

---

## Installation and startup

### Prerequisites

- Python 3.12
- A configured `.env` file (see [Configuration](#configuration) section)

### Using Docker Compose (recommended)

```bash
docker compose up --build
```

The SQLite database is persisted in a local directory. Default is `./database/`; To point to a different location, define `LOCAL_DATABASE_DIR` in `.env` or in the environment:

```env
LOCAL_DATABASE_DIR=/path/to/database
```

**Development** — source code mounted as a volume (changes take effect without rebuilding), seed data available:

```bash
docker run -p 8000:8000 --env-file .env -v oceens_db:/app/database -v ./import:/app/import -v .:/app oceens:1.0
```

> The `Dockerfile` includes `--reload` in the Uvicorn command: Uvicorn detects file changes and automatically reloads the application when the source code is mounted via `-v .:/app`. Remove `--reload` for production deployment.

> The SQLite database is persisted in the Docker volume `oceens_db` (`/app/database`).
> The `.env` file is never copied into the image; it is passed via `--env-file` at startup.

### Without Docker (manual installation)

### Steps

1. **Clone the project**

```bash
git clone <url-du-repo>
cd OceENS
```

2. **Create and activate a virtual environment**

```bash
python -m venv env
env/scripts/activate        # Windows
source env/bin/activate     # Linux / macOS
```

3. **Install dependencies**

```bash
pip install -r requirements.txt
```

4. **Add the database**
Create a `database/` folder and place the `db_oceens.db` file inside it, or let `seed_all_if_necessary()` initialize an empty database upon first startup.

5. **Launch the application**:

```bash
fastapi dev
```

Or directly using Uvicorn:

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

In production, `launch.sh` starts the application and the summary daemon in separate `screen` sessions.

6. **(Optional) Launch the LLM summary daemon**:

```bash
python summaries_generator_daemon.py
```

This process runs in a loop, writes to the database, and contacts an external LLM service; launch it only when necessary. 

> Setting the `RUN_SUMMARIES_DAEMON=1` environment variable in the
> `.env` file automatically launches the daemon as a separate process when
> Uvicorn starts (and stops it upon shutdown). Use this in production with Docker. 
> Note: `launch.sh` (without Docker) already manages the daemon in its own `screen` session.

7. Open your browser to **http://localhost:8000**.

---

## Logging

Application logs use the standard Python `logging` module and the
`uvicorn` logger. This allows messages from the application, `auth.py`, and `seed.py` to adopt
the format, colors, and handlers already configured by the server.

Levels are used according to their severity:

| Level | Usage |
|--------|-------------|
| `DEBUG` | Detailed information useful for development and seeding. |
| `INFO` | Startup, shutdown, and normal application operations. |
| `WARNING` | Expected resource missing or non-blocking situation. |
| `ERROR` / `EXCEPTION` | Operation failure; `logger.exception()` preserves the traceback. |
| `CRITICAL` | Essential configuration missing, preventing startup. |

Example:

```python
import logging

logger = logging.getLogger("uvicorn")

logger.info("Operation completed")

try:
operation_risquee()
except Exception:
logger.exception("Operation failed")
```

New diagnostic messages should use the appropriate logger rather than
`print()`. The application level is currently set to `DEBUG` in
`core/dependencies.py`. Application logs go through the Uvicorn handler, typically
writing to `stderr`; with separate redirection, use `2> error.log`, for example,
to capture them. ---

## Configuration

Create a `.env` file at the project root:

```env
# Azure Entra ID
ENTRA_CLIENT_ID=your_app_id_here
ENTRA_CLIENT_SECRET=your_secret_here
ENTRA_TENANT_ID=your_tenant_id_here
REDIRECT_URI=http://localhost:8000/auth/callback
ALLOWED_DOMAINS=epf.fr,epfedu.fr

# Session (required unless AUTH_MODE=dev)
SECRET_KEY=your_secure_random_key_here

# LLM Summaries
LLM_API_KEY=your_llm_api_key_here
```

The `SECRET_KEY` signs session cookies; anyone who knows it can forge an admin session. It is **mandatory unless `AUTH_MODE=dev`** is set: if it is missing or empty, the application logs a critical error and shuts down on startup (exit code 1). Generate it using `python -c "import secrets; print(secrets.token_urlsafe(32))"`.

> [!CAUTION]
> Never commit the `.env` file. It is already listed in `.gitignore`, along with `*.db` files (`database/db_oceens.db`, ​​`cache_llm.db`).

---

## LLM Providers (verbatim summaries)

Verbatim summaries are generated by an LLM. The provider is
**configurable via the interface** (`/backend/providers`, admin only),
without modifying the code. The default provider is **Ollama EPF**
(`https://locallm.mde.epf.fr/ollama`), created automatically upon
first startup.

### Supported API types

| `api_type` | Covers |
|------------|--------|
| | `ollama`    | Ollama servers (local, EPF, third-party) |
| `openai`    | OpenAI **and any OpenAI-compatible endpoint**: vLLM, Groq, Mistral, LM Studio… |
| `anthropic` | Claude API (Anthropic) |

### Security principle: no keys in the database

The SQLite database is unencrypted and included in backups. **Therefore, no API
keys are stored in it.** The `llm_providers` table contains only the *name*
of the environment variable (`api_key_env`, e.g., `OPENAI_API_KEY`); the value
remains in the `.env` file and is resolved only at the time of the call. This name
is validated against a whitelist (`LLM_*` or `*_API_KEY`) to prevent pointing to
system secrets (`SECRET_KEY`, `ENTRA_CLIENT_SECRET`, etc.).

### Adding a new provider

1. **Add the key to the `.env` file** using a compliant name (`LLM_*` or `*_API_KEY`):

```env
OPENAI_API_KEY=sk-...
```

2. **Restart the synthesis daemon** (the `.env` variables read only
at startup):

```bash
python summaries_generator_daemon.py
```

3. **Create the provider** in `/backend/providers` → *+ New provider*:
enter the name, API type, base URL, environment variable name
(`OPENAI_API_KEY`), and a default model. The **"key present /
absent"** indicator confirms that the variable is successfully loaded. The **Test**
button verifies that the URL and key are responsive, then triggers a token
generation to confirm the account can actually generate content (see below).

4. **Link a prompt** to the provider: in `/backend/prompts`, a `<select>`
dropdown allows you to choose the provider for a prompt. A prompt without a provider
(`provider_id` NULL) automatically falls back to Ollama EPF.

> [!NOTE]
> A provider referenced by at least one prompt cannot be deleted
> (to avoid breaking the configuration of those prompts).

### Exhausted credit and other provider errors

Each provider reports failures in a different format: exhausted credit
appears as a `429 insufficient_quota` for OpenAI, but as a `400 "Your credit
balance is too low"` for Anthropic. `services/llm_client.py` normalizes these
responses into categories (`quota`, `rate_limit`, `auth`, `model`, `server`) and
generates a human-readable message:

> ⚠️ Credit or quota exhausted at the provider: the key is valid, but the
> account can no longer generate content. Top up the account or choose another
> provider. (provider OpenAI, model gpt-4o-mini, HTTP 429)

This message is written to `Summary.metadata_text` instead of the raw JSON—making
it directly visible in the interface when a summary fails. The provider's raw
response remains in the daemon logs for diagnostic purposes.

> [!IMPORTANT]
> The **Test** button does more than just list models: with both OpenAI and
> Anthropic, `GET /v1/models` still returns a successful response even when the
> balance is zero. Therefore, a single-token generation ping (negligible cost)
> is sent afterward—this is the only way to detect exhausted credit **before**
> launching a summarization campaign.

---

## Summarization costs

The cost of each summary is **measured, not estimated**. At the time of
generation, the daemon records the token counts returned by the provider
(`Summary.input_tokens`, `output_tokens`, `model_used`): this is the only
opportunity to capture them, as no API allows retrieving them later. The
amount is then calculated by cross-referencing these counts with the pricing
schedule.

> [!NOTE]
> This section replaces the old `llm-utils/token-counting/` scripts, which
> counted tokens from the **repository source code** and multiplied them by a
> hard-coded rate. That measurement did not reflect the application's actual
> expenditure. Tracking now focuses on calls that are actually billed.

### Pricing schedule — `/backend/llm/prices`

Prices are stored in the database (in the `llm_model_prices` table) in
**dollars per million tokens**, matching the rates published by the
providers. They can be edited via the
admin interface: there is no need to ship a new version to handle price
adjustments or to cover a locally added provider.

They are pre-populated at startup (`seed_model_prices`, idempotent—a manually
corrected rate is never overwritten):

| Model | Input $/M | Output $/M |
| --- | ---: | ---: |
| `claude-opus-5` | 5.00 | 25.00 |
| `claude-sonnet-5` | 3.00 | 15.00 |
| `claude-haiku-4-5` | 1.00 | 5.00 |
| `gemma4:26b` (Ollama EPF, self-hosted) | 0.00 | 0.00 |

Rates for other providers (OpenAI, Mistral, Groq, etc.) must be **entered
manually**: they are not automatically inferred. A provider-specific rate
takes precedence over a generic rate sharing the same model name.

### Viewing

| Where | What |
| --- | --- |
| `/backend/llm/costs` | Overall cost, broken down by survey and model (admin) |
| 💰 button on a survey row | Cost of summaries for that survey |

### Uncosted items

A summary cannot be costed if its usage counters are missing (e.g., generated
before this feature existed, or from a provider that does not expose them)
or if its model has no recorded rate. In such cases, it is **tracked
separately**—never estimated or set to zero: a fabricated figure would be
more misleading than a missing one, as it would appear with the authority
of an actual cost. The screens
explicitly indicate when a total is partial.

This is distinct from a **zero** cost: self-hosted models actually
cost $0.00, which conveys different information than "unknown."

> [!IMPORTANT]
> Tracking begins upon deployment: summaries generated prior to this
> lack corresponding database counters and cannot be quantified
> retroactively.

---

## Project structure

```
OceENS/
├── main.py                       # FastAPI factory, middleware, and router assembly
├── sondage_loader.py             # Loading a complete survey for export
├── survey_loader_from_xlsx.py    # Survey import from an Excel file
├── summaries_generator_daemon.py # Asynchronous processing of LLM summaries (separate process)
├── launch.sh                     # Launch script (production, non-Docker)
├── requirements.txt              # Python dependencies
├── Dockerfile                    # Application Docker image
├── .dockerignore                 # Files excluded from Docker build
├── .env                          # Environment variables (⚠️ not committed)
├── .gitignore                    # Files and folders ignored by Git
│
├── core/                         # Low-level access and security
│   ├── auth.py                   #   Microsoft Entra ID authentication (login, logout, callback) and dev login
│   ├── database.py               #   SQLite engine and SessionDep dependency
│   ├── security.py               #   Roles, scopes, access control
│   ├── dependencies.py           #   Shared Jinja templates and logger
│   └── seed.py                   #   Initial data and training course synchronization
│
├── models/                       # SQLModel schema, one file per table
│   ├── __init__.py               #   Re-exports all classes (see docstring)
│   └── User.py, Survey.py, ...
│
├── routers/                      # Routes organized by business domain
│   ├── pages.py                  #   Home and role-based dashboards
│   ├── surveys.py                #   Surveys: CRUD, status, export, visualization
│   ├── students.py               #   Student registration for surveys
│   ├── users.py                  #   User role management
│   ├── summaries.py              #   Triggering LLM summaries
│   ├── prompts.py                #   Prompt administration
│   ├── survey_templates.py       #   Survey template administration
│   ├── sections_questions.py     #   Section and question administration
│   └── llm/                      #   LLM administration (URLs unchanged)
│       ├── _access.py            #     Access control LLM screen access
│       ├── providers.py          #     LLM providers (CRUD + connection test)
│       ├── prices.py             #     Pricing schedule by model
│       └── costs.py              #     Overall cost and cost per survey
│
├── database/                     # Folder containing the database (ignored by Git)
│   └── db_oceens.db
│
├── services/                     # Business logic
│   ├── helpers.py                # Navigation, statistics, filters, sorting
│   ├── visualisation_data.py     # Aggregations and visualization context
│   ├── llm_client.py             # Multi-provider LLM client (ollama/openai/anthropic)
│   ├── llm_costs.py              # Summary costs (measured tokens × pricing schedule)
│   └── export_csv.py             # CSV export of responses
│
├── llm-utils/                    # Standalone LLM tools
│   └── README.md                 # (cost tracking moved to the app; see above)
│
├── templates/                    # HTML templates (Jinja2)
│   ├── index.html                     # Home page / login
│   ├── dashboard/
│   │   ├── admin.html
│   │   ├── student.html
│   │   ├── program_manager.html
│   │   ├── facilitator.html
│   │   ├── campus_manager.html
│   │   ├── teachers-analytics.html       # Teacher satisfaction (campus_manager, program_manager)
│   │   ├── survey.html                   # Survey response
│   │   ├── survey_create.html            # Survey creation
│   │   └── visualisation.html            # Response visualization
│   ├── backend/                       # Admin pages (admin only)
│   │   ├── prompts.html               # List of LLM prompts
│   │   ├── prompt_form.html           # Shared create/edit form
│   │   └── llm/                       # LLM screens (providers, rates, costs)
│   │       ├── providers.html
│   │       ├── provider_form.html
│   │       ├── prices.html            # Editable rate schedule
│   │       └── costs.html             # Overall and per-survey costs
│   └── template_parts/                # Reusable fragments across dashboards
│       ├── part_site_header.html
│       ├── part_dashboard_navigation.html
│       ├── part_theme_switcher.html
│       └── ...
│
├── static/
│   ├── css/                      # admin.css, student.css, program_manager.css, survey.css,
│   │                              # survey_create.css, visualisation.css, prompt_form.css,
│   │                              # llm_backend.css (LLM screens), theme.css, site_header.css,
│   │                              # dashboard_navigation.css, responsive.css
│   ├── js/
│   │   └── survey.js
│   └── img/
│
└── env/                           # Python virtual environment (not committed)
```

---

## Authentication (OAuth 2.0)

The authentication flow relies on **Microsoft Entra ID** via the MSAL library:

```
1. User clicks "Log in"
→ FastAPI generates a random state (UUID, CSRF protection)
→ Redirects to the Microsoft login page

2. The user authenticates with Microsoft
→ Microsoft redirects to `/auth/callback` with a code + state

3. The server exchanges the code for an access token
→ Retrieves user info via Microsoft Graph
→ Queries the database to obtain role(s) and their scope
→ Creates the session `{name, email, roles}`
→ Redirects to the corresponding dashboard

4. Upon logout (`/logout`)
→ Deletes the session and cookies
→ Logs out from Microsoft
→ Returns to the home page
```

Authentication alone does not authorize any business actions: each route subsequently verifies the role and scope (training program or campus) via `require_roles()` and associated helper functions.

---

## Development mode authentication

To work on a fork without an Azure application, the **development login** allows logging in as any user without proof of identity. It must **never** be used in production.

| Variable | Role |
|----------|------|
| `AUTH_MODE` | `entra` (default) or `dev`, case- and space-insensitive. Any other value causes the application to stop on startup. In `dev` mode, `ENTRA_*` variables are not required. |
| `DEV_LOGIN_KEY` | Optional; `dev` mode only. If set, every login attempt must provide it (in the `key` field), otherwise a `401` error is returned. If not set, login is open. Ignored (with a warning) in `entra` mode. |
| `SECRET_KEY` | Optional in `dev`: if missing, a random key is generated at startup (triggering a warning), and sessions are lost upon restart. Mandatory in `entra`. |
| `ALLOWED_DOMAINS` | Also applies in `dev` (returns `403` for other domains); defaults to `epf.fr,epfedu.fr` in this mode. |

In `dev` mode, the session cookie is no longer restricted to HTTPS (`http://localhost` works), `/login` redirects to `/dev/login`, `/auth/callback` does not exist, and `/logout` clears the session before redirecting to `/`. A warning is logged at startup. A non-dismissible red banner appears at the top of every page containing the shared header; it displays the currently logged-in address, offers a "Change user" option (`/dev/login`), and indicates "access open to all" when `DEV_LOGIN_KEY` is not set.

`POST /dev/login` expects a form containing `email`, `name` (optional), and `key` (if `DEV_LOGIN_KEY` is set). The user is retrieved or created just as with the Entra callback: an unknown email results in a new student. If `name` is omitted, the display name is derived from the email (`bob.leponge@epfedu.fr` → "Bob Leponge"). A new login replaces the existing session; this is how users are switched.

In a browser, `GET /dev/login` displays a list of users from the database, grouped by role name (ignoring scope; a user without a role appears under `student`, while a user with multiple roles appears under each of them). Clicking an entry logs the user in as the selected user; a free-text field allows logging in with a different address and an optional name. If `DEV_LOGIN_KEY` is set, a single key field appears and is used for all logins on the page; the key is never stored in the session. You return to this page to switch users.

```bash
AUTH_MODE=dev DEV_LOGIN_KEY=my-key uvicorn main:app

# Log in as the seed admin; -c saves the session cookie
curl -i -c cookies.txt \
-d email=antoine.gademer@epf.fr -d key=my-key \
http://localhost:8000/dev/login

# Reuse the cookie (-b) for subsequent requests
curl -b cookies.txt -c cookies.txt -L http://localhost:8000/
```

> [!WARNING]
> `dev` mode does not require a `SECRET_KEY`. Without one, the key is random and unknown; however, if a known `SECRET_KEY` is set (shared, copied from an example, etc.), anyone who knows it can forge a session cookie and bypass `DEV_LOGIN_KEY`: `dev` mode accepts it, as it is intended for local use only.

---

## Notable features

### Teacher analytics

The `/dashboard/teachers/analytics` route (`campus_manager`, `program_manager`) aggregates satisfaction scores by `(teacher, survey)` based on `QCU_Satisfaction` responses linked to an `Answer.teacher` (ME sections). The list of teachers is sorted using `teacher_sort_key()`—which is case- and accent-insensitive—and remains filterable by academic year, semester, program, and teacher. ### Instructor filter in the visualization

A client-side selector filters the visualization without reloading the page: only the selected instructor's modules remain visible, while the Campus and Training sections are hidden. The page reads `?teacher=<name>` upon loading to apply the initial filter; links from the analytics dashboard pass this parameter, allowing a click on an instructor's score to open their specific view directly.

### Surveys imported via Excel

Surveys loaded via `survey_loader_from_xlsx.py` lack a `QCU_Attendance` question; consequently, `services/visualisation_data.py` uses `satisfaction_responses_count` as a fallback denominator for the instructor score. Instructor names are normalized using `.title()` during both import and aggregation to merge case variants (e.g., `"GADEMER Antoine"` and `"Gademer Antoine"` become a single entry). Questions are sorted by `question_id` in the template, ensuring that graphs appear before verbatim responses regardless of the insertion order.

### Campus management scope

The `campus_manager` dashboard displays only closed surveys that have at least one respondent. The link to the questionnaire and the QR code are hidden (`can_view_survey_link=False`), as this role is intended for viewing results rather than distributing surveys. The `{% if can_view_survey_link | default(true) %}` guard ensures other dashboards remain unaffected.

### Cleaning up orphaned students

When a survey is deleted, students who are no longer linked to **any other** survey are also removed to prevent the accumulation of unused accounts (`services/helpers.py`, `_delete_orphan_students`). A safeguard protects users with privileged roles (`admin`, `program_manager`, `facilitator`, `campus_manager`): instructors or administrators who have responded to a survey are never deleted. ### Adding a user via email

The "Users" tab in the admin dashboard features an
**"+ Add user"** button: an email address is all that is needed to create the account,
assigning the default `student` role (`POST /api/users`, admin only). The email
is validated (format and allowed domain), and duplicates are rejected.

---

## Deployment checklist

- [ ] `.env` created with actual Azure credentials and a dedicated `SECRET_KEY` (mandatory unless `AUTH_MODE=dev`; otherwise, the application will not start)
- [ ] `AUTH_MODE` unset or set to `entra`
- [ ] Valid SSL certificate (Let's Encrypt or equivalent)
- [ ] `https_only=True` in SessionMiddleware (automatic unless `AUTH_MODE=dev`)
- [ ] Database present (`database/db_oceens.db`) or Docker volume mounted
- [ ] Secure environment variables, including `LLM_API_KEY`
- [ ] **Docker Compose**: `.env` loaded via `env_file` (never copied into the image); `LOCAL_DATABASE_DIR` pointing to the correct base directory
- [ ] `summaries_generator_daemon.py` daemon running if LLM summaries are used

---

## Pre-contribution validation

The repository does not include an automated test suite or CI pipeline. Before proposing a change:

```bash
python -m compileall -q main.py \
sondage_loader.py survey_loader_from_xlsx.py summaries_generator_daemon.py \
core models routers services
git diff --check
```

Then, manually test the relevant routes using a disposable SQLite database (never a production copy), using the appropriate roles and survey statuses. ---

## Resources

- [FastAPI](https://fastapi.tiangolo.com/)
- [FastAPI and Uvicorn logging guide](https://apitally.io/blog/fastapi-logging-guide)
- [MSAL Python](https://github.com/AzureAD/microsoft-authentication-library-for-python)
- [Microsoft Graph](https://learn.microsoft.com/en-us/graph/)
- [Jinja2](https://jinja.palletsprojects.com/)
- [SQLAlchemy](https://www.sqlalchemy.org/)
- [SQLModel](https://sqlmodel.tiangolo.com/)
- [Pandas](https://pandas.pydata.org/)

---

**OcéEns Team** — EPF