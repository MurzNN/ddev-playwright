# AGENTS.md

This file provides guidance to AI agents when working with the **ddev-playwright** add-on repository.

For general DDEV agent instructions and conventions, see the [organization-wide AGENTS.md](https://github.com/ddev/.github/blob/main/AGENTS.md).

## Project Overview

This repository is a [DDEV add-on](https://docs.ddev.com/en/stable/users/extend/additional-services/) that installs [Playwright](https://playwright.dev/) and browser binaries in the DDEV `web` container. It also configures X11 forwarding so headed browsers (Playwright headed mode, Chrome, etc.) can display on the host.

## Key Files

| File | Purpose |
|------|---------|
| `install.yaml` | Add-on manifest: name, `project_files`, DDEV version constraint |
| `config.playwright.yaml` | Hooks, `web_environment` (X11/GUI), Playwright install on container start |
| `docker-compose.playwright.yaml` | Shared Playwright browser cache mount and `/tmp/.X11-unix` bind mount |
| `web-build/Dockerfile.playwright` | Installs Playwright system dependencies at image build time |
| `tests/test.bats` | Bats integration tests (primary testing strategy) |

## Development Workflow

1. Edit add-on files in the repository root.
2. Run tests locally from the repo root:
   ```bash
   bats ./tests/test.bats
   ```
3. Exclude release tests while iterating:
   ```bash
   bats ./tests/test.bats --filter-tags '!release'
   ```
4. Manual smoke test in a throwaway project:
   ```bash
   mkdir -p ~/tmp/playwright-test && cd ~/tmp/playwright-test
   ddev config --project-name=playwright-test
   ddev add-on get /path/to/ddev-playwright
   ddev restart
   ddev npx playwright install --list
   ```

## Testing

`tests/test.bats` installs the add-on into a temporary DDEV project and runs `health_checks()`, which verifies:

- Playwright browsers are installed and listed via `npx playwright install --list`
- `DISPLAY=:0` is set in the `web` container
- `LIBGL_ALWAYS_SOFTWARE=1` is set in the `web` container
- `config.playwright.yaml` contains the X11 pre-start hook and software rendering env var

CI runs via `.github/workflows/tests.yml` using `ddev/github-action-add-on-test@v2` against stable and HEAD DDEV versions.

## Configuration Notes

- **X11 forwarding**: `pre-start` runs `xhost +local:docker` on the host; `docker-compose.playwright.yaml` bind-mounts `/tmp/.X11-unix`.
- **Software rendering**: `LIBGL_ALWAYS_SOFTWARE=1` avoids Chromium/Mesa hanging on GPU/DRM inside Docker when no window appears.
- **macOS rootless Docker**: When `/tmp/.X11-unix` bind-mounting is not possible, use the commented alternative in `config.playwright.yaml` with `DISPLAY=host.docker.internal:0`.

## Important Reminders

- Do not commit secrets or local `.ddev/` project state from test runs.
- Keep changes focused on add-on behavior; this repo does not contain application code.
- Update `tests/test.bats` when adding or changing configuration that should be validated in CI.
