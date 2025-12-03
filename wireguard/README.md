# WireGuard - VPN Rápida y Segura

WireGuard es una VPN moderna, rápida y segura. Este stack usa `wg-easy` para gestión simplificada de clientes.

## 📋 Descripción

Este stack despliega WireGuard con:
- Interfaz web para gestión de clientes (wg-easy)
- Generación automática de configuraciones de clientes
- Códigos QR para configuración móvil rápida
- Panel web protegido con Authentik
- Puerto UDP para el túnel VPN

## 🚀 Despliegue

### Prerequisitos

1. **Red Docker**: Asegúrate de que la red `proxy` existe
2. **Registro DNS**: Configura el registro A para tu dominio WireGuard
3. **Puerto UDP**: El puerto UDP debe estar abierto en el firewall (por defecto 51820)
4. **Kernel modules**: El servidor debe soportar WireGuard

### Verificar Soporte WireGuard

```bash
# Verificar módulo WireGuard
lsmod | grep wireguard

# O intentar cargarlo
sudo modprobe wireguard

# Si falla, instala wireguard-tools
sudo dnf install wireguard-tools  # Fedora/RHEL
sudo apt install wireguard         # Debian/Ubuntu
```

### Desde Portainer

1. Ve a **Stacks** → **Add stack**
2. Nombre: `wireguard`
3. Selecciona **Repository** o **Git repository**
4. Configura:
   - Repository URL: `<tu-repositorio>`
   - Repository reference: `main`
   - Compose path: `wireguard/docker-compose.yml`
5. Carga el archivo de variables de entorno: `wireguard/stack.env`
6. Haz clic en **Deploy the stack**

### Variables de Entorno

Edita el archivo `stack.env`:

```env
# Imagen
WG_EASY_IMAGE=ghcr.io/wg-easy/wg-easy:latest

# Dominio o IP pública del servidor
WG_HOST=vpn.tudominio.com

# Puerto UDP de WireGuard (debe estar abierto en el firewall)
WG_PORT=51820
WG_UDP_PORT=51820

# Puerto HTTP de la UI (interno)
WG_UI_PORT=51821

# Credenciales iniciales para la UI
INIT_ENABLED=true
INIT_USERNAME=admin
INIT_PASSWORD=tu-password-seguro

# Desactivar IPv6 (si el servidor no lo soporta)
DISABLE_IPV6=true

# Rutas
WG_DATA_PATH=/opt/wireguard/data
WG_MODULES_PATH=/lib/modules

# Dominio de la UI
WG_DOMAIN=vpn-admin.tudominio.com

# Traefik
TRAEFIK_DOCKER_NETWORK=proxy
TRAEFIK_ENTRYPOINT_SECURE=websecure
TRAEFIK_CERTRESOLVER=letsencrypt
TRAEFIK_AUTH_MIDDLEWARE=authentik@docker
```

> **⚠️ Importante**: 
> - Cambia `INIT_PASSWORD` por una contraseña segura
> - Usa tu dominio o IP pública en `WG_HOST`

### Abrir Puerto UDP en el Firewall

```bash
# Firewalld (Fedora/RHEL)
sudo firewall-cmd --permanent --add-port=51820/udp
sudo firewall-cmd --reload

# UFW (Ubuntu/Debian)
sudo ufw allow 51820/udp

# iptables
sudo iptables -A INPUT -p udp --dport 51820 -j ACCEPT
sudo iptables-save > /etc/iptables/rules.v4
```

## ⚙️ Configuración Post-Instalación

### 1. Acceso al Panel Web

1. Accede a `https://vpn-admin.tudominio.com`
2. Si configuraste Authentik, serás redirigido al SSO
3. Inicia sesión con las credenciales configuradas en `INIT_USERNAME` y `INIT_PASSWORD`

### 2. Crear Cliente VPN

1. En el panel web, haz clic en **+ Add Client**
2. Nombre del cliente: `mi-laptop`, `mi-movil`, etc.
3. El cliente se crea automáticamente con:
   - Par de claves pública/privada
   - IP asignada dentro del túnel VPN
   - Configuración completa

### 3. Configurar Cliente

#### Móvil (Android/iOS)

1. Instala la app **WireGuard** desde Play Store o App Store
2. En el panel web, haz clic en el icono **QR** del cliente
3. Escanea el código QR con la app
4. Activa el túnel VPN

#### Windows/Mac/Linux Desktop

##### Opción A: Escanear QR

1. Descarga WireGuard desde [wireguard.com](https://www.wireguard.com/install/)
2. Instala la aplicación
3. Haz clic en **Import tunnel(s) from QR code**
4. Escanea el QR del panel web con tu webcam

##### Opción B: Descargar archivo de configuración

1. En el panel web, haz clic en el icono **Download** del cliente
2. Guarda el archivo `.conf`
3. En WireGuard app, haz clic en **Import tunnel(s) from file**
4. Selecciona el archivo `.conf`

##### Opción C: Configuración manual

Copia la configuración del panel web:

```ini
[Interface]
PrivateKey = tu_clave_privada
Address = 10.8.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = clave_publica_servidor
PresharedKey = clave_compartida
Endpoint = vpn.tudominio.com:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

Guarda como `cliente.conf` y carga en WireGuard.

### 4. Verificar Conexión

Una vez activado el túnel:

```bash
# Verificar IP pública (debe ser la de tu servidor VPN)
curl ifconfig.me

# Ping al servidor VPN (IP interna)
ping 10.8.0.1

# Verificar DNS
nslookup google.com
```

## 🔧 Configuración Avanzada

### Rutear Solo Tráfico Específico (Split Tunneling)

Por defecto, WireGuard enruta TODO el tráfico (`AllowedIPs = 0.0.0.0/0`).

Para rutear solo ciertos rangos:

1. Descarga el archivo `.conf` del cliente
2. Edita `AllowedIPs`:
   ```ini
   AllowedIPs = 10.8.0.0/24, 192.168.1.0/24
   ```
3. Importa el archivo modificado en el cliente

Ejemplos:
- Solo red VPN: `10.8.0.0/24`
- Red VPN + red local del servidor: `10.8.0.0/24, 192.168.1.0/24`
- Todo excepto red local del cliente: `0.0.0.0/1, 128.0.0.0/1`

### Cambiar Rango de IPs de la VPN

Por defecto usa `10.8.0.0/24`. Para cambiar:

1. Detén el stack
2. Edita `/opt/wireguard/data/wireguard.conf`
3. Cambia:
   ```ini
   [Interface]
   Address = 10.9.0.1/24
   ```
4. Actualiza también en cada cliente
5. Reinicia el stack

### Usar DNS Personalizado

Para que los clientes usen tu AdGuard Home:

1. En el panel web, ve a **Settings**
2. Cambia **DNS** a la IP de tu servidor (ej: `192.168.1.10`)
3. O edita manualmente en cada archivo `.conf`:
   ```ini
   DNS = IP-de-tu-AdGuard
   ```

### Limitar Ancho de Banda

No soportado directamente por wg-easy, pero puedes usar `tc` en el servidor:

```bash
# Limitar a 10 Mbps
sudo tc qdisc add dev wg0 root tbf rate 10mbit burst 32kbit latency 400ms
```

### Configurar Post-Up/Post-Down Scripts

Para ejecutar scripts al iniciar/detener el túnel:

Edita `/opt/wireguard/data/wireguard.conf`:

```ini
[Interface]
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
```

## 🛠️ Troubleshooting

### No puedo conectar al VPN

1. Verifica que el puerto UDP está abierto:
   ```bash
   sudo netstat -tulpn | grep 51820
   nc -zuv vpn.tudominio.com 51820
   ```

2. Verifica que WireGuard está corriendo:
   ```bash
   docker logs wg-easy
   ```

3. Verifica que el módulo WireGuard está cargado:
   ```bash
   lsmod | grep wireguard
   ```

4. Verifica la configuración del cliente (especialmente `Endpoint`)

### Conecta pero no hay Internet

1. Verifica que el forwarding está habilitado en el servidor:
   ```bash
   cat /proc/sys/net/ipv4/ip_forward  # Debe ser 1
   ```

2. Verifica las reglas de iptables:
   ```bash
   sudo iptables -L -v -n
   ```

3. Verifica la configuración de NAT en wg-easy

### Panel web no accesible

1. Verifica que está corriendo:
   ```bash
   docker ps | grep wg-easy
   ```

2. Verifica los logs:
   ```bash
   docker logs wg-easy
   ```

3. Verifica Traefik:
   ```bash
   docker logs traefik | grep wg
   ```

### Error "Cannot find device wg0"

El módulo WireGuard no está cargado:

```bash
# Intentar cargar
sudo modprobe wireguard

# Si falla, instalar
sudo dnf install wireguard-tools  # Fedora/RHEL
sudo apt install wireguard         # Debian/Ubuntu
```

### Clientes se desconectan frecuentemente

1. Aumenta `PersistentKeepalive` en la configuración del cliente:
   ```ini
   PersistentKeepalive = 25
   ```

2. Verifica el MTU (puede ser muy alto para tu red):
   ```ini
   MTU = 1280
   ```

### IPv6 causa problemas

Desactiva IPv6 en `stack.env`:

```env
DISABLE_IPV6=true
```

Y reinicia el stack.

## 📚 Recursos Adicionales

- [Documentación oficial de WireGuard](https://www.wireguard.com/)
- [wg-easy en GitHub](https://github.com/wg-easy/wg-easy)
- [Guía de WireGuard](https://www.wireguard.com/quickstart/)

## 🔒 Seguridad

- **Cifrado**: WireGuard usa criptografía moderna (ChaCha20, Curve25519)
- **Panel web**: Protegido con Authentik (SSO)
- **Claves**: Cada cliente tiene su par de claves único
- **PresharedKey**: Capa adicional de seguridad cuántica
- **Firewall**: Solo el puerto UDP de WireGuard debe estar abierto

### Best Practices

1. **Cambia la contraseña** del panel web
2. **Usa Authentik** para proteger el panel
3. **Limita IPs** si solo necesitas acceso desde ciertos rangos
4. **Revoca clientes** cuando ya no los uses
5. **Haz backups** de `/opt/wireguard/data/`

## 💾 Backups

```bash
# Backup de configuraciones
sudo tar -czf wireguard-backup-$(date +%Y%m%d).tar.gz /opt/wireguard/data/

# Restaurar
sudo tar -xzf wireguard-backup-YYYYMMDD.tar.gz -C /
docker restart wg-easy
```

> **Importante**: Guarda los backups de forma segura. Contienen las claves privadas.

## 🔄 Actualizaciones

1. Haz backup de `/opt/wireguard/data/`
2. Actualiza en `stack.env`:
   ```env
   WG_EASY_IMAGE=ghcr.io/wg-easy/wg-easy:latest
   ```
3. Actualiza el stack en Portainer
4. Verifica que los clientes siguen conectando

## 📊 Monitoreo

### Ver Clientes Conectados

En el panel web, verás:
- Clientes activos (verde)
- Última conexión
- Transferencia de datos
- IP asignada

### Ver Stats del Túnel

```bash
# Desde el servidor
docker exec wg-easy wg show

# Ver configuración
docker exec wg-easy cat /etc/wireguard/wg0.conf

# Ver logs
docker logs -f wg-easy
```

## 🎯 Casos de Uso

### Acceso Remoto Seguro

Accede a tu red doméstica desde cualquier lugar:
- Administra servidores
- Accede a servicios internos
- Usa impresoras de red

### Protección en Redes Públicas

Usa WireGuard en WiFi públicas para:
- Cifrar todo el tráfico
- Evitar ataques Man-in-the-Middle
- Proteger tus datos

### Bypass de Restricciones Geográficas

Usa la IP de tu servidor para:
- Acceder a contenido local
- Evitar censura
- Mantener tu IP original

### Combinar con AdGuard

Configura los clientes VPN para usar tu AdGuard Home:
- Bloqueo de anuncios en móvil
- Protección contra rastreadores
- DNS filtrado incluso fuera de casa
