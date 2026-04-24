# Arcane

Sistema de gestión de contenedores Docker con interfaz web intuitiva. Permite gestionar, monitorear y controlar contenedores Docker de forma visual y sencilla.

Referencia oficial de instalación: https://getarcane.app/docs/setup/installation

## Características

- 🐳 **Gestión de Contenedores**: Control completo de tus contenedores
- 📊 **Monitorización en Tiempo Real**: CPU, memoria, red y disco
- 🔍 **Logs en Vivo**: Visualización de logs de contenedores
- 🖥️ **Interfaz Moderna**: UI intuitiva y responsive
- 🔐 **Seguro**: Autenticación y cifrado de datos
- 📝 **Base de Datos SQLite**: Almacenamiento local eficiente
- 🎨 **Personalizable**: Configuración flexible

## Requisitos Previos

- Docker Engine instalado
- Docker Compose instalado
- URL de acceso definida para la aplicación
- Claves generadas: ENCRYPTION_KEY y JWT_SECRET
- Ruta local de proyectos que quieras exponer a Arcane
- Red Docker externa `proxy` creada, ya que el `compose.yaml` de este repositorio está preparado para conectarse a ella

⚠️ **IMPORTANTE**: Arcane necesita acceso al socket de Docker (`/var/run/docker.sock`).

## Archivos de este Repositorio

Este repositorio contiene archivos de ejemplo:
- `compose.yaml` - Configuración base del contenedor
- `.env.example` - Plantilla de variables de entorno
- `README.md` - Esta documentación

> 💡 **Tip**: Puedes copiar estos archivos manualmente o clonar el repositorio.

---

## Generar Claves Seguras

**Antes de cualquier despliegue**, genera las claves necesarias:

```bash
# You can use OpenSSL in your terminal to generate the secrets
echo "      - ENCRYPTION_KEY=$(openssl rand -hex 32)"
echo "      - JWT_SECRET=$(openssl rand -hex 32)"
```

Guarda los resultados, los necesitarás en el archivo `.env`.

> ⚠️ **Importante**: Nunca compartas estas claves. Cada instalación debe tener claves únicas.

---

## Despliegue con Docker Compose

### 1. Crear Directorio y Archivos

```bash
# Crear directorio
mkdir arcane
cd arcane
```

### 2. Crear compose.yaml

Crea el archivo `compose.yaml`:

```yaml
services:
  arcane:
    image: ghcr.io/getarcaneapp/arcane:latest
    container_name: arcane
    restart: unless-stopped
    ports:
      - 3552:3552
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - arcane_data:/app/data
      - /host/path/to/projects:/host/path/to/projects
    environment:
      - APP_URL=${APP_URL}
      - PROJECTS_DIRECTORY=/host/path/to/projects
      - PUID=1000
      - PGID=1000
      - ENCRYPTION_KEY=${ENCRYPTION_KEY}
      - JWT_SECRET=${JWT_SECRET}

volumes:
  arcane_data:
    name: arcane_data

# Red externa opcional para proxy inverso genérico
networks:
  default:
    external: true
    name: proxy
```

### 3. Configurar Variables de Entorno

Copia `.env.example` a `.env` y ajústalo a tu entorno:

```bash
cp .env.example .env
```

Contenido esperado de `.env`:

```env
# Usa la URL final con la que accederás a la aplicación
# Ejemplo local:  http://localhost:3552
# Ejemplo con proxy: https://arcane.midominio.com
APP_URL=https://arcane.midominio.com

# Claves de Seguridad (GENERAR NUEVAS)
ENCRYPTION_KEY=tu_encryption_key_generada
JWT_SECRET=tu_jwt_secret_generado
```

### 4. Ajustar la Ruta de Proyectos

Antes de desplegar, cambia esta ruta en `compose.yaml` por la ubicación real de tus proyectos:

```yaml
- /host/path/to/projects:/host/path/to/projects
```

Y asegúrate de que la variable `PROJECTS_DIRECTORY` en `compose.yaml` tenga exactamente esa misma ruta.

### 5. Desplegar

```bash
# Crear la red externa requerida por este compose
docker network create proxy

# Iniciar servicios
docker compose up -d

# Ver logs
docker compose logs -f arcane
```

---

## Método Alternativo: Clonar desde Git

Si prefieres usar Git para mantener la configuración actualizada:

```bash
# Clonar repositorio
git clone https://github.com/groales/arcane.git
cd arcane

# Copiar y editar variables
cp .env.example .env
nano .env

# Crear la red externa requerida por este compose
docker network create proxy

# Desplegar
docker compose up -d
```

Si no quieres usar la red `proxy`, comenta o elimina el bloque `networks` del `compose.yaml` antes de arrancar el servicio.

---

## Acceso Inicial

Una vez desplegado, accede a Arcane usando la URL que hayas definido en `APP_URL`.
Puede ser una URL local o una publicada detrás de un proxy inverso genérico.

```text
http://localhost:3552
# o
https://arcane.midominio.com
```

### Primera Configuración

1. Arcane se configurará automáticamente en el primer inicio
2. Tendrás acceso a todos tus contenedores Docker
3. Configura las preferencias desde la interfaz web

---

## Comandos Útiles

### Ver Logs
```bash
docker compose logs -f arcane
```

### Reiniciar Servicio
```bash
docker compose restart arcane
```

### Actualizar Contenedor
```bash
docker compose pull
docker compose up -d
```

### Detener y Eliminar
```bash
docker compose down
# Para eliminar también volúmenes:
docker compose down -v
```

---

## Estructura de Volúmenes

```
Volumen Docker: arcane_data
└── /app/data/arcane.db   # Base de datos SQLite dentro del contenedor

Bind mount configurable:
└── /host/path/to/projects # Directorio de proyectos visible para Arcane
```

---

## Configuración Avanzada

### Variables de Entorno Disponibles

| Variable | Descripción | Valor |
|----------|-------------|-------|
| `APP_URL` | URL de acceso a Arcane | Configurar en `.env` |
| `PROJECTS_DIRECTORY` | Ruta interna donde Arcane verá tus proyectos | Configurar en `compose.yaml` |
| `ENCRYPTION_KEY` | Clave de cifrado (hex 64) | Configurar en `.env` |
| `JWT_SECRET` | Secreto JWT (hex 64) | Configurar en `.env` |
| `PUID` | User ID para permisos | `1000` (hardcoded) |
| `PGID` | Group ID para permisos | `1000` (hardcoded) |


## Solución de Problemas


### Ver Logs Detallados

Para cambiar el nivel de logs, edita `compose.yaml`:

```yaml
environment:
  - PROJECTS_DIRECTORY=/host/path/to/projects
  - LOG_LEVEL=debug  # Cambiar de info a debug
```

Luego reinicia:

```bash
docker compose up -d

# Ver logs
docker compose logs -f arcane
```

---

## Seguridad

### Recomendaciones

1. **Cambiar claves por defecto**: Genera claves únicas con `openssl`
2. **Usar HTTPS si publicas el servicio**: Protege el acceso remoto
3. **Restringir acceso**: Usar firewall o VPN para acceso remoto
4. **Backups regulares**: Respalda periódicamente el volumen `arcane_data`
5. **Actualizar regularmente**: Mantén Arcane actualizado

### Acceso al Socket de Docker

Arcane necesita acceso al socket de Docker. Esto le da control total sobre Docker. Asegúrate de:

- Proteger el acceso a Arcane con autenticación fuerte
- Usar HTTPS exclusivamente
- Considerar restricciones de red

---

## Backup y Restauración

### Backup

```bash
# Detener contenedor
docker compose down

# Backup del volumen de datos
docker run --rm -v arcane_data:/source -v $(pwd):/backup alpine \
  tar -czf /backup/arcane-backup-$(date +%Y%m%d).tar.gz -C /source .

# Reiniciar
docker compose up -d
```

### Restauración

```bash
# Detener contenedor
docker compose down

# Restaurar datos en el volumen
docker run --rm -v arcane_data:/target -v $(pwd):/backup alpine \
  sh -c "rm -rf /target/* /target/.[!.]* /target/..?* 2>/dev/null; tar -xzf /backup/arcane-backup-YYYYMMDD.tar.gz -C /target"

# Reiniciar
docker compose up -d
```

---

## Actualización

```bash
# Detener contenedor
docker compose down

# Hacer backup
docker run --rm -v arcane_data:/source -v $(pwd):/backup alpine \
  tar -czf /backup/arcane-backup-$(date +%Y%m%d).tar.gz -C /source .

# Actualizar imagen
docker compose pull

# Reiniciar
docker compose up -d

# Verificar logs
docker compose logs -f arcane
```

---

## Recursos

- **Instalación Oficial**: [Arcane Installation Guide](https://getarcane.app/docs/setup/installation)
- **Documentación Oficial**: [Arcane Docs](https://getarcane.app/docs)
- **Repositorio Oficial**: [Arcane GitHub](https://github.com/getarcaneapp/arcane)
- **Docker Hub**: [ghcr.io/getarcaneapp/arcane](https://github.com/getarcaneapp/arcane/pkgs/container/arcane)
- **Issues**: [GitHub Issues](https://github.com/getarcaneapp/arcane/issues)

---


## Licencia

Este repositorio de configuración es de uso libre. Arcane tiene su propia licencia, revisa el [repositorio oficial](https://github.com/getarcaneapp/arcane) para más detalles.
