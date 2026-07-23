# Audio Y Bluetooth En Gentoo-HP

Esta guía documenta la pila multimedia usada por Gentoo-HP, la causa del fallo
encontrado en la HP Pavilion 15-eh0xxx y las comprobaciones que permiten
distinguir un problema de kernel de uno de PipeWire o BlueZ.

## Hardware Verificado

La solución se comprobó con:

- AMD Ryzen 5 4500U;
- codec interno Realtek ALC287;
- controlador de micrófono AMD ACP/Renoir;
- audio HDMI/DisplayPort de AMDGPU;
- Intel Wi-Fi 6 AX200 con Bluetooth;
- kernel `6.18.39-gentoo-dist`;
- PipeWire 1.6.7;
- WirePlumber 0.5.15;
- BlueZ 5.86.

El kernel detectaba correctamente las tarjetas ALSA y el adaptador `hci0`. Los
fallos estaban en la configuración de espacio de usuario.

## Causa Del Audio Sin Salida

PipeWire estaba compilado con soporte Bluetooth, pero sin:

```text
sound-server
pipewire-alsa
```

Sin `sound-server`, el ebuild de Gentoo no instalaba el fragmento que activa
`hardware.audio` y `hardware.bluetooth` en el perfil principal de WirePlumber.
Por eso `wpctl status` solo mostraba:

```text
Dummy Output
```

ALSA sí detectaba el ALC287, pero las aplicaciones heredadas tampoco podían
entrar al grafo de PipeWire porque faltaba `pipewire-alsa`.

## Solución De Audio

El perfil instala explícitamente:

```text
media-libs/alsa-ucm-conf
media-sound/alsa-utils
media-video/pipewire
media-video/wireplumber
sys-auth/rtkit
kde-plasma/plasma-pa
```

Y aplica:

```text
media-video/pipewire bluetooth dbus pipewire-alsa sound-server systemd
sys-auth/rtkit systemd
```

`alsa-ucm-conf` aporta las rutas y perfiles del hardware. WirePlumber selecciona
el perfil y el puerto apropiados, mientras RTKit permite solicitar prioridad de
tiempo real mediante D-Bus para reducir cortes y variaciones de latencia.
`plasma-pa` añade el control de volumen y perfiles en KDE Plasma.

El archivo:

```text
/etc/wireplumber/wireplumber.conf.d/10-gentoo-hp-audio-bluetooth.conf
```

activa explícitamente:

```text
hardware.audio
hardware.bluetooth
```

No se fuerza el hardware a 96 kHz. PipeWire y WirePlumber conservan 48 kHz como
valor estable para el dispositivo y pueden negociar los formatos compatibles.
Forzar una frecuencia superior no mejora una fuente de menor resolución y
puede aumentar consumo o incompatibilidades.

## Causa De Bluetooth Sin Dispositivos

El módulo `btusb`, el firmware Intel y `hci0` estaban cargados, sin bloqueo
físico ni bloqueo por RFKill. El problema era:

```text
bluetooth.service: disabled, inactive
```

Sin `bluetoothd`, Bluedevil y `bluetoothctl` no tienen un servicio BlueZ al que
consultar y no pueden descubrir ni emparejar dispositivos.

## Solución De Bluetooth

El perfil instala explícitamente:

```text
net-wireless/bluez
kde-plasma/bluedevil
```

BlueZ se compila con `obex`, `readline`, `systemd` y `udev`. Al terminar la
instalación se habilita:

```bash
systemctl enable bluetooth.service
```

`AutoEnable` vale `true` de forma predeterminada en BlueZ, por lo que el
controlador se enciende cuando aparece. WirePlumber utiliza todos los codecs
Bluetooth disponibles por defecto y negocia automáticamente A2DP o el perfil de
llamadas apropiado. El build de PipeWire para este perfil incluye, cuando las
dependencias del ebuild están disponibles, SBC/SBC-XQ, AAC, aptX, LDAC, LC3 y
Opus.

## Servicios De Usuario

Para todos los usuarios nuevos se habilitan globalmente:

```text
pipewire.socket
pipewire-pulse.socket
wireplumber.service
```

`pipewire-pulse` ofrece compatibilidad con aplicaciones que utilizan el
protocolo de PulseAudio; no instala ni ejecuta el daemon PulseAudio tradicional.

## Comprobaciones

Audio:

```bash
wpctl status
speaker-test -c 2 -t wav -l 1
```

La salida esperada debe incluir el altavoz interno como sink predeterminado y la
prueba debe completar `Front Left` y `Front Right`.

Servicios:

```bash
systemctl --user status pipewire pipewire-pulse wireplumber
systemctl status bluetooth
```

Bluetooth:

```bash
rfkill list
bluetoothctl show
bluetoothctl scan on
```

El controlador debe indicar `Powered: yes`, `Pairable: yes` y cambiar
temporalmente a `Discovering: yes`.

## Reparación De Una Instalación Existente

Configura Portage:

```text
# /etc/portage/package.use/gentoo-hp-audio-bluetooth
media-video/pipewire bluetooth dbus pipewire-alsa sound-server systemd
net-wireless/bluez obex readline systemd udev
sys-auth/rtkit systemd
```

Instala o reconstruye la pila:

```bash
sudo emerge --ask --verbose --newuse \
    media-video/pipewire media-video/wireplumber \
    sys-auth/rtkit kde-plasma/plasma-pa \
    net-wireless/bluez kde-plasma/bluedevil
```

Habilita Bluetooth:

```bash
sudo systemctl enable --now bluetooth.service
```

Después cierra y vuelve a abrir la sesión gráfica, o reinicia los servicios del
usuario:

```bash
systemctl --user restart pipewire pipewire-pulse wireplumber
```

No ejecutes `sudo systemctl --user`: los servicios multimedia pertenecen al
usuario de la sesión, no a `root`.
