# AdGuard Home - Bloqueador de Anuncios DNS

AdGuard Home es un servidor DNS que bloquea anuncios y rastreadores a nivel de red.

## 📋 Descripción

Este stack despliega AdGuard Home con:
- Bloqueo de anuncios y rastreadores
- DNS-over-TLS (DoT) para privacidad
- Panel web protegido con Authentik
- Integración con Traefik para HTTPS
- Configuración persistente

## 🚀 Despliegue

### Prerequisitos

1. **Red Docker**: Asegúrate de que la red `proxy` existe
2. **Registro DNS**: Configura el registro A para tu dominio AdGuard
3. **IP Fija**: AdGuard necesita una IP fija en la red Docker
4. **Certificados DoT**: Necesitarás certificados SSL para DoT

### Configurar IP Fija en la Red Proxy

Primero, verifica el rango de IPs de la red `proxy`:

```bash
docker network inspect proxy | grep Subnet
```

Luego, elige una IP fija dentro del rango (ej: `172.18.0.10`).

### Generar Certificados para DoT

```bash
# Crear directorio para certificados
sudo mkdir -p /opt/adguard/certs

# Copiar certificados de Let's Encrypt (después de que Traefik los genere)
sudo cp /opt/traefik/letsencrypt/certificates/adguard.tudominio.com.crt /opt/adguard/certs/adguard.crt
sudo cp /opt/traefik/letsencrypt/certificates/adguard.tudominio.com.key /opt/adguard/certs/adguard.key

# O genera certificados autofirmados para pruebas
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /opt/adguard/certs/adguard.key \
  -out /opt/adguard/certs/adguard.crt \
  -subj "/CN=adguard.tudominio.com"
```

### Desde Portainer

1. Ve a **Stacks** → **Add stack**
2. Nombre: `adguard`
3. Selecciona **Repository** o **Git repository**
4. Configura:
   - Repository URL: `<tu-repositorio>`
   - Repository reference: `main`
   - Compose path: `adguard/docker-compose.yml`
5. Carga el archivo de variables de entorno: `adguard/stack.env`
6. Haz clic en **Deploy the stack**

### Variables de Entorno

Edita el archivo `stack.env`:

```env
# Imagen
ADGUARD_IMAGE=adguard/adguardhome:latest

# Dominio
ADGUARD_DOMAIN=adguard.tudominio.com

# IP fija en la red proxy
ADGUARD_IPV4=172.18.0.10

# Puerto DoT (DNS-over-TLS)
ADGUARD_DOT_PORT=853

# Puerto HTTP del panel (interno)
ADGUARD_HTTP_PORT=80

# Rutas
ADGUARD_WORK_PATH=/opt/adguard/work
ADGUARD_CONF_PATH=/opt/adguard/conf
ADGUARD_CERT_CRT_PATH=/opt/adguard/certs/adguard.crt
ADGUARD_CERT_KEY_PATH=/opt/adguard/certs/adguard.key

# Traefik
TRAEFIK_DOCKER_NETWORK=proxy
TRAEFIK_ENTRYPOINT_SECURE=websecure
TRAEFIK_CERTRESOLVER=letsencrypt
TRAEFIK_AUTH_MIDDLEWARE=authentik@docker
```

## ⚙️ Configuración Post-Instalación

### 1. Configuración Inicial

La primera vez que accedas a `https://adguard.tudominio.com`, se iniciará el asistente:

1. **Puerto de escucha**: Deja 3000 (se cambiará automáticamente a 80)
2. **Usuario y contraseña**: Crea las credenciales de administrador
3. **Interfaces de red**: Escucha en todas las interfaces
4. **DNS upstream**: Configura servidores DNS (ej: `1.1.1.1`, `8.8.8.8`)

> **Nota**: Si el panel escucha en puerto 3000, actualiza `ADGUARD_HTTP_PORT=3000` en el `stack.env` temporalmente.

### 2. Configurar DNS-over-TLS (DoT)

1. En el panel de AdGuard, ve a **Settings** → **Encryption settings**
2. Habilita **DNS-over-TLS**
3. Configura:
   - **Server name**: `adguard.tudominio.com`
   - **Certificate**: `/certs/adguard.crt`
   - **Private key**: `/certs/adguard.key`
4. **Puerto**: 853
5. Haz clic en **Save**

### 3. Configurar Listas de Filtros

AdGuard viene con listas por defecto, pero puedes añadir más:

1. Ve a **Filters** → **DNS blocklists**
2. Añade listas populares:
   - **Steven Black's Hosts**: Bloquea malware y anuncios
   - **OISD**: Lista curada de dominios maliciosos
   - **Hagezi**: Listas agresivas de bloqueo
3. Haz clic en **Add blocklist** y pega la URL

Ejemplos:
```
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
https://big.oisd.nl/
https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/domains/pro.txt
```

### 4. Configurar DNS Upstream

Para mejor privacidad, usa DNS cifrados:

1. Ve a **Settings** → **DNS settings**
2. En **Upstream DNS servers**, añade:
   ```
   https://dns.cloudflare.com/dns-query
   https://dns.google/dns-query
   tls://1.1.1.1
   tls://8.8.8.8
   ```
3. En **Bootstrap DNS servers** (para resolver los DNS cifrados):
   ```
   1.1.1.1
   8.8.8.8
   ```

## 📱 Configurar Clientes

### Android - DNS Privado (DoT)

1. Ve a **Ajustes** → **Conexiones** → **Más ajustes de conexión**
2. Selecciona **DNS privado**
3. Elige **Nombre del host del proveedor de DNS privado**
4. Introduce: `adguard.tudominio.com`
5. Guarda

> Todo el tráfico DNS de tu móvil pasará por AdGuard, incluso fuera de casa.

### iOS - DNS-over-HTTPS

1. Descarga un perfil de configuración DoH
2. O usa apps como **AdGuard** o **DNSCloak**

### Windows

#### Opción 1: Cambiar DNS del Sistema

1. Panel de Control → Redes → Propiedades del adaptador
2. IPv4 → Propiedades
3. DNS preferido: `IP-de-tu-servidor`
4. DNS alternativo: `8.8.8.8`

#### Opción 2: DNS-over-HTTPS (Windows 11)

1. Ajustes → Red e Internet → Ethernet/Wi-Fi
2. Editar DNS
3. Añade servidor DNS-over-HTTPS: `https://adguard.tudominio.com/dns-query`

### Linux

Edita `/etc/resolv.conf`:

```bash
nameserver IP-de-tu-servidor
nameserver 8.8.8.8
```

O con systemd-resolved:

```bash
sudo nano /etc/systemd/resolved.conf
```

```ini
[Resolve]
DNS=IP-de-tu-servidor
FallbackDNS=8.8.8.8
```

```bash
sudo systemctl restart systemd-resolved
```

### Router

Configura AdGuard como DNS primario en tu router:
1. Accede al panel de tu router
2. Ve a configuración DHCP/DNS
3. DNS primario: `IP-de-tu-servidor`
4. DNS secundario: `8.8.8.8`

> **Ventaja**: Todos los dispositivos de tu red usarán AdGuard automáticamente.

## 🔧 Configuración Avanzada

### Listas Blancas (Whitelist)

Para permitir dominios bloqueados por error:

1. Ve a **Filters** → **Custom filtering rules**
2. Añade reglas:
   ```
   @@||ejemplo-permitido.com^
   @@||*.ejemplo.com^
   ```

### Bloquear Dominios Específicos

```
||dominio-bloqueado.com^
||ads.ejemplo.com^
```

### Reglas de Reescritura DNS

Para resolver dominios internos:

1. Ve a **Filters** → **DNS rewrites**
2. Añade entradas:
   - **Domain**: `home.local`
   - **Answer**: `192.168.1.10`

### Clientes Personalizados

Para aplicar configuraciones diferentes por cliente:

1. Ve a **Settings** → **Client settings**
2. Añade cliente por IP o ID
3. Configura filtros específicos, upstream DNS, etc.

### Estadísticas y Logs

- **Dashboard**: Ve estadísticas de consultas, clientes, dominios bloqueados
- **Query log**: Revisa todas las consultas DNS en tiempo real
- **Filtros por cliente**: Ve qué dispositivos hacen más consultas

## 🛠️ Troubleshooting

### No puedo acceder al panel web

1. Verifica que AdGuard está corriendo:
   ```bash
   docker ps | grep adguard
   ```

2. Verifica los logs:
   ```bash
   docker logs adguardhome
   ```

3. Si el panel está en puerto 3000, cambia `ADGUARD_HTTP_PORT=3000` temporalmente

### DNS no resuelve dominios

1. Verifica que AdGuard está escuchando:
   ```bash
   docker exec adguardhome netstat -tulpn
   ```

2. Prueba DNS desde el servidor:
   ```bash
   nslookup google.com 172.18.0.10
   ```

3. Verifica que los upstream DNS están configurados

### DoT no funciona en Android

1. Verifica que el puerto 853 está abierto:
   ```bash
   sudo netstat -tulpn | grep 853
   ```

2. Verifica los certificados en AdGuard:
   - Settings → Encryption → Verifica que están cargados correctamente

3. Prueba DoT desde el servidor:
   ```bash
   kdig -d @adguard.tudominio.com +tls google.com
   ```

### Algunos sitios no cargan

1. Ve a **Query log** en AdGuard
2. Busca el dominio bloqueado
3. Añádelo a la whitelist si es necesario

### IP fija causa conflictos

Si la IP configurada está en uso:

1. Elige otra IP del rango de la red `proxy`
2. Actualiza `ADGUARD_IPV4` en `stack.env`
3. Actualiza el stack

## 📚 Recursos Adicionales

- [Documentación oficial de AdGuard Home](https://github.com/AdguardTeam/AdGuardHome/wiki)
- [Listas de filtros](https://filterlists.com/)
- [Comparativa con Pi-hole](https://github.com/AdguardTeam/AdGuardHome#comparison)

## 🔒 Seguridad

- **Panel web**: Siempre protegido con Authentik (SSO)
- **DoT**: Cifra las consultas DNS entre cliente y servidor
- **Upstream DNS**: Usa DNS cifrados (DoH/DoT) para privacidad total
- **Logs**: Revisa regularmente los logs para detectar anomalías
- **Actualizaciones**: Mantén AdGuard actualizado

## 💾 Backups

```bash
# Backup de configuración
sudo tar -czf adguard-backup-$(date +%Y%m%d).tar.gz /opt/adguard/

# Restaurar
sudo tar -xzf adguard-backup-YYYYMMDD.tar.gz -C /
```

## 🔄 Actualizaciones

1. Haz backup de la configuración
2. Actualiza en `stack.env`:
   ```env
   ADGUARD_IMAGE=adguard/adguardhome:latest
   ```
3. Actualiza el stack en Portainer
4. Verifica los logs y el panel web

## 📊 Estadísticas

AdGuard ofrece estadísticas detalladas:
- **Consultas bloqueadas**: Porcentaje y cantidad
- **Top clientes**: Dispositivos que más consultas hacen
- **Top dominios**: Sitios más visitados
- **Top dominios bloqueados**: Anuncios y rastreadores más frecuentes
- **Upstream DNS**: Rendimiento de los servidores DNS configurados

Accede a estas estadísticas en el Dashboard de AdGuard.
