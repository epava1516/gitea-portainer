# Traefik - Reverse Proxy

Traefik es un reverse proxy moderno que gestiona automáticamente certificados SSL con Let's Encrypt y enruta el tráfico a tus servicios.

## 📋 Descripción

Este stack despliega Traefik configurado para:
- Redirección automática HTTP → HTTPS
- Certificados SSL automáticos con Let's Encrypt
- Integración con Docker para descubrimiento automático de servicios
- Dashboard web para monitoreo
- Soporte para configuración dinámica mediante archivos

## 🚀 Despliegue

### Prerequisitos

1. **Red Docker**: Crear la red `proxy`
   ```bash
   docker network create proxy
   ```

2. **Registros DNS**: Configurar los registros A para todos tus dominios apuntando a tu servidor

3. **Directorios**: Crear los directorios necesarios
   ```bash
   sudo mkdir -p /opt/traefik/dynamic
   sudo mkdir -p /opt/traefik/letsencrypt
   ```

### Desde Portainer

1. Ve a **Stacks** → **Add stack**
2. Nombre: `traefik`
3. Selecciona **Repository** o **Git repository**
4. Configura:
   - Repository URL: `<tu-repositorio>`
   - Repository reference: `main`
   - Compose path: `Traefik/docker-compose.yml`
5. Carga el archivo de variables de entorno: `Traefik/stack.env`
6. Haz clic en **Deploy the stack**

### Variables de Entorno

Edita el archivo `stack.env`:

```env
# Imagen de Traefik
TRAEFIK_IMAGE=traefik:v3.2

# Red Docker
TRAEFIK_DOCKER_NETWORK=proxy

# Puertos
TRAEFIK_HTTP_PORT=80
TRAEFIK_HTTPS_PORT=443

# Let's Encrypt
TRAEFIK_ACME_EMAIL=tu-email@ejemplo.com
TRAEFIK_ACME_STORAGE=/letsencrypt/acme.json

# Nivel de log (DEBUG, INFO, WARN, ERROR)
TRAEFIK_LOG_LEVEL=INFO

# Directorios
TRAEFIK_DYNAMIC_DIR=/opt/traefik/dynamic
TRAEFIK_LETSENCRYPT_DIR=/opt/traefik/letsencrypt
```

> **⚠️ Importante**: Cambia `TRAEFIK_ACME_EMAIL` con tu email real.

## ⚙️ Configuración

### Características Principales

- **Redirección automática**: Todo el tráfico HTTP (puerto 80) se redirige automáticamente a HTTPS (puerto 443)
- **Let's Encrypt**: Certificados SSL automáticos mediante HTTP Challenge
- **Docker Provider**: Detecta automáticamente servicios Docker con labels de Traefik
- **File Provider**: Permite configuración dinámica mediante archivos en `/opt/traefik/dynamic`

### Configuración Dinámica

Puedes añadir configuraciones adicionales en `/opt/traefik/dynamic/`. Ejemplo:

**Archivo: `/opt/traefik/dynamic/middlewares.yml`**

```yaml
http:
  middlewares:
    # Middleware de seguridad
    security-headers:
      headers:
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        stsPreload: true
        contentTypeNosniff: true
        frameDeny: true
        
    # Middleware de compresión
    compression:
      compress: {}
      
    # Rate limiting
    rate-limit:
      rateLimit:
        average: 100
        burst: 50
```

### Acceso al Dashboard

El dashboard está deshabilitado por defecto (`api.insecure=false`). Para habilitarlo de forma segura:

**Opción 1: Mediante Traefik con Authentik**

Añade las siguientes labels al servicio de Traefik:

```yaml
labels:
  traefik.http.routers.dashboard.rule: "Host(`traefik.tudominio.com`)"
  traefik.http.routers.dashboard.entrypoints: "websecure"
  traefik.http.routers.dashboard.tls.certresolver: "letsencrypt"
  traefik.http.routers.dashboard.service: "api@internal"
  traefik.http.routers.dashboard.middlewares: "authentik@docker"
```

**Opción 2: Acceso local (inseguro - solo desarrollo)**

Cambia en el `docker-compose.yml`:
```yaml
- "--api.insecure=true"
```

Luego expón el puerto:
```yaml
ports:
  - "8080:8080"
```

Accede a: `http://tu-servidor:8080/dashboard/`

## 🔧 Uso con Servicios

### Ejemplo: Proteger un servicio con Traefik

En el `docker-compose.yml` de tu servicio:

```yaml
services:
  mi-servicio:
    image: mi-imagen:latest
    networks:
      - proxy
    labels:
      # Habilitar Traefik
      traefik.enable: "true"
      traefik.docker.network: "proxy"
      
      # Router
      traefik.http.routers.mi-servicio.rule: "Host(`miservicio.tudominio.com`)"
      traefik.http.routers.mi-servicio.entrypoints: "websecure"
      traefik.http.routers.mi-servicio.tls.certresolver: "letsencrypt"
      
      # Service (puerto interno del contenedor)
      traefik.http.services.mi-servicio.loadbalancer.server.port: "80"
      
      # Middleware (opcional)
      traefik.http.routers.mi-servicio.middlewares: "authentik@docker"

networks:
  proxy:
    external: true
```

### Ejemplo: Servicio con múltiples rutas

```yaml
labels:
  # UI protegida con SSO
  traefik.http.routers.app-ui.rule: "Host(`app.tudominio.com`)"
  traefik.http.routers.app-ui.entrypoints: "websecure"
  traefik.http.routers.app-ui.tls.certresolver: "letsencrypt"
  traefik.http.routers.app-ui.middlewares: "authentik@docker"
  traefik.http.routers.app-ui.priority: "10"
  
  # API pública sin protección
  traefik.http.routers.app-api.rule: "Host(`app.tudominio.com`) && PathPrefix(`/api`)"
  traefik.http.routers.app-api.entrypoints: "websecure"
  traefik.http.routers.app-api.tls.certresolver: "letsencrypt"
  traefik.http.routers.app-api.priority: "20"
```

> **Nota**: Mayor `priority` = mayor prioridad. Las rutas más específicas deben tener mayor prioridad.

## 🛠️ Troubleshooting

### Los certificados SSL no se generan

1. Verifica que los registros DNS están configurados correctamente:
   ```bash
   nslookup tudominio.com
   ```

2. Verifica que los puertos 80 y 443 están abiertos:
   ```bash
   sudo netstat -tulpn | grep -E ':(80|443)'
   ```

3. Revisa los logs de Traefik:
   ```bash
   docker logs traefik
   ```

4. Verifica el archivo de almacenamiento ACME:
   ```bash
   ls -lh /opt/traefik/letsencrypt/acme.json
   # Debe tener permisos 600
   sudo chmod 600 /opt/traefik/letsencrypt/acme.json
   ```

5. Comprueba los límites de Let's Encrypt:
   - Let's Encrypt tiene límite de 5 certificados duplicados por semana
   - Usa staging para pruebas: `--certificatesresolvers.letsencrypt.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory`

### Un servicio no es accesible

1. Verifica que el servicio está en la red `proxy`:
   ```bash
   docker network inspect proxy
   ```

2. Revisa las labels de Traefik en el servicio:
   ```bash
   docker inspect nombre-servicio | grep -A 20 Labels
   ```

3. Verifica los logs de Traefik:
   ```bash
   docker logs traefik | grep nombre-servicio
   ```

4. Comprueba que `traefik.enable: "true"` está configurado

### Error "Gateway Timeout" o "Bad Gateway"

1. Verifica que el puerto del servicio es correcto:
   ```yaml
   traefik.http.services.mi-servicio.loadbalancer.server.port: "PUERTO_CORRECTO"
   ```

2. Verifica que el servicio está corriendo:
   ```bash
   docker ps | grep mi-servicio
   ```

3. Comprueba la conectividad desde Traefik al servicio:
   ```bash
   docker exec traefik ping nombre-servicio
   ```

### Dashboard no accesible

1. Si habilitaste `api.insecure=true`, accede a: `http://IP:8080/dashboard/`
2. Si usas dominio, verifica las labels y que el DNS está configurado
3. Revisa los logs: `docker logs traefik`

## 📚 Recursos Adicionales

- [Documentación oficial de Traefik](https://doc.traefik.io/traefik/)
- [Traefik con Docker](https://doc.traefik.io/traefik/providers/docker/)
- [Let's Encrypt con Traefik](https://doc.traefik.io/traefik/https/acme/)
- [Middlewares de Traefik](https://doc.traefik.io/traefik/middlewares/overview/)

## 🔒 Seguridad

- **Dashboard**: Protege siempre el dashboard con autenticación (Authentik, BasicAuth, etc.)
- **Logs**: Considera cambiar el nivel de log a `WARN` o `ERROR` en producción
- **Rate Limiting**: Implementa rate limiting en servicios públicos
- **Headers de seguridad**: Usa middlewares para añadir headers de seguridad

## 💾 Backups

Realiza backups regulares de:
- `/opt/traefik/letsencrypt/acme.json` (certificados SSL)
- `/opt/traefik/dynamic/` (configuración dinámica)

```bash
# Backup
sudo tar -czf traefik-backup-$(date +%Y%m%d).tar.gz /opt/traefik/

# Restaurar
sudo tar -xzf traefik-backup-YYYYMMDD.tar.gz -C /
```

## 🔄 Actualizaciones

Para actualizar Traefik:

1. En Portainer, ve al stack de Traefik
2. Edita el archivo `stack.env` y cambia la versión:
   ```env
   TRAEFIK_IMAGE=traefik:v3.3
   ```
3. Haz clic en **Update the stack**
4. O desde línea de comandos:
   ```bash
   cd Traefik
   docker compose pull
   docker compose up -d
   ```

> **⚠️ Nota**: Revisa el [changelog de Traefik](https://github.com/traefik/traefik/releases) antes de actualizar versiones mayores.
