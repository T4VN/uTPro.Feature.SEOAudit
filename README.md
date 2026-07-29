# uTPro SEO Audit for Umbraco

Site-wide SEO & content audit for **Umbraco 16, 17 and 18**. Crawls every Content and Media URL,
checks for broken links/images/resources, and audits each page — producing a **composite SEO score**,
a **health score**, prioritised **issues**, per-page details and **CSV export**, all in a dedicated
backoffice section.

[![NuGet](https://img.shields.io/nuget/v/uTPro.Feature.SEOAudit.svg)](https://www.nuget.org/packages/uTPro.Feature.SEOAudit)
[![NuGet Downloads](https://img.shields.io/nuget/dt/uTPro.Feature.SEOAudit.svg)](https://www.nuget.org/packages/uTPro.Feature.SEOAudit)
[![Umbraco Marketplace](https://img.shields.io/badge/Umbraco-Marketplace-blue)](https://marketplace.umbraco.com/package/utpro.feature.seoaudit)
[![Umbraco 16+](https://img.shields.io/badge/Umbraco-16%2B-3544B1)](https://umbraco.com)
[![License: Free (proprietary)](https://img.shields.io/badge/License-Free%20(proprietary)-green.svg)](LICENSE.txt)

![uTPro SEO Audit — Site Audit view](https://raw.githubusercontent.com/T4VN/uTPro.Feature.SEOAudit/refs/heads/main/Images/Screenshots/SiteScan.png)

## What it adds

Installing this package extends the **SEO Audit** section (created by `uTPro.Feature.UrlViewer` or by
this package when installed standalone) with:

| Where | What |
|---|---|
| **Site Audit** dashboard | Full site crawl, health score, SEO score, issues, per-page detail modal, CSV export |
| **Error URLs** dashboard | Standing list of failing URLs with single or bulk re-scan |
| Content/Media editors | **SEO Audit** tab per node (auto-scans on open, warns on issues) |

## Architecture

```
T4VN.Seo.Core  (engine, no Umbraco)
      ▲                  ▲
      │                  │
uTPro.Feature.UrlViewer  uTPro.Feature.SEOAudit
(URL Viewer tab +        (this package — site crawler
 shared UI + section)     + node audit tab)
```

Both packages share the same [**T4VN.Seo.Core**](https://www.nuget.org/packages/T4VN.Seo.Core) engine
and `SeoScorer` — the single source of truth for the SEO score. Install order and combination are
flexible; each package works independently.

## Site Audit

A recurring background job (auto-discovered by **uTPro Job Monitor**) crawls every **Content** and
**Media** URL and, optionally, URLs in `sitemap.xml`.

- **Broken links, images & resources** — `<a>`, `<img>`, stylesheets, scripts, fonts checked for a valid status code.
- **Health score** per run + **Overview** of totals (pages, links, images, resources, broken, orphaned, duplicate).
- **Composite SEO score** per page and **Avg SEO Score** across the run.
- **Issues** grouped by category with severity, type and priority.
- **Orphaned pages** (no internal link) and **duplicate content** (identical HTML hash) detection.
- **Incremental scans** — unchanged pages (ETag / Last-Modified) are skipped; previous result is carried forward.
- **Respects `robots.txt`** (disallow rules + `Sitemap:` discovery); supports **include/exclude URL patterns**.
- **Core Web Vitals** (optional) via the Google PageSpeed Insights API — no local browser needed.
- **CSV export** of any run.
- **Extensible checks** — implement `IUrlScanIssue` and register in DI; it appears automatically in the Issues view.

### Per-page audit (via modal detail)

- **SEO** — title (+ length), meta description (+ length), canonical, H1/H2/H3, `noindex`, `nofollow`, `lang`.
- **Social** — Open Graph, Twitter Card, pixel detection, social-profile links.
- **Technical SEO** — charset, gzip/br, browser caching, HTTPS, schema.org, viewport, favicon.
- **Content** — word/paragraph count, Flesch readability, keyword density, thin-content flag.
- **Accessibility** — aria counts, skip-to-content, heading structure, `lang`, inputs without label.
- **Carbon** — CO₂e estimate with A–F rating.
- **Core Web Vitals** (optional) — Lighthouse score, LCP, CLS, FCP, TBT, Speed Index + real-user field data.

### Built-in issue checks

HTTP errors, server errors, broken links, broken resources, missing/too-long meta description,
missing H1, missing H2, `nofollow`, `noindex`, canonicalised URLs, orphaned pages, thin content,
duplicate content, images missing alt text, high carbon intensity, spam/hack keywords, cloaking,
JavaScript errors, poor Core Web Vitals.

## Per-node SEO Audit tab

Adds a **SEO Audit** tab inside every **Content** and **Media** editor. Shown only for users with
**Settings** access, and only when the node has a public routable URL (template-less nodes are
automatically excluded). Opening a node auto-audits its URL(s) in the background and shows a footer
warning when an issue is found.

## Security

Backoffice-secured under `/umbraco/management/api/v1/utpro/url-scan`, requires Settings-section access.
Crawl fetches run behind the same SSRF guard as the URL Viewer (RFC-1918 / localhost / `.local`
blocked, re-checked on every redirect hop). `SkipNodesWithoutTemplate = true` (default) prevents
template-less nodes (folders, navigation-link containers, etc.) from being scanned — they have no
routable URL and would produce only false-positive 404s.

## Installation

```bash
dotnet add package uTPro.Feature.SEOAudit
```

> **Tip:** install `uTPro.Feature.UrlViewer` as well to get the manual **URL Viewer** fetch tool in
> the same section.

## Configuration

Bind under `uTPro:Feature:SEOAudit` in `appsettings.json` (all keys optional — defaults shown):

```json
{
  "uTPro": {
    "Feature": {
      "SEOAudit": {
        "Enabled": true,
        "Period": "24:00:00",
        "Delay": "00:05:00",
        "MaxConcurrency": 4,
        "ThrottleDelayMs": 150,
        "SkipCloakingCheck": true,
        "AllowInternalHosts": false,
        "AllowInvalidCertificates": false,
        "RedirectWarningThreshold": 3,
        "MaxRunHistory": 20,
        "SkipNodesWithoutTemplate": true,
        "CheckLinks": true,
        "CheckExternalLinks": true,
        "CheckImages": true,
        "LinkCheckTimeoutSeconds": 15,
        "MaxLinkChecksPerRun": 5000,
        "UseIncrementalScan": true,
        "UseSitemapDiscovery": true,
        "RespectRobotsTxt": true,
        "ExcludePatterns": [],
        "IncludePatterns": [],
        "CollectCoreWebVitals": false,
        "PageSpeedApiKey": "",
        "PageSpeedStrategy": "mobile"
      }
    }
  }
}
```

| Key | Default | Description |
|-----|---------|-------------|
| `Enabled` | `true` | Master switch for the recurring audit job. |
| `Period` | `24:00:00` | How often the audit runs. |
| `Delay` | `00:05:00` | Delay before the first run after startup. |
| `MaxConcurrency` | `4` | Max concurrent HTTP fetches (clamped 1–20). |
| `ThrottleDelayMs` | `150` | Delay after each fetch (ms). |
| `SkipCloakingCheck` | `true` | Skip bot-vs-Chrome comparison during bulk audits (faster). |
| `AllowInternalHosts` | `false` | Relax SSRF guard for internal/dev sites. |
| `AllowInvalidCertificates` | `false` | Accept self-signed / invalid TLS (dev only). |
| `RedirectWarningThreshold` | `3` | Hop count above which a redirect chain is flagged. |
| `MaxRunHistory` | `20` | Runs retained in the DB before old runs are pruned. |
| `SkipNodesWithoutTemplate` | `true` | Skip content nodes with no template (they have no routable URL). Set `false` for controller-routed template-less nodes. |
| `CheckLinks` | `true` | Check link status codes. |
| `CheckExternalLinks` | `true` | Also check external links. `false` to avoid any outbound requests. |
| `CheckImages` | `true` | Check image URLs. |
| `LinkCheckTimeoutSeconds` | `15` | Per-request timeout for link/image checks (clamped 1–60). |
| `MaxLinkChecksPerRun` | `5000` | Safety cap on distinct URLs checked per run. |
| `UseIncrementalScan` | `true` | Skip unchanged pages on scheduled runs. |
| `UseSitemapDiscovery` | `true` | Discover additional URLs from `sitemap.xml`. |
| `RespectRobotsTxt` | `true` | Honour `robots.txt` disallow rules and `Sitemap:` directives. |
| `ExcludePatterns` | `[]` | URL patterns to skip (wildcards `*` / `**`). |
| `IncludePatterns` | `[]` | If set, only matching URLs are scanned. |
| `CollectCoreWebVitals` | `false` | Collect Core Web Vitals via PageSpeed Insights (requires API key). |
| `PageSpeedApiKey` | `""` | Google PageSpeed Insights API key. |
| `PageSpeedStrategy` | `mobile` | `mobile` or `desktop`. |

## Extensibility

Register a custom issue check in DI:

```csharp
services.AddScoped<IUrlScanIssue, MyCustomIssue>();
```

`IUrlScanIssue` exposes `Alias`, `Name`, `Description`, `Category`, `Severity`, `Type`,
`Priority` and `bool Matches(ScanResultRow row)`. It appears automatically in the Issues view.

## License

Free to use (including commercially) under a proprietary [End User License Agreement](LICENSE.txt).
