# Contributing to `AgriLogy/.github`

This repository holds **org-wide GitHub assets**, not product code. Nothing here
is built, deployed, or imported — but every file is *rendered by GitHub* on the
org profile or *inherited* by every other AgriLogy repository.

> Looking for build/test commands? They are **not here.** Each product repo
> (`agri-api`, `agri-core`, `agri-db`, `agri-web`, `agri-admin`, …) has its own
> `CONTRIBUTING.md` / `CLAUDE.md` with the concrete toolchain. This file covers
> only the org-wide assets in this repo.

---

## 1. What lives here, and what surfaces it

| Path | Purpose | Where GitHub surfaces it |
| --- | --- | --- |
| `profile/README.md` | Public org landing page: elevator pitch, repo map, 3-tier diagram, release flow | Rendered at **https://github.com/AgriLogy** — the first thing any visitor sees |
| `PULL_REQUEST_TEMPLATE.md` | Default PR body (summary, `Closes #`, checklist) | Pre-filled into **every PR in every org repo** that has no template of its own |
| `.github/ISSUE_TEMPLATE/bug_report.yml` | Bug form (`bug: ` title prefix, auto-assign `mks-zakaria`) | "New issue" chooser in every org repo lacking its own templates |
| `.github/ISSUE_TEMPLATE/feature_request.yml` | Feature form (`feat: ` title prefix, auto-assign) | Same chooser |
| `ARCHITECTURE.md` | Canonical platform map: topology, repo map, Django→FastAPI strangler ledger, data-flow walkthroughs, backlog | Linked from `profile/README.md`; the onboarding/handoff document for the whole platform |
| `CONTRIBUTING.md` (this file) | Rules for editing the above | Linked by GitHub on issue/PR creation in repos without their own |

**Inheritance rule:** a defaults file here applies to an org repo **only if that
repo has no file of the same kind.** A repo-local `PULL_REQUEST_TEMPLATE.md` or
`.github/ISSUE_TEMPLATE/` always wins. `profile/` and `ARCHITECTURE.md` are
special to this repo and are not inherited.

---

## 2. Blast radius

Edits here are **org-wide and immediate** — there is no staging, no version pin,
no rollout. Concretely:

- A typo in `PULL_REQUEST_TEMPLATE.md` shows up in the **next PR opened in any
  repo**, by anyone, with no deploy step.
- A broken Mermaid block in `profile/README.md` renders as a raw error box on the
  **public org page**.
- Stale `ARCHITECTURE.md` content silently misleads onboarding and handoffs — the
  costliest failure mode here, because it looks authoritative.

So: keep changes small and self-contained, one concern per PR, and **preview the
rendered output** (GitHub's PR "Rich diff" tab, or the file view on your branch)
before merging. Never bundle an `ARCHITECTURE.md` refresh with a PR-template
change — different blast radii, different review needs.

---

## 3. Updating `ARCHITECTURE.md`

It is the **map, not the territory.** It must describe what is *actually live*,
never what is planned-but-unmerged.

### When to update

| Trigger | Sections that must be refreshed |
| --- | --- |
| A migration phase lands (F11, F12, …) | §1 ecosystem diagram, §3 strangler — routing diagram, **phase ledger**, "What is LEFT in Django" table, "Next two steps", §7 backlog |
| A repo is added or retired | §2 repository map **and** the repo table in `profile/README.md` (they must agree) |
| A prod topology change (port, container, nginx route, DB) | §1, §3 routing diagram, §5 Celery if jobs move |
| A feature stops being mocked / a provider is wired | §4 worked example, §7 backlog |
| Release or deploy rules change | §6 **and** the "Release flow" section of `profile/README.md` |

Rules:

1. **Diagrams stay Mermaid.** No images, no ASCII art, no external diagram
   services — GitHub renders Mermaid natively and it must stay diffable.
2. **Verify every diagram renders before merging.** Push the branch and open the
   file on GitHub (or the PR's *Rich diff*): a good block renders as a diagram, a
   broken one renders as an error panel or as raw code. Common breakers: an
   unescaped `(`/`)` or `#` inside a node label (wrap labels in `"…"`), a `<br/>`
   outside quotes, or a stray blank line inside the fence.
3. **Bump the `_Last updated: YYYY-MM-DD._` line** near the top on *every*
   substantive edit. A doc whose date lies is worse than no doc.
4. Keep §2/§7 tables in the same shape (don't restructure to make one row fit).
5. Cross-check `profile/README.md`: repo map, migration status and release flow
   are duplicated there and must not drift.

---

## 4. Updating `PULL_REQUEST_TEMPLATE.md`

The template is the checklist every contributor sees, so it must stay short and
every line must be actionable. Keep the `Closes #` line and the Conventional-
Commit-title item — both are load-bearing for the release flow.

Verification, required before merge:

1. Merge (or push) the change, then open a **scratch PR in a consuming repo** —
   any org repo without its own template. Branch, make a trivial change, open the
   PR as a **draft**.
2. Confirm the body pre-fills with the new template, the checkboxes render as
   checkboxes, and no HTML comment leaks into the visible body.
3. Close the scratch PR and delete the branch.

Deleting or renaming this file silently removes the default from every repo —
do it deliberately, never as drive-by cleanup.

---

## 5. Branch and PR rules

Same rules as every AgriLogy repo:

- **Branch off `main`.** No direct commits to `main`.
- **One dedicated, scope-matched issue per PR.** File the issue first; the PR
  body carries `Closes #N`. Don't reuse an umbrella issue across PRs.
- **PR title is a Conventional Commit** — `docs: …` for `ARCHITECTURE.md` /
  `profile/README.md`, `chore: …` for template plumbing. Squash-merge uses it.
- **Assign both the issue and the PR to the author** (`mks-zakaria`).
- **Zero AI/assistant attribution** anywhere — commit messages, PR bodies, issue
  text, branch names, footers, co-author trailers. The author is the sole voice.
- Prefer several small PRs over one sweep touching profile + template + architecture.

---

## 6. Quick checklist before you open the PR

- [ ] Only org-wide assets touched — no product code, no unrelated files
- [ ] Mermaid diagrams verified rendering on GitHub (not just locally)
- [ ] `ARCHITECTURE.md` "Last updated" bumped, if it was edited
- [ ] `profile/README.md` and `ARCHITECTURE.md` still agree (repo map, migration status, release flow)
- [ ] PR-template change validated via a scratch draft PR in a consuming repo
- [ ] Conventional-Commit title, `Closes #N`, issue + PR assigned, no AI attribution
