# County Directory Pages from Master Spreadsheet

## Overview

Build county-specific directory pages from a master Google Sheets spreadsheet. Each advertiser row has a multiselect field for which county directories they appear on. Pages are generated at build time and served via subdomains (e.g. `reeves-county-texas.mineralrightsforum.com`).

## Architecture

```mermaid
flowchart TB
    subgraph DataLayer
        GS[Google Sheets API]
        BuildScript[Build Script]
        JSON[src/data/directories.json]
    end
    
    subgraph Build
        getStaticPaths[getStaticPaths]
        PageTemplate[src/pages/county/[slug].astro]
        Components[DirectoryCard, CategorySection]
    end
    
    subgraph Deploy
        CFPages[Cloudflare Pages]
        Worker[Subdomain Worker]
    end
    
    GS -->|fetch at build| BuildScript
    BuildScript -->|write| JSON
    JSON --> getStaticPaths
    getStaticPaths --> PageTemplate
    PageTemplate --> Components
    PageTemplate -->|static HTML| CFPages
    CFPages --> Worker
    Worker -->|rewrite subdomain to path| CFPages
```

## 1. Spreadsheet Schema

### Master Advertisers Sheet

| Column       | Type        | Description                                                         |
| ------------ | ----------- | ------------------------------------------------------------------- |
| Company Name | string      | Display name                                                        |
| Category     | string      | e.g. "1031 Exchange Services", "Mineral Buyers"                     |
| Description  | string      | Short blurb                                                         |
| Featured     | boolean     | Show FEATURED badge                                                 |
| Email        | string      | mailto: link                                                        |
| Phone        | string      | tel: link                                                           |
| Website      | string      | Visit website link                                                  |
| Counties     | multiselect | Comma-separated or array: "reeves-county-texas,loving-county-texas" |

### Counties Reference Sheet (optional)

| County Slug         | Display Name      |
| ------------------- | ----------------- |
| reeves-county-texas | Reeves County, TX |
| loving-county-texas | Loving County, TX |

## 2. Data Pipeline

### 2.1 Build Script: `scripts/fetch-directory-data.mjs`

- Runs before or during `astro build`
- Uses `googleapis` with Service Account credentials
- Fetches advertiser rows from master sheet
- Parses multiselect Counties field (split by comma)
- Groups advertisers by county, then by category
- Outputs `src/data/directories.json`:

```json
{
  "reeves-county-texas": {
    "displayName": "Reeves County, TX",
    "categories": {
      "1031 Exchange Services": [
        { "name": "Accruit", "description": "...", "featured": true, "email": "...", "phone": "...", "website": "..." }
      ],
      "Mineral Buyers": [ ... ]
    }
  }
}
```

### 2.2 Environment Variables

- `GOOGLE_SHEETS_SPREADSHEET_ID` - Sheet ID from URL
- `GOOGLE_SERVICE_ACCOUNT_KEY` - JSON key file content (base64 or path for CI)

### 2.3 package.json Scripts

```json
"scripts": {
  "fetch-data": "node scripts/fetch-directory-data.mjs",
  "prebuild": "npm run fetch-data",
  "build": "astro build"
}
```

## 3. Page Generation

### 3.1 Dynamic Route: `src/pages/county/[slug].astro`

- Uses `getStaticPaths()` to read `src/data/directories.json`
- One static page per county slug
- Passes county data to layout components

### 3.2 getStaticPaths Pattern

```astro
---
export async function getStaticPaths() {
  const directories = await import('../data/directories.json');
  return Object.keys(directories).map((slug) => ({
    params: { slug },
    props: { data: directories[slug] }
  }));
}
Let me search for current information on both of these.Good, I have solid recent data on both. Here's a comparison:

---

**OpenAI Codex vs. Cursor Composer 1.5**

These two tools represent different philosophies in agentic coding — Codex is a standalone cloud agent you delegate work to, while Composer 1.5 is an interactive model baked into the Cursor IDE for daily coding sessions.

**What they are**

OpenAI Codex is a cloud-based software engineering agent that can work on many tasks in parallel, running each task in its own isolated sandbox environment preloaded with your repository. It's less of a coding companion and more of an autonomous worker you send off to do a job and come back to later.

Cursor's Composer 1.5, by contrast, is designed to "strike a strong balance between speed and intelligence for daily use." It lives inside the Cursor IDE and is optimized for real-time, interactive development sessions.

**How they're trained**

Codex is powered by codex-1 (now GPT-5.3-Codex), a version of OpenAI's o3/GPT-5 optimized for software engineering via reinforcement learning on real-world coding tasks.

Composer 1.5 was built by scaling reinforcement learning 20x on the same pretrained model as Composer 1, with post-training compute ultimately surpassing pretraining compute.

**Key technical differentiators**

Composer 1.5 has two standout architectural features that directly address real developer pain points. Its adaptive thinking calibrates reasoning depth to task difficulty — simple tasks like variable renaming get quick responses with minimal thinking tokens, while complex multi-step tasks trigger deeper reasoning. And when context runs out, Composer 1.5 produces trained self-summaries to continue exploring solutions, triggering recursively on hard examples without accuracy loss.

Codex's differentiator is parallelism and autonomy. Its app functions as a command center where agents work in parallel across projects, with Automations that let Codex work unprompted on routine tasks like issue triage, alert monitoring, and CI/CD.

**Where they live**

Codex runs everywhere — a dedicated web app, CLI, IDE extension (including VS Code), and GitHub integration. The Codex CLI is open-source and supports attaching images like screenshots and wireframes to build shared context on design decisions.

Composer 1.5 is exclusive to the Cursor IDE (built on VS Code). You can't access it via a standalone API or other editors, which is a deliberate design choice that enables tight integration with your local codebase.

**Pricing**

Composer 1.5 is priced at $3.50 per million input tokens and $17.50 per million output tokens, available on paid Cursor plans. Codex is included with ChatGPT Plus, Pro, Business, and Enterprise plans.

If you want an autonomous agent that runs tasks in the background — writing features, filing PRs, running tests while you're focused elsewhere — Codex is built for that. If you're a developer who lives in your IDE and wants a fast, deeply integrated assistant that thinks alongside you in real time, Composer 1.5 is the stronger fit. They're not really competing for the same workflow; many developers could reasonably use both. { slug, data } = Astro.props;
---
```

## 4. Components

### 4.1 DirectoryCard.astro

- Props: `name`, `category`, `description`, `featured`, `email`, `phone`, `website`
- Uses Starwind `Card`, `Badge` (for FEATURED), `Button`
- Renders contact links (Email us, Call now, Visit website)

### 4.2 CategorySection.astro

- Props: `categoryName`, `listings` (array)
- Renders `<h2>` category heading
- Maps listings to `DirectoryCard` components in a grid

### 4.3 DirectoryPage.astro (or inline in [slug].astro)

- Props: `countyName`, `categories` (object)
- Renders hero/intro, category filter (Tabs), then `CategorySection` for each category
- Client-side filter: Tabs to show/hide categories (or scroll-to)

### 4.4 Shared Layout

- Reuse existing `Navigation`, `Footer`, `Layout`
- Navigation links: Back to Forum, Browse County Directories

## 5. Subdomain Routing (Cloudflare)

### 5.1 Problem

Cloudflare Pages serves one build. Adding `reeves-county-texas.mineralrightsforum.com` as a custom domain would serve the root `/` (index.html), not the county content.

### 5.2 Solution: Cloudflare Worker

- Deploy a Worker in front of Pages (or use Pages Functions)
- On request to `*.mineralrightsforum.com`:
  1. Extract subdomain (e.g. `reeves-county-texas`)
  2. Rewrite request to `https://pages-project.mineralrightsforum.com/county/reeves-county-texas/`
  3. Fetch from Pages origin and return response

### 5.3 Worker Logic (pseudo)

```js
const subdomain = url.hostname.split('.')[0];
if (subdomain !== 'www' && subdomain !== 'directories') {
  const newUrl = new URL(`/county/${subdomain}/`, pagesOrigin);
  return fetch(newUrl);
}
return env.ASSETS.fetch(request);
```

### 5.4 Cloudflare Setup

- Pages project: `mineralrightsforum.com` (or `directories.mineralrightsforum.com`)
- Worker: Route `*.mineralrightsforum.com/*` to Worker
- Worker forwards to Pages for static assets and county paths

## 6. File Structure

```
src/
├── data/
│   └── directories.json          # Generated at build
├── pages/
│   ├── index.astro               # Main landing / county index
│   └── county/
│       └── [slug].astro          # County directory page
├── components/
│   ├── DirectoryCard.astro
│   ├── CategorySection.astro
│   ├── DirectoryPage.astro       # Optional wrapper
│   ├── Navigation.astro
│   └── ...
scripts/
├── fetch-directory-data.mjs      # Google Sheets fetch + transform
└── package.json updates
functions/                        # If using Pages Functions for subdomain
└── [[path]].ts                   # Optional: subdomain rewrite
```

## 7. Dependencies to Add

- `googleapis` - Google Sheets API client
- `dotenv` - Load env vars for local dev

## 8. .cursorrules Additions

Suggested rules for the real project:

- **Data flow**: Directory data comes from `src/data/directories.json`; never hardcode advertiser data in components
- **Component props**: DirectoryCard and CategorySection receive data via props; no direct data imports
- **County slugs**: Use kebab-case, lowercase (e.g. `reeves-county-texas`)
- **Spreadsheet**: Multiselect Counties field format is comma-separated slugs

## 9. Implementation Order

1. Create `scripts/fetch-directory-data.mjs` with mock/static data first
2. Define `directories.json` schema and sample file
3. Build `DirectoryCard` and `CategorySection` components
4. Create `src/pages/county/[slug].astro` with `getStaticPaths`
5. Wire up Google Sheets API (credentials, env vars)
6. Add prebuild script
7. Deploy to Cloudflare Pages
8. Add Worker for subdomain routing
9. Configure wildcard custom domain

## 10. Reference: Mineral Rights Directory Structure

From [reeves-county-texas.mineralrightsforum.com](https://reeves-county-texas.mineralrightsforum.com):

- Filter by Category (pills)
- Category sections with h2 headers
- Cards: FEATURED badge, company name, category, description, Email/Call/Website links
- Tips section at bottom
- "Browse County Directories" CTA
