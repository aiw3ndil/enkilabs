---
layout: post
title:  "Cómo Desplegar Rails Gratis Usando Coolify"
date:   2025-12-02 18:18:00 +0200
categories: desarrollo rails deployment
tags: [ruby on rails, coolify, deployment, devops, gratis, open-source]
---
Desplegar aplicaciones Rails puede ser costoso si utilizas plataformas tradicionales. En este artículo, te mostraré cómo puedes **desplegar tu aplicación Rails completamente gratis** utilizando **Coolify**, una alternativa open-source a Heroku y Netlify.

## 🎯 ¿Qué es Coolify?

**Coolify** es una plataforma de hosting self-hosted que te permite desplegar aplicaciones, bases de datos y servicios de forma sencilla. Es como tener tu propio Heroku pero con control total sobre tu infraestructura.

### Características principales:
- **Open Source**: Código 100% abierto y gratuito
- **Self-hosted**: Despliega en tu propio servidor
- **Multi-aplicación**: Gestiona múltiples proyectos desde una interfaz
- **Docker-based**: Utiliza contenedores para un despliegue consistente
- **CI/CD integrado**: Despliegue automático desde Git
- **SSL automático**: Certificados HTTPS gratuitos con Let's Encrypt

## 🛠️ Requisitos Previos

Antes de comenzar, necesitarás:

1. Un servidor VPS (puedes obtener uno gratis en Oracle Cloud, Google Cloud o AWS Free Tier)
2. Una aplicación Rails lista para desplegar
3. Un dominio (opcional, pero recomendado)
4. Conocimientos básicos de terminal y Git

## 📋 Paso 1: Configurar tu Servidor

Primero, necesitas un servidor con al menos:
- 2 GB de RAM
- 1 CPU
- 20 GB de almacenamiento
- Ubuntu 22.04 LTS (recomendado)

Proveedores con niveles gratuitos:
- **Oracle Cloud**: 2 instancias ARM gratuitas permanentemente
- **Google Cloud**: $300 en créditos por 90 días
- **AWS**: 12 meses de t2.micro gratuito

## 🚀 Paso 2: Instalar Coolify

Conéctate a tu servidor via SSH y ejecuta:

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

Este script instalará automáticamente:
- Docker y Docker Compose
- Coolify y sus dependencias
- Configuración inicial del sistema

El proceso toma aproximadamente 5-10 minutos.

## 🔐 Paso 3: Acceder a Coolify

Una vez instalado, accede a Coolify en:
```
http://tu-ip-servidor:8000
```

Crea tu cuenta de administrador siguiendo el asistente de configuración inicial.

## 📦 Paso 4: Preparar tu Aplicación Rails

Asegúrate de que tu aplicación Rails tenga:

### Dockerfile

Si no tienes un Dockerfile, crea uno en la raíz de tu proyecto:

```dockerfile
FROM ruby:3.2

RUN apt-get update -qq && apt-get install -y nodejs postgresql-client

WORKDIR /app

COPY Gemfile Gemfile.lock ./
RUN bundle install

COPY . .

# Precompilar assets
RUN RAILS_ENV=production bundle exec rake assets:precompile

EXPOSE 3000

CMD ["rails", "server", "-b", "0.0.0.0"]
```

### Variables de Entorno

Prepara un archivo `.env.example` con las variables necesarias:

```
DATABASE_URL=postgresql://user:password@postgres:5432/myapp_production
RAILS_MASTER_KEY=tu_master_key_aqui
SECRET_KEY_BASE=tu_secret_key_base_aqui
```

## 🎨 Paso 5: Configurar el Proyecto en Coolify

1. **Crear un nuevo proyecto**: En el dashboard de Coolify, haz clic en "New Project"

2. **Agregar un recurso**: Selecciona "New Resource" → "Application"

3. **Conectar tu repositorio Git**: 
   - Autoriza tu cuenta de GitHub/GitLab
   - Selecciona el repositorio de tu aplicación Rails

4. **Configurar el build**:
   - Build Pack: Docker
   - Dockerfile Path: `./Dockerfile`
   - Branch: `main` (o la que uses)

5. **Agregar Base de Datos**:
   - En el mismo proyecto, añade un nuevo recurso "Database"
   - Selecciona PostgreSQL
   - Coolify creará automáticamente la base de datos

6. **Variables de Entorno**:
   - Ve a la sección "Environment Variables"
   - Añade todas las variables necesarias
   - La `DATABASE_URL` la proporciona automáticamente Coolify

## 🌐 Paso 6: Configurar Dominio y SSL

1. **Agregar dominio**:
   - En la configuración de tu aplicación
   - Sección "Domains"
   - Añade tu dominio (ej: `tuapp.com`)

2. **Configurar DNS**:
   - En tu proveedor de dominios
   - Crea un registro A apuntando a la IP de tu servidor
   - Espera a que se propague (puede tomar hasta 24 horas)

3. **Activar SSL**:
   - Coolify generará automáticamente un certificado SSL de Let's Encrypt
   - No necesitas configuración adicional

## ⚡ Paso 7: Desplegar

1. Haz clic en el botón **"Deploy"**
2. Coolify:
   - Clonará tu repositorio
   - Construirá la imagen Docker
   - Ejecutará las migraciones (si las configuras)
   - Iniciará la aplicación

3. Monitorea el progreso en los logs en tiempo real

## 🔄 Despliegue Continuo

Configura webhooks para despliegue automático:

1. Ve a "Webhooks" en tu aplicación
2. Copia la URL del webhook
3. En GitHub/GitLab:
   - Settings → Webhooks
   - Pega la URL
   - Selecciona eventos "Push"

Ahora cada vez que hagas push a tu rama principal, Coolify desplegará automáticamente.

## 🎯 Comandos Útiles

### Ejecutar migraciones:
```bash
docker exec -it container_name rails db:migrate
```

### Acceder a la consola Rails:
```bash
docker exec -it container_name rails console
```

### Ver logs:
Los logs están disponibles directamente en la interfaz de Coolify en tiempo real.

## 💡 Tips y Mejores Prácticas

1. **Backups**: Configura backups automáticos de tu base de datos en Coolify
2. **Monitoreo**: Usa la sección de métricas para monitorear CPU, RAM y disco
3. **Health Checks**: Configura health checks para reiniciar automáticamente si falla
4. **Recursos**: Ajusta los límites de CPU y memoria según tus necesidades
5. **Staging**: Crea un entorno de staging en el mismo servidor para probar cambios

## 🆚 Coolify vs Otras Alternativas

| Característica | Coolify | Heroku | Railway | Render |
|---------------|---------|--------|---------|--------|
| Costo | Gratis (solo servidor) | $7+/mes | $5+/mes | $7+/mes |
| Control total | ✅ | ❌ | ❌ | ❌ |
| Open Source | ✅ | ❌ | ❌ | ❌ |
| Self-hosted | ✅ | ❌ | ❌ | ❌ |
| Auto-SSL | ✅ | ✅ | ✅ | ✅ |

## 🎉 Conclusión

Coolify es una excelente opción para desplegar aplicaciones Rails sin costo, manteniendo control total sobre tu infraestructura. Si bien requiere un poco más de configuración inicial que las soluciones PaaS comerciales, el ahorro en costos y la flexibilidad lo hacen una opción muy atractiva.

Con un servidor gratuito de Oracle Cloud y Coolify, puedes tener múltiples aplicaciones Rails en producción sin gastar un centavo.

## 📚 Recursos Adicionales

- [Documentación oficial de Coolify](https://coolify.io/docs)
- [Repositorio GitHub de Coolify](https://github.com/coollabsio/coolify)
- [Comunidad de Coolify en Discord](https://coollabs.io/discord)
