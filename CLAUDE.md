# CLAUDE.md

Guidance for Claude Code when working in this repository.
Full API reference → see `ODOO_REFERENCE.md`.

## Project

Odoo 19.0 website theme module, deployed on **Odoo.sh**.
Stack: XML/QWeb templates, SCSS, JavaScript.

## Module Naming

- Prefix: `website_` (e.g. `website_airproof`)
- Lowercase ASCII alphanumerics and underscores only
- Template names: same rule — lowercase alphanumerics and underscores only
- Always add an empty line at the end of every XML file

## Odoo.sh Deployment

1. Push to `main` → Odoo.sh rebuilds automatically
2. If build is stuck on an AI session → click **Rebuild** in Odoo.sh dashboard
3. After rebuild → **Apps → [Module] → Upgrade** in the Odoo backend
4. **Rebuild alone does NOT apply XML/template changes** — you must also upgrade the module

## Module Structure

```
<module_name>/
├── __init__.py
├── __manifest__.py
├── i18n/                              # .po translation files
├── lib/                               # External JS libraries
├── data/
│   ├── presets.xml                    # Enable/disable built-in Odoo views
│   ├── website.xml                    # Site name, logo, favicon, social links
│   ├── images.xml                     # Media library images (ir.attachment)
│   ├── shapes.xml                     # Custom SVG shapes (ir.attachment)
│   └── pages/                         # Page records (home.xml, about.xml, …)
├── views/
│   ├── website_templates.xml          # Header, footer, general layout overrides
│   ├── website_sale_templates.xml     # eCommerce overrides
│   ├── new_page_template_templates.xml
│   └── snippets/
│       ├── snippets.xml               # Snippet registration
│       └── s_<name>.xml               # Per-snippet templates
└── static/
    ├── fonts/                         # .woff / .woff2
    ├── image_shapes/                  # SVG image clipping masks
    ├── shapes/                        # SVG background shapes
    └── src/
        ├── img/
        │   ├── content/               # Images, icons, branding
        │   ├── snippets/              # Snippet preview images
        │   └── wbuilder/              # Website Builder thumbnails
        ├── js/
        ├── scss/
        │   ├── primary_variables.scss
        │   ├── bootstrap_overridden.scss
        │   ├── fonts.scss
        │   ├── theme.scss
        │   ├── layout/
        │   ├── components/
        │   ├── snippets/
        │   └── pages/
        ├── snippets/
        │   └── s_<name>/
        │       ├── <name>.js
        │       ├── <name>.edit.js
        │       ├── 000.scss
        │       └── 000.xml
        └── website_builder/           # JS option plugins + XML templates
```

## Asset Bundles (which file goes where)

| File | Bundle |
|------|--------|
| `primary_variables.scss` | `web._assets_primary_variables` |
| `bootstrap_overridden.scss` | `web._assets_frontend_helpers` with **`prepend`** |
| `fonts.scss`, `theme.scss`, custom JS | `web.assets_frontend` |
| Website Builder JS/XML plugins | `website.website_builder_assets` |

**Critical:** Never use wildcards (e.g. `myfolder/*.scss`) — they don't work on Odoo SaaS. List every file manually.

## SCSS Rules

- `primary_variables.scss` and `bootstrap_overridden.scss` must contain **only variable definitions and mixin overrides** — no output CSS
- All custom CSS rules go in `theme.scss` or component files
- Always scope custom rules inside `#wrapwrap`
- Never override Bootstrap variables that depend on Odoo variables — it breaks user customization via the Website Builder. If an option exists in both `primary_variables.scss` and Bootstrap, always override via primary variables

## Critical Odoo 19 Gotchas

### QWeb — no Python builtins
```xml
<!-- WRONG — KeyError at render time -->
<header t-attf-class="#{' overlay' if hasattr(obj, 'field') and obj.field else ''}"/>

<!-- CORRECT -->
<header t-attf-class="#{' overlay' if main_object and main_object.get('field') else ''}"/>
```

### QWeb — translatable attributes
```xml
<!-- CORRECT — treated as translatable -->
<div t-attf-title="Hello #{user.name}" />

<!-- WARNING — NOT translatable -->
<div t-att-title="'Hello' + user.name" />
```

### Broken eCommerce XPaths (don't exist in Odoo 19)
- `//t[@t-set='columns']` in `website_sale.products`
- `//div[hasclass('css_quantity')]` in `website_sale.product`

Always inspect live templates in developer mode before writing XPath overrides.

### Snippets — registration
- Use `group` (singular) to link a snippet to a group — never `groups` (that's for access rights)
- `data-name` and `data-snippet` **must** be set when a snippet is declared on a theme page
- Never put a `<section>` inside another `<section>` — use inner content snippets instead

### noupdate — editing source language breaks translations
Every modification to the source language (in the Website Builder or source code) breaks the link with existing translations. Re-create translations after editing source content.

### Wildcards don't work on SaaS
Never use `myfolder/*.scss` in `__manifest__.py` — always list files explicitly.

### Rebuild ≠ Upgrade
Pushing to `main` and rebuilding does not apply XML/template changes. You must also upgrade the module.

## Design Philosophy

Always use Odoo's default options first. Enable the default, then extend it. Recoding a default option risks breaking user customization and other Odoo features that depend on it.

## Extending Templates — file conventions

- General layout overrides → `website_templates.xml`
- eCommerce overrides → `website_sale_templates.xml`
- Blog overrides → `website_blog_templates.xml`
- Always update the module when creating a new template or record

## JavaScript Conventions

- Use native JS modules
- `js_` prefix for CSS classes targeted by JS
- `camelCase` for variables/functions, `PascalCase` for classes
- Use `===` not `==`, double quotes for strings
- **Never name a variable `event`** — use `ev` (Chrome's global `event` silently overrides it on other browsers)
- Always call parent when overriding: `super.start(...arguments)`
- Use minified external libraries; use ESLint

## Dependencies

```python
'depends': ['website']               # always required
# Add as needed:
# 'website_sale'           eCommerce
# 'website_sale_wishlist'  Wishlist
# 'website_blog'           Blog
# 'website_mass_mailing'   Newsletter — only add when actively using it
```

## Going Live

**Odoo SaaS:** ZIP the module → Apps → install `base_import_module` → Import Module → Force init → Import App. ZIP < 50MB. On re-imports: Become Superuser first, do NOT tick Force init again.

**Odoo.sh:** Apps → Update Apps List → install.

Before launch: check SEO, URL redirects, domain name configuration.
