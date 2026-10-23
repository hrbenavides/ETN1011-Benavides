# Laboratorio 2 — FRRouting en WSL/Ubuntu y equivalencias con Cisco IOS

## 1. Datos del entorno

**Estudiante:** Henry Rafael Benavides Gutierrez
**Asignatura:** ETN1011 — Laboratorio de Sistemas de Comunicación II
**Entorno:** Ubuntu sobre WSL2
**Usuario Linux:** `rafo`
**Kernel:** `6.18.54.1-microsoft-standard-WSL2`
**Distribución:** Ubuntu 26.04.1 LTS (Resolute Raccoon)
**FRRouting:** 10.5.1
**Herramientas principales:** `iproute2`, FRRouting, `vtysh`, Git, GitHub y Markdown.

Para identificar el entorno se ejecutaron:

```bash
whoami
uname -r
cat /etc/os-release
```

Los resultados confirmaron el usuario `rafo`, el kernel de WSL2 y la distribución Ubuntu 26.04.1 LTS. La versión de FRRouting se verificó con `sudo vtysh -c 'show version'`, obteniéndose FRRouting 10.5.1.

---

## 2. Objetivo

Instalar y configurar FRRouting sobre Ubuntu en WSL2, comprender su arquitectura, administrar rutas estáticas y observar las tablas de rutas desde la perspectiva de FRR y del kernel Linux, en un escenario de un único router.

El laboratorio se desarrolló siguiendo la guía `guia_laboratorio_frr_wsl_ubuntu_cisco.md`, complementada con el material `curso_frrouting_wsl_ubuntu_un_router.md`. Se trabajó con la interfaz de laboratorio `dummy0` y una ruta `blackhole`, sin modificar `eth0` ni la ruta por defecto de WSL.

---

## 3. Topología

```text
        Windows / WSL2
              |
       interfaz eth0 (WSL)
              |
      +-----------------+
      | R1 = Ubuntu+FRR |
      | único router    |
      +-----------------+
              |
       interfaz dummy0
       10.10.10.1/24
       red de prueba local
```

La interfaz `dummy0` simula una red conectada sin requerir una segunda máquina. No representa una LAN externa real.

| Elemento | Valor |
| --- | --- |
| Interfaz de acceso a Internet | `eth0` (no modificada) |
| Interfaz de laboratorio | `dummy0` |
| Dirección de `dummy0` | `10.10.10.1/24` |
| Ruta estática de práctica | `198.51.100.0/24 blackhole` |
| Prefijo reservado para documentación | `198.51.100.0/24` |

---

## 4. Fundamentos: FRR y el kernel Linux

GNU/Linux dispone de una pila IP y una tabla de rutas en el kernel. Para intercambiar rutas dinámicamente se utilizan suites de protocolos en espacio de usuario. La evolución histórica es:

```text
GNU Zebra (finales de los años 1990)
   └── Quagga (fork de Zebra, 2002)
         └── FRRouting / FRR (fork de Quagga, 2017)
```

FRR separa responsabilidades en daemons:

| Componente | Función |
| --- | --- |
| `zebra` | Coordina información de interfaces y rutas con el kernel |
| `staticd` | Administra rutas estáticas de FRR |
| `ospfd` | Implementa OSPFv2 (IPv4) |
| `bgpd` | Implementa BGP |
| `vtysh` | Consola integrada de administración |
| `watchfrr` | Supervisa los procesos de FRR |
| `mgmtd` | Daemon de gestión introducido en versiones recientes de FRR |

El **plano de control** (cálculo y administración de rutas) corresponde a FRR. El **plano de datos** (reenvío real de paquetes y tabla del kernel) corresponde a Linux. FRR no sustituye la configuración de interfaces, direcciones IP ni firewall de Ubuntu.

---

## 5. Construcción y configuración

### 5.1. Verificación de interfaces y rutas iniciales

Antes de instalar FRR se verificó el estado de red del entorno:

```bash
ip -br addr
ip -4 route
```

Se observaron las interfaces `lo` y `eth0`, con `eth0` en estado `UP` y dirección `172.24.235.74/20`. La ruta por defecto era:

```text
default via 172.24.224.1 dev eth0 proto kernel onlink
```

y la red conectada:

```text
172.24.224.0/20 dev eth0 proto kernel scope link src 172.24.235.74
```

Esta información se conservó como referencia y **no se modificó** durante el laboratorio.

### 5.2. Instalación de FRRouting

Se instalaron los paquetes requeridos:

```bash
sudo apt update
sudo apt install -y frr frr-pythontools iproute2 iputils-ping
```

Se verificó que `systemd` está activo como PID 1:

```bash
ps -p 1 -o comm=
```

El resultado fue `systemd`, por lo que se utilizó `systemctl` para administrar el servicio. Se habilitó y arrancó FRR:

```bash
sudo systemctl enable --now frr
sudo systemctl status frr --no-pager
```

El servicio quedó `active (running)`, con los daemons `mgmtd`, `zebra` y `staticd` en estado `up`.

### 5.3. Daemons activos

Se inspeccionó `/etc/frr/daemons`:

```bash
sudo cat -n /etc/frr/daemons
sudo grep -nE '(zebra|staticd|ospfd|bgpd)' /etc/frr/daemons
```

El contenido relevante fue:

```text
17  bgpd=no
18  ospfd=no
42  zebra_options="  -A 127.0.0.1 -s 90000000"
58  staticd_options="-A 127.0.0.1"
```

**Observación técnica:** en FRR 10.5.1 (Ubuntu 26.04) el archivo `/etc/frr/daemons` ya no contiene las líneas `zebra=yes` ni `staticd=yes`. Según el comentario de la línea 15, `watchfrr`, `zebra` y `staticd` **se inician siempre**. Los daemons `ospfd` y `bgpd` sí se controlan explícitamente con `=no`. El servicio arranca además con `mgmtd`, un daemon de gestión nuevo que no figura en la guía. Esto no afecta el objetivo del ejercicio, pero sí el formato esperado del archivo.

### 5.4. Creación de la interfaz `dummy0`

Se creó la interfaz de laboratorio y se le asignó dirección:

```bash
sudo ip link add dummy0 type dummy
sudo ip addr add 10.10.10.1/24 dev dummy0
sudo ip link set dummy0 up
```

La interfaz quedó operativa:

```text
dummy0           UNKNOWN        10.10.10.1/24 fe80::646b:4eff:fe3a:102f/64
```

Linux creó automáticamente la ruta conectada:

```text
10.10.10.0/24 dev dummy0 proto kernel scope link src 10.10.10.1
```

### 5.5. Ruta estática de descarte (`blackhole`)

Se configuró una ruta `blackhole` para el prefijo reservado para documentación `198.51.100.0/24`:

```text
sudo vtysh
configure terminal
ip route 198.51.100.0/24 blackhole
end
```

FRR instaló la ruta y la propagó al kernel. Se verificó desde ambas perspectivas:

Desde FRR:

```text
Routing entry for 198.51.100.0/24
  Known via "static", distance 1, metric 0, best
  Status: Installed
  * unreachable (blackhole), weight 1
```

Desde el kernel Linux:

```text
blackhole 198.51.100.0/24 proto static metric 20
```

La configuración activa de FRR incluyó la ruta:

```text
ip route 198.51.100.0/24 blackhole
```

### 5.6. Persistencia con `write memory`

Se guardó la configuración:

```text
write memory
```

La salida confirmó:

```text
Integrated configuration saved to /etc/frr/frr.conf
[OK]
```

Se comprobó el archivo:

```bash
sudo grep -n "198.51.100.0" /etc/frr/frr.conf
```

Resultado:

```text
8:ip route 198.51.100.0/24 blackhole
```

**Observación:** antes de ejecutar `write memory`, la ruta **no** estaba en `/etc/frr/frr.conf`. Esto confirma que FRR 10.5.1 con `service integrated-vtysh-config` **no persiste automáticamente** los cambios; `write memory` es necesario.

### 5.7. Eliminación y restauración de la ruta

Se eliminó la ruta:

```text
configure terminal
no ip route 198.51.100.0/24 blackhole
end
```

Tras la eliminación, `show ip route 198.51.100.0/24` devolvió `% Network not in table` y `ip -4 route show 198.51.100.0/24` quedó vacío. Esto demuestra que eliminar la ruta en FRR la quita también del kernel.

Posteriormente se restauró:

```text
configure terminal
ip route 198.51.100.0/24 blackhole
end
write memory
```

La ruta volvió a aparecer tanto en FRR como en el kernel, y se persistió de nuevo en `/etc/frr/frr.conf`.

---

## 6. Verificación

### 6.1. Interfaces

La interfaz `dummy0` se verificó desde Linux y desde FRR.

Desde Linux `[Bash]`:

```text
3: dummy0: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN
    link/ether 66:6b:4e:3a:10:2f brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.1/24 scope global dummy0
```

Desde FRR `[vtysh]`:

```text
Interface dummy0 is up, line protocol is up
  index 3 metric 0 mtu 1500
  flags: <UP,LOWER_UP,BROADCAST,RUNNING,NOARP>
  inet 10.10.10.1/24
  Interface Type dummy
```

La interfaz `eth0` se mantuvo intacta con `172.24.235.74/20`, y `lo` en estado `UP`.

### 6.2. Rutas

La tabla de rutas de FRR `[vtysh]` mostró:

```text
IPv4 unicast VRF default:
K>* 0.0.0.0/0 [0/0] via 172.24.224.1, eth0 onlink
C>* 10.10.10.0/24 is directly connected, dummy0
L>* 10.10.10.1/32 is directly connected, dummy0
C>* 172.24.224.0/20 is directly connected, eth0
L>* 172.24.235.74/32 is directly connected, eth0
S>* 198.51.100.0/24 [1/0] unreachable (blackhole)
```

La ruta conectada de `dummy0` se verificó desde Linux `[Bash]`:

```bash
ip -4 route show dev dummy0
```

Resultado:

```text
10.10.10.0/24 proto kernel scope link src 10.10.10.1
```

### 6.3. Comparación FRR frente a kernel

| Consulta | FRR (`vtysh`) | Kernel Linux (`ip`) |
| --- | --- | --- |
| Ruta `198.51.100.0/24` | `S>* 198.51.100.0/24 unreachable (blackhole)` | `blackhole 198.51.100.0/24 proto static metric 20` |
| Ruta conectada `dummy0` | `C>* 10.10.10.0/24 is directly connected, dummy0` | `10.10.10.0/24 dev dummy0 proto kernel scope link src 10.10.10.1` |
| Ruta por defecto | `K>* 0.0.0.0/0 via 172.24.224.1, eth0 onlink` | `default via 172.24.224.1 dev eth0 proto kernel onlink` |

La comparación muestra que FRR calcula la ruta (plano de control) y el kernel la instala y la utiliza para el reenvío (plano de datos). Ambas vistas coinciden.

### 6.4. Intento de verificación del descarte

Se intentó verificar el descarte con `ip route get` y `ping` `[Bash]`:

```bash
ip route get 198.51.100.10
ping -c 2 -W 2 198.51.100.10
```

Ambos comandos devolvieron error en este entorno:

```text
RTNETLINK answers: Invalid argument
ping: connect: Invalid argument
```

No se determinó la causa exacta de este error. Se documenta como una limitación observada en WSL2 para estas consultas. Tampoco se hicieron pruebas en un Linux nativo para compararlo.

La ruta blackhole sí se verificó por otras vías `[Bash]`:

```bash
ip route list | grep 198.51.100.0
ip -d route show 198.51.100.0/24
```

Resultados:

```text
blackhole 198.51.100.0/24 proto static metric 20
blackhole 198.51.100.0/24 proto static scope global metric 20
```

Y desde FRR `[vtysh]`: `show ip route 198.51.100.0/24` muestra la ruta instalada.

Traceroute y tracepath también fallaron. Por eso el descarte se confirma con la tabla de rutas y no con una prueba de tráfico.

### 6.5. Contadores de interfaz

Se consultaron los contadores de `dummy0` `[Bash]`:

```bash
ip -s link show dummy0
```

Resultado:

```text
RX:  bytes packets errors dropped  missed   mcast
         0       0      0       0       0       0
TX:  bytes packets errors dropped carrier collsns
      1400      20      0       0       0       0
```

Los contadores RX permanecen en cero porque ninguna fuente externa envía tráfico a `dummy0`. El origen de los 20 paquetes TX no se identificó durante el laboratorio y no se atribuye aquí a ninguna prueba concreta.

### 6.6. Logs y diagnóstico

Se consultaron los logs del servicio `[Bash]`:

```bash
systemctl status frr --no-pager
sudo journalctl -u frr -n 50 --no-pager
```

El servicio se mantuvo `active (running)` durante todo el laboratorio. Los logs mostraron el evento de arranque (`all daemons up, doing startup-complete notify`) y la escritura de configuración (`configuration write completed with exit code 0`).

En el registro revisado no aparecieron entradas de los cambios individuales de rutas. Esto se observó en esta instalación con el nivel de log por defecto; no se probó si un nivel de log mayor los registra.

---

## 7. OSPF sin vecinos

Como último ejercicio se habilitó `ospfd` y se configuró OSPF en `dummy0`, para comprobar que un único router no forma adyacencias.

Se cambió `ospfd=no` a `ospfd=yes` en `/etc/frr/daemons`:

```bash
sudo sed -i 's/^ospfd=no/ospfd=yes/' /etc/frr/daemons
sudo systemctl restart frr
```

Tras el reinicio, el servicio corrió con `mgmtd zebra ospfd staticd`.

Se configuró OSPF:

```text
configure terminal
router ospf
 network 10.10.10.0/24 area 0
end
```

Se verificó la interfaz OSPF:

```text
dummy0 is up
  Internet Address 10.10.10.1/24, Broadcast 10.10.10.255, Area 0.0.0.0
  Router ID 172.24.235.74, Network Type BROADCAST, Cost: 10
  State Waiting, Priority 1
  Neighbor Count is 0, Adjacent neighbor count is 0
```

La tabla de vecinos quedó vacía:

```text
Neighbor ID     Pri State           Up Time         Dead Time Address         Interface
```

Y el resumen de OSPF confirmó:

```text
OSPF Routing Process, Router ID: 172.24.235.74
Number of areas attached to this router: 1
Area ID: 0.0.0.0 (Backbone)
  Number of fully adjacent neighbors in this area: 0
```

**Observación:** el Router ID de OSPF es `172.24.235.74` (la dirección de `eth0`), no la de `dummy0`. FRR elige el Router ID más alto entre las interfaces activas al arrancar OSPF. Además, con `ospfd` desactivado, `show ip ospf neighbor` devuelve `ospfd is not running`, no una tabla vacía, lo cual es más preciso que lo que asume la guía.

La configuración se persistió con `write memory`, y el `running-config` final incluyó:

```text
ip route 198.51.100.0/24 blackhole
!
router ospf
 network 10.10.10.0/24 area 0
exit
```

---

## 8. Comparación FRR vs Cisco IOS

| Operación | FRR / Linux | Cisco IOS clásico |
| --- | --- | --- |
| Ver interfaces | `ip -br addr` / `show interface` | `show ip interface brief` |
| Ver rutas | `ip -4 route` / `show ip route` | `show ip route` |
| Entrar a CLI | `sudo vtysh` | `enable` |
| Configurar | `configure terminal` | `configure terminal` |
| IP de interfaz | `ip addr add ... dev ...` | `interface ...` + `ip address ...` |
| Activar interfaz | `ip link set ... up` | `no shutdown` |
| Ruta estática | `ip route PREFIJO NEXT_HOP` | `ip route RED MÁSCARA NEXT_HOP` |
| Ruta de descarte | `blackhole` | `Null0` |
| Ver configuración | `show running-config` | `show running-config` |
| Guardar | `write memory` | `copy running-config startup-config` |
| Logs | `journalctl -u frr` | `show logging` |
| OSPF vecinos | `show ip ospf neighbor` | `show ip ospf neighbor` |

La diferencia de fondo es que Cisco IOS integra interfaces, direcciones y protocolos en un mismo sistema operativo de red, mientras que FRR se apoya en Linux: la administración de interfaces, reenvío y firewall pertenece al sistema operativo, y FRR administra los protocolos y las rutas.

---

## 9. Uso de IA

Durante el desarrollo del laboratorio se utilizó una herramienta de IA como apoyo para comprender conceptos de FRRouting, interpretar salidas de comandos y diagnosticar comportamientos inesperados.

### Problema

Fue necesario comprender la relación entre FRR y el kernel Linux, interpretar las tablas de rutas desde ambas perspectivas, diagnosticar errores de la pila de red de WSL2 y verificar la configuración de OSPF en un escenario de un solo router.

### Consulta

Se realizaron consultas orientadas a la explicación y al diagnóstico, no a la sustitución del trabajo práctico. Por ejemplo, se consultó por qué `ip route get` y `ping` devolvían `Invalid argument` en WSL2, y cómo verificar el descarte de paquetes por vía alternativa.

### Propuesta obtenida

La herramienta explicó que WSL2 rechaza ciertas operaciones de socket para destinos no enrutables, y propuso verificar la ruta mediante `ip route list`, `ip -d route show` y `show ip route`, en lugar de `ip route get` y `ping`.

### Verificación

Las propuestas fueron comprobadas directamente en el entorno WSL2. Se confirmó que la ruta `blackhole` estaba correctamente instalada en FRR y en el kernel, y que el fallo de `ip route get` y `ping` era una limitación de WSL2, no de la configuración.

### Explicación técnica

El uso de IA se mantuvo como apoyo para el análisis y la comprensión. Todos los resultados incluidos en este informe corresponden a pruebas ejecutadas directamente en el entorno de laboratorio y fueron interpretados a partir de las salidas observadas.

---

## 10. Respuestas a preguntas de evaluación

1. **¿Qué diferencia hay entre FRR y el kernel Linux?**
   FRR calcula y administra rutas (plano de control). El kernel Linux mantiene la tabla de rutas y realiza el reenvío de paquetes (plano de datos).

2. **¿Qué función cumple `zebra`?**
   Coordina la información de interfaces y rutas entre los daemons de FRR y el kernel Linux.

3. **¿Qué diferencia hay entre `show ip route` e `ip -4 route`?**
   `show ip route` muestra la vista de FRR (plano de control). `ip -4 route` muestra la vista del kernel Linux (plano de datos).

4. **¿Qué conserva `write memory`?**
   Guarda la configuración de FRR en `/etc/frr/frr.conf`. No conserva las interfaces creadas con `ip link`, que son temporales.

5. **¿Por qué no aparecen vecinos OSPF?**
   Porque no existe otro router OSPF conectado a la misma red. Un solo router no puede formar una adyacencia OSPF.

6. **¿Qué hace una ruta `blackhole`?**
   Descarta los paquetes destinados al prefijo, sin enviarlos a ninguna interfaz ni next-hop.

7. **¿Cuál es la equivalencia de `write memory` en Cisco IOS?**
   `copy running-config startup-config`.

8. **¿Por qué la configuración de una IP se hace fuera de `vtysh` en este laboratorio?**
   Porque en Linux la administración de interfaces y direcciones corresponde al sistema operativo (con `ip addr`), no a FRR. FRR administra protocolos y rutas, no interfaces.

---

## 11. Conclusiones

1. Se instaló y configuró FRRouting 10.5.1 sobre Ubuntu 26.04 en WSL2. Los daemons `zebra` y `staticd` estuvieron activos; `ospfd` y `bgpd` se controlaron explícitamente.

2. Se creó la interfaz `dummy0` con dirección `10.10.10.1/24`, y Linux generó automáticamente la ruta conectada `10.10.10.0/24`.

3. Se configuró una ruta estática `blackhole` para `198.51.100.0/24`. La ruta aparece en FRR (`S>* ... unreachable (blackhole)`) y en el kernel (`blackhole ... proto static`).

4. Se verificó que `write memory` es necesario para persistir la configuración de FRR, y que eliminar una ruta en FRR la quita también del kernel.

5. `ip route get` y `ping` devolvieron `Invalid argument` en este entorno WSL2. La causa exacta no se determinó; la ruta se verificó con `ip route list`, `ip -d route show` y `show ip route`.

6. Se habilitó `ospfd` y se configuró OSPF en `dummy0`. El resultado fue cero vecinos (`Neighbor Count is 0`), lo esperado con un solo router.

7. El Router ID de OSPF se tomó de `eth0` (`172.24.235.74`), porque por defecto FRR usa la dirección IP más alta de sus interfaces activas.

8. El laboratorio permitió distinguir el plano de control (FRR) del plano de datos (kernel Linux) y relacionar la sintaxis de FRR con la de Cisco IOS.