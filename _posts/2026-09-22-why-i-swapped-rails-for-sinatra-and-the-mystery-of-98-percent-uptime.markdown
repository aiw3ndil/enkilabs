---
layout: post
title:  "Why I Swapped Rails for Sinatra (And the Mystery of My 98.59% Uptime)"
date:   2026-09-22 18:30:00 +0300
categories: ruby sinatra rails backend devops performance
---

When building web applications in the Ruby world, **Ruby on Rails** is almost universally treated as the default choice. And for good reason: its convention-over-configuration philosophy, battle-tested tooling, and unmatched ecosystem have powered everything from weekend MVPs to multi-billion-dollar tech giants.

However, as [Enkihost](https://www.enkihost.com) and our supporting backend services grew, the overhead of a full Rails stack began to show on smaller cloud virtual machines. Boot times were sluggish, background processes hogged memory, and a significant portion of the framework's built-in baggage sat completely untouched.

That prompted a deliberate architectural decision: **migrating the core API engine to Sinatra**.

The result? Noticeably faster response times and a massive reduction in resource usage. But production engineering always has a plot twist: despite the system running lighter than ever, I am currently chasing down an elusive **98.59% uptime** issue with intermittent drops.

Here is an honest look at the benefits of Sinatra, what the modern lightweight stack looks like, and the ongoing mystery I’m trying to solve.

---

### The Myth: "Without Rails, You Lose the Ruby Magic"

The biggest objection developers raise when considering Sinatra is the fear of reinventing the wheel: *“What about my database migrations? What about model associations? How do I handle background jobs?”*

Here is the secret: **you do not lose ActiveRecord, and you do not lose your favorite gems**.

Sinatra is essentially a thin, expressive routing DSL built directly on Rack. It doesn't force a database layer or an opinionated structure onto your project, but it welcomes whatever tooling you choose to bring. By pairing `sinatra` with `sinatra-activerecord`, you get first-class database migrations, models, relations, validations, and connection pooling—exactly as you would in Rails.

Here is the actual production `Gemfile` currently powering our Sinatra service:

```ruby
source "https://rubygems.org"

ruby "~> 3.3.0"

# Core Sinatra framework
gem "sinatra"
gem "sinatra-contrib"
gem "sinatra-activerecord"
gem "activerecord", "~> 7.1.0"

# Database & Server
gem "pg", "~> 1.5"
gem "puma", ">= 5.0"
gem "rake"

# Security & API Utilities
gem "bcrypt", "~> 3.1.7"
gem "pundit"
gem "rack-cors"
gem "dotenv"
gem "jwt"
gem "json", "~> 2.8"
gem "jsonapi-serializer"

# Background Processing & Network
gem "sidekiq"
gem "faraday-retry"

# Third-party Integrations
gem "octokit", "~> 9.0"
gem "gitlab", "~> 4.19"
gem "aws-sdk-s3", "~> 1.0"
gem "enkimail", "~> 0.1.6"
```

Look at that list:
- **ActiveRecord 7.1**: The exact same battle-tested ORM you know from Rails, with migrations managed via simple `rake db:migrate` tasks.
- **Pundit & BCrypt**: Clean, modular policy-based authorization and secure authentication.
- **Sidekiq**: High-performance asynchronous job queues backed by Redis.
- **Octokit & GitLab**: Direct Git provider automation for deployments.
- **AWS S3**: Object and asset storage.
- **Enkimail**: Native transactional email dispatching.

You give up virtually nothing from the broader Ruby ecosystem. What you strip away is the heavyweight middleware chain, ActionCable, ActionMailbox, ActiveStorage abstractions you may not need, and the memory footprint that comes with loading them all at boot.

---

### The Wins: Snappy Loads and Slender RAM Footprint

The immediate upside of switching to Sinatra was impossible to miss:

1. **Drastically Reduced Memory Usage**: The entire server setup now consumes around **5 GB of RAM** (with 8 GB swap configured as a safety buffer). Under Rails, comparable processes frequently pushed our RAM to its threshold, forcing aggressive restarts or expensive hardware upgrades.
2. **Instant Boot & Reloads**: Cold-booting the application or restarting Puma takes seconds rather than half a minute. This dramatically accelerates local development and shortens deployment downtime.
3. **Faster Request-Response Cycles**: Without dozens of Rails middlewares intercepting every HTTP call, the raw response times on API endpoints dropped visibly. Requests feel immediate.
4. **Architectural Clarity**: In Sinatra, every route, filter, and helper exists because you explicitly wrote it or required it. There is no hidden convention doing magic behind the scenes.

From a pure resource efficiency perspective, the move from Rails to Sinatra has been an unambiguous success.

---

### The Mystery: The 98.59% Uptime Puzzle

Now comes the frustrating part.

With our server running cool, RAM usage stable at ~5 GB, and CPU utilization remaining exceptionally low, monitoring continues to log **intermittent cuts and brief outages**. Over the last tracking period, our uptime has hovered at **98.59%**.

For any production service—especially one providing infrastructure—98.59% is simply not good enough. It translates to hours of cumulative downtime or degraded connectivity over a month.

Here is what makes this issue particularly baffling:
- **No Out-Of-Memory (OOM) Events**: The kernel logs (`dmesg`) show zero OOM killer interventions. The 8 GB swap remains mostly idle.
- **No CPU Spikes**: System monitoring shows no sudden CPU lockups preceding the drops.
- **Puma Worker Health**: Puma is running with sensible worker counts and thread allocations, and the database connection pool isn't showing starvation errors.
- **Ghost Cuts**: The downtime incidents tend to last anywhere from 30 seconds to a couple of minutes before auto-resolving without any manual restart required.

When software logs are clean and internal resource limits aren't being breached, suspicion naturally turns to the underlying infrastructure layer.

---

### Is It OVH? Considering a Host Migration

The current virtual server is hosted on **OVH**. While OVH has historically provided great compute-per-dollar value, troubleshooting sporadic micro-cuts on shared hypervisors can be a nightmare:
- **Hypervisor Steal Time / CPU Noisy Neighbors**: Even if your VM appears idle, virtualization contention on the host node can cause sudden I/O pauses or temporary network stalls.
- **Host-Level Routing Blips**: Intermittent BGP flapping, upstream packet loss, or DDoS mitigation scrubbers kicking in unexpectedly can cause external monitors to report outages while the process itself is technically still running.

Because I have thoroughly optimized the code, slimmed down the framework, and confirmed that memory and CPU are well within safe operating margins, I am seriously contemplating **migrating the service off OVH to an alternative provider** (such as Hetzner or AWS Lightsail/EC2).

Switching hosts will provide a clean baseline: if the 98.59% uptime figure suddenly jumps to 99.9% on a new node with identical software configurations, the culprit was hardware or network instability outside our control. If the drops persist, it points to a deeper application-level bottleneck—perhaps Puma socket backlog drops under concurrent external webhook spikes or an unhandled TCP timeout in one of our third-party integrations (like GitHub or GitLab webhooks).

---

### Conclusion

Sinatra in 2026 remains an underrated superpower for Ruby developers. You don’t have to sacrifice ActiveRecord, Sidekiq, or modern gems to build lightning-fast, lean backends that sip resources instead of devouring them.

But software is only half the battle; reliable hosting is the other half. Over the coming days, I'll be instrumenting deeper network-level observability and testing an alternate cloud environment to get our uptime back where it belongs: 99.9%+.

Have you experienced unexplained micro-outages on OVH cloud instances, or tracked down mysterious Puma downtime while memory was plenty? I’d love to hear your insights.