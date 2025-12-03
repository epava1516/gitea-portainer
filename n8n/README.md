# n8n - Plataforma de Automatización

n8n es una plataforma de automatización de código abierto que permite conectar diferentes servicios y crear flujos de trabajo (workflows).

## 📋 Descripción

Este stack despliega n8n con:
- Base de datos PostgreSQL para persistencia
- Integración con Traefik para acceso HTTPS
- Webhooks públicos sin autenticación
- UI protegida con Authentik (SSO)
- Cifrado de credenciales

## 🚀 Despliegue

### Prerequisitos

1. **Red Docker**: Asegúrate de que la red `proxy` existe
2. **Registro DNS**: Configura el registro A para tu dominio n8n
3. **Clave de cifrado**: Genera una clave segura para cifrar credenciales

### Desde Portainer

1. Ve a **Stacks** → **Add stack**
2. Nombre: `n8n`
3. Selecciona **Repository** o **Git repository**
4. Configura:
   - Repository URL: `<tu-repositorio>`
   - Repository reference: `main`
   - Compose path: `n8n/docker-compose.yml`
5. Carga el archivo de variables de entorno: `n8n/stack.env`
6. Haz clic en **Deploy the stack**

### Variables de Entorno Importantes

Edita el archivo `stack.env`:

```env
# Base de datos PostgreSQL
POSTGRES_PASSWORD=tu-password-seguro-aleatorio
N8N_DB_PASSWORD=tu-password-seguro-aleatorio

# Dominio y URLs
N8N_DOMAIN=n8n.tudominio.com
N8N_HOST=n8n.tudominio.com
N8N_WEBHOOK_URL=https://n8n.tudominio.com/

# Cifrado de credenciales (IMPORTANTE: genera una clave única y guárdala)
N8N_ENCRYPTION_KEY=tu-clave-de-cifrado-aleatoria-muy-larga

# Zona horaria
N8N_TIMEZONE=Europe/Madrid
```

> **⚠️ CRÍTICO**: La `N8N_ENCRYPTION_KEY` es **fundamental**. Si la pierdes, perderás acceso a todas las credenciales guardadas. Haz backup de esta clave.

### Generar Clave de Cifrado

```bash
openssl rand -hex 32
```

## ⚙️ Configuración Post-Instalación

### 1. Primer Acceso

1. Accede a `https://n8n.tudominio.com`
2. Si Authentik está configurado, serás redirigido al login de SSO
3. Completa la configuración inicial de n8n:
   - Email del propietario
   - Nombre de la instancia (opcional)
   - Preferencias de uso

### 2. Crear tu Primer Workflow

1. Haz clic en **Create new workflow**
2. Añade un nodo trigger (ej: **Webhook**, **Schedule**, **Manual**)
3. Añade nodos de acción (ej: **HTTP Request**, **Gmail**, **Slack**)
4. Conecta los nodos
5. Haz clic en **Execute workflow** para probar
6. Activa el workflow

### 3. Configurar Credenciales

Para servicios externos (Gmail, Slack, etc.):

1. Haz clic en un nodo que requiera credenciales
2. Haz clic en **Create New Credential**
3. Completa los datos de autenticación
4. Las credenciales se cifran automáticamente con `N8N_ENCRYPTION_KEY`

## 🔧 Uso de Webhooks

### Webhooks Públicos

Los webhooks NO están protegidos por Authentik, permitiendo que servicios externos los activen:

- URL de producción: `https://n8n.tudominio.com/webhook/tu-id-webhook`
- URL de test: `https://n8n.tudominio.com/webhook-test/tu-id-webhook`

### Ejemplo de Webhook

1. Crea un workflow con un nodo **Webhook**
2. Configura:
   - **HTTP Method**: `POST` (o el que necesites)
   - **Path**: `mi-webhook` (se generará automáticamente)
3. Activa el workflow
4. Copia la URL del webhook
5. Pruébalo:
   ```bash
   curl -X POST https://n8n.tudominio.com/webhook/mi-webhook \
     -H "Content-Type: application/json" \
     -d '{"mensaje": "Hola desde webhook"}'
   ```

### Webhook con Autenticación

Para proteger webhooks, añade autenticación en el propio workflow:

1. Añade un nodo **IF** después del webhook
2. Verifica un token o firma en los headers:
   ```javascript
   {{ $node["Webhook"].json.headers.authorization }} === "Bearer mi-token-secreto"
   ```

## 🔧 Configuración Avanzada

### Variables de Entorno

n8n soporta muchas variables de entorno. Algunas útiles:

```env
# Ejecución
N8N_DEFAULT_BINARY_DATA_MODE=filesystem
EXECUTIONS_DATA_SAVE_ON_ERROR=all
EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=true

# Timeout
EXECUTIONS_TIMEOUT=300
EXECUTIONS_TIMEOUT_MAX=3600

# Comunidad
N8N_TEMPLATES_ENABLED=true
N8N_TEMPLATES_HOST=https://api.n8n.io/api/

# Logs
N8N_LOG_LEVEL=info
N8N_LOG_OUTPUT=console
```

### Integración con Servicios Externos

#### Gmail
1. Crea credenciales OAuth2 en Google Cloud Console
2. En n8n, añade credencial de tipo **Gmail OAuth2**
3. Completa Client ID y Client Secret
4. Autoriza la aplicación

#### Slack
1. Crea una Slack App en api.slack.com
2. Añade permisos necesarios (chat:write, etc.)
3. En n8n, añade credencial de tipo **Slack OAuth2**
4. Autoriza la aplicación

#### Webhooks de GitHub
1. En tu repositorio de GitHub, ve a **Settings** → **Webhooks**
2. Añade webhook URL: `https://n8n.tudominio.com/webhook/github`
3. Selecciona eventos (push, pull request, etc.)
4. En n8n, procesa los eventos con nodos **IF** y **Switch**

### Programar Ejecuciones

Usa el nodo **Schedule Trigger**:

```
# Cada hora
0 * * * *

# Cada día a las 9:00
0 9 * * *

# Cada lunes a las 8:30
30 8 * * 1

# Cada 5 minutos
*/5 * * * *
```

## 🛠️ Troubleshooting

### No puedo acceder a n8n

1. Verifica que el DNS apunta correctamente:
   ```bash
   nslookup n8n.tudominio.com
   ```

2. Verifica los logs:
   ```bash
   docker logs n8n
   ```

3. Verifica Traefik:
   ```bash
   docker logs traefik | grep n8n
   ```

### Los webhooks no funcionan

1. Verifica que la URL del webhook es correcta
2. Asegúrate de que el workflow está **activado**
3. Revisa los logs de ejecución en n8n
4. Verifica que la configuración de Traefik permite webhooks sin autenticación
5. Prueba con curl:
   ```bash
   curl -v https://n8n.tudominio.com/webhook/test
   ```

### Error al guardar credenciales

1. Verifica que `N8N_ENCRYPTION_KEY` está configurada
2. No cambies nunca esta clave después del primer uso
3. Revisa los logs: `docker logs n8n`

### La base de datos no conecta

1. Verifica que PostgreSQL está corriendo:
   ```bash
   docker ps | grep n8n-pg
   ```

2. Verifica los logs de PostgreSQL:
   ```bash
   docker logs n8n-pg
   ```

3. Verifica las credenciales en `stack.env`

### Workflows se ejecutan lentamente

1. Aumenta los recursos del contenedor
2. Verifica el timeout en variables de entorno
3. Optimiza el workflow (divide en workflows más pequeños)
4. Revisa los logs de ejecución

### Error "Execution timed out"

Aumenta el timeout en `stack.env`:

```env
EXECUTIONS_TIMEOUT=600
EXECUTIONS_TIMEOUT_MAX=7200
```

## 📚 Recursos Adicionales

- [Documentación oficial de n8n](https://docs.n8n.io/)
- [Workflows de ejemplo](https://n8n.io/workflows/)
- [Integrations](https://n8n.io/integrations/)
- [Forum de la comunidad](https://community.n8n.io/)

## 🎯 Casos de Uso

### Automatización de GitHub
- Notificar en Slack cuando hay un nuevo PR
- Ejecutar tests automáticos
- Crear issues desde emails

### Backup Automático
- Backup de bases de datos a Google Drive
- Notificaciones de éxito/error
- Programar backups diarios

### Monitoreo
- Verificar disponibilidad de sitios web
- Alertas por email/Slack si un servicio está caído
- Recopilar métricas y enviar a InfluxDB

### Integración con CRM
- Sincronizar contactos entre sistemas
- Automatizar seguimiento de leads
- Generar reportes automáticos

### Procesamiento de Datos
- Extraer datos de APIs
- Transformar y limpiar datos
- Cargar en bases de datos

## 🔒 Seguridad

- **Credenciales**: Todas las credenciales se cifran con `N8N_ENCRYPTION_KEY`
- **Webhooks**: Implementa autenticación personalizada en webhooks sensibles
- **UI**: Protegida con Authentik (SSO)
- **Backups**: Haz backup de la clave de cifrado y la base de datos
- **Variables**: No expongas secrets en los workflows; usa credenciales

## 💾 Backups

### Backup de la Base de Datos

```bash
# Backup
docker exec n8n-pg pg_dump -U n8n n8n > n8n_backup_$(date +%Y%m%d).sql

# Restaurar
cat n8n_backup_YYYYMMDD.sql | docker exec -i n8n-pg psql -U n8n n8n
```

### Backup de la Clave de Cifrado

Guarda `N8N_ENCRYPTION_KEY` en un lugar seguro:

```bash
# Extraer del stack.env
grep N8N_ENCRYPTION_KEY stack.env > encryption_key_backup.txt

# Guardar en un gestor de contraseñas o vault
```

### Backup de Workflows

n8n guarda workflows en la base de datos, por lo que el backup de PostgreSQL incluye los workflows.

También puedes exportar workflows manualmente:
1. En n8n, abre un workflow
2. Haz clic en los 3 puntos → **Download**
3. Guarda el archivo JSON

## 🔄 Actualizaciones

1. Haz **backup** de la base de datos y la clave de cifrado
2. Actualiza en `stack.env`:
   ```env
   N8N_IMAGE=n8nio/n8n:latest
   ```
3. Actualiza el stack en Portainer
4. Verifica los logs: `docker logs n8n`
5. Prueba algunos workflows críticos

> **Nota**: n8n se actualiza frecuentemente. Revisa las [release notes](https://github.com/n8n-io/n8n/releases) antes de actualizar.

## 📊 Monitoreo

### Ver Ejecuciones

En n8n:
1. Ve a **Executions**
2. Filtra por estado (éxito, error, corriendo)
3. Revisa logs detallados de cada ejecución

### Logs del Contenedor

```bash
# Logs en tiempo real
docker logs -f n8n

# Últimas 100 líneas
docker logs --tail 100 n8n

# Logs con timestamps
docker logs -t n8n
```

### Métricas

n8n no tiene métricas Prometheus por defecto, pero puedes:
1. Crear un workflow que publique métricas
2. Usar el nodo HTTP Request para enviar a un collector
3. Monitorear logs con herramientas como Loki/Grafana
