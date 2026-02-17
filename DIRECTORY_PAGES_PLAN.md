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
const { slug, data } = Astro.props;
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
