# HANDOFF-APM: Fork-to-Marketplace Integration

## What This Fork Is

This is a fork of Microsoft's [`dotnet/skills`](https://github.com/microsoft/skills) — a
curated collection of .NET SKILL.md files designed to help AI agents work with
.NET technologies (MSBuild, testing, NuGet, MAUI, ASP.NET, diagnostics, etc.).

### Upstream structure (plugin-based)

Skills are organized into **plugins** under `plugins/`. Each plugin is a named
collection of related skills, agents, and metadata:

```
plugins/
├── dotnet-msbuild/        # MSBuild skills (binlogs, build parallelism, etc.)
├── dotnet-test/           # Testing skills (coverage, test frameworks, etc.)
├── dotnet-diag/           # Diagnostics skills (dump collection, profiling, etc.)
├── dotnet-nuget/          # NuGet skills (packaging, CPM conversion, etc.)
├── dotnet-ai/             # AI integration skills
├── dotnet-aspnet/         # ASP.NET skills (OpenTelemetry, etc.)
├── dotnet-data/           # Data access skills
├── dotnet-maui/           # MAUI UI skills
├── dotnet-template-engine/# Template authoring skills
├── dotnet-upgrade/        # .NET version upgrade skills
├── dotnet-experimental/   # Experimental/prototype skills
└── dotnet/                # Core .NET skills (lsp.json config)
```

Each plugin contains:
```
plugins/<name>/
├── plugin.json            # Manifest: name, version, description, skill/agent lists
├── skills/<skill-name>/   # Individual SKILL.md files
│   └── SKILL.md
├── agents/<agent>.agent.md# Agent definitions
└── training-logs/         # Optional training data
```

### Current fork state (2026-05-13)

The working tree is **mid-restructure**. The committed HEAD has a flat symlink
farm (`skills/<name>/` → `../plugins/<plugin>/skills/<name>/`). The working tree
deletes those flat symlinks and replaces them with categorized symlinks
(`skills/android/`, `skills/apple/`, `skills/dotnet/` with categorized symlinks
inside). The `plugins/` directory (actual skill content) and new categorized
directories are untracked.

This restructure must be stabilized (committed or reverted) before layering APM
on top. See the corresponding entry in `TODO.md`.

## APM Mapping

Each Microsoft plugin maps naturally to an APM package:

| Microsoft | APM Equivalent |
|-----------|---------------|
| `plugins/<name>/plugin.json` | `packages/<name>/apm.yml` |
| `plugins/<name>/skills/*/SKILL.md` | `packages/<name>/.apm/skills/*/SKILL.md` |
| `plugins/<name>/agents/*.agent.md` | `packages/<name>/.apm/agents/*.agent.md` |

The `plugin.json` already contains the fields APM's `apm.yml` needs:
```json
{
  "name": "dotnet-msbuild",
  "version": "0.1.0",
  "description": "Comprehensive MSBuild and .NET build skills...",
  "skills": ["./skills/"],
  "agents": ["./agents/build-perf.agent.md", ...]
}
```

A transform script can mechanically generate `apm.yml` from `plugin.json`.

## Design Decisions

### 1. Separate marketplace repo

The .NET skills will be published as a **standalone APM marketplace** at a new
repo (`apm-dotnet-skills`), separate from `apm-user-repository`. Consumers
register it as a second marketplace. This keeps .NET-specific content isolated
from general-purpose skills like `development-practices` or `architectural-review`.

### 2. Curated on-demand

Not all ~70 skills become APM packages at once. Skills are packaged **when a
consumer needs them**. The first plugin to transform serves as the template for
the rest. This avoids maintaining 70+ packages that nobody is using.

### 3. Transform strategy (deferred)

Two options exist for bridging Microsoft's `plugin.json` format to APM's
`.apm/` package structure. The decision is deferred to implementation time:

- **Script approach:** A Node.js script reads `plugins/*/plugin.json`, generates
  `packages/<name>/apm.yml` and `.apm/` directory structure, and writes the
  output to the marketplace repo. Deterministic, repeatable, debuggable.
  Duplicates files on disk (original plugin files stay untouched in this fork;
  APM copies live in the marketplace repo).

- **Symlink approach:** Created within the marketplace repo or this fork, APM
  package directories use symlinks to point back to the original plugin content.
  No file duplication, always in sync with upstream changes. Must verify that
  APM follows symlinks during `apm install`, `apm pack`, and `apm compile`.
  Test before committing to this approach.

The script approach is recommended by default (deterministic, no symlink
dependency on APM's internals). The symlink approach is a potential optimization
if APM supports it.

### 4. Working tree restructure (deferred)

This fork's working tree is mid-restructure. Before adding any APM files, the
tree must be stabilized:
- **Option A:** Commit the restructure as-is (new categorized symlinks,
  untracked `plugins/` directory, deleted flat symlinks)
- **Option B:** Revert to the last clean HEAD (`git checkout HEAD -- skills/`)
  and work from the flat symlink structure

The decision is deferred to the implementation session. Option A is preferred
if the restructure was intentional and should be preserved.

## Implementation Sequence

When work resumes, execute in this order:

### Phase 1 — Stabilize this fork
1. Decide: commit the working tree restructure, or revert to HEAD
2. Commit or revert, push
3. Working tree is clean — ready for APM layering

### Phase 2 — Create the marketplace repo
1. Create `apm-dotnet-skills` repo under `pngdeity`
2. Scaffold with:
   - `apm.yml` (root marketplace manifest, `tagPattern: "v{version}"`)
   - `.gitignore` (exclude `apm_modules/`, generated artifacts)
   - `.github/workflows/validate.yml` (CI: `apm marketplace check` + `apm pack --dry-run`)
   - `README.md` (marketplace description, install instructions)
   - `LICENSE` (MIT)

### Phase 3 — Choose and implement transform strategy
1. Test APM symlink support (if considering symlink approach)
2. Implement the chosen strategy:
   - Script: `scripts/transform.mjs` reads `plugins/*/plugin.json`, generates
     APM packages under `packages/`
   - Symlink: `scripts/link-packages.sh` creates symlink-based APM packages
3. Document the strategy in `CONTRIBUTING.md`

### Phase 4 — Transform first curated plugin
1. Pick a high-value plugin (e.g., `dotnet-msbuild` — 7 MSBuild skills)
2. Run the transform — produce `packages/dotnet-msbuild/` with:
   - `apm.yml`
   - `.apm/skills/<name>/SKILL.md` for each skill
   - `.apm/agents/<name>.agent.md` for each agent
3. Test: `apm marketplace add <apm-dotnet-skills-url>` then
   `apm install dotnet-msbuild@apm-dotnet-skills` (or local-path workaround)
4. Verify: skill files deploy to `.agents/skills/`, agents to `.agents/agents/`
5. Iterate on the transform script until the output is clean

### Phase 5 — CI and versioning
1. Add `.github/workflows/validate.yml` with:
   - `apm marketplace check` — validates all packages in the manifest
   - `apm pack --dry-run` — verifies all packages are packable
2. Tag `v0.1.0`
3. Push

### Phase 6 — Repeat for additional plugins
As consumers request .NET skills, repeat Phase 4 for the relevant plugins.

## Dependencies

- APM CLI v0.13.0+ (with marketplace remote install fix, or local-path workaround)
- This fork (`pngdeity/skills`) tracking upstream `microsoft/skills`
- Node.js 18+ (for transform script, if script approach chosen)
- `js-yaml` (npm) — if transform script needs YAML generation

## Notes

- The upstream Microsoft repo (`microsoft/skills`) must be added as a second
  remote so changes can be pulled: `git remote add upstream https://github.com/microsoft/skills.git`
- After each upstream sync, re-run the transform script to update the marketplace
  packages
- The `plugin.json` format may change upstream — the transform script should
  validate against a known schema and fail loudly on unrecognized fields
- This fork's AGENTS.md (21 lines) contains build/validate instructions for the
  .NET Agent Skills repo — it should remain as-is unless the repo adopts APM as
  a consumer (separate concern from marketplace publishing)
- The existing `gemini-extension.json` and `skills.json` are vendor-specific
  manifests — they coexist with APM marketplace metadata without conflict
