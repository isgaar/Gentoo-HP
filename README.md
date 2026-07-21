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

## Advertencia Importante

La configuracion incluida esta pensada para instalar Gentoo usando un disco completo.

Si apuntas `TARGET_DISK` al NVMe donde esta Fedora, el instalador va a borrar Fedora y reparticionar ese disco. Haz respaldo antes. Si quieres dual boot o conservar particiones existentes, no uses este `gentoo.conf` tal cual.

Por seguridad, el archivo `gentoo.conf` esta bloqueado por defecto:

```bash
I_HAVE_READ_AND_EDITED_THE_CONFIG_PROPERLY=false
```

No cambies eso a `true` hasta revisar y corregir `TARGET_DISK`.

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

Antes de instalar, identifica el disco. En tu Fedora actual se vio como:

```text
/dev/nvme0n1  WD Green SN350 2TB
```

En el LiveGUI revisalo otra vez:

```bash
lsblk -o NAME,MODEL,SIZE,TYPE,FSTYPE,MOUNTPOINTS
ls -l /dev/disk/by-id/
```

Si aparece un nombre estable en `/dev/disk/by-id/`, usalo. Suele verse parecido a:

```text
/dev/disk/by-id/nvme-WD_Green_SN350_...
```

Si no aparece `/dev/disk/by-id/`, puedes usar `/dev/nvme0n1`, pero revisa dos veces con `lsblk`.

## Editar La Configuracion

Abre `gentoo.conf`:

```bash
nano gentoo.conf
```

Cambia esta linea:

```bash
TARGET_DISK="/dev/disk/by-id/REPLACE_ME_WITH_TARGET_NVME"
```

por el disco real. Ejemplo:

```bash
TARGET_DISK="/dev/disk/by-id/nvme-WD_Green_SN350_..."
```

Luego cambia al final:

```bash
I_HAVE_READ_AND_EDITED_THE_CONFIG_PROPERLY=true
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

El instalador va a mostrar el layout de disco antes de hacer cambios. Lee esa pantalla con calma. Si el disco no es el correcto, cancela.

Cuando confirmes, hara en resumen:

1. Particionar el disco en modo EFI.
2. Crear swap de 16 GiB.
3. Crear root Btrfs cifrado con LUKS.
4. Descargar y extraer stage3 `amd64-systemd`.
5. Configurar Portage para Ryzen 5 4500U y Radeon Vega.
6. Compilar kernel Gentoo desde fuente.
7. Instalar firmware, NetworkManager, iwd y herramientas del portatil.
8. Crear initramfs con soporte temprano para `amdgpu` y `nvme`.
9. Crear entrada EFI para arrancar Gentoo.

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
