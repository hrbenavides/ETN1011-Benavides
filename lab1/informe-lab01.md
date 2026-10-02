# Laboratorio 1 — Redes virtuales con Linux

## 1. Datos del entorno

**Estudiante:** Henry Rafael Benavides Gutierrez
**Asignatura:** ETN1011 — Laboratorio de Sistemas de Comunicación II
**Entorno:** Ubuntu sobre WSL2
**Usuario Linux:** `rafo`
**Kernel:** `6.18.40.1-microsoft-standard-WSL2`
**Herramientas principales:** `iproute2`, `tcpdump`, Git, GitHub y Markdown.

Para identificar el usuario y las características del sistema se utilizaron:

```bash
whoami
uname -a
cat /etc/os-release
```

El comando `whoami` permitió verificar que el usuario de trabajo es `rafo`. Mediante `uname -a` se comprobó que el sistema utiliza un kernel de Linux proporcionado por WSL2. Finalmente, `/etc/os-release` permitió identificar la distribución Ubuntu utilizada durante la práctica.

Para verificar las herramientas de networking se ejecutó:

```bash
ip -V
```

obteniéndose:

```text
ip utility, iproute2-6.19.0, libbpf 1.6.3
```

También se verificó previamente la capacidad de crear network namespaces mediante:

```bash
sudo ip netns add prueba
sudo ip netns list
sudo ip netns del prueba
```

La prueba fue exitosa, por lo que el entorno WSL2 permitió trabajar con los mecanismos de networking requeridos para el laboratorio.

---

## 2. Objetivo

Construir y verificar una red virtual aislada utilizando las capacidades de networking del kernel Linux.

Para ello se crearon dos `network namespaces`, `hostA` y `hostB`, conectados mediante un par de interfaces virtuales `veth`. Posteriormente se configuró direccionamiento IPv4, se verificaron interfaces, rutas y vecinos, se comprobó la conectividad mediante ICMP y se realizó una captura de tráfico con `tcpdump`.

Finalmente, se provocó una falla controlada sobre una de las interfaces para observar sus efectos y realizar un diagnóstico utilizando las herramientas de Linux.

---

## 3. Topología

La topología construida fue:

```text
                 Linux / WSL2

        ┌─────────────────┐
        │     hostA       │
        │ network namespace│
        │                 │
        │  10.10.1.1/30   │
        │      vethA      │
        └────────┬────────┘
                 │
                 │  veth
                 │
        ┌────────┴────────┐
        │      hostB      │
        │ network namespace│
        │                 │
        │  10.10.1.2/30   │
        │      vethB      │
        └─────────────────┘
```

Los elementos principales fueron:

| Elemento           | Valor          |
| ------------------ | -------------- |
| Namespace 1        | `hostA`        |
| Namespace 2        | `hostB`        |
| Interfaz de hostA  | `vethA`        |
| Interfaz de hostB  | `vethB`        |
| Dirección de hostA | `10.10.1.1/30` |
| Dirección de hostB | `10.10.1.2/30` |

El enlace entre `vethA` y `vethB` representa un enlace Ethernet virtual. Cada extremo fue colocado en un namespace diferente.

---

## 4. Direccionamiento

Se utilizó la red:

```text
10.10.1.0/30
```

Un prefijo `/30` utiliza 30 bits para identificar la red y deja 2 bits para hosts, por lo que existen cuatro direcciones:

| Dirección   | Función          |
| ----------- | ---------------- |
| `10.10.1.0` | Dirección de red |
| `10.10.1.1` | HostA            |
| `10.10.1.2` | HostB            |
| `10.10.1.3` | Broadcast        |

La máscara correspondiente es:

```text
255.255.255.252
```

Por lo tanto, las direcciones `10.10.1.1` y `10.10.1.2` pertenecen a la misma subred y pueden comunicarse directamente mediante el enlace virtual.

No fue necesario configurar un gateway, ya que ambos nodos pertenecen a la misma red directamente conectada.

---

## 5. Construcción

### Creación de los namespaces

Se crearon los dos espacios de red:

```bash
sudo ip netns add hostA
sudo ip netns add hostB
```

La existencia de ambos namespaces se verificó mediante:

```bash
sudo ip netns list
```

El resultado mostró:

```text
hostB (id: 1)
hostA (id: 0)
```

Un `network namespace` permite disponer de un espacio de red aislado, incluyendo sus propias interfaces, direcciones, rutas y tabla de vecinos.

### Creación del enlace virtual

Se creó un par `veth`:

```bash
sudo ip link add vethA type veth peer name vethB
```

Un par `veth` funciona como un cable Ethernet virtual: los paquetes que ingresan por un extremo aparecen en el otro.

Posteriormente se trasladó cada extremo al namespace correspondiente:

```bash
sudo ip link set vethA netns hostA
sudo ip link set vethB netns hostB
```

La existencia de las interfaces dentro de cada namespace se comprobó mediante:

```bash
sudo ip netns exec hostA ip link
sudo ip netns exec hostB ip link
```

En `hostA` se observó `vethA@if3` y en `hostB` se observó `vethB@if4`.

Además, cada interfaz mostraba una referencia `link-netns` hacia el namespace donde se encontraba su extremo opuesto, confirmando la relación entre ambas interfaces.

### Configuración de direcciones

Se asignaron las direcciones IPv4 indicadas en la topología:

```bash
sudo ip netns exec hostA ip addr add 10.10.1.1/30 dev vethA
sudo ip netns exec hostB ip addr add 10.10.1.2/30 dev vethB
```

La configuración se verificó mediante:

```bash
sudo ip netns exec hostA ip addr
sudo ip netns exec hostB ip addr
```

En `hostA` se observó:

```text
inet 10.10.1.1/30
```

y en `hostB`:

```text
inet 10.10.1.2/30
```

### Puesta en servicio de las interfaces

Se habilitaron las interfaces:

```bash
sudo ip netns exec hostA ip link set lo up
sudo ip netns exec hostA ip link set vethA up

sudo ip netns exec hostB ip link set lo up
sudo ip netns exec hostB ip link set vethB up
```

Al verificarlas nuevamente mediante `ip link`, las interfaces mostraron los estados `UP` y `LOWER_UP`, indicando que estaban activas y que el enlace virtual estaba operativo.

---

## 6. Verificación

Antes de construir la topología se realizó un reconocimiento del entorno de red general de Linux mediante:

```bash
ip link
ip addr
ip route
ip neigh
```

Esto permitió establecer una referencia del estado de red del sistema antes de trabajar con los namespaces.

### Interfaces

En el entorno general se identificaron principalmente las interfaces `lo` y `eth0`.

La interfaz `lo` presentó:

```text
inet 127.0.0.1/8
inet 10.255.255.254/32
```

Mientras que `eth0` presentó:

```text
inet 172.27.70.154/20
```

La interfaz `eth0` se encontraba en estado `UP` y `LOWER_UP`.

Una vez construida la topología, dentro de los namespaces se identificaron:

* `lo`: interfaz de loopback.
* `vethA` en `hostA`.
* `vethB` en `hostB`.

La interfaz `vethA` presentó la dirección MAC:

```text
4e:25:3a:40:2b:0c
```

Mientras que `vethB` presentó:

```text
32:66:45:9d:db:c4
```

La referencia `link-netns` permitió comprobar que ambas interfaces formaban parte del mismo enlace virtual.

### Direcciones

Las direcciones del entorno general fueron verificadas con:

```bash
ip addr
```

La interfaz `eth0` utilizó la dirección:

```text
172.27.70.154/20
```

Para la topología del laboratorio se utilizaron posteriormente:

```text
hostA → 10.10.1.1/30
hostB → 10.10.1.2/30
```

También aparecieron direcciones IPv6 link-local generadas automáticamente por Linux. Estas no fueron utilizadas para la comunicación del laboratorio, que se realizó mediante IPv4.

### Rutas

En el entorno general se observó la siguiente ruta por defecto:

```text
default via 172.27.64.1 dev eth0 proto kernel onlink
```

También se observó la ruta directamente conectada:

```text
172.27.64.0/20 dev eth0 proto kernel scope link src 172.27.70.154
```

Esto permitió identificar que el entorno WSL2 contaba con conectividad hacia redes externas mediante `eth0`.

Después de construir la topología, las tablas de rutas de los namespaces fueron consultadas con:

```bash
sudo ip netns exec hostA ip route
sudo ip netns exec hostB ip route
```

En `hostA` se obtuvo:

```text
10.10.1.0/30 dev vethA proto kernel scope link src 10.10.1.1
```

En `hostB`:

```text
10.10.1.0/30 dev vethB proto kernel scope link src 10.10.1.2
```

Estas rutas fueron creadas automáticamente por Linux al configurar las direcciones IP.

### Vecinos

En el entorno general se identificó como vecino el gateway:

```text
172.27.64.1 dev eth0 lladdr 00:15:5d:2e:f4:36 REACHABLE
```

Posteriormente, dentro de la topología se consultaron las tablas de vecinos mediante:

```bash
sudo ip netns exec hostA ip neigh
sudo ip netns exec hostB ip neigh
```

En `hostA` se observó:

```text
10.10.1.2 dev vethA lladdr 32:66:45:9d:db:c4 STALE
```

En `hostB`:

```text
10.10.1.1 dev vethB lladdr 4e:25:3a:40:2b:0c STALE
```

La tabla permite relacionar una dirección IPv4 con la dirección MAC correspondiente del vecino.

El estado `STALE` no representa una falla. Indica que la entrada existe, pero no ha sido confirmada recientemente.

### Conectividad

Primero se verificó la conectividad del entorno general mediante ICMP:

```bash
ping -c 4 google.com
ping -c 4 8.8.8.8
```

La prueba hacia `google.com` obtuvo:

```text
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = 33.168/89.996/210.643/72.348 ms
```

La prueba hacia `8.8.8.8` obtuvo:

```text
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = 35.610/91.205/257.188/95.830 ms
```

Esto permitió comprobar inicialmente que el entorno Linux disponía de conectividad externa.

Posteriormente se comprobó la conectividad entre los namespaces:

```bash
sudo ip netns exec hostA ping -c 4 10.10.1.2
```

El resultado fue:

```text
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = 0.287/2.720/8.183/3.234 ms
```

La prueba demostró que ambos namespaces podían intercambiar paquetes IPv4 correctamente mediante el enlace `veth`.

---

## 7. Captura y análisis de tráfico

Para observar directamente los paquetes que atravesaban el enlace virtual se utilizó `tcpdump` sobre `vethA`:

```bash
sudo ip netns exec hostA tcpdump -n -i vethA
```

Inicialmente la herramienta no estaba instalada y Linux respondió:

```text
exec of "tcpdump" failed: No such file or directory
```

Por ello se instaló mediante:

```bash
sudo apt update
sudo apt install tcpdump
```

Después de la instalación, `tcpdump` quedó disponible para realizar la captura.

Se generó tráfico mediante:

```bash
sudo ip netns exec hostA ping -c 4 10.10.1.2
```

La captura fue almacenada en:

```text
evidencias/tcpdump-ping.txt
```

El contenido obtenido fue:

```text
12:03:21.185984 IP 10.10.1.1 > 10.10.1.2: ICMP echo request, id 4010, seq 1, length 64
12:03:21.186003 IP 10.10.1.2 > 10.10.1.1: ICMP echo reply, id 4010, seq 1, length 64
12:03:22.203892 IP 10.10.1.1 > 10.10.1.2: ICMP echo request, id 4010, seq 2, length 64
12:03:22.203936 IP 10.10.1.2 > 10.10.1.1: ICMP echo reply, id 4010, seq 2, length 64
12:03:23.227560 IP 10.10.1.1 > 10.10.1.2: ICMP echo request, id 4010, seq 3, length 64
12:03:23.227581 IP 10.10.1.2 > 10.10.1.1: ICMP echo reply, id 4010, seq 3, length 64
12:03:24.251759 IP 10.10.1.1 > 10.10.1.2: ICMP echo request, id 4010, seq 4, length 64
12:03:24.251804 IP 10.10.1.2 > 10.10.1.1: ICMP echo reply, id 4010, seq 4, length 64
```

La captura muestra cuatro solicitudes ICMP (`echo request`) enviadas desde `hostA` hacia `hostB` y cuatro respuestas (`echo reply`) en sentido contrario.

La correspondencia observada es:

```text
hostA                         hostB

10.10.1.1  ── ICMP Request ──>  10.10.1.2
10.10.1.1  <── ICMP Reply ───  10.10.1.2

10.10.1.1  ── ICMP Request ──>  10.10.1.2
10.10.1.1  <── ICMP Reply ───  10.10.1.2
```

En esta captura no apareció tráfico ARP. Esto se debe a que la resolución de la dirección MAC del vecino ya había ocurrido antes de iniciar esta captura, como también se pudo comprobar mediante la tabla de vecinos.

Por lo tanto, la captura permite demostrar directamente el intercambio de paquetes ICMP generado por `ping`, mientras que la tabla `ip neigh` permite comprobar previamente la asociación entre dirección IP y dirección MAC.

---

## 8. Falla y diagnóstico

Para realizar una falla controlada se deshabilitó la interfaz de `hostB`:

```bash
sudo ip netns exec hostB ip link set vethB down
```

### Predicción

Antes de ejecutar la prueba se esperaba que la comunicación entre `hostA` y `hostB` dejara de funcionar debido a que uno de los extremos del enlace virtual estaría deshabilitado.

### Síntoma observado

Se ejecutó desde `hostA`:

```bash
sudo ip netns exec hostA ping -c 4 10.10.1.2
```

El resultado fue:

```text
Destination Host Unreachable
```

y los cuatro paquetes enviados no obtuvieron respuesta.

Además, `tcpdump` no mostró tráfico ICMP correspondiente a una comunicación exitosa.

### Diagnóstico

Se revisaron las interfaces:

```bash
sudo ip netns exec hostA ip link
sudo ip netns exec hostB ip link
```

La interfaz `vethB` se encontraba deshabilitada.

En `hostA`, el estado del enlace también reflejó la pérdida de conectividad con el extremo remoto.

Posteriormente se verificó la tabla de rutas:

```bash
sudo ip netns exec hostA ip route
```

La ruta directamente conectada presentó el estado:

```text
linkdown
```

También se consultó la tabla de vecinos:

```bash
sudo ip netns exec hostA ip neigh
```

La entrada correspondiente al vecino pasó al estado:

```text
FAILED
```

### Causa

La causa de la falla fue la desactivación de `vethB`.

El comportamiento observado puede resumirse como:

```text
vethB DOWN
      ↓
Se pierde el estado operativo del enlace
      ↓
hostA detecta el enlace como LINKDOWN
      ↓
No puede resolver/mantener al vecino
      ↓
Neighbor → FAILED
      ↓
Ping → Destination Host Unreachable
```

### Acción correctiva

Se restauró la interfaz:

```bash
sudo ip netns exec hostB ip link set vethB up
```

Finalmente se repitió la prueba de conectividad y se comprobó nuevamente la comunicación entre `hostA` y `hostB`.

Esto confirmó que la recuperación de la interfaz restauró el funcionamiento de la red virtual.

---

## 9. Uso de OpenCode

Durante el desarrollo del laboratorio se utilizó **GPT** como herramienta de apoyo para comprender conceptos de Linux, interpretar salidas de comandos y relacionar los procedimientos realizados con conocimientos previos de redes.

### Problema

Durante la práctica fue necesario comprender el funcionamiento de elementos como:

* `network namespaces`;
* interfaces `veth`;
* tablas de rutas;
* tabla de vecinos;
* estados `UP`, `LOWER_UP`, `LINKDOWN`, `STALE` y `FAILED`;
* captura de paquetes mediante `tcpdump`.

### Consulta

Se realizaron consultas orientadas principalmente a la explicación y diagnóstico de los resultados obtenidos, en lugar de utilizar la herramienta para sustituir la ejecución del laboratorio.

Por ejemplo, se utilizó GPT para interpretar las salidas de:

```bash
ip link
ip addr
ip route
ip neigh
```

y para relacionarlas con conceptos previamente conocidos de redes, como interfaces, rutas directamente conectadas y resolución ARP.

### Propuesta obtenida

La herramienta proporcionó explicaciones sobre el funcionamiento esperado de los comandos y ayudó a plantear posibles relaciones entre los estados observados.

### Verificación

Las propuestas fueron comprobadas directamente en el entorno WSL2 mediante la ejecución de los comandos correspondientes.

Por ejemplo, la interpretación de la falla provocada al ejecutar:

```bash
sudo ip netns exec hostB ip link set vethB down
```

fue contrastada mediante `ip link`, `ip route`, `ip neigh`, `ping` y `tcpdump`.

### Explicación técnica

El uso de IA se mantuvo como apoyo para el análisis y comprensión. Los resultados incluidos en este informe corresponden a pruebas ejecutadas directamente en el entorno de laboratorio y fueron interpretados a partir de las salidas observadas.

---

## 10. Conclusiones

1. Se construyó exitosamente una red virtual aislada utilizando dos `network namespaces` de Linux.

2. Se creó un enlace Ethernet virtual mediante un par `veth`, colocando cada extremo en un namespace diferente.

3. Se configuró correctamente el direccionamiento IPv4 `10.10.1.1/30` para `hostA` y `10.10.1.2/30` para `hostB`.

4. Se comprobó que Linux genera automáticamente una ruta directamente conectada al configurar una dirección IP, por lo que no fue necesario establecer un gateway para la comunicación entre ambos nodos.

5. La tabla de vecinos permitió observar la asociación entre las direcciones IPv4 y las direcciones MAC utilizadas en el enlace.

6. La captura mediante `tcpdump` permitió observar directamente los mensajes ICMP `echo request` y `echo reply` generados durante la prueba de conectividad.

7. La falla controlada permitió comprobar que la desactivación de una interfaz afecta el estado del enlace, la ruta y la resolución del vecino, produciendo la pérdida de conectividad.

8. Finalmente, la restauración de `vethB` permitió recuperar la comunicación, confirmando el diagnóstico realizado.

9. La práctica permitió relacionar herramientas de networking de Linux con conceptos previamente estudiados en redes, particularmente interfaces, direccionamiento, rutas, ARP, ICMP y diagnóstico de fallas.
