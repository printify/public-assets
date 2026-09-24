---
name: review-pr-additions
description: Repo-specific additions to the code-reviewer plugin's universal review-pr skill for public-assets. Data file consumed by the plugin's review-pr skill at runtime; not auto-invoked by the model.
disable-model-invocation: true
---

# review-pr additions — public-assets

_Last verified: 2026-09-23. Refresh when this repo's architecture, domain boundaries, or test/process conventions change — re-run `/generate-review-rubric` or edit this file._

These instructions LAYER OVER the universal contract that the `code-reviewer` plugin's `review-pr` skill enforces. They do NOT replace it; the universal contract still applies unchanged. If anything below appears to conflict with it, the baseline wins — report the conflict as a finding rather than silently overriding.

## Repo context

- Blast radius: public, world-readable static images. The repo is public. There is no build, no deploy step and no CDN: consumers load files straight from `raw.githubusercontent.com/printify/public-assets/main/...` or `.../refs/heads/main/...`, so a merge to `main` publishes at once, and a file once merged stays readable in git history after deletion.
- Tree: `README.md`, `.github/CODEOWNERS`, and images under `images/` (PNG and SVG). No source code, dependency manifest, tests or workflows other than the `ai-review.yml` caller this rubric ships with.
- Known consumers hardcode the path and the `main` branch: `printify/qa` (`ci-interactive-inputs/src/slack/slack.ts` uses `images/ci-cd.png` as a Slack bot icon; `.github/workflows/report-test-results.yml` uses `images/allure-logo-horizontal.png` in PR comments), `printify/vr` (`.github/actions/visual-review/action.yml` uses `images/vr-magnifying-glass.png`), and `printify/admin_new` storybook data (`src/stories/meta-data/providers.ts` sets this repo as the base URL; `src/stories/blueprints/general/data/caresets.data.ts` loads `images/machine-wash.svg` and `images/do-not-wash.svg`). Other consumers (the README names Jenkins and custom apps) may exist and are not searchable.
- `review-pr` cannot see the content of a binary image, nor its metadata (EXIF, XMP, `tEXt`/`iTXt`). A diff to a PNG shows only that bytes changed.

## Risk tiers
Classify each PR and populate the `tier` field in the verdict block. Tier numbers are integers 0–3 only; `null` means "could not classify," never "low risk." Classification rules (apply in order; when in doubt prefer the higher tier or `null` — never guess low):
- **Behavior-neutral carve-out (hybrid) — a subtraction, never a grant:** an inert hunk contributes no tier of its own. Set it aside, tier the remainder of the diff on its own merits, and let the rules below decide; the carve-out only ever removes, and never reaches `tier: 0` by itself. When every hunk is inert there is no remainder, and the diff keeps the tier of the surfaces it touches — tier 0 only where those surfaces are already tier 0. Inert means a comment, prose, or whitespace edit with no effect on how the code compiles, type-checks, builds, runs, or renders, or on how an agent or tool behaves. Never inert, whatever it looks like: **a live line becoming a comment, or a comment becoming a live line** — commenting out a setting is not editing a comment, it is deleting a setting, and in a file that overrides defaults deleting a setting restores whatever the default was; **a comment a tool reads** — a suppression, pragma, directive, type annotation or coverage marker; **whitespace where whitespace is syntax** — indentation, line continuation or chomping that changes what is produced rather than how it reads; **prose in instruction and config files** (`CLAUDE.md`, agent rules, skills, Dockerfiles, CI and tooling config), which is behavior. Not certain a hunk is inert, for any reason: set `tier` to `null`, never 0. Never applies to `.github/` or this rubric file.
  - In this repo the carve-out also never applies to: any file under `images/**` (an SVG comment, attribute or whitespace is served to the public byte for byte), `CLAUDE.md`, `AGENTS.md`, `.claude/**`, and any workflow, manifest, build, container or deploy file a PR adds. Tier those by their row below; not certain which row, set `tier` to `null`.
- **Test-only changes:** a change confined to tests takes tier 0 when it only adds coverage. Adding a case while removing assertions is not added coverage. It is not tier 0 when you cannot tell which surface the tests guard, or cannot tell whether coverage grew or shrank. Tests guarding a tier-3 surface take that surface's tier whatever the change. Test infrastructure — runner config, bootstrap, base test classes, fixtures and fakes — is outside this rule; tier it by what it can affect. A test change that alters expected behavior, or weakens or removes existing coverage, takes tier 1, or the guarded surface's tier where it guards one.
  - In this repo no tests exist, so a PR adding tests is unmapped: set `tier` to `null`.
- Start from the HIGHEST tier among all surfaces the diff touches — however small that portion of the diff (a one-line **functional** `src/auth/` change in an otherwise docs-only PR is still the auth tier).
- If the diff touches any file you cannot confidently map to a row below, do NOT settle on a low/mid tier on the strength of the mapped files — treat the unmapped portion as unknown risk and set `tier` to `null` (unless a rule below already assigns the top tier).
- If the diff introduces a sensitive surface kind (auth, money, PII, migrations, prod infra) not yet represented below, or modifies this rubric file (`.claude/skills/review-pr-additions/SKILL.md`) itself, classify at the top tier and note the rubric may be stale.
- If two tiers are plausible, assign the higher.

| Tier | What it looks like (paths/surfaces) | Reviewer attention |
|---|---|---|
| 0 | Edits to the existing root `README.md` only. A new prose file, in any directory, is unmapped: `null`. **Behaviorally:** human-facing prose no tool or agent reads. Never an image, anything under `images/`, `.github/` or `.claude/`, nor `CLAUDE.md` or `AGENTS.md`. A hunk that adds a credential, token, internal hostname or URL, email address, personal name, or a command to run is not prose; nor is a hunk that adds a link, an image embed or raw HTML: set `tier` to `null`. | Confirm the diff is prose only. |
| 1 | A new image file (PNG, JPEG, GIF, WebP or SVG) under `images/` at a path not present at the base ref. Any other new file under `images/` is unmapped: `null`. **Behaviorally:** a new public asset that no known consumer loads yet. | A human must look at the image itself: it becomes public on merge and cannot be unpublished. |
| 2 | Any modification, rename, move or deletion of an existing file under `images/`, and any new top-level directory holding assets. **Behaviorally:** anything that changes what an existing public URL serves, or makes it 404. | Consumers in other repos hardcode each path on `main`; a rename or delete breaks them with no error here. |
| 3 | `.github/**` (including `CODEOWNERS` and `workflows/ai-review.yml`), `.claude/**`, `CLAUDE.md`, `AGENTS.md`, this rubric file. **Behaviorally:** any change to who must review; any new workflow; any credential or secret reference; any build, deploy or publish config (Pages, CDN, bucket sync) wherever it lands. | Always a human. |

**Bot dependency bumps — rollout-wide policy.** A diff that changes *only* the ref of an
already-pinned `uses:` for a **GitHub Actions reusable workflow or action** — the
`owner/repo` half byte-identical, only the part after `@` moving — or only a
package-manager dependency version, and nothing else, is **capped at tier 2** even when it
sits under a tier-3 path. A cap, never a promotion: where a row below already places the
bumped surface lower, it keeps that lower tier. A tier-2 read is still a real read of the
upstream release notes, not a rubber stamp — the reusable workflow is where the test gate
lives. A changed `owner/repo` is a different supplier rather than a bump, and never
qualifies however familiar the new owner looks.

Stays **tier 3**, no exceptions: any change to `on:`, `permissions:` or `secrets:`; any
change inside `jobs:` beyond the pinned ref itself; a new workflow file; and **any ref,
version or constraint that is applied rather than merged**. Under `terraform/**` that means
all of: a module source or ref, a **provider pin**, and `required_version` — not just the
module ref. Also a helm chart version or `Chart.lock` digest, and a container base image
**that is deployed**. A git revert does not undo an apply, so those never qualify as a
low-risk bump however small the diff looks.

## Additional checks for this repo

For `review-pr`'s severity ranking and re-run de-dupe, treat findings from the sections below as **PR hygiene** unless a finding is clearly security or correctness (so they slot into the universal category ordering and don't re-post on re-runs).

### Domain boundaries
- Flag a file added outside `images/` that is not repo governance (`README.md`, `.github/**`, `.claude/**`, `CLAUDE.md`, `AGENTS.md`) — non-blocking.
- Flag any file that is not a static asset: script, HTML, executable, archive, office document or data export — blocking. This repo publishes to the world with no review gate after merge.

### Pattern consistency
- Flag a new asset without a lowercase kebab-case name and a file extension (siblings: `images/do-not-wash.svg`, `images/allure-logo-horizontal.png`) — non-blocking. Check that the extension matches the content for SVG and text files only; for a binary, ask the author to confirm the format.
- Flag a new asset that duplicates an existing one under a second path — non-blocking. Compare blob `sha` values from a `get_file_contents` listing of `images/` at the head ref; if that is not possible, say so rather than claim no duplicate.

### Test conventions
No repo-specific rules beyond the baseline.

### Process integrations
- This repo uses no ticket prefix: `git log` shows none. Do not flag a missing ticket reference.
- Flag a rename, move or deletion of an existing file under `images/` when the PR does not name the consumers it checked — blocking. Name the known consumers from `## Repo context` that load that path.
- Flag an SVG that contains `<script>`, an event-handler attribute (`on*=`), `<foreignObject>`, `<!DOCTYPE` or `<!ENTITY`, `<?xml-stylesheet`, a `<style>` with `@import`, a `<set>` or `<animate>` that targets `href`, or an `href`/`xlink:href` or CSS `url()` to anything other than a fragment in the same file — blocking, security.
- Flag any added text content (SVG body, README) carrying a credential, token, internal hostname, internal URL, email address or personal name — blocking, security.
- Flag a new or changed binary image whose PR does not say what it shows — non-blocking. The reviewer cannot see it or its metadata; ask the author to confirm metadata was stripped, say so in the summary, and never report a binary image as checked.
- Flag a change to `.github/CODEOWNERS` that leaves any path without an owner, or changes the owning team without the PR saying why — blocking, security.
- Flag a PR that adds a build, deploy or publish mechanism (GitHub Pages, a CDN or bucket sync, a workflow) without updating this rubric's `## Risk tiers` table — non-blocking.

## Known-acceptable in this repo

The baseline may flag the following; in this repo they are intentional and should be named on the review summary's Known-acceptable line, not raised as actionable findings.

- `images/ci-cd` (no extension) and `images/ci-cd.png` are byte-identical while unchanged. Unknown consumers may load either path, so neither is removed without the consumer check above.
- `images/machine-wash.svg` carries a third-party generator comment (`SVG Repo`) while unchanged.
