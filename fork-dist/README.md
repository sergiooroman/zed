# Zed Fork — distribución

Infraestructura para compilar y distribuir el fork de Zed (uso privado).

## Qué hay aquí

- `landing/index.html` — página de descargas (macOS + Windows).
- La CI vive en `.github/workflows/fork-release.yml` (raíz del repo).
- `Caddyfile` — config alternativa si algún día se sirve la landing con Caddy
  en vez de Dokploy (no se usa en el despliegue actual).

## Cómo se compila

- **Windows** sale de GitHub Actions: **Actions → Fork Release → Run workflow**,
  o creando un tag `fork-v*`. Produce el `.exe` sin firmar y lo sube a un
  release en borrador.
- **macOS** se compila en local (los runners estándar de GitHub fallan en
  `webrtc-sys` con su Xcode):
  `ZED_BUNDLE_EXTRA_FEATURES="runtime_shaders" ./script/bundle-mac aarch64-apple-darwin`
  y se sube el `.dmg` al mismo release.

Los binarios viven en **GitHub Releases** del fork; la landing enlaza a
`releases/latest/download/...`. Publica el release (deja de ser borrador) para
que esos enlaces resuelvan.

## Despliegue de la landing (VPS actual)

Servida en `zed.sergioroman.tech` vía **Dokploy + Traefik** (Docker Swarm), sin
login. Componentes en el VPS `mvpvps`:

- Contenedor `zed-fork` (nginx) montando `/opt/zed-fork/index.html`, en la red
  `dokploy-network`.
- Router en `/etc/dokploy/traefik/dynamic/zedfork.yml` (file provider de
  Traefik), con HTTPS automático (Let's Encrypt) por el certResolver de Dokploy.

Para actualizar la landing: copiar `landing/index.html` a
`/opt/zed-fork/index.html` en el VPS (nginx lo sirve al vuelo).

## Instalación (usuario final)

La app no está firmada:

- **macOS:** click derecho sobre `Zed Fork.app` → **Abrir**, o
  `xattr -dr com.apple.quarantine "/Applications/Zed Fork.app"`.
- **Windows:** en SmartScreen, **Más información → Ejecutar de todas formas**.

## Auto-update

Cómo funciona (Fase 2, ya montada):

1. La app (canal `dev`, con `poll_for_updates` forzado a `true`) consulta cada
   hora `https://zed.sergioroman.tech/update/zed-{os}-{arch}.json`
   (cambio en `auto_update.rs` → `get_release_asset`).
2. Un **cron en el VPS** (`/opt/zed-fork/update-manifest.sh`, cada 5 min) lee el
   último release **publicado** de GitHub y escribe esos JSON:
   `{"version": "X.Y.Z", "url": "<asset .dmg/.exe en GitHub>"}`.
3. Si la versión del manifiesto es mayor que la instalada, la app descarga el
   binario de GitHub y lo aplica (mac: monta el dmg y rsync-ea `Zed Fork.app`;
   windows: ejecuta el `.exe`).

La versión se embebe en el binario desde `crates/zed/Cargo.toml` en tiempo de
compilación; el workflow la fija desde el tag.

## Cómo publicar una versión nueva

1. Crea un tag `fork-vX.Y.Z` **con una versión mayor que la anterior** y **>= la
   base de upstream** (hoy la base es `1.11.0`, así que usa `fork-v1.11.1`,
   `fork-v1.11.2`… ; tras sincronizar upstream a 1.12.0, pasa a `fork-v1.12.1`).
   Si la versión no sube, el auto-update no la detecta.
2. La CI compila Windows y sube el `.exe`; sube el `.dmg` de macOS (build local)
   al mismo release.
3. **Publica** el release (deja de ser borrador). El cron del VPS lo detecta en
   ≤5 min y actualiza los manifiestos → las apps se auto-actualizan.

## Sync con upstream

Automatizado en `.github/workflows/sync-upstream.yml`: abre un PR con cada
release stable de Zed. Al mergearlo, crea un tag `fork-v*` para publicar.
