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

## Pendiente

- **Auto-update:** repuntar la URL del updater (`auto_update.rs` →
  `get_release_asset`) a un manifiesto servido desde el VPS que reapunte al
  binario en GitHub, y cambiar el canal a `stable` para que sondee.
- **Sync con upstream:** ya automatizado en `.github/workflows/sync-upstream.yml`
  (abre un PR con cada release stable de Zed).
