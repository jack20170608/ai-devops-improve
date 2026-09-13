# Copilot instructions for ai-devops-improve

## Project purpose
This repository is a documentation and research project about AI DevOps and engineering productivity. It is currently a content-first repo rather than a compiled application or service. The main project artifact is `README.md`; future additions should remain consistent with that research/knowledge-management scope.

## Build, test, and lint
No build, test, or lint configuration was found in this repository.

- No `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `Makefile`, `justfile`, or CI workflow was detected.
- There is no automated test suite or formatter/linter setup in the repository today.
- There are no project-specific commands to run for a single test or a full suite; the repo is currently documentation-only.

If code is added later, prefer to adopt the tooling that matches the new project and document the exact commands in the repo's primary docs rather than inventing commands that are not present.

## High-level architecture
- `README.md` is the canonical overview and the repository's main source of project direction.
- `00-ai-common/` contains foundational AI and AI-assisted development research.
- `10-Devops-SOP/` contains the enterprise DevOps standards for AI-assisted project structure, Java, security, and HTTP APIs.
- `.github/instructions/` contains focused Copilot-readable standards for Java, HTTP APIs, configuration security, GitHub Actions, and build dependencies.
- The repo is intentionally lightweight: research notes and prose live at the top level rather than in an application module layout.
- Local editor and session metadata such as `.idea/` and `.remember/` are environment-specific. They are not source-of-truth project structure.
- This repo does not currently have an app/service architecture, package boundaries, or deployment flow to navigate.

## Key conventions
- Keep new work in Markdown unless the repository intentionally expands beyond documentation.
- Preserve the AI DevOps theme and terminology consistently across documents.
- Use MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY consistently in `10-Devops-SOP/`; distinguish protocol requirements from organizational choices.
- When changing enterprise standards, update the relevant index and cite authoritative primary sources.
- Keep the root structure simple and explicit; avoid adding implementation directories without corresponding documentation.
- Treat IDE-generated files and local session artifacts as working state, not project logic.
- If the repo grows beyond research notes, add the relevant toolchain and document commands before relying on them.

## Working expectations
- Do not assume there is a build pipeline, test suite, or linter to run.
- Before proposing a large refactor or new app structure, confirm whether the repo is still documentation-first.
- Favor minimal, explicit changes that fit the repository's research/documentation purpose.
