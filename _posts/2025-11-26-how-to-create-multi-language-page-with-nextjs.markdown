---
layout: post
title:  "How to Create a Multi-language Page with Next.js"
date:   2025-11-26 18:53:00 +0200
categories: integration
tags: [next.js, react, internationalization, i18n, web]
---
🌍 How to Create a Multi-language Website with Next.js: Complete Guide

Next.js is the best framework for creating modern web applications in React, and one of its greatest advantages is the ease of implementing internationalization (i18n). Whether for a personal website, a SaaS, or an online store, offering multiple languages allows you to reach more users and improve the browsing experience.

In this guide, you will learn how to create a multi-language website with Next.js using next-intl, the most recommended library in 2025 for its simplicity, SEO friendliness, and full compatibility with App Router.

🧠 Why use Next.js for a multilingual website?

Next.js makes everything related to languages easier:

Automatic routes by language (/en, /es, /fi)

Excellent SEO thanks to hreflang

Optimized translation loading

Server-rendered components with the correct language

Strong typing if you use TypeScript

🛠️ Recommended library: next-intl

There are several options, but today the best for App Router is next-intl, which:

✔️ Works with Server Components
✔️ Has support for multi-language layouts
✔️ Does not generate unnecessary bundle
✔️ Is simple to maintain

🚀 Step-by-step implementation in Next.js
1. Install dependencies
npm install next-intl

2. Create the translation structure

In a project with App Router:

```javascript
/app
  /[locale]
    page.js
    layout.js
/locales
  /en
    common.json
  /es
    common.json
```

Example of locales/es/common.json:

```javascript
{
  "welcome": "Bienvenido a mi web en Next.js",
  "description": "Esta página es multiidioma usando next-intl."
}
```

Example of locales/en/common.json:

```javascript
{
  "welcome": "Welcome to my Next.js website",
  "description": "This page is multilingual using next-intl."
}
```

3. Configure the middleware to detect language

Create middleware.js:

```javascript
import createMiddleware from 'next-intl/middleware';

export default createMiddleware({
  locales: ['en', 'es'],
  defaultLocale: 'en'
});

export const config = {
  matcher: ['/((?!_next|.*\\..*).*)']
};
```

This makes / redirect to /en by default or to the browser's language.

4. Create the multi-language layout

```javascript
File: app/[locale]/layout.js
import { NextIntlClientProvider } from 'next-intl';
import { notFound } from 'next/navigation';
import React from 'react';
 
export default async function RootLayout({ children, params }) {
  let messages;
  try {
    messages = (await import(`../../locales/${params.locale}/common.json`)).default;
  } catch (error) {
    notFound();
  }
 
  return (
    <html lang={params.locale}>
      <body>
        <NextIntlClientProvider messages={messages}>
          {children}
        </NextIntlClientProvider>
      </body>
    </html>
  );
}
```

5. Use translations in a component

File: app/[locale]/page.js

```javascript
import { useTranslations } from 'next-intl';

export default function Home() {
  const t = useTranslations();

  return (
    <div>
      <h1>{t("welcome")}</h1>
      <p>{t("description")}</p>
    </div>
  );
}
```

6. Create a language switcher

File: app/components/lang-switcher.js

```javascript
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";

export default function LangSwitcher() {
  const pathname = usePathname();

  const createPath = (locale) => {
    const parts = pathname.split('/');
    parts[1] = locale;
    return parts.join('/');
  };

  return (
    <div style={{ display: "flex", gap: "10px" }}>
      <Link href={createPath("es")}>🇪🇸 Español</Link>
      <Link href={createPath("en")}>🇬🇧 English</Link>
    </div>
  );
}
```

📦 Final result

Upon finishing you will have:

✔️ A Next.js website with multi-language routes (/es, /en)
✔️ Automatic user language detection
✔️ Server-loaded translations
✔️ Fully optimized SEO 
✔️ A functional language switcher

🧩 Conclusion

Next.js makes creating a multi-language website fast and elegant. With next-intl you get:

Clean code

Correct SSR

Perfect routes for SEO

Optimal performance

It is the ideal solution for serious projects in production.