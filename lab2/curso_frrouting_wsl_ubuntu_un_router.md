# Curso práctico de FRRouting en WSL con Ubuntu: un solo router

**Nivel:** introductorio–intermedio  
**Duración sugerida:** 90 minutos (o dos sesiones de 45 minutos)  
**Requisitos:** Windows 11, WSL2 con Ubuntu, terminal, conexión a Internet y nociones de IPv4 y rutas.  
**Objetivo:** instalar FRRouting, entender su arquitectura, administrar rutas estáticas y observar sus tablas en un **único router FRR**, sin depender de un segundo router ni de una vecindad OSPF.

> **Seguridad del laboratorio.** Trabaja en una instancia WSL de práctica. No borres la ruta por defecto ni cambies la interfaz usada para acceder a Internet. WSL2 puede usar NAT o modo mirrored; sus interfaces y permisos dependen de la configuración de Windows. Los comandos de FRR se muestran separados de los de Bash.

## 1. Historia y evolución (10 minutos)

GNU/Linux siempre ha dispuesto de pila IP y tabla de rutas en el kernel. Para intercambiar rutas dinámicamente se desarrollaron suites de protocolos en espacio de usuario.

```text
GNU Zebra (finales de los años 1990)
   └── Quagga (fork de Zebra, 2002)
         └── FRRouting / FRR (fork de Quagga, 2017)
               └── Desarrollo comunitario actual
```

- **GNU Zebra:** arquitectura modular de demonios para protocolos de enrutamiento.
- **Quagga:** continuó el proyecto y su interfaz de configuración similar a Cisco.
- **FRRouting:** modernizó el desarrollo y amplió las capacidades de routing, con protocolos como BGP, OSPF, IS-IS y rutas estáticas.

No confundas el proyecto histórico *GNU Zebra* con el demonio `zebra` que sigue existiendo en FRR.

**Pregunta:** ¿por qué FRR si Linux ya tiene `ip route`? Porque el kernel reenvía paquetes y mantiene rutas, pero no implementa por sí solo los protocolos de intercambio de rutas como BGP u OSPF.

## 2. Arquitectura: Linux + FRR (10 minutos)

```text
        Administrador
             |
           vtysh (CLI)
             |
    +--------+---------+
    |        |         |
 staticd    ospfd     bgpd       procesos de control
    +--------+---------+
             |
           zebra       coordina rutas e interfaces
             |
         Kernel Linux  tabla de rutas / forwarding
             |
      Interfaz virtual WSL
```

| Componente | Función |
|---|---|
| `zebra` | Coordina información de interfaces y rutas con el kernel |
| `staticd` | Administra rutas estáticas de FRR |
| `ospfd` | Implementa OSPFv2 (IPv4) |
| `bgpd` | Implementa BGP |
| `vtysh` | Consola integrada de administración |
| `watchfrr` | Supervisa los procesos de FRR |

**Plano de control:** cálculo y administración de rutas. **Plano de datos:** reenvío real de paquetes, normalmente realizado por el kernel Linux. FRR no sustituye la configuración de IP, firewall o interfaces de Ubuntu.

## 3. Preparación e instalación en WSL con Ubuntu (15 minutos)

### 3.1. Comprobar WSL desde PowerShell

```powershell
wsl --list --verbose
```

Si Ubuntu aún no está instalado, desde PowerShell como administrador:

```powershell
wsl --install -d Ubuntu
```

Completa la creación del usuario Linux y reinicia Windows si se solicita. Para iniciar Ubuntu: `wsl -d Ubuntu`.

### 3.2. Verificar Ubuntu y red

Los comandos siguientes se ejecutan **dentro de Ubuntu**, no en PowerShell:

```bash
cat /etc/os-release
uname -r
ip -br addr
ip -4 route
```

Anota la interfaz principal y la ruta `default`; no las modifiques en este laboratorio.

### 3.3. Instalar paquetes

```bash
sudo apt update
sudo apt install -y frr frr-pythontools iproute2 iputils-ping
apt policy frr
```

Se usa el paquete de Ubuntu, cuya versión depende de la distribución. Para entornos productivos también existe el repositorio oficial `https://deb.frrouting.org/`, pero no es necesario añadirlo en esta clase.

### 3.4. Servicios en WSL

Comprueba si `systemd` está activo:

```bash
ps -p 1 -o comm=
```

Si devuelve `systemd`, utiliza:

```bash
sudo systemctl status frr --no-pager
```

Si **no** devuelve `systemd`, habilítalo editando `/etc/wsl.conf`:

```ini
[boot]
systemd=true
```

Luego **desde PowerShell**, cierra las sesiones de Ubuntu y ejecuta:

```powershell
wsl --shutdown
```

Vuelve a abrir Ubuntu y verifica `ps -p 1 -o comm=`. Habilitar systemd afecta la instancia WSL; no ejecutes `wsl --shutdown` mientras tengas procesos importantes activos en otras distribuciones.

## 4. Activar FRR y entrar a vtysh (10 minutos)

Edita la lista de demonios:

```bash
sudo nano /etc/frr/daemons
```

Asegura estos valores, sin duplicar líneas:

```ini
zebra=yes
staticd=yes
ospfd=no
bgpd=no
```

No activamos OSPF ni BGP: un solo router no permite formar vecindad OSPF ni sesión BGP real sin otro extremo.

```bash
sudo systemctl enable --now frr
sudo systemctl restart frr
sudo systemctl status frr --no-pager
sudo vtysh -c 'show version'
sudo vtysh -c 'show ip route'
```

Para entrar a la CLI:

```bash
sudo vtysh
```

Dentro de `vtysh`:

```text
show version
show interface
show ip route
show running-config
exit
```

**Regla importante:** `sudo vtysh` se ejecuta en Bash; `show ip route` se ejecuta dentro de `vtysh` o mediante `sudo vtysh -c 'show ip route'`.

## 5. Laboratorio de un solo router: rutas estáticas (20 minutos)

### 5.1. Topología

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

Una interfaz **dummy** simula una red conectada sin requerir una segunda máquina. No representa una LAN externa real. La ruta estática a una red de ejemplo se instalará como **blackhole**, evitando depender de un siguiente salto inexistente. Esto enseña sintaxis y tablas, **no demuestra conectividad entre routers**.

### 5.2. Crear interfaz de laboratorio en Linux

```bash
sudo ip link add dummy0 type dummy
sudo ip addr add 10.10.10.1/24 dev dummy0
sudo ip link set dummy0 up
ip -br addr show dummy0
ip -4 route show 10.10.10.0/24
```

Si `ip link add` responde `Operation not permitted`, revisa permisos/capacidades de la versión de WSL; puedes continuar con las consultas de FRR y la ruta blackhole sin la interfaz dummy. No alteres `eth0` para compensarlo.

**Observación:** Linux crea automáticamente una ruta conectada para `10.10.10.0/24` cuando se asigna esa dirección.

### 5.3. Configurar una ruta estática en FRR

En Bash:

```bash
sudo vtysh
```

En `vtysh`:

```text
configure terminal
ip route 198.51.100.0/24 blackhole
end
show ip route
show running-config
write memory
```

`198.51.100.0/24` es un prefijo reservado para documentación. `blackhole` descarta los paquetes destinados a ese prefijo: **no es una ruta hacia Internet**. Sirve para practicar la administración de rutas sin un router vecino.

De vuelta en Bash:

```bash
ip -4 route show 198.51.100.0/24
sudo vtysh -c 'show ip route 198.51.100.0/24'
```

Comprueba la diferencia entre la vista del kernel y la de FRR. La instalación efectiva puede depender de otras rutas y políticas del sistema.

### 5.4. Eliminar y restaurar la ruta

En `vtysh`:

```text
configure terminal
no ip route 198.51.100.0/24 blackhole
end
show ip route
```

Restáurala solo si deseas conservar el ejercicio:

```text
configure terminal
ip route 198.51.100.0/24 blackhole
end
write memory
```

### 5.5. Persistencia: dos mundos

- `write memory` guarda la configuración de **FRR** en `/etc/frr/frr.conf`.
- `ip link add` y `ip addr add` son **temporales**; no los conserva `write memory`.
- Para interfaces persistentes se necesita un mecanismo de Ubuntu, como `systemd-networkd`, NetworkManager o un script de laboratorio explícito.
- En WSL, además, las interfaces virtuales de red pueden recrearse al reiniciar la instancia.

**Limpieza opcional**, desde Bash y después de terminar:

```bash
sudo ip link delete dummy0
```

Esto elimina únicamente la interfaz de laboratorio. Si conservaste una configuración FRR referida a ella, revísala.

## 6. Comandos más comunes (10 minutos)

| Tarea | En Ubuntu (Bash) | En FRR (`vtysh`) |
|---|---|---|
| Estado del servicio | `systemctl status frr` | — |
| Registros | `journalctl -u frr -n 50` | — |
| Interfaces | `ip -br addr` | `show interface` |
| Rutas IPv4 | `ip -4 route` | `show ip route` |
| Versión | `apt policy frr` | `show version` |
| Configuración | Archivos en `/etc/frr` | `show running-config` |
| Modo configuración | — | `configure terminal` |
| Añadir ruta estática | `ip route add ...` | `ip route PREFIJO NEXT_HOP` |
| Borrar ruta estática | `ip route del ...` | `no ip route ...` |
| Guardar configuración FRR | — | `write memory` |
| Diagnóstico OSPF (si está activo) | — | `show ip ospf neighbor` |
| Ayuda | `man ip-route` | `?` |

Otros comandos útiles en FRR: `show ip route static`, `show running-config`, `show interface`, `show version` y `show ip route summary` (según versión). `show ip ospf neighbor` no mostrará vecinos en este laboratorio.

## 7. Comparación FRRouting y Cisco IOS clásico (10 minutos)

| Operación | FRR en `vtysh` | Cisco IOS clásico |
|---|---|---|
| Configurar | `configure terminal` | `configure terminal` |
| Ver configuración | `show running-config` | `show running-config` |
| Guardar | `write memory` | `copy running-config startup-config` |
| Ver rutas | `show ip route` | `show ip route` |
| Ruta a siguiente salto | `ip route 10.20.0.0/24 10.0.0.2` | `ip route 10.20.0.0 255.255.255.0 10.0.0.2` |
| Eliminar ruta | `no ip route 10.20.0.0/24 10.0.0.2` | `no ip route 10.20.0.0 255.255.255.0 10.0.0.2` |
| Ver interfaces | `show interface` | `show interfaces` |
| Dirección IP de interfaz | Normalmente `ip addr add ... dev ...` en Linux | `interface ...` y `ip address ...` |
| OSPF | `router ospf` | `router ospf 1` |
| Red OSPF | `network 10.0.0.0/30 area 0` | `network 10.0.0.0 0.0.0.3 area 0` |
| Vecinos OSPF | `show ip ospf neighbor` | `show ip ospf neighbor` |

**Diferencias de fondo:** Cisco IOS integra normalmente interfaz, direcciones y protocolos en un mismo sistema operativo de red. FRR se apoya en Linux: la administración de interfaces, reenvío y firewall pertenece al sistema operativo, mientras FRR administra los protocolos y las rutas. La sintaxis se parece, pero no es idéntica; IOS XE, IOS XR y NX-OS también presentan diferencias.

**Ejercicio:** traduce a Cisco IOS la ruta FRR `ip route 172.20.0.0/16 10.0.0.2`. Respuesta: `ip route 172.20.0.0 255.255.0.0 10.0.0.2`.

## 8. Diagnóstico y problemas habituales (5 minutos)

| Síntoma | Comprobación |
|---|---|
| `systemctl` no funciona | `ps -p 1 -o comm=`; habilitación de systemd en WSL |
| `vtysh` no conecta | `systemctl status frr`; `/etc/frr/daemons`; `journalctl -u frr` |
| No aparece la ruta | `show running-config`, `show ip route`, `ip -4 route` |
| No se crea `dummy0` | Revisar `sudo` y capacidades de WSL |
| Desaparece la IP tras reiniciar | `ip addr add` no es persistente |
| No hay vecinos OSPF | Es lo esperado: no hay otro router y `ospfd` está deshabilitado |

No uses `ip route flush`, ni desactives globalmente el firewall, ni borres la ruta predeterminada para resolver un fallo del ejercicio.

## 9. Evaluación de salida

1. ¿De qué proyectos históricos deriva FRRouting?
2. ¿Cuál es la función de `zebra` y cuál la del kernel Linux?
3. ¿Por qué no configuramos una vecindad OSPF con un solo router?
4. ¿Qué hace `ip route 198.51.100.0/24 blackhole`?
5. ¿Qué conserva `write memory` y qué no conserva?
6. ¿Cuál es la diferencia entre `show ip route` de FRR y `ip -4 route` de Linux?

**Respuestas:** (1) Quagga y GNU Zebra. (2) Zebra coordina rutas con el kernel; el kernel mantiene rutas y reenvía paquetes. (3) No existe un segundo extremo con el cual formar adyacencia. (4) Crea una ruta de descarte para el prefijo de documentación. (5) Guarda FRR, no las interfaces temporales de Linux. (6) Muestran las perspectivas de FRR y del kernel, respectivamente.

## 10. Referencias y ampliación

- [Documentación oficial FRRouting](https://docs.frrouting.org/en/latest/)
- [Instalación de FRR](https://docs.frrouting.org/en/latest/installation.html)
- [Configuración y demonios](https://docs.frrouting.org/en/latest/setup.html)
- [vtysh](https://docs.frrouting.org/en/latest/vtysh.html)
- [Rutas estáticas](https://docs.frrouting.org/en/latest/static.html)
- [Instalar WSL (Microsoft)](https://learn.microsoft.com/en-us/windows/wsl/install)
- [systemd en WSL (Microsoft)](https://learn.microsoft.com/en-us/windows/wsl/systemd)

**Conclusión:** *Linux proporciona las interfaces y el reenvío; FRRouting ofrece la lógica de enrutamiento y una CLI familiar para operadores acostumbrados a Cisco IOS.*
