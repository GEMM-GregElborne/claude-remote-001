# Session Log — Django + Docker + Git Setup

**Date:** 2026-09-18
**Host:** `bage-cdmcentral`
**Repo:** [github.com/GEMM-GregElborne/claude-remote-001](https://github.com/GEMM-GregElborne/claude-remote-001)

## Summary

Set up a Django project from scratch on a machine with no working `pip`/`venv`,
using Docker for the dev environment, then pushed it to a new GitHub repo and
confirmed it's reachable from the main computer's browser.

## What we did

### 1. Checked the environment

- Python 3.12.3 was present, but `pip` and `venv`'s `ensurepip` were both
  missing, and there was no passwordless `sudo` to install them.
- Docker was installed and usable (user already in the `docker` group).
- **Decision:** use Docker to run Django rather than pasting the sudo
  password into chat (chat history would have persisted it in plaintext).

### 2. Scaffolded the project

Created in `~/claude`:

- `requirements.txt` — pins `Django>=5.0,<6.0`
- `Dockerfile` — `python:3.12-slim`, installs requirements, runs
  `manage.py runserver 0.0.0.0:8000`
- `docker-compose.yml` — builds the image, mounts `.:/app`, publishes port
  `8000`

Built the image and ran:

```bash
docker compose build
docker compose run --rm --no-deps --user "$(id -u):$(id -g)" web \
  django-admin startproject config .
```

(`--user` kept the generated files owned by the normal user, not root.)

Verified with `docker compose up -d` + `curl localhost:8000/` → `HTTP 200`.

### 3. Set up git

- `git init`, branch renamed `master` → `main`
- `.gitignore` covering `__pycache__/`, `*.sqlite3`, `.env`, `.venv/`,
  `staticfiles/`, `media/`, and `.claude/` (local Claude Code session config,
  not project code)
- Local git identity set (not global): `Greg Elborne <greg.elborne@gmail.com>`
- Initial commit: *"Initial Django project scaffold with Docker dev
  environment"*

### 4. Pushed to GitHub

- No `gh` CLI and no existing SSH key on the box, so generated a fresh
  ed25519 keypair (`~/.ssh/id_ed25519`) rather than using a token in chat.
- Public key added to the GitHub account; new empty repo created:
  `GEMM-GregElborne/claude-remote-001`
- Confirmed auth with `ssh -T git@github.com`
- Added remote and pushed:

  ```bash
  git remote add origin git@github.com:GEMM-GregElborne/claude-remote-001.git
  git push -u origin main
  ```

### 5. Accessed it from the main computer

- Clarified that `bage-cdmcentral` is the actual host in front of the user
  (not a separate remote box), and that the Docker container running there
  is the *same* container this session had been driving.
- First attempt at `http://bage-cdmcentral:8000/` failed with
  `DisallowedHost` (Django's `ALLOWED_HOSTS = []` by default).
- Fix: `config/settings.py`

  ```python
  ALLOWED_HOSTS = ["localhost", "127.0.0.1", "bage-cdmcentral"]
  ```

  Django's autoreloader picked this up without a container restart (volume
  mount).
- Confirmed working: Django's "The install worked successfully!" page loaded
  in the browser at `http://bage-cdmcentral:8000/`.
- Committed and pushed the fix (`dc92a9e`).

## Repo layout at end of session

```
claude/
├── config/               # Django project package (settings, urls, wsgi, asgi)
├── docs/
│   └── session-2026-09-18-django-setup.md
├── Dockerfile
├── docker-compose.yml
├── manage.py
├── requirements.txt
└── .gitignore
```

## Handy commands going forward

| Task | Command |
|---|---|
| Start dev server | `docker compose up` (or `-d` for background) |
| Stop dev server | `docker compose down` |
| Django management command | `docker compose run --rm web python manage.py <cmd>` |
| Create a new app | `docker compose run --rm web python manage.py startapp <name>` |
| Run migrations | `docker compose run --rm web python manage.py migrate` |
| Rebuild image (after requirements.txt changes) | `docker compose build` |

## Notes / things to revisit before production

- `ALLOWED_HOSTS` currently lists the dev hostname directly — fine with
  `DEBUG=True` locally, but needs a real strategy (env var, proper domain)
  before deploying anywhere.
- `SECRET_KEY` is still the Django-generated dev default committed in
  `config/settings.py` — should move to an environment variable before this
  goes anywhere public.
- No database beyond SQLite yet (Postgres was discussed as a possible
  next step via a sibling `docker-compose` service).

## Open threads for next session

- Add a Postgres service to `docker-compose.yml`
- Scaffold a first real Django app
- Decide on a deployment target
