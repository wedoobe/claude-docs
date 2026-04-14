# Claude Docs — Wedoobe

Reference files and guides for working with Claude Code, GitHub MCP, and Odoo 19 theme development.

---

## Table of Contents

1. [Setting up Claude Code in the terminal](#1-setting-up-claude-code-in-the-terminal)
2. [Connecting GitHub via MCP](#2-connecting-github-via-mcp)
3. [Creating a new Odoo 19 theme](#3-creating-a-new-odoo-19-theme)

---

## 1. Setting up Claude Code in the terminal

### Install

Claude Code requires Node.js 18 or higher.

```bash
npm install -g @anthropic-ai/claude-code
```

### Authenticate

```bash
claude
```

On first run it will open a browser window to log in with your Anthropic account. Follow the prompts.

### Start a session

```bash
claude
```

Or open it in a specific project folder:

```bash
cd /path/to/your/project
claude
```

### Key commands inside Claude Code

| Command | What it does |
|---------|-------------|
| `/help` | Show available commands |
| `/clear` | Clear the conversation |
| `/exit` | Exit Claude Code |
| `! <command>` | Run a shell command directly (e.g. `! git status`) |

---

## 2. Connecting GitHub via MCP

MCP (Model Context Protocol) lets Claude Code talk to external services like GitHub.

### Add the GitHub MCP server

Run this once in your Claude Code session:

```bash
claude mcp add-json github '{"type":"http","url":"https://api.githubcopilot.com/mcp","headers":{"Authorization":"Bearer YOUR_GITHUB_TOKEN"}}'
```

Replace `YOUR_GITHUB_TOKEN` with a GitHub Personal Access Token.  
Generate one at: **GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens**

Required token permissions:
- **Contents** — read & write (to create/update files)
- **Metadata** — read

### Verify the connection

```bash
claude mcp list
```

You should see `github: https://api.githubcopilot.com/mcp (HTTP) - Connected`.

### What you can do with GitHub MCP

Once connected, Claude Code can:
- Read and write files directly in any GitHub repo (no local clone needed)
- Create and manage repositories
- List branches, commits, and file trees
- Push commits via the GitHub API

### Useful `gh` CLI commands

The `gh` CLI works alongside MCP for repo management:

```bash
gh repo list                          # list your repos
gh repo create wedoobe/my-repo --private   # create a new repo
gh api "repos/wedoobe/my-repo/git/trees/HEAD?recursive=1" --jq '.tree[].path'  # list all files
```

---

## 3. Creating a new Odoo 19 theme

Always follow this workflow before writing any code:

1. **Ask for a design** — get a screenshot, mockup, or reference website
2. **Cut the design into sections** — identify every distinct visual block (hero, cards, expertise grid, CTA, footer, etc.)
3. **Map each section to a block** — each block gets its own XML, SCSS, and optionally JS
4. **Never start building** until the design is confirmed and sections are agreed upon

---

### Module naming

- Prefix: `website_` (e.g. `website_defender`)
- Lowercase, alphanumerics and underscores only

---

### Module structure

```
website_<name>/
├── __init__.py
├── __manifest__.py
├── data/
│   ├── presets.xml
│   └── pages/
│       └── home.xml
├── views/
│   └── website_templates.xml      # header/footer overrides
└── static/
    └── src/
        └── scss/
            ├── primary_variables.scss
            ├── bootstrap_overridden.scss
            ├── theme.scss
            ├── layout/
            │   └── header.scss
            └── pages/
                └── home.scss
```

---

### `__init__.py`

```python
# -*- coding: utf-8 -*-
```

---

### `__manifest__.py`

```python
# -*- coding: utf-8 -*-
{
    'name': 'My Theme',
    'description': 'Website theme for mysite.be',
    'category': 'Theme/Corporate',
    'version': '19.0.1.0.0',
    'author': 'wedoobe',
    'license': 'LGPL-3',
    'depends': ['website'],
    'data': [
        'data/presets.xml',
        'data/pages/home.xml',
        'views/website_templates.xml',
    ],
    'assets': {
        'web._assets_primary_variables': [
            'website_<name>/static/src/scss/primary_variables.scss',
        ],
        'web._assets_frontend_helpers': [
            ('prepend', 'website_<name>/static/src/scss/bootstrap_overridden.scss'),
        ],
        'web.assets_frontend': [
            'website_<name>/static/src/scss/theme.scss',
            'website_<name>/static/src/scss/layout/header.scss',
            'website_<name>/static/src/scss/pages/home.scss',
        ],
    },
}
```

> **Never use wildcards** like `scss/*.scss` — always list files explicitly.

---

### `primary_variables.scss`

Use individual color variables (not the `$o-colors` map) to avoid SCSS compilation errors on fresh installs:

```scss
// Fonts
$o-theme-font-urls: (
    'Oswald': 'https://fonts.googleapis.com/css2?family=Oswald:wght@400;600;700&display=swap',
) !default;

$o-theme-fonts: (
    'Oswald': (
        'family': "'Oswald', sans-serif",
        'type': 'sans-serif',
    ),
) !default;

$o-headings-font-name: 'Oswald' !default;
$o-base-font-name: 'Open Sans' !default;

// Colors — use individual vars, NOT the $o-colors map
$o-color-1: #111111 !default;
$o-color-2: #C8A96E !default;
$o-color-3: #1A1A1A !default;
$o-color-4: #FFFFFF !default;
$o-color-5: #0A0A0A !default;

$o-header-template: 'default' !default;
$o-footer-template: 'default' !default;
```

---

### `bootstrap_overridden.scss`

Variable definitions only — no output CSS:

```scss
$h1-font-size: 3.5rem !default;
$h2-font-size: 2.75rem !default;
$btn-border-radius: 0 !default;
$btn-font-weight: 700 !default;
$input-border-radius: 0 !default;
$navbar-padding-y: 1.25rem !default;
```

---

### `data/presets.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
</odoo>
```

---

### `data/pages/home.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo noupdate="1">
    <record id="page_home" model="website.page">
        <field name="name">Home</field>
        <field name="is_published" eval="True"/>
        <field name="key">website_<name>.page_home</field>
        <field name="url">/</field>
        <field name="type">qweb</field>
        <field name="arch" type="xml">
            <t t-name="website_<name>.page_home">
                <t t-call="website.layout">
                    <t t-set="additional_title">Page Title</t>
                    <div id="wrap" class="oe_structure">

                        <!-- Section 1: Hero -->
                        <section class="s_hero o_cc o_cc1">
                            ...
                        </section>

                        <!-- Section 2: Cards -->
                        <section class="s_cards o_cc o_cc2 pt80 pb80">
                            ...
                        </section>

                    </div>
                </t>
            </t>
        </field>
    </record>
</odoo>
```

---

### Deploying on Odoo.sh

1. Push to `main` → Odoo.sh triggers a rebuild automatically
2. If the branch is locked by an AI session → click **Rebuild** in the Odoo.sh dashboard
3. After rebuild → **Apps → Update Apps List → install or upgrade your theme**
4. **Rebuild alone does NOT apply XML/template changes** — you must also upgrade the module

---

### Common SCSS gotchas

- Do not use `lighten()` / `darken()` — use literal hex values instead (Dart Sass deprecation)
- Do not use the `$o-colors` map directly — use individual `$o-color-1` … `$o-color-5` variables
- Never use wildcards in asset paths
- All rules must be scoped inside `#wrapwrap`
- `primary_variables.scss` and `bootstrap_overridden.scss` must contain **only variable definitions** — no output CSS

---

## Files in this repo

| File | Purpose |
|------|---------|
| `CLAUDE.md` | Quick rules and gotchas for Claude Code when working on Odoo themes |
| `ODOO_REFERENCE.md` | Full Odoo 19 website theme API reference |
| `README.md` | This file — setup and workflow guide |
