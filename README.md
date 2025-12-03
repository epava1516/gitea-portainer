# Portainer Stacks Repository

Este repositorio contiene la configuración de Portainer y múltiples stacks de servicios Docker gestionados mediante Docker Compose.

## 📋 Tabla de Contenidos

- [Descripción](#descripción)
- [Arquitectura](#arquitectura)
- [Prerequisitos](#prerequisitos)
- [Instalación y Despliegue](#instalación-y-despliegue)
- [Stacks Disponibles](#stacks-disponibles)
- [Configuración](#configuración)
- [Uso](#uso)

## 📖 Descripción

Este proyecto proporciona una infraestructura completa de servicios containerizados utilizando:
- **Portainer** como gestor visual de contenedores Docker
- **Traefik** como reverse proxy con certificados SSL automáticos
- **Authentik** para autenticación SSO
- Múltiples servicios adicionales (Gitea, n8n, AdGuard, etc.)

## 🏗️ Arquitectura

La arquitectura sigue este orden de despliegue:

1. **Portainer** (gestor de contenedores) - acceso directo por puerto 9443
2. **Traefik** (reverse proxy) - desplegado desde Portainer
3. **Authentik** (SSO) - desplegado desde Portainer
4. **Resto de stacks** - desplegables desde Portainer

Todos los servicios se comunican a través de la red Docker `proxy` y están protegidos por Traefik con SSL.

> **⚠️ Importante**: Los registros DNS deben estar configurados ANTES de desplegar Traefik y Authentik para que los certificados SSL se generen correctamente.

## ✅ Prerequisitos

- Docker Engine (20.10+)
- Docker Compose (v2.0+)
- Dominio(s) configurado(s) apuntando a tu servidor
- Puertos 80 y 443 abiertos (o 9443 para modo directo)

## 🚀 Instalación y Despliegue

### Paso 1: Clonar el Repositorio

```bash
git clone <repository-url>
cd Portainer_repo
```

### Paso 2: Crear la Red de Docker

Todos los servicios comparten la red `proxy`:

```bash
docker network create proxy
```

### Paso 3: Preparar Secretos de Portainer

Crea el archivo de secreto para la clave de cifrado de Portainer:

```bash
# Crear directorio para secretos
sudo mkdir -p /opt/portainer/secrets

# Generar una clave aleatoria de 32 caracteres
openssl rand -base64 32 | sudo tee /opt/portainer/secrets/portainer

# Asegurar permisos correctos
sudo chmod 600 /opt/portainer/secrets/portainer
```

### Paso 4: Desplegar Portainer (PRIMERO)

Despliega Portainer con acceso directo por puerto 9443:

```bash
docker compose -f docker-compose.9443.yml up -d
```

Verifica que Portainer esté corriendo:
```bash
docker ps | grep portainer
```

Accede a: `https://tu-servidor:9443`

### Paso 5: Configuración Inicial de Portainer

1. Accede a Portainer mediante `https://tu-servidor:9443`
2. Crea el usuario administrador (primera vez - tienes 5 minutos)
3. Selecciona el entorno Docker local
4. Completa la configuración inicial

### Paso 6: Configurar Registros DNS

**Antes de desplegar Traefik y Authentik**, configura los registros DNS:

- Registro A para Traefik Dashboard (ej: `traefik.tudominio.com`)
- Registro A para Portainer UI (ej: `portainer.tudominio.com`)
- Registro A para Portainer API (ej: `portainer-api.tudominio.com`)
- Registro A para Authentik (ej: `auth.tudominio.com`)
- Cualquier otro dominio que vayas a usar

> **⏰ Espera** a que los registros DNS se propaguen antes de continuar (puede tardar de minutos a horas).

Verifica la propagación:
```bash
nslookup traefik.tudominio.com
nslookup portainer.tudominio.com
nslookup auth.tudominio.com
```

### Paso 7: Desplegar Traefik desde Portainer

1. En Portainer, ve a **Stacks** → **Add stack**
2. Nombre: `traefik`
3. Selecciona **Repository** o **Git repository**
4. Configura:
   - Repository URL: `<tu-repositorio>`
   - Repository reference: `main`
   - Compose path: `Traefik/docker-compose.yml`
5. Añade el archivo de variables de entorno: `Traefik/stack.env` (o configúralas manualmente)
6. Haz clic en **Deploy the stack**

Verifica que Traefik esté funcionando:
```bash
docker logs traefik
```

### Paso 8: Desplegar Authentik desde Portainer

1. En Portainer, ve a **Stacks** → **Add stack**
2. Nombre: `authentik`
3. Selecciona **Repository** o **Git repository**
4. Configura:
   - Repository URL: `<tu-repositorio>`
   - Repository reference: `main`
   - Compose path: `authentik/docker-compose.yml`
5. Añade las variables de entorno necesarias
6. Haz clic en **Deploy the stack**

#### Configuración de Authentik

Una vez desplegado Authentik, necesitarás configurar:
- **Applications** (aplicaciones a proteger)
- **Providers** de tipo Forward Auth
- **Authorization Flow** de tipo Implicit
- **Outposts** para ejecutar los providers
- **Middleware** en Traefik

Para instrucciones detalladas, consulta el [README de Authentik](authentik/README.md).

### Paso 9: Configurar Variables de Entorno para Portainer UI

Una vez Traefik y Authentik están funcionando, actualiza el archivo `.env` en la raíz del proyecto:

```bash
nano .env
```

Variables principales a configurar:
- `PORTAINER_DOMAIN`: Tu dominio para Portainer UI (ej: `portainer.tudominio.com`)
- `PORTAINER_API_DOMAIN`: Tu dominio para la API de Portainer (ej: `portainer-api.tudominio.com`)
- `PORTAINER_API_IP_WHITELIST`: IPs permitidas para acceso directo a la API
- `TRAEFIK_AUTH_MIDDLEWARE`: Middleware de autenticación (ej: `authentik@docker`)

### Paso 10: Actualizar Stack de Portainer (Opcional)

Si deseas acceder a Portainer mediante dominio con SSL (en lugar del puerto 9443):

1. En Portainer, ve a **Stacks** → **Add stack**
2. Nombre: `portainer`
3. Selecciona **Upload** y sube el archivo `docker-compose.yml` de la raíz
4. O usa **Repository** apuntando a la raíz del repositorio
5. Añade las variables de entorno del archivo `.env`
6. Haz clic en **Deploy the stack**

> **Nota**: Esto reemplazará el despliegue inicial por puerto 9443 con acceso mediante dominio.

### Paso 11: Desplegar Otros Stacks desde Portainer

Una vez la infraestructura base está funcionando, puedes desplegar el resto de stacks:

#### Método 1: Desde la UI de Portainer (Recomendado)

1. Ve a **Stacks** → **Add stack**
2. Selecciona **Repository**
3. Configura el repositorio Git:
   - URL: `<tu-repositorio>`
   - Reference: `main` (o tu rama)
   - Compose path: `<stack-name>/docker-compose.yml`
4. Añade variables de entorno si es necesario
5. Haz clic en **Deploy the stack**

#### Método 2: Desde línea de comandos

```bash
# Ejemplo: Desplegar Authentik
cd authentik
docker compose up -d
cd ..

# Ejemplo: Desplegar Gitea
cd gitea
docker compose up -d
cd ..
```

## 📦 Stacks Disponibles

| Stack | Descripción | Carpeta | Documentación |
|-------|-------------|---------|---------------|
| **Traefik** | Reverse proxy con SSL automático | `Traefik/` | [README](Traefik/README.md) |
| **Portainer** | Gestor visual de Docker | Raíz (docker-compose.yml) | - |
| **Authentik** | Sistema de autenticación SSO (Forward Auth) | `authentik/` | [README](authentik/README.md) |
| **Gitea** | Servidor Git autoalojado con Actions | `gitea/` | [README](gitea/README.md) |
| **n8n** | Plataforma de automatización de workflows | `n8n/` | [README](n8n/README.md) |
| **AdGuard** | Bloqueador de anuncios DNS con DoT | `adguard/` | [README](adguard/README.md) |
| **Trilium** | Aplicación de notas jerárquicas | `trilium/` | [README](trilium/README.md) |
| **Wireguard** | VPN rápida y segura | `wireguard/` | [README](wireguard/README.md) |
| **Ruleta** | Aplicación Next.js personalizada | `ruleta/` | [README](ruleta/README.md) |

> **📖 Nota**: Cada stack tiene su propio README con instrucciones detalladas de configuración, uso y troubleshooting.

## ⚙️ Configuración

### Archivo .env Principal

El archivo `.env` en la raíz contiene las configuraciones globales:

```env
# Imagen de Portainer
PORTAINER_IMAGE=portainer/portainer-ce:latest

# Rutas de almacenamiento
PORTAINER_SECRET_PATH=/opt/portainer/secrets/portainer
PORTAINER_DATA_PATH=/opt/portainer/data

# Configuración de red y proxy
TRAEFIK_DOCKER_NETWORK=proxy
TRAEFIK_ENTRYPOINT_SECURE=websecure
TRAEFIK_CERTRESOLVER=letsencrypt

# Dominios
PORTAINER_DOMAIN=portainer.example.com
PORTAINER_API_DOMAIN=portainer-api.example.com

# Seguridad
PORTAINER_API_IP_WHITELIST=10.8.0.0/24,172.18.0.1/32
TRAEFIK_AUTH_MIDDLEWARE=authentik@docker
```

### Configuraciones por Stack

Cada stack puede tener su propio archivo `stack.env` o `.env` en su carpeta correspondiente.

## 🎯 Uso

### Ver Logs de Portainer

```bash
docker logs -f portainer
```

### Ver Estado de Todos los Contenedores

```bash
docker ps -a
```

### Actualizar Portainer

```bash
docker compose pull
docker compose up -d
```

### Reiniciar un Stack

Desde Portainer UI o:
```bash
cd <stack-folder>
docker compose restart
```

### Eliminar un Stack

```bash
cd <stack-folder>
docker compose down
# Para eliminar también volúmenes:
docker compose down -v
```

## 🔒 Seguridad

### Acceso a Portainer UI

- **Protegido por SSO**: La UI principal está protegida mediante Authentik (configurable con `TRAEFIK_AUTH_MIDDLEWARE`)
- **Dominio**: Accesible solo mediante el dominio configurado con SSL

### Acceso a API de Portainer

- **Whitelist IP**: La API está restringida a IPs específicas (VPN, localhost)
- **Dominio separado**: Usa un dominio diferente sin SSO para apps móviles
- **SSL**: Todo el tráfico está cifrado

### Base de Datos Cifrada

Portainer utiliza una clave de cifrado para proteger su base de datos, montada en:
- `/run/secrets/portainer`
- `/run/portainer/portainer`

## 🛠️ Troubleshooting

### Portainer no arranca

1. Verifica que la red `proxy` existe: `docker network ls | grep proxy`
2. Revisa los logs: `docker logs portainer`
3. Verifica que el archivo de secreto existe y tiene permisos correctos
4. Verifica que el puerto 9443 no esté ocupado: `sudo netstat -tulpn | grep 9443`

### No puedo acceder mediante dominio

1. Verifica que el dominio apunta a tu servidor: `nslookup tudominio.com`
2. Verifica que los registros DNS están propagados
3. Verifica que Traefik esté corriendo: `docker ps | grep traefik`
4. Verifica la configuración de Traefik: `docker logs traefik`
5. Revisa las labels de Traefik en el docker-compose.yml
6. Verifica que los puertos 80 y 443 estén abiertos: `sudo netstat -tulpn | grep -E ':(80|443)'`
7. Como respaldo, siempre puedes acceder por `https://tu-servidor:9443`

### Error de permisos con volúmenes (SELinux)

Si usas SELinux, los volúmenes ya tienen la opción `:Z` configurada. Si persisten problemas:

```bash
sudo chcon -Rt svirt_sandbox_file_t /opt/portainer/data
```

### Authentik no protege los servicios

1. Verifica que el outpost esté corriendo y en estado **Healthy**
2. Revisa que las aplicaciones y providers estén correctamente configurados
3. Verifica que el authorization flow sea de tipo **Implicit**
4. Comprueba que el middleware de Traefik esté correctamente referenciado en las labels
5. Revisa los logs de Authentik: `docker logs authentik-server`
6. Verifica la conectividad entre Traefik y Authentik en la red `proxy`

## 📝 Notas Adicionales

- **Orden de despliegue**: Siempre desplegar Portainer (9443) → Traefik → Authentik → Resto de stacks
- **Registros DNS**: Configurar ANTES de desplegar Traefik para que Let's Encrypt funcione correctamente
- **Puerto 9443**: Mantén el acceso por puerto 9443 como respaldo en caso de problemas con Traefik
- **Backups**: Considera hacer backup regular de `/opt/portainer/data` y el archivo de secretos
- **Actualizaciones**: Actualiza las imágenes regularmente desde Portainer UI
- **Monitoreo**: Usa Portainer para monitorear recursos y logs de todos los contenedores

## 📄 Licencia

Consulta el archivo LICENSE en este repositorio.

## 🤝 Contribuciones

Las contribuciones son bienvenidas. Por favor, abre un issue o pull request para sugerencias o mejoras.
