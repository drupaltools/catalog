# drupaltools/catalog Homepage Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn `drupaltools/catalog` into a site whose homepage lists the tools, skills and agents the drupaltools organisation builds, with the 187-tool community list kept as a secondary page.

**Architecture:** Jekyll static site built by GitHub Pages from `master`. A Node sync script reads the GitHub API for `drupaltools/skills` and `drupaltools/tools` and writes `_data/catalog/generated.yml` plus `_data/catalog/skills.yml`. Hand-maintained `overrides.yml` fills gaps the manifests cannot supply, and `manual.yml` holds entries added through a GitHub Issue Form that an Action commits. The homepage merges all three at build time.

**Tech Stack:** Jekyll 4.4, Liquid, SCSS, vanilla ES modules (no framework), Node 20+ with `node:test`, `js-yaml`, `smol-toml`, GitHub Actions, GitHub Pages legacy build.

## Global Constraints

- `baseurl: "/catalog"`, `url: "https://drupaltools.github.io"` in `_config.yml`. Every internal link and asset path must work under `/catalog`.
- The source repo `drupaltools/drupaltools.github.io` is never modified by this plan.
- No description or install command is ever invented. Unresolvable values fail the build loudly. An override may set `install: ""` to mean "deliberately none"; an *omitted* install is a failure. The two are different.
- Node 20 or newer. GitHub Actions runner uses `node-version: '20'`.
- No new client-side frameworks. No jQuery for new code; the existing `js/scripts.js` jQuery is left as is for `/all/`.
- Card copy is one line, maximum 160 characters in `manual.yml` entries.
- No emoji, no em dashes, no smart quotes in any committed file or rendered string.
- Commit messages follow Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `test:`).

## Verified external facts

These were confirmed against the live services on 2026-09-28. Do not re-derive them from memory.

| Package | Manifest | `registry` | `install` | Description source |
|---|---|---|---|---|
| `drupal-news` | `packages/python/drupal-news/pyproject.toml` | `pypi` | `pip install drupal-news` | manifest: "Automated Drupal news aggregator with AI-powered summaries" |
| `drupal-web-search` | `packages/python/drupal-web-search/pyproject.toml` | `none` | `pip install -e .` | manifest: "Generic web search CLI with DDGS, Exa, Tavily, Perplexity, engine fallback, and site preferences" |
| `drupal-code-search` | none (directory of 3 scripts) | `none` | `python3 drupal_code_search_tresbien.py "query"` | override |
| `drupal-versions` | none (single script) | `none` | `python3 drupal_versions.py` | override |
| `module-info` | none (single script) | `none` | `python3 module-info.py <path>` | override |
| `stand-with-cyprus` | `packages/php/stand-with-cyprus/composer.json` | `none` | `""` | manifest: "Stand with Cyprus" |

- PyPI has `drupal-news` (HTTP 200) but not `drupal-web-search` (404).
- Packagist has no `tplcom/stand-with-cyprus` (404).
- npm has `@drupaltools/mcp` at version `1.0.8`.

---

### Task 1: Divergence for the `/catalog` subpath

**Files:**
- Create: `scripts/check-baseurl.mjs`
- Modify: `_config.yml:7-8`, `_includes/head.html:28-32`, `_includes/header.html:6-8`, `_includes/footer.html:22-24`, `index.html:23`, `all/index.html:20`, `deprecated/index.html:22`, `_layouts/project.html:11`, `robots.txt` (no change, listed for completeness)
- Delete: `CNAME`

**Interfaces:**
- Consumes: nothing
- Produces: `node scripts/check-baseurl.mjs` exits `0` when `_site` is subpath-clean and `1` otherwise. Tasks 4, 5, 8 and CI call it.

- [ ] **Step 1: Write the failing check script**

Create `scripts/check-baseurl.mjs`:

```js
#!/usr/bin/env node
// Fails if a built page references an absolute path outside /catalog.
// GitHub Pages serves project sites under a subpath, so a bare href="/css/x.css"
// resolves to the org root and 404s.
import { readdirSync, readFileSync, statSync } from 'node:fs';
import { join, relative } from 'node:path';

const ROOT = '_site';
const BASEURL = '/catalog';
const TEXT_EXTENSIONS = ['.html', '.css', '.xml'];
const PATTERN = /\b(?:href|src|action)="(\/[^/][^"]*)"/g;

function walk(dir) {
  const out = [];
  for (const entry of readdirSync(dir)) {
    const full = join(dir, entry);
    if (statSync(full).isDirectory()) out.push(...walk(full));
    else if (TEXT_EXTENSIONS.some((ext) => full.endsWith(ext))) out.push(full);
  }
  return out;
}

const offenders = [];
for (const file of walk(ROOT)) {
  const text = readFileSync(file, 'utf8');
  for (const match of text.matchAll(PATTERN)) {
    const path = match[1];
    if (path.startsWith(`${BASEURL}/`) || path === BASEURL) continue;
    // Protocol-relative and external URLs never start with a single slash,
    // so anything reaching here is a real subpath bug.
    offenders.push(`${relative(ROOT, file)}: ${path}`);
  }
}

if (offenders.length > 0) {
  console.error(`Found ${offenders.length} absolute path(s) outside ${BASEURL}:`);
  for (const line of offenders) console.error(`  ${line}`);
  process.exit(1);
}
console.log(`OK: no absolute paths outside ${BASEURL}`);
```

- [ ] **Step 2: Build the site and run the check to verify it fails**

```bash
bundle exec jekyll build
node scripts/check-baseurl.mjs
```

Expected: exit 1, listing at least `index.html: /img/no-image.png` and `head.html`-derived
`/icons/favicon-32x32.png` entries under every built page.

- [ ] **Step 3: Remove the CNAME**

```bash
git rm CNAME
```

The original repo keeps `www.drupaltools.com`. Two repos claiming one custom domain
breaks one of them.

- [ ] **Step 4: Set the subpath and extend the exclude list in `_config.yml`**

Replace lines 7 and 8:

```yaml
baseurl: "/catalog" # the subpath of your site, e.g. /blog
url: "https://drupaltools.github.io" # the base hostname & protocol for your site
```

Then extend the `exclude` list so the sync tooling is not published as site assets.
Replace the existing `exclude` block with:

```yaml
exclude:
  - Gemfile
  - Gemfile.lock
  - node_modules
  - vendor
  - .git
  - .github
  - mcp-package
  - api
  - scripts
  - docs
  - package.json
  - package-lock.json
```

`scripts` currently ships to `_site` and would publish the whole `__fixtures__` tree.
Adding `docs` keeps the spec and this plan off the public site.

- [ ] **Step 5: Fix the favicon paths in `_includes/head.html`**

Replace lines 28 to 32 so every path is prefixed with `{{ site.baseurl }}`:

```html
  <link rel="apple-touch-icon" sizes="114x114" href="{{ site.baseurl }}/icons/apple-touch-icon.png">
  <link rel="icon" type="image/png" sizes="32x32" href="{{ site.baseurl }}/icons/favicon-32x32.png">
  <link rel="icon" type="image/png" sizes="16x16" href="{{ site.baseurl }}/icons/favicon-16x16.png">
  <link rel="manifest" href="{{ site.baseurl }}/icons/manifest.json">
  <link rel="mask-icon" href="{{ site.baseurl }}/icons/safari-pinned-tab.svg" color="#5bbad5">
```

- [ ] **Step 6: Fix the placeholder image paths in four templates**

In `index.html` line 23, `all/index.html` line 20, `deprecated/index.html` line 22 and
`_layouts/project.html` line 11, replace the `src` of the no-image branch:

```html
<img src="{{ site.baseurl }}/img/no-image.png" />
```

Each of those four lines currently reads
`<img src="/img/no-image.png" />` inside a `{% if project.logo == null %}` (or
`{% if page.logo == null %}`) conditional. Change only the `src` value.

- [ ] **Step 7: Repoint the repo links at `drupaltools/catalog`**

In `_includes/header.html` replace lines 6 to 8:

```html
    <a href="https://github.com/drupaltools/catalog"><img alt="Badge" src="https://img.shields.io/badge/GitHub-drupaltools.catalog-blue?colorA=000000" /></a>
    <a href="https://github.com/drupaltools/catalog/network"><img alt="Badge" src="https://img.shields.io/github/forks/drupaltools/catalog.svg?colorA=000000" /></a>
    <a href="https://github.com/drupaltools/catalog/stargazers"><img alt="Badge" src="https://img.shields.io/github/stars/drupaltools/catalog.svg?colorA=000000" /></a>
```

In `_includes/footer.html` replace lines 22 to 24:

```html
        <a href="https://github.com/drupaltools/catalog/issues"><img src="https://img.shields.io/github/issues/drupaltools/catalog.svg" /></a>
        <a href="https://raw.githubusercontent.com/drupaltools/catalog/master/LICENSE.md"><img src="https://img.shields.io/badge/license-MIT-blue.svg" /></a>
```

Leave the third line (the TheodorosPloumis badge) unchanged.

- [ ] **Step 8: Repoint the MCP package at its new home**

In `mcp-package/package.json`, replace the `repository`, `bugs` and `homepage` values:

```json
  "repository": {
    "type": "git",
    "url": "git+https://github.com/drupaltools/catalog.git",
    "directory": "mcp-package"
  },
  "bugs": {
    "url": "https://github.com/drupaltools/catalog/issues"
  },
  "homepage": "https://github.com/drupaltools/catalog#readme",
```

In `.github/workflows/publish-npm.yml` no change is needed: it uses `actions/checkout`
and relative `cd mcp-package` paths. Confirm this by reading the file, and only edit if
a hardcoded slug is present.

- [ ] **Step 9: Rebuild and verify the check passes**

```bash
bundle exec jekyll build
node scripts/check-baseurl.mjs
```

Expected: `OK: no absolute paths outside /catalog`, exit 0.

- [ ] **Step 10: Verify the site still renders**

```bash
bundle exec jekyll serve --port 4000
```

Open `http://localhost:4000/catalog/`. Expected: homepage tiles render, favicon loads
with no 404 in the console, and the header badges point at `drupaltools/catalog`.

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat: serve catalog under the /catalog subpath"
```

---

### Task 2: Sync script, overrides and fixtures

**Files:**
- Create: `scripts/lib/paths.mjs`, `scripts/lib/derive.mjs`, `scripts/lib/github.mjs`, `scripts/lib/emit.mjs`, `scripts/sync-catalog.mjs`
- Create: `scripts/__tests__/derive.test.mjs`, `scripts/__tests__/sync.test.mjs`
- Create: `scripts/__fixtures__/` (see Step 1)
- Create: `_data/catalog/overrides.yml`, `_data/catalog/manual.yml`
- Modify: `package.json` (add `smol-toml`, add `catalog:sync` and `catalog:test` scripts)

**Interfaces:**
- Consumes: nothing from Task 1
- Produces:
  - `resolveEntry({ id, name, language, manifestText, registry, overrides }) -> { id, name, type, language, badge, description, source, source_path, install, registry, docs, homepage, status }`, throws `Error` naming the id when a value cannot be resolved
  - `parseManifest(language, text) -> { name, description }`, both possibly `undefined`
  - `parseFrontmatter(text) -> Record<string, string>`
  - CLI: `node scripts/sync-catalog.mjs [--offline]`, exit 0 on success, 1 on any unresolved value
  - Writes `_data/catalog/generated.yml` and `_data/catalog/skills.yml`

- [ ] **Step 1: Create the fixtures**

Fixtures mirror the layout of the real repositories under
`scripts/__fixtures__/<repo-name>/`. Offline mode reads them with `readdir` and
`readFileSync`, so there is no separate listing format to keep in sync. A directory
that exists is a `dir` entry, a file is a `file` entry, and the same path used for
`listContents` is the path used for `readRaw`.

Create `scripts/__fixtures__/skills/skills/drupaltools-best-practices/SKILL.md`:

```markdown
---
name: drupaltools-best-practices
description: Audit code against Drupal best practices.
---

# Best practices
```

Create `scripts/__fixtures__/skills/skills/drupaltools-code-review/SKILL.md`:

```markdown
---
name: drupaltools-code-review
description: Review and score Drupal code across security, correctness and testing.
---

# Drupal Code Review
```

Create `scripts/__fixtures__/skills/agents/drupal-backend-dev.md`:

```markdown
---
name: drupal-backend-dev
description: >-
  Use this agent for Drupal 10 and 11 backend development work.
color: cyan
---

You are an elite Drupal backend architect.
```

Create `scripts/__fixtures__/skills/agents/drupal-code-reviewer.md`:

```markdown
---
name: drupal-code-reviewer
description: Reviews Drupal code changes before they are committed.
color: red
---

You review Drupal code.
```

Create `scripts/__fixtures__/tools/packages/js/example-package/package.json`. This one
exists specifically to exercise the skip list:

```json
{
  "name": "@monorepo/example-package",
  "version": "0.1.0",
  "description": "Example TypeScript package from polyglot monorepo"
}
```

Create `scripts/__fixtures__/tools/packages/php/stand-with-cyprus/composer.json`:

```json
{
  "name": "tplcom/stand-with-cyprus",
  "description": "Stand with Cyprus",
  "type": "library"
}
```

Create `scripts/__fixtures__/tools/packages/python/drupal-news/pyproject.toml`:

```toml
[project]
name = "drupal-news"
description = "Automated Drupal news aggregator with AI-powered summaries"
requires-python = ">=3.10"
```

Create `scripts/__fixtures__/tools/packages/python/drupal-web-search/pyproject.toml`:

```toml
[project]
name = "drupal-web-search"
description = "Generic web search CLI with DDGS, Exa, Tavily, Perplexity, engine fallback, and site preferences"
requires-python = ">=3.10"
```

Create `scripts/__fixtures__/tools/packages/python/drupal-versions/drupal_versions.py`
containing only its docstring, so the directory exists without a manifest and the
override path is exercised:

```python
#!/usr/bin/env python3
"""Get the latest stable and supported Drupal versions from the CLI."""
```

Create `scripts/__fixtures__/tools/packages/python/drupal-code-search/drupal-code-search-tresbien/drupal_code_search_tresbien.py`
with a single comment line, for the same reason:

```python
# Fixture placeholder. This directory has no manifest by design.
```

Create `scripts/__fixtures__/tools/packages/python/module-info/module-info.py`:

```python
#!/usr/bin/env python3
"""Find the Drupal module or theme owning a given file."""
```

Create `scripts/__fixtures__/mcp/mcp-package/package.json`:

```json
{
  "name": "@drupaltools/mcp",
  "version": "1.0.8",
  "description": "Model Context Protocol (MCP) server for discovering Drupal development tools, utilities, and plugins"
}
```

Do not create `go` or `rust` fixture directories. Leaving them absent keeps the fixture
tree honest about the real monorepo, whose go and rust directories hold nothing but
`example-package`.

- [ ] **Step 2: Add the dependency and npm scripts**

```bash
npm install --save-dev smol-toml@^1.9.0
```

In `package.json`, add to the `scripts` block:

```json
    "catalog:sync": "node scripts/sync-catalog.mjs",
    "catalog:sync:offline": "node scripts/sync-catalog.mjs --offline",
    "catalog:test": "node --test scripts/__tests__/",
    "check:baseurl": "node scripts/check-baseurl.mjs"
```

- [ ] **Step 3: Write the failing derive tests**

Create `scripts/__tests__/derive.test.mjs`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { readFileSync } from 'node:fs';
import { fileURLToPath } from 'node:url';
import { dirname, join } from 'node:path';
import { parseManifest, resolveEntry, parseFrontmatter } from '../lib/derive.mjs';

const FIXTURES = join(dirname(fileURLToPath(import.meta.url)), '..', '__fixtures__');
const read = (name) => readFileSync(join(FIXTURES, name), 'utf8');

test('reads name and description from a pyproject [project] table', () => {
  const result = parseManifest('python', read('tools/packages/python/drupal-news/pyproject.toml'));
  assert.equal(result.name, 'drupal-news');
  assert.equal(result.description, 'Automated Drupal news aggregator with AI-powered summaries');
});

test('reads name and description from a composer.json', () => {
  const result = parseManifest('php', read('tools/packages/php/stand-with-cyprus/composer.json'));
  assert.equal(result.name, 'tplcom/stand-with-cyprus');
  assert.equal(result.description, 'Stand with Cyprus');
});

test('returns undefined fields rather than throwing when the [project] table is absent', () => {
  const result = parseManifest('python', '[build-system]\nrequires = []\n');
  assert.equal(result.name, undefined);
  assert.equal(result.description, undefined);
});

test('derives install from the manifest name when a registry is known', () => {
  const entry = resolveEntry({
    id: 'drupal-news',
    language: 'python',
    manifestText: read('tools/packages/python/drupal-news/pyproject.toml'),
    registry: 'pypi',
    overrides: {},
  });
  assert.equal(entry.install, 'pip install drupal-news');
  assert.equal(entry.registry, 'pypi');
});

test('throws when the description cannot be resolved', () => {
  assert.throws(
    () => resolveEntry({ id: 'mystery', language: 'python', manifestText: '', registry: 'none', overrides: {} }),
    /mystery.*description/i,
  );
});

test('throws when install is omitted for a non-registry entry', () => {
  assert.throws(
    () => resolveEntry({
      id: 'mystery',
      language: 'python',
      manifestText: '',
      registry: 'none',
      overrides: { mystery: { description: 'A thing' } },
    }),
    /mystery.*install/i,
  );
});

test('accepts an explicit empty install string as a deliberate "no install"', () => {
  const entry = resolveEntry({
    id: 'stand-with-cyprus',
    language: 'php',
    manifestText: read('tools/packages/php/stand-with-cyprus/composer.json'),
    registry: 'none',
    overrides: { 'stand-with-cyprus': { install: '' } },
  });
  assert.equal(entry.install, '');
  assert.equal(entry.description, 'Stand with Cyprus');
});

test('override description wins over the manifest description', () => {
  const entry = resolveEntry({
    id: 'stand-with-cyprus',
    language: 'php',
    manifestText: read('tools/packages/php/stand-with-cyprus/composer.json'),
    registry: 'none',
    overrides: { 'stand-with-cyprus': { description: 'A political message for composer', install: '' } },
  });
  assert.equal(entry.description, 'A political message for composer');
});

test('parses SKILL.md frontmatter including simple and folded descriptions', () => {
  const skill = read('skills/skills/drupaltools-code-review/SKILL.md');
  assert.equal(parseFrontmatter(skill).name, 'drupaltools-code-review');

  const agent = read('skills/agents/drupal-backend-dev.md');
  assert.match(parseFrontmatter(agent).description, /Drupal 10 and 11 backend/);
});
```

- [ ] **Step 4: Run the tests to verify they fail**

```bash
node --test scripts/__tests__/derive.test.mjs
```

Expected: FAIL, `Cannot find module '../lib/derive.mjs'`.

- [ ] **Step 5: Implement `scripts/lib/paths.mjs`**

```js
// Central constants. Keeping the repo slugs and the skip list in one place means
// a restructure upstream is a one-line change here.
export const SKILLS_REPO = 'drupaltools/skills';
export const TOOLS_REPO = 'drupaltools/tools';
export const BRANCH = 'master';
export const SKILLS_DIR = 'skills';
export const AGENTS_DIR = 'agents';
export const PACKAGES_DIR = 'packages';
export const MCP_PACKAGE_JSON = 'mcp-package/package.json';

export const SKIP_PACKAGE_NAMES = new Set(['example-package']);

// Monorepo language directory to the ecosystem we report and filter on.
export const LANGUAGE_BY_DIR = {
  js: 'node',
  python: 'python',
  php: 'php',
  go: 'go',
  rust: 'rust',
};

export const MANIFEST_BY_LANGUAGE = {
  node: 'package.json',
  python: 'pyproject.toml',
  php: 'composer.json',
};

export const INSTALL_BY_REGISTRY = {
  npm: (name) => `npm install ${name}`,
  pypi: (name) => `pip install ${name}`,
  packagist: (name) => `composer require ${name}`,
};

export const GENERATED_FILE = '_data/catalog/generated.yml';
export const SKILLS_FILE = '_data/catalog/skills.yml';
export const OVERRIDES_FILE = '_data/catalog/overrides.yml';
```

- [ ] **Step 6: Implement `scripts/lib/derive.mjs`**

```js
import { parse as parseToml } from 'smol-toml';
import { INSTALL_BY_REGISTRY } from './paths.mjs';

// Parsers return { name, description } with undefined for anything absent.
// They never throw on shape: a missing field is reported by resolveEntry,
// which has enough context to name the offending id.
export function parseManifest(language, text) {
  if (language === 'python') {
    let data;
    try {
      data = parseToml(text);
    } catch {
      return { name: undefined, description: undefined };
    }
    const project = data.project ?? {};
    return { name: project.name, description: project.description };
  }

  if (language === 'node' || language === 'php') {
    let data;
    try {
      data = JSON.parse(text);
    } catch {
      return { name: undefined, description: undefined };
    }
    return { name: data.name, description: data.description };
  }

  return { name: undefined, description: undefined };
}

// Frontmatter is a leading `---` block. Handles `key: value` and the folded
// `key: >-` form used by the agent files.
export function parseFrontmatter(text) {
  const match = text.match(/^---\r?\n([\s\S]*?)\r?\n---/);
  if (!match) return {};
  const out = {};
  const lines = match[1].split(/\r?\n/);
  let currentKey = null;

  for (const line of lines) {
    const keyed = line.match(/^(\w[\w-]*):\s*(.*)$/);
    if (keyed) {
      const [, key, rawValue] = keyed;
      if (rawValue === '>-' || rawValue === '>' || rawValue === '|' || rawValue === '|-') {
        currentKey = key;
        out[key] = '';
      } else {
        currentKey = null;
        out[key] = rawValue.trim();
      }
      continue;
    }
    if (currentKey && /^\s+\S/.test(line)) {
      out[currentKey] = `${out[currentKey]} ${line.trim()}`.trim();
    }
  }
  return out;
}

// Resolves one catalog entry, or throws naming the id and the missing field.
// Precedence: manifest, then override. An override value of "" is a deliberate
// "none", which is different from omitting the key.
export function resolveEntry({ id, name, language, manifestText, registry, overrides }) {
  const override = overrides[id] ?? {};
  const manifest = manifestText ? parseManifest(language, manifestText) : {};

  const description = override.description ?? manifest.description;
  if (!description) {
    throw new Error(`${id}: no description. Add one to _data/catalog/overrides.yml.`);
  }

  const resolvedName = override.name ?? manifest.name ?? name ?? id;
  const resolvedRegistry = override.registry ?? registry ?? 'none';

  let install;
  if (override.install !== undefined) {
    install = override.install;
  } else if (resolvedRegistry !== 'none' && INSTALL_BY_REGISTRY[resolvedRegistry]) {
    install = INSTALL_BY_REGISTRY[resolvedRegistry](resolvedName);
  }
  if (install === undefined) {
    throw new Error(
      `${id}: no install command. Add one to _data/catalog/overrides.yml, ` +
        `or set install: "" there to record that there is none.`,
    );
  }

  return {
    id,
    name: resolvedName,
    type: override.type ?? 'tool',
    language,
    badge: override.badge ?? language,
    description,
    source: override.source ?? `https://github.com/${override.source_repo ?? 'drupaltools/tools'}`,
    source_path: override.source_path ?? '',
    install,
    registry: resolvedRegistry,
    docs: override.docs ?? '',
    homepage: override.homepage ?? '',
    status: override.status ?? 'active',
  };
}
```

- [ ] **Step 7: Write `_data/catalog/overrides.yml`**

```yaml
# Hand-maintained. Never written by scripts/sync-catalog.mjs.
# Each key matches the `id` of a generated entry.
#
# This file exists because four of the six packages in drupaltools/tools have no
# manifest that states a description or a published install. The values below
# come from each package's own docstring or README, verified 2026-09-28.

drupal-code-search:
  description: "Search Drupal contrib and core source code from the CLI via GitLab, Sourcegraph or tresbien.tech."
  install: "python3 drupal_code_search_tresbien.py \"query\""
  registry: none

drupal-versions:
  # Docstring of packages/python/drupal-versions/drupal_versions.py
  description: "Get the latest stable and supported Drupal versions from the CLI."
  install: "python3 drupal_versions.py"
  registry: none

module-info:
  # Docstring of packages/python/module-info/module-info.py
  description: "Find the Drupal module or theme that owns a given file or directory."
  install: "python3 module-info.py <path>"
  registry: none

drupal-news:
  # The only package in the monorepo with a published artefact.
  # Verified: https://pypi.org/pypi/drupal-news/json returns 200.
  # Description comes from pyproject.toml, install is derived from the registry.
  registry: pypi

drupal-web-search:
  # Published to PyPI? No: https://pypi.org/pypi/drupal-web-search/json returns 404.
  install: "pip install -e ."
  registry: none

stand-with-cyprus:
  # Published to Packagist? No: repo.packagist.org/p2/tplcom/stand-with-cyprus.json returns 404.
  # Empty install is deliberate: the package is not installable from a registry.
  install: ""
  registry: none
```

- [ ] **Step 8: Write `_data/catalog/manual.yml`**

```yaml
# Hand-maintained and appended to by .github/workflows/add-tool-from-issue.yml.
# Never written by scripts/sync-catalog.mjs.
# Same schema as generated.yml. `type` also accepts `other`.
[]
```

- [ ] **Step 9: Run the derive tests to verify they pass**

```bash
node --test scripts/__tests__/derive.test.mjs
```

Expected: 10 passing.

- [ ] **Step 10: Implement `scripts/lib/github.mjs`**

```js
import { readdirSync, readFileSync, statSync } from 'node:fs';
import { join } from 'node:path';
import { BRANCH } from './paths.mjs';

const API = 'https://api.github.com';
const RT = 'https://raw.githubusercontent.com';
const FIXTURES = 'scripts/__fixtures__';

function headers() {
  const token = process.env.GITHUB_TOKEN;
  const base = { Accept: 'application/vnd.github+json', 'User-Agent': 'drupaltools-catalog-sync' };
  return token ? { ...base, Authorization: `Bearer ${token}` } : base;
}

// Offline mode reads scripts/__fixtures__/<repo-name>/<path>, which mirrors the
// real repository layout. One fixture tree serves both the directory listings
// and the file reads, so there is nothing extra to keep in sync.
export function createClient({ offline }) {
  const fixturePath = (repo, path) => join(FIXTURES, repo.split('/')[1], path);

  async function listContents(repo, path) {
    if (offline) {
      let names;
      try {
        names = readdirSync(fixturePath(repo, path));
      } catch {
        throw new Error(`No fixture directory for ${repo}/${path}`);
      }
      return names
        .filter((name) => !name.startsWith('.'))
        .sort()
        .map((name) => ({
          name,
          type: statSync(fixturePath(repo, join(path, name))).isDirectory() ? 'dir' : 'file',
        }));
    }

    const response = await fetch(`${API}/repos/${repo}/contents/${path}?ref=${BRANCH}`, {
      headers: headers(),
    });
    if (!response.ok) {
      throw new Error(`GET ${repo}/${path} failed: ${response.status} ${response.statusText}`);
    }
    const listing = await response.json();
    return listing
      .map((entry) => ({ name: entry.name, type: entry.type === 'dir' ? 'dir' : 'file' }))
      .sort((a, b) => a.name.localeCompare(b.name));
  }

  async function readRaw(repo, path) {
    if (offline) {
      // Throws ENOENT for a path with no fixture, which is how the "no manifest
      // here, fall back to overrides" branch gets exercised in tests.
      return readFileSync(fixturePath(repo, path), 'utf8');
    }
    const response = await fetch(`${RT}/${repo}/${BRANCH}/${path}`);
    if (!response.ok) {
      throw new Error(`Raw ${repo}/${path} failed: ${response.status} ${response.statusText}`);
    }
    return response.text();
  }

  function readLocal(relativePath) {
    return readFileSync(offline ? fixturePath('mcp', relativePath) : relativePath, 'utf8');
  }

  return { listContents, readRaw, readLocal };
}
```

The listing is sorted in both branches so the generated YAML is byte-stable between
runs. Without that, the sync Action would open a commit every day for no reason.

- [ ] **Step 11: Implement `scripts/lib/emit.mjs`**

```js
import { renameSync, writeFileSync } from 'node:fs';
import yaml from 'js-yaml';

// Write to a temporary path and rename, so a crash mid-write cannot leave a
// half-written data file for Jekyll to read.
export function writeYamlAtomically(path, value) {
  const tmp = `${path}.tmp`;
  const body = yaml.dump(value, { lineWidth: 100, noRefs: true, quotingType: '"' });
  writeFileSync(tmp, body, 'utf8');
  renameSync(tmp, path);
}
```

- [ ] **Step 12: Implement `scripts/sync-catalog.mjs`**

```js
#!/usr/bin/env node
import { readFileSync } from 'node:fs';
import yaml from 'js-yaml';
import { createClient } from './lib/github.mjs';
import { parseFrontmatter, resolveEntry } from './lib/derive.mjs';
import { writeYamlAtomically } from './lib/emit.mjs';
import {
  AGENTS_DIR, GENERATED_FILE, LANGUAGE_BY_DIR, MANIFEST_BY_LANGUAGE, MCP_PACKAGE_JSON,
  OVERRIDES_FILE, PACKAGES_DIR, SKILLS_DIR, SKILLS_FILE, SKILLS_REPO, SKIP_PACKAGE_NAMES, TOOLS_REPO,
} from './lib/paths.mjs';

const offline = process.argv.includes('--offline');
const client = createClient({ offline });
const overrides = yaml.load(readFileSync(OVERRIDES_FILE, 'utf8')) ?? {};

async function buildToolEntries() {
  const languages = await client.listContents(TOOLS_REPO, PACKAGES_DIR);
  const entries = [];
  const pending = [];

  for (const languageDir of languages.filter((entry) => entry.type === 'dir')) {
    const language = LANGUAGE_BY_DIR[languageDir.name];
    if (!language) {
      console.warn(`skip: unknown language directory ${languageDir.name}`);
      continue;
    }
    const manifestFile = MANIFEST_BY_LANGUAGE[language];
    if (!manifestFile) {
      console.warn(`skip: no manifest convention for ${language}`);
      continue;
    }
    const packages = await client.listContents(TOOLS_REPO, `${PACKAGES_DIR}/${languageDir.name}`);
    for (const pkg of packages.filter((entry) => entry.type === 'dir')) {
      if (SKIP_PACKAGE_NAMES.has(pkg.name)) {
        console.log(`skip: ${pkg.name} is monorepo scaffolding, not a tool`);
        continue;
      }
      pending.push({ id: pkg.name, language, manifestFile, dir: languageDir.name });
    }
  }

  for (const item of pending) {
    const manifestPath = `${PACKAGES_DIR}/${item.dir}/${item.id}/${item.manifestFile}`;
    let manifestText = '';
    try {
      manifestText = await client.readRaw(TOOLS_REPO, manifestPath);
    } catch {
      console.warn(`note: ${item.id} has no ${item.manifestFile}, relying on overrides`);
    }
    const entry = resolveEntry({
      id: item.id,
      language: item.language,
      manifestText,
      registry: 'none',
      overrides,
    });
    entry.source_path = `${PACKAGES_DIR}/${item.dir}/${item.id}`;
    entries.push(entry);
  }

  return entries;
}

async function buildSkillsEntry() {
  const skillDirs = await client.listContents(SKILLS_REPO, SKILLS_DIR);
  const agentFiles = await client.listContents(SKILLS_REPO, AGENTS_DIR);
  const skills = skillDirs.filter((entry) => entry.type === 'dir').map((entry) => entry.name);
  const agents = agentFiles.filter((entry) => entry.type === 'file' && entry.name.endsWith('.md'));

  return {
    entry: {
      id: 'skills',
      name: 'skills & agents',
      type: 'skills',
      language: 'ai',
      badge: 'agent',
      description: `${skills.length} skills and ${agents.length} agents for Drupal development, review and audit work.`,
      source: `https://github.com/${SKILLS_REPO}`,
      source_path: '',
      install: `npx skills add https://github.com/${SKILLS_REPO}`,
      registry: 'none',
      docs: '',
      homepage: '',
      status: 'active',
      skills_count: skills.length,
      agents_count: agents.length,
      page: '/skills/',
    },
    skillNames: skills,
    agentFiles: agents,
  };
}

async function buildSkillsPage(skillNames, agentFiles) {
  const rows = [];
  for (const name of skillNames) {
    const path = `${SKILLS_DIR}/${name}/SKILL.md`;
    const meta = parseFrontmatter(await client.readRaw(SKILLS_REPO, path));
    rows.push({
      name: meta.name ?? name,
      kind: 'skill',
      description: meta.description ?? '',
      path,
      url: `https://github.com/${SKILLS_REPO}/blob/master/${path}`,
    });
  }
  for (const file of agentFiles) {
    const path = `${AGENTS_DIR}/${file.name}`;
    const meta = parseFrontmatter(await client.readRaw(SKILLS_REPO, path));
    rows.push({
      name: meta.name ?? file.name.replace(/\.md$/, ''),
      kind: 'agent',
      description: meta.description ?? '',
      path,
      url: `https://github.com/${SKILLS_REPO}/blob/master/${path}`,
    });
  }
  return rows;
}

async function buildMcpEntry() {
  const manifest = JSON.parse(client.readLocal(MCP_PACKAGE_JSON));
  if (!manifest.description) {
    throw new Error('mcp-package/package.json: no description.');
  }
  return {
    id: 'mcp',
    name: manifest.name,
    type: 'mcp',
    language: 'node',
    badge: 'mcp',
    description: manifest.description,
    source: 'https://github.com/drupaltools/catalog',
    source_path: 'mcp-package',
    install: `npx ${manifest.name}@latest`,
    registry: 'npm',
    docs: '',
    homepage: '',
    status: 'active',
    version: manifest.version,
  };
}

async function main() {
  const [tools, skills, mcp] = await Promise.all([
    buildToolEntries(),
    buildSkillsEntry(),
    buildMcpEntry(),
  ]);
  const skillsPage = await buildSkillsPage(skills.skillNames, skills.agentFiles);

  writeYamlAtomically(GENERATED_FILE, [mcp, skills.entry, ...tools]);
  writeYamlAtomically(SKILLS_FILE, skillsPage);

  console.log(`wrote ${GENERATED_FILE}: ${tools.length} tools, 1 skills entry, 1 mcp entry`);
  console.log(`wrote ${SKILLS_FILE}: ${skillsPage.length} rows`);
  return { tools: tools.length, skills: skillsPage.filter((r) => r.kind === 'skill').length, agents: skillsPage.filter((r) => r.kind === 'agent').length };
}

main().catch((error) => {
  console.error(`sync failed: ${error.message}`);
  process.exit(1);
});
```

- [ ] **Step 13: Write the offline sync test**

Create `scripts/__tests__/sync.test.mjs`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { execFileSync } from 'node:child_process';
import { readFileSync } from 'node:fs';
import yaml from 'js-yaml';

test('offline sync writes the expected catalog', () => {
  execFileSync('node', ['scripts/sync-catalog.mjs', '--offline'], { stdio: 'pipe' });

  const generated = yaml.load(readFileSync('_data/catalog/generated.yml', 'utf8'));
  const ids = generated.map((entry) => entry.id).sort();
  assert.deepEqual(ids, [
    'drupal-code-search', 'drupal-news', 'drupal-versions', 'drupal-web-search',
    'mcp', 'module-info', 'skills', 'stand-with-cyprus',
  ]);

  const skills = generated.find((entry) => entry.id === 'skills');
  assert.equal(skills.skills_count, 2);
  assert.equal(skills.agents_count, 2);
  assert.equal(skills.page, '/skills/');

  const news = generated.find((entry) => entry.id === 'drupal-news');
  assert.equal(news.install, 'pip install drupal-news');
  assert.equal(news.registry, 'pypi');

  const cyprus = generated.find((entry) => entry.id === 'stand-with-cyprus');
  assert.equal(cyprus.install, '');

  const rows = yaml.load(readFileSync('_data/catalog/skills.yml', 'utf8'));
  assert.equal(rows.length, 4);
  assert.equal(rows.filter((row) => row.kind === 'agent').length, 2);
});

test('offline sync exits 1 and names the id when a description is unresolvable', () => {
  // An empty package directory with no manifest and no override. The sync must
  // refuse to emit a card with a blank description.
  const dir = 'scripts/__fixtures__/tools/packages/python/mystery-tool';
  mkdirSync(dir, { recursive: true });
  try {
    assert.throws(
      () => execFileSync('node', ['scripts/sync-catalog.mjs', '--offline'], { stdio: 'pipe' }),
      /Command failed/,
    );
    assert.throws(
      () => execFileSync('node', ['scripts/sync-catalog.mjs', '--offline'], { stdio: 'pipe', encoding: 'utf8' }),
      /mystery-tool/,
    );
  } finally {
    rmSync(dir, { recursive: true, force: true });
  }
});
```

Add `mkdirSync` and `rmSync` to the `node:fs` import at the top of the test file:

```js
import { readFileSync, mkdirSync, rmSync } from 'node:fs';
```

- [ ] **Step 14: Run the sync tests to verify they pass**

```bash
node scripts/sync-catalog.mjs --offline
node --test scripts/__tests__/sync.test.mjs
```

Expected: exit 0, 2 passing. `_data/catalog/generated.yml` now has 8 entries.

**Important:** `--offline` writes fixture content straight over `_data/catalog/generated.yml`
and `_data/catalog/skills.yml`. It does not write to a scratch path. After any local
offline run, always do Step 15 before committing, or you will commit 2 skills and a
placeholder catalogue. In CI this is harmless, because the runner is ephemeral.

- [ ] **Step 15: Run against the live API to produce the real data**

```bash
GITHUB_TOKEN=$(gh auth token) node scripts/sync-catalog.mjs
cat _data/catalog/generated.yml
```

Expected: 6 tools, 1 skills entry, 1 mcp entry with real descriptions, and 43 rows in
`skills.yml` (32 skills, 11 agents). If the script exits 1 naming an id, add that id to
`overrides.yml` using the package's own docstring or README, then re-run. Do not guess.

- [ ] **Step 16: Commit**

```bash
git add scripts _data/catalog package.json package-lock.json
git commit -m "feat: add catalog sync script with overrides and fixtures"
```

---

### Task 3: Sync GitHub Action

**Files:**
- Create: `.github/workflows/sync-catalog.yml`

**Interfaces:**
- Consumes: `node scripts/sync-catalog.mjs` from Task 2
- Produces: `_data/catalog/generated.yml` and `_data/catalog/skills.yml` refreshed on a schedule, committed to `master`

- [ ] **Step 1: Write the workflow**

```yaml
name: Sync catalog data

on:
  schedule:
    # 04:17 UTC daily. Off the hour so we are not competing with every other cron.
    - cron: '17 4 * * *'
  workflow_dispatch:
  repository_dispatch:
    types: [catalog-sync]

concurrency:
  group: sync-catalog
  cancel-in-progress: false

jobs:
  sync:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Run sync
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: node scripts/sync-catalog.mjs

      - name: Commit if changed
        run: |
          if git diff --quiet -- _data/catalog; then
            echo "No catalog changes."
            exit 0
          fi
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add _data/catalog/generated.yml _data/catalog/skills.yml
          git commit -m "chore(catalog): sync generated data"
          git push
```

- [ ] **Step 2: Verify the commit guard locally**

```bash
git add -A && git commit -m "feat: add catalog sync workflow" 2>/dev/null || true
node scripts/sync-catalog.mjs --offline
git diff --quiet -- _data/catalog && echo "guard would skip" || echo "guard would commit"
```

Expected: `guard would skip` when the offline fixtures match what is already committed.
If it says `guard would commit`, run the live sync from Task 2 Step 15 and commit that
result instead, so the committed data matches production.

- [ ] **Step 3: Test the workflow with a manual dispatch**

```bash
git push -u origin HEAD
gh workflow run sync-catalog.yml
sleep 30
gh run list --workflow=sync-catalog.yml --limit 1
```

Expected: the run completes green. On the first run it may commit a data refresh, which
is correct.

- [ ] **Step 4: Commit and push**

```bash
git add .github/workflows/sync-catalog.yml
git commit -m "ci: sync catalog data on a schedule"
git push
```

---

### Task 4: Homepage card grid and filters

**Files:**
- Create: `_includes/catalog-card.html`, `_includes/catalog-filters.html`, `js/catalog-filters.js`
- Modify: `index.html` (full rewrite of the body), `_includes/head.html` (add the script tag)
- Test: `scripts/__tests__/homepage.test.mjs`

**Interfaces:**
- Consumes: `_data/catalog/generated.yml`, `_data/catalog/manual.yml` from Task 2
- Produces: `/index.html` with one `[data-card]` element per entry, each carrying `data-type` and `data-language`; `data-add-tool` card for the form link

- [ ] **Step 1: Write the failing build assertion**

Create `scripts/__tests__/homepage.test.mjs`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { readFileSync, existsSync } from 'node:fs';
import yaml from 'js-yaml';

test('homepage renders one card per catalog entry plus the add-tool card', () => {
  const html = readFileSync('_site/index.html', 'utf8');
  const generated = yaml.load(readFileSync('_data/catalog/generated.yml', 'utf8'));
  const manual = yaml.load(readFileSync('_data/catalog/manual.yml', 'utf8')) ?? [];

  const cards = html.match(/data-card="/g) ?? [];
  assert.equal(cards.length, generated.length + manual.length);

  for (const id of ['drupal-news', 'drupal-web-search', 'module-info', 'stand-with-cyprus', 'mcp', 'skills']) {
    assert.ok(html.includes(`data-card="${id}"`), `missing card for ${id}`);
  }

  assert.ok(html.includes('data-add-tool'), 'missing the add-tool card');
  assert.ok(html.includes('issues/new?template=add-tool.yml'), 'missing add-tool link');
  assert.ok(html.includes('data-type="skills"'), 'missing type attribute for filtering');
  assert.ok(html.includes('data-language="python"'), 'missing language attribute for filtering');
});

test('homepage filter chips cover every type present in the data', () => {
  const html = readFileSync('_site/index.html', 'utf8');
  for (const type of ['all', 'tool', 'skills', 'mcp']) {
    assert.ok(html.includes(`data-filter-type="${type}"`), `missing type chip ${type}`);
  }
});

test('the filter script is present and linked', () => {
  const html = readFileSync('_site/index.html', 'utf8');
  assert.ok(html.includes('/catalog/js/catalog-filters.js'));
  assert.ok(existsSync('_site/js/catalog-filters.js'));
});
```

- [ ] **Step 2: Build and run to verify it fails**

```bash
bundle exec jekyll build
node --test scripts/__tests__/homepage.test.mjs
```

Expected: FAIL, `missing card for drupal-news`.

- [ ] **Step 3: Write `_includes/catalog-card.html`**

```html
{% assign entry = include.entry %}
<article class="card" data-card="{{ entry.id }}" data-type="{{ entry.type }}" data-language="{{ entry.language }}">
  <header class="card-head">
    <span class="card-name">{{ entry.name }}</span>
    <span class="card-badge">{{ entry.badge }}</span>
  </header>
  <p class="card-description">{{ entry.description }}</p>
  {% if entry.install and entry.install != "" %}
    <code class="card-install">{{ entry.install }}</code>
  {% endif %}
  <footer class="card-links">
    {% if entry.page and entry.page != "" %}
      <a href="{{ site.baseurl }}{{ entry.page }}">browse</a>
    {% endif %}
    {% if entry.source and entry.source != "" %}
      <a href="{{ entry.source }}{% if entry.source_path and entry.source_path != '' %}/tree/master/{{ entry.source_path }}{% endif %}">source</a>
    {% endif %}
    {% if entry.registry == 'pypi' %}<a href="https://pypi.org/project/{{ entry.name }}/">pypi</a>{% endif %}
    {% if entry.registry == 'npm' %}<a href="https://www.npmjs.com/package/{{ entry.name }}">npm</a>{% endif %}
    {% if entry.registry == 'packagist' %}<a href="https://packagist.org/packages/{{ entry.name }}">packagist</a>{% endif %}
    {% if entry.docs and entry.docs != "" %}<a href="{{ entry.docs }}">docs</a>{% endif %}
  </footer>
</article>
```

- [ ] **Step 4: Write `_includes/catalog-filters.html`**

```html
{% assign entries = site.data.catalog.generated | concat: site.data.catalog.manual %}
{% assign languages = entries | map: 'language' | uniq | sort %}
<div class="catalog-filters">
  <div class="filter-row" role="group" aria-label="Filter by type">
    <button type="button" class="chip" data-filter-type="all" aria-pressed="true">All</button>
    <button type="button" class="chip" data-filter-type="tool" aria-pressed="false">Tools</button>
    <button type="button" class="chip" data-filter-type="skills" aria-pressed="false">Skills &amp; Agents</button>
    <button type="button" class="chip" data-filter-type="mcp" aria-pressed="false">MCP</button>
  </div>
  <div class="filter-row" role="group" aria-label="Filter by language">
    <button type="button" class="chip" data-filter-language="all" aria-pressed="true">All languages</button>
    {% for language in languages %}
      <button type="button" class="chip" data-filter-language="{{ language }}" aria-pressed="false">{{ language }}</button>
    {% endfor %}
  </div>
  <p class="filter-count"><span data-catalog-count>{{ entries | size }}</span> shown</p>
</div>
```

- [ ] **Step 5: Write `js/catalog-filters.js`**

```js
// Progressive enhancement: without JS every card stays visible, which is the
// correct fallback for a catalog page.
(function () {
  var grid = document.querySelector('[data-catalog-grid]');
  if (!grid) return;

  var cards = Array.prototype.slice.call(grid.querySelectorAll('[data-card]'));
  var countEl = document.querySelector('[data-catalog-count]');
  var state = { type: 'all', language: 'all' };

  function apply() {
    var shown = 0;
    cards.forEach(function (card) {
      var matchesType = state.type === 'all' || card.dataset.type === state.type;
      var matchesLanguage = state.language === 'all' || card.dataset.language === state.language;
      var visible = matchesType && matchesLanguage;
      card.hidden = !visible;
      if (visible) shown += 1;
    });
    if (countEl) countEl.textContent = String(shown);
  }

  function wire(attribute, key) {
    var chips = document.querySelectorAll('[data-filter-' + attribute + ']');
    Array.prototype.forEach.call(chips, function (chip) {
      chip.addEventListener('click', function () {
        state[key] = chip.dataset['filter' + attribute.charAt(0).toUpperCase() + attribute.slice(1)];
        Array.prototype.forEach.call(chips, function (other) {
          other.setAttribute('aria-pressed', String(other === chip));
        });
        apply();
      });
    });
  }

  wire('type', 'type');
  wire('language', 'language');
  apply();
})();
```

- [ ] **Step 6: Rewrite `index.html`**

```
---
layout: front
title: Tools we build
hero_title: Tools we build
---
{% assign entries = site.data.catalog.generated | concat: site.data.catalog.manual %}
{% include catalog-filters.html %}

<div class="catalog-grid" data-catalog-grid aria-live="polite">
  {% for entry in entries %}
    {% include catalog-card.html entry=entry %}
  {% endfor %}

  <article class="card card-add" data-add-tool>
    <header class="card-head">
      <span class="card-name">Add a tool</span>
    </header>
    <p class="card-description">Open an issue with the details and a bot commits the entry to the catalog.</p>
    <footer class="card-links">
      <a href="https://github.com/drupaltools/catalog/issues/new?template=add-tool.yml">open the form</a>
    </footer>
  </article>
</div>

<p class="catalog-community">
  Looking for third-party tools?
  <a href="{{ site.baseurl }}/all/">Browse the community list of 187 Drupal tools</a>.
</p>
```

- [ ] **Step 7: Register the script in `_includes/head.html`**

Add after the `scripts.js` line:

```html
  <script defer src="{{ "/js/catalog-filters.js" | prepend: site.baseurl }}" type="text/javascript"></script>
```

- [ ] **Step 8: Build and run the tests to verify they pass**

```bash
bundle exec jekyll build
node --test scripts/__tests__/homepage.test.mjs
node scripts/check-baseurl.mjs
```

Expected: 3 passing, and the baseurl check clean.

- [ ] **Step 9: Verify the filters by hand**

```bash
bundle exec jekyll serve --port 4000
```

Open `http://localhost:4000/catalog/`. Click "Tools" then "Skills & Agents", then click a
language chip, then "All". Expected: the count updates, cards hide and show, the pressed
chip is the only one with `aria-pressed="true"`, and the "Add a tool" card stays visible
under every filter. Disable JS and reload: all cards visible.

- [ ] **Step 10: Commit**

```bash
git add index.html _includes/catalog-card.html _includes/catalog-filters.html js/catalog-filters.js _includes/head.html scripts/__tests__/homepage.test.mjs
git commit -m "feat: render the org tool catalog on the homepage"
```

---

### Task 5: Skills and agents page

**Files:**
- Create: `skills/index.html`
- Test: `scripts/__tests__/skills-page.test.mjs`

**Interfaces:**
- Consumes: `_data/catalog/skills.yml` from Task 2
- Produces: `/skills/index.html` listing every row, split into `skill` and `agent` sections

- [ ] **Step 1: Write the failing assertion**

Create `scripts/__tests__/skills-page.test.mjs`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { readFileSync } from 'node:fs';
import yaml from 'js-yaml';

test('skills page lists every row from skills.yml, grouped by kind', () => {
  const rows = yaml.load(readFileSync('_data/catalog/skills.yml', 'utf8'));
  const html = readFileSync('_site/skills/index.html', 'utf8');

  const skillCount = rows.filter((row) => row.kind === 'skill').length;
  const agentCount = rows.filter((row) => row.kind === 'agent').length;

  assert.ok(html.includes(`${skillCount} skills`), `missing skill count ${skillCount}`);
  assert.ok(html.includes(`${agentCount} agents`), `missing agent count ${agentCount}`);

  for (const row of rows) {
    assert.ok(html.includes(row.url), `missing link for ${row.name}`);
  }
});
```

- [ ] **Step 2: Build and run to verify it fails**

```bash
bundle exec jekyll build
node --test scripts/__tests__/skills-page.test.mjs
```

Expected: FAIL, `ENOENT ... _site/skills/index.html`.

- [ ] **Step 3: Write `skills/index.html`**

```
---
layout: default
title: Skills and agents
hero_title: Skills and agents
---
{% assign rows = site.data.catalog.skills %}
{% assign skills = rows | where: "kind", "skill" %}
{% assign agents = rows | where: "kind", "agent" %}

<p class="section-intro">
  AI skills and agents for Drupal development, review and audit work. Install them with
  <code>npx skills add https://github.com/drupaltools/skills</code>.
</p>

<section class="listing">
  <h2 class="listing-title">{{ skills | size }} skills</h2>
  <ul class="listing-items">
    {% for row in skills %}
      <li class="listing-item">
        <a class="listing-name" href="{{ row.url }}">{{ row.name }}</a>
        <p class="listing-description">{{ row.description }}</p>
      </li>
    {% endfor %}
  </ul>
</section>

<section class="listing">
  <h2 class="listing-title">{{ agents | size }} agents</h2>
  <ul class="listing-items">
    {% for row in agents %}
      <li class="listing-item">
        <a class="listing-name" href="{{ row.url }}">{{ row.name }}</a>
        <p class="listing-description">{{ row.description | truncate: 240 }}</p>
      </li>
    {% endfor %}
  </ul>
</section>
```

Note: agent descriptions and some skill descriptions are long. Agent ones are truncated
to 240 characters for the list; the full text stays in `skills.yml`.

- [ ] **Step 4: Build and run to verify it passes**

```bash
bundle exec jekyll build
node --test scripts/__tests__/skills-page.test.mjs
node scripts/check-baseurl.mjs
```

Expected: 1 passing, baseurl check clean.

- [ ] **Step 5: Commit**

```bash
git add skills/index.html scripts/__tests__/skills-page.test.mjs
git commit -m "feat: add the skills and agents page"
```

---

### Task 6: Design tokens and base typography

**Files:**
- Create: `_sass/_tokens.scss`
- Modify: `css/main.scss` (import list), `_sass/_base.scss` (rewrite)

**Interfaces:**
- Consumes: nothing
- Produces: CSS custom properties on `:root` and a `@media (prefers-color-scheme: dark)` override block. Tasks 7 and 8 use only these variables; no task after this one introduces a raw colour literal.

**Before writing any CSS in this task or Task 7, invoke the `frontend-design` skill.**
The spec requires it, and the values below are a starting point to be refined against
its guidance rather than a finished design. Keep the same variable names when refining;
Task 7 consumes them.

- [ ] **Step 1: Write `_sass/_tokens.scss`**

```scss
// Single source of truth for colour and spacing. Every later partial reads these
// variables; nothing downstream hardcodes a hex value.
:root {
  --surface: #ffffff;
  --surface-sunken: #f9fafb;
  --border: #e4e7ec;
  --text: #101828;
  --text-muted: #475467;
  --text-subtle: #98a2b3;
  --accent: #1570ef;
  --accent-contrast: #ffffff;
  --chip-active-bg: #101828;
  --chip-active-text: #ffffff;
  --status-active: #12b76a;
  --status-quiet: #f79009;
  --status-archived: #98a2b3;

  --font-sans: ui-sans-serif, system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  --font-mono: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;

  --radius-sm: 4px;
  --radius-md: 6px;
  --radius-lg: 10px;
  --radius-pill: 99px;

  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-6: 24px;
  --space-8: 32px;
  --space-12: 48px;

  --content-width: 1080px;
}

@media (prefers-color-scheme: dark) {
  :root {
    --surface: #0d1117;
    --surface-sunken: #161b22;
    --border: #30363d;
    --text: #e6edf3;
    --text-muted: #8b98a5;
    --text-subtle: #6e7781;
    --accent: #58a6ff;
    --accent-contrast: #0d1117;
    --chip-active-bg: #e6edf3;
    --chip-active-text: #0d1117;
  }
}
```

- [ ] **Step 2: Rewrite `_sass/_base.scss`**

Replace the whole file. It currently sets a light-only baseline tied to the old
`$background-color` and `$text-color` Sass variables, which no longer exist.

```scss
*,
*::before,
*::after {
  box-sizing: border-box;
}

html {
  font-size: 16px;
  -webkit-text-size-adjust: 100%;
}

body {
  margin: 0;
  background: var(--surface);
  color: var(--text);
  font-family: var(--font-sans);
  line-height: 1.55;
}

a {
  color: var(--accent);
  text-decoration: none;
}

a:hover,
a:focus {
  text-decoration: underline;
}

code,
pre {
  font-family: var(--font-mono);
  font-size: 0.875em;
}

img {
  max-width: 100%;
  height: auto;
}

:focus-visible {
  outline: 2px solid var(--accent);
  outline-offset: 2px;
}

.wrapper {
  max-width: var(--content-width);
  margin: 0 auto;
  padding: 0 var(--space-4);
}
```

- [ ] **Step 3: Point `css/main.scss` at the new tokens**

Replace the import block at the bottom of `css/main.scss`:

```scss
// Import partials from `sass_dir` (defaults to `_sass`)
@import
        "tokens",
        "base",
        "layout",
        "components",
        "catalog"
;
```

`_layout.scss`, `_components.scss` and `_catalog.scss` are written in Task 7. To keep the
site building between this task and that one, create the three files now each containing
only a comment, then fill them in there.

`_syntax-highlighting.scss` drops out of the import list, so the Rouge styles for
highlighted code no longer ship. Nothing on the site renders a highlighted code block
today, but verify that after Step 4 by checking `_site/all/index.html` for `<div class="highlight">`.
If Rouge markup is present, re-add `"syntax-highlighting"` to the import list.

- [ ] **Step 4: Build and verify the tokens land in the CSS**

```bash
bundle exec jekyll build
grep -c -- '--surface:' _site/css/main.css
```

Expected: `2` (one in `:root`, one in the dark mode block).

- [ ] **Step 5: Commit**

```bash
git add _sass/_tokens.scss _sass/_base.scss css/main.scss
git commit -m "feat: add design tokens and base typography"
```

---

### Task 7: Card, filter and layout components

**Files:**
- Create: `_sass/_components.scss`, `_sass/_catalog.scss`
- Modify: `_sass/_layout.scss` (rewrite header, footer, page-content and intro rules)

**Interfaces:**
- Consumes: variables from `_sass/_tokens.scss` (Task 6)
- Produces: classes `.card`, `.card-head`, `.card-name`, `.card-badge`, `.card-description`, `.card-install`, `.card-links`, `.card-add`, `.catalog-grid`, `.catalog-filters`, `.filter-row`, `.chip`, `.filter-count`, `.search-input`, `.sort-by`, `.results`, `.site-header`, `.site-title`, `.site-nav`, `.site-footer`, `.page-content`, `.intro`, `.listing`, `.listing-title`, `.listing-items`, `.listing-item`, `.listing-name`, `.listing-description`, `.section-intro`, `.catalog-community`

- [ ] **Step 1: Write `_sass/_catalog.scss`**

```scss
.catalog-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: var(--space-4);
  margin-bottom: var(--space-8);
}

.card {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  padding: var(--space-4);
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: var(--radius-lg);
}

.card[hidden] {
  display: none;
}

.card-head {
  display: flex;
  align-items: baseline;
  gap: var(--space-2);
}

.card-name {
  font-family: var(--font-mono);
  font-size: 0.875rem;
  font-weight: 600;
  word-break: break-word;
}

.card-badge {
  padding: 1px 6px;
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  color: var(--text-muted);
  font-size: 0.6875rem;
}

.card-description {
  flex: 1;
  margin: 0;
  color: var(--text-muted);
  font-size: 0.8125rem;
}

.card-install {
  display: block;
  padding: 5px 8px;
  overflow-x: auto;
  background: var(--surface-sunken);
  border: 1px solid var(--border);
  border-radius: var(--radius-md);
  color: var(--text-muted);
  font-size: 0.6875rem;
  white-space: nowrap;
}

.card-links {
  display: flex;
  flex-wrap: wrap;
  gap: var(--space-3);
  font-size: 0.75rem;
}

.card-add {
  border-style: dashed;
}

.catalog-community {
  color: var(--text-muted);
  font-size: 0.875rem;
}
```

- [ ] **Step 2: Write `_sass/_components.scss`**

```scss
// Named catalog-filters, not filters: _includes/filters.html on the community
// page already owns the .filters class for its search and sort controls.
.catalog-filters {
  display: flex;
  flex-direction: column;
  gap: var(--space-3);
  margin-bottom: var(--space-6);
}

.filter-row {
  display: flex;
  flex-wrap: wrap;
  gap: var(--space-2);
}

.chip {
  padding: var(--space-1) var(--space-3);
  background: transparent;
  border: 1px solid var(--border);
  border-radius: var(--radius-pill);
  color: var(--text-muted);
  font-family: inherit;
  font-size: 0.8125rem;
  cursor: pointer;
}

.chip[aria-pressed="true"] {
  background: var(--chip-active-bg);
  border-color: var(--chip-active-bg);
  color: var(--chip-active-text);
}

.filter-count {
  margin: 0;
  color: var(--text-subtle);
  font-size: 0.75rem;
}

.listing {
  margin-bottom: var(--space-12);
}

.listing-title {
  margin: 0 0 var(--space-4);
  font-size: 1.125rem;
}

.listing-items {
  margin: 0;
  padding: 0;
  list-style: none;
  border-top: 1px solid var(--border);
}

.listing-item {
  padding: var(--space-3) 0;
  border-bottom: 1px solid var(--border);
}

.listing-name {
  font-family: var(--font-mono);
  font-size: 0.875rem;
  font-weight: 600;
}

.listing-description {
  margin: var(--space-1) 0 0;
  color: var(--text-muted);
  font-size: 0.8125rem;
}

.section-intro {
  margin-bottom: var(--space-8);
  color: var(--text-muted);
  font-size: 0.875rem;
}
```

- [ ] **Step 3: Rewrite `_sass/_layout.scss`**

The current file is 688 lines built around the removed Sass variables
(`$site-blue`, `$grey-color`, `$spacing-unit`, `$content-width`). Replace it entirely:

```scss
.site-header {
  border-bottom: 1px solid var(--border);
}

.site-header .wrapper {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: var(--space-4);
  padding-top: var(--space-4);
  padding-bottom: var(--space-4);
}

.site-title {
  color: var(--text);
  font-size: 0.9375rem;
  font-weight: 650;
  letter-spacing: -0.01em;
}

.site-header img {
  height: 18px;
  vertical-align: middle;
}

.site-nav {
  display: flex;
  gap: var(--space-4);
  margin-left: auto;
  font-size: 0.8125rem;
}

.site-nav a {
  color: var(--text-muted);
}

.intro {
  padding: var(--space-3) 0;
}

.page-content {
  padding: var(--space-8) 0 var(--space-12);
}

.page-content .title {
  margin: 0 0 var(--space-8);
  font-size: 1.5rem;
  font-weight: 650;
  letter-spacing: -0.02em;
}

.site-footer {
  padding: var(--space-6) 0;
  border-top: 1px solid var(--border);
}

.footer-col-wrapper {
  display: flex;
  flex-wrap: wrap;
  gap: var(--space-3);
  align-items: center;
}

.footer {
  color: var(--text-muted);
  font-size: 0.75rem;
}

.footer p {
  margin: 0 0 var(--space-2);
}

.footer img {
  height: 18px;
  margin-right: var(--space-1);
}
```

- [ ] **Step 4: Add the nav to `_includes/header.html`**

Insert after the `.site-title` anchor:

```html
    <nav class="site-nav">
      <a href="{{ site.baseurl }}/">Catalog</a>
      <a href="{{ site.baseurl }}/skills/">Skills</a>
      <a href="{{ site.baseurl }}/all/">Community</a>
      <a href="https://github.com/drupaltools/catalog">GitHub</a>
    </nav>
```

- [ ] **Step 5: Drop the tagline band from the homepage only**

`_includes/description.html` renders exactly this copy:

```html
<p>
  A list of popular <b>open source and free tools</b> that can help people accomplish <b>Drupal related tasks</b>.
</p>
```

That is still accurate for the community list, so keep the include in
`_layouts/default.html`. It is wrong for the new homepage, which now leads with the
organisation's own tools. Remove the `{% include description.html %}` line from
`_layouts/front.html` only, leaving the `{% include buttons-front.html %}` line that
follows it in place.

- [ ] **Step 6: Build and verify the CSS is complete**

```bash
bundle exec jekyll build
grep -c -- '\.card-name\|\.chip\|\.catalog-grid' _site/css/main.css
node scripts/check-baseurl.mjs
```

Expected: a non-zero count and a clean baseurl check.

- [ ] **Step 7: Check contrast and the dark variant**

```bash
bundle exec jekyll serve --port 4000
```

In the browser, on `http://localhost:4000/catalog/`:
- Toggle the OS dark preference and confirm no text becomes unreadable.
- Run a contrast check on `.card-description` against `.card` and on `.chip` in both
  pressed and unpressed states. Both must clear 4.5:1. If `.card-description` fails,
  darken `--text-muted`.

- [ ] **Step 8: Commit**

```bash
git add _sass/_components.scss _sass/_catalog.scss _sass/_layout.scss _includes/header.html
git commit -m "feat: retheme the catalog with new layout and card components"
```

---

### Task 8: Restyle the community list and drop its dead links

**Files:**
- Modify: `all/index.html`, `deprecated/index.html`, `index.html` (community anchor already added in Task 4), `about.html`
- Test: `scripts/__tests__/community-list.test.mjs`

**Interfaces:**
- Consumes: classes from Task 7
- Produces: `/all/index.html` with no link to a `/projects/` path

- [ ] **Step 1: Write the failing assertion**

Create `scripts/__tests__/community-list.test.mjs`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { readFileSync } from 'node:fs';

test('the community list has no link to the non-existent project detail pages', () => {
  for (const page of ['_site/all/index.html', '_site/deprecated/index.html']) {
    const html = readFileSync(page, 'utf8');
    assert.ok(!html.includes('class="js-more"'), `${page} still links to a project page that 404s`);
  }
});

test('the community list links each tool to its source', () => {
  const html = readFileSync('_site/all/index.html', 'utf8');
  assert.ok(html.includes('class="project-source"') || html.includes('card-links'));
});
```

- [ ] **Step 2: Build and run to verify it fails**

```bash
bundle exec jekyll build
node --test scripts/__tests__/community-list.test.mjs
```

Expected: FAIL, `_site/all/index.html still links to a project page that 404s`.

These links are already broken in production: `/projects/<name>/` returns 404 because
GitHub Pages ignores `_plugins/data_page_generator.rb` in the legacy build.

- [ ] **Step 3: Remove the dead "more details" link**

In `all/index.html` and `deprecated/index.html`, delete this line:

```html
        <a class="js-more" href="{{ project.name }}">more details</a>
```

Keep the "Edit on GitHub" link that follows it, and repoint it at the catalog repo:

```html
        <a class="project-edit" title="Edit on GitHub" href="https://github.com/drupaltools/catalog/edit/master/_data/projects/{{ project_hash[0] }}.yml" target="_blank">Edit</a>
```

Also remove the now-unused `✎` character from that link text.

- [ ] **Step 4: Restyle the community page's own filter bar**

`_includes/filters.html` holds the search box, sort select and the four
created/drupal/requires/category selects. It relies on `.filters`, `.search-input`,
`.sort-by` and `.results` rules that lived in the old `_sass/_layout.scss` and are now
gone, so those controls currently render unstyled. Add to `_sass/_components.scss`:

```scss
.search-input,
.sort-by,
.filter-created,
.filter-drupal,
.filter-requires,
.filter-category {
  padding: var(--space-2) var(--space-3);
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: var(--radius-md);
  color: var(--text);
  font-family: inherit;
  font-size: 0.8125rem;
}

.search-input {
  width: 100%;
  max-width: 320px;
}

.filters .search-input {
  display: block;
  margin-bottom: var(--space-3);
}

.results {
  color: var(--text-muted);
  font-size: 0.75rem;
}

.js-hidden-search {
  display: none;
}
```

Note `.filters` itself gets no rule here. It was a bare container in the old
stylesheet; the controls above provide the spacing.

- [ ] **Step 5: Swap the containers to the new classes**

In `all/index.html` and `deprecated/index.html`, change the outer wrapper:

```html
<div class="catalog-grid">
```

and change `class="column result-row"` to `class="card"`, keeping every `data-*`
attribute intact so `js/scripts.js` keeps working. Leave `.project-wrapper-tile`,
`.logo`, `.project-icons` and the filter fields as they are; `js/scripts.js` depends on
`.result-row`, `.description`, `data-name`, `data-category` and `data-requires`, so keep
`result-row` in the class list: use `class="card result-row"`.

- [ ] **Step 6: Build and verify**

```bash
bundle exec jekyll build
node --test scripts/__tests__/community-list.test.mjs
node scripts/check-baseurl.mjs
```

Expected: 2 passing, baseurl clean.

- [ ] **Step 7: Verify search still works on the community page**

```bash
bundle exec jekyll serve --port 4000
```

Open `http://localhost:4000/catalog/all/`, type `ddev` in the search box. Expected: the
result count changes and only matching cards remain. Then check the sort and filter
selects still change the list.

- [ ] **Step 8: Commit**

```bash
git add all/index.html deprecated/index.html scripts/__tests__/community-list.test.mjs
git commit -m "feat: restyle the community list and drop dead project links"
```

---

### Task 9: Add-a-tool issue form and action

**Files:**
- Create: `.github/ISSUE_TEMPLATE/add-tool.yml`, `.github/workflows/add-tool-from-issue.yml`, `scripts/lib/issue-form.mjs`
- Test: `scripts/__tests__/issue-form.test.mjs`

**Interfaces:**
- Consumes: `_data/catalog/manual.yml` from Task 2
- Produces: `parseIssueForm(body) -> { name, type, language, description, source, install, docs }`, throws `Error` naming the first missing required field

- [ ] **Step 1: Write the failing parser test**

Create `scripts/__tests__/issue-form.test.mjs`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { parseIssueForm } from '../lib/issue-form.mjs';

const BODY = `### Tool name

drupal-news

### Type

tool

### Language

python

### Description

Automated Drupal news aggregator with AI-powered summaries

### Source URL

https://github.com/drupaltools/tools

### Install command

pip install drupal-news

### Docs URL

_No response_
`;

test('parses a complete form body', () => {
  const parsed = parseIssueForm(BODY);
  assert.equal(parsed.name, 'drupal-news');
  assert.equal(parsed.type, 'tool');
  assert.equal(parsed.language, 'python');
  assert.equal(parsed.source, 'https://github.com/drupaltools/tools');
  assert.equal(parsed.install, 'pip install drupal-news');
});

test('treats _No response_ as empty', () => {
  const parsed = parseIssueForm(BODY);
  assert.equal(parsed.docs, '');
});

test('throws naming the missing field', () => {
  const missing = BODY.replace('drupal-news\n\n### Type', '\n### Type');
  assert.throws(() => parseIssueForm(missing), /Tool name/);
});

test('throws when the description exceeds 160 characters', () => {
  const long = BODY.replace(
    'Automated Drupal news aggregator with AI-powered summaries',
    'x'.repeat(161),
  );
  assert.throws(() => parseIssueForm(long), /160/);
});
```

- [ ] **Step 2: Run to verify it fails**

```bash
node --test scripts/__tests__/issue-form.test.mjs
```

Expected: FAIL, `Cannot find module '../lib/issue-form.mjs'`.

- [ ] **Step 3: Implement `scripts/lib/issue-form.mjs`**

```js
// GitHub renders an Issue Form submission as "### Label" headings followed by a
// blank line and the value. Empty optional fields render as "_No response_".
const LABELS = {
  'Tool name': 'name',
  Type: 'type',
  Language: 'language',
  Description: 'description',
  'Source URL': 'source',
  'Install command': 'install',
  'Docs URL': 'docs',
};

const REQUIRED = ['name', 'type', 'language', 'description', 'source'];
const MAX_DESCRIPTION = 160;

export function parseIssueForm(body) {
  const fields = {};
  const sections = body.split(/^###\s+/m).slice(1);

  for (const section of sections) {
    const newline = section.indexOf('\n');
    if (newline === -1) continue;
    const label = section.slice(0, newline).trim();
    const key = LABELS[label];
    if (!key) continue;
    const raw = section.slice(newline + 1).trim();
    fields[key] = raw === '_No response_' ? '' : raw;
  }

  for (const key of REQUIRED) {
    if (!fields[key]) {
      const label = Object.keys(LABELS).find((name) => LABELS[name] === key);
      throw new Error(`Missing required field: ${label}`);
    }
  }

  if (fields.description.length > MAX_DESCRIPTION) {
    throw new Error(
      `Description is ${fields.description.length} characters; the limit is ${MAX_DESCRIPTION}.`,
    );
  }

  for (const key of ['install', 'docs']) {
    if (fields[key] === undefined) fields[key] = '';
  }

  return fields;
}

export function toManualEntry(fields) {
  return {
    id: fields.name.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-|-$/g, ''),
    name: fields.name,
    type: fields.type,
    language: fields.language,
    badge: fields.language,
    description: fields.description,
    source: fields.source,
    source_path: '',
    install: fields.install,
    registry: 'none',
    docs: fields.docs,
    homepage: '',
    status: 'experimental',
  };
}
```

- [ ] **Step 4: Run to verify the parser tests pass**

```bash
node --test scripts/__tests__/issue-form.test.mjs
```

Expected: 4 passing.

- [ ] **Step 5: Write the issue form template**

Create `.github/ISSUE_TEMPLATE/add-tool.yml`:

```yaml
name: Add a tool
description: Add a tool, skill pack or service to the drupaltools catalog
title: "[Add tool] "
labels: ["add-tool"]
body:
  - type: markdown
    attributes:
      value: |
        Submitting this form adds a card to the catalog homepage. A maintainer
        must apply the `add-tool` label for the entry to be committed.
  - type: input
    id: name
    attributes:
      label: Tool name
      description: The name shown on the card. Use the package or repository name.
    validations:
      required: true
  - type: dropdown
    id: type
    attributes:
      label: Type
      options:
        - tool
        - skills
        - mcp
        - other
    validations:
      required: true
  - type: dropdown
    id: language
    attributes:
      label: Language
      options:
        - python
        - node
        - php
        - rust
        - go
        - ai
        - other
    validations:
      required: true
  - type: textarea
    id: description
    attributes:
      label: Description
      description: One line, 160 characters maximum. It is shown verbatim on the card.
      maxlength: 160
    validations:
      required: true
  - type: input
    id: source
    attributes:
      label: Source URL
      description: Link to the repository or source file.
    validations:
      required: true
  - type: input
    id: install
    attributes:
      label: Install command
      description: Leave blank if the tool is not installable from a registry.
    validations:
      required: false
  - type: input
    id: docs
    attributes:
      label: Docs URL
    validations:
      required: false
```

- [ ] **Step 6: Write the workflow**

Create `.github/workflows/add-tool-from-issue.yml`:

```yaml
name: Add tool from issue

on:
  issues:
    types: [opened, labeled]

permissions:
  contents: write
  issues: write

jobs:
  add:
    if: contains(github.event.issue.labels.*.name, 'add-tool')
    runs-on: ubuntu-latest
    steps:
      - name: Check author is a maintainer
        id: guard
        run: |
          case "${{ github.event.issue.author_association }}" in
            OWNER|MEMBER|COLLABORATOR) echo "allowed=true" >> "$GITHUB_OUTPUT" ;;
            *) echo "allowed=false" >> "$GITHUB_OUTPUT" ;;
          esac

      - name: Comment on a non-maintainer submission
        if: steps.guard.outputs.allowed == 'false'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh issue comment "${{ github.event.issue.number }}" \
            --repo "${{ github.repository }}" \
            --body "Thanks. A maintainer needs to review this and apply the \`add-tool\` label before it can be committed to the catalog."
          exit 0

      - uses: actions/checkout@v4
        if: steps.guard.outputs.allowed == 'true'

      - uses: actions/setup-node@v4
        if: steps.guard.outputs.allowed == 'true'
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
        if: steps.guard.outputs.allowed == 'true'

      - name: Append the entry
        if: steps.guard.outputs.allowed == 'true'
        env:
          ISSUE_BODY: ${{ github.event.issue.body }}
        run: node scripts/add-tool-from-issue.mjs

      - name: Commit and close
        if: steps.guard.outputs.allowed == 'true'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add _data/catalog/manual.yml
          git commit -m "chore(catalog): add ${{ github.event.issue.title }} from issue #${{ github.event.issue.number }}"
          git push
          gh issue comment "${{ github.event.issue.number }}" \
            --repo "${{ github.repository }}" \
            --body "Added to the catalog. It will appear on the homepage once the site rebuilds."
          gh issue close "${{ github.event.issue.number }}" \
            --repo "${{ github.repository }}"
```

- [ ] **Step 7: Write `scripts/add-tool-from-issue.mjs`**

```js
#!/usr/bin/env node
import { readFileSync } from 'node:fs';
import yaml from 'js-yaml';
import { parseIssueForm, toManualEntry } from './lib/issue-form.mjs';
import { writeYamlAtomically } from './lib/emit.mjs';

const MANUAL = '_data/catalog/manual.yml';

const body = process.env.ISSUE_BODY;
if (!body) {
  console.error('ISSUE_BODY is empty.');
  process.exit(1);
}

let fields;
try {
  fields = parseIssueForm(body);
} catch (error) {
  console.error(`Could not parse the issue body: ${error.message}`);
  process.exit(1);
}

const entry = toManualEntry(fields);
const existing = yaml.load(readFileSync(MANUAL, 'utf8')) ?? [];
const withoutDuplicate = existing.filter((item) => item.id !== entry.id);

writeYamlAtomically(MANUAL, [...withoutDuplicate, entry]);
console.log(`Added ${entry.id} to ${MANUAL}`);
```

- [ ] **Step 8: Add a duplicate-replacement test**

Append to `scripts/__tests__/issue-form.test.mjs`:

```js
import { toManualEntry } from '../lib/issue-form.mjs';

test('toManualEntry slugs the id and defaults status to experimental', () => {
  const entry = toManualEntry({
    name: 'Drupal News!', type: 'tool', language: 'python',
    description: 'A thing', source: 'https://example.com', install: '', docs: '',
  });
  assert.equal(entry.id, 'drupal-news');
  assert.equal(entry.status, 'experimental');
  assert.equal(entry.install, '');
});
```

```bash
node --test scripts/__tests__/issue-form.test.mjs
```

Expected: 5 passing.

- [ ] **Step 9: Test the parsers against a real issue body**

```bash
gh issue create --repo drupaltools/catalog \
  --template add-tool.yml \
  --title "[Add tool] parser smoke test" \
  --body "placeholder"
```

Expected: GitHub opens the form in a browser rather than accepting `--body` for a form
template. Instead, open `https://github.com/drupaltools/catalog/issues/new?template=add-tool.yml`
in a browser, submit the form once as yourself, and confirm the workflow either commits
the entry or comments. Then delete the test entry from `manual.yml` and close the issue.

- [ ] **Step 10: Commit**

```bash
git add .github/ISSUE_TEMPLATE/add-tool.yml .github/workflows/add-tool-from-issue.yml scripts/lib/issue-form.mjs scripts/add-tool-from-issue.mjs scripts/__tests__/issue-form.test.mjs
git commit -m "feat: add tools from a GitHub issue form"
```

---

### Task 10: Enable Pages and verify end to end

**Files:**
- Create: `.github/workflows/ci.yml`

**Interfaces:**
- Consumes: everything above
- Produces: a green CI check on pull requests and a live site at `https://drupaltools.github.io/catalog/`

- [ ] **Step 1: Write the CI workflow**

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [master]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.0'
          bundler-cache: true

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Sync catalog data offline
        run: node scripts/sync-catalog.mjs --offline

      - name: Script tests
        run: |
          node --test \
            scripts/__tests__/derive.test.mjs \
            scripts/__tests__/sync.test.mjs \
            scripts/__tests__/issue-form.test.mjs

      - name: Build site
        run: bundle exec jekyll build

      - name: Subpath check
        run: node scripts/check-baseurl.mjs

      - name: Page tests
        run: |
          node --test \
            scripts/__tests__/homepage.test.mjs \
            scripts/__tests__/skills-page.test.mjs \
            scripts/__tests__/community-list.test.mjs
```

The two test groups must stay separate. The page tests read `_site`, which does not
exist until the build runs, so lumping every test into one pre-build `node --test
scripts/__tests__/` call fails on a clean checkout.

`sync.test.mjs` runs the offline sync itself, so the explicit sync step above only
matters for producing the data the build consumes.

The offline sync regenerates `generated.yml` from fixtures, so CI validates the pipeline
rather than the committed content. That is deliberate: the committed content is validated
by the live sync Action on `master`.

- [ ] **Step 2: Push and confirm CI is green**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: build and test on pull requests"
git push
gh run list --limit 3
```

Expected: the CI run completes green.

- [ ] **Step 3: Enable GitHub Pages**

```bash
gh api -X POST repos/drupaltools/catalog/pages \
  -f "source[branch]=master" \
  -f "source[path]=/"
gh api repos/drupaltools/catalog/pages
```

Expected: `"build_type": "legacy"`, `"source": {"branch": "master", "path": "/"}`.

- [ ] **Step 4: Confirm the deployed site**

```bash
sleep 90
curl -s -o /dev/null -w "%{http_code}\n" https://drupaltools.github.io/catalog/
curl -s -o /dev/null -w "%{http_code}\n" https://drupaltools.github.io/catalog/skills/
curl -s -o /dev/null -w "%{http_code}\n" https://drupaltools.github.io/catalog/all/
curl -s -o /dev/null -w "%{http_code}\n" https://drupaltools.github.io/catalog/css/main.css
```

Expected: four `200`s.

- [ ] **Step 5: Confirm the original site is untouched**

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://www.drupaltools.com/
curl -s -o /dev/null -w "%{http_code}\n" https://www.drupaltools.com/all/
```

Expected: two `200`s. The original repo still owns the custom domain.

- [ ] **Step 6: Confirm the deployed homepage content**

```bash
curl -s https://drupaltools.github.io/catalog/ | grep -o 'data-card="[^"]*"' | sort
```

Expected: exactly the ids in `_data/catalog/generated.yml` plus `manual.yml`, and the
`data-add-tool` card.

- [ ] **Step 7: Final commit**

```bash
git add -A
git commit -m "docs: record the deployed catalog URL in the README"
git push
```

Update `README.md` to describe the catalog: what it lists, how the sync works, how to add
a tool, and the `/catalog` URL. Replace the current "Contributing" section, which
describes the old community-list workflow, with both: the issue form for org tools and
the existing YAML pull request flow for community tools.

---

## Self-review notes

**Spec coverage:** every spec section maps to a task. Divergence to Task 1, data model
to Task 2, sync Action to Task 3, homepage to Task 4, skills page to Task 5, styling
to Tasks 6 and 7, community list to Task 8, issue form to Task 9, Pages and rollout
verification to Task 10. Error-handling rows in the spec are covered by the fail-loud
tests in Task 2, the guard in Task 9, and the baseurl check in Task 1.

**Two spec refinements made while planning, both narrowing, neither contradicting:**

1. The spec said the sync derives `registry` from the manifest. It cannot: none of the
   Python packages declare a registry, and checking PyPI or Packagist per package would
   add three network dependencies to every sync. `registry` now comes from
   `overrides.yml` and defaults to `none`. Task 2 Step 15 records the verified registry
   facts.
2. The spec made a missing `install` a hard failure. That is kept, with one escape:
   an override may set `install: ""` to record a deliberate "no install". This matters
   for `stand-with-cyprus`, which is not on Packagist. Without the escape the only ways
   forward would be inventing a command or omitting the package.

**Deliberately not covered:** the `mcp-server/` copy rewrite and the `stand-with-cyprus`
inclusion question. You chose to keep both as they are, and both are recorded as accepted
risks in the spec.
