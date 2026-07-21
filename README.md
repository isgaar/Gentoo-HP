# Gentoo-HP

Perfil de instalacion de Gentoo para una **HP Pavilion Laptop 15-eh0xxx** como la de Ismael.

Esta version parte de `oddlama/gentoo-install`, pero ya viene ajustada para este hardware:

- CPU AMD Ryzen 5 4500U / Renoir, usando `-march=znver2`
- GPU AMD Radeon Vega integrada, usando `VIDEO_CARDS="amdgpu radeonsi"`
- Wi-Fi Intel Wi-Fi 6 AX200
- Disco NVMe
- Gentoo `amd64` con `systemd`
- Kernel compilado localmente con `sys-kernel/gentoo-kernel`
- Root en Btrfs con LUKS
- NetworkManager + iwd para Wi-Fi
- `power-profiles-daemon` e `irqbalance` para portatil
- Instalacion automatica al unico NVMe detectado
- Arranque UEFI obligatorio, sin modo BIOS

## Advertencia Importante

La configuracion incluida esta pensada para instalar Gentoo usando un disco completo.

El modo por defecto es:

```bash
TARGET_DISK="auto-nvme"
```

Eso significa que el instalador buscara automaticamente un disco `/dev/nvme*n*` y lo usara como destino. Para evitar accidentes, solo continua si encuentra exactamente un NVMe. Si encuentra cero o mas de uno, se detiene.

Si ejecutas esto antes de cambiar el NVMe, y el unico NVMe es el de Fedora, el instalador va a borrar Fedora. Cambia fisicamente al NVMe nuevo antes de correr `./install`.

El perfil es UEFI-only. Si arrancas el USB en modo legacy/BIOS, aborta.

El archivo `gentoo.conf` ya viene desbloqueado para el modo automatico:

```bash
I_HAVE_READ_AND_EDITED_THE_CONFIG_PROPERLY=true
```

Aunque este desbloqueado, el instalador todavia muestra el layout antes de particionar. Confirma solo si el NVMe mostrado es el correcto.

## Ya Tengo El USB LiveGUI, Ahora Que Hago

Si Fedora Media Writer ya termino de grabar `livegui-amd64-...iso`, no necesitas montar el USB desde Fedora. Lo que sigue es reiniciar y arrancar desde ese USB.

1. Reinicia la laptop.
2. En HP normalmente entra al menu de arranque con `Esc` y luego `F9`.
3. Elige el USB en modo UEFI.
4. Entra al entorno LiveGUI de Gentoo.
5. Conectate a internet desde la interfaz grafica o por cable.
6. Abre una terminal.

En la terminal entra como root:

```bash
sudo -i
```

Si ya estas como root, puedes seguir.

## Clonar Este Repositorio

Dentro del LiveGUI:

```bash
git clone https://github.com/isgaar/Gentoo-HP.git
cd Gentoo-HP
```

Si `git` no estuviera disponible en el ISO:

```bash
emerge --sync
emerge --ask dev-vcs/git
```

## Encontrar El Disco Correcto

Antes de instalar, identifica el disco. Si ya cambiaste el NVMe, deberia aparecer el nuevo disco como algo similar a:

```text
/dev/nvme0n1  MODELO_DEL_NVME_NUEVO
```

En el LiveGUI revisalo:

```bash
lsblk -o NAME,MODEL,SIZE,TYPE,FSTYPE,MOUNTPOINTS
ls -l /dev/disk/by-id/
```

Si solo hay un NVMe, no tienes que editar `TARGET_DISK`; el instalador lo detecta solo.

Si hay mas de un NVMe, el modo automatico se detiene. En ese caso edita `gentoo.conf` y cambia `TARGET_DISK="auto-nvme"` por el disco exacto, por ejemplo `TARGET_DISK="/dev/nvme0n1"`.

## Revisar La Configuracion

Normalmente ya no tienes que editar `gentoo.conf` para el disco. Aun asi, puedes revisarlo:

Abre `gentoo.conf`:

```bash
nano gentoo.conf
```

El modo rapido es:

```bash
TARGET_DISK="auto-nvme"
```

El perfil tambien viene listo para UEFI:

```bash
create_classic_single_disk_layout swap=16GiB type=efi luks=true root_fs=btrfs "$TARGET_DISK"
```

Revisa tambien:

```bash
KEYMAP="la-latin1"
TIMEZONE="America/Mexico_City"
LOCALE="es_MX.UTF-8"
```

Si tu contrasena de cifrado va a tener simbolos raros, conviene usar una frase larga con letras y numeros para evitar problemas de teclado en el arranque.

## Instalar

Puedes dejar que el instalador te pregunte la contrasena LUKS, o definirla antes:

```bash
export GENTOO_INSTALL_ENCRYPTION_KEY="una frase larga y segura"
```

Despues ejecuta:

```bash
./install
```

El instalador va a mostrar el layout de disco antes de hacer cambios. Lee esa pantalla con calma. Debe decir que usara el NVMe nuevo. Si ves el disco equivocado, cancela.

Cuando confirmes, hara en resumen:

1. Verificar que arrancaste en UEFI.
2. Detectar automaticamente el unico NVMe.
3. Particionar el NVMe en modo EFI.
4. Crear swap de 16 GiB.
5. Crear root Btrfs cifrado con LUKS.
6. Descargar y extraer stage3 `amd64-systemd`.
7. Configurar Portage para Ryzen 5 4500U y Radeon Vega.
8. Compilar kernel Gentoo desde fuente.
9. Instalar firmware, NetworkManager, iwd y herramientas del portatil.
10. Crear initramfs con soporte temprano para `amdgpu` y `nvme`.
11. Crear entrada EFI para arrancar Gentoo.

## Primer Arranque

Cuando termine:

```bash
reboot
```

Retira el USB o elige el disco interno desde el menu UEFI.

Al arrancar Gentoo te pedira la contrasena LUKS. Luego entraras a un sistema base, no a KDE completo. La idea es dejar una base optimizada para instalar despues el escritorio que quieras.

Para conectarte por Wi-Fi en consola:

```bash
nmtui
```

O con NetworkManager:

```bash
nmcli device wifi list
nmcli device wifi connect "NOMBRE_DE_TU_WIFI" password "TU_PASSWORD"
```

## Despues De Instalar

Actualiza el sistema:

```bash
emerge --sync
emerge --ask --verbose --update --deep --newuse @world
```

Comprueba las banderas de CPU:

```bash
cpuid2cpuflags
```

El perfil ya deja estas banderas configuradas:

```bash
aes avx avx2 bmi1 bmi2 f16c fma3 mmx mmxext pclmul popcnt rdrand sha sse sse2 sse3 sse4_1 sse4_2 sse4a ssse3
```

## Archivos Importantes

- `gentoo.conf`: perfil listo para la HP de Ismael.
- `gentoo.conf.example`: ejemplo general con las variables nuevas de Portage.
- `scripts/main.sh`: aplica las optimizaciones de hardware durante la instalacion.
- `configure`: conserva las variables nuevas si usas el configurador TUI.

## Notas Sobre El Proyecto Base

Este repositorio conserva el instalador original de `oddlama/gentoo-install` como base. El remoto original queda como `upstream`, y este perfil personalizado se publica en:

```text
https://github.com/isgaar/Gentoo-HP.git
```

Para ver ayuda del instalador:

```bash
./install --help
```
