# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Single-file HTML slide deck (`devops-agentic-workflow-tf-aws-webslides.html`) for a live/talk titled "DevOps com Agentes de IA · Claude Code, Terraform e AWS" (Portuguese, pt-BR). No build system, no package manager, no tests — it's a self-contained static HTML file with inline `<style>` and `<script>`, plus Google Fonts loaded via CDN link tags.
Will be deployed to AWS using S3 + CloudFront, provisioned with Terraform, and automated via GitHub Actions.

## Running it

Open the HTML file directly in a browser (or serve the directory with any static file server, e.g. `python3 -m http.server`). There is no build/lint/test step.

## Architecture

### Application (Static Site)
Everything lives in one file, in three parts:

1. **`<style>` (head)** — CSS custom properties define the theme (`--bg`, `--accent`, `--accent-2`, `--accent-3`, fonts). Slide layout uses absolutely-positioned `.slide` panels inside a full-viewport `.deck` container, cross-faded via `.active`/`.leaving` classes.
2. **`<body>`** — a sequence of `<section class="slide" data-phase="...">` elements, one per slide. `data-phase` groups slides into named phases (`intro`, `0`–`4`), each mapped to a display label in the script's `phaseNames` object (e.g. `'1':'FASE 01 · FUNDAMENTOS'`). Each slide typically contains a `.slide-head` (with an `.eyebrow` phase label) and a `.slide-body`. There's also a HUD (progress bar, slide counter `hudCount`, phase tag `phaseTag`) and prev/next controls (`btnPrev`, `btnNext`, `zoneLeft`, `zoneRight` click zones).
3. **`<script>` (end of body, IIFE)** — a tiny hand-rolled deck engine: `slides` array queried from the DOM, `idx` tracks current slide, `goTo`/`next`/`prev` mutate `idx` and call `render()`, which toggles `.active`/`.leaving` classes, updates the HUD, and syncs the URL hash (`#s<n>`, 1-indexed) via `history.replaceState`. Deep-linking on load reads `#s<n>` from the URL. Keyboard nav: arrows/PageUp/PageDown/Space to navigate, Home/End to jump to first/last slide, `f` to toggle fullscreen. The engine is exposed on `window.__deck` for console debugging (`__deck.goTo(n)`, `.next()`, `.prev()`, `.idx`, `.total`).
- Pure HTML5 + CSS3, no JavaScript, no build step

### Infrastructure (`terraform/`)
- AWS S3 bucket for static site hosting (private, OAC-based access)
- CloudFront distribution as CDN with S3 origin
- GitHub OIDC provider + IAM role for keyless CI/CD auth
- Terraform state stored in S3 backend with DynamoDB locking
- All resources tagged with `Project` and `Environment`

### CI/CD (`.github/workflows/`)
- GitHub Actions workflow triggers on push to `main`
- Syncs site files to S3, then invalidates CloudFront cache
- Uses OIDC for AWS authentication (no long-lived keys)

## MCP Servers (`.mcp.json`)

Two MCP servers are configured for Claude Code:
- **aws** (`awslabs.aws-api-mcp-server`) — Direct AWS API access for querying and managing resources
- **terraform** (`hashicorp/terraform-mcp-server`) — Terraform operations via Docker, workspace mounted at `/workspace`

AWS credentials and region are configured in `.claude/settings.local.json` (gitignored), not in `.mcp.json`. This keeps secrets out of version control and provides a single source of truth for all tools.

## Custom Agents (`.claude/agents/`)

This project has 4 specialized subagents. Use them by name when delegating tasks:
- **tf-writer** — generates Terraform code (has Write access + project memory)
- **security-auditor** — audits TF for security issues (Read-only, Sonnet)
- **cost-optimizer** — reviews infra cost (Read-only, Haiku)
- **drift-detector** — detects state drift (Bash, Haiku)

## Skills (`.claude/skills/`)

All infrastructure and deployment tasks are handled via skills. Do not write Terraform or CI/CD code manually — use the appropriate skill. Action skills have `disable-model-invocation: true` (manual only). The `project-scope` skill has `user-invocable: false` (auto-loaded by Claude as background knowledge).

```
/scaffold-terraform [region] [name]  → Generate all Terraform files (uses tf-writer agent)
/scaffold-cicd [aws-account-id]      → Generate GitHub Actions + OIDC IAM role
/tf-plan                             → Run terraform plan + risk analysis
/tf-apply                            → Run terraform apply + verify
/deploy                              → Sync S3 + invalidate CloudFront
/infra-status                        → Health dashboard of all resources
/infra-audit                         → Parallel security + cost + drift audit (forked context)
/setup-gh-actions [create|validate]  → Create or validate CI workflow
/tf-destroy                          → Safe destroy with confirmation
project-scope                        → Background knowledge: AWS service constraints (auto-loaded)
/commit                              → Auto-generate commit message (built-in)
/compact                             → Compress long conversation context (built-in)
```

## Commands

```bash
# Terraform
cd terraform && terraform init
cd terraform && terraform plan
cd terraform && terraform apply

# Local preview
open index.html

# Manual S3 sync (CI does this automatically)
aws s3 sync . s3://$BUCKET_NAME --exclude "terraform/*" --exclude ".git/*" --exclude ".github/*" --exclude "*.md" --exclude ".claude/*"
```

## Safety Layers
1. **UserPromptSubmit hook** — catches destructive intent ("delete all", "nuke", "wipe") before Claude starts
2. **PreToolUse hook** — blocks dangerous commands (terraform destroy, aws s3 rm) at execution time
3. **Permissions** — auto-allows safe reads, blocks IAM and rm -rf
4. **PostToolUse hook** — logs all terraform apply executions to `.claude/deploy.log`

## Conventions
- Terraform files use `terraform/` directory with standard layout (main.tf, variables.tf, outputs.tf)
- GitHub Actions uses OIDC — no stored AWS access keys
- All infrastructure changes go through Terraform — never modify AWS resources manually
- Site content changes deploy automatically via GitHub Actions on push to main
