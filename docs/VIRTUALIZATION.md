# Virtualización Con QEMU/KVM Y VirtualBox

Gentoo-HP instala dos alternativas de virtualización de escritorio:

- QEMU/KVM con libvirt y Virt-Manager como opción recomendada;
- VirtualBox como alternativa para máquinas o flujos que dependan de su
  formato e interfaz.

Ambas pueden estar instaladas al mismo tiempo, pero no deben ejecutar máquinas
virtuales simultáneamente porque compiten por AMD-V.

## Hardware Y Kernel

La HP Pavilion 15-eh0xxx probada incorpora un Ryzen 5 4500U con AMD-V (`svm`).
El kernel `gentoo-kernel` utilizado por el perfil contiene como módulos:

```text
CONFIG_KVM
CONFIG_KVM_AMD
CONFIG_VHOST_NET
CONFIG_TUN
CONFIG_BRIDGE
CONFIG_VFIO
CONFIG_VFIO_PCI
```

El instalador carga al arrancar:

```text
kvm
kvm_amd
vhost_net
tun
```

Después del arranque `/dev/kvm` debe existir. El usuario normal pertenece a los
grupos `kvm`, `libvirt` y `vboxusers`.

## QEMU/KVM

El backend principal incluye:

```text
app-emulation/qemu
app-emulation/libvirt
app-emulation/virt-manager
app-emulation/virt-viewer
```

QEMU se compila únicamente con el target de sistema `x86_64` y con:

- aceleración KVM y Vhost-Net;
- audio nativo PipeWire y compatibilidad PulseAudio;
- interfaces GTK, Wayland y X11;
- gráficos OpenGL/VirGL, SPICE y VNC;
- red Slirp, TAP y USB redirection;
- VirtIOFS para compartir directorios;
- Seccomp, I/O asíncrono e io_uring.

Libvirt añade TPM virtual mediante `swtpm`, firmware UEFI mediante EDK2, red NAT
con dnsmasq y soporte VirtIOFS. Virt-Manager proporciona la interfaz gráfica.

El perfil habilita:

```text
libvirtd.service
libvirtd.socket
libvirtd-ro.socket
virtlockd.socket
virtlogd.socket
libvirt-guests.service
```

La red `default` de libvirt queda marcada para inicio automático y
`net.ipv4.ip_forward=1` permite el tráfico NAT de los invitados.

### Crear Una Máquina Con Virt-Manager

Abre:

```bash
virt-manager
```

La conexión recomendada es:

```text
QEMU/KVM - system
```

Después:

1. Selecciona una ISO.
2. Asigna normalmente 2 o 4 vCPU.
3. En esta laptop conviene comenzar con 4 GiB de RAM.
4. Usa almacenamiento QCOW2.
5. Conserva la red virtual `default` para obtener NAT.
6. Para invitados modernos usa firmware UEFI y dispositivos VirtIO.

Para comprobar el host:

```bash
test -c /dev/kvm
virsh -c qemu:///system list --all
virsh -c qemu:///system net-list --all
```

## VirtualBox

El perfil instala:

```text
app-emulation/virtualbox
app-emulation/virtualbox-modules
```

El segundo paquete es una dependencia exacta del primero y compila `vboxdrv`,
`vboxnetflt` y `vboxnetadp` para el kernel instalado. VirtualBox utiliza su GUI
Qt y envía audio por la compatibilidad PulseAudio proporcionada por PipeWire.

La combinación validada es VirtualBox 7.2.8 con kernel 6.18. No fuerces
versiones distintas entre `virtualbox` y `virtualbox-modules`; Portage mantiene
esa igualdad mediante una dependencia exacta.

Para abrirlo:

```bash
virtualbox
```

No se instala el Oracle Extension Pack porque tiene una licencia distinta y
requiere aceptación explícita. VirtualBox básico, discos virtuales, NAT y la
interfaz gráfica funcionan sin ese paquete.

Cuando se actualice el kernel, Portage debe reconstruir
`app-emulation/virtualbox-modules` para la nueva versión antes de iniciar una
máquina VirtualBox.

## Convivencia Entre KVM Y VirtualBox

Desde Linux 6.12, KVM puede reservar AMD-V al cargar el módulo. El ebuild de
Gentoo para `virtualbox-modules` instala automáticamente:

```text
options kvm enable_virt_at_load=0
```

Esto hace que KVM adquiera la virtualización al crear una VM y la libere cuando
deja de haber VMs. Reduce el conflicto con VirtualBox, pero sigue siendo
necesario cerrar todas las máquinas de un backend antes de iniciar el otro.

Si VirtualBox informa que KVM mantiene ocupado AMD-V:

```bash
virsh -c qemu:///system list
sudo modprobe -r kvm_amd kvm
sudo modprobe vboxdrv vboxnetflt vboxnetadp
```

No descargues módulos mientras una máquina QEMU/KVM esté ejecutándose.

Para volver a QEMU/KVM:

```bash
sudo modprobe -r vboxnetadp vboxnetflt vboxdrv
sudo modprobe kvm_amd
sudo modprobe vhost_net
```

## Diagnóstico

Comprueba AMD-V y KVM:

```bash
lscpu | grep Virtualization
ls -l /dev/kvm
lsmod | grep -E 'kvm|vbox'
```

Comprueba libvirt:

```bash
systemctl status libvirtd
virsh -c qemu:///system uri
virsh -c qemu:///system net-list --all
```

Comprueba VirtualBox:

```bash
VBoxManage --version
modinfo vboxdrv
```

Después de añadir o cambiar grupos es necesario cerrar y volver a iniciar la
sesión para que KDE reciba la nueva pertenencia.
