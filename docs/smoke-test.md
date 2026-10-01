# Manual smoke test

The repository contains neither an automated test suite nor CI (the initial
test setup is tracked in #85, CI in #78). This procedure is carried out entirely
**outside the standard process**: starting from a fresh clone, the application
is launched, and its response and exit code are observed.

Perform this check before proposing any changes affecting startup,
configuration, dependencies, or the container.

## System-specific conventions

Commands are provided for **Windows (PowerShell)** followed by **macOS /
Linux (bash)**. Only four elements differ:

| | Windows (PowerShell) | macOS / Linux (bash) |
|---|---|---|
| Virtual environment interpreter | `.venv\Scripts\python.exe` | `.venv/bin/python` |
| Set variable for a command | `$env:VAR = "x"` then `Remove-Item Env:VAR` | `VAR=x command` |
| Read exit code | `$LASTEXITCODE` | `echo $?` |
| Copy / rename file | `Copy-Item`, `Rename-Item` | `cp`, `mv` |

Commands invoke the interpreter **via its path** (`.venv\Scripts\python.exe`)
rather than activating the environment; on Windows, `Activate.ps1` is blocked
by default due to PowerShell's execution policy, and that is outside the
scope of this test. ## Static checks

Identical on both systems (single line, no line continuation):

```
python -m compileall -q main.py sondage_loader.py survey_loader_from_xlsx.py summaries_generator_daemon.py core models routers services
git diff --check
```

## 1. Local startup, without credentials

In a fresh clone of the branch, with an empty virtual environment.

**Windows (PowerShell)**

```powershell
Copy-Item .env.example .env
python -m venv .venv
.venv\Scripts\python.exe -m pip install -r requirements.txt
.venv\Scripts\uvicorn.exe main:app --port 8000
```

**macOS / Linux (bash)**

```bash
cp .env.example .env
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/uvicorn main:app --port 8000
```

Expected behavior, without any Entra credentials or LLM keys:

| Route | Response |
|---|---|
| `GET /` | 200 |
| `GET /dev/login` | 200 |
| `GET /nope` | 303 to `/` (404 middleware → `/`) |

Startup logs show table creation and insertion of the demo dataset,
with no errors or exception traces.

## 2. Startup with Docker

Requires a **running Docker daemon** — Docker Desktop on Windows
(using the WSL 2 backend) or macOS, or the native daemon on Linux. The
command is the same everywhere:

```
docker compose up --build
```

Expected outcome: the image builds, the container starts without entering a
restart loop, and `/`, `/dev/login`, and `/nope` respond just as they did in step 1.

Without a `.env` file, `docker compose` fails with `env file .env not found`—this
is intentional; the first step after forking the repo is to copy `.env.example`.

To stop and clean up:

```
docker compose down
```

## 3. Exit codes for invalid configuration

An invalid startup configuration must exit with **code 1** so that a
supervisor or CI system detects the failure.

The `.env` file must be set aside for the last two cases; otherwise,
`load_dotenv()` would read `AUTH_MODE=dev` from it, and the application
would start normally with exit code 0.

**Windows (PowerShell)**

```powershell
# Invalid AUTH_MODE
$env:AUTH_MODE = "bogus"
.venv\Scripts\python.exe -c "import main"; $LASTEXITCODE   # 1
Remove-Item Env:AUTH_MODE

# Missing ENTRA_* variables, without .env
Rename-Item .env .env.bak
'AUTH_MODE','ENTRA_CLIENT_ID','ENTRA_CLIENT_SECRET','ENTRA_TENANT_ID' | 
ForEach-Object { Remove-Item "Env:$_" -ErrorAction SilentlyContinue }
.venv\Scripts\python.exe -c "import main"; $LASTEXITCODE   # 1

# Missing SECRET_KEY in Entra mode, without .env
$env:ENTRA_CLIENT_ID = "x"; $env:ENTRA_CLIENT_SECRET = "x";
``` $env:ENTRA_TENANT_ID = "x"
Remove-Item Env:SECRET_KEY -ErrorAction SilentlyContinue
.venv\Scripts\python.exe -c "import main"; $LASTEXITCODE   # 1
'ENTRA_CLIENT_ID','ENTRA_CLIENT_SECRET','ENTRA_TENANT_ID' | 
ForEach-Object { Remove-Item "Env:$_" }
Rename-Item .env.bak .env
```

**macOS / Linux (bash)**

```bash
# Invalid AUTH_MODE
AUTH_MODE=bogus .venv/bin/python -c "import main"; echo $? # 1

# Missing ENTRA_* variables, without .env
mv .env .env.bak
env -u AUTH_MODE -u ENTRA_CLIENT_ID -u ENTRA_CLIENT_SECRET -u ENTRA_TENANT_ID \
.venv/bin/python -c "import main"; echo $? # 1

# Missing SECRET_KEY for entra mode, without .env
env -u AUTH_MODE -u SECRET_KEY ENTRA_CLIENT_ID=x ENTRA_CLIENT_SECRET=x ENTRA_TENANT_ID=x \
.venv/bin/python -c "import main"; echo $? # 1
mv .env.bak .env
```

Expected output: the log line `INVALID AUTH_MODE 'bogus'` for the first case,
`MISSING ENTRA INFO. Please check .env` for the second,
`MISSING SECRET_KEY. Required with AUTH_MODE=entra, please check .env` for the
third. As a reference, `AUTH_MODE=dev` exits with code 0, even without a `SECRET_KEY`.

## 4. Missing LLM key

`.env.example` provides an **empty** `LLM_API_KEY`: the application starts
normally, but summaries are unavailable. With the daemon `summaries_generator_daemon.py` is running, a summarization request is flagged with a
configuration error (`http_status` 500, "environment variable
missing or empty") and no call is made to the provider.

## 5. Using an LLM key

Each student retrieves their own key from <https://locallm.mde.epf.fr> by
logging in with their EPF account, then enters it into their `.env` file:

```
LLM_API_KEY=<your key>
```

Quick check, without using the interface. **The key must be present in
the command's environment, not just in the `.env` file**:
`load_dotenv()` is called by the application, the daemon, and the
authentication module, but not by `services/llm_client.py`—the only module
imported here. Without the prefix shown below, the command raises an
`LLMConfigError` regardless of the `.env` file's contents.

The `python -c` command fits on a single line and is identical on both
systems; only the interpreter path and the method of defining the
variable differ. **Windows (PowerShell)**

```powershell
$env:LLM_API_KEY = "<your key>"
.venv\Scripts\python.exe -c "from types import SimpleNamespace; from services import llm_client as c; p = SimpleNamespace(name='Ollama EPF', api_type='ollama', base_url='https://locallm.mde.epf.fr/ollama', api_key_env='LLM_API_KEY', default_model='gemma4:26b'); print(c.check_model(p, 'gemma4:26b')); print(c.ping_generation(p, 'gemma4:26b'))"
Remove-Item Env:LLM_API_KEY
```

**macOS / Linux (bash)**

```bash
LLM_API_KEY=<your key> .venv/bin/python -c "from types import SimpleNamespace; from services import llm_client as c; p = SimpleNamespace(name='Ollama EPF', api_type='ollama', base_url='https://locallm.mde.epf.fr/ollama', api_key_env='LLM_API_KEY', default_model='gemma4:26b'); print(c.check_model(p, 'gemma4:26b')); print(c.ping_generation(p, 'gemma4:26b'))"
```

Expected output: `True`, followed by `(True, None, None)`. `check_model` alone is not enough—
the model list still responds normally with an account that has no credit;
only the generation call reveals the issue. With an empty value, or if the variable is missing, the same command raises `LLMConfigError`: this is the behavior from step 4.

Next, an end-to-end test: request the generation of survey summaries while `summaries_generator_daemon.py` is running. This part does not require the prefix, as the daemon reads the `.env` file itself. The lines transition from `http_status` 0 to 200, one at a time (since the daemon operates sequentially), and the summary is displayed in HTML. Never commit the key; the `.env` file is ignored by Git.

## Next steps

Manually test the routes affected by the change using a disposable SQLite database (never a production copy), employing the relevant roles and survey statuses.