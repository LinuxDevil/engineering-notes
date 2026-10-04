---
title: "Next.js Internationalization, Metadata & SEO"
tags: [frontend, frontend/nextjs, frontend/seo, frontend/i18n]
aliases: [Next.js i18n, Next.js SEO, Next.js Metadata, generateMetadata, next-intl, hreflang, sitemap.ts, robots.ts, opengraph-image]
updated: 2026-09-27
---
# Next.js Internationalization, Metadata & SEO

<Note>
**TL;DR**

In the App Router, i18n is **routing + dictionaries**: a `[locale]` root segment, locale negotiation in `proxy.ts` (redirect `/` to `/de` from `Accept-Language` or a cookie), and translations loaded **on the server** so they don't ship as client JS. **next-intl** (4.x, current 4.14 in Sept 2026) packages this, and since Next **16.3** reads the locale from `next/root-params` instead of `setRequestLocale`. SEO comes from the **Metadata API** (`metadata` / `generateMetadata`, merged down the tree), file conventions (`opengraph-image`, `sitemap.ts`, `robots.ts`), JSON-LD in the page, and correct `canonical` + `hreflang` alternates. Metadata **streams** for normal browsers and Googlebot but still **blocks** for HTML-limited bots (e.g. `facebookexternalhit`), configurable with `htmlLimitedBots`.
</Note>

## 1. i18n routing model

```text
app/
├── [locale]/                 # root segment: /en/..., /de/...
│   ├── layout.tsx            # the root layout: renders <html lang={locale}>
│   ├── page.tsx
│   ├── products/[id]/page.tsx
│   └── not-found.tsx
├── sitemap.ts
└── robots.ts
proxy.ts                      # locale negotiation + redirect
i18n/
├── config.ts                 # locales, defaultLocale
└── dictionaries/{en,de}.json
```

Decisions to make up front:

| Decision | Options | Recommendation |
|---|---|---|
| URL strategy | Prefix (`/de/...`), subdomain (`de.acme.dev`), ccTLD (`acme.de`), cookie-only | **Prefix**: one deployment, crawlable, shareable. Cookie-only locales are invisible to search engines |
| Default locale | Prefixed (`/en`) or unprefixed (`/`) | `as-needed` prefix is common; always prefixed is simplest to reason about |
| Negotiation | `Accept-Language`, cookie (`NEXT_LOCALE`), geo | Cookie (explicit choice) > `Accept-Language` > default. **Never** redirect bots by geo |
| Translated slugs | `/de/produkte/...` vs `/de/products/...` | Only if SEO in that market needs it (next-intl `pathnames`) |

{/* diagram source: /diagrams/diagram-289.mmd */}
![diagram](/diagrams/diagram-289.svg)

## 2. Without a library: the official pattern

```ts
// i18n/config.ts
export const locales = ["en", "de", "fr"] as const;
export type Locale = (typeof locales)[number];
export const defaultLocale: Locale = "en";
export const hasLocale = (l: string): l is Locale => (locales as readonly string[]).includes(l);
```

```ts
// proxy.ts  (npm i negotiator @formatjs/intl-localematcher)
import { NextResponse, type NextRequest } from "next/server";
import Negotiator from "negotiator";
import { match } from "@formatjs/intl-localematcher";
import { locales, defaultLocale, hasLocale } from "./i18n/config";

function getLocale(req: NextRequest): string {
  const cookie = req.cookies.get("NEXT_LOCALE")?.value;
  if (cookie && hasLocale(cookie)) return cookie;
  const languages = new Negotiator({
    headers: { "accept-language": req.headers.get("accept-language") ?? "" },
  }).languages();
  try {
    return match(languages, locales as unknown as string[], defaultLocale);
  } catch {
    return defaultLocale; // malformed header
  }
}

export function proxy(req: NextRequest) {
  const { pathname } = req.nextUrl;
  const hasPrefix = locales.some((l) => pathname === `/${l}` || pathname.startsWith(`/${l}/`));
  if (hasPrefix) return NextResponse.next();

  const url = req.nextUrl.clone();
  url.pathname = `/${getLocale(req)}${pathname}`;
  return NextResponse.redirect(url);
}

export const config = {
  // skip API, Next internals, metadata files and anything with a file extension
  matcher: ["/((?!api|_next/static|_next/image|favicon.ico|sitemap.xml|robots.txt|.*\\..*).*)"],
};
```

```ts
// i18n/dictionaries.ts
import "server-only";                          // translations never ship as a client module
import type { Locale } from "./config";

const dictionaries = {
  en: () => import("./dictionaries/en.json").then((m) => m.default),
  de: () => import("./dictionaries/de.json").then((m) => m.default),
  fr: () => import("./dictionaries/fr.json").then((m) => m.default),
};
export const getDictionary = async (locale: Locale) => dictionaries[locale]();
```

```tsx
// app/[locale]/layout.tsx
import { notFound } from "next/navigation";
import { locales, hasLocale } from "@/i18n/config";

export function generateStaticParams() {
  return locales.map((locale) => ({ locale }));   // prerender every locale
}

export default async function LocaleLayout(props: LayoutProps<"/[locale]">) {
  const { locale } = await props.params;
  if (!hasLocale(locale)) notFound();
  return (
    <html lang={locale} dir={locale === "ar" ? "rtl" : "ltr"}>
      <body>{props.children}</body>
    </html>
  );
}
```

```tsx
// app/[locale]/products/[id]/page.tsx
import { getDictionary } from "@/i18n/dictionaries";
import type { Locale } from "@/i18n/config";

export default async function ProductPage(props: PageProps<"/[locale]/products/[id]">) {
  const { locale, id } = await props.params;
  const [t, product] = await Promise.all([getDictionary(locale as Locale), getProduct(id, locale)]);
  const price = new Intl.NumberFormat(locale, { style: "currency", currency: product.currency }).format(product.price);
  return (
    <article>
      <h1>{product.name}</h1>
      <p>{t.product.price}: {price}</p>
      <AddToCartButton label={t.product.addToCart} productId={id} />  {/* pass strings, not the dictionary */}
    </article>
  );
}
```

Since **16.3**, `next/root-params` lets any Server Component read the root param without prop drilling (and it works inside `'use cache'` scopes):

```tsx
import { locale } from "next/root-params";   // the export name matches your [locale] segment
export async function Footer() {
  const t = await getDictionary((await locale()) as Locale);
  return <footer>{t.footer.copyright}</footer>;
}
```

## 3. next-intl (4.x)

next-intl is the most used App Router i18n library: ICU message syntax (plurals, select, rich text), typed messages, locale-aware navigation, formatting (dates, numbers, relative time), and translations in **Server Components without shipping messages to the client** unless you pass them to a provider.

```ts
// src/i18n/routing.ts
import { defineRouting } from "next-intl/routing";
export const routing = defineRouting({
  locales: ["en", "de", "fr"],
  defaultLocale: "en",
  localePrefix: "as-needed",              // "/" stays English, "/de" for German
});
```

```ts
// src/proxy.ts   (middleware.ts before Next 16)
import createMiddleware from "next-intl/middleware";
import { routing } from "./i18n/routing";
export default createMiddleware(routing);  // negotiation, redirects, NEXT_LOCALE cookie, alternate Link headers
export const config = { matcher: "/((?!api|trpc|_next|_vercel|.*\\..*).*)" };
```

```ts
// src/i18n/request.ts  (current docs: locale from next/root-params on Next 16.3+)
import * as rootParams from "next/root-params";
import { notFound } from "next/navigation";
import { getRequestConfig } from "next-intl/server";
import { hasLocale } from "next-intl";
import { routing } from "./routing";

export default getRequestConfig(async ({ locale }) => {
  if (!locale) {
    const paramValue = await rootParams.locale();
    if (hasLocale(routing.locales, paramValue)) locale = paramValue;
    else notFound();
  }
  return {
    locale,
    messages: (await import(`../../messages/${locale}.json`)).default,
  };
});
```

```ts
// src/i18n/navigation.ts: locale-aware Link/redirect/router
import { createNavigation } from "next-intl/navigation";
import { routing } from "./routing";
export const { Link, redirect, usePathname, useRouter, getPathname } = createNavigation(routing);
```

```ts
// next.config.ts
import createNextIntlPlugin from "next-intl/plugin";
const withNextIntl = createNextIntlPlugin();   // points to src/i18n/request.ts by default
export default withNextIntl({ cacheComponents: true });
```

Using translations:

```tsx
// Server Component (async): getTranslations
import { getTranslations } from "next-intl/server";
export default async function CartPage() {
  const t = await getTranslations("Cart");
  const items = await getCartItems();
  return <h1>{t("title", { count: items.length })}</h1>;   // "{count, plural, =0 {Empty cart} one {# item} other {# items}}"
}

// Non-async Server Component or Client Component: useTranslations
import { useTranslations } from "next-intl";
export function EmptyState() {
  const t = useTranslations("Cart");
  return <p>{t("empty")}</p>;
}
```

```tsx
// app/[locale]/layout.tsx: only client-needed namespaces go to the browser
import { NextIntlClientProvider } from "next-intl";
import { getMessages } from "next-intl/server";
import pick from "lodash/pick";

export default async function LocaleLayout(props: LayoutProps<"/[locale]">) {
  const { locale } = await props.params;
  const messages = await getMessages();
  return (
    <html lang={locale}>
      <body>
        <NextIntlClientProvider messages={pick(messages, ["Cart", "Toasts"])}>
          {props.children}
        </NextIntlClientProvider>
      </body>
    </html>
  );
}
```

Notes on the current API:
- Since 4.0, `NextIntlClientProvider` **inherits `messages` and `formats` from the server config automatically**. That's convenient but means every message ships to the browser; pass an explicit subset (as above) on large apps.
- **`setRequestLocale` is legacy** in the current docs: it was needed for static rendering before `next/root-params`; still supported, but on 16.3+ use root params. On older Next versions root params need `experimental.rootParams`.
- The dynamic segment must be named **`[locale]`** for the integration.
- `useExtracted` (Nov 2025) lets you write source-language strings inline and extract message keys automatically.
- **Ahead-of-time message compilation** (next-intl 4.8, Jan 2026, experimental `experimental.messages.precompile`) removes the ICU parser from the client bundle (about 9 KB less) and speeds up formatting; `t.raw` isn't compatible.

## 4. The Metadata API

```tsx
// app/[locale]/layout.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  metadataBase: new URL("https://shop.acme.dev"),  // resolves relative OG/canonical URLs
  title: { default: "Acme Shop", template: "%s | Acme Shop" },
  description: "Outdoor gear, delivered.",
  openGraph: { siteName: "Acme Shop", type: "website" },
  twitter: { card: "summary_large_image" },
  robots: { index: true, follow: true },
};
```

```tsx
// app/[locale]/products/[id]/page.tsx
import type { Metadata } from "next";
import { getTranslations } from "next-intl/server";
import { cache } from "react";
import { notFound } from "next/navigation";
import { locales } from "@/i18n/config";

const getProductCached = cache(getProduct);   // dedupe with the page's own call in this request

export async function generateMetadata(props: PageProps<"/[locale]/products/[id]">): Promise<Metadata> {
  const { locale, id } = await props.params;
  const product = await getProductCached(id, locale);
  if (!product) notFound();
  const t = await getTranslations({ locale, namespace: "Product" });

  return {
    title: product.name,                                 // "Trail Runner 2 | Acme Shop"
    description: t("metaDescription", { name: product.name }),
    alternates: {
      canonical: `/${locale}/products/${id}`,
      languages: {
        ...Object.fromEntries(locales.map((l) => [l, `/${l}/products/${id}`])),
        "x-default": `/en/products/${id}`,
      },
    },
    openGraph: { type: "website", locale, url: `/${locale}/products/${id}` },
  };
}
```

How it works:
- `metadata` (static) or `generateMetadata` (dynamic) can be exported from `layout` and `page` in Server Components only; not both from the same segment.
- Metadata **merges from root to leaf**; later segments override fields. Nested objects like `openGraph` are **replaced, not deep-merged**, so share base OG fields through a helper.
- File-based metadata (`opengraph-image`, `icon`, `sitemap`, `robots`) has higher priority than the object.
- `fetch` calls inside `generateMetadata` are memoized with the page's; for DB/ORM calls wrap them in React `cache()`.
- Viewport and theme color go in `generateViewport`/`viewport`, not `metadata`.

### Streaming metadata and bots (15.2+)

| Visitor | Behaviour |
|---|---|
| Browsers and JS-executing bots (Googlebot) | The initial UI is sent **without waiting** for `generateMetadata`; tags are appended when it resolves (inside `<body>`). Next says it verified Googlebot reads them from the rendered DOM |
| **HTML-limited bots** (e.g. `facebookexternalhit`, detected by User-Agent) | Metadata **blocks** rendering and is placed in `<head>` |
| Prerendered pages where `generateMetadata` has no dynamic behaviour | Metadata is in the initial HTML `<head>` for everyone |

```ts
// next.config.ts: add bots to the blocking list, or disable streaming metadata entirely
const nextConfig: NextConfig = {
  htmlLimitedBots: /facebookexternalhit|LinkedInBot|Slackbot|Twitterbot|MyInternalCrawler/,
  // htmlLimitedBots: /.*/,   // always block (slower TTFB)
};
```

**With Cache Components**: if `generateMetadata` reads request data (`cookies()`, `headers()`, `params`, `searchParams`) or uncached data while the rest of the page is fully prerenderable, Next raises an error asking you to decide: cache it with `'use cache'` inside `generateMetadata` if it only depends on external data, or mark the page as intentionally dynamic (a Suspense-wrapped component that awaits `connection()`).

```ts
export async function generateMetadata() {
  "use cache";
  const { title, description } = await cms.siteMeta();
  return { title, description };
}
```

## 5. Open Graph images with `ImageResponse`

```tsx
// app/[locale]/products/[id]/opengraph-image.tsx
import { ImageResponse } from "next/og";

export const alt = "Acme Shop product";
export const size = { width: 1200, height: 630 };
export const contentType = "image/png";

export default async function Image({ params }: { params: Promise<{ locale: string; id: string }> }) {
  const { locale, id } = await params;                 // Promise since Next 16
  const product = await getProduct(id, locale);
  return new ImageResponse(
    (
      <div style={{ display: "flex", width: "100%", height: "100%", background: "#0f172a", color: "white", padding: 64, flexDirection: "column", justifyContent: "space-between" }}>
        <div style={{ fontSize: 36, opacity: 0.8 }}>Acme Shop</div>
        <div style={{ fontSize: 72, fontWeight: 700 }}>{product.name}</div>
        <div style={{ fontSize: 40 }}>{new Intl.NumberFormat(locale, { style: "currency", currency: product.currency }).format(product.price)}</div>
      </div>
    ),
    size,
  );
}
```

- Next adds the `og:image` tags (with width, height, alt) automatically for that segment; `twitter-image` works the same way.
- `ImageResponse` uses Satori: **flexbox only**, a subset of CSS, no `display: grid`. 16.2 made it 2–20x faster, added more CSS support and switched the default font to **Geist Sans**.
- The image is generated at build (static params) or on request and cached. Load custom fonts with `fetch`/`readFile` and pass `fonts` for non-Latin scripts.
- **Security**: Next.js **16.3.6 (Sept 22, 2026)** fixed a critical RCE in the Node `ImageResponse` path affecting `>=16.2.0 <16.3.6`. Don't pass untrusted markup to it and keep Next patched.

## 6. `sitemap.ts` and `robots.ts`

```ts
// app/sitemap.ts
import type { MetadataRoute } from "next";
import { locales } from "@/i18n/config";
const base = "https://shop.acme.dev";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const products = await listProductSlugs();       // [{ id, updatedAt }]
  return products.map((p) => ({
    url: `${base}/en/products/${p.id}`,
    lastModified: p.updatedAt,
    alternates: {
      languages: Object.fromEntries(locales.map((l) => [l, `${base}/${l}/products/${p.id}`])),
    },
  }));
}
```

For more than 50,000 URLs, split with `generateSitemaps()` (returns `[{ id: 0 }, { id: 1 }, ...]`); since **Next 16 the `id` passed to `sitemap({ id })` is a Promise**: `const n = Number(await id)`.

```ts
// app/robots.ts
import type { MetadataRoute } from "next";
export default function robots(): MetadataRoute.Robots {
  const isProd = process.env.APP_ENV === "production";
  return {
    rules: isProd
      ? [{ userAgent: "*", allow: "/", disallow: ["/account/", "/cart", "/api/"] }]
      : [{ userAgent: "*", disallow: "/" }],       // keep previews and staging out of the index
    sitemap: "https://shop.acme.dev/sitemap.xml",
  };
}
```

`robots.txt` controls **crawling**, not indexing. For pages that must not be indexed (search results, filters, account pages) use `robots: { index: false }` metadata (`noindex`), and don't also block them in robots.txt, or crawlers never see the `noindex`.

## 7. JSON-LD structured data

Render it as a script in the page (Server Component). Escape `<` so user content can't close the script tag.

```tsx
// app/[locale]/products/[id]/page.tsx (excerpt)
import type { Product, WithContext } from "schema-dts";   // optional typing

const jsonLd: WithContext<Product> = {
  "@context": "https://schema.org",
  "@type": "Product",
  name: product.name,
  image: product.images,
  sku: product.id,
  offers: {
    "@type": "Offer",
    price: product.price.toFixed(2),
    priceCurrency: product.currency,
    availability: product.inStock ? "https://schema.org/InStock" : "https://schema.org/OutOfStock",
    url: `https://shop.acme.dev/${locale}/products/${product.id}`,
  },
  aggregateRating: product.reviewCount
    ? { "@type": "AggregateRating", ratingValue: product.rating, reviewCount: product.reviewCount }
    : undefined,
};

return (
  <section>
    <script
      type="application/ld+json"
      dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd).replace(/</g, "\\u003c") }}
    />
    {/* ...product UI */}
  </section>
);
```

Validate with Google's Rich Results Test and the Schema.org validator. Structured data must match visible content (price, availability), or search engines can ignore it or apply manual actions.

## 8. Canonical and hreflang rules

- Every page gets a **self-referencing canonical** (absolute after `metadataBase` resolution). Filter/sort/UTM variants canonicalize to the clean URL.
- `hreflang` alternates must be **reciprocal** (each language version lists all others and itself) and include **`x-default`** for the fallback/selector page.
- Use the same URLs in `alternates.languages`, the sitemap `alternates`, and internal links. Mismatches are the most common hreflang error.
- Don't canonicalize translated pages to the default language: that tells search engines the translations are duplicates.
- Locale redirects: redirect `/` → `/de` for users, but make sure every localized URL is directly reachable (no redirect loops based on `Accept-Language`; Googlebot usually crawls without it).

## 9. SEO checklist for a Next.js app

| Area | Check |
|---|---|
| Rendering | Content is server-rendered (Server Components, static or streamed); not only after client fetches |
| Status codes | `notFound()` returns 404; moved content uses `permanentRedirect()` (308) |
| Metadata | Unique `title` and `description` per page; `title.template` in the root layout |
| Canonical/hreflang | Self canonical, reciprocal alternates, `x-default` |
| Discovery | `sitemap.ts` with `lastModified`, `robots.ts` per environment |
| Rich results | JSON-LD for Product, BreadcrumbList, Organization, Article |
| Performance | Core Web Vitals pass at p75 ([Next.js Performance Optimization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Performance%20Optimization)) |
| Social | `opengraph-image` per template; HTML-limited bots get blocking metadata |
| Environments | Previews/staging `noindex` + robots disallow; production only indexed |

## 10. Pitfalls

<Warning>
**Common pitfalls**

- Importing the whole dictionary into a Client Component: all translations ship as JS. Pass strings or scoped namespaces.
- Reading `cookies()`/`headers()` in the root layout for the locale: the whole app becomes dynamic. Use the `[locale]` param / root params.
- Proxy matcher that also catches `/_next/static`, images, `sitemap.xml` or `robots.txt` (broken assets or redirect loops).
- `generateMetadata` doing a second uncached DB call per request (wrap in `cache()` or `'use cache'`).
- Deep `openGraph` fields defined in a layout silently disappearing when a page defines `openGraph` (objects replace, not merge).
- Missing `x-default`, non-reciprocal hreflang, or hreflang URLs that redirect.
- Blocking a page in robots.txt and expecting `noindex` to work.
- Forgetting `lang` (and `dir`) on `<html>`: hurts accessibility and SEO.
- Upgrading to 16 and not awaiting `params` in `opengraph-image`/`icon`, or `id` in `sitemap`.
</Warning>

## Senior perspective

- **i18n is a product and content-ops problem**: who translates, how strings flow (TMS like Crowdin/Lokalise/Phrase), how you detect missing keys in CI (typed messages, a check that every locale has every key), and how long untranslated fallbacks may live.
- **Bundle and cache impact**: translations in Server Components cost nothing on the client; each namespace passed to `NextIntlClientProvider` costs bytes. With Cache Components, include the locale in cache keys (root params do this naturally) so German users never get English cached HTML.
- **URL design is forever**: pick prefix strategy, slugs and trailing-slash policy once; changing later means redirects and lost ranking.
- **Measure SEO like performance**: Search Console coverage and hreflang reports, CrUX, crawl stats; alert on spikes of 404/5xx for bots.
- **Metadata and bots**: understand streaming metadata. If a partner crawler (a marketplace feed, an internal search) isn't JS-capable, add it to `htmlLimitedBots`.
- **Security**: JSON-LD escaping, validating locale params before using them in file paths (`import(\`./$\{locale\}.json\`)` must only see allow-listed locales), and patching `ImageResponse` (Sept 2026 RCE).

<Info>
**🆕 2025–2026**

- **15.2 (2025-02-26)**: streaming metadata; `htmlLimitedBots` controls which bots get blocking metadata.
- **next-intl 4.0 (2025-03-12)**: strictly typed `Locale` + `hasLocale`, typed ICU arguments, session locale cookie by default (GDPR), ESM-only, provider inherits server messages. **`useExtracted` (2025-11-07)**. **Ahead-of-time compilation** (next-intl 4.8, Jan 2026, experimental). Latest on npm in late Sept 2026: **4.14.x**; no v5 yet.
- **16.0 (2025-10-21)**: `middleware.ts` → `proxy.ts` (next-intl's middleware goes in `proxy.ts`); `params` for `opengraph-image`/`twitter-image`/`icon`/`apple-icon` and `id` for `sitemap` are Promises; `generateMetadata` follows Cache Components rules.
- **16.2 (2026-03-18)**: `ImageResponse` 2–20x faster, more CSS, default font Geist Sans; multiple icon formats (`icon.png` + `icon.svg`) handled automatically.
- **16.3 (2026-08-03)**: **root params** (`next/root-params`) available by default (earlier versions needed `experimental.rootParams`; Server Components only for now); next-intl's docs switch from `setRequestLocale` to root params (next-intl blog, 2026-08-04).
- **16.3.6 (2026-09-22)**: critical RCE fix in Node `ImageResponse` (`>=16.2.0 <16.3.6`).
</Info>

## Interview questions

<Accordion title="How do you implement i18n routing in the App Router?">
Put all routes under a `[locale]` root segment whose layout renders `<html lang>`, generate static params for every locale, negotiate the locale in `proxy.ts` (cookie, then `Accept-Language`, then default) and redirect unprefixed URLs, and load dictionaries on the server with `server-only`. Libraries like next-intl package this with ICU messages and locale-aware navigation.
</Accordion>

<Accordion title="How do you avoid shipping all translations to the client?">
Translate in Server Components (`getTranslations`/`getDictionary`) so only rendered text ships. For Client Components pass translated strings as props or give `NextIntlClientProvider` only the namespaces they need.
</Accordion>

<Accordion title="What does `generateMetadata` do and how does it relate to the page's data fetching?">
It returns a `Metadata` object per request or prerender, using params and data. Metadata merges from the root layout down. Identical `fetch` calls are memoized with the page's; wrap DB calls in React `cache()` or `'use cache'` to avoid double work.
</Accordion>

<Accordion title="What is streaming metadata and does it hurt SEO?">
Since 15.2, Next sends the UI before `generateMetadata` finishes and appends the tags when ready. Googlebot executes JS and reads them; HTML-limited bots (detected by User-Agent, configurable with `htmlLimitedBots`) still get blocking metadata in `<head>`. Prerendered pages include metadata in the initial HTML anyway.
</Accordion>

<Accordion title="How do you set canonical and hreflang in Next.js?">
Use `alternates.canonical` and `alternates.languages` in `metadata`/`generateMetadata` (resolved against `metadataBase`), including `x-default`, and mirror the same URLs in `sitemap.ts` `alternates.languages`. Alternates must be reciprocal and point at non-redirecting URLs.
</Accordion>

<Accordion title="How do you generate dynamic Open Graph images?">
Add `opengraph-image.tsx` in the route segment and return an `ImageResponse` from `next/og` with `size`, `contentType` and `alt` exports. Next wires the meta tags automatically. In Next 16, `params` is a Promise. Keep Next patched: 16.3.6 fixed an `ImageResponse` RCE.
</Accordion>

<Accordion title="How do you add structured data to a Next.js page?">
Render a `<script type="application/ld+json">` in the Server Component with `JSON.stringify(data)` escaped (`<` → `<`), matching visible content; validate with the Rich Results Test.
</Accordion>

<Accordion title="Why would staging pages show up in Google, and how do you prevent it?">
Because staging is publicly reachable and indexable. Return a disallow-all `robots.ts` and `robots: { index: false }` metadata outside production, protect previews with auth, and set canonical URLs to production only on production.
</Accordion>

## Related
- [Next.js Index](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Index)
- [Next.js App Router Fundamentals](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20App%20Router%20Fundamentals)
- [Next.js Rendering & Caching](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Rendering%20%26%20Caching)
- [Next.js Project Structure & Architecture](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Project%20Structure%20%26%20Architecture)
- [Next.js Error Handling, Loading & Not-Found](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Error%20Handling%2C%20Loading%20%26%20Not-Found)
- [Next.js Performance Optimization](/Software%20Engineering/Frontend/07%20Next.js/Next.js%20Performance%20Optimization)
- [Next.js 15 to 16.3](/Software%20Engineering/Frontend/07%20Next.js/Next.js%2015%20to%2016.3)
- [Rendering Patterns - CSR, SSR, SSG, ISR, Streaming & Islands](/Software%20Engineering/Frontend/09%20Architecture/Rendering%20Patterns%20-%20CSR%2C%20SSR%2C%20SSG%2C%20ISR%2C%20Streaming%20%26%20Islands)
- [Accessibility Essentials](/Software%20Engineering/Frontend/12%20Accessibility/Accessibility%20Essentials)
- [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types)
- [Frontend Home](/Software%20Engineering/Frontend/Frontend%20Home)

## Sources
- Next.js docs: [Internationalization guide](https://nextjs.org/docs/app/guides/internationalization), [generateMetadata](https://nextjs.org/docs/app/api-reference/functions/generate-metadata), [Metadata file conventions](https://nextjs.org/docs/app/api-reference/file-conventions/metadata), [opengraph-image](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/opengraph-image), [ImageResponse](https://nextjs.org/docs/app/api-reference/functions/image-response), [sitemap](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/sitemap), [generateSitemaps](https://nextjs.org/docs/app/api-reference/functions/generate-sitemaps), [robots](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/robots), [JSON-LD guide](https://nextjs.org/docs/app/guides/json-ld), [htmlLimitedBots](https://nextjs.org/docs/app/api-reference/config/next-config-js/htmlLimitedBots), [next/root-params](https://nextjs.org/docs/app/api-reference/functions/next-root-params), [proxy.js](https://nextjs.org/docs/app/api-reference/file-conventions/proxy), [Upgrading to 16](https://nextjs.org/docs/app/guides/upgrading/version-16)
- Next.js blog: [15.2](https://nextjs.org/blog/next-15-2), [16.2](https://nextjs.org/blog/next-16-2), [16.3](https://nextjs.org/blog/next-16-3), [Security update Sept 22, 2026](https://nextjs.org/blog/nextjs-security-update-september-22-2026)
- next-intl: [Routing setup](https://next-intl.dev/docs/routing/setup), [App Router with i18n routing](https://next-intl.dev/docs/getting-started/app-router/with-i18n-routing), [Blog](https://next-intl.dev/blog) (4.0, useExtracted, AOT compilation, root params), [npm](https://www.npmjs.com/package/next-intl)
- Google Search Central: [Localized versions (hreflang)](https://developers.google.com/search/docs/specialty/international/localized-versions), [Canonicalization](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls), [Structured data intro](https://developers.google.com/search/docs/appearance/structured-data/intro-structured-data), [robots.txt vs noindex](https://developers.google.com/search/docs/crawling-indexing/block-indexing)
