# 🔎 SearXNG + Open WebUI en Docker

[![GitHub](https://img.shields.io/badge/GitHub-Repo-blue?logo=github)](https://github.com/tu-usuario/searxng-openwebui-docker)
[![Docker](https://img.shields.io/badge/Docker-Ready-blue?logo=docker)](https://hub.docker.com/r/searxng/searxng)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

## 📋 Descripción general

Este repositorio proporciona la configuración completa para desplegar **SearXNG** (metabuscador privado) y **Open WebUI** (interfaz de chat para modelos de IA) en contenedores Docker separados, comunicados mediante una red Docker compartida llamada `ai`. Permite a tus modelos de IA realizar búsquedas web en tiempo real y citar fuentes, todo autoalojado sin API keys ni cuotas externas.

## ✨ Características principales

- 🔍 **Búsqueda web privada**: SearXNG consulta múltiples motores (DuckDuckGo, Brave, Startpage…) sin rastreo ni perfilado
- 🤖 **IA con acceso a internet**: Open WebUI puede buscar información actual y citar fuentes en sus respuestas
- 📦 **Servicios desacoplados**: Cada servicio en su propio `docker-compose.yml`; actualizaciones y reinicios independientes
- 🌐 **Red compartida reutilizable**: La red `ai` sirve para añadir Ollama, n8n u otros servicios futuros
- 🛡️ **Sin API keys ni cuotas**: Totalmente autoalojado, sin dependencias de servicios de pago
- 🥧 **Ligero y compatible**: Funciona en Raspberry Pi (arm64) y arquitecturas x86_64
- 🖥️ **Interfaz propia de SearXNG**: Accesible opcionalmente desde el navegador como buscador privado familiar

## 📋 Requisitos del sistema

- Docker Engine ≥ 20.10
- Docker Compose ≥ 2.0 (plugin `docker compose`)
- Puerto **8082** libre en el host (para la interfaz web de SearXNG, opcional)
- Puerto **7000** libre en el host (para Open WebUI)
- Open WebUI ya desplegado o listo para desplegar (ver paso 3)
- (Opcional) `openssl` para generar la clave secreta de SearXNG

## 🐳 Instalación

### Paso 1: Crear la red compartida (una sola vez)

```bash
docker network create ai
```

### Paso 2: Desplegar SearXNG

```bash
mkdir -p ~/docker/searxng/searxng
cd ~/docker/searxng
```

Crea el archivo de configuración `searxng/settings.yml`:

```yaml
use_default_settings: true
server:
  secret_key: "CAMBIA_ESTO"
  limiter: false
  image_proxy: false
search:
  formats:
    - html
    - json
```

> **Importante**: El formato `json` es **imprescindible**. Sin él, Open WebUI recibe un error 403 al buscar.

Genera una clave secreta segura y aplícala:

```bash
sed -i "s|CAMBIA_ESTO|$(openssl rand -hex 32)|" searxng/settings.yml
```

Crea `docker-compose.yml` para SearXNG:

```yaml
services:
  searxng:
    container_name: searxng
    image: searxng/searxng:latest
    ports:
      - "8082:8080"  # host:interno (cambia el 8082 si está ocupado)
    volumes:
      - ./searxng:/etc/searxng
    networks:
      - ai
    restart: unless-stopped

networks:
  ai:
    external: true
```

> **Nota**: Si solo vas a usar SearXNG desde Open WebUI, elimina el bloque `ports` y no publicarás nada en el host.

Arranca el servicio:

```bash
docker compose up -d
```

### Paso 3: Conectar Open WebUI a la red `ai`

En tu carpeta de Open WebUI (ej. `~/docker/open-webui`), edita o crea `compose.yml`:

```yaml
services:
  openwebui:
    container_name: open-webui
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "7000:8080"
    environment:
      - WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY}
      - WEBUI_AUTH=True
      - WEBUI_REGISTER=false
      - ENABLE_WEB_SEARCH=true
      - WEB_SEARCH_ENGINE=searxng
      - SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>
    volumes:
      - ./open-webui-data:/app/backend/data
    networks:
      - ai
    restart: unless-stopped

networks:
  ai:
    external: true
```

Arranca/actualiza Open WebUI:

```bash
docker compose up -d
```

## ⚙️ Configuración

1. **Clave secreta de SearXNG**: Generada automáticamente con `openssl rand -hex 32` en el paso 2. No la compartas.
2. **Formato JSON en SearXNG**: Verifica que `search.formats` incluya `json` en `settings.yml`.
3. **Variables de entorno de Open WebUI**: Se leen **solo en el primer arranque**. Si ya tenías configuración guardada, configúralo desde la interfaz (ver Primeros pasos).
4. **URL de consulta interna**: Usa siempre `http://searxng:8080/search?q=<query>` (puerto interno 8080). El 8082 es solo para acceso desde el host.
5. **Permisos de carpeta SearXNG**: Si hay errores de permisos, ejecuta `sudo chown -R 977:977 ~/docker/searxng/searxng`.
6. **Red `ai`**: Debe estar declarada como `external: true` en **ambos** `docker-compose.yml`.

## 🚀 Primeros pasos

1. Crea la red compartida: `docker network create ai`
2. Despliega SearXNG siguiendo el **Paso 2** (configuración + `docker compose up -d`)
3. Despliega Open WebUI siguiendo el **Paso 3** (`docker compose up -d`)
4. Accede a Open WebUI en `http://IP-DEL-HOST:7000`
5. Si es el primer arranque, la búsqueda web ya estará activa. Si ya tenías configuración:
   - Ve a **Panel de administración → Ajustes → Búsqueda web**
   - Activa el interruptor **Búsqueda web**
   - Motor de búsqueda: `searxng`
   - URL de consulta: `http://searxng:8080/search?q=<query>`
   - Resultados: 3-5 | Concurrencia: 1-2 (en Raspberry Pi)
   - Guarda
6. En un chat nuevo, activa la búsqueda con el botón `+` o el icono del globo 🌐
7. (Opcional) Accede a la interfaz de SearXNG en `http://IP-DEL-HOST:8082`

## 💡 Casos de uso

- Preguntar a tu modelo por noticias, versiones recientes de software o documentación actualizada
- Usar SearXNG como buscador privado para toda la red local / familia
- Montar asistentes que consulten fuentes externas antes de responder (RAG ligero sin indexar)
- Combinar con Ollama (en la misma red `ai`) para inferencia 100% local con búsqueda web
- Integrar en flujos de n8n que necesiten búsqueda web autónoma

## 🔒 Acceso remoto seguro

> **No abras el puerto 8082 (ni el 7000) a internet directamente.**

Para acceder desde fuera de tu red local:
- **VPN recomendada**: WireGuard, Tailscale o ZeroTier
- **Proxy inverso con HTTPS y autenticación**: Nginx Proxy Manager, Traefik o Caddy + Authelia / Authentik
- **Túnel seguro**: Cloudflare Tunnel (con Access policies) o ngrok (solo pruebas)

## 🛠️ Gestión y mantenimiento

### Actualizar servicios (dentro de cada carpeta)

```bash
docker compose pull
docker compose up -d
```

### Ver logs

```bash
# SearXNG
docker logs searxng --tail 50

# Open WebUI
docker logs open-webui --tail 50
```

### Verificar conectividad

```bash
# Contenedores en la red ai
docker network inspect ai --format '{{range .Containers}}{{.Name}} {{end}}'

# Open WebUI alcanza a SearXNG y recibe JSON
docker exec open-webui curl -s "http://searxng:8080/search?q=test&format=json" | head -c 200
```

### Problemas frecuentes

| Síntoma | Solución |
|---------|----------|
| `curl` devuelve 403 | Falta `json` en `search.formats`. Añádelo y `docker compose restart searxng` |
| `Could not resolve host: searxng` | Los contenedores no comparten red. Revisa `networks: - ai` en ambos composes |
| SearXNG no arranca por permisos | `sudo chown -R 977:977 ~/docker/searxng/searxng` |
| Variables del compose no hacen efecto | Configúralo en **Ajustes → Búsqueda web**. Alternativa: `ENABLE_PERSISTENT_CONFIG=False` |
| No aparece el motor `searxng` en Open WebUI | Imagen antigua: `docker compose pull openwebui && docker compose up -d` |
| Un motor te bloquea | Desactívalo en `settings.yml` y deja DuckDuckGo, Brave o Startpage |

## 📝 Licencia

Este proyecto se distribuye bajo licencia **MIT**. Consulta el archivo [LICENSE](LICENSE) para más detalles.

---

> 📖 **Tutorial completo**: [Cómo instalar SearXNG + Open WebUI en Docker: búsqueda web para tu IA](https://genbyte.blogspot.com/2026/11/como-instalar-searxng-open-webui-en.html)  
> 🎥 **Vídeo en YouTube**: [Canal GENBYTE](https://www.youtube.com/@genbyte)  
> ☕ **Apoya el canal**: [Invítame a un Ko-fi](https://ko-fi.com/genbyte)