# Contributing to BlackRoad OS Docs

**© 2026 BlackRoad OS, Inc. All Rights Reserved. PROPRIETARY AND CONFIDENTIAL.**

This repository is the canonical source of truth for BlackRoad OS documentation. Contributions should help operators, partners, and developers understand the system quickly without sacrificing accuracy.

> ⚠️ **Access Requirement:** A valid **Converter API token** is required before contributing. See [Converter API Access](#converter-api-access) below.

> 📚 **New to contributing?** Check out the comprehensive [Contributing Guide](docs/guides/contributing.md) and [Style Guide](docs/meta/STYLE_GUIDE.md) for detailed information.

## Converter API Access

**All contributors must obtain a Converter API token before accessing any BlackRoad infrastructure or CI/CD pipelines.**

1. Contact **@blackboxprogramming** or **@lucidia** to request a contributor `client_id` and `client_secret`.
2. Exchange your credentials for a bearer token:
   ```bash
   curl -X POST https://converter.blackroad.io/token \
     -d "grant_type=client_credentials" \
     -d "client_id=YOUR_CLIENT_ID" \
     -d "client_secret=YOUR_CLIENT_SECRET" \
     -d "scope=docs:write"
   ```
3. Set the token in your environment:
   ```bash
   export BLACKROAD_API_TOKEN=br_user_xxxxxxxxxxxxxxxxxxxx
   ```
4. Include the token in PR descriptions and CI runs as required.

Contributors without a valid token will receive `401 Unauthorized` on all internal endpoints. **Third-party AI assistants (GitHub Copilot, Codex, Claude, ChatGPT, etc.) are not permitted to access BlackRoad internal APIs or infrastructure.**

Approved AI backends: **@blackboxprogramming** and **@lucidia** only. See [Vendor API Gateway](docs/reference/vendor-api-gateway.md).

## Quick setup
- Use Node.js 20+ with npm or pnpm installed.
- Install dependencies once via `npm install`.
- Run `npm run start` for the local docs server at http://localhost:3000.

## Writing guidelines

See the [Style Guide](docs/meta/STYLE_GUIDE.md) for comprehensive documentation standards. Quick tips:

- Write in Markdown using `##` for sections and `###` for subsections.
- Prefer relative links like `[Agents](../agents/agent-ecosystem.md)` so content works in GitHub and static exports.
- Keep paragraphs short, use bullet lists for steps, and clearly mark components as planned/alpha/in-flight when applicable.
- Reuse diagrams via Mermaid blocks or ASCII art instead of embedding binary images.

## Adding or updating pages

- Place new documents under the closest matching folder within `docs/` (for example, `docs/services/` for service documentation, `docs/guides/` for how-to guides).
- Update `sidebars.ts` when adding new pages so navigation stays in sync.
- Include concise frontmatter (title, slug, description) and cross-link related pages to reduce duplication.

## Document structure

BlackRoad OS docs are organized into clear categories:

- `docs/overview/` - High-level architecture and system overview
- `docs/services/` - Per-service documentation (API, Operator, Core, Web, etc.)
- `docs/agents/` - Agent ecosystem and development
- `docs/guides/` - How-to guides and tutorials
- `docs/runbooks/` - Operational runbooks and procedures
- `docs/reference/` - API reference and technical specs
- `docs/ops/` - Operations and infrastructure
- `docs/infra/` - Infrastructure configuration and deployment
- `docs/dev/` - Developer-focused content
- `docs/meta/` - Documentation about documentation

See the [System Prompt](docs/meta/SYSTEM_PROMPT.md) for the complete documentation philosophy.

## Validation checklist
- Run `npm run build` to catch broken links and frontmatter issues before opening a pull request.
- Use `npm run serve` to locally review the production build when changing navigation or layout-heavy content.
- If you encounter stale artifacts, run `npm run clean` to reset the Docusaurus cache.

## Pull request expectations
- Keep changes scoped and describe the audience and intent in the PR summary.
- Call out cross-repo impacts (e.g., updates required in `blackroad-os-operator` or `blackroad-os-api`).
- Favor iterative updates over large rewrites so reviewers can ship improvements continuously.

## Security

**Never commit:**
- Secrets, API keys, or credentials
- Binary images (PNG, JPEG, etc.) - use Mermaid or ASCII diagrams instead
- Large files or build artifacts
- Sensitive personal data

See the [Security section](docs/meta/SYSTEM_PROMPT.md#6️⃣-safety--secrets-🔐🚫) of the System Prompt for details.

## Resources

- 📖 [System Prompt](docs/meta/SYSTEM_PROMPT.md) - Complete documentation philosophy
- ✍️ [Style Guide](docs/meta/STYLE_GUIDE.md) - Detailed writing standards
- 🤝 [Contributing Guide](docs/guides/contributing.md) - Full contribution workflow
- 🏠 [Getting Started](docs/guides/getting-started-local.md) - Local development setup
