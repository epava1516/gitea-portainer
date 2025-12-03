# Authentik - Sistema de Autenticación SSO

Authentik es un sistema de autenticación y autorización de código abierto que proporciona Single Sign-On (SSO) para tus aplicaciones.

## 📋 Descripción

Este stack despliega Authentik configurado para funcionar con Traefik mediante **Forward Authentication**, protegiendo tus servicios con autenticación centralizada.

## 🚀 Despliegue

### Desde Portainer

1. Ve a **Stacks** → **Add stack**
2. Nombre: `authentik`
3. Selecciona **Repository** o **Git repository**
4. Configura:
   - Repository URL: `<tu-repositorio>`
   - Repository reference: `main`
   - Compose path: `authentik/docker-compose.yml`
5. Carga el archivo de variables de entorno: `authentik/stack.env`
6. O añade manualmente las variables de entorno necesarias
7. Haz clic en **Deploy the stack**

### Variables de Entorno Importantes

Edita el archivo `stack.env` con tus valores:

```env
# Secretos (genera valores aleatorios seguros)
AUTHENTIK_SECRET_KEY=tu-clave-secreta-aleatoria
AUTHENTIK_POSTGRESQL_PASSWORD=tu-password-db-aleatorio

# Dominio de Authentik
AUTHENTIK_DOMAIN=auth.tudominio.com

# Email y SMTP (opcional pero recomendado)
AUTHENTIK_EMAIL__HOST=smtp.tudominio.com
AUTHENTIK_EMAIL__PORT=587
AUTHENTIK_EMAIL__USERNAME=noreply@tudominio.com
AUTHENTIK_EMAIL__PASSWORD=tu-password-email
AUTHENTIK_EMAIL__FROM=noreply@tudominio.com
```

## ⚙️ Configuración Post-Instalación

Una vez desplegado Authentik, accede a su interfaz web en `https://auth.tudominio.com` y completa la configuración inicial.

### 1. Acceso Inicial

- Usuario por defecto: `akadmin`
- Contraseña: La que configures en el primer acceso
- Cambia la contraseña inmediatamente

### 2. Crear Applications (Aplicaciones)

Para cada servicio que quieras proteger (ej: Portainer, Traefik Dashboard, Gitea, etc.):

1. Ve a **Applications** → **Create**
2. Completa el formulario:
   - **Name**: `Portainer` (nombre descriptivo)
   - **Slug**: `portainer` (identificador único en minúsculas)
   - **Provider**: (lo crearás en el siguiente paso - déjalo vacío por ahora)
   - **Policy engine mode**: `any` (permite acceso si alguna política coincide)
   - **UI settings**: Configura el icono y apariencia (opcional)
3. Haz clic en **Create**

### 3. Crear Providers de tipo Forward Auth

Los providers conectan tus aplicaciones con Authentik. Para Forward Auth con Traefik:

1. Ve a **Applications** → Tu aplicación → **Provider** → **Create**
2. Selecciona tipo: **Proxy Provider**
3. Completa el formulario:
   - **Name**: `Portainer Provider`
   - **Authorization flow**: Selecciona `default-provider-authorization-implicit-consent` (créalo si no existe - ver paso 4)
   - **Type**: **Forward auth (single application)**
   - **External host**: `https://portainer.tudominio.com` (URL completa de tu servicio)
   - **Internal host**: `http://portainer:9000` (opcional - URL interna si Authentik debe hacer reverse proxy)
   - **Internal host SSL validation**: Desactivado (si usas HTTP interno)
4. **Advanced settings**:
   - **Token validity**: `hours=24` (duración de la sesión)
   - **Cookie domain**: `.tudominio.com` (permite SSO entre subdominios)
5. Haz clic en **Create**
6. Vuelve a la aplicación y selecciona el provider que acabas de crear

### 4. Crear Authorization Flow (Implicit)

El authorization flow de tipo **implicit** permite autorización sin pantalla de consentimiento explícita:

#### Opción A: Usar el flow por defecto

Authentik suele incluir un flow llamado `default-provider-authorization-implicit-consent`. Si existe, úsalo directamente en tus providers.

#### Opción B: Crear un flow personalizado

Si necesitas crear uno nuevo:

1. Ve a **Flows & Stages** → **Flows** → **Create**
2. Completa el formulario:
   - **Name**: `Authorization Flow Implicit`
   - **Title**: `Redirecting to application`
   - **Slug**: `default-provider-authorization-implicit-consent`
   - **Designation**: **Authorization**
   - **Authentication**: `Require authentication` (el usuario debe estar autenticado)
   - **Behavior settings**: 
     - **Layout**: `content_left` o el que prefieras
3. Haz clic en **Create**

#### Añadir Stages al Flow

1. Abre el flow que acabas de crear
2. Ve a la pestaña **Stage Bindings**
3. Añade el stage de consentimiento (opcional):
   - Haz clic en **Bind existing stage**
   - Selecciona `default-provider-authorization-consent` (o crea uno nuevo)
   - **Order**: 10
   - **Evaluate on plan**: Activado
   - **Re-evaluate policies**: Activado

> **Nota**: Para implicit consent, puedes crear un flow sin stages de consentimiento, lo que permite autorización automática.

#### Crear Stage de Consent (si no existe)

Si necesitas crear el stage de consentimiento:

1. Ve a **Flows & Stages** → **Stages** → **Create**
2. Tipo: **Consent Stage**
3. Completa:
   - **Name**: `default-provider-authorization-consent`
   - **Mode**: `always_require` o `hidden` (para implicit)
   - **Consent expire in**: `weeks=4` (duración del consentimiento)
4. Haz clic en **Create**

### 5. Configurar Outpost

Los **outposts** son los componentes que ejecutan los providers y procesan las peticiones de autenticación:

1. Ve a **Outposts** → **Outposts** → **Create**
2. Completa el formulario:
   - **Name**: `authentik-embedded-outpost` (o el nombre que prefieras)
   - **Type**: **Proxy**
   - **Integration**: Déjalo vacío (el outpost embedded usa la integración por defecto)
3. **Applications**: Selecciona todas las aplicaciones que creaste (Portainer, Traefik, etc.)
4. **Configuration**:
   ```yaml
   authentik_host: https://auth.tudominio.com
   authentik_host_insecure: false
   log_level: info
   object_naming_template: ak-outpost-%(name)s
   docker_network: proxy
   docker_labels:
     traefik.enable: "true"
   ```
5. Haz clic en **Create**

#### Verificar Estado del Outpost

1. Ve a **Outposts** → **Outposts**
2. Verifica que tu outpost aparece en estado **Healthy** (saludable)
3. Si aparece como **Unhealthy**:
   - Verifica los logs: `docker logs authentik-worker`
   - Asegúrate de que el contenedor puede comunicarse con el servidor de Authentik
   - Verifica la red Docker `proxy`

### 6. Configurar Middleware en Traefik

Para que Traefik use Authentik como forward auth, necesitas configurar el middleware.

#### Opción A: Middleware en el stack de Authentik

Añade las siguientes labels al servicio `authentik-server` en el `docker-compose.yml`:

```yaml
labels:
  # Middleware de Forward Auth
  traefik.http.middlewares.authentik.forwardauth.address: "http://authentik-server:9000/outpost.goauthentik.io/auth/traefik"
  traefik.http.middlewares.authentik.forwardauth.trustForwardHeader: "true"
  traefik.http.middlewares.authentik.forwardauth.authResponseHeaders: "X-authentik-username,X-authentik-groups,X-authentik-email,X-authentik-name,X-authentik-uid"
```

#### Opción B: Middleware en archivo de configuración de Traefik

En el archivo de configuración dinámica de Traefik (`dynamic.yml`):

```yaml
http:
  middlewares:
    authentik:
      forwardAuth:
        address: "http://authentik-server:9000/outpost.goauthentik.io/auth/traefik"
        trustForwardHeader: true
        authResponseHeaders:
          - "X-authentik-username"
          - "X-authentik-groups"
          - "X-authentik-email"
          - "X-authentik-name"
          - "X-authentik-uid"
```

### 7. Proteger Servicios con Authentik

Una vez configurado el middleware, añade la label a los servicios que quieras proteger:

```yaml
labels:
  traefik.http.routers.portainer.middlewares: "authentik@docker"
```

O si definiste el middleware en archivo:

```yaml
labels:
  traefik.http.routers.portainer.middlewares: "authentik@file"
```

## 🔐 Gestión de Usuarios y Grupos

### Crear Usuarios

1. Ve a **Directory** → **Users** → **Create**
2. Completa los datos del usuario
3. Asigna grupos si es necesario
4. Haz clic en **Create**

### Crear Grupos

1. Ve a **Directory** → **Groups** → **Create**
2. Nombre del grupo (ej: `admins`, `users`)
3. Añade usuarios al grupo
4. Haz clic en **Create**

### Crear Políticas de Acceso

Para controlar quién puede acceder a cada aplicación:

1. Ve a **Applications** → Tu aplicación → **Policy / Group / User Bindings**
2. Haz clic en **Bind existing policy/group/user**
3. Selecciona un grupo (ej: `admins`)
4. **Order**: 0 (menor número = mayor prioridad)
5. Haz clic en **Create**

## 🔧 Configuración Avanzada

### Configurar Múltiples Dominios (SSO entre subdominios)

En la configuración del provider:
- **Cookie domain**: `.tudominio.com` (con el punto inicial)
- Esto permite que la sesión se comparta entre todos los subdominios

### Configurar Email (SMTP)

Para notificaciones y recuperación de contraseñas:

1. Ve a **System** → **Settings** → **Email**
2. Configura los parámetros SMTP (o usa las variables de entorno en `stack.env`)

### Configurar Autenticación de Dos Factores (2FA)

1. Ve a **Flows & Stages** → **Stages**
2. Crea stages de tipo **Authenticator Validation Stage** (TOTP, WebAuthn, etc.)
3. Añade estos stages a tu authentication flow

### Integración con Proveedores Externos (OAuth, SAML)

1. Ve a **System** → **Providers** → **Create**
2. Selecciona el tipo (OAuth2, SAML, etc.)
3. Configura según el proveedor externo (Google, GitHub, etc.)

## 🛠️ Troubleshooting

### El outpost aparece como Unhealthy

```bash
# Ver logs del worker
docker logs authentik-worker

# Ver logs del servidor
docker logs authentik-server

# Verificar conectividad de red
docker exec authentik-server ping authentik-postgresql
docker exec authentik-server ping authentik-redis
```

### Los servicios no están protegidos

1. Verifica que el middleware de Traefik esté correctamente configurado
2. Comprueba que las labels en el servicio están correctas
3. Verifica los logs de Traefik: `docker logs traefik`
4. Asegúrate de que el provider está asociado a la aplicación
5. Verifica que el outpost esté en estado **Healthy**

### Error "Invalid redirect_uri"

1. Verifica que el **External host** en el provider coincide exactamente con la URL del servicio
2. Asegúrate de incluir el protocolo (`https://`)
3. No incluyas barras finales

### Sesiones no persisten / Se desconecta constantemente

1. Verifica que el **Cookie domain** esté configurado correctamente (`.tudominio.com`)
2. Comprueba que Redis esté funcionando: `docker logs authentik-redis`
3. Aumenta el **Token validity** en el provider

### No puedo acceder al panel de Authentik

1. Accede directamente por IP: `http://IP-servidor:9000`
2. Verifica los logs: `docker logs authentik-server`
3. Comprueba que el dominio DNS está configurado correctamente
4. Verifica que Traefik está redirigiendo correctamente

## 📚 Recursos Adicionales

- [Documentación oficial de Authentik](https://goauthentik.io/docs/)
- [Authentik con Traefik](https://goauthentik.io/docs/providers/proxy/forward_auth)
- [Configuración de Flows](https://goauthentik.io/docs/flow/)
- [Configuración de Outposts](https://goauthentik.io/docs/outposts/)

## 🔒 Seguridad

- **Cambia inmediatamente** el password por defecto de `akadmin`
- Genera valores **aleatorios** para `AUTHENTIK_SECRET_KEY` y `AUTHENTIK_POSTGRESQL_PASSWORD`
- Habilita **2FA** para usuarios administradores
- Realiza **backups regulares** de la base de datos PostgreSQL
- Mantén Authentik **actualizado** a la última versión

## 💾 Backups

Para hacer backup de la configuración de Authentik:

```bash
# Backup de la base de datos PostgreSQL
docker exec authentik-postgresql pg_dump -U authentik authentik > authentik_backup.sql

# Backup de Redis (sesiones)
docker exec authentik-redis redis-cli --rdb /data/dump.rdb
```

## 🔄 Actualizaciones

1. En Portainer, ve a tu stack de Authentik
2. Haz clic en **Editor**
3. Actualiza las versiones de las imágenes si es necesario
4. Haz clic en **Update the stack**
5. O ejecuta: `docker compose pull && docker compose up -d`
