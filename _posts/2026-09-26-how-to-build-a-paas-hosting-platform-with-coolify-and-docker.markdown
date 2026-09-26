---
layout: post
title:  "How We Built a Cloud Hosting Platform on Top of Coolify and Docker: The Enkihost Architecture"
date:   2026-09-26 14:00:00 +0300
categories: devops docker coolify sinatra ruby traefik paas cloud hosting
---

Developers love the simplicity of modern PaaS platforms like Heroku, Render, and Fly.io: you push your git repository, configure a custom domain, add environment variables, and your application goes live with automated SSL certificates in seconds.

When building [Enkihost.com](https://www.enkihost.com){:target="_blank"}—a specialized hosting platform for Ruby on Rails, Sinatra, and Jekyll websites—we faced an architectural dilemma: **should we build an entire cloud orchestration layer from scratch, or could we leverage existing open-source tools?**

That's when we turned to **Coolify**.

Coolify is widely known as a self-hostable open-source Heroku and Netlify alternative. However, out of the box, Coolify is designed as an admin control panel for solo developers and DevOps teams managing internal infrastructure. It is not built to act as a multi-tenant commercial hosting service with user accounts, custom billing, resource tiering, and white-label deployments.

Instead of writing a custom reverse proxy and certificate manager from scratch, we designed **Enkihost** to stand on the shoulders of Coolify, Traefik, and the Docker engine. 

In this article, we’ll take a deep dive into the real-world architecture of our backend engine (`enkihost-sinatra`) and explore how we dynamically generate Docker containers, configure Traefik routing labels, manage custom domains with automated Let's Encrypt certificates, and inject environment variables per deployed website.

---

### High-Level Architecture Overview

Here is how the entire system connects together:

```
 ┌────────────────────────────────────────────────────────┐
 │             Enkihost Dashboard (Next.js)               │
 └───────────────────────────┬────────────────────────────┘
                             │ REST API / WebSockets
                             ▼
 ┌────────────────────────────────────────────────────────┐
 │            Enkihost Engine (Ruby / Sinatra)            │
 │                                                        │
 │  • App Management & User Quotas (CPU/Memory Limits)   │
 │  • Deployment Worker (Sidekiq / Async Jobs)           │
 │  • DockerService (Container Lifecycle & Traefik)      │
 │  • CoolifyService (API Sync & Fallback Orchestration) │
 └─────────────────────┬──────────────────┬───────────────┘
                       │                  │
        Docker Socket  │                  │ Coolify v4 API
        / CLI Control  │                  │ (Optional Sync)
                       ▼                  ▼
 ┌────────────────────────────────────────────────────────┐
 │                     Host Server (VPS)                  │
 │                                                        │
 │  ┌──────────────────────────────────────────────────┐  │
 │  │      Traefik Reverse Proxy (Coolify Gateway)     │  │
 │  │      - Auto-discovers containers on Docker net   │  │
 │  │      - Automated Let's Encrypt TLS generation   │  │
 │  └──────────────────────────┬───────────────────────┘  │
 │                             │ Docker Network: "coolify"│
 │     ┌───────────────────────┼────────────────────┐     │
 │     ▼                       ▼                    ▼     │
 │ ┌───────────────┐   ┌───────────────┐    ┌───────────┐ │
 │ │ enkihost-app- │   │ enkihost-app- │    │ Managed   │ │
 │ │  12-34 (Web)  │   │  15-35 (Web)  │    │ Addons:   │ │
 │ │ (Rails/Sinatra│   │(Jekyll+Nginx) │    │ Postgres/ │ │
 │ │  Container)   │   │  Container)   │    │  Redis    │ │
 │ └───────────────┘   └───────────────┘    └───────────┘ │
 └────────────────────────────────────────────────────────┘
```

---

### 1. The Strategy: Standing on the Shoulders of Coolify & Traefik

Coolify configures a battle-tested infrastructure environment on your VPS:
1. A **Traefik** reverse proxy running in a container listening on ports `80` and `443`.
2. A dedicated Docker bridge network named `coolify`.
3. Traefik configured with Docker provider integration and an automated Let's Encrypt SSL certificate resolver.

Traefik listens to the Docker daemon event socket (`/var/run/docker.sock`). Whenever a container is launched with specific `traefik.*` labels on the `coolify` network, Traefik immediately reads those labels, registers new HTTP/HTTPS routing rules, requests Let's Encrypt SSL certificates, and forwards incoming traffic to the container's private port—**all with zero configuration file reloads and zero proxy downtime.**

By taking advantage of this existing setup, our Sinatra backend only needs to:
- Clone the user's repository (handling private GitHub credentials securely).
- Generate an optimized Dockerfile (if the user repo doesn't provide one).
- Build the Docker image.
- Spawn the container attached to the `coolify` Docker network with the correct **Traefik labels**, **environment variables**, **resource limits**, and **persistent storage mounts**.

---

### 2. Automated Multi-Stage Dockerfile Generation

Not every developer wants to maintain a `Dockerfile`. In Enkihost, users can push standard Rails, Sinatra, or Jekyll repositories, and our `DockerService` generates an optimized build recipe on the fly:

```ruby
def generate_dockerfile
  # If the user already provides a Dockerfile, respect their custom setup
  if File.exist?(File.join(@build_path, "Dockerfile"))
    log("Using existing Dockerfile from repository")
    return
  end

  dockerfile_content = case @app.kind
                       when 'rails'
                         rails_dockerfile
                       when 'sinatra'
                         sinatra_dockerfile
                       when 'jekyll'
                         jekyll_dockerfile
                       end
  
  File.write(File.join(@build_path, 'Dockerfile'), dockerfile_content)
  log("Generated optimized Dockerfile for #{@app.kind}")
end
```

#### Multi-Stage Builds for Static Jekyll Sites

For static sites like **Jekyll**, we don't want a heavy Ruby runtime running in production. Instead, we use a two-stage build: stage one compiles the site using Ruby and Jekyll, and stage two serves the static `_site` directory using an ultra-lightweight Alpine Nginx web server:

```dockerfile
FROM ruby:3.3-slim AS build
RUN apt-get update -qq && apt-get install -y build-essential libyaml-dev
WORKDIR /rails
COPY Gemfile* ./
RUN bundle install
COPY . .
RUN bundle exec jekyll build

FROM nginx:alpine
RUN echo 'server { \
    listen 80; \
    server_name localhost; \
    location / { \
        root /usr/share/nginx/html; \
        index index.html index.htm; \
        try_files $uri $uri/ /index.html; \
    } \
}' > /etc/nginx/conf.d/default.conf

COPY --from=build /rails/_site /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

This reduces the final container size from ~600MB down to under ~25MB, boots in 100 milliseconds, and consumes almost zero RAM.

---

### 3. Dynamic Subdomains, Custom Domains, and Automatic SSL

The real magic happens when provisioning the container. We assign each hosted app:
1. A default platform subdomain: `#{app.subdomain}.enkihost.com`.
2. Any custom vanity domains the user has added to their dashboard (`app.domains.pluck(:fqdn)`).

We format these domains into Traefik routing rules and pass them as container labels:

```ruby
def start_new_container
  image_tag = "enkihost-#{@app.id}-#{@deployment.id}"
  container_name = "enkihost-app-#{@app.id}-#{@deployment.id}"
  
  internal_port = case @app.kind
                  when 'rails' then 3000
                  when 'sinatra' then 4567
                  when 'jekyll' then 80
                  end

  proxy_network = 'coolify'

  # Build domains list
  base_domain = Rails.env.production? ? "enkihost.com" : "localhost"
  default_subdomain = "#{@app.subdomain}.#{base_domain}"
  custom_domains = @app.domains.pluck(:fqdn)
  
  # Format rule for Traefik v3: Host("subdomain.enkihost.com") || Host("custom.com")
  all_domains_rule = ([default_subdomain] + custom_domains)
                     .map { |domain| "Host(\"#{domain}\")" }
                     .join(" || ")

  router_name = "enkihost-app-#{@app.id}-#{@deployment.id}"
  service_name = "enkihost-app-#{@app.id}-#{@deployment.id}"

  labels = [
    "traefik.enable=true",
    "traefik.http.routers.#{router_name}.rule=#{all_domains_rule}",
    "traefik.http.routers.#{router_name}.priority=1000",
    "traefik.http.routers.#{router_name}.service=#{service_name}",
    "traefik.http.routers.#{router_name}.entrypoints=http,https",
    "traefik.http.routers.#{router_name}.tls=true",
    "traefik.http.routers.#{router_name}.tls.certresolver=letsencrypt",
    "traefik.http.services.#{service_name}.loadbalancer.server.port=#{internal_port}",
    "enkihost.app_id=#{@app.id}"
  ]

  # ...
```

#### What happens here?
- **`traefik.http.routers.*.rule`**: Traefik intercepts incoming HTTP requests matching either the assigned Enkihost subdomain or any of the user's custom domains.
- **`traefik.http.routers.*.tls.certresolver=letsencrypt`**: Traefik automatically contacts Let's Encrypt via ACME challenge, verifies the domain, and mounts an SSL certificate without manual intervention.
- **`loadbalancer.server.port`**: Traefik proxies traffic directly to the container's internal listening port over the Docker internal virtual bridge, meaning we never have to bind or expose arbitrary host ports on the VPS!

---

### 4. Dynamic Environment Variables and Addon Provisioning

Each application needs custom environment variables: API keys, secrets, and database connection strings.

In `DockerService`, we assemble these into standard `-e KEY=VALUE` flags:

```ruby
# 1. Custom user-defined environment variables
env_args = @app.environment_variables.map do |ev|
  ["-e", "#{ev.key}=#{ev.value}"]
end.flatten

# 2. System environment defaults
if %w[rails sinatra].include?(@app.kind)
  env_args += ["-e", "RAILS_ENV=production", "-e", "RACK_ENV=production"]
end

# 3. Automatic Addon Injection (PostgreSQL, Redis)
@app.addons.running.each do |addon|
  case addon.kind
  when 'postgresql'
    env_args += ["-e", "DATABASE_URL=#{addon.config['url']}"]
  when 'redis'
    env_args += ["-e", "REDIS_URL=#{addon.config['url']}"]
  end
end
```

When a user toggles a PostgreSQL or Redis addon from their Enkihost dashboard, our system spins up an isolated database container and automatically injects `DATABASE_URL` or `REDIS_URL` into the application container on the next deployment.

---

### 5. Persistent Volumes and Tier-Based Resource Limits

In a multi-tenant hosting platform, you cannot allow a single rogue process to consume 100% of the host server's CPU or memory. You also need to ensure that user-uploaded files (like images in `/rails/storage`) survive redeployments.

We enforce resource quotas using native Docker cgroup controls:

```ruby
cmd_args = [
  docker_bin, "run", "-d", 
  "--name", container_name,
  "--network", proxy_network,
  "--cpus", @app.cpu_limit.to_s,       # e.g., "1.0" or "0.5"
  "--memory", @app.memory_limit.to_s,   # e.g., "512m" or "1g"
  "--restart", "unless-stopped"
]

# Attach Persistent Storage
if @app.storages.any?
  @app.storages.each do |storage|
    cmd_args += ["-v", "#{storage.source}:#{storage.destination}"]
  end
else
  # Default persistent volume for Rails ActiveStorage / file uploads
  default_source = "enkihost-app-#{@app.id}-data"
  default_dest = @app.kind == 'rails' ? "/rails/storage" : "/app/storage"
  cmd_args += ["-v", "#{default_source}:#{default_dest}"]
end

# Add Traefik labels, Environment Variables, and Docker Image
labels.each { |label| cmd_args += ["--label", label] }
cmd_args += env_args
cmd_args << image_tag

system_cmd_array(cmd_args)
```

By assigning a persistent named Docker volume (`enkihost-app-#{app.id}-data`), user assets remain safe across code updates, rebuilds, and restarts.

---

### 6. Zero-Downtime Container Swapping

When deploying a new version of an app, you cannot immediately kill the existing container—if the new build crashes during boot, your user's site will go down.

We implement a safe swap pattern:
1. **Boot new container:** Start `enkihost-app-#{app.id}-#{new_deployment.id}`.
{% raw %}
2. **Health poll loop:** Poll `docker inspect --format '{{.State.Running}}'` until the container is confirmed healthy and active.
3. **Graceful cleanup:** Only once the new container is healthy do we search for older containers belonging to this app (or squatting on the same domain rules) and remove them:

```ruby
def wait_for_readiness(container_name)
  log("Waiting for container #{container_name} to be ready...")
  max_retries = 30
  retries = 0

  loop do
    is_running = `#{docker_bin} inspect -f '{{.State.Running}}' #{container_name}`.strip == 'true' rescue false
    
    if is_running
      log("Container is up and running.")
      break
    end

    retries += 1
    if retries >= max_retries
      # Abort: clean up failed container and do NOT touch the old running container
      system("#{docker_bin} stop #{container_name}") rescue nil
      system("#{docker_bin} rm #{container_name}") rescue nil
      raise "Container readiness timeout"
    end

    sleep 1
  end
end

def stop_old_container
  current_container = "enkihost-app-#{@app.id}-#{@deployment.id}"

  # Locate previous containers by App ID label
  old_ids = `#{docker_bin} ps -a --filter "label=enkihost.app_id=#{@app.id}" --format "{{.ID}}"`.split("\n")
  
  # Ensure we NEVER kill the new active deployment
  current_id = `#{docker_bin} inspect --format '{{.Id}}' #{current_container}`.strip rescue nil
  old_ids.reject! { |id| id == current_id || current_id&.start_with?(id) }

  old_ids.each do |id|
    log("Pruning superseded container #{id}...")
    system("#{docker_bin} stop #{id}")
    system("#{docker_bin} rm #{id}")
  end
end
```
{% endraw %}

Because Traefik routes traffic by container IP and label priority, the moment the new container joins the network and the old one is stopped, Traefik routes 100% of new requests to the new container without dropping a connection.

---

### 7. Alternative Path: Coolify v4 REST API Orchestration

While direct Docker daemon control gives us millisecond-level responsiveness for container orchestration, we also built a full integration with Coolify's official **v4 REST API** via `CoolifyService`.

If you prefer to let Coolify handle the build queue directly rather than building on the host Docker daemon, you can interact with its API programmatically:

```ruby
class CoolifyService
  def initialize
    @url = "#{ENV['COOLIFY_URL']}/api/v1"
    @conn = Faraday.new(url: @url) do |f|
      f.request :json
      f.response :json
      f.headers['Authorization'] = "Bearer #{ENV['COOLIFY_TOKEN']}"
    end
  end

  def create_and_deploy(app)
    # 1. Create Application
    res = @conn.post('applications/public', {
      project_uuid: ENV['COOLIFY_PROJECT_UUID'],
      server_uuid: ENV['COOLIFY_SERVER_UUID'],
      environment_name: 'production',
      git_repository: clean_repo_url(app),
      git_branch: app.branch || 'main',
      build_pack: 'nixpacks'
    })
    coolify_uuid = res.body['uuid']

    # 2. Sync Domains
    domains = ([app.subdomain + ".enkihost.com"] + app.domains.pluck(:fqdn)).join(',')
    @conn.patch("applications/#{coolify_uuid}", {
      domains: domains,
      ports_exposes: "3000"
    })

    # 3. Sync Environment Variables
    app.environment_variables.each do |env|
      @conn.post("applications/#{coolify_uuid}/envs", {
        key: env.key,
        value: env.value.to_s,
        is_literal: true
      })
    end

    # 4. Trigger Deployment
    @conn.post("deploy", { uuid: coolify_uuid, force: true })
  end
end
```

This hybrid flexibility allows us to deploy either via lightweight Docker commands directly on the server or delegate complex multi-service stacks to Coolify’s native build engine.

---

### 8. Hard-Won Production Gotchas & Lessons Learned

Building a hosting platform taught us several valuable lessons that aren't in the documentation:

1. **Traefik v3 Rule Syntax Changes:**  
   In Traefik v2, multiple host rules could be combined with commas (`Host(`a.com`, `b.com`)`). In Traefik v3, passing multiple arguments to `Host()` throws a router parse error. You must explicitly chain them with boolean OR operators: `Host("a.com") || Host("b.com")`.
2. **Never Expose Host Ports:**  
   Avoid mapping random host ports like `-p 10042:3000`. By placing all containers on the shared `coolify` Docker network, Traefik can talk to containers directly on their internal IP address and internal listening port. This completely eliminates port collisions on the host machine.
3. **Token Scrubbing in Build Logs:**  
   When cloning private repositories with tokens (e.g. `https://oauth2:TOKEN@github.com/...`), you must sanitize your deployment logs. If Git throws an authentication error or a command echo executes, personal access tokens could leak into public deployment logs. Always pipe build output through a tokenizer sanitizer before broadcasting to WebSockets.
4. **Volume Naming Hygiene:**  
   Always namespace your Docker volumes (e.g. `enkihost-app-#{app_id}-data`). When an application is deleted by the user, run a comprehensive resource cleanup job (`CleanupAppResourcesJob`) to remove orphan images, volumes, and stopped containers to prevent disk leaks.

---

### Conclusion

By combining the robustness of **Coolify** and **Traefik** with a lightweight **Sinatra** orchestration backend, we were able to launch [Enkihost](https://www.enkihost.com){:target="_blank"} with the features developers expect from modern cloud hosting—instant deployments, automated Let's Encrypt certificates, custom domains, and dynamic environment variables—without reinventing the wheel or running bloated infrastructure.

Are you running a self-hosted PaaS or building developer tools on top of Docker and Coolify? Let us know your thoughts, or try deploying your next Ruby or Jekyll site on [Enkihost.com](https://www.enkihost.com){:target="_blank"}!
