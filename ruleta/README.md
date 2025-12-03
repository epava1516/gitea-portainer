# Ruleta - Aplicación Next.js

Aplicación web personalizada construida con Next.js.

## 📋 Descripción

Este stack despliega una aplicación Next.js con:
- Dos rutas de acceso: subdominio y path
- Integración con Traefik para HTTPS
- Soporte para múltiples dominios
- Configuración opcional de Authentik (SSO)

## 🚀 Despliegue

### Prerequisitos

1. **Red Docker**: Asegúrate de que la red `proxy` existe
2. **Registros DNS**: Configura los registros A para tus dominios
3. **Imagen Docker**: La aplicación debe estar construida como imagen Docker

### Desde Portainer

1. Ve a **Stacks** → **Add stack**
2. Nombre: `ruleta`
3. Selecciona **Repository** o **Git repository**
4. Configura:
   - Repository URL: `<tu-repositorio>`
   - Repository reference: `main`
   - Compose path: `ruleta/docker-compose.yml`
5. Carga el archivo de variables de entorno: `ruleta/stack.env`
6. Haz clic en **Deploy the stack**

### Variables de Entorno

Edita el archivo `stack.env`:

```env
# Imagen de la aplicación (construida previamente)
RULETA_IMAGE=tu-usuario/ruleta:latest

# Entorno de Node.js
RULETA_NODE_ENV=production
RULETA_NEXT_TELEMETRY_DISABLED=1

# Puerto interno de Next.js
RULETA_APP_PORT=3000

# Dominios
# Opción 1: Subdominio dedicado
RULETA_SUBDOMAIN=ruleta.tudominio.com

# Opción 2: Path en dominio principal
RULETA_MAIN_DOMAIN=tudominio.com

# Traefik
TRAEFIK_DOCKER_NETWORK=proxy
TRAEFIK_ENTRYPOINT_SECURE=websecure
TRAEFIK_CERTRESOLVER=letsencrypt

# Si usas Supabase, descomenta y configura:
# NEXT_PUBLIC_SUPABASE_URL=https://tu-proyecto.supabase.co
# NEXT_PUBLIC_SUPABASE_ANON_KEY=tu-clave-publica
# SUPABASE_SERVICE_ROLE_KEY=tu-clave-privada
```

## ⚙️ Rutas de Acceso

Esta aplicación tiene dos formas de acceso configuradas:

### 1. Subdominio Dedicado

Accede mediante: `https://ruleta.tudominio.com`

- URL completa del subdominio
- Next.js recibe las peticiones en el path raíz `/`
- No requiere configuración especial en Next.js

### 2. Path en Dominio Principal

Accede mediante: `https://tudominio.com/ruleta`

- Path dentro del dominio principal
- Traefik elimina el prefijo `/ruleta` antes de enviar a Next.js
- Next.js recibe las peticiones como si estuvieran en `/`

> **Nota**: Puedes usar ambas rutas simultáneamente o deshabilitar una editando el `docker-compose.yml`.

## 🔧 Configuración de Next.js

### Configurar basePath (si usas path)

Si quieres que Next.js sea consciente del path `/ruleta`, edita `next.config.js`:

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  basePath: '/ruleta',
  assetPrefix: '/ruleta',
  // ... otras opciones
}

module.exports = nextConfig
```

> **Nota**: Con la configuración actual (middleware de stripprefix en Traefik), esto NO es necesario. Traefik se encarga de eliminar el prefijo.

### Variables de Entorno en Next.js

Para exponer variables al cliente (navegador):

```javascript
// next.config.js
module.exports = {
  env: {
    CUSTOM_VAR: process.env.CUSTOM_VAR,
  },
  // o usa NEXT_PUBLIC_ prefix
}
```

En `stack.env`:
```env
NEXT_PUBLIC_API_URL=https://api.tudominio.com
CUSTOM_VAR=valor-personalizado
```

## 🔒 Proteger con Authentik (Opcional)

Por defecto, la aplicación NO está protegida con SSO. Para protegerla:

### Opción 1: Proteger Todo

Edita el `docker-compose.yml` y descomenta:

```yaml
labels:
  # Para subdominio
  traefik.http.routers.ruleta-sub.middlewares: "authentik@docker"
  
  # Para path (requiere cadena de middlewares)
  traefik.http.routers.ruleta-path.middlewares: "authentik@docker,ruleta-strip@docker"
```

### Opción 2: Proteger Solo Ciertas Rutas

Crea routers adicionales en Traefik:

```yaml
# Router para rutas públicas
traefik.http.routers.ruleta-public.rule: "Host(`ruleta.tudominio.com`) && PathPrefix(`/api`, `/public`)"
traefik.http.routers.ruleta-public.priority: "20"

# Router para rutas protegidas
traefik.http.routers.ruleta-private.rule: "Host(`ruleta.tudominio.com`) && PathPrefix(`/admin`)"
traefik.http.routers.ruleta-private.middlewares: "authentik@docker"
traefik.http.routers.ruleta-private.priority: "30"
```

## 🏗️ Construir la Imagen Docker

### Dockerfile de Ejemplo

```dockerfile
# Build stage
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine AS runner

WORKDIR /app
ENV NODE_ENV=production

# Copiar archivos necesarios
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

EXPOSE 3000
CMD ["node", "server.js"]
```

### next.config.js para Standalone

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone',
  // ... otras opciones
}

module.exports = nextConfig
```

### Construir y Publicar

```bash
# Construir
docker build -t tu-usuario/ruleta:latest .

# Probar localmente
docker run -p 3000:3000 tu-usuario/ruleta:latest

# Publicar en Docker Hub
docker push tu-usuario/ruleta:latest

# O publicar en GitHub Container Registry
docker tag tu-usuario/ruleta:latest ghcr.io/tu-usuario/ruleta:latest
docker push ghcr.io/tu-usuario/ruleta:latest
```

## 🔧 Configuración Avanzada

### Integración con Supabase

Si tu aplicación usa Supabase:

1. Crea un proyecto en [supabase.com](https://supabase.com)
2. Obtén las credenciales:
   - **URL del proyecto**: Settings → API → Project URL
   - **Anon key**: Settings → API → anon public
   - **Service role key**: Settings → API → service_role (secreto)

3. Añade a `stack.env`:
   ```env
   NEXT_PUBLIC_SUPABASE_URL=https://tu-proyecto.supabase.co
   NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbG...
   SUPABASE_SERVICE_ROLE_KEY=eyJhbG...
   ```

4. En tu código Next.js:
   ```javascript
   import { createClient } from '@supabase/supabase-js'
   
   const supabase = createClient(
     process.env.NEXT_PUBLIC_SUPABASE_URL,
     process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY
   )
   ```

### Variables de Entorno Sensibles

Para secrets (API keys, tokens):

1. No las incluyas en `stack.env` si el repositorio es público
2. Configúralas manualmente en Portainer:
   - Edita el stack
   - Ve a la pestaña **Environment variables**
   - Añade variables adicionales

3. O usa Docker secrets:
   ```yaml
   services:
     app:
       secrets:
         - api_key
   
   secrets:
     api_key:
       external: true
   ```

### Volúmenes para Datos Persistentes

Si necesitas persistir datos (uploads, cache):

```yaml
services:
  app:
    volumes:
      - ${RULETA_DATA_PATH}:/app/data:Z
      - ${RULETA_UPLOADS_PATH}:/app/public/uploads:Z
```

Añade a `stack.env`:
```env
RULETA_DATA_PATH=/opt/ruleta/data
RULETA_UPLOADS_PATH=/opt/ruleta/uploads
```

## 🛠️ Troubleshooting

### La aplicación no inicia

1. Verifica los logs:
   ```bash
   docker logs ruleta-app
   ```

2. Verifica que la imagen existe:
   ```bash
   docker images | grep ruleta
   ```

3. Verifica variables de entorno:
   ```bash
   docker inspect ruleta-app | grep -A 20 Env
   ```

### No puedo acceder por dominio

1. Verifica DNS:
   ```bash
   nslookup ruleta.tudominio.com
   ```

2. Verifica Traefik:
   ```bash
   docker logs traefik | grep ruleta
   ```

3. Verifica labels de Traefik:
   ```bash
   docker inspect ruleta-app | grep -A 30 Labels
   ```

### Error 502 Bad Gateway

1. Verifica que el puerto interno es correcto:
   - Next.js por defecto usa puerto 3000
   - Verifica `RULETA_APP_PORT=3000`

2. Verifica que la aplicación está escuchando:
   ```bash
   docker exec ruleta-app netstat -tulpn
   ```

3. Verifica conectividad desde Traefik:
   ```bash
   docker exec traefik ping ruleta-app
   ```

### Las rutas no funcionan con basePath

Si configuraste `basePath` en Next.js:

1. **Opción A**: Elimina el middleware `stripprefix` de Traefik
2. **Opción B**: Elimina `basePath` de Next.js (Traefik se encarga)

### Recursos estáticos (CSS/JS) no cargan

Verifica:
1. El `assetPrefix` en `next.config.js` (debe coincidir con basePath)
2. Los logs del navegador (F12 → Console)
3. Las rutas en el inspector de red (F12 → Network)

### Variables de entorno no disponibles

Recuerda:
- Solo variables con prefijo `NEXT_PUBLIC_` están disponibles en el navegador
- Variables sin prefijo solo están disponibles en el servidor (API routes, getServerSideProps)

## 📚 Recursos Adicionales

- [Documentación de Next.js](https://nextjs.org/docs)
- [Deploying Next.js](https://nextjs.org/docs/deployment)
- [Next.js con Docker](https://github.com/vercel/next.js/tree/canary/examples/with-docker)
- [Traefik con Next.js](https://doc.traefik.io/traefik/routing/routers/)

## 🔄 Actualizaciones

### Actualizar la Aplicación

1. Construye una nueva versión de la imagen:
   ```bash
   docker build -t tu-usuario/ruleta:v2.0 .
   docker push tu-usuario/ruleta:v2.0
   ```

2. Actualiza en `stack.env`:
   ```env
   RULETA_IMAGE=tu-usuario/ruleta:v2.0
   ```

3. Actualiza el stack en Portainer

### Rolling Updates

Para actualizaciones sin downtime, usa múltiples réplicas (requiere Docker Swarm o Kubernetes).

### Rollback

Si algo sale mal:

1. Vuelve a la versión anterior en `stack.env`
2. Actualiza el stack en Portainer

## 📊 Monitoreo

### Logs de la Aplicación

```bash
# Logs en tiempo real
docker logs -f ruleta-app

# Últimas 100 líneas
docker logs --tail 100 ruleta-app

# Logs con timestamps
docker logs -t ruleta-app
```

### Logs de Next.js

Next.js registra en stdout/stderr, accesibles con `docker logs`.

Para logs estructurados, considera usar:
- [Pino](https://github.com/pinojs/pino)
- [Winston](https://github.com/winstonjs/winston)

### Métricas

Para monitorear la aplicación:
1. Implementa un endpoint `/api/health`
2. Usa herramientas como Prometheus + Grafana
3. O servicios de APM como New Relic, Datadog

## 🎯 Casos de Uso

Este stack es ideal para:
- Aplicaciones Next.js personalizadas
- Landing pages
- Dashboards administrativos
- Aplicaciones full-stack con API routes
- JAMstack con SSR/SSG

## 💡 Tips

- Usa `output: 'standalone'` en Next.js para imágenes Docker más pequeñas
- Implementa caching con Redis para mejor rendimiento
- Usa ISR (Incremental Static Regeneration) para contenido dinámico
- Implementa rate limiting en API routes
- Usa CDN para assets estáticos (imágenes, CSS, JS)
