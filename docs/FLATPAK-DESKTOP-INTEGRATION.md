# Integración Visual De Flatpak Con KDE

Gentoo-HP configura Flatpak para que sus aplicaciones reciban las fuentes,
iconos, cursores, modo claro u oscuro y estilo GTK de la sesión Plasma sin
copiar archivos dentro de cada sandbox ni conceder acceso completo al
directorio personal.

## Componentes

El perfil instala estos paquetes:

```text
kde-plasma/breeze-gtk
kde-plasma/kde-gtk-config
kde-plasma/xdg-desktop-portal-kde
sys-apps/xdg-desktop-portal
sys-apps/xdg-desktop-portal-gtk
sys-apps/flatpak
```

También instala desde Flathub la extensión:

```text
org.gtk.Gtk3theme.Breeze//3.22
```

`xdg-desktop-portal-kde` continúa siendo el backend principal en Plasma.
`xdg-desktop-portal-gtk` complementa la interfaz de preferencias para
aplicaciones GTK.

## Fuentes

Flatpak expone automáticamente las fuentes del anfitrión dentro de cada
sandbox:

| Ubicación del anfitrión | Ubicación dentro del sandbox |
| --- | --- |
| `/usr/share/fonts` | `/run/host/fonts` |
| `/usr/local/share/fonts` | `/run/host/local-fonts` |
| `$XDG_DATA_HOME/fonts` | `/run/host/user-fonts` |

Por ello no se usan overrides como `--filesystem=home`, no se duplica el
contenido de `/usr/share/fonts` y no se copian Noto CJK o Noto Color Emoji
dentro de cada aplicación.

El instalador reconstruye la caché de Fontconfig después de configurar Noto.
Cuando el usuario instala una fuente nueva en `~/.local/share/fonts`, puede
actualizar la caché con:

```bash
fc-cache -f
```

Después debe cerrar completamente y volver a abrir la aplicación Flatpak.

## Tema, Iconos Y Tipografía De KDE

`kde-plasma/kde-gtk-config` instala un módulo de KDED que se carga
automáticamente al iniciar Plasma. El módulo vigila la configuración de KDE y
sincroniza:

- fuente de interfaz y fuente monoespaciada;
- tema de iconos;
- tema y tamaño del cursor;
- preferencia de modo claro u oscuro;
- animaciones y escalado de texto;
- colores y decoración GTK cuando se utiliza Breeze.

En Wayland usa GSettings. En X11 también actualiza XSettings y los archivos de
configuración GTK. Esto permite que los cambios realizados desde Preferencias
del sistema se propaguen sin fijar permanentemente la variable `GTK_THEME`.

La extensión `org.gtk.Gtk3theme.Breeze` hace que el tema exista dentro de los
runtimes Flatpak. Tener solamente `/usr/share/themes/Breeze` en el anfitrión no
es suficiente porque `/usr` pertenece al runtime dentro del sandbox.

Para que los colores de acento y la decoración generada por
`kde-gtk-config` también estén disponibles, Gentoo-HP aplica este override
global del usuario:

```bash
flatpak override --user --filesystem=xdg-config/gtk-3.0:ro
```

El permiso está limitado a `~/.config/gtk-3.0` y es de solo lectura. No expone
el resto de `~/.config` ni el directorio personal completo. Como consecuencia,
cualquier marcador o servidor GTK que el usuario guarde en ese mismo
directorio también será legible por sus aplicaciones Flatpak.

## Comprobación Con Brave

Brave de Flathub utiliza este identificador:

```text
com.brave.Browser
```

Comprueba que las fuentes del sistema se resuelven dentro del sandbox:

```bash
flatpak run --command=sh com.brave.Browser -c \
    'fc-match sans-serif; fc-match serif; fc-match monospace; fc-match emoji'
```

Para una familia concreta:

```bash
flatpak run --command=fc-match com.brave.Browser "Inter"
flatpak run --command=fc-match com.brave.Browser "JetBrains Mono"
```

Comprueba el tema instalado y las preferencias sincronizadas:

```bash
flatpak list --runtime | grep org.gtk.Gtk3theme.Breeze
flatpak override --user --show
gsettings get org.gnome.desktop.interface gtk-theme
gsettings get org.gnome.desktop.interface icon-theme
gsettings get org.gnome.desktop.interface font-name
gsettings get org.gnome.desktop.interface monospace-font-name
gsettings get org.gnome.desktop.interface color-scheme
```

Después abre Brave:

```bash
flatpak run com.brave.Browser
```

Si Brave estaba abierto durante el cambio, ciérralo desde su menú para detener
todos sus procesos y vuelve a iniciarlo. El estilo de la interfaz del navegador
y la tipografía elegida por cada sitio web son conceptos distintos: una página
puede seguir usando sus propias fuentes CSS aunque la familia del sistema esté
disponible en el sandbox.

## Diagnóstico

Comprueba los portales de la sesión:

```bash
systemctl --user status xdg-desktop-portal.service
busctl --user call \
    org.freedesktop.portal.Desktop \
    /org/freedesktop/portal/desktop \
    org.freedesktop.portal.Settings \
    Read ss org.freedesktop.appearance color-scheme
```

Comprueba que el módulo de sincronización GTK está cargado:

```bash
qdbus6 org.kde.kded6 /kded org.kde.kded6.loadedModules
```

Después de instalar los componentes en una sesión que ya estaba abierta, lo
más fiable es cerrar la sesión de Plasma y volver a entrar. En una instalación
nueva todo se carga desde el primer inicio de sesión.

GTK 4 y libadwaita pueden respetar el modo claro u oscuro sin reproducir todos
los colores o widgets de Breeze. Es una limitación del toolkit, no un problema
de acceso a las fuentes.
