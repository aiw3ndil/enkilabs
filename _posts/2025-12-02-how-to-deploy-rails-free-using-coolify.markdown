---
layout: post
title:  "How to Deploy Rails for Free Using Coolify"
date:   2025-12-02 18:18:00 +0200
categories: development rails deployment
tags: [ruby on rails, coolify, deployment, devops, free, open-source]
---
Deploying Rails applications can be expensive if you use traditional platforms. In this article, I will show you how you can **deploy your Rails application completely for free** using **Coolify**, an open-source alternative to Heroku and Netlify.

## 🎯 What is Coolify?

**Coolify** is a self-hosted hosting platform that allows you to easily deploy applications, databases, and services. It's like having your own Heroku but with total control over your infrastructure.

### Main features:
- **Open Source**: 100% open and free code
- **Self-hosted**: Deploy on your own server
- **Multi-application**: Manage multiple projects from one interface
- **Docker-based**: Uses containers for consistent deployment
- **Integrated CI/CD**: Automatic deployment from Git
- **Automatic SSL**: Free HTTPS certificates with Let's Encrypt

## 🛠️ Prerequisites

Before starting, you will need:

1. A VPS server (you can get one for free on Oracle Cloud, Google Cloud, or AWS Free Tier)
2. A Rails application ready to deploy
3. A domain (optional, but recommended)
4. Basic knowledge of terminal and Git

## 📋 Step 1: Configure your Server

First, you need a server with at least:
- 2 GB of RAM
- 1 CPU
- 20 GB of storage
- Ubuntu 22.04 LTS (recommended)

Providers with free tiers:
- **Oracle Cloud**: 2 ARM instances permanently free
- **Google Cloud**: $300 in credits for 90 days
- **AWS**: 12 months of free t2.micro

## 🚀 Step 2: Install Coolify

Connect to your server via SSH and run:

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

This script will automatically install:
- Docker and Docker Compose
- Coolify and its dependencies
- Initial system configuration

The process takes approximately 5-10 minutes.

## 🔐 Step 3: Access Coolify

Once installed, access Coolify at:
```
http://your-server-ip:8000
```

Create your admin account following the initial setup wizard.

## 📦 Step 4: Prepare your Rails Application

Make sure your Rails application has:

### Dockerfile

If you don't have a Dockerfile, create one in the root of your project:

```dockerfile
FROM ruby:3.2

RUN apt-get update -qq && apt-get install -y nodejs postgresql-client

WORKDIR /app

COPY Gemfile Gemfile.lock ./
RUN bundle install

COPY . .

# Precompile assets
RUN RAILS_ENV=production bundle exec rake assets:precompile

EXPOSE 3000

CMD ["rails", "server", "-b", "0.0.0.0"]
```

### Environment Variables

Prepare an `.env.example` file with the necessary variables:

```
DATABASE_URL=postgresql://user:password@postgres:5432/myapp_production
RAILS_MASTER_KEY=your_master_key_here
SECRET_KEY_BASE=your_secret_key_base_here
```

## 🎨 Step 5: Configure the Project in Coolify

1. **Create a new project**: In the Coolify dashboard, click "New Project"

2. **Add a resource**: Select "New Resource" → "Application"

3. **Connect your Git repository**: 
   - Authorize your GitHub/GitLab account
   - Select your Rails application repository

4. **Configure the build**:
   - Build Pack: Docker
   - Dockerfile Path: `./Dockerfile`
   - Branch: `main` (or whichever you use)

5. **Add Database**:
   - In the same project, add a new "Database" resource
   - Select PostgreSQL
   - Coolify will automatically create the database

6. **Environment Variables**:
   - Go to the "Environment Variables" section
   - Add all necessary variables
   - The `DATABASE_URL` is automatically provided by Coolify

## 🌐 Step 6: Configure Domain and SSL

1. **Add domain**:
   - In your application configuration
   - "Domains" section
   - Add your domain (e.g., `yourapp.com`)

2. **Configure DNS**:
   - At your domain provider
   - Create an A record pointing to your server's IP
   - Wait for it to propagate (can take up to 24 hours)

3. **Activate SSL**:
   - Coolify will automatically generate a Let's Encrypt SSL certificate
   - No additional configuration needed

## ⚡ Step 7: Deploy

1. Click the **"Deploy"** button
2. Coolify will:
   - Clone your repository
   - Build the Docker image
   - Run migrations (if configured)
   - Start the application

3. Monitor progress in the logs in real-time

## 🔄 Continuous Deployment

Configure webhooks for automatic deployment:

1. Go to "Webhooks" in your application
2. Copy the webhook URL
3. In GitHub/GitLab:
   - Settings → Webhooks
   - Paste the URL
   - Select "Push" events

Now every time you push to your main branch, Coolify will automatically deploy.

## 🎯 Useful Commands

### Run migrations:
```bash
docker exec -it container_name rails db:migrate
```

### Access Rails console:
```bash
docker exec -it container_name rails console
```

### View logs:
Logs are available directly in the Coolify interface in real-time.

## 💡 Tips and Best Practices

1. **Backups**: Configure automatic backups of your database in Coolify
2. **Monitoring**: Use the metrics section to monitor CPU, RAM, and disk
3. **Health Checks**: Configure health checks to automatically restart if it fails
4. **Resources**: Adjust CPU and memory limits according to your needs
5. **Staging**: Create a staging environment on the same server to test changes

## 🆚 Coolify vs Other Alternatives

| Feature | Coolify | Heroku | Railway | Render |
|---------------|---------|--------|---------|--------|
| Cost | Free (server only) | $7+/mo | $5+/mo | $7+/mo |
| Full Control | ✅ | ❌ | ❌ | ❌ |
| Open Source | ✅ | ❌ | ❌ | ❌ |
| Self-hosted | ✅ | ❌ | ❌ | ❌ |
| Auto-SSL | ✅ | ✅ | ✅ | ✅ |

## 🎉 Conclusion

Coolify is an excellent option for deploying Rails applications at no cost, while maintaining full control over your infrastructure. While it requires a bit more initial setup than commercial PaaS solutions, the cost savings and flexibility make it a very attractive option.

With a free server from Oracle Cloud and Coolify, you can have multiple Rails applications in production without spending a dime.

## 📚 Additional Resources

- [Official Coolify Documentation](https://coolify.io/docs)
- [Coolify GitHub Repository](https://github.com/coollabsio/coolify)
- [Coolify Community on Discord](https://coollabs.io/discord)lify.io/docs)
- [Repositorio GitHub de Coolify](https://github.com/coollabsio/coolify)
- [Comunidad de Coolify en Discord](https://coollabs.io/discord)
