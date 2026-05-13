# skills — Fork of Microsoft dotnet/skills

This file tracks tasks for maintaining this fork and integrating it with APM.

---

## Deferred: APM Marketplace Integration

- [ ] **Stabilize working tree** — The repo is mid-restructure. The committed HEAD
  has a flat symlink farm (`skills/<name>/` → `../plugins/<plugin>/skills/<name>/`).
  The working tree deletes those flat symlinks and creates categorized alternatives
  (`skills/android/`, `skills/apple/`, `skills/dotnet/` with categorized symlinks
  inside). The `plugins/` directory (actual skill content) and new categorized
  directories are untracked. Before layering APM: either commit the restructure
  as-is (preferred if intentional), or revert to the last clean HEAD
  (`git checkout HEAD -- skills/`). See `git status` for the full diff.

- [ ] **Add upstream remote** — The Microsoft upstream is not configured.
  Run: `git remote add upstream https://github.com/microsoft/skills.git`
  This enables pulling upstream changes to keep the fork in sync.

- [ ] **Create apm-dotnet-skills marketplace repo** — Separate from
  `apm-user-repository`. Consumers register it as a second marketplace.
  Scaffold with:
  - `apm.yml` (root marketplace manifest, `tagPattern: "v{version}"`)
  - `.gitignore` (exclude `apm_modules/`, generated artifacts)
  - `.github/workflows/validate.yml` (CI: `apm marketplace check` + `apm pack --dry-run`)
  - `README.md` (marketplace description, install instructions)
  - `LICENSE` (MIT)
  See `HANDOFF-APM.md` for the full implementation sequence.

- [ ] **Choose transform strategy** — Two options for mapping Microsoft's
  `plugins/<name>/plugin.json` format to APM's `packages/<name>/apm.yml` format:
  - **Script:** Node.js script reads `plugin.json`, generates APM package
    structure. Deterministic, repeatable. Duplicates files on disk but original
    plugins stay untouched. Recommended default.
  - **Symlink:** APM package directories symlink back to plugin content. No
    duplication. Must verify APM follows symlinks during install/pack.
  Decision rationale in `HANDOFF-APM.md`.

- [ ] **Curate first plugin** — On demand, when a consumer needs a .NET skill.
  Pick one plugin (e.g., `dotnet-msbuild` — 7 MSBuild skills), apply the
  transform strategy, test `apm install` from the new marketplace, iterate on
  the process until the output is clean.

- [ ] **CI and versioning** — Add `.github/workflows/validate.yml` (apm
  marketplace check + apm pack --dry-run). Tag `v0.1.0`. Push.

- [ ] **Repeat for additional plugins** — As consumers request .NET skills,
  transform the relevant plugins. Curated on-demand — not all ~70 skills at once.
