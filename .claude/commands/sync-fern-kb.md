# Sync Fern KB from Notion

Syncs Truemed KB article content from Notion to GitHub so Fern can publish the latest docs.

**Usage:** `/sync-fern-kb [merchant|customer|both]`
- Defaults to asking if no argument is provided.
- `$ARGUMENTS`

---

## Step 1 — Determine which KB(s) to sync

If `$ARGUMENTS` is empty, ask the user: merchant KB, customer KB, or both?

- **Merchant KB**
  - Notion DB: `https://www.notion.so/26410cce66ec461f9d587b62315363d7`
  - Notion view ID: `204b4c0c-4aa7-4d7a-a107-156ca0343455`
  - GitHub repo: `https://github.com/truemedicine/truemed-merchant-knowledge-base`
  - Local clone path: `/tmp/truemed-merchant-kb`
  - Fern pages root: `/tmp/truemed-merchant-kb/fern/pages/`

- **Customer KB**
  - Notion DB: `https://www.notion.so/a753b051350641309b1a9e4095cce6c2` *(fetch the DB to get the view ID)*
  - GitHub repo: `https://github.com/truemedicine/truemed-customer-knowledge-base`
  - Local clone path: `/tmp/truemed-kb`
  - Fern pages root: `/tmp/truemed-kb/fern/pages/`

---

## Step 2 — Clone or pull the repo

For each KB being synced:

```bash
# If not already cloned:
git clone https://github.com/truemedicine/[repo-name].git /tmp/[local-path]

# If already cloned:
cd /tmp/[local-path] && git pull origin main
```

---

## Step 3 — Query Notion for articles

Use `notion-query-database-view` with the view URL:
- Merchant: `https://www.notion.so/26410cce66ec461f9d587b62315363d7?v=204b4c0c-4aa7-4d7a-a107-156ca0343455`
- Customer: fetch the DB first to get its view URL

**Filter:** Only process articles that are NOT blank and NOT "Blocked" status. Skip articles where the Notion page returns `<blank-page>`.

---

## Step 4 — Fetch each article's content

For each article URL from the query results, call `notion-fetch` to get full page content.

Note which articles are blank (return `<blank-page>`) — skip writing those files, leave existing stubs in place.

---

## Step 5 — Convert Notion content to MDX

Apply these transformations to each article's `<content>` block:

### Frontmatter
Add at the top of every file:
```
---
title: [Article title from Notion Markdown property]
description: [First sentence of the article body, or a short summary]
---
```

### Notion HTML tables → Markdown tables
Convert:
```html
<table header-row="true">
<tr><td>Col1</td><td>Col2</td></tr>
<tr><td>Val1</td><td>Val2</td></tr>
</table>
```
To:
```
| Col1 | Col2 |
|------|------|
| Val1 | Val2 |
```

### Blockquote callouts → Fern components
- `> 📸 **Image placeholder:** ...` → Remove entirely (placeholder, not real content)
- `> 🎬 **Video embed:** Title — URL` → `<Tip>Watch the [Title](URL) walkthrough video.</Tip>`
- `> **Important:**` or `> ⚠️` text → `<Warning>text</Warning>`
- `> **Tip:**` text → `<Tip>text</Tip>`
- `> **Note:**` text or standalone blockquote notes → `<Note>text</Note>`
- Regular blockquotes (non-callout) → keep as `> text`

### Internal links
Replace Notion page URLs with Fern relative paths:
- `https://www.notion.so/docs/[slug]` or `(https://www.notion.so/[id])` → use the path mapping tables below
- Links to external sites (truemed.com, app.truemed.com, stripe.com, etc.) → keep as-is, ensure `https://` prefix

### Checkboxes
Keep `☐` as-is (used in prerequisite checklists).

### Everything else
Code blocks, bullet lists, numbered lists, bold, italic, horizontal rules → keep as-is.

---

## Step 6 — Determine file path for each article

### Merchant KB path mapping

Use the article's **Nav Group** and **Section** properties:

| Nav Group | Section | Folder |
|-----------|---------|--------|
| Partnering with Truemed | (any) | `partnering-with-truemed/` |
| Getting Started | Payment Integration | `getting-started/payment-integration/` |
| Getting Started | Stripe Setup | `getting-started/stripe-setup/` |
| Implementation | Shopify | `implementation/shopify/` |
| Implementation | Reimbursements | `implementation/reimbursements/` |
| Troubleshooting | (any) | `troubleshooting/` |
| Account Management | (any) | `account-management/` |

**File name:** Convert the article title to kebab-case + `.mdx`.

Special cases where Notion title doesn't match the file name:
| Notion Title | Platform | File |
|--------------|----------|------|
| Welcome / Intro | Shopify | `welcome-intro-shopify.mdx` |
| Welcome / Intro (1) | WooCommerce | `welcome-intro-woocommerce.mdx` |
| Setup Guide | Shopify | `implementation/shopify/setup-guide.mdx` |
| Setup Guide | Reimbursements | `implementation/reimbursements/setup-guide.mdx` |
| Shopify Issues (including Manual Capture) | — | `shopify-issues.mdx` |

### Customer KB path mapping

Use the article's **Nav Group** and **Section** properties. Map to the exact file paths defined in `fern/docs.yml`. Fetch the docs.yml first if the exact paths are unclear.

---

## Step 7 — Write the MDX files

For each article with content, write the converted MDX to the correct path under `fern/pages/`. Overwrite existing stubs.

Do not write files for blank Notion pages — leave existing stubs in place.

---

## Step 8 — Commit and push

```bash
cd /tmp/[local-path]
git add fern/pages/
git diff --cached --stat   # show what changed
git commit -m "Sync KB articles from Notion — $(date +%Y-%m-%d)"
git push origin main
```

If no files changed, tell the user everything is already up to date.

---

## Step 9 — Monitor the build

```bash
gh run list --repo truemedicine/[repo-name] --limit 1
gh run watch [run-id] --exit-status
```

Report the result to the user. If the build fails, show the error output from `gh run view [run-id] --log-failed`.

---

## After syncing

Tell the user:
- How many articles were updated vs. skipped (blank)
- Which articles were skipped and why
- The build status (pass/fail)
- The live URL to check: `https://truemed-merchants.docs.buildwithfern.com` or `https://truemed-support.docs.buildwithfern.com`
