# Zed Fork — distribución

Infraestructura para compilar y distribuir el fork privado de Zed (tú + un amigo).

## Qué hay aquí

- `landing/index.html` — página de descargas con botones para macOS y Windows.
- `Caddyfile` — config del VPS con HTTPS automático + login (basic auth).
- La CI vive en `.github/workflows/fork-release.yml` (raíz del repo).

## Cómo se compila

La build sale de GitHub Actions en tu fork:

1. Ve a **Actions → Fork Release → Run workflow** (o crea un tag `fork-v1.0.0`).
2. El workflow compila **macOS arm64** (`.dmg`) y **Windows x64** (`.exe`), sin firmar,
   y los sube como assets de un **release en borrador**.
3. Revisa el release y públicalo cuando esté OK. Al publicarlo, las URLs
   `releases/latest/download/...` quedan accesibles para la landing.

> En tu Mac también puedes compilar en local con:
> `ZED_BUNDLE_EXTRA_FEATURES="runtime_shaders" ./script/bundle-mac aarch64-apple-darwin`
> (el `runtime_shaders` evita necesitar el compilador Metal de Xcode).

## Cómo se despliega la landing

En el VPS de mvpfactory:

1. Apunta un registro DNS `A` de `zed.tudominio.com` a la IP del VPS.
2. Instala Caddy: <https://caddyserver.com/docs/install>
3. Copia `landing/index.html` a `/var/www/zed-fork/index.html` y edita la URL
   `USER` por tu usuario de GitHub.
4. Genera la contraseña: `caddy hash-password` y pégala en el `Caddyfile`.
5. Copia el `Caddyfile` a `/etc/caddy/Caddyfile` y arranca: `sudo systemctl restart caddy`.

Listo: `https://zed.tudominio.com` pide usuario/contraseña y sirve las descargas.

## Instalación (para el usuario final)

La app no está firmada, así que la primera vez:

- **macOS:** click derecho sobre `Zed Fork.app` → **Abrir**. O en terminal:
  `xattr -dr com.apple.quarantine "/Applications/Zed Fork.app"`
- **Windows:** en el aviso de SmartScreen, **Más información → Ejecutar de todas formas**.

## Pendiente (fases siguientes)

- **Fase 2 — auto-update:** repuntar la URL del updater al VPS y cambiar el canal
  a `stable` para que sondee. Requiere servir un manifiesto JSON
  (`{"version": "...", "url": "..."}`) tras un token secreto.
- **Fase 3 — sync con upstream:** Action programada que, al salir un release de
  Zed, rebase la rama de features y abra una PR en el fork para que la revises.
