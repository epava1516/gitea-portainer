# Trilium Notes - Aplicación de Notas Jerárquicas

Trilium Notes es una aplicación de toma de notas jerárquicas con editor WYSIWYG, soporte para scripts, y cifrado de notas.

## 📋 Descripción

Este stack despliega Trilium con:
- Almacenamiento persistente de notas
- Acceso mediante HTTPS con Traefik
- Sincronización entre dispositivos
- Sin protección SSO (Trilium tiene su propio sistema de autenticación)
- Headers de seguridad configurados

## 🚀 Despliegue

### Prerequisitos

1. **Red Docker**: Asegúrate de que la red `proxy` existe
2. **Registro DNS**: Configura los registros A para tus dominios Trilium

### Desde Portainer

1. Ve a **Stacks** → **Add stack**
2. Nombre: `trilium`
3. Selecciona **Repository** o **Git repository**
4. Configura:
   - Repository URL: `<tu-repositorio>`
   - Repository reference: `main`
   - Compose path: `trilium/docker-compose.yml`
5. Carga el archivo de variables de entorno: `trilium/stack.env`
6. Haz clic en **Deploy the stack**

### Variables de Entorno

Edita el archivo `stack.env`:

```env
# Imagen
TRILIUM_IMAGE=zadam/trilium:latest

# Hostname (para identificación interna)
TRILIUM_HOSTNAME=trilium-server

# Puerto HTTP interno
TRILIUM_HTTP_PORT=8080

# Dominios (puedes tener múltiples dominios apuntando a la misma instancia)
TRILIUM_DOMAIN_1=notes.tudominio.com
TRILIUM_DOMAIN_2=trilium.tudominio.com

# Ruta de datos
TRILIUM_DATA_PATH=/opt/trilium/data

# Zona horaria
TZ=Europe/Madrid

# Traefik
TRAEFIK_DOCKER_NETWORK=proxy
TRAEFIK_ENTRYPOINT_SECURE=websecure
TRAEFIK_CERTRESOLVER=letsencrypt
```

## ⚙️ Configuración Post-Instalación

### 1. Primer Acceso

1. Accede a `https://notes.tudominio.com`
2. En el primer acceso, Trilium te pedirá crear una contraseña
3. **Guarda bien esta contraseña**: Es la contraseña maestra que protege todas tus notas

### 2. Configuración Inicial

Trilium te guiará por un tutorial interactivo. Explora:
- **Crear notas**: Haz clic en el botón + o usa `Ctrl+P`
- **Jerarquía**: Organiza notas en árbol jerárquico
- **Atributos**: Añade etiquetas (#tag) y relaciones
- **Scripts**: Automatiza tareas con JavaScript

### 3. Cifrado de Notas

Para cifrar notas sensibles:

1. Haz clic derecho en una nota → **Note info**
2. Añade el atributo `#encrypted`
3. Trilium te pedirá una contraseña de cifrado
4. La nota se cifrará localmente

### 4. Sincronización entre Dispositivos

Trilium soporta sincronización servidor-cliente:

#### Configurar Servidor (ya está hecho)

Tu instancia de Trilium ya funciona como servidor.

#### Configurar Cliente Desktop

1. Descarga Trilium Desktop desde [GitHub](https://github.com/zadam/trilium/releases)
2. Instálalo en tu PC/Mac/Linux
3. En el primer inicio, selecciona **Sync from server**
4. Configura:
   - **Server URL**: `https://notes.tudominio.com`
   - **Username**: Crea un usuario de sincronización en el servidor
   - **Password**: Password del usuario

#### Crear Usuario de Sincronización

1. En el servidor web, ve a **Options** → **Sync**
2. Haz clic en **Create sync user**
3. Usuario: `mi-desktop`
4. Password: (genera uno seguro)
5. Copia las credenciales para usarlas en el cliente

## 🔧 Características Principales

### Notas Jerárquicas

- Organiza notas en árbol (padres e hijos)
- Múltiples padres por nota (clones)
- Drag & drop para reorganizar

### Tipos de Notas

- **Text**: Notas de texto con editor WYSIWYG
- **Code**: Notas de código con syntax highlighting
- **Render**: Notas que ejecutan HTML/JavaScript
- **Book**: Agrupaciones de notas
- **Relation map**: Mapas de relaciones entre notas
- **Canvas**: Notas de dibujo libre

### Atributos y Relaciones

```
#etiqueta              - Etiqueta simple
#etiqueta=valor        - Etiqueta con valor
~relacion=@notaId      - Relación a otra nota
#cssClass=mi-clase    - Clase CSS personalizada
#hideChildrenOverview - Oculta hijos en overview
```

### Scripts

Trilium permite automatización con JavaScript:

1. Crea una nota de tipo **Code** (JavaScript)
2. Añade el atributo `#run=frontendStartup` o `#run=backendStartup`
3. El script se ejecutará automáticamente

Ejemplo - Script de búsqueda personalizada:
```javascript
api.addButtonToToolbar({
    title: 'Buscar TODO',
    icon: 'check',
    action: () => {
        const notes = api.searchForNotes('#todo !#done');
        api.showMessage(`Encontradas ${notes.length} tareas pendientes`);
    }
});
```

### Plantillas

Crea plantillas para notas recurrentes:

1. Crea una nota con la estructura deseada
2. Añade `#template`
3. Usa la plantilla: Clic derecho en padre → **Create note from template**

### Web Clipper

Captura contenido web directamente a Trilium:

1. Ve a **Options** → **Web clipper**
2. Sigue las instrucciones para instalar la extensión del navegador
3. Captura páginas web con un clic

## 🔧 Configuración Avanzada

### Backup Automático

Trilium hace backups automáticos diarios en `/opt/trilium/data/backup/`.

Para configurar backups externos:

```bash
# Script de backup
#!/bin/bash
BACKUP_DIR=/backups/trilium
DATE=$(date +%Y%m%d_%H%M%S)

# Backup de toda la data
tar -czf $BACKUP_DIR/trilium-$DATE.tar.gz /opt/trilium/data/

# Mantener solo últimos 30 días
find $BACKUP_DIR -name "trilium-*.tar.gz" -mtime +30 -delete
```

### API

Trilium tiene una API ETAPI para integración:

1. Ve a **Options** → **ETAPI**
2. Crea un token de API
3. Úsalo en tus scripts:

```bash
# Obtener nota
curl -H "Authorization: YOUR_ETAPI_TOKEN" \
     https://notes.tudominio.com/etapi/notes/noteId

# Crear nota
curl -X POST https://notes.tudominio.com/etapi/notes \
     -H "Authorization: YOUR_ETAPI_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"noteId":"new-note","title":"Mi Nota","content":"Contenido"}'
```

### Temas y Estilos

Personaliza la apariencia:

1. Crea una nota de tipo **Code** (CSS)
2. Añade `#appTheme`
3. Escribe tu CSS personalizado:

```css
/* Tema oscuro personalizado */
body {
    --main-background-color: #1e1e1e;
    --main-text-color: #d4d4d4;
}
```

### Shortcuts Personalizados

1. Ve a **Options** → **Keyboard shortcuts**
2. Personaliza o añade nuevos shortcuts

## 🛠️ Troubleshooting

### No puedo acceder a Trilium

1. Verifica que está corriendo:
   ```bash
   docker ps | grep trilium
   ```

2. Verifica los logs:
   ```bash
   docker logs trilium
   ```

3. Verifica DNS:
   ```bash
   nslookup notes.tudominio.com
   ```

### Olvidé la contraseña maestra

Si perdiste la contraseña maestra, NO HAY forma de recuperarla. Las notas están cifradas localmente.

**Prevención**:
- Guarda la contraseña en un gestor de contraseñas
- Haz backups regulares de `/opt/trilium/data/`
- Considera no usar cifrado para todas las notas

### Sincronización no funciona

1. Verifica las credenciales del usuario de sync
2. Revisa los logs del servidor:
   ```bash
   docker logs trilium | grep sync
   ```
3. En el cliente, ve a **Options** → **Sync** → **Check for updates**
4. Fuerza sincronización: **Options** → **Sync** → **Force full sync**

### Notas no se guardan

1. Verifica espacio en disco:
   ```bash
   df -h /opt/trilium/
   ```

2. Verifica permisos:
   ```bash
   ls -lh /opt/trilium/data/
   ```

3. Revisa los logs para errores

### Performance lento

1. Verifica el tamaño de la base de datos:
   ```bash
   du -sh /opt/trilium/data/document.db
   ```

2. Considera optimizar la base de datos:
   - Ve a **Options** → **Advanced** → **Anonymize database**
   - Esto elimina historial antiguo

3. Aumenta recursos del contenedor si es necesario

## 📚 Recursos Adicionales

- [Documentación oficial de Trilium](https://github.com/zadam/trilium/wiki)
- [Galería de plantillas](https://github.com/zadam/trilium/wiki/Gallery)
- [Scripts de ejemplo](https://github.com/zadam/trilium/wiki/Scripts)
- [Forum de la comunidad](https://github.com/zadam/trilium/discussions)

## 🔒 Seguridad

- **Contraseña maestra**: Protege todas tus notas
- **Cifrado opcional**: Usa `#encrypted` para notas sensibles
- **HTTPS**: Todo el tráfico está cifrado con SSL
- **Backups**: Haz backups regulares de tus notas
- **Sin SSO**: Trilium gestiona su propia autenticación (más seguro para notas personales)

## 💾 Backups

### Backup Manual

```bash
# Backup completo
sudo tar -czf trilium-backup-$(date +%Y%m%d).tar.gz /opt/trilium/data/

# Solo base de datos
sudo cp /opt/trilium/data/document.db trilium-db-backup-$(date +%Y%m%d).db
```

### Restaurar

```bash
# Detener Trilium
docker stop trilium

# Restaurar backup
sudo tar -xzf trilium-backup-YYYYMMDD.tar.gz -C /

# Reiniciar
docker start trilium
```

### Backup Automático (dentro de Trilium)

Trilium hace backups automáticos:
- **Ubicación**: `/opt/trilium/data/backup/`
- **Frecuencia**: Diaria
- **Retención**: Configurable en **Options** → **Other**

## 🔄 Actualizaciones

1. Haz **backup completo** antes de actualizar
2. Actualiza en `stack.env`:
   ```env
   TRILIUM_IMAGE=zadam/trilium:0.63.7
   ```
3. Actualiza el stack en Portainer
4. Verifica los logs: `docker logs trilium`
5. Accede y verifica que todo funciona

> **Nota**: Revisa el [changelog](https://github.com/zadam/trilium/blob/master/CHANGELOG.md) antes de actualizar.

## 📊 Uso

### Casos de Uso

- **Notas personales**: Diario, ideas, recordatorios
- **Base de conocimiento**: Wiki personal, documentación
- **Gestión de proyectos**: Tareas, planificación
- **Investigación**: Organización de información
- **Snippets de código**: Biblioteca de código reutilizable
- **Journaling**: Diario personal cifrado

### Tips

- Usa `Ctrl+P` para búsqueda rápida
- Usa `Ctrl+S` para guardar (automático)
- Usa `Alt+Up/Down` para navegar por el árbol
- Usa `Ctrl+K` para crear links internos
- Usa `#` para etiquetas en el título
- Usa backlinks para ver referencias a una nota
