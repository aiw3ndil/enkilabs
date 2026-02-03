---
layout: post
title: "Rails vs Next.js: Cuándo usar cada uno en un proyecto real"
date: 2025-12-05 20:31:00 -0000
categories: desarrollo web
tags: [ruby on rails, next.js, comparación, backend, frontend, react]
---

Elegir entre Rails y Next.js es una de las decisiones más importantes al iniciar un proyecto web. Ambos frameworks son excelentes, pero brillan en contextos diferentes. Aquí te cuento cuándo usar cada uno basándome en experiencia real.

## Ruby on Rails: El framework todo en uno

Rails es perfecto cuando necesitas:

### 1. Desarrollo rápido de MVPs

Rails fue diseñado para maximizar la productividad del desarrollador. Con convenciones sobre configuración, puedes tener una aplicación con autenticación, base de datos y CRUD en minutos.

```ruby
# Un scaffold completo en una línea
rails generate scaffold Article title:string content:text published:boolean

# Autenticación con Devise
rails generate devise:install
rails generate devise User
```

### 2. Aplicaciones con lógica de negocio compleja

Si tu aplicación tiene mucha lógica en el backend (facturación, inventarios, workflows complejos), Rails brilla:

```ruby
class Order < ApplicationRecord
  belongs_to :user
  has_many :order_items
  has_many :products, through: :order_items
  
  validates :status, inclusion: { in: %w[pending paid shipped delivered] }
  
  def calculate_total
    order_items.sum { |item| item.quantity * item.price }
  end
  
  def process_payment!
    transaction do
      update!(status: 'paid', paid_at: Time.current)
      OrderMailer.confirmation(self).deliver_later
      InventoryService.reserve_items(self)
    end
  end
end
```

### 3. Equipos pequeños o solopreneurs

Con Rails, una sola persona puede manejar frontend y backend. No necesitas coordinar entre múltiples repos o equipos especializados.

### 4. Aplicaciones tradicionales

Si tu app es principalmente server-rendered con formularios, dashboards administrativos, o no requiere interactividad extrema en el frontend, Rails con Hotwire/Turbo es increíblemente eficiente.

```erb
<!-- Actualización en tiempo real sin JavaScript -->
<%= turbo_frame_tag "notifications" do %>
  <%= render @notifications %>
<% end %>

<!-- Formulario con validación instantánea -->
<%= turbo_frame_tag "new_comment" do %>
  <%= form_with model: @comment do |f| %>
    <%= f.text_area :body %>
    <%= f.submit %>
  <% end %>
<% end %>
```

## Next.js: El rey de las experiencias modernas

Next.js es tu mejor opción cuando:

### 1. Necesitas SEO perfecto

Con App Router y Server Components, Next.js ofrece SSR y SSG de forma nativa:

```typescript
// app/blog/[slug]/page.tsx
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const post = await getPost(params.slug);
  
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [post.coverImage],
    },
  };
}

export default async function BlogPost({ params }: Props) {
  const post = await getPost(params.slug);
  
  return (
    <article>
      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  );
}
```

### 2. Interfaces altamente interactivas

Si tu app es más parecida a una aplicación de escritorio (editores, dashboards en tiempo real, herramientas colaborativas):

```typescript
'use client';

import { useState, useEffect } from 'react';
import { useRealtimeData } from '@/hooks/useRealtimeData';

export default function LiveDashboard() {
  const { metrics, isConnected } = useRealtimeData();
  const [selectedPeriod, setSelectedPeriod] = useState('day');
  
  return (
    <div className="grid grid-cols-3 gap-4">
      {metrics.map(metric => (
        <MetricCard
          key={metric.id}
          data={metric}
          period={selectedPeriod}
          isLive={isConnected}
        />
      ))}
    </div>
  );
}
```

### 3. Tienes equipo especializado

Next.js permite que frontend y backend sean completamente independientes. Tu equipo de frontend puede trabajar con la mejor DX mientras el backend se construye en paralelo:

```typescript
// app/api/products/route.ts
export async function GET(request: Request) {
  const products = await prisma.product.findMany({
    include: { category: true }
  });
  
  return Response.json(products);
}

// O conecta con cualquier backend
const products = await fetch('https://api.example.com/products');
```

### 4. Aplicaciones globales con edge computing

Con Vercel Edge Functions, puedes servir tu app desde el edge más cercano al usuario:

```typescript
export const runtime = 'edge';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const country = request.headers.get('x-vercel-ip-country');
  
  // Personalización por región sin latencia
  const content = await getLocalizedContent(country);
  
  return Response.json(content);
}
```

## La decisión práctica

### Usa Rails si:
- Estás solo o con un equipo pequeño
- Necesitas entregar rápido
- La lógica de negocio está en el servidor
- No necesitas una SPA compleja
- Presupuesto limitado (hosting más barato)

### Usa Next.js si:
- SEO es crítico para tu negocio
- Necesitas una experiencia de usuario súper fluida
- Tienes equipo frontend especializado
- Vas a construir una aplicación mobile después (reutilizas el API)
- Necesitas performance global con CDN/Edge

## El híbrido: Lo mejor de ambos mundos

En mi experiencia, la combinación más poderosa es:

**Rails como API + Next.js como frontend**

```typescript
// next.config.js
module.exports = {
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: 'https://rails-api.example.com/:path*'
      }
    ];
  }
};
```

Esto te da:
- La robustez de Rails para lógica compleja
- La flexibilidad de Next.js para UX moderna
- Separación de responsabilidades
- Escalabilidad independiente

## Mi recomendación

Para startups y proyectos nuevos:
1. **Empieza con Rails** si no estás seguro. Es más rápido llegar al mercado.
2. **Añade Next.js** cuando realmente necesites una experiencia frontend superior.
3. **No sobre-ingenieres**: muchos proyectos funcionan perfectamente con solo Rails o solo Next.js.

La mejor tecnología es la que te permite enviar valor a tus usuarios más rápido. Ambos frameworks son excelentes; la clave está en entender tu contexto específico.

¿Qué framework prefieres usar? ¿Has tenido experiencias combinando ambos?
