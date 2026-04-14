# ODOO_REFERENCE.md

Complete Odoo 19.0 website theme API reference.
Quick rules and gotchas → see `CLAUDE.md`.

---

## Table of Contents

1. [__manifest__.py](#manifestpy)
2. [Page Layout](#page-layout)
3. [XPath](#xpath)
4. [QWeb & t-call](#qweb--t-call)
5. [Pages](#pages)
6. [Header](#header)
7. [Footer](#footer)
8. [Colors & Palette](#colors--palette)
9. [Fonts](#fonts)
10. [Gradients](#gradients)
11. [Bootstrap Variables](#bootstrap-variables)
12. [Website Settings](#website-settings)
13. [Presets](#presets)
14. [Media — Images](#media--images)
15. [Media — Videos](#media--videos)
16. [Media — Icons](#media--icons)
17. [Background Shapes](#background-shapes)
18. [Image Shapes](#image-shapes)
19. [Building Blocks — Layout & CSS](#building-blocks--layout--css)
20. [Building Blocks — Custom Snippet Workflow](#building-blocks--custom-snippet-workflow)
21. [Building Blocks — Options API](#building-blocks--options-api)
22. [Animations](#animations)
23. [Forms](#forms)
24. [Translations](#translations)
25. [Going Live](#going-live)

---

## `__manifest__.py`

```python
{
    'name': 'My Theme',
    'description': '...',           # reStructuredText format
    'category': 'Website/Theme',
    'version': '19.0.1.0.0',        # odoo_major.module_major.module_minor.module_patch
    'author': '...',
    'license': 'LGPL-3',
    'depends': ['website'],
    'data': [
        # List all XML files explicitly — wildcards fail on Odoo SaaS
        'data/presets.xml',
        'data/website.xml',
        'data/images.xml',
        'data/pages/home.xml',
        'views/website_templates.xml',
        'views/snippets/snippets.xml',
        'views/snippets/s_mysnippet.xml',
    ],
    'assets': {
        'web._assets_primary_variables': [
            'module_name/static/src/scss/primary_variables.scss',
        ],
        'web._assets_frontend_helpers': [
            ('prepend', 'module_name/static/src/scss/bootstrap_overridden.scss'),
        ],
        'web.assets_frontend': [
            'module_name/static/src/scss/fonts.scss',
            'module_name/static/src/scss/theme.scss',
            'module_name/static/src/js/theme.js',
        ],
        'website.website_builder_assets': [
            'module_name/static/src/website_builder/my_option_plugin.js',
            'module_name/static/src/website_builder/my_option.xml',
        ],
    },
    # Optional: custom new-page templates
    'new_page_templates': {
        'airproof': {
            'faq': ['s_airproof_text_block_h1', 's_title', 's_faq_collapse', 's_call_to_action']
        }
    }
}
```

### Asset bundle reference

| Bundle | Purpose |
|--------|---------|
| `web._assets_primary_variables` | `primary_variables.scss` only |
| `web._assets_secondary_variables` | `secondary_variables.scss` only |
| `web._assets_frontend_helpers` | `bootstrap_overridden.scss` (must use `prepend`) |
| `web.assets_frontend` | All other SCSS, JS, and QWeb JS files |
| `website.website_builder_assets` | JS/XML plugins for Website Builder options, shapes, snippets |
| `web._assets_bootstrap_frontend` | Bootstrap Utilities API extensions |

### `__manifest__.py` field reference

| Field | Description |
|-------|-------------|
| `name` | Human-readable module name (required) |
| `description` | Extended description in reStructuredText |
| `category` | Classification within Odoo (use `Website/Theme`) |
| `version` | `odoo_major.module_major.module_minor.module_patch` — patch optional |
| `author` | Module author name |
| `license` | Default: `LGPL-3` |
| `depends` | Modules that must be loaded first |
| `data` | List of XML files to load |
| `assets` | Dict of SCSS/JS files by bundle |

---

## Page Layout

Every Odoo page is wrapped in `#wrapwrap`:

```html
<div id="wrapwrap">
    <header/>
    <main>
        <div id="wrap" class="oe_structure">
            <!-- Page Content -->
        </div>
    </main>
    <footer/>
</div>
```

All XML files start with:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<odoo>
    ...
</odoo>
```

---

## XPath

Used to extend existing Odoo templates:
```xml
<template id="my_override" inherit_id="module.template_id" name="Human-readable name">
    <xpath expr="..." position="...">
        <!-- Content -->
    </xpath>
</template>
```

Use the same `id` as the original record — final IDs are prefixed by module name so there's no overlap.

### Descendent selectors

| Selector | Selects |
|----------|---------|
| `/` | From the root node |
| `//` | Anywhere in the document |
| `*[@id="id"]` | Specific ID |
| `*[hasclass("class")]` | Specific class |
| `*[@name="name"]` | Tag with specific name attribute |
| `*[@t-call="template"]` | Specific `t-call` |

### Position values

| Position | Description |
|----------|-------------|
| `replace` | Replaces the targeted node entirely |
| `inside` | Appends inside the targeted node |
| `before` | Inserts before the targeted node |
| `after` | Inserts after the targeted node |
| `attributes` | Adds or modifies attributes on the node |

### Attribute manipulation

```xml
<!-- Remove a class -->
<xpath expr="//header" position="attributes">
    <attribute name="class" remove="x_airproof_header" />
</xpath>

<!-- Add a class (separator adds a space before the new class) -->
<xpath expr="//header" position="attributes">
    <attribute name="class" add="x_airproof_header" separator=" " />
</xpath>
```

### Moving elements

```xml
<xpath expr="//div[@id='footer']" position="before">
    <xpath expr="//div[@id='o_footer_scrolltop_wrapper']" position="move" />
</xpath>
```

When using `move` inside an XPath, that XPath block can only contain `move` directives.

### Good vs bad XPath with move

```xml
<!-- GOOD — separate XPaths -->
<xpath expr="//*[hasclass('o_wsale_products_main_row')]" position="before">
    <xpath expr="//t[@t-if='opt_wsale_categories_top']" position="move" />
</xpath>
<xpath expr="//*[hasclass('o_wsale_products_main_row')]" position="before">
    <div><!-- content --></div>
</xpath>

<!-- BAD — mixing move and content in same XPath -->
<xpath expr="//*[hasclass('o_wsale_products_main_row')]" position="before">
    <xpath expr="//t[@t-if='opt_wsale_categories_top']" position="move" />
    <div><!-- content --></div>
</xpath>
```

---

## QWeb & `t-call`

QWeb is Odoo's XML templating engine. `t-call` includes a sub-template and can pass parameters.

### Parameter syntax (two styles)

```xml
<!-- Old style — t-set inside t-call -->
<t t-call="portal.user_dropdown">
    <t t-set="_icon" t-value="True" />
    <t t-set="_item_class" t-valuef="dropdown" />
</t>

<!-- New parametric style (preferred) -->
<t t-call="portal.user_dropdown"
    _icon="true"
    _item_class.f="dropdown" />
```

### Parameter suffix meanings

| Suffix | Description |
|--------|-------------|
| `_icon="true"` | Raw value (boolean `true`) |
| `_item_class.f="dropdown"` | Pass value as string |
| `additional_title.translate="My Page"` | Pass as translatable string |

### `website.layout` parameters

```xml
<t t-call="website.layout"
    additional_title.translate="My Page Title"
    meta_description.translate="Description shown in search engines."
    pageName.f="homepage">
    <div id="wrap" class="oe_structure oe_empty" />
</t>
```

### Translatable strings

Always use `t-attf-` for translatable attributes:
```xml
<!-- CORRECT — treated as translatable -->
<div t-attf-title="Hello #{user.name}" />

<!-- WARNING — works but NOT translatable -->
<div t-att-title="'Hello' + user.name" />
```

`t-value` and `t-valuef` are NOT explicitly translatable. Text between two XML tags IS translatable.

Reuse a translatable string in multiple places:
```xml
<t t-set="label">Foo</t>
<div t-att-title="label" />
<nav t-att-title="label" />
```

Pass translatable string via `t-call`:
```xml
<t t-call="website.layout" additional_title.translate="My Page" />
```

### Do NOT use Python builtins in QWeb expressions

```xml
<!-- WRONG — hasattr is not available, causes KeyError -->
<header t-attf-class="#{' overlay' if hasattr(obj, 'field') and obj.field else ''}"/>

<!-- CORRECT -->
<header t-attf-class="#{' overlay' if main_object and main_object.get('field') else ''}"/>
```

---

## Pages

### Default pages (Home, Contact Us, 404, …)

Built into Odoo as templates. To deactivate:
```xml
<record id="website.homepage" model="ir.ui.view">
    <field name="active" eval="False"/>
</record>
<record id="website.contactus" model="ir.ui.view">
    <field name="active" eval="False"/>
</record>
```

Replace content via XPath:
```xml
<template id="404" inherit_id="http_routing.404">
    <xpath expr="//t[@t-call='web.frontend_layout']" position="attributes">
        <attribute name="additional_title.translate">404 – Not found</attribute>
    </xpath>
    <xpath expr="//*[@id='wrap']" position="replace">
        <div id="wrap" class="oe_structure">
            <!-- Content -->
        </div>
    </xpath>
</template>
```

### Theme pages (`website.page` records)

Create in `data/pages/*.xml`. Always use `noupdate="1"` to protect user edits on upgrade:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo noupdate="1">
    <record id="page_about_us" model="website.page">
        <field name="name">About us</field>
        <field name="is_published" eval="True"/>
        <field name="key">module_name.page_about_us</field>
        <field name="url">/about-us</field>
        <field name="website_id" eval="1"/>
        <field name="type">qweb</field>
        <field name="header_overlay" eval="True"/>
        <field name="header_visible" eval="True"/>
        <field name="footer_visible" eval="True"/>
        <field name="arch" type="xml">
            <t t-name="module_name.page_about_us">
                <t t-call="website.layout">
                    <div id="wrap" class="oe_structure">
                        <!-- content -->
                    </div>
                </t>
            </t>
        </field>
    </record>
</odoo>
```

### Page record field reference

| Field | Description |
|-------|-------------|
| `name` | Page name (human-readable) |
| `is_published` | Visible to visitors |
| `is_homepage` | Set as the homepage |
| `key` | Unique view key |
| `url` | Relative URL path |
| `website_id` | Which website owns this page (multisite) |
| `type` | View type (`qweb`) |
| `arch` | View architecture (page markup) |
| `header_overlay` | Header floats over content (transparent header) |
| `header_visible` | Show/hide header on this page |
| `footer_visible` | Show/hide footer on this page |

In multisite setups, always specify `website_id` so the page is findable on the right site.

### `noupdate` — protecting records from upgrade overwrites

```xml
<!-- Protect all records in the file -->
<odoo noupdate="1">
    <record .../>
</odoo>

<!-- Protect only specific records -->
<odoo>
    <record id="menu_company" model="website.menu"><!-- always updated --></record>
    <data noupdate="1">
        <record id="menu_faq" model="website.menu"><!-- protected --></record>
        <record id="menu_legal" model="website.menu"><!-- protected --></record>
    </data>
</odoo>
```

If a `noupdate` record is manually deleted from the database, it will NOT be recreated on the next upgrade.

### New page templates

Declare snippet lists in `__manifest__.py` under `new_page_templates`, then create the templates in `views/new_page_template_templates.xml`. Templates appear in the New Page dialog. Use `primary="True"` on snippet template instances. Add to existing groups (`basic`, `about`, `landing`, `gallery`, `services`, `pricing`, `team`) or create a custom group via XPath on `website.new_page_template_groups`.

---

## Header

The header contains two templates: desktop and mobile (`template_header_mobile`).

### Using a standard header template

Always disable the currently active header first:
```xml
<!-- In data/presets.xml -->
<record id="website.template_header_default" model="ir.ui.view">
    <field name="active" eval="False"/>
</record>
<record id="website.template_header_stretch" model="ir.ui.view">
    <field name="active" eval="True"/>
</record>
```

Set in `primary_variables.scss`:
```scss
$o-website-values-palettes: (
    (
        'header-template': 'stretch',
    ),
);
```

### Custom header

**Important:** Disable the active header template first.

**Step 1 — Register the option** in `website.website_builder_assets`:
```xml
<!-- static/src/website_builder/header_template_option.xml -->
<t t-name="website_airproof.HeaderTemplateOption"
   t-inherit="website.HeaderTemplateOption"
   t-inherit-mode="extension">
    <xpath expr="//BuilderRow[@label.translate='Template']//BuilderSelect" position="inside">
        <BuilderSelectItem
            id="'header_airproof_opt'"
            title.translate="Airproof"
            actionParam="[{
                action: 'websiteConfig',
                actionParam: {
                    views: ['website_airproof.header'],
                    vars: { 'header-template': 'airproof' },
                    checkVars: false,
                },
            }]">
            <Img src="'/website_airproof/static/src/img/wbuilder/template-header-opt.svg'"
                 attrs="{ style: 'width: calc(100% - 8px)' }" />
        </BuilderSelectItem>
    </xpath>
</t>
```

**Step 2 — Activate in `primary_variables.scss`:**
```scss
$o-website-values-palettes: (
    (
        'header-template': 'airproof',
    ),
);
```

**Step 3 — Create the template** in `views/website_templates.xml`:
```xml
<template id="header" inherit_id="website.layout" name="Airproof – Header" active="True">
    <xpath expr="//header/nav" position="replace">
        <!-- Static content, components, editable areas -->
    </xpath>
</template>

<!-- Also adapt mobile header for consistency -->
<template id="template_header_mobile" inherit_id="website.template_header_mobile" name="Airproof – Template Header Mobile">
    <!-- XPaths -->
</template>
```

### Header components (`t-call` references)

```xml
<!-- Logo -->
<t t-call="website.placeholder_header_brand" _link_class.f="..." />

<!-- Navigation menu -->
<t t-foreach="website.menu_id.child_id" t-as="submenu">
    <t t-call="website.submenu"
       _item_class.f="nav-item"
       _link_class.f="nav-link" />
</t>

<!-- Sign in link -->
<t t-call="portal.placeholder_user_sign_in"
   _item_class.f="nav-item"
   _link_class.f="nav-link" />

<!-- User dropdown -->
<t t-call="portal.user_dropdown"
   _user_name="true"
   _icon="false"
   _avatar="false"
   _item_class.f="nav-item dropdown"
   _link_class.f="nav-link"
   _dropdown_menu_class.f="..." />

<!-- Language selector -->
<t t-call="website.placeholder_header_language_selector" _div_classes.f="..." />

<!-- Call to action button -->
<t t-call="website.placeholder_header_call_to_action" _div_classes.f="..." />

<!-- Mobile navbar toggler -->
<t t-call="website.navbar_toggler" _toggler_class.f="..." />
```

**Important:** Create a record for the website logo in `data/website.xml` before using `placeholder_header_brand`.

### Header overlay (transparent over content)

Set per page in the page record:
```xml
<field name="header_overlay" eval="True"/>
```

Or hide header on all pages via a preset:
```xml
<record id="website.option_layout_hide_header" model="ir.ui.view">
    <field name="active" eval="True"/>
</record>
```

---

## Footer

### Using a standard footer template

Disable the active footer first:
```xml
<record id="website.footer_custom" model="ir.ui.view">
    <field name="active" eval="False"/>
</record>
<record id="website.template_footer_links" model="ir.ui.view">
    <field name="active" eval="True"/>
</record>
```

Set in `primary_variables.scss`:
```scss
$o-website-values-palettes: (
    (
        'footer-template': 'Links',
    ),
);
```

### Custom footer

Register via a JS plugin in `website.website_builder_assets`:
```js
import { Plugin } from "@html_editor/plugin";
import { _t } from "@web/core/l10n/translation";
import { registry } from "@web/core/registry";
import { FooterTemplateChoice } from "@website/builder/plugins/options/footer_template_option";

export class AirproofFooterOptionPlugin extends Plugin {
    static id = "airproofFooterOption";
    resources = {
        footer_templates_providers: () => [{
            key: "airproof",
            Component: FooterTemplateChoice,
            props: {
                title: _t("Airproof"),
                view: "website_airproof.footer",
                varName: "airproof",
                imgSrc: "/website_airproof/static/src/img/wbuilder/template-footer-opt.svg",
            },
        }],
    };
}
registry.category("website-plugins").add(AirproofFooterOptionPlugin.id, AirproofFooterOptionPlugin);
```

Footer plugin props:

| Prop | Description |
|------|-------------|
| `title` | Display title in the template selector |
| `view` | Template that is enabled |
| `varName` | Value used in `primary_variables.scss` under `footer-template` |
| `imgSrc` | Thumbnail shown in the template selector |

Activate in `primary_variables.scss`:
```scss
$o-website-values-palettes: (
    (
        'footer-template': 'airproof',
    ),
);
```

Create the template in `views/website_templates.xml`:
```xml
<template id="footer" inherit_id="website.layout" name="Airproof – Footer" active="True">
    <xpath expr="//div[@id='footer']" position="replace">
        <div id="footer" class="oe_structure oe_structure_solo"
             t-ignore="true" t-if="not no_footer">
            <!-- Content -->
        </div>
    </xpath>
</template>
```

### Copyright bar

```xml
<template id="copyright" inherit_id="website.layout">
    <xpath expr="//*[hasclass('o_footer_copyright')]" position="replace">
        <div class="o_footer_copyright" data-name="Copyright">
            <!-- Content -->
        </div>
    </xpath>
</template>
```

---

## Colors & Palette

The Website Builder uses 5 named palette colors:

| Variable | Role |
|----------|------|
| `o-color-1` | Primary |
| `o-color-2` | Secondary |
| `o-color-3` | Extra (Light) |
| `o-color-4` | Whitish |
| `o-color-5` | Blackish |

### Declare a custom palette

In `primary_variables.scss`:
```scss
$o-color-palettes: map-merge($o-color-palettes,
    (
        'mytheme': (
            'o-color-1': #bedb39,
            'o-color-2': #2c3e50,
            'o-color-3': #f2f2f2,
            'o-color-4': #ffffff,
            'o-color-5': #000000,
        ),
    )
);

// Register in the Website Builder palette list
$o-selected-color-palettes-names: append($o-selected-color-palettes-names, 'mytheme');

// Activate it
$o-website-values-palettes: (
    (
        'color-palettes-name': 'mytheme',
    ),
);
```

### Color combinations (`o-cc`)

The Website Builder auto-generates 5 color combinations from the palette. Each defines background, text, headings, links, primary buttons, and secondary buttons. The default text color is `o-color-5`; it auto-switches to `o-color-4` when the background is too dark.

Override individual combinations using the `o-cc` prefix in `$o-color-palettes`.

Demo page: `http://localhost:8069/website/demo/color-combinations`

### Page background via SCSS

Color:
```scss
$o-color-palettes: map-merge($o-color-palettes,
    (
        'airproof': (
            'o-cc1-bg': 'o-color-5',
            'o-cc5-bg': 'o-color-1',
        ),
    )
);
```

Image or pattern:
```scss
$o-website-values-palettes: (
    (
        'body-image': '/website_airproof/static/src/img/background-lines.svg',
        'body-image-type': 'image',  // or 'pattern'
    ),
);
```

---

## Fonts

### Declare fonts in `primary_variables.scss`

```scss
$so-theme-font-configs: (
    '<font-name>': (
        'family':     <CSS font family list>,
        'url':        '<Google Fonts URL>',       // optional
        'properties': (                           // optional
            '<website-value-key>': <value>,
        ),
    ),
);
```

### Apply fonts

```scss
$o-website-values-palettes: (
    (
        'font':          '<font-name>',
        'headings-font': '<font-name>',
        'navbar-font':   '<font-name>',
        'buttons-font':  '<font-name>',
    ),
);
```

### Google Fonts example

```scss
$so-theme-font-configs: (
    'Poppins': (
        'family':     ('Poppins', sans-serif),
        'url':        'Poppins:400,500',
        'properties': (
            'base': ('font-size-base': 1rem),
        ),
    ),
);
```

### Self-hosted fonts

1. Place `.woff` and `.woff2` files in `static/fonts/`
2. Create `static/src/scss/fonts.scss` in `web.assets_frontend`:
```scss
@font-face {
    font-family: "My Custom Font";
    font-weight: 400;
    font-style: normal;
    src: url('/fonts/my-custom-font.woff2') format('woff2'),
         url('/fonts/my-custom-font.woff') format('woff');
}
```
3. Register in `$so-theme-font-configs` without a `url` key

---

## Gradients

In `primary_variables.scss`:
```scss
$o-website-values-palettes: (
    (
        'menu-gradient':         linear-gradient(135deg, rgb(203, 94, 238) 0%, rgb(75, 225, 236) 100%),
        'header-boxed-gradient': <gradient>,
        'footer-gradient':       <gradient>,
        'copyright-gradient':    <gradient>,
    ),
);
```

---

## Bootstrap Variables

Use `bootstrap_overridden.scss` (declared in `web._assets_frontend_helpers` with `prepend`). This file must contain only variable definitions and mixin overrides — no output CSS.

```scss
// Typography
$h1-font-size:              4rem !default;

// Navbar
$navbar-nav-link-padding-x: 1rem !default;

// Buttons + Forms
$input-placeholder-color:   o-color('o-color-1') !default;

// Cards
$card-border-width:         0 !default;
```

**Warning:** Do not override Bootstrap variables that depend on Odoo variables — it breaks the user's ability to customize them in the Website Builder. When an option exists in both `primary_variables.scss` and Bootstrap, always prefer the primary variable. Use `bootstrap_overridden.scss` only when no primary variable equivalent exists.

Demo page: `http://localhost:8069/website/demo/bootstrap`

---

## Website Settings

In `data/website.xml`. Use `noupdate="1"` to protect from upgrade overwrites:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo noupdate="1">
    <record id="website.default_website" model="website">
        <field name="name">My Site</field>
        <field name="logo" type="base64" file="module_name/static/src/img/content/logo.png"/>
        <field name="favicon" type="base64" file="module_name/static/src/img/content/favicon.png"/>
        <field name="shop_ppg">3</field>
        <field name="shop_ppr">10</field>
        <field name="shop_gap">16px</field>
        <field name="cookies_bar" eval="True"/>
        <field name="contact_us_button_url">/contact-us</field>
        <field name="social_facebook">https://www.facebook.com/...</field>
        <field name="social_instagram">https://www.instagram.com/...</field>
        <field name="social_linkedin">https://www.linkedin.com/company/...</field>
        <field name="social_youtube">https://www.youtube.com/...</field>
    </record>
</odoo>
```

### Website record field reference

| Field | Description |
|-------|-------------|
| `name` | Site name (shown in browser tab) |
| `logo` | Path to logo (declare as base64 record) |
| `favicon` | Path to favicon |
| `shop_ppg` | Products shown per page in eCommerce |
| `shop_ppr` | Products per row in eCommerce |
| `shop_gap` | Gutter value for product grid |
| `shop_opt_products_design_classes` | CSS classes for product design options |
| `cookies_bar` | Enable the cookie consent bar |
| `contact_us_button_url` | URL for the Contact Us header button |
| `social_facebook` | Facebook profile URL |
| `social_instagram` | Instagram profile URL |
| `social_linkedin` | LinkedIn company URL |
| `social_youtube` | YouTube channel URL |

`website.default_website` = default reference for single-website setups. With multiple websites it refers to the first (default) one.

---

## Presets

Enable or disable built-in Odoo views in `data/presets.xml`. Prefer `<template>` syntax over `<record model="ir.ui.view">`:

```xml
<template id="..." inherit_id="..." active="False"/>
```

### Common preset examples

```xml
<!-- Disable default header -->
<record id="website.template_header_default" model="ir.ui.view">
    <field name="active" eval="False"/>
</record>

<!-- Enable stretch header -->
<record id="website.template_header_stretch" model="ir.ui.view">
    <field name="active" eval="True"/>
</record>

<!-- Center header menu items -->
<record id="website.template_header_default_align_center" model="ir.ui.view">
    <field name="active" eval="True"/>
</record>

<!-- Hide header on all pages -->
<record id="website.option_layout_hide_header" model="ir.ui.view">
    <field name="active" eval="True"/>
</record>

<!-- Disable eCommerce product categories sidebar -->
<record id="website_sale.products_categories" model="ir.ui.view">
    <field name="active" eval="False"/>
</record>

<!-- Disable language selector in footer -->
<record id="portal.footer_language_selector" model="ir.ui.view">
    <field name="active" eval="False"/>
</record>
```

For views that need both a `BuilderCheckbox` activation and a preset, use both `<template active="..."/>` and the preset record. Prefer `<template>` syntax in those cases.

---

## Media — Images

### Declare as `ir.attachment` records

```xml
<!-- data/images.xml -->
<record id="img_about_01" model="ir.attachment">
    <field name="name">About Image 01</field>
    <field name="datas" type="base64" file="module_name/static/src/img/content/img_about_01.jpg"/>
    <field name="res_model">ir.ui.view</field>
    <field name="public" eval="True"/>
</record>
```

| Field | Description |
|-------|-------------|
| `id` | Used to reference the image in code |
| `name` | Descriptive name |
| `datas` | Path to the image file (base64 encoded) |
| `res_model` | Link to `ir.ui.view` for Website Builder access |
| `public` | Make available to all visitors |

**Warning:** Some Website Builder options (e.g. image shapes, crop) only work for images declared as records.

### Use in templates

```xml
<!-- Regular image -->
<img src="/web/image/module_name.img_about_01" alt="" />

<!-- Background image -->
<section style="background-image: url('/web/image/module_name.img_about_01');">
    <!-- Content -->
</section>
```

### Company logo

Declare in `data/website.xml`, then call in templates via:
```xml
<t t-call="website.placeholder_header_brand" _link_class.f="navbar-brand" />
```

### Image optimization guidelines

- **Weight:** < 200KB
- **Size:** ≤ 1500px wide if not displayed full-screen
- **Extension:** `.svg` or `.jpg`, `.png`, `.gif` — use `.svg` or `.jpg` whenever possible
- **Name:** no spaces, accents, or special characters; separate words with dashes; use relevant words
- Images > 1920px are heavily compressed by the Website Builder; images < 1920px remain intact

---

## Media — Videos

### Background video

```html
<section class="o_background_video" data-bg-video-src="https://...">
    <!-- Content -->
</section>
```

| Attribute | Description |
|-----------|-------------|
| `data-bg-video-src` | Video URL |

### Content video (embedded iframe)

```html
<div class="media_iframe_video" data-oe-expression="https://...">
    <div class="css_editable_mode_display" />
    <div class="media_iframe_video_size" contenteditable="false" />
    <iframe
        src="https://..."
        frameborder="0"
        contenteditable="false"
        allowfullscreen="allowfullscreen" />
</div>
```

| Attribute | Description |
|-----------|-------------|
| `data-oe-expression` | Video URL |
| `src` | Video URL (on the iframe) |

---

## Media — Icons

Font Awesome v4 is included by default. Use `<span>` (semantically preferred over `<i>`):

```html
<!-- Basic usage -->
<span class="fa fa-picture-o" />

<!-- With Website Builder style options enabled -->
<span class="fa fa-2x fa-picture-o rounded-circle" />

<!-- Size variants: fa-2x fa-3x fa-4x fa-5x -->
<span class="fa fa-2x fa-picture-o" />
```

---

## Background Shapes

SVG files used as decorative section backgrounds. Always use Odoo's default palette colors in SVGs — they are auto-adapted when the palette changes.

### Use a standard shape

```xml
<section data-oe-shape-data="{'shape':'html_builder/Zigs/06'}">
    <div class="o_we_shape o_html_builder_Zigs_06" />
    <div class="container">
        <!-- Content -->
    </div>
</section>
```

`data-oe-shape-data` is JSON containing shape location, repeat, flip options, etc.

Flip horizontally and/or vertically:
```xml
<section data-oe-shape-data="{'shape':'html_builder/Zigs/06','flip':{'x':true,'y':true}}">
```

### Remap default colors of a built-in shape

In `primary_variables.scss` (replaces original mapping):
```scss
// With palette index references
$o-bg-shapes: change-shape-colors-mapping('html_builder', 'Zigs/06', (4: 3, 5: 1));

// With custom color values
$o-bg-shapes: change-shape-colors-mapping('html_builder', 'Zigs/06', (4: 3, 5: rgb(187, 27, 152)));
```

### Add an extra color variant (keeps original)

In `bootstrap_overridden.scss`:
```scss
$o-bg-shapes: add-extra-shape-colors-mapping('html_builder', 'Zigs/06', 'second', (4: 3, 5: 1));
```

This creates a second selectable color variant in the Website Builder. Use it in templates:
```xml
<div class="o_we_shape o_html_builder_Zigs_06 o_second_extra_shape_mapping" />
```

### Custom background shapes (5 steps)

**Step 1 — Create SVG** in `static/shapes/<group>/<name>.svg`. Use colors from Odoo's default palette only.

**Step 2 — Declare as `ir.attachment`** in `data/shapes.xml`:
```xml
<record id="shape_hexagon_01" model="ir.attachment">
    <field name="name">01.svg</field>
    <field name="datas" type="base64" file="module_name/static/shapes/hexagons/01.svg"/>
    <field name="url">/html_editor/shape/illustration/hexagons/01.svg</field>
    <field name="public" eval="True"/>
</record>
```

The `url` uses the `illustration/` prefix — the Website Builder auto-duplicates the file there.

**Step 3 — Register SCSS** in `primary_variables.scss`:
```scss
$o-bg-shapes: map-merge($o-bg-shapes,
    (
        'illustration': map-merge(
            map-get($o-bg-shapes, 'illustration') or (),
            (
                'hexagons/01': (
                    'position': center center,
                    'size':     auto 100%,
                    'colors':   (1),
                    'repeat-x': true,
                ),
            ),
        ),
    )
);
```

SCSS shape key reference:

| Key | Description |
|-----|-------------|
| `position` | CSS background-position |
| `size` | CSS background-size |
| `colors` | Array of palette color indices (overrides SVG colors) |
| `repeat-x` | Repeat horizontally (only define if `true`) |
| `repeat-y` | Repeat vertically (only define if `true`) |

**Step 4 — JS plugin** in `website.website_builder_assets`:
```js
import { Plugin } from "@html_editor/plugin";
import { _t } from "@web/core/l10n/translation";
import { registry } from "@web/core/registry";

export class AirproofBackgroundShapesOptionPlugin extends Plugin {
    static id = "airproofBackgroundShapesOption";
    resources = {
        background_shape_groups_providers: () => ({
            airproof: {
                label: _t("Airproof"),
                subgroups: {
                    airproof: {
                        label: _t("Airproof"),
                        shapes: {
                            "website_airproof/hexagons/01": {
                                selectLabel: _t("Hexagon 01"),
                            },
                        },
                    },
                },
            }),
        }),
    };
}

registry.category("website-plugins").add(
    AirproofBackgroundShapesOptionPlugin.id,
    AirproofBackgroundShapesOptionPlugin
);
```

**Step 5 — Use in pages:**
```xml
<section data-oe-shape-data="{'shape': 'illustration/airproof/hexagons/01', 'colors': {'c4': '#8595A2', 'c5': 'rgba(0,255,0,1)'}}">
    <div class="o_we_shape o_illustration_airproof_hexagons_01" />
    <div class="container"><!-- Content --></div>
</section>
```

Colors in `data-oe-shape-data` are optional and override the SCSS defaults.

---

## Image Shapes

SVG clipping masks applied on top of images. Some have customizable colors and animations.

**Important:** If not manually saved via the Website Builder, a shaped image is **generated in real-time for each visitor** — this can impact page load performance.

### Use a standard image shape

The image must first be declared as an `ir.attachment`. The Website Builder re-processes it when a shape is applied.

```html
<img
    src="/html_editor/image_shape/html_builder/solid/solid_blob_2"
    class="img img-fluid mx-auto"
    alt="..."
    data-shape="html_builder/solid/solid_blob_2"
    data-shape-colors="#0B8EE6;;;;"
    data-mimetype="image/svg+xml"
    data-mimetype-before-conversion="image/jpeg"
    data-original-src="/website/static/src/img/snippets_demo/s_picture.jpg"
    data-file-name="s_text_image.svg"
    data-original-id="500"
    data-attachment-id="500"
/>
```

### Image shape data attribute reference

| Attribute | Description |
|-----------|-------------|
| `data-shape` | Path to the shape SVG |
| `data-shape-colors` | Up to 5 colors, semicolon-separated. Empty slot = SVG default. Each maps to a palette position. |
| `data-shape-flip` | Flip along `x`, `y`, or `xy` |
| `data-shape-rotate` | Rotate 90°: values `90`, `180`, `270` |
| `data-aspect-ratio` | `1/1` = shape fills image; `0/0` = stretch to fit image aspect ratio |
| `data-shape-animation-speed` | `-2` to `2`, steps of `0.1` |
| `data-mimetype` | MIME type of the shaped (output) image |
| `data-mimetype-before-conversion` | MIME type of the original image |
| `data-original-src` | Path to the original image file |
| `data-file-name` | Name of the created SVG file (always `.svg`) |
| `data-original-id` | `ir.attachment` ID of the original image |
| `data-attachment-id` | `ir.attachment` ID of the shaped image |

Shaped images are served via:
```
/html_editor/image_shape/<module>/<path:filename>
```

### Custom image shapes (3 steps)

**Step 1 — Create SVG** in `static/image_shapes/<group>/<name>.svg`:

```xml
<svg xmlns="http://www.w3.org/2000/svg"
     xmlns:xlink="http://www.w3.org/1999/xlink"
     width="800" height="800">

    <defs>
        <!-- Mask: clipPath references the filterPath vector -->
        <clipPath id="clip-path" clipPathUnits="objectBoundingBox">
            <use xlink:href="#filterPath" fill="none" />
        </clipPath>
        <!-- The actual clipping path vector (normalized 0–1 coordinates) -->
        <path id="filterPath" d="M0.325,0.75H0.125c-0.069,..." />
    </defs>

    <!-- Optional decorative elements (outside defs) — must use Odoo default palette colors -->
    <svg viewBox="0 0 1 1" preserveAspectRatio="none">
        <rect x="0.494" y="0.325" width="0.0125" height="0.35"
              rx="0.00625" ry="0.00625" fill="#7C6570" />
    </svg>

    <!-- Preview of the mask (shown in the editor) -->
    <svg viewBox="0 0 1 1" id="preview" preserveAspectRatio="none">
        <use xlink:href="#filterPath" fill="darkgrey" />
    </svg>

    <!-- Future image that receives the mask -->
    <image xlink:href="" clip-path="url(#clip-path)">
        <!-- Required compatibility hack for non-animated shapes (Safari/Firefox) -->
        <animateMotion dur="1ms" repeatCount="indefinite" />
    </image>
</svg>
```

Key SVG rules:
- `width` and `height` in **pixels** for sharpness, but `viewBox` always `0 0 1 1` (normalized)
- Mask is defined in `<defs>` — not rendered until referenced
- Any decorative `fill` color **must come from Odoo's default palette**
- Always include the `<animateMotion>` compatibility hack even for non-animated shapes

**Step 2 — JS plugin** in `website.website_builder_assets`:
```js
import { Plugin } from "@html_editor/plugin";
import { _t } from "@web/core/l10n/translation";
import { registry } from "@web/core/registry";

export class AirproofImageShapesOptionPlugin extends Plugin {
    static id = "airproofImageShapesOption";
    resources = {
        image_shape_groups_providers: () => ({
            airproof: {
                label: _t("Airproof"),
                subgroups: {
                    airproof_duo: {
                        label: _t("Duo"),
                        shapes: {
                            "website_airproof/duo/01": {
                                selectLabel: _t("Airproof 01"),
                                transform: true,
                                togglableRatio: true,
                            },
                        },
                    },
                },
            }),
        }),
    };
}

registry.category("website-plugins").add(
    AirproofImageShapesOptionPlugin.id,
    AirproofImageShapesOptionPlugin
);
```

Image shape plugin shape properties:

| Property | Description |
|----------|-------------|
| `selectLabel` | Name shown in the shape list |
| `transform` | Show/hide transform options (flip, rotate) |
| `togglableRatio` | Show/hide the stretch (aspect ratio) option |
| `animated` | Indicates the shape has CSS animations |
| `imgSize` | Image ratio used for device display (e.g. `0.46:1`) |

**Step 3 — Use in pages:**
```html
<img
    src="/html_editor/image_shape/website_airproof/duo/01"
    class="img img-fluid mx-auto"
    alt="..."
    data-shape="website_airproof/duo/01"
    data-shape-colors="#0B8EE6;;;;"
    data-mimetype="image/svg+xml"
    data-mimetype-before-conversion="image/webp"
    data-original-src="website_airproof/static/src/img/content/drone-robin.webp"
    data-file-name="drone-robin.svg"
    data-original-id="560"
    data-attachment-id="560"
/>
```

---

## Building Blocks — Layout & CSS

### Snippet types

- **Structure blocks** — whole rows; wrapper is `<section>`
- **Inner Content blocks** — used inside other blocks; wrapper is any HTML tag

### File structure

```
views/snippets/
    snippets.xml              # Registration (groups + snippet list)
    s_snippet_name.xml        # Snippet template

static/src/snippets/
    s_snippet_name/
        snippet_name.js       # Frontend interactions
        snippet_name.edit.js  # Edit-mode behavior
        000.scss              # Styles
        000.xml               # Versioned QWeb template

static/src/website_builder/
    snippet_name_option_plugin.js   # Options plugin
    snippet_name_option.xml         # Options XML template
```

### Wrapper attributes

```xml
<!-- Structure block -->
<section class="s_snippet_name" data-name="My Snippet" data-snippet="s_snippet_name">
    <!-- Content -->
</section>

<!-- Inner content block -->
<div class="s_snippet_name" data-name="My Snippet" data-snippet="s_snippet_name">
    <!-- Content -->
</div>
```

| Attribute | Description |
|-----------|-------------|
| `class` | Unique CSS class name for this snippet |
| `data-name` | Shown in the right panel (falls back to "Block") |
| `data-snippet` | Used internally to identify the snippet |

**Warning:** `data-name` and `data-snippet` must be specified when a snippet is declared on a theme page. Without them the Website Builder won't recognise it and database upgrades may break.

**Warning:** Never put a `<section>` inside another `<section>` — use inner content snippets instead.

**Tip:** Building static pages — either use the Website Builder (drag and drop, then copy/paste the source code and clean it up), or code directly (but beware that some classes, names, and data attributes must be exact for the Website Builder to work correctly).

### Column & sizing classes

```html
<!-- Resizable columns — large Bootstrap columns descending from .row -->
.row > .col-lg-*

<!-- Section padding — multiples of 8, max 256 -->
class="pt80 pb80"

<!-- Enable columns selector on container -->
<div class="container s_allow_columns">

<!-- Disable column count option -->
<div class="row s_nb_column_fixed">

<!-- Disable resize on all columns in a row -->
<div class="row s_col_no_resize">

<!-- Disable resize on one specific column -->
<div class="col-lg-* s_col_no_resize">
```

### Color classes

```html
<!-- Add background from color palette (replace * with 1–5) -->
class="o_cc o_cc*"

<!-- Disable background color option for all columns -->
<div class="row s_col_no_bgcolor">

<!-- Disable background color option for one column -->
<div class="col-lg-* s_col_no_bgcolor">
```

### Color overlays

```html
<section>
    <!-- Black 50% overlay -->
    <div class="o_we_bg_filter bg-black-50" />

    <!-- White 85% overlay -->
    <div class="o_we_bg_filter bg-white-85" />

    <!-- Custom color overlay -->
    <div class="o_we_bg_filter" style="background-color: rgba(39, 110, 114, 0.54) !important;" />

    <!-- Custom gradient overlay -->
    <div class="o_we_bg_filter" style="background-image: linear-gradient(135deg, rgba(255,204,51,0.5) 0%, rgba(226,51,255,0.5) 100%) !important;" />

    <div class="container"><!-- Content --></div>
</section>
```

### Non-editable / non-removable elements

```html
<div class="o_not_editable">   <!-- User cannot edit -->
<div class="oe_unremovable">   <!-- User cannot remove -->
```

### Background images

Simple:
```html
<div class="oe_img_bg o_bg_img_center" style="background-image: url('...')">
```

Parallax:
```html
<section class="parallax s_parallax_is_fixed s_parallax_no_overflow_hidden"
         data-scroll-background-ratio="1">
    <span class="s_parallax_bg oe_img_bg o_bg_img_center"
          style="background-image: url('...'); background-position: 50% 75%;" />
    <div class="container"><!-- Content --></div>
</section>
```

### Text highlights

```html
<h2>
    Title with
    <span class="o_text_highlight o_text_highlight_fill"
          style="--text-highlight-width: 10px !important; --text-highlight-color: #FFFF00;">
        <span class="o_text_highlight_item">
            highlighted text
            <svg fill="none" class="o_text_highlight_svg o_content_no_merge position-absolute overflow-visible top-0 start-0 w-100">
                <!-- SVG path -->
            </svg>
        </span>
    </span>
</h2>
```

| CSS custom property | Description |
|---------------------|-------------|
| `--text-highlight-width` | Thickness of the SVG border |
| `--text-highlight-color` | Color of the SVG highlight |

### Grid layout

```html
<div class="row o_grid_mode" data-row-count="13" style="gap: 20px 10px;">
    <div class="o_grid_item g-height-5 g-col-lg-7"
         style="grid-area: 2 / 1 / 7 / 8; z-index: 3;">
        <!-- Content -->
    </div>
    <div class="o_grid_item o_grid_item_image g-height-8 g-col-lg-7"
         style="grid-area: 1 / 6 / 9 / 13; z-index: 2;">
        <img src="..." alt="..." />
    </div>
</div>
```

| CSS custom property | Description |
|---------------------|-------------|
| `--grid-item-padding-y` | Vertical padding (Y axis) |
| `--grid-item-padding-x` | Horizontal padding (X axis) |

Grid always has 12 columns. `data-row-count` defines the number of rows. Gap is set in the `style` attribute.

### Drop zones

```html
<div id="oe_structure_layout_01" class="oe_structure" />
```

| Class | Description |
|-------|-------------|
| `oe_structure` | Drag-and-drop area |
| `oe_structure_solo` | Only one snippet can be dropped here |
| `oe_structure_not_nearest` | Block dropped outside moves to the nearest Drop Zone |

Populate an existing drop zone:
```xml
<template id="oe_structure_layout_01" inherit_id="..." name="...">
    <xpath expr="//*[@id='oe_structure_layout_01']" position="replace">
        <div id="oe_structure_layout_01" class="oe_structure oe_structure_solo">
            <!-- Pre-filled content -->
        </div>
    </xpath>
</template>
```

### Responsive visibility

```html
class="o_snippet_mobile_invisible"   <!-- Hidden on mobile (Website Builder aware) -->
class="o_snippet_desktop_invisible"  <!-- Hidden on desktop (Website Builder aware) -->
class="d-none"                       <!-- Hidden everywhere -->
class="d-lg-block"                   <!-- Shown from large breakpoint (desktop) -->
```

**Important:** Always keep `o_snippet_mobile_invisible` or `o_snippet_desktop_invisible` on hidden elements so the Website Builder's visibility conditions panel stays functional. If an element is hidden on desktop, the Website Builder still displays it in the "Invisible Elements" list so the user can force-show and edit it.

### Compatibility system

When a snippet has `data-vcss`, `data-vjs`, and/or `data-vxml` attributes it is an updated version:
```html
<section class="s_snippet_name" data-vcss="001" data-vxml="001" data-vjs="001">
```
These tell the system which versioned file to load (e.g. `001.js`, `002.scss`).

---

## Building Blocks — Custom Snippet Workflow

### Step 1 — Template (`views/snippets/s_airproof_snippet.xml`)

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <template id="s_airproof_snippet" name="Airproof Snippet">
        <section class="s_airproof_snippet" data-name="Airproof Snippet" data-snippet="s_airproof_snippet">
            <!-- Content — use Bootstrap classes, prefix custom classes with s_ or x_ -->
        </section>
    </template>
</odoo>
```

Tips:
- Use Bootstrap native classes as much as possible
- Prefix all custom classes (e.g. `x_nav`, `x_nav_item`)
- Use underscore notation for class names
- Avoid `id` attributes inside `<section>` — multiple snippet instances may appear on the same page

### Step 2 — Register group + snippet (`views/snippets/snippets.xml`)

```xml
<template id="snippets" inherit_id="website.snippets" name="Airproof – Snippets">

    <!-- Create the group tab -->
    <xpath expr="//snippets[@id='snippet_groups']/*[1]" position="before">
        <t snippet-group="airproof"
           t-snippet="website.s_snippet_group"
           string="Airproof"
           t-thumbnail="/website_airproof/static/src/img/wbuilder/s_airproof_group_thumbnail.svg" />
    </xpath>

    <!-- Add the snippet to the group -->
    <xpath expr="//snippets[@id='snippet_structure']/*[1]" position="before">
        <t t-snippet="website_airproof.s_airproof_snippet"
           string="Custom Snippet"
           group="airproof">
            <keywords>airproof custom snippet</keywords>
        </t>
    </xpath>

</template>
```

Group registration attributes:

| Attribute | Description |
|-----------|-------------|
| `snippet-group` | Unique ID of the group |
| `t-snippet` | Inherited template ID (always `website.s_snippet_group`) |
| `string` | Group label shown to users |
| `t-thumbnail` | Path to group thumbnail SVG |

Snippet registration attributes:

| Attribute | Description |
|-----------|-------------|
| `t-snippet` | The snippet template to use |
| `string` | Snippet label in the panel |
| `group` | Group to add it to (**singular** — `groups` is for access rights) |
| `<keywords>` | Search keywords in the Snippets panel |

### Inner content snippets

To make a snippet appear in the **Inner Content** list instead of Structure:
- Register under `snippet_content` instead of `snippet_structure`
- Include `t-thumbnail` (required for inner content)
- Remove the `group` attribute

To make it **droppable** inside another building block, add its selector to `so_content_addition_selector` via a JS plugin:
```js
resources = {
    so_content_addition_selector: [".s_airproof_snippet"],
};
```

---

## Building Blocks — Options API

Options allow users to edit a snippet's appearance via the Website Builder panel. They are OWL components defined in JS and rendered with XML templates. Add all JS and XML to `website.website_builder_assets`.

### JS plugin + component

```js
// static/src/website_builder/airproof_snippet_option_plugin.js
import { BaseOptionComponent } from "@html_editor/core/utils";
import { Plugin } from "@html_editor/plugin";
import { registry } from "@web/core/registry";

export class AirproofSnippetOption extends BaseOptionComponent {
    static template = "website_airproof.AirproofSnippetOption";
    static selector = ".s_airproof_snippet";   // CSS selector — when to show this option
    static applyTo = ":scope > .row";          // Apply actions to this child element
    static exclude = ".navbar-nav";            // Exclude these from the selector match
}

export class AirproofSnippetOptionPlugin extends Plugin {
    static id = "airproofSnippetOption";
    resources = {
        builder_options: [AirproofSnippetOption],
    };
}

registry.category("website-plugins").add(
    AirproofSnippetOptionPlugin.id,
    AirproofSnippetOptionPlugin
);
```

### XML options template

```xml
<!-- static/src/website_builder/airproof_snippet_option.xml -->
<templates xml:space="preserve">
    <t t-name="website_airproof.AirproofSnippetOption">
        <BuilderRow label.translate="Layout">
            <BuilderSelect>
                <BuilderSelectItem classAction="''">Default</BuilderSelectItem>
                <BuilderSelectItem classAction="'s_airproof_snippet_portrait'">Portrait</BuilderSelectItem>
                <BuilderSelectItem classAction="'s_airproof_snippet_square'">Square</BuilderSelectItem>
            </BuilderSelect>
        </BuilderRow>
        <BuilderRow label.translate="Space">
            <BuilderButtonGroup>
                <BuilderButton classAction="'mt-0'">1</BuilderButton>
                <BuilderButton classAction="'mt-3'">2</BuilderButton>
                <BuilderButton classAction="'mt-5'">3</BuilderButton>
            </BuilderButtonGroup>
        </BuilderRow>
        <BuilderRow label.translate="Tooltip">
            <BuilderCheckbox classAction="'s_airproof_snippet_tooltip'" />
        </BuilderRow>
        <BuilderRow label.translate="Speed">
            <BuilderNumberInput
                styleAction="'animation-duration'"
                unit="'s'"
                saveUnit="'ms'"
                step="0.1" />
        </BuilderRow>
        <BuilderRow label.translate="Background">
            <BuilderColorPicker
                action="'selectFilterColor'"
                defaultOpacity="50"
                enabledTabs="['theme', 'gradient', 'custom']" />
        </BuilderRow>
    </t>
</templates>
```

### Builder UI component reference

| Component | Description |
|-----------|-------------|
| `<BuilderRow label.translate="...">` | Labeled row grouping fields. Use `level` to indent nested rows. |
| `<BuilderButton classAction="'...'">` | Toggle button that adds/removes a CSS class |
| `<BuilderButtonGroup>` | Group of buttons with mutual selection state |
| `<BuilderSelect>` | Dropdown selector |
| `<BuilderSelectItem classAction="'...'">` | Item inside a `BuilderSelect` |
| `<BuilderCheckbox classAction="'...'">` | Toggle switch |
| `<BuilderRange action="'...'" min="0" max="10" step="1" displayRangeValue="true">` | Slider |
| `<BuilderNumberInput styleAction="'...'" unit="'s'" saveUnit="'ms'" step="0.1">` | Numeric input field |
| `<BuilderColorPicker action="'selectFilterColor'" defaultOpacity="50" enabledTabs="[...]">` | Color/gradient picker |
| `<BuilderList action="'setPrefilledOptions'" itemShape="{...}" sortable="true">` | Editable list (add, remove, reorder) |

### `BuilderNumberInput` props

| Prop | Description |
|------|-------------|
| `unit` | Display unit shown to user (e.g. `'s'`) |
| `saveUnit` | Unit the value is converted to on save (e.g. `'ms'`) |
| `step` | Increment step |

### `BuilderList` props

| Prop | Description |
|------|-------------|
| `itemShape` | Field types per column. Supported: `text`, `number`, `boolean`, `exclusive_boolean` |
| `default` | Default values for new items (keys must match `itemShape`) |
| `records` | Optional list of available records (JSON). Makes add button a dropdown. |
| `defaultNewValue` | Extra values injected when creating a new item |
| `sortable` | Allow drag-and-drop reordering (default: `true`) |
| `hiddenProperties` | Hide specific columns by name |
| `columnWidth` | CSS class per column (e.g. `w-25`) |
| `forbidLastItemRemoval` | Prevent removing the last remaining item |

### `BuilderColorPicker` props

| Prop | Description |
|------|-------------|
| `enabledTabs` | Restrict available picker tabs: `['theme', 'gradient', 'custom']` |
| `defaultColor` | Default color when no selection |
| `defaultOpacity` | Default alpha value (0–100) |

### Core actions (shorthand props on builder components)

| Action prop | Description |
|-------------|-------------|
| `classAction` | Add/remove CSS class on the target element |
| `styleAction` | Set an inline style property |
| `attributeAction` | Set or remove a DOM attribute |
| `dataAttributeAction` | Set or remove a dataset key (without `data-` prefix) |
| `setClassRange` | Apply one class from a predefined list based on range value |
| `action + actionParam` | Call a custom `BuilderAction` registered in the plugin |

### `BaseOptionComponent` static properties

| Property | Description |
|----------|-------------|
| `static template` | OWL template name rendered in the options panel |
| `static selector` | CSS selector — option appears when a matching element is selected |
| `static exclude` | CSS selector of elements to exclude from the match |
| `static applyTo` | CSS selector targeting a child element (actions apply to it) |
| `static components` | Child builder OWL components used in the template |
| `setup()` | Initialize component state |

### Plugin API — static properties

| Property | Purpose |
|----------|---------|
| `static dependencies` | Declare required plugins (run before `setup()`) |
| `static shared` | Expose shared methods to other plugins |
| `setup()` | Initialize plugin state |

### Plugin API — resources

| Resource | Purpose |
|----------|---------|
| `builder_options` | Register `BaseOptionComponent` classes |
| `builder_actions` | Register `BuilderAction` classes |
| `dropzone_selector` | Define drop rules (`selector`, `dropIn`, `dropNear`, `exclude`, `excludeNearParent`) |
| `so_content_addition_selector` | Extend inner content selectors list |
| `on_snippet_dropped_handlers` | Called after snippet is dropped and inserted in DOM |
| `on_cloned_handlers` | Called after an element is cloned and inserted |
| `on_will_remove_handlers` | Called just before an element is removed |
| `on_removed_handlers` | Called after an element is removed |
| `clean_for_save_handlers` | Called on a cleaned DOM clone before saving |
| `before_save_handlers` | Called at start of save process |
| `after_save_handlers` | Called after save process completes |

### `BuilderAction` lifecycle methods

| Method | Description |
|--------|-------------|
| `prepare` | Prepare async data before component is used |
| `load` | Load data before `apply` is previewed |
| `apply` | Apply the action to the targeted element |
| `clean` | Reset the action when a new choice supersedes it |
| `getValue` | Return current value (for input widgets) |
| `isApplied` | Return whether action is active (for selectable widgets) |
| `getPriority` | Decide which selectable item wins when multiple are valid |

### Custom `BuilderAction` example

```js
import { BuilderAction } from "@html_editor/core/builder_action";
import { Plugin } from "@html_editor/plugin";
import { registry } from "@web/core/registry";

export class AirproofLayoutAction extends BuilderAction {
    static id = "airproofLayout";

    apply({ editingElement, params: { mainParam } }) {
        editingElement.classList.toggle("s_airproof_snippet_portrait", mainParam === "portrait");
        editingElement.classList.toggle("s_airproof_snippet_square", mainParam === "square");
    }

    isApplied({ editingElement, params: { mainParam } }) {
        return editingElement.classList.contains(`s_airproof_snippet_${mainParam}`);
    }
}

export class AirproofSnippetOptionPlugin extends Plugin {
    static id = "airproofSnippetOption";
    resources = {
        builder_actions: [AirproofLayoutAction],
    };
}

registry.category("website-plugins").add(
    AirproofSnippetOptionPlugin.id,
    AirproofSnippetOptionPlugin
);
```

Call from XML template:
```xml
<BuilderRow label.translate="Layout">
    <BuilderButtonGroup action="'airproofLayout'">
        <BuilderButton actionParam="'portrait'">Portrait</BuilderButton>
        <BuilderButton actionParam="'square'">Square</BuilderButton>
    </BuilderButtonGroup>
</BuilderRow>
```

### Dropzone selectors

```js
resources = {
    dropzone_selector: {
        selector: ".s_airproof_snippet",
        dropIn: ".s_custom_location",    // Where this snippet can be dropped INTO
        dropNear: ".card",               // What it can be dropped NEXT TO
    },
};
```

### Dynamic Content templates

Dynamic Content blocks use `data-template-key` to select a rendering template. The `<div class="dynamic_snippet_template"/>` placeholder is replaced with the selected template at render time.

`data-custom-template-data` (JSON string) passes custom variables to the template for show/hide logic or content rendering.

Template ID must start with `dynamic_filter_template_blog_post_`.

### Dynamic products snippet (eCommerce)

Set styling options by adding CSS classes to the `<section>`:

| Option | CSS Class |
|--------|-----------|
| Hover background | `o_wsale_products_opt_hover_background` |
| Hover background zoom | `o_wsale_products_opt_hover_background o_wsale_products_opt_hover_background_zoom` |
| Roundness 0–5 | `o_wsale_products_opt_rounded_0` … `_5` |
| Color combination 1–5 | `o_wsale_products_opt_cc o_wsale_products_opt_cc1` … `_cc5` |

Card design presets (mutually exclusive):

| Preset | CSS Class |
|--------|-----------|
| Thumbnails | `o_wsale_products_opt_design_thumbs` |
| Chips | `o_wsale_products_opt_design_chips` |
| Cards | `o_wsale_products_opt_design_cards` |
| Grid | `o_wsale_products_opt_design_grid` |
| Catalog layout | `o_wsale_products_opt_layout_catalog` |

---

## Animations

### On appearance

Triggered when element enters the viewport. Add to any column, text, or image element.

```html
<div class="col-lg-6 o_animate o_anim_fade_in o_anim_from_right"
     style="--wanim-intensity: 50; animation-duration: 1s; animation-delay: 0;">
    <!-- Content -->
</div>
```

- Add `o_animate_both_scroll` to re-trigger every time the element appears (default: only once)
- `animation-duration` and `animation-delay` go in the `style` attribute

Available effect classes: `o_anim_fade_in`, `o_anim_bounce_in`, `o_anim_rotate_in`, `o_anim_zoom_in`, `o_anim_slide_in`

Available direction modifiers: `o_anim_from_bottom`, `o_anim_from_right`, `o_anim_from_left`, `o_anim_from_top`

### On scroll

Triggered as the user scrolls through the element.

```html
<div class="col-lg-6 o_animate o_animate_on_scroll o_animate_out o_anim_fade_in o_anim_from_right"
     data-scroll-zone-start="50"
     data-scroll-zone-end="100"
     style="--wanim-intensity: 100;">
    <!-- Content -->
</div>
```

| Option | How to set | Type |
|--------|-----------|------|
| Intensity | `--wanim-intensity` CSS variable in `style` | Integer |
| Effect direction | `o_animate_in` or `o_animate_out` class | — |
| Scroll Zone Start | `data-scroll-zone-start` | Integer (0–100) |
| Scroll Zone End | `data-scroll-zone-end` | Integer (0–100) |

Available effects: Fade, Slide, Bounce, Rotate, Zoom Out, Zoom In.

### On hover (images only)

```html
<img src="..."
     class="img img-fluid o_we_custom_image o_animate_on_hover"
     data-hover-effect="overlay"
     data-hover-effect-color="rgba(0, 0, 0, 0.25)"
     data-hover-effect-intensity="50" />
```

| Option | Data attribute | Type |
|--------|---------------|------|
| Animation type | `data-hover-effect` | String |
| Intensity | `data-hover-effect-intensity` | Integer |
| Overlay / Stroke color | `data-hover-effect-color` | Hex or RGBA |
| Stroke width | `data-hover-stroke-width` | Integer (saved as px) |

Available effects: `overlay`, `zoom_in`, `zoom_out`, `dolly_zoom`, `outline`, `mirror_blur`.

---

## Forms

Forms are deeply integrated with Odoo apps. Build the form in the Website Builder first, then copy/paste the generated HTML to ensure all required fields are included.

```html
<form action="/website/form/" method="post"
      enctype="multipart/form-data"
      class="o_mark_required"
      data-mark="*"
      data-pre-fill="true"
      data-success-mode="redirect"
      data-success-page="/contactus-thank-you"
      data-model_name="mail.mail">
    <div class="s_website_form_rows row s_col_no_bgcolor">
        <div class="form-group s_website_form_field col-12 s_website_form_dnone" data-name="Field">
            <!-- form fields -->
        </div>
    </div>
</form>
```

### `data-model_name` actions

| Value | Action |
|-------|--------|
| `mail.mail` | Send an email (default) |
| `hr.applicant` | Apply for a job |
| `res.partner` | Create a customer |
| `helpdesk.ticket` | Create a support ticket |
| `crm.lead` | Create an opportunity |
| `project.task` | Create a task |

Note: Some actions need specific fields to be functional. Use the Website Builder to preset them, then copy the source.

### Success handling

| `data-success-mode` | Behavior |
|--------------------|---------|
| `redirect` | Redirect to the URL in `data-success-page` |
| `message` | Show a message on the same page |

Inline success message — always add `d-none` so it stays hidden until submitted:
```html
<div class="s_website_form_end_message d-none">
    <div class="oe_structure">
        <section class="s_text_block pt64 pb64" data-snippet="s_text_block">
            <div class="container">
                <h2 class="text-center">This is a success!</h2>
            </div>
        </section>
    </div>
</div>
```

---

## Translations

### Translatable QWeb attributes

Use `t-attf-` for translatable attributes (not `t-att-`):
```xml
<div t-attf-title="Hello #{user.name}" />     <!-- CORRECT — translatable -->
<div t-att-title="'Hello' + user.name" />     <!-- NOT translatable -->
```

`t-value` and `t-valuef` are NOT translatable. Text between XML tags IS translatable.

Reuse a translatable string:
```xml
<t t-set="label">Foo</t>
<div t-att-title="label" />
<nav t-att-title="label" />
```

Translatable `t-call` parameter:
```xml
<t t-call="website.layout" additional_title.translate="My Page" />
```

**Warning:** Every modification to the source language breaks the link with existing translations. Re-create all translations after editing source-language content.

### Export translations

1. Enable developer mode
2. **Settings → Translations → Export Translation**
3. Select the language, select **PO File** under File Format, select your module under Apps To Export
4. Download the file → move to `i18n/`

Or via backend: **Settings → Technical → User Interface: Views** → find the page → **Edit Translations**.

### Import translations

**Settings → Translations → Import Translation** → upload `.po` file.

Must be done manually on each new database after installation.

### PO file structure

Location: `/module_name/i18n/fr_BE.po`

```po
#. module: module_name
#: model_terms:ir.ui.view,arch_db:module_name.s_custom_snippet
msgid "Original text"
msgstr "Translated text"
```

---

## Going Live

### Odoo SaaS — first import

1. Create a ZIP of the module
2. Connect to the project database and enable developer mode
3. **Apps** → search for `base_import_module` → install if needed
4. **Apps** → **Import Module** → upload ZIP → tick **Force init** → **Import App**

On re-imports (after making changes):
1. Open developer menu → **Become Superuser** (log out and back in to exit)
2. Repeat the import steps — **do NOT tick Force init** on re-imports or existing data may be lost

ZIP size limit: **50MB**.

### Odoo.sh

**Apps** → **Update Apps List** → search for your module → install.

### Pre-launch checklist

- SEO: check page titles, meta descriptions, structured data
- URL redirects: set up any redirect mappings for old URLs
- Domain name: configure the domain in Website settings

---

## Theme Development Workflow

**Always follow this process before writing any code:**

1. **Ask for a design** — request a screenshot, mockup, or reference website before starting
2. **Cut the design into sections** — identify each distinct visual block:
   - Hero / banner
   - Card grids
   - Icon/feature rows
   - CTA / contact sections
   - Header & footer
3. **Map each section to a block** — each block gets its own:
   - XML template (`data/pages/home.xml` or `views/snippets/s_<name>.xml`)
   - SCSS file (`static/src/scss/pages/` or `static/src/scss/snippets/`)
   - JS only if interactive behaviour is needed
4. **Never start building** until the design is confirmed and all sections are identified
