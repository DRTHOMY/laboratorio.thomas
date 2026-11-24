# Reporte Técnico de Configuración de Laboratorio: Máquina de Prácticas

**Autor:** Thomas De Santis
**Fecha:** 24 de noviembre de 2026

## Transparencia sobre cómo se elaboró este reporte

Este reporte fue elaborado con asistencia de una IA (Claude, de Anthropic). Para dejar claro qué es evidencia real y qué es documentación:

| Sección | Estado |
|---------|--------|
| 1. Red aislada (VirtualBox) | **Guía técnica.** No ejecutada en este reporte; no incluye capturas. |
| 2. Capa Windows | **Guía técnica.** No ejecutada en este reporte; no incluye capturas. |
| 3. Capa Linux | **Ejecutada de verdad**, pero en un entorno Ubuntu 24.04 de pruebas (contenedor aislado), **no** en una VM de VirtualBox. Las salidas son reales y están en la carpeta [`evidencia/`](evidencia/). |
| 4. Snapshot | **Guía técnica.** No ejecutada en este reporte; no incluye capturas. |

En el entorno de pruebas el usuario de sesión era `root` y no existe `sudo`, por eso los comandos de la sección 3 se ejecutaron sin ese prefijo.

---

## Introducción

Un laboratorio seguro es un entorno aislado donde se puede practicar ciberseguridad sin poner en riesgo el equipo real (el *host*) ni la red doméstica o de la oficina. Este documento describe la configuración de una "Máquina de Prácticas" y explica, en cada capa, qué medida se aplica y por qué importa para la seguridad.

---

## 1. La fundación: VirtualBox y red aislada

**Configuración:** en *Configuración → Red → Adaptador 1* de la máquina virtual, el modo de conexión se establece en **NAT** (o, alternativamente, **Red Interna**) antes de encenderla por primera vez.

**Por qué NAT:**
- Con NAT, la VM sale a Internet a través del equipo anfitrión, pero **no es visible desde la red local**. Nadie en el Wi-Fi de la casa o la oficina puede iniciar una conexión hacia ella.
- Permite descargar actualizaciones y herramientas sin exponer la VM.
- Una **Red Interna** va un paso más allá: la VM solo se comunica con otras VMs de esa misma red, sin salida al exterior. Es útil para prácticas con malware o ataques entre máquinas virtuales.

**Cómo protege al host:** al estar aislada, un fallo o una herramienta peligrosa dentro de la VM no tiene un camino directo hacia el equipo real ni hacia el resto de los dispositivos de la red.

**Error a evitar: modo Bridge (Puente).** En ese modo la VM obtiene una IP de la red real y se vuelve un dispositivo "visible" y potencialmente vulnerable para cualquiera en esa red.

---

---

## 2. Capa Windows: usuarios y actualizaciones

### Usuario estándar separado del administrador

Para las prácticas se utiliza una cuenta de **Usuario Estándar**, distinta de la cuenta de Administrador. Se gestiona desde *Configuración → Cuentas → Otros usuarios* (o con `netplwiz`).

**Por qué importa:** es el **principio de mínimo privilegio**. Si una cuenta estándar es comprometida o ejecuta algo malicioso, el daño queda limitado, porque no puede instalar software del sistema, modificar configuraciones críticas ni deshabilitar protecciones. Trabajar siempre como Administrador es el error más común de un principiante, y suele hacerse "por comodidad".

### Sistema actualizado

Se revisa *Configuración → Windows Update* hasta que el sistema indique que está al día.

**Por qué importa:** las actualizaciones corrigen vulnerabilidades conocidas. Gran parte de los ataques reales aprovechan fallas que ya tenían parche disponible.

---

---

## 3. Capa Linux: permisos y gestión (ejecutada)

Entorno: Ubuntu 24.04.5 LTS (contenedor de pruebas).

### 3.1 Usuario sin privilegios

Se creó el usuario `practicas`, que **no** pertenece al grupo `sudo`:

```
$ cat /etc/os-release | head -2
PRETTY_NAME="Ubuntu 24.04.5 LTS"
NAME="Ubuntu"

$ adduser --disabled-password --gecos "" practicas
info: Creating home directory `/home/practicas' ...
info: Copying files from `/etc/skel' ...
info: Adding new user `practicas' to supplemental / extra groups `users' ...
info: Adding user `practicas' to group `users' ...

$ id practicas
uid=1002(practicas) gid=1002(practicas) groups=1002(practicas),100(users)

$ getent passwd practicas
practicas:x:1002:1002:,,,:/home/practicas:/bin/bash

$ groups practicas
practicas : practicas users
```

### 3.2 Permisos con `ls -l`

Se creó un archivo, se mostró su permiso por defecto y luego se restringió con `chmod 600` para que solo lo pueda leer su propietario:

```
$ runuser -u practicas -- sh -c "echo laboratorio seguro > ~/mi_archivo.txt"

$ ls -l /home/practicas/mi_archivo.txt   (permisos por defecto)
-rw-r--r-- 1 practicas practicas 19 Nov 24 19:20 /home/practicas/mi_archivo.txt

$ chmod 600 /home/practicas/mi_archivo.txt

$ ls -l /home/practicas/mi_archivo.txt   (solo el propietario)
-rw------- 1 practicas practicas 19 Nov 24 19:20 /home/practicas/mi_archivo.txt
```

La primera columna se lee así: el primer carácter es el tipo (`-` archivo), los tres siguientes son los permisos del propietario, los tres del medio los del grupo y los últimos tres los de otros usuarios. Pasar de `-rw-r--r--` a `-rw-------` quita el acceso de lectura a grupo y otros.

### 3.3 Búsqueda y aplicación de actualizaciones

```
$ apt update
Get:17 http://archive.ubuntu.com/ubuntu noble-backports/main amd64 Packages [49.0 kB]
Get:18 http://archive.ubuntu.com/ubuntu noble-backports/multiverse amd64 Packages [669 B]
Get:19 http://archive.ubuntu.com/ubuntu noble-backports/universe amd64 Packages [36.0 kB]
Fetched 34.2 MB in 2s (15.7 MB/s)
Reading package lists...
Building dependency tree...
Reading state information...
24 packages can be upgraded. Run 'apt list --upgradable' to see them.

$ apt list --upgradable 2>/dev/null | wc -l
25
```

```
$ apt upgrade -y   (últimas líneas)
Setting up systemd-sysv (255.4-1ubuntu8.19) ...
Setting up libpam-systemd:amd64 (255.4-1ubuntu8.19) ...
Processing triggers for dbus (1.14.10-4ubuntu4.1) ...
Processing triggers for libc-bin (2.39-0ubuntu8.12) ...
Processing triggers for ca-certificates-java (20240118ubuntu0.24.04.2) ...
done.

$ apt update
Building dependency tree...
Reading state information...
1 package can be upgraded. Run 'apt list --upgradable' to see it.
```

Tras el `apt upgrade` quedó **1 paquete pendiente** (queda sin actualizar; suele ocurrir con paquetes retenidos o que requieren dependencias nuevas). Por eso no puede afirmarse que el sistema esté completamente al día.

Mantener los paquetes actualizados cierra vulnerabilidades conocidas. Y se evita trabajar como `root`: en un sistema real se usa `sudo` solo cuando hace falta.

---

## 4. La red de seguridad: snapshot inicial

Una vez instalado el sistema, actualizado y con el usuario de trabajo creado, se apaga la máquina virtual y se toma una **instantánea** desde el Administrador de Instantáneas de VirtualBox, con el nombre **"Clean Install - Hardening applied"**.

**Por qué importa:** el snapshot funciona como una "máquina del tiempo". Si en una práctica algo sale mal (un malware de prueba, una configuración rota, un error de permisos), se vuelve al estado limpio y seguro en segundos, sin reinstalar todo.

---

---

## Conclusión

Un laboratorio bien armado combina cuatro ideas de seguridad: **aislamiento** (red NAT o interna), **mínimo privilegio** (usuarios estándar y sin `sudo`), **actualización constante** (Windows Update y `apt`) y **capacidad de recuperación** (snapshots). Aplicarlas desde el primer día es lo que diferencia un entorno de práctica seguro de uno que pone en riesgo el equipo real.

---

## Estructura del repositorio

```
.
├── README.md
└── evidencia/
    ├── 01_usuario_sin_privilegios.txt
    ├── 02_permisos_ls-l.txt
    ├── 03_apt_update.txt
    └── 04_apt_upgrade.txt
```
