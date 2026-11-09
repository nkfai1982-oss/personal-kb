# OpenWiki: Architecture, Implementation & Operational Reference

OpenWiki (developed by LangChain) is an open-source agentic documentation engine that generates, links, and continuously maintains structured Markdown documentation for codebases and personal knowledge graphs. Instead of saturating LLM context windows or packing monolithic `AGENTS.md` / `CLAUDE.md` files, OpenWiki builds a curated, cross-linked wiki designed for on-demand query resolution via the Model Context Protocol (MCP).

---

## 1. Operating Modes: Code Mode vs. Personal Mode

| Dimension | Code Mode (`openwiki`) | Personal Mode (`openwiki personal`) |
| :--- | :--- | :--- |
| **Storage Root** | `<repo-root>/openwiki/` | `~/.openwiki/wiki/` (User Home Directory) |
| **Audience** | Entire engineering team, CI/CD, repository-scoped AI agents | Personal workstation, individual developer |
| **Version Control** | Committed directly into project Git repository | Local only (private; excluded from project Git) |
| **Ingestion Scope** | Current repository tree, manifests, entrypoints, and configs | Notion, Gmail, local directories, web feeds, linked Git repos |
| **Fact Verification** | Strict Grounded Claims (`.claims/`) tied to Git SHAs and diffs | Broad semantic entity resolution across disparate sources |
| **Target Question** | *"How does this specific repository or service function?"* | *"What do I know about X across all my notes, projects, and apps?"* |

**Note on Initialization:** Both modes are completely independent. Run `openwiki --init` to document a project repository. Run `openwiki personal --init` only if you wish to construct a private cross-cutting second brain across multiple life and work streams.

---

## 2. Code Relationship Discovery: Agentic vs. AST

OpenWiki **does not use Abstract Syntax Tree (AST) parsers** (e.g., Tree-sitter, Babel, compiler frontends) or deterministic language server symbol tables. Instead, it relies on **agentic semantic discovery**.

### How It Discovers Structure Without an AST

* **Inspection Toolset:** The planner and authoring agents are equipped with non-destructive workspace tools: `ls`, `glob`, `grep`, and `read_file`.
* **Execution Flow:**
  1. Identifies top-level manifests (`package.json`, `pom.xml`, `build.gradle`, `pyproject.toml`, `Cargo.toml`) and directory layouts.
  2. Inspects key entrypoints and configuration files.
  3. Uses iterative `grep` and regex traces to track imports, route definitions, interface implementations, and event subscribers across files.
* **Visualization Graph (`src/visualize/graph.ts`):** The interactive node graph rendered by `openwiki visualize` does not parse code syntax trees. It builds nodes and edges by scanning the Open Knowledge Format (OKF v0.2) front matter and relative Markdown links (`[Module B](../modules/b.md)`) authored by the worker agents.

### Advantages for Legacy Enterprise Stacks (Struts, Spring XML, EJB, iBatis)

Static AST/LSP parsers analyze code in single-language silos and often break down when code relationships are wired via XML configuration files. OpenWiki handles these environments effectively because the LLM treats XML and Java artifacts with equal weight:

* **Apache Struts (1.x / 2.x):** Correlates `<action path="/login" type="com.app.LoginAction">` in `struts-config.xml` to Java action classes, forward paths, and JSP view pages.
* **Spring XML DI & Beans:** Maps `<bean id="userService" class="com.app.UserServiceImpl">` and associated `<property ref="userDao"/>` injection definitions directly to target Java interfaces and classes.
* **Enterprise JavaBeans (EJB 2.x / 3.x):** Parses `ejb-jar.xml` and vendor deployment descriptors (`jboss.xml`, `weblogic-ejb-jar.xml`), mapping home/remote interfaces to session bean implementations.
* **iBatis / MyBatis:** Traces `SqlMapConfig.xml` / `mybatis-config.xml` and mapper XML namespaces to Java DAO/Mapper interfaces, SQL statements (`<select id="...">`), and POJO result mappings.

---

## 3. Grounded Claims & Drift Tracking (`.claims/`)

To prevent hallucinations, minimize unnecessary token consumption, and avoid churn:

* **Evidence Pointers:** Every factual assertion authored into the wiki is backed by sidecar claims in `openwiki/.claims/` containing URI pointers, line numbers, commit SHAs, and file content hashes (e.g., `repo://src/main/resources/applicationContext.xml#L45-L72`).
* **Deterministic Diff Checks (`openwiki --update`):** During updates, OpenWiki inspects Git diffs against `.last-update.json` and existing claims.
* **Zero-Token No-Op:** If source code edits do not invalidate any recorded claims or architectural boundaries, OpenWiki completes without making external LLM calls.

---

## 4. Taxonomy & Custom Directory Structures

By default, the Planner Agent generates an Open Knowledge Format (OKF) layout:

* `openwiki/architecture/`: System design, module maps, component data flows.
* `openwiki/concepts/`: Domain logic, entities, schemas, lifecycle rules.
* `openwiki/integrations/`: Third-party APIs, database connectors, runtime hosts.
* `openwiki/operations/`: CI/CD workflows, environment parameters, operational tasks.
* `openwiki/quickstart.md`: Fixed landing point registered into `AGENTS.md` / `CLAUDE.md`.

### Overriding the Default Hierarchy

To force a custom taxonomy (e.g., domain-driven, by microservice, or flat):

1. Create `openwiki/INSTRUCTIONS.md` at the repository root. This file is **user-owned and never overwritten** by OpenWiki.
2. Define the explicit folder structure and documentation rules:

```markdown
# Documentation Rules
Organize the wiki strictly into:
- `modules/`: Internal package APIs and responsibilities.
- `domain/`: Business entities and transaction workflows.
- `contracts/`: REST payloads, event schemas, and database tables.
Do not generate `concepts/` or `operations/`.
```

3. Existing Markdown files can be manually renamed or moved; OpenWiki preserves relocated paths on subsequent `--update` passes as long as front matter remains intact.

---

## 5. Query Handling & Documentation Update Lifecycle

OpenWiki enforces a **"wiki-first, code-second"** pattern:

1. **Agent Search:** A coding assistant (Claude Code, Cursor, Copilot) queries OpenWiki's MCP tools (`openwiki_search`, `openwiki_read`) to find existing architectural summaries.
2. **Codebase Fallback:** If the answer is not documented in the wiki, the assistant falls back to workspace inspection tools (`grep`, file reads) to answer the query directly from source code.
3. **No Automatic Writes on Query:** Querying does **not** dynamically append notes to `.md` files. This prevents wiki sprawl, preserves claim verification integrity, and avoids injecting ungrounded conversational summaries.

### How to Add Missing Documentation

* **In-Chat Directive:** Instruct the host agent:
  > *"Create a new page `openwiki/architecture/auth-flow.md` explaining the authentication logic you just discovered, add cross-links in `openwiki/quickstart.md`, and format it to OKF standards."*
* **Declarative Prompting:** Add the scope requirement to `openwiki/INSTRUCTIONS.md`, then execute:

```bash
openwiki --update
```

---

## 6. Microservices & Multi-Component Deployments

### Separate Repositories (Polyrepo) -> Use `openwiki link`

1. Initialize documentation independently in each service:

```bash
cd ~/projects/auth-service && openwiki --init
cd ~/projects/order-service && openwiki --init
```

2. Link services into a federated workspace:

```bash
cd ~/projects
openwiki link # Scans subdirectories containing openwiki/ and creates a named workspace
openwiki workspace use <WorkspaceName>
```

3. **Outcome:** Agents querying via MCP gain federated cross-service search capabilities (`openwiki_search`), querying all member wikis simultaneously without requiring hardcoded cross-repo Markdown links.

### Monorepo Setup -> Do NOT use `openwiki link`

* Run `openwiki --init` once at the monorepo root.
* Define component boundaries in `openwiki/INSTRUCTIONS.md` so the planner generates distinct architectural modules per service directory.

---

## 7. Ingesting Legacy Docs & Enterprise Business Sources

### Ingesting Stale Docs for Business Context & Ubiquitous Language

Outdated architecture documents, functional specs, or glossaries can provide business terminology while code provides technical implementation truth:

1. Place legacy documentation in a dedicated directory (e.g., `docs/legacy/`).
2. Add reconciliation directives to `openwiki/INSTRUCTIONS.md`:

```markdown
# Legacy Reconciliation
- Read `docs/legacy/business-glossary.md` for domain terminology and ubiquitous language.
- Reconcile legacy architecture against current implementations in `src/`.
- If legacy documentation conflicts with current source code, document the actual code behavior while preserving the original business terms in `openwiki/concepts/glossary.md`.
```

### Connecting Confluence & SharePoint

OpenWiki integrates with Confluence and SharePoint through three patterns:

1. **Host-Driven MCP Authoring (Recommended for Interactive Work):**
   * Configure a Confluence or SharePoint MCP server in your local editor config (Cursor, Claude Code, Copilot).
   * Instruct the agent to read the Confluence spaces for business requirements while inspecting the local repository to write grounded OpenWiki pages.
2. **Export-and-Ground Pipeline (Recommended for CI/CD):**
   * Export relevant Confluence spaces or SharePoint specs as Markdown/HTML into `docs/specs/`.
   * Reference the exported path in `openwiki/INSTRUCTIONS.md`.
   * Run `openwiki --init` or `openwiki --update`. This avoids hardcoding live corporate credentials in CI pipelines.
3. **Personal Mode `custom-mcp` Connector:**
   * Configure `~/.openwiki/connectors/custom-mcp/config.json` pointing to an MCP server (e.g., `@modelcontextprotocol/server-confluence` or Microsoft Graph MCP).
   * Run `openwiki personal --update` to pull external enterprise documents into your local personal graph.

---

## 8. Running OpenWiki in GitHub Actions with GitHub Copilot

`langchain-ai/openwiki` natively supports **GitHub Copilot** as a model provider (`OPENWIKI_PROVIDER=copilot`) and provides an official GitHub Actions workflow template (`examples/openwiki-update.yml`) for running `openwiki code --update --print` in CI.

### How OpenWiki's Copilot Support Works

OpenWiki integrates with GitHub Copilot in two ways:

1. **Direct Model Provider (`OPENWIKI_PROVIDER=copilot`):** Used by the standalone CLI (`openwiki --init` / `openwiki code --update --print`). Locally, it reads your authenticated session from the GitHub CLI (`gh auth login`) credential store. In headless CI environments, it reads `COPILOT_API_KEY` (and optional `COPILOT_BASE_URL`) from environment variables. Note that `openwiki code --update` automatically initializes the `openwiki/` documentation directory on its first run if it does not exist yet.
2. **Host Coding-Agent Integration (`openwiki integrations install copilot`):** Wires OpenWiki skills and MCP tools into interactive GitHub Copilot CLI sessions on a developer's machine (not intended for headless CI).

### Step 1: Obtain a GitHub OAuth Token for `COPILOT_API_KEY`

The GitHub Copilot API rejects standard Personal Access Tokens (`ghp_...` classic or `github_pat_...` fine-grained) and the default GitHub Actions `GITHUB_TOKEN`. You must supply a GitHub OAuth token (`gho_...`):

1. Authenticate locally with the GitHub CLI:

```bash
gh auth login
```

2. Print your active OAuth token (starts with `gho_`):

```bash
gh auth token
```

3. In your GitHub repository, navigate to **Settings -> Secrets and variables -> Actions** and create a repository secret named `COPILOT_API_KEY` with that `gho_...` token.
4. Navigate to **Settings -> Actions -> General -> Workflow permissions**, select **Read and write permissions**, and enable **Allow GitHub Actions to create and approve pull requests**.

### Step 2: Configure `.github/workflows/openwiki-update.yml`

```yaml
name: OpenWiki Update

on:
  workflow_dispatch:
  schedule:
    # Runs daily at 08:00 UTC (midnight PST)
    - cron: "0 8 * * *"

permissions:
  contents: write
  pull-requests: write

jobs:
  update-wiki:
    runs-on: ubuntu-latest
    steps:
      - name: Check out repository
        uses: actions/checkout@v4
        with:
          persist-credentials: true
          # Full history is required so `openwiki code --update` can diff HEAD
          # against the last documented commit.
          fetch-depth: 0

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"

      - name: Install OpenWiki
        # mermaid + jsdom are optional; they validate Mermaid diagrams in the wiki
        run: npm install --global openwiki mermaid@11.16.0 jsdom@29.1.1

      - name: Run OpenWiki
        id: openwiki
        continue-on-error: true
        run: openwiki code --update --print
        env:
          OPENWIKI_PROVIDER: copilot
          COPILOT_API_KEY: ${{ secrets.COPILOT_API_KEY }}
          OPENWIKI_MODEL_ID: gpt-5.5
          # Optional: speed up multi-page generation (1 to 8 workers)
          # OPENWIKI_PAGE_CONCURRENCY: "3"
          # Optional: disable telemetry in CI
          # OPENWIKI_TELEMETRY_DISABLED: "1"

      - name: Remove transient OpenWiki run state
        if: ${{ !cancelled() }}
        run: rm -f -- openwiki/.run.json

      - name: List OpenWiki update paths
        id: paths
        if: ${{ !cancelled() }}
        run: |
          paths=openwiki,AGENTS.md,.github/workflows/openwiki-update.yml
          if [ -e CLAUDE.md ]; then
            paths="$paths,CLAUDE.md"
          fi
          echo "list=$paths" >> "$GITHUB_OUTPUT"

      - name: Create OpenWiki update pull request
        id: create-pr
        if: ${{ !cancelled() }}
        uses: peter-evans/create-pull-request@v7
        with:
          add-paths: ${{ steps.paths.outputs.list }}
          branch: openwiki/update
          commit-message: "docs: update OpenWiki"
          title: "docs: update OpenWiki"
          body: |
            Automated OpenWiki documentation update.

            OpenWiki result: ${{ steps.openwiki.outcome }}

            When the result is `failure`, this PR intentionally preserves only the
            pages completed before the failure. Merge it to make that progress the
            baseline for the next scheduled run.

      - name: Propagate OpenWiki failure
        if: ${{ steps.openwiki.outcome == 'failure' }}
        run: exit 1
```

### Key CI Operational Details

* **`fetch-depth: 0`:** Mandatory in `actions/checkout`. OpenWiki tracks the commit SHA of its last successful run and diffs against `HEAD` to update only affected pages and stale Grounded Claims.
* **`--print` Flag:** Required in CI (`openwiki code --update --print`) so the CLI runs non-interactively, outputs the final summary to stdout, and exits cleanly.
* **Partial Failure Recovery:** Setting `continue-on-error: true` on the OpenWiki step, removing `openwiki/.run.json`, and opening a PR preserves completed pages even if the run hits a rate limit partway through.
* **Node.js Requirement:** OpenWiki requires **Node.js 22.22.0 or newer**.

---

## 9. CLI Command Quick Reference

| Command | Description |
| :--- | :--- |
| `npm install -g openwiki` | Install the OpenWiki CLI globally. |
| `openwiki --init` | Discover repo structure, generate `openwiki/`, and create `.claims/`. |
| `openwiki --update` | Run incremental drift check against git diffs and update invalidated claims. |
| `openwiki code --update --print` | Run non-interactive incremental wiki update (recommended for CI/CD and GitHub Actions). |
| `openwiki visualize` | Launch local interactive graph viewer and Markdown browser. |
| `openwiki integrations install <target>` | Register MCP tools with `claude`, `cursor`, `copilot`, or `codex`. |
| `openwiki link` | Group multiple repository wikis into a unified federated workspace. |
| `openwiki workspace use <name>` | Select the active federated multi-repo workspace. |
| `openwiki personal --init` | Initialize personal knowledge graph at `~/.openwiki/wiki/`. |
| `openwiki personal --update` | Synchronize personal knowledge connectors (Notion, Gmail, local sources). |
