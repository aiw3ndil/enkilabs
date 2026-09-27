---
layout: default
comments: false
---

<div class="hero-section">
  <div class="hero-badge">
    <span class="pulse-dot"></span>
    <span>Built for Rubyists, Indie Hackers &amp; Builders</span>
  </div>
  <h1 class="hero-title">Independent Developer Lab for <span class="highlight-orange">Ruby &amp; Modern Web</span></h1>
  <p class="hero-description">
    Enkilabs is an independent platform hosting the personal development projects, production systems, and technical writings of <strong>Albert Oliva</strong>.
  </p>
  <div class="hero-actions">
    <a href="/blog/" class="btn btn-primary">Explore the Blog</a>
    <a href="https://www.enkihost.com" target="_blank" rel="noopener" class="btn btn-secondary">Enkihost Platform ↗</a>
  </div>
</div>

<h2 class="section-title">Featured Projects</h2>
<p class="section-subtitle">Production platforms, mailer infrastructure, and open web initiatives.</p>

<div class="projects-container">

  <div class="project-card">
    <div class="project-item">
      <div class="project-image">
        <img src="/assets/images/enkihost-logo.png" alt="Enkihost.com">
      </div>
      <div class="project-info">
        <div class="project-header-row">
          <h3><a href="https://www.enkihost.com" target="_blank" rel="noopener">Enkihost.com ↗</a></h3>
          <span class="project-badge"><span class="project-badge-dot"></span>Live in Production</span>
        </div>
        <p>Production-ready hosting infrastructure designed specifically for modern Ruby applications with zero-downtime deployments and isolated environments.</p>
        <ul>
          <li>Automated provisioning of PostgreSQL and Redis with instant connection URL injection.</li>
          <li>High-performance Linux isolation powered by <strong>cgroups</strong> for strict resource quotas.</li>
          <li>First-class support and automated Git pipelines for <strong>Ruby on Rails</strong>, <strong>Sinatra</strong>, and <strong>Jekyll</strong>.</li>
        </ul>
        <div class="project-tags">
          <span class="hashtag">#RubyHosting</span>
          <span class="hashtag">#PostgreSQL</span>
          <span class="hashtag">#Redis</span>
          <span class="hashtag">#ZeroDowntime</span>
          <span class="hashtag">#Traefik</span>
        </div>
      </div>
    </div>
  </div>

  <div class="project-card">
    <div class="project-item">
      <div class="project-image">
        <img src="/assets/images/enkimail-logo.png" alt="Enkimail.com">
      </div>
      <div class="project-info">
        <div class="project-header-row">
          <h3><a href="https://www.enkimail.com" target="_blank" rel="noopener">Enkimail.com ↗</a></h3>
          <span class="project-badge"><span class="project-badge-dot"></span>Transactional Email</span>
        </div>
        <p>High-deliverability queue processor and transactional mail pipeline built for simplicity, high throughput, and strict inbox placement.</p>
        <ul>
          <li>Dedicated outbound Postfix daemon paired with OpenDKIM, SPF, DMARC, and rDNS validation.</li>
          <li>Sub-5ms local loopback dispatching with background retry queues and bounce handling.</li>
        </ul>
        <div class="project-tags">
          <span class="hashtag">#EmailMarketing</span>
          <span class="hashtag">#Deliverability</span>
          <span class="hashtag">#Postfix</span>
          <span class="hashtag">#OpenDKIM</span>
          <span class="hashtag">#Docker</span>
        </div>
      </div>
    </div>
  </div>

  <div class="project-card">
    <div class="project-item">
      <div class="project-image">
        <img src="/assets/images/jombo-logo.png" alt="Jombo.fi">
      </div>
      <div class="project-info">
        <div class="project-header-row">
          <h3><a href="https://www.jombo.fi" target="_blank" rel="noopener">Jombo.fi ↗</a></h3>
          <span class="project-badge"><span class="project-badge-dot"></span>Mobility</span>
        </div>
        <p>Ethical carpooling platform without commissions, facilitating direct community transportation.</p>
        <ul>
          <li>Ride-sharing platform to connect drivers and passengers without intermediary fees.</li>
          <li>Architecture based on Ruby API and Next.js frontend hosted on Vercel.</li>
        </ul>
        <div class="project-tags">
          <span class="hashtag">#Carpooling</span>
          <span class="hashtag">#RubyAPI</span>
          <span class="hashtag">#Nextjs</span>
          <span class="hashtag">#Vercel</span>
          <span class="hashtag">#Mobility</span>
        </div>
      </div>
    </div>
  </div>

  <div class="project-card">
    <div class="project-item">
      <div class="project-image">
        <img src="/assets/images/truek-logo.png" alt="Truek.xyz">
      </div>
      <div class="project-info">
        <div class="project-header-row">
          <h3><a href="https://www.truek.xyz" target="_blank" rel="noopener">Truek.xyz ↗</a></h3>
          <span class="project-badge"><span class="project-badge-dot"></span>Community</span>
        </div>
        <p>Modern object exchange and bartering platform built with Ruby and Next.js.</p>
        <ul>
          <li>Implementation of exchange logic, user profiles, and advanced search systems.</li>
          <li>Agile development through Vibe Coding and automated deployment with Coolify.</li>
        </ul>
        <div class="project-tags">
          <span class="hashtag">#ObjectExchange</span>
          <span class="hashtag">#RubyAPI</span>
          <span class="hashtag">#Nextjs</span>
          <span class="hashtag">#Coolify</span>
          <span class="hashtag">#VibeCoding</span>
        </div>
      </div>
    </div>
  </div>

</div>

<div class="about-card">
  <h2>About Albert Oliva</h2>
  <p>
    Software developer sharing my life in Finland, technology experiments, and reflections on infrastructure, language learning, and personal growth.
  </p>
  <div style="margin-top: 18px; display: flex; flex-wrap: wrap; gap: 20px; align-items: center;">
    <div><strong>Email:</strong> <a href="mailto:albert.oliva@protonmail.com">albert.oliva@protonmail.com</a></div>
    <div style="display: flex; gap: 14px;">
      <a href="https://github.com/{{ site.github_username }}" target="_blank" rel="noopener">GitHub</a>
      <a href="https://twitter.com/{{ site.twitter_username }}" target="_blank" rel="noopener">Twitter</a>
      <a href="https://albertoliva.xyz" target="_blank" rel="noopener">albertoliva.xyz ↗</a>
    </div>
  </div>
</div>
