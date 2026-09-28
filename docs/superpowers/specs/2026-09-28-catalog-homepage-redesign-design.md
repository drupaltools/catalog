# drupaltools/catalog - homepage redesign

Date: 2026-09-28

## Problem

`drupaltools.github.io` presents a homepage of 187 third-party Drupal tools filtered to
`recommended: true`. The tools the drupaltools organisation actually builds
(`drupaltools/skills`, `drupaltools/tools`, `@drupaltools/mcp`) are invisible on it.

We want a second site, `drupaltools/catalog`, whose homepage is a catalog of the
organisation's own work, with the community list kept as a secondary page.

## Decisions already made

| Question | Decision |
|---|---|
| Fate of the 187 curated tools | Kept, reachable as a secondary page |
| Data source for org tools | Generated file, re-synced by a scheduled GitHub Action |
| Publishing | Public repo, GitHub Pages project URL `https://drupaltools.github.io/catalog/` |
| Tool granularity | One card per real package in `drupaltools/tools` (6 tools) |
| Additional sources | `@drupaltools/mcp` gets a card |
| Skills card destination | Card links to a local `/skills/` page |
| Add-a-tool mechanism | GitHub Issue Form, parsed by an Action that commits to `manual.yml` |
| Homepage layout | Flat filterable grid with type and language chips |
| Visual direction | New skin applied site-wide, not just the homepage |
| Card density | Name, type/language badge, one-line description, install command, source and registry links |
| Dark variant | In scope, via `prefers-color-scheme` and CSS variables |

## Non-goals

- Changing `drupaltools.github.io`. That repo keeps its content, its CNAME and its Pages setup.
- Editing the source repos (`skills`, `tools`). The catalog only reads from them.
- Replacing the community list with a new data format. The 187 YAML files stay as they are.

## Architecture

```
drupaltools/skills  ----\
                          >-- GitHub API -- scripts/sync-catalog.mjs -- _data/catalog/generated.yml
drupaltools/tools   ----/                                          \-- _data/catalog/skills.yml
local mcp-package/package.json ------/

GitHub Issue Form -- add-tool-from-issue.yml -- _data/catalog/manual.yml

generated.yml + skills.yml + manual.yml -- Jekyll -- index.html, /skills/, /all/
```

Three data files, one merge point. The merge is what keeps automated sync and
human additions from fighting each other: the sync script owns `generated.yml` and
`skills.yml` and rewrites them wholesale; `manual.yml` is only ever appended to.

## Repository changes

The repo was created as a clone of `drupaltools.github.io` at `d568c2e`, pushed to
`drupaltools/catalog`. Divergence from the source repo:

1. Delete `CNAME`. It contains `www.drupaltools.com`, which must stay with the
   original site. Two repos claiming the same custom domain breaks one of them.
2. `_config.yml`: set `baseurl: "/catalog"` and `url: "https://drupaltools.github.io"`.
3. Fix absolute paths that break under the `/catalog` subpath. The complete list,
   established by scanning every template for `href="/` and `src="/`:
   - `_includes/head.html` lines 28 to 32: favicon and manifest `href`s (`/icons/...`).
   - `index.html` line 23, `all/index.html` line 20, `deprecated/index.html` line 22,
     `_layouts/project.html` line 11: `<img src="/img/no-image.png" />`.
   - `_includes/header.html` and `_includes/footer.html`: outbound badge and issue
     links that reference `drupaltools.github.io` and must be repointed at
     `drupaltools/catalog`.

   `robots.txt` needs no change: it has no sitemap host. The ShareThis script in
   `_includes/footer.html` uses a protocol-relative URL, which is unaffected.
4. `mcp-package/package.json` and `.github/workflows/publish-npm.yml`: repoint
   repository URLs from `drupaltools.github.io` to `drupaltools/catalog`. The clone
   carries the MCP server, so its publish workflow must target the new home.
5. Do not copy `.superpowers/` from the source working copy. It is brainstorming
   tooling output, not project content.

Pages is enabled with `gh api -X POST repos/drupaltools/catalog/pages` using
`source[branch]=master` and `source[path]=/`. This is the same legacy Jekyll build
the original site uses. No Actions-based deploy is needed, because the sync Action
commits data and Pages rebuilds on push.

## Data model

### `_data/catalog/generated.yml`

Written wholesale by the sync script.

```yaml
- id: drupal-news
  name: drupal-news
  type: tool            # tool | skills | mcp
  language: python      # filter dimension
  badge: python         # optional display text, defaults to language
  description: "Automated Drupal news aggregation with AI summarization."
  source: https://github.com/drupaltools/tools
  source_path: packages/python/drupal-news
  install: "pip install drupal-news"
  registry: pypi        # npm | pypi | packagist | none
  docs: ""              # optional
  homepage: ""          # optional
  status: active        # active | experimental | archived, stored but not rendered
  skills_count: 0       # skills entries only
  agents_count: 0       # skills entries only
  page: ""              # skills entries only, e.g. /skills/
```

The `status` field is stored but not rendered on the card, so it can be surfaced
later without another sync.

### `_data/catalog/skills.yml`

Written wholesale by the sync script. Feeds `/skills/`.

```yaml
- name: drupaltools-code-review
  kind: skill          # skill | agent
  description: "Review and score Drupal code across security, correctness, ..."
  path: skills/drupaltools-code-review/SKILL.md
  url: https://github.com/drupaltools/skills/blob/master/skills/drupaltools-code-review/SKILL.md
```

`description` is taken verbatim from the frontmatter. It is truncated to 240
characters for display only; the stored value is complete.

### `_data/catalog/manual.yml`

Appended to by the issue-form Action or by hand. Same schema as `generated.yml`,
except `type` also accepts `other` and `status` defaults to `experimental`.

### `_data/catalog/overrides.yml`

Hand-written, checked in, never machine-generated. Supplies `description`,
`install` and `registry` for packages whose manifest cannot provide them. Merged
over generated entries by `id`.

## Sync script

`scripts/sync-catalog.mjs`, Node ESM. `js-yaml` is already a dependency of the repo,
so no new packages.

Sources and what is read from each:

| Source | Call | Extracted |
|---|---|---|
| `drupaltools/skills` | `GET /repos/drupaltools/skills/contents/skills` | one directory per skill |
| `drupaltools/skills` | `GET /repos/drupaltools/skills/contents/agents` | one file per agent |
| `drupaltools/skills` | raw `SKILL.md` and agent `.md` frontmatter | `name`, `description` |
| `drupaltools/tools` | `GET /repos/drupaltools/tools/contents/packages` | language directories |
| `drupaltools/tools` | `GET .../contents/packages/<lang>` | package directories |
| `drupaltools/tools` | package manifest via raw URL | `name`, `description` |
| local | `mcp-package/package.json` | `name`, `version`, `description` |

Manifest priority per language: `package.json` for node, `pyproject.toml` for
python, `composer.json` for php.

Skipped: directories named `example-package` in any language. They are monorepo
scaffolding, not tools. Skips are logged.

**Description resolution, and the rule that matters:**

1. Manifest description field.
2. `overrides.yml`.
3. Neither available: the script logs the offending `id` to stderr and exits `1`.

A missing description is a hard failure, never an empty or invented string. Three
packages currently trip this: `drupal-code-search`, `drupal-versions` and
`module-info` are single scripts with no manifest. Their overrides are written by
hand from their actual READMEs.

**Install command derivation**, and the same three-strikes rule: from the manifest
name plus registry (`pip install <name>`, `npm install <name>`, `composer require
<name>`), else from `overrides.yml`, else hard failure. `drupal-web-search` and
`drupal-versions` install from a checkout, not a registry, so their override text
says so rather than inventing a published package name.

**Offline mode:** `--offline` reads `scripts/__fixtures__/*.json` instead of calling
GitHub. Used by tests and by the PR check, so the script is testable without network
and without burning API quota.

**Failure behaviour:** the script writes to temporary files and moves them into
place only after every entry resolves. A failed run leaves the previous data intact
and exits non-zero, so the Action reports red instead of committing half a catalog.

## GitHub Action: sync

`.github/workflows/sync-catalog.yml`

- Triggers: `schedule` daily at an off-minute cron, `workflow_dispatch`, and
  `repository_dispatch` with type `catalog-sync` so the skills repo can trigger it.
- Permissions: `contents: write`.
- Steps: checkout, setup-node 20, run `node scripts/sync-catalog.mjs`, then
  `git diff --quiet` guard, then commit and push if changed.
- Commit message: `chore(catalog): sync generated data`.

Uncached GitHub API calls are made with the workflow's `GITHUB_TOKEN` to avoid the
60 requests per hour unauthenticated limit.

## Add-a-tool flow

`.github/ISSUE_TEMPLATE/add-tool.yml`, a GitHub Issue Form with labels applied
automatically. Fields, all required except the last two:

| Field | Input | Validation |
|---|---|---|
| Tool name | text | required |
| Type | dropdown | tool, skills, mcp, other |
| Language | dropdown | python, node, php, rust, go, ai, other |
| Description | textarea | required, 160 character cap |
| Source URL | input | required |
| Install command | input | optional |
| Docs URL | input | optional |

`.github/workflows/add-tool-from-issue.yml`

- Triggers: `issues` with types `opened`, `labeled`. Runs only when the `add-tool`
  label is present.
- **Authorization guard:** proceeds only when `github.event.issue.author_association`
  is `OWNER`, `MEMBER` or `COLLABORATOR`. Otherwise it comments that a maintainer
  must apply the label, and stops. Without this guard any GitHub account could commit
  arbitrary content to the repo through the form.
- Parses the rendered form body (`### Field` headings followed by values), appends one
  entry to `manual.yml`, commits as `chore(catalog): add <name> from issue #<n>`,
  comments with the commit link, closes the issue.
- A malformed or missing required field causes a comment naming the field and leaves
  the issue open. No partial write.

The homepage carries an "Add a tool" card linking to
`https://github.com/drupaltools/catalog/issues/new?template=add-tool.yml`. This is the
entry point the "add new tools from the homepage UI" requirement is satisfied by: the
form is a GitHub-hosted page the card opens, and a bot commits the result. There is no
server-side write path, because the site is static.

## Pages

### `/` (homepage, layout `front`)

1. Header: site title, nav (Catalog, Skills, Community, GitHub).
2. Hero: one-line statement of what the org builds, and the count.
3. Filter bar: type chips (All, Tools, Skills & Agents, MCP) and language chips
   (only the languages actually present in the data).

   `type: other` entries added through the issue form get no dedicated type chip.
   They appear under All and under their language chip. This is deliberate: `other`
   means "not yet classified", and promoting it to a chip would reward leaving it
   unset.
4. Card grid. Card contents, matching the approved "Standard" density:
   - name, monospace
   - badge (language, or `badge` override)
   - one-line description
   - install command in a monospace strip
   - links: source, registry when present, docs when present
5. "Add a tool" card at the end of the grid, linking to the issue form.
6. A line linking to the community list.

The grid renders from the three merged data files at build time. Filtering is client
side: each card carries `data-type` and `data-language`, and a small vanilla script
toggles a hidden class. This follows the `data-*` + hide-class pattern already used by
`js/scripts.js`. Chips are `<button>` elements with `aria-pressed`, and the grid
carries `aria-live="polite"` so the count is announced.

### `/skills/`

Lists all 32 skills and 11 agents from `skills.yml`, grouped into two sections. Each
row shows the name, its description, and a link to the file on GitHub. Counts in the
section headings come from the data, not from hardcoded text.

### `/all/`

Kept, restyled. The community list of 187 tools. Its per-card "more details" link is
removed: it points at `/projects/<name>/`, and those pages 404 on the live site today
because GitHub Pages ignores `_plugins/data_page_generator.rb`, which never runs in the
legacy build. Cards keep their source and docs links, which work.

`_plugins/data_page_generator.rb` is left in place but is inert. Removing it is a
separate cleanup, not part of this work.

### `/deprecated/` and project pages

Carried over and restyled only as far as the shared CSS reaches them.

## Styling

`_sass/_base.scss` and `_sass/_layout.scss` are replaced by a new set of partials
imported from `css/main.scss`. The new system is defined with CSS custom properties so
the dark variant is one variable block behind `@media (prefers-color-scheme: dark)`.

Direction, matching the approved mockup: neutral surfaces, hairline borders rather than
drop shadows, monospace for package and skill names, a system sans stack for prose, and
one accent colour reserved for links and the active filter chip. The existing blue
`#0678be` header band, the grey `#ddd` intro strip and the `box-shadow: 0 0 6px #ddd`
tiles are removed.

Templates referencing the old classes (`_layouts/default.html`, `_layouts/front.html`,
`_layouts/page.html`, `_layouts/project.html`, `_includes/header.html`,
`_includes/footer.html`, `_includes/filters.html`, `_includes/buttons.html`,
`_includes/buttons-front.html`, `index.html`, `all/index.html`, `deprecated/index.html`,
`about.html`) are updated to the new class names in the same change, so no template is
left pointing at CSS that no longer exists.

The `frontend-design` skill is applied when writing the CSS.

## Error handling

| Failure | Behaviour |
|---|---|
| GitHub API unreachable during sync | Script exits non-zero, data files untouched, Action fails visibly |
| Description or install not derivable | Script exits non-zero naming the `id`; author adds an override |
| Skills or agents repo restructured | Script fails on the missing path rather than emitting zero entries |
| Issue form body malformed | Action comments naming the missing field, issue stays open, no write |
| Issue opened by a non-collaborator | Action comments, does not commit, issue stays open |
| Pages build failure | Visible in the repo's Actions and Pages tab; no partial deploy |

## Testing

**Sync script.** `node scripts/sync-catalog.mjs --offline` run against fixtures, then
assert the emitted `generated.yml` contains exactly the expected `id` set. A second
case feeds a fixture whose package has no manifest and asserts exit code 1 and that
nothing was written. Runs in CI on pull requests that touch the script.

**Jekyll build.** `bundle exec jekyll build`, then assert against `_site`:

- `_site/index.html` contains a card for each expected `id`, plus the add-tool card.
- `_site/skills/index.html` contains 43 entries.
- No `href="/` or `src="/` in any built file that does not resolve under `/catalog/`.
  This is the check that catches subpath breakage, which is the most likely way this
  change silently ships broken.

**Manual pass.** `bundle exec jekyll serve`, then click through homepage filters, the
skills page, the community list, and the add-tool card link. Verify the dark variant by
toggling the OS preference.

**Action dry runs.** `workflow_dispatch` on a branch for the sync workflow. A test
issue opened by a collaborator for the add-tool workflow, on a scratch label, to
confirm the comment-write-close path before it is relied on.

## Rollout

1. Repo clone and divergence (CNAME, baseurl, path fixes) in one commit.
2. Data model, sync script, fixtures and overrides.
3. Sync Action.
4. Homepage, skills page, restyle.
5. Issue form and add-tool Action.
6. Enable Pages, verify the deployed URL end to end.

Each step is independently reviewable. Steps 1 and 2 leave the site fully functional,
so a partial landing is not a broken site.

## Open risks

- `stand-with-cyprus` is a political message package, not a Drupal development tool. It
  will appear on the homepage because it is a package in `drupaltools/tools`. Flagging
  so the decision to include it is deliberate.
- The clone carries `mcp-server/`, whose content describes the 186-tool community list.
  Its copy will read oddly once the catalog homepage is about org tools.
