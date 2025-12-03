# Gitea - Servidor Git Autoalojado

Gitea es un servidor Git ligero y autoalojado, similar a GitHub o GitLab.

## 📋 Descripción

Este stack despliega Gitea con:
- Base de datos PostgreSQL
- Servidor SSH para operaciones Git
- Gitea Actions (CI/CD similar a GitHub Actions)
- Act Runner para ejecutar pipelines
- Integración con Traefik y Authentik

## 🚀 Despliegue

### Prerequisitos

1. **Red Docker**: Asegúrate de que la red `proxy` existe
2. **Registro DNS**: Configura el registro A para tu dominio Gitea
3. **Puerto SSH**: El puerto SSH debe estar disponible (por defecto 222)

### Desde Portainer

1. Ve a **Stacks** → **Add stack**
2. Nombre: `gitea`
3. Selecciona **Repository** o **Git repository**
4. Configura:
   - Repository URL: `<tu-repositorio>`
   - Repository reference: `main`
   - Compose path: `gitea/docker-compose.yml`
5. Carga el archivo de variables de entorno: `gitea/stack.env`
6. Haz clic en **Deploy the stack**

### Variables de Entorno Importantes

Edita el archivo `stack.env`:

```env
# Imágenes
GITEA_IMAGE=gitea/gitea:latest
GITEA_POSTGRES_IMAGE=postgres:16
GITEA_RUNNER_IMAGE=gitea/act_runner:latest

# Base de datos
GITEA_DB_PASSWORD=tu-password-seguro-aleatorio

# Dominios
GITEA_DOMAIN=git.tudominio.com
GITEA_ROOT_URL=https://git.tudominio.com

# SSH
GITEA_SSH_DOMAIN=git.tudominio.com
GITEA_SSH_PORT=222

# Gitea Actions Runner
GITEA_RUNNER_REGISTRATION_TOKEN=tu-token-de-registro
```

## ⚙️ Configuración Post-Instalación

### 1. Primer Acceso

1. Accede a `https://git.tudominio.com`
2. Completa la configuración inicial:
   - Base de datos: Ya está configurada (PostgreSQL)
   - Configuración del servidor: Usa los valores predefinidos
   - **Importante**: Crea la cuenta de administrador en este paso
3. Haz clic en **Install Gitea**

### 2. Configurar Gitea Actions

Para habilitar CI/CD con Gitea Actions:

#### Paso 1: Generar Token de Registro

1. Accede a Gitea como administrador
2. Ve a **Site Administration** → **Actions** → **Runners**
3. Haz clic en **Create new Runner**
4. Copia el **Registration Token**

#### Paso 2: Actualizar el Stack

1. Edita el archivo `stack.env`
2. Añade el token:
   ```env
   GITEA_RUNNER_REGISTRATION_TOKEN=tu-token-copiado
   ```
3. Actualiza el stack en Portainer

#### Paso 3: Verificar el Runner

1. En Gitea, ve a **Site Administration** → **Actions** → **Runners**
2. Deberías ver el runner registrado y activo
3. Nombre del runner: `act-runner-docker` (o el configurado en `GITEA_RUNNER_NAME`)

### 3. Configurar SSH

Para usar Git con SSH:

```bash
# Clonar repositorio con SSH
git clone ssh://git@git.tudominio.com:222/usuario/repositorio.git

# O configurar SSH en ~/.ssh/config
Host git.tudominio.com
    HostName git.tudominio.com
    Port 222
    User git
    IdentityFile ~/.ssh/id_ed25519
```

### 4. Integración con Authentik (SSO)

Este stack tiene una configuración especial para Authentik:

#### Rutas Protegidas

- `/user/login` - Página de login (protegida con SSO)
- `/explore` - Explorar repositorios públicos (protegida)
- `/TheHomelessSherlock` - Perfil de usuario específico (protegido)

#### Rutas Públicas

- Repositorios públicos accesibles sin autenticación
- API Git (clone, pull, push) funciona sin SSO

#### Configurar OAuth con Authentik

1. En Authentik, crea una aplicación para Gitea:
   - Tipo: **OAuth2/OpenID Provider**
   - Client type: **Confidential**
   - Redirect URIs: `https://git.tudominio.com/user/oauth2/authentik/callback`

2. En Gitea, ve a **Site Administration** → **Authentication Sources** → **Add Authentication Source**
3. Tipo: **OAuth2**
4. Configura:
   - Provider: **OpenID Connect**
   - Client ID: (de Authentik)
   - Client Secret: (de Authentik)
   - OpenID Connect Auto Discovery URL: `https://auth.tudominio.com/application/o/gitea/.well-known/openid-configuration`

5. Actualiza las variables en `stack.env`:
   ```env
   GITEA_ENABLE_OPENID_SIGNIN=true
   GITEA_ENABLE_OPENID_SIGNUP=true
   GITEA_DISABLE_LOGIN_FORM=false  # Mantén false para permitir login local también
   ```

## 🔧 Configuración Avanzada

### Personalización de la UI

Las siguientes opciones están preconfiguradas:

- **Tema por defecto**: Oscuro (`gitea-dark`)
- **Registro deshabilitado**: Solo administradores pueden crear usuarios (o usar SSO)
- **Visibilidad por defecto**: Privada
- **Requiere login para ver**: Activado

Para cambiar estas opciones, edita `stack.env` y actualiza el stack.

### Limitar Acceso por Organización

```env
GITEA_DEFAULT_ALLOW_CREATE_ORGANIZATION=false
GITEA_DEFAULT_ORG_VISIBILITY=private
```

### Notificaciones por Email

Añade al archivo `stack.env`:

```env
GITEA__mailer__ENABLED=true
GITEA__mailer__FROM=gitea@tudominio.com
GITEA__mailer__PROTOCOL=smtp
GITEA__mailer__SMTP_ADDR=smtp.tudominio.com
GITEA__mailer__SMTP_PORT=587
GITEA__mailer__USER=gitea@tudominio.com
GITEA__mailer__PASSWD=tu-password-smtp
```

### Configurar Webhooks

Gitea soporta webhooks para integración con otros servicios:

1. Ve a tu repositorio → **Settings** → **Webhooks**
2. Añade un webhook con la URL de destino
3. Selecciona los eventos que quieres recibir

### Ejemplo de Workflow con Gitea Actions

Crea `.gitea/workflows/build.yml` en tu repositorio:

```yaml
name: Build and Test
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm install
      
      - name: Build
        run: npm run build
      
      - name: Test
        run: npm test
```

## 🛠️ Troubleshooting

### No puedo hacer push/pull por SSH

1. Verifica que el puerto SSH está abierto:
   ```bash
   telnet git.tudominio.com 222
   ```

2. Verifica tu clave SSH:
   ```bash
   ssh -T git@git.tudominio.com -p 222
   ```

3. Añade tu clave SSH en Gitea:
   - Ve a **Settings** → **SSH / GPG Keys** → **Add Key**

### El runner de Actions no se registra

1. Verifica los logs:
   ```bash
   docker logs gitea-act-runner
   ```

2. Verifica que el token es correcto en `stack.env`

3. Regenera el token en Gitea si es necesario

4. Asegúrate de que el runner puede comunicarse con Gitea:
   ```bash
   docker exec gitea-act-runner ping gitea
   ```

### Error de permisos con volúmenes

Si usas SELinux, los volúmenes ya tienen `:Z`:

```bash
# Si persisten problemas
sudo chcon -Rt svirt_sandbox_file_t /opt/gitea/data
sudo chcon -Rt svirt_sandbox_file_t /opt/gitea/postgres
```

### OAuth con Authentik no funciona

1. Verifica que la Redirect URI en Authentik es exacta
2. Comprueba que el Discovery URL es accesible desde el contenedor:
   ```bash
   docker exec gitea curl https://auth.tudominio.com/application/o/gitea/.well-known/openid-configuration
   ```
3. Revisa los logs de Gitea: `docker logs gitea`

### La base de datos no inicia

1. Verifica los logs de PostgreSQL:
   ```bash
   docker logs gitea-postgres
   ```

2. Verifica los permisos del directorio de datos:
   ```bash
   ls -lh /opt/gitea/postgres
   ```

3. Si necesitas reiniciar desde cero:
   ```bash
   docker compose down
   sudo rm -rf /opt/gitea/postgres/*
   docker compose up -d
   ```

## 📚 Recursos Adicionales

- [Documentación oficial de Gitea](https://docs.gitea.io/)
- [Gitea Actions](https://docs.gitea.io/en-us/usage/actions/overview/)
- [Act Runner](https://gitea.com/gitea/act_runner)
- [Migrar desde GitHub/GitLab](https://docs.gitea.io/en-us/usage/migrations/)

## 🔒 Seguridad

- **SSH**: Usa claves SSH en lugar de passwords para Git
- **2FA**: Habilita autenticación de dos factores en tu cuenta
- **Tokens**: Usa tokens de acceso con permisos limitados para CI/CD
- **Webhooks**: Verifica siempre las firmas de webhooks
- **Backups**: Realiza backups regulares de la base de datos y repositorios

## 💾 Backups

### Backup de la Base de Datos

```bash
# Backup completo
docker exec gitea-postgres pg_dump -U gitea gitea > gitea_backup_$(date +%Y%m%d).sql

# Restaurar
cat gitea_backup_YYYYMMDD.sql | docker exec -i gitea-postgres psql -U gitea gitea
```

### Backup de Repositorios y Datos

```bash
# Backup completo de datos
sudo tar -czf gitea-data-backup-$(date +%Y%m%d).tar.gz /opt/gitea/data

# Restaurar
sudo tar -xzf gitea-data-backup-YYYYMMDD.tar.gz -C /
```

### Backup Automático

Considera usar el comando nativo de Gitea:

```bash
docker exec -u git gitea gitea dump -c /data/gitea/conf/app.ini
```

Esto crea un archivo `gitea-dump-*.zip` con todo lo necesario para restaurar.

## 🔄 Actualizaciones

1. **Backup primero**: Siempre haz backup antes de actualizar
2. Actualiza la versión en `stack.env`:
   ```env
   GITEA_IMAGE=gitea/gitea:1.21.0
   ```
3. Actualiza el stack en Portainer
4. Verifica los logs: `docker logs gitea`
5. Prueba que todo funciona correctamente

## 📊 Monitoreo

### Ver Estadísticas

- Accede a **Site Administration** → **Dashboard**
- Revisa usuarios, repositorios, organizaciones, etc.

### Logs

```bash
# Logs de Gitea
docker logs -f gitea

# Logs de PostgreSQL
docker logs -f gitea-postgres

# Logs del Runner
docker logs -f gitea-act-runner
```

### Métricas

Gitea soporta exportación de métricas para Prometheus:

1. Habilita métricas en `stack.env`:
   ```env
   GITEA__metrics__ENABLED=true
   GITEA__metrics__TOKEN=tu-token-secreto
   ```

2. Las métricas estarán disponibles en: `https://git.tudominio.com/metrics?token=tu-token-secreto`
