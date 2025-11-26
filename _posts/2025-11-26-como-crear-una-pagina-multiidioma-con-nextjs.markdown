---
layout: post
title:  "Como crear una pagina multiidioma con react"
date:   2025-11-26 18:53:00 +0200
categories: integracion
---
🌍 Cómo Crear una Página Web Multiidioma con Next.js: Guía Completa

Next.js es el mejor framework para crear aplicaciones web modernas en React, y una de sus mayores ventajas es la facilidad para implementar internacionalización (i18n). Ya sea para una web personal, un SaaS o una tienda online, ofrecer varios idiomas te permite llegar a más usuarios y mejorar la experiencia de navegación.

En esta guía aprenderás a crear una web multiidioma con Next.js usando next-intl, la librería más recomendada en 2025 por su sencillez, SSR amigable y compatibilidad total con App Router.

🧠 ¿Por qué usar Next.js para una web multilingüe?

Next.js facilita todo lo relacionado con idiomas:

Rutas automáticas por idioma (/en, /es, /fi)

Excelente SEO gracias a hreflang

Carga de traducciones optimizada

Componentes renderizados en servidor con el idioma correcto

Tipado fuerte si usas TypeScript

🛠️ Librería recomendada: next-intl

Existen varias opciones, pero hoy la mejor para el App Router es next-intl, que:

✔️ Funciona con Server Components
✔️ Tiene soporte para layouts multiidioma
✔️ No genera bundle innecesario
✔️ Es sencilla de mantener

🚀 Implementación paso a paso en Next.js
1. Instalar dependencias
npm install next-intl

2. Crear la estructura de traducciones

En un proyecto con App Router:

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

Ejemplo de locales/es/common.json:

```javascript
{
  "welcome": "Bienvenido a mi web en Next.js",
  "description": "Esta página es multiidioma usando next-intl."
}
```

Ejemplo de locales/en/common.json:

```javascript
{
  "welcome": "Welcome to my Next.js website",
  "description": "This page is multilingual using next-intl."
}
```

3. Configurar el middleware para detectar idioma

Crea middleware.js:

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

Esto hace que / redirija a /en por defecto o al idioma del navegador.

4. Crear el layout multiidioma

```javascript
Archivo: app/[locale]/layout.js
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

5. Usar traducciones en un componente

Archivo: app/[locale]/page.js

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

6. Crear un selector de idioma

Archivo: app/components/lang-switcher.js

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

📦 Resultado final

Al terminar tendrás:

✔️ Una web Next.js con rutas multiidioma (/es, /en)
✔️ Detección automática del idioma del usuario
✔️ Traducciones cargadas en servidor
✔️ SEO completamente optimizado
✔️ Un selector de idioma funcional

🧩 Conclusión

Next.js hace que crear una web multiidioma sea rápido y elegante. Con next-intl consigues:

Código limpio

SSR correcto

Rutas perfectas para SEO

Rendimiento óptimo

Es la solución ideal para proyectos serios en producción.