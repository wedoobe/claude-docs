# How to create an Odoo 19 theme with Claude Code

A practical step-by-step guide. Follow this in order.

---

## Step 1 — Install Claude Code

Open your terminal and run:

```bash
npm install -g @anthropic-ai/claude-code
```

Then start it:

```bash
claude
```

Log in with your Anthropic account when prompted.

---

## Step 2 — Connect GitHub

You need a GitHub Personal Access Token so Claude can read and write files directly in your repos without cloning.

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens**
2. Create a token with these permissions:
   - **Contents** — read & write
   - **Metadata** — read
3. Copy the token, then run this in Claude Code:

```
claude mcp add-json github '{"type":"http","url":"https://api.githubcopilot.com/mcp","headers":{"Authorization":"Bearer YOUR_TOKEN_HERE"}}'
```

4. Verify it works:

```
claude mcp list
```

You should see `github - Connected`.

---

## Step 3 — Set up your Odoo.sh repo

1. Create a new repo on GitHub (e.g. `wedoobe/my-project`)
2. Connect it to Odoo.sh as a new branch/build
3. Add `CLAUDE.md` and `ODOO_REFERENCE.md` from this repo to the root of your project repo — Claude will use these as its rulebook

---

## Step 4 — Get a design

Before asking Claude to build anything, you need a design reference.

- Take a screenshot of a website you want to replicate, **or**
- Export a mockup from Figma / Adobe XD, **or**
- Draw a rough sketch and photograph it

> Claude will refuse to start building without a design. This is intentional.

---

## Step 5 — Start a Claude session in your project

Open Claude Code and tell it which repo to work in:

```
I want to create a new Odoo 19 theme in the repo wedoobe/my-project
```

Then paste or attach your design screenshot.

Claude will:
1. Analyse the design
2. Break it into sections (hero, cards, expertise, CTA, footer, etc.)
3. Ask you to confirm the section breakdown before writing any code
4. Scaffold the full module directly into your GitHub repo via the API

---

## Step 6 — Review the sections

Claude will list every section it identified from your design, for example:

```
Section 1 — Hero: full-width background image, centered title, diagonal bottom cut
Section 2 — Cards: 3-column image cards with title and description
Section 3 — Expertise: 4-column icon grid
Section 4 — Contact form: dark background, minimal input fields
Section 5 — Header: dark nav, centered logo, social link right
Section 6 — Footer: dark, social icons, nav links
```

Confirm or correct this before Claude starts building.

---

## Step 7 — Let Claude build

Once sections are confirmed, Claude pushes all files to your GitHub repo:

- `__init__.py`, `__manifest__.py`
- `data/presets.xml`, `data/pages/home.xml`
- `views/website_templates.xml`
- SCSS files for variables, theme, header, and each page section

No local clone needed — everything goes straight to GitHub.

---

## Step 8 — Deploy on Odoo.sh

1. The push triggers an automatic rebuild on Odoo.sh
2. If the branch is locked → click **Rebuild** in the Odoo.sh dashboard
3. Wait for the build to complete (it will cycle through `base → web → website → your module`)
4. Go to **Apps → Update Apps List**
5. Find your theme and click **Install**

> Rebuild alone does not apply XML changes — you must also install or upgrade the module.

---

## Step 9 — Review and iterate

Open your website in Odoo and compare it to your design reference.

If something is off, describe it to Claude:

```
The header logo is not centered — in the reference it sits exactly in the middle
The hero section is missing the diagonal bottom cut
The card images are not showing
```

Claude will update the files and push again. Repeat until the design matches.

---

## Step 10 — Add your images

Claude cannot upload images — you add those yourself via the Odoo Website Builder:

1. Click **Edit** on the page
2. Click on a placeholder image block
3. Upload your photo
4. Save and publish

---

## Files in this repo

| File | What it is |
|------|-----------|
| `README.md` | This guide |
| `CLAUDE.md` | Rules Claude follows when building Odoo themes |
| `ODOO_REFERENCE.md` | Full Odoo 19 theme API reference (Claude reads this too) |

---

## Quick reference — useful commands

| What you want | Say this to Claude |
|---------------|--------------------|
| Create a new theme | "Create a new Odoo theme called X in repo wedoobe/Y" |
| Fix a visual issue | "The header background is white, it should be black" |
| Add a new section | "Add a testimonials section below the expertise block" |
| Sync docs | "Sync CLAUDE.md and ODOO_REFERENCE.md to repo wedoobe/Y" |
| Check what's in a repo | "List all files in wedoobe/Y" |
