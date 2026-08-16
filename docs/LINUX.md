# videodrome — Linux (GUI + player embebido)

Notas específicas para Linux. La GUI usa WebKitGTK 2.44+ (`webkit2gtk-4.1`)
con Media Source Extensions (MSE) para reproducir HLS vía `hls.js`. El
transmux lo hace `ffmpeg` local. Todo lo que sigue son problemas reales
observados en la ruta AppImage / `.deb`.

## Instalación rápida

```bash
curl -fsSL https://ser356.github.io/videodrome-releases/setup.sh | bash
```

Modos alternativos: `--deb` (system-wide), `--cli` (headless, sin GUI).
Ver [README](../README.md#linux--appimage--deb--tarball-cli).

## Dependencias en tiempo de ejecución

| Componente | AppImage | `.deb` | Notas |
|---|---|---|---|
| WebKitGTK 4.1 | Sistema | Auto (apt) | Requerido para la GUI |
| GTK 3 | Sistema | Auto (apt) | Requerido para la GUI |
| AyatanaAppIndicator3 | Sistema | Auto (apt) | Tray / notificaciones |
| GStreamer good/bad/libav | **Embebido** | Recomendado | Codecs H.264/AAC para MSE |
| ffmpeg + ffprobe | Host | Auto (apt) | Transmux HLS |
| libfuse2 | Host | N/A | Requerido para AppImage v1 |

En el AppImage los plugins de GStreamer van embebidos gracias a
`bundleMediaFramework: true` en `tauri.conf.json`. En el `.deb` NO se
declaran como dependencia dura (mucho peso), así que si el vídeo va sin
audio o negro, instala:

```bash
sudo apt install gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-libav
```

## Reproducción HLS embebida

El player usa `hls.js` (misma ruta que Windows). Detección:

1. `<video>.canPlayType('application/vnd.apple.mpegurl')` → **vacío** en
   WebKitGTK. No hay HLS nativo.
2. `Hls.isSupported()` → **true** cuando MSE está disponible (WebKitGTK
   2.32+ con MSE compilado en, que es el default en Ubuntu 22.04+).
3. `hls.js` alimenta el `MediaSource` con segmentos `fMP4` fragmentados
   que sirve el backend Rust vía HTTP local (`127.0.0.1:<puerto>`).

Si `Hls.isSupported()` devuelve `false`, tu WebKitGTK viene sin MSE
compilado (raro en 2026, pero posible en distros minimalistas). Solución:
usar VLC externo desde **Ajustes → Player = VLC**.

## Problemas conocidos

### AppImage no arranca (nada pasa al doble-click)

Causa habitual: falta `libfuse2` (AppImage v1 lo usa para montarse).

```bash
sudo apt install libfuse2
```

En distros muy nuevas, alternativa: `--appimage-extract` y ejecutar
`squashfs-root/AppRun`.

### Pantalla en negro en NVIDIA propietario

`libwebkit2gtk` puede fallar al inicializar el compositor GL con drivers
NVIDIA. Workaround:

```bash
WEBKIT_DISABLE_DMABUF_RENDERER=1 videodrome
```

Si funciona, hazlo permanente añadiéndolo a `~/.profile` o al desktop
entry (`Exec=env WEBKIT_DISABLE_DMABUF_RENDERER=1 videodrome`).

### Vídeo en negro / "no compatible source" en `<video>`

WebKitGTK usa GStreamer como backend. Sin los plugins de codecs
propietarios (H.264, AAC), MSE inicializa pero no decodifica.

```bash
sudo apt install gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-libav
```

El AppImage ya los trae embebidos, así que este error solo aparece con
la ruta `.deb` o con `cargo tauri build` local.

### Audio sin vídeo (o al revés)

Suele ser el mismo problema de plugins — `bad` incluye HEVC/VP9, `libav`
cubre codecs propietarios. Instala los tres.

### "ffmpeg: command not found"

El transmux HLS lo hace `ffmpeg` en el host, no dentro del bundle
(sería demasiado peso). Si falta:

```bash
sudo apt install ffmpeg          # Debian/Ubuntu
sudo dnf install ffmpeg          # Fedora (requiere RPM Fusion)
sudo pacman -S ffmpeg            # Arch
```

### Tray / notificaciones no aparecen (GNOME)

GNOME 3.26+ eliminó el tray legacy. Instala la extensión
[AppIndicator Support](https://extensions.gnome.org/extension/615/appindicator-support/)
si quieres el icono de bandeja. La GUI funciona igual sin él.

### Wayland: cursor invisible / DnD no funciona

Bug histórico de WebKitGTK bajo Wayland en algunas versiones. Fuerza X11
al lanzar la app:

```bash
GDK_BACKEND=x11 videodrome
```

## Compilar desde código en Linux

Sistema Ubuntu 24.04 (o equivalente):

```bash
sudo apt update
sudo apt install -y \
  libwebkit2gtk-4.1-dev libgtk-3-dev libayatana-appindicator3-dev \
  librsvg2-dev libssl-dev patchelf file libfuse2 \
  gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-libav \
  ffmpeg

cd ui && npm ci && npm run build && cd ..
cargo tauri build --bundles deb,appimage --features gui
```

Artefactos en `target/<triple>/release/bundle/{deb,appimage}/`.

Para desarrollo (hot-reload):

```bash
cd ui && npm ci && cd ..
cargo tauri dev --features gui
```

## Rutas de sistema en Linux

| Uso | Ruta |
|---|---|
| Config, `.env`, `preferences.json`, tokens | `~/.config/videodrome/` |
| Caché de streams (`.ts`, `resume.json`) | `~/.cache/videodrome/streams/` |
| Logs (`debug.log`) | `~/.local/share/videodrome/` |

Se limpian desde la GUI en **Ajustes** individualmente o de golpe.
