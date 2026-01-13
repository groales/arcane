# Arcane

Sistema de gestión de contenedores Docker con interfaz web intuitiva. Permite gestionar, monitorear y controlar contenedores Docker de forma visual y sencilla.

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
- **Para Traefik o NPM**: Red Docker `proxy` creada
- **Dominio configurado**: Para acceso HTTPS
- **Claves generadas**: ENCRYPTION_KEY y JWT_SECRET

⚠️ **IMPORTANTE**: Arcane necesita acceso al socket de Docker (`/var/run/docker.sock`).

## Archivos de este Repositorio

Este repositorio contiene archivos de ejemplo:
- `docker-compose.yml` - Configuración base del contenedor
- `.env.example` - Plantilla de variables de entorno
- `docker-compose.override.traefik.yml.example` - Labels para Traefik
- `README.md` - Esta documentación

> 💡 **Tip**: Puedes copiar estos archivos manualmente o clonar el repositorio.

---

## Generar Claves Seguras

**Antes de cualquier despliegue**, genera las claves necesarias:

```bash
# ENCRYPTION_KEY (64 caracteres hexadecimales)
openssl rand -hex 32

# JWT_SECRET (64 caracteres hexadecimales)
openssl rand -hex 32
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

### 2. Crear docker-compose.yml

Crea el archivo `docker-compose.yml`:

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
      - data:/app/data
    environment:
      - APP_URL=${APP_URL}
      - PUID=1000
      - PGID=1000
      - ENCRYPTION_KEY=${ENCRYPTION_KEY}
      - JWT_SECRET=${JWT_SECRET}
      - LOG_LEVEL=info
      - LOG_JSON=false
      - DATABASE_URL=file:data/arcane.db?_pragma=journal_mode(WAL)&_pragma=busy_timeout(2500)&_txlock=immediate

volumes:
  data:

# añadir estas líneas al final del archivo para proxy inverso 
networks:
  default:
    external: true
    name: proxy
```

### 3. Configurar Variables de Entorno

Crea el archivo `.env`:

```env
# Dominio
DOMAIN_HOST=arcane.dominio.com
APP_URL=https://arcane.dominio.com

# Claves de Seguridad (GENERAR NUEVAS)
# Generar con: openssl rand -hex 32
ENCRYPTION_KEY=tu_encryption_key_generada
JWT_SECRET=tu_jwt_secret_generado
```

### 4. (Opcional) Configurar Traefik

Si usas Traefik, crea `docker-compose.override.yml`:

```yaml
services:
  arcane:
    labels:
      - traefik.enable=true
      - traefik.http.routers.arcane.rule=Host(`${DOMAIN_HOST}`)
      - traefik.http.routers.arcane.entrypoints=websecure
      - traefik.http.routers.arcane.tls.certresolver=letsencrypt
      - traefik.http.services.arcane.loadbalancer.server.port=3552
```

### 5. Desplegar

```bash
# Crear red proxy si no existe
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
git clone https://git.ictiberia.com/groales/arcane.git
cd arcane

# Copiar y editar variables
cp .env.example .env
nano .env

# Para Traefik
cp docker-compose.override.traefik.yml.example docker-compose.override.yml

# Desplegar
docker network create proxy
docker compose up -d
```

---

## Acceso Inicial

Una vez desplegado, accede a Arcane:

```
https://arcane.dominio.com
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
./data/                    # Directorio de datos persistentes
└── arcane.db             # Base de datos SQLite
```

---

## Configuración Avanzada

### Variables de Entorno Disponibles

| Variable | Descripción | Valor |
|----------|-------------|-------|
| `DOMAIN_HOST` | Dominio para Traefik | Configurar en `.env` |
| `APP_URL` | URL de acceso a Arcane | Configurar en `.env` |
| `ENCRYPTION_KEY` | Clave de cifrado (hex 64) | Configurar en `.env` |
| `JWT_SECRET` | Secreto JWT (hex 64) | Configurar en `.env` |
| `PUID` | User ID para permisos | `1000` (hardcoded) |
| `PGID` | Group ID para permisos | `1000` (hardcoded) |
| `LOG_LEVEL` | Nivel de log | `info` (hardcoded) |
| `LOG_JSON` | Logs en formato JSON | `false` (hardcoded) |
| `DATABASE_URL` | URL de conexión SQLite | `file:data/arcane.db...` (hardcoded) |

### Cambiar Puerto

Si necesitas cambiar el puerto (por defecto 3552):

```yaml
ports:
  - 8080:3552  # Puerto_host:Puerto_contenedor
```

### Personalizar Base de Datos

Puedes modificar los parámetros de SQLite en `DATABASE_URL`:

```env
DATABASE_URL=file:data/arcane.db?_pragma=journal_mode(WAL)&_pragma=busy_timeout(5000)&_txlock=immediate
```

---

## Solución de Problemas

### Error: Cannot connect to Docker socket

```bash
# Verificar permisos del socket
ls -l /var/run/docker.sock

# Si es necesario, añadir usuario al grupo docker
sudo usermod -aG docker $USER
```

### Base de Datos Corrupta

```bash
# Detener contenedor
docker compose down

# Hacer backup de la base de datos
cp data/arcane.db data/arcane.db.backup

# Reiniciar
docker compose up -d
```

### Ver Logs Detallados

Para cambiar el nivel de logs, edita `docker-compose.yml`:

```yaml
environment:
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
2. **Usar HTTPS**: Configura Traefik/NPM con SSL
3. **Restringir acceso**: Usar firewall o VPN para acceso remoto
4. **Backups regulares**: Respalda `./data/` periódicamente
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

# Backup de datos
tar -czf arcane-backup-$(date +%Y%m%d).tar.gz data/

# Reiniciar
docker compose up -d
```

### Restauración

```bash
# Detener contenedor
docker compose down

# Restaurar datos
tar -xzf arcane-backup-YYYYMMDD.tar.gz

# Reiniciar
docker compose up -d
```

---

## Actualización

```bash
# Detener contenedor
docker compose down

# Hacer backup
tar -czf arcane-backup-$(date +%Y%m%d).tar.gz data/

# Actualizar imagen
docker compose pull

# Reiniciar
docker compose up -d

# Verificar logs
docker compose logs -f arcane
```

---

## Recursos

- **Documentación Oficial**: [Arcane GitHub](https://github.com/getarcaneapp/arcane)
- **Docker Hub**: [ghcr.io/getarcaneapp/arcane](https://github.com/getarcaneapp/arcane/pkgs/container/arcane)
- **Issues**: [GitHub Issues](https://github.com/getarcaneapp/arcane/issues)

---

## Wiki Completa

Para documentación detallada, visita la [Wiki de Arcane](../arcane.wiki).

### Contenidos de la Wiki

- **[Home](../arcane.wiki/Home.md)** - Inicio y características
- **[Traefik](../arcane.wiki/Traefik.md)** - Configuración con Traefik
- **[NPM](../arcane.wiki/NPM.md)** - Configuración con Nginx Proxy Manager
- **[Configuración Inicial](../arcane.wiki/Configuración-Inicial.md)** - Primer acceso
- **[Personalización](../arcane.wiki/Personalización.md)** - Configuración avanzada
- **[Backup y Restauración](../arcane.wiki/Backup-y-Restauración.md)** - Protección de datos
- **[Actualización](../arcane.wiki/Actualización.md)** - Mantener actualizado
- **[Solución de Problemas](../arcane.wiki/Solución-de-Problemas.md)** - Diagnóstico

---

## Licencia

Este repositorio de configuración es de uso libre. Arcane tiene su propia licencia, revisa el [repositorio oficial](https://github.com/getarcaneapp/arcane) para más detalles.
