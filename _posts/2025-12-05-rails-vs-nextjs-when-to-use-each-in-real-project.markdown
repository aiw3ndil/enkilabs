---
layout: post
title: "Rails vs Next.js: When to use each in a real project"
date: 2025-12-05 20:31:00 -0000
categories: web development
tags: [ruby on rails, next.js, comparison, backend, frontend, react]
---

Choosing between Rails and Next.js is one of the most important decisions when starting a web project. Both frameworks are excellent, but they shine in different contexts. Here I tell you when to use each one based on real experience.

## Ruby on Rails: The all-in-one framework

Rails is perfect when you need:

### 1. Rapid development of MVPs

Rails was designed to maximize developer productivity. With conventions over configuration, you can have an application with authentication, database, and CRUD in minutes.

```ruby
# A complete scaffold in one line
rails generate scaffold Article title:string content:text published:boolean

# Authentication with Devise
rails generate devise:install
rails generate devise User
```

### 2. Applications with complex business logic

If your application has a lot of logic on the backend (billing, inventory, complex workflows), Rails shines:

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

### 3. Small teams or solopreneurs

With Rails, a single person can handle frontend and backend. You don't need to coordinate between multiple repos or specialized teams.

### 4. Traditional applications

If your app is primarily server-rendered with forms, administrative dashboards, or doesn't require extreme frontend interactivity, Rails with Hotwire/Turbo is incredibly efficient.

```erb
<!-- Real-time update without JavaScript -->
<%= turbo_frame_tag "notifications" do %>
  <%= render @notifications %>
<% end %>

<!-- Form with instant validation -->
<%= turbo_frame_tag "new_comment" do %>
  <%= form_with model: @comment do |f| %>
    <%= f.text_area :body %>
    <%= f.submit %>
  <% end %>
<% end %>
```

## Next.js: The king of modern experiences

Next.js is your best option when:

### 1. You need perfect SEO

With App Router and Server Components, Next.js offers SSR and SSG natively:

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

### 2. Highly interactive interfaces

If your app is more like a desktop application (editors, real-time dashboards, collaborative tools):

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

### 3. You have a specialized team

Next.js allows frontend and backend to be completely independent. Your frontend team can work with the best DX while the backend is built in parallel:

```typescript
// app/api/products/route.ts
export async function GET(request: Request) {
  const products = await prisma.product.findMany({
    include: { category: true }
  });
  
  return Response.json(products);
}

// Or connect with any backend
const products = await fetch('https://api.example.com/products');
```

### 4. Global applications with edge computing

With Vercel Edge Functions, you can serve your app from the edge closest to the user:

```typescript
export const runtime = 'edge';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const country = request.headers.get('x-vercel-ip-country');
  
  // Customization by region without latency
  const content = await getLocalizedContent(country);
  
  return Response.json(content);
}
```

## The practical decision

### Use Rails if:
- You are alone or with a small team
- You need to deliver fast
- Business logic is on the server
- You don't need a complex SPA
- Limited budget (cheaper hosting)

### Use Next.js if:
- SEO is critical for your business
- You need a super smooth user experience
- You have a specialized frontend team
- You are going to build a mobile application later (reusing the API)
- You need global performance with CDN/Edge

## The hybrid: Best of both worlds

In my experience, the most powerful combination is:

**Rails as API + Next.js as frontend**

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

This gives you:
- Rails robustness for complex logic
- Next.js flexibility for modern UX
- Separation of concerns
- Independent scalability

## My recommendation

For startups and new projects:
1. **Start with Rails** if you are not sure. It's faster to get to market.
2. **Add Next.js** when you really need a superior frontend experience.
3. **Don't over-engineer**: many projects work perfectly with just Rails or just Next.js.

The best technology is the one that allows you to deliver value to your users faster. Both frameworks are excellent; the key is understanding your specific context.

What framework do you prefer to use? Have you had experiences combining both?prefieres usar? ¿Has tenido experiencias combinando ambos?
