# OpenWiki: Architecture, Implementation & Operational Reference

OpenWiki (developed by LangChain) is an open-source agentic documentation engine that generates, links, and continuously maintains structured Markdown documentation for codebases and personal knowledge graphs. Instead of saturating LLM context windows or packing monolithic `AGENTS.md` / `CLAUDE.md` files, OpenWiki builds a curated, cross-linked wiki designed for on-demand query resolution via the Model Context Protocol (MCP).

---

## 1. Operating Modes: Code Mode vs. Personal Mode

| Dimension | Code Mode (`openwiki`) | Personal Mode (`openwiki personal`) |
|---|---|---|
| **Storage Root** | `<repo-root>/openwiki/` | `~/.openwiki/wiki/` (User Home Directory) |
| **Audience** | Entire engineering team, CI/CD, repository-scoped AI agents | Personal workstation, individual developer |
| **Version Control** | Committed directly into project Git repository | Local only (private; excluded from project Git) |
| **Ingestion Scope** | Current repository tree, manifests, entrypoints, and configs | Notion, Gmail, local directories, web feeds, linked Git repos |
| **Fact Verification** | Strict Grounded Claims (`.claims/`) tied to Git SHAs and diffs | Broad semantic entity resolution across disparate sources |
| **Target Question** | *"How does this specific repository or service function?"* | *"What do I know about X across all my notes, projects, and apps?"* |

> **Note on Initialization:** Both modes are completely independent. Run `openwiki --init` to document a project repository. Run `openwiki personal --init` only if you wish to construct a private cross-cutting second brain across multiple life and work streams.

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

## 8. CLI Command Quick Reference

| Command | Description |
|---|---|
| `npm install -g openwiki` | Install the OpenWiki CLI globally. |
| `openwiki --init` | Discover repo structure, generate `openwiki/`, and create `.claims/`. |
| `openwiki --update` | Run incremental drift check against git diffs and update invalidated claims. |
| `openwiki visualize` | Launch local interactive graph viewer and Markdown browser. |
| `openwiki integrations install <target>` | Register MCP tools with `claude`, `cursor`, `copilot`, or `codex`. |
| `openwiki link` | Group multiple repository wikis into a unified federated workspace. |
| `openwiki workspace use <name>` | Select the active federated multi-repo workspace. |
| `openwiki personal --init` | Initialize personal knowledge graph at `~/.openwiki/wiki/`. |
| `openwiki personal --update` | Synchronize personal knowledge connectors (Notion, Gmail, local sources). |