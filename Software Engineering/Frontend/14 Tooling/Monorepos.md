---
title: "Monorepos"
tags: [frontend, frontend/tooling, frontend/architecture]
aliases: [Monorepo, Polyrepo, Nx, Turborepo, pnpm workspaces, Changesets]
updated: 2026-09-27
---
# Monorepos

<Note>
**TL;DR**

A **monorepo** keeps many apps and libraries in **one Git repository**; a **polyrepo** setup gives each its own repo. Monorepos make sharing code, atomic cross-project changes and centralised tooling easy, but they need tooling to stay fast: **workspaces** (pnpm/npm/Yarn/Bun) to link packages, and a **task runner** (Nx, Turborepo) for **caching** and **affected-only** builds. Versioning published packages is handled with **Changesets** (or Nx Release). Monorepos pair naturally with micro-frontends and design systems.
</Note>

## Monorepo vs polyrepo (from *Building Micro-frontends with React*, Chapter 3)

There are many strategies to manage repos — Git submodules, Git subtrees, polyrepos, monorepos, and more.

- **Monorepo**: mono = single — all code lives in **one Git repository** that contains multiple applications; all team members work in that common repo.

![Untitled 53](/Attachments/Untitled%2053.png)

- **Polyrepo**: each app has **its own repository**; teams work across multiple repos, switching as they work on different apps (ideally one at a time).

![Untitled 54](/Attachments/Untitled%2054.png)

### Differences

| Dimension | Polyrepo | Monorepo |
|---|---|---|
| **Team collaboration** | Each team defines its own rules and guidelines — quick and easy, but teams become isolated and must duplicate build pipelines and pre-commit hooks in every repo | Teams are pushed to collaborate and agree on one way to maintain code |
| **Build tools & quality gates** | Each team implements its own build system and quality gates → duplicated effort, higher maintenance; teams with strong leads do well, junior teams need cross-team help | Build tools and quality gates are **centrally managed**; one team's improvements benefit everyone immediately |
| **Code ownership** | Permissions set per repository (who can view/change it) | Everyone can see all code; granular folder-level control via a **`CODEOWNERS`** file (required reviewers per path) |
| **Flexibility** | Highest — each team manages code its own way | **Intentionally restricted** so everyone benefits from shared best practices and tooling |
| **Refactoring code** | Cross-repo refactors are slow: check out every repo and raise a PR in each | Easy: **atomic commits** — one PR changes all affected apps |
| **Ownership of upgrades** | Each team decides when to upgrade libraries/tooling (freedom, but some teams fall far behind) | Upgrade a shared library or tool once, atomically, for all apps (everyone gets the latest; but everyone must be ready) |
| **Codebase size** | Grows gradually per repo | Large from day one, and keeps growing with every app — needs tooling (caching, affected builds, sparse checkouts) to stay fast |

Chapter 2 of the book adds: polyrepos are the easiest to start with and give complete team independence (simpler DevOps), at the cost of isolation and duplicated tooling; monorepos increase collaboration and centralise tooling/automation, with risks of a more complex DevOps setup and the need for folder-level permissions so teams don't overwrite each other's code. For a shared **component library**, you can publish it as an npm package, or — in a monorepo — **build directly from source** (the library lives in `libs/`).

Book notes: [Chapter 3  Monorepo versus Polyrepos for Microfrontends](/Software%20Engineering/Frontend/Courses%20%26%20Books/Building%20Micro-frontends%20with%20React/Chapter%203%20%20Monorepo%20versus%20Polyrepos%20for%20Microfrontends) · [Building Micro-frontends with React](/Software%20Engineering/Frontend/Courses%20%26%20Books/Building%20Micro-frontends%20with%20React/Building%20Micro-frontends%20with%20React)

### Popular monorepo tools (owner's list, updated)
1. **Lerna** — the original JS monorepo tool; package-based. Stewardship was taken over by **Nrwl** (the company behind Nx) in 2022; it now uses Nx under the hood for task running.
2. **Nx** — supports **integrated** monorepos (Nx manages build config through plugins/executors, one root `package.json` possible) **and package-based** ones.
3. **Turborepo** — **package-based**: a fast task runner/cache on top of your package manager's workspaces (by Vercel).
4. Others: **pnpm workspaces** alone (for small repos), **Rush** (Microsoft), **moon**, **Bazel** (polyglot, very large scale).

<Note>
**Package-based vs integrated**

- **Package-based**: every app/lib is a real package with its own `package.json` and scripts; the tool orchestrates `build`/`test` scripts across packages. Low lock-in.
- **Integrated**: the tool owns more (generators, executors, dependency rules, often a single version policy). More consistency and automation, more coupling to the tool.
</Note>

## Why monorepos (and when not)

**Good fit**
- Several apps sharing a design system, API clients, types, utilities.
- Micro-frontends owned by different teams but needing shared tooling and consistent versions.
- Frequent cross-cutting changes (a shared type changes → update BE-for-FE, web and mobile in one PR).
- Single version policy (one React version everywhere).

**Poor fit**
- Truly independent products with different stacks and release processes.
- Organisations that can't invest in CI/tooling — a slow monorepo is worse than polyrepos.
- Strict access-control needs where some code must not be visible to all engineers.

## Structure

```text
travel-platform/
├─ apps/
│  ├─ flights-web/        (Next.js)
│  ├─ hotels-web/         (Vite + React)
│  └─ admin/              (Angular)
├─ packages/
│  ├─ ui/                 (design system: @travel/ui)
│  ├─ api-client/         (@travel/api-client)
│  ├─ config-eslint/      (shared lint config)
│  └─ tsconfig/           (shared tsconfig bases)
├─ pnpm-workspace.yaml
├─ turbo.json   (or nx.json)
├─ package.json
└─ .github/CODEOWNERS
```

{/* diagram source: /diagrams/diagram-335.mmd */}
![diagram](/diagrams/diagram-335.svg)
If `packages/ui` changes → `flights-web` and `hotels-web` are **affected**; `admin` is not.

## pnpm workspaces

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
catalog:                 # one version per dependency across the repo
  react: ^19.2.0
  typescript: ^6.0.0
```

```json
// apps/flights-web/package.json
{
  "name": "flights-web",
  "private": true,
  "dependencies": {
    "@travel/ui": "workspace:*",
    "@travel/api-client": "workspace:*",
    "react": "catalog:"
  }
}
```
- `workspace:*` links the local package (symlink), and is replaced with the real version on publish.
- `catalog:` centralises versions (single version policy).
- Filter commands: `pnpm --filter flights-web dev`, `pnpm --filter "...[origin/main]" test` (changed packages and their dependents), `pnpm -r build` (all, topologically ordered).

## Task runners: caching and affected builds

The core problem: `build` + `lint` + `test` for 30 packages on every PR is too slow. Task runners fix it with:

1. **Task graph** — knows `flights-web#build` depends on `ui#build` (`dependsOn: ["^build"]` = "build my dependencies first"); runs independent tasks in parallel.
2. **Local caching** — hash the inputs (source files, deps, env vars, config); if the hash was seen before, **replay the outputs and logs** instead of running the task.
3. **Remote caching** — share that cache across developers and CI (Nx Cloud / self-hosted, Vercel Remote Cache / self-hosted Turborepo cache): if CI already built `ui` for this hash, your laptop downloads it.
4. **Affected** — only run tasks for projects changed since a base commit, plus their dependents.

{/* diagram source: /diagrams/diagram-336.mmd */}
![diagram](/diagrams/diagram-336.svg)

### Turborepo

```json
// turbo.json
{
  "$schema": "https://turborepo.com/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "!.next/cache/**"]
    },
    "lint": {},
    "test": { "dependsOn": ["^build"], "outputs": ["coverage/**"] },
    "dev": { "cache": false, "persistent": true }
  }
}
```
```bash
turbo run build lint test                      # whole graph, cached
turbo run build --filter=flights-web...        # flights-web and its dependencies
turbo run test --affected                      # only changed packages + dependents
```

### Nx

```json
// nx.json (excerpt)
{
  "targetDefaults": {
    "build": { "dependsOn": ["^build"], "outputs": ["{projectRoot}/dist"], "cache": true },
    "test": { "cache": true }
  },
  "defaultBase": "main"
}
```
```bash
nx run-many -t build lint test          # all projects
nx affected -t build test               # only affected by changes vs main
nx graph                                # interactive project graph
nx g @nx/react:lib feature-seatmap      # generators for consistent scaffolding
```
Nx extras: **module boundary rules** (`@nx/enforce-module-boundaries` — e.g. `type:feature` may depend on `type:ui` but not vice versa), generators, migrations (`nx migrate`) that upgrade framework versions across the repo, distributed task execution in CI, and first-class Module Federation support.

| | Turborepo | Nx |
|---|---|---|
| Model | Package-based | Integrated or package-based |
| Setup | Minimal (`turbo.json`) | More concepts, more power |
| Caching | Local + remote | Local + remote (Nx Cloud or self-hosted) |
| Affected | `--affected`, `--filter` | `nx affected` (project graph aware) |
| Code generation, boundaries, migrations | No | Yes |
| Best for | JS/TS repos wanting speed with little lock-in | Large orgs wanting consistency and guard rails; Angular heritage |

## Versioning and publishing: Changesets
For packages you **publish** (design system, SDK) you need per-package semver and changelogs.

```bash
pnpm changeset          # in your PR: pick changed packages + bump type, write a summary
# → creates .changeset/brave-owls-dance.md
pnpm changeset version  # on release: bumps versions, updates dependents, writes CHANGELOG.md
pnpm changeset publish  # publishes changed packages (use trusted publishing in CI)
```
```md
---
"@travel/ui": minor
"@travel/api-client": patch
---
Add `SeatMap` component; fix retry on 429 in api-client.
```
The Changesets GitHub Action opens a "Version Packages" PR automatically. Alternatives: **Nx Release**, semantic-release (commit-message driven). Internal apps that aren't published usually don't need versions at all — they deploy from `main`.

## Ties to micro-frontends
- A monorepo hosts multiple micro-frontends (multi-SPA or Module Federation remotes) plus the shell, shared design system and contracts — with **independent deployment** still possible per app, driven by affected builds.
- Shared libs can be consumed **from source** (fast iteration, always in sync) or as **versioned packages** (independent upgrades, the polyrepo-like model).
- Watch out for tight coupling: the monorepo makes cross-app imports easy — enforce boundaries (Nx rules, ESLint import restrictions, `exports` maps) so micro-frontends stay independent.
- See [Micro-frontends & Module Federation](/Software%20Engineering/Frontend/09%20Architecture/Micro-frontends%20%26%20Module%20Federation) and [Design Systems](/Software%20Engineering/Frontend/09%20Architecture/Design%20Systems).

## Other practical concerns
- **CODEOWNERS** for review ownership per folder.
- **Shared configs** as packages (`@travel/eslint-config`, `@travel/tsconfig`); Biome 2 supports `"extends": "//"` to inherit the root config in nested packages.
- **TypeScript project references** or per-package builds so type-checking scales.
- **CI**: affected-only pipelines, remote cache, per-app deploy jobs.
- **Git at scale**: shallow clones, sparse checkout, `git maintenance`.
- **Single version policy** vs allowing multiple versions — decide explicitly (pnpm catalogs help).

<Warning>
**Common pitfalls**

- No caching/affected setup → CI time explodes as the repo grows.
- Undeclared cache inputs (env vars, generated files) → stale cache hits and "works on CI, broken in prod".
- Circular dependencies between packages.
- Everything depends on one giant `shared` package → every change affects everything.
- Treating the monorepo as a monolith: apps importing each other's internals.
</Warning>

<Info>
**🆕 2025–2026**

- **pnpm 11** (2026-04-28) and **pnpm 12** (2026-08-26, Rust rewrite) — faster installs for large workspaces, plus supply-chain defaults (1-day `minimumReleaseAge`). See [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security).
- **Biome v2** (2025-06-17) added `"extends": "//"` for monorepo-root config inheritance.
- **Vite+ beta** (2026-07-02) bundles `run`/`check`/`test`/`build`/`pack` in one `vp` CLI aimed at monorepos too — see [Linting, Formatting, Package Managers & Runtimes](/Software%20Engineering/Frontend/14%20Tooling/Linting%2C%20Formatting%2C%20Package%20Managers%20%26%20Runtimes).
- **Vitest 5** (2026-09-03) supports nested projects, useful for per-package test configs.
</Info>

## Interview questions

<Accordion title="Monorepo vs polyrepo — how do you decide?">
Monorepo when apps share a lot of code, need atomic cross-cutting changes and consistent tooling/versions, and the org can invest in caching/affected CI. Polyrepo when teams/products are truly independent, need different stacks or release processes, or strict code visibility separation.
</Accordion>

<Accordion title="How do you keep a big monorepo's CI fast?">
Task graph with parallelism, content-hash caching (local + remote), affected-only runs, distributed execution, and good package boundaries so a change affects few projects.
</Accordion>

<Accordion title="What does `dependsOn: [&quot;^build&quot;]` mean?">
Before running this package's `build`, run `build` in all of its workspace dependencies (the `^` means "upstream dependencies").
</Accordion>

<Accordion title="How is code ownership handled in a monorepo?">
A `CODEOWNERS` file maps folders to teams who must review changes, plus branch protection; Nx module-boundary rules or lint rules prevent unwanted imports.
</Accordion>

<Accordion title="Nx vs Turborepo?">
Turborepo is a lightweight, package-based task runner with caching and filtering. Nx is a fuller platform: project graph, affected commands, generators, migrations, module-boundary enforcement, distributed CI, and it supports both integrated and package-based repos.
</Accordion>

<Accordion title="How do you version packages in a monorepo?">
Changesets: developers add a changeset per PR describing bumps; on release the tool bumps versions, updates internal dependents and changelogs, and publishes. Alternatives: Nx Release, semantic-release. Unpublished apps don't need versions.
</Accordion>

<Accordion title="How do monorepos relate to micro-frontends?">
They're orthogonal but complementary: a monorepo can host many independently deployed micro-frontends with a shared design system and tooling; the trade-off is making sure easy imports don't recreate a coupled monolith.
</Accordion>

## Related
- [Chapter 3  Monorepo versus Polyrepos for Microfrontends](/Software%20Engineering/Frontend/Courses%20%26%20Books/Building%20Micro-frontends%20with%20React/Chapter%203%20%20Monorepo%20versus%20Polyrepos%20for%20Microfrontends)
- [Building Micro-frontends with React](/Software%20Engineering/Frontend/Courses%20%26%20Books/Building%20Micro-frontends%20with%20React/Building%20Micro-frontends%20with%20React)
- [Micro-frontends & Module Federation](/Software%20Engineering/Frontend/09%20Architecture/Micro-frontends%20%26%20Module%20Federation)
- [Design Systems](/Software%20Engineering/Frontend/09%20Architecture/Design%20Systems)
- [Build Tools & Bundlers](/Software%20Engineering/Frontend/14%20Tooling/Build%20Tools%20%26%20Bundlers)
- [Linting, Formatting, Package Managers & Runtimes](/Software%20Engineering/Frontend/14%20Tooling/Linting%2C%20Formatting%2C%20Package%20Managers%20%26%20Runtimes)
- [Supply Chain Security](/Software%20Engineering/Frontend/11%20Security/Supply%20Chain%20Security)

## Sources
- Owner's notes: [Chapter 3  Monorepo versus Polyrepos for Microfrontends](/Software%20Engineering/Frontend/Courses%20%26%20Books/Building%20Micro-frontends%20with%20React/Chapter%203%20%20Monorepo%20versus%20Polyrepos%20for%20Microfrontends) and Chapter 2 of [Building Micro-frontends with React](/Software%20Engineering/Frontend/Courses%20%26%20Books/Building%20Micro-frontends%20with%20React/Building%20Micro-frontends%20with%20React)
- [pnpm – Workspaces](https://pnpm.io/workspaces), [Catalogs](https://pnpm.io/catalogs)
- [Turborepo docs](https://turborepo.com/docs)
- [Nx – Mental model / affected](https://nx.dev/concepts/mental-model), [Nx – Integrated vs package-based](https://nx.dev/concepts/integrated-vs-package-based)
- [Changesets](https://github.com/changesets/changesets)
- [GitHub – About CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
- [monorepo.tools](https://monorepo.tools/)
- [Biome v2](https://biomejs.dev/blog/biome-v2/)
