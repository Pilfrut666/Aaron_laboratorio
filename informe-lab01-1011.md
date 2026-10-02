# Laboratorio 1 — Redes virtuales con Linux

**Materia:** ETN1011 — Laboratorio de Sistemas de Comunicación II
**Estudiante:** [Aaron Valdair Rojas Calani]
**Usuario de github** [Pilfrut666]
**Usuario del sistema:** `lobito`

---

## 1. Datos del entorno

| Elemento | Valor |
|---|---|
| Usuario (`whoami`) | `lobito` |
| Fecha de la sesión inicial | 18 de junio de 2026 (21:54 UTC) |
| Distribución | Ubuntu 26.04.2 LTS (Resolute Raccoon) |
| Tipo de entorno | WSL2 (Linux sobre Windows) |
| Kernel | 6.18.32.2-microsoft-standard-WSL2, x86_64 |
| Directorio de trabajo | `~/2026-s2/lab1` (con subcarpeta `evidencias/`) |

Comandos utilizados:

```bash
cd ~/2026-s2/lab1
mkdir -p evidencias
whoami
uname -a
cat /etc/os-release
```

> **Nota:** el laboratorio se realizó en sesiones distintas por falta de tiempo continuo (por ejemplo, la captura `mtr` es del 23/09/2026 y la captura `tcpdump` de otra sesión). Los resultados de las distintas sesiones son coherentes entre sí.

---

## 2. Objetivo

Construir y verificar una red virtual aislada en una sola computadora Linux, usando *network namespaces* como nodos (`hostA` y `hostB`) y un par de interfaces `veth` como enlace. Con ello se busca:

- Configurar direccionamiento IPv4 y comprobar conectividad con `ping`.
- Interpretar interfaces, rutas y tabla de vecinos.
- Observar el tráfico ARP e ICMP.
- Provocar una falla controlada, diagnosticarla y restaurar el servicio.

---

## 3. Topología

```text
             Linux

       hostA                           hostB
┌───────────────────┐          ┌───────────────────┐
│ network namespace │          │ network namespace │
│                   │          │                   │
│  10.10.1.1/30     │          │  10.10.1.2/30     │
│       vethA       ├══════════┤       vethB       │
│                   │   veth   │                   │
└───────────────────┘          └───────────────────┘
```

### Reconocimiento previo de la red del sistema

Antes de crear la topología se inspeccionó el entorno con `ip link`, `ip addr`, `ip route` e `ip neigh` (Actividad 1).

| Elemento | Observación |
|---|---|
| Interfaces existentes | `lo` (bucle local) y `eth0` (Ethernet virtual, interfaz principal) |
| Estado | `lo`: UNKNOWN (normal en loopback) · `eth0`: UP |
| IPv4 principal | `172.31.32.207` en `eth0` |
| Prefijo | `/20` (máscara `255.255.240.0`) |
| Ruta por defecto | `default via 172.31.32.1 dev eth0` |
| Gateway | `172.31.32.1` |
| Interfaz de la ruta por defecto | `eth0` |
| Vecinos conocidos | Gateway `172.31.32.1`, MAC `00:15:5d:e1:48:99`, estado `REACHABLE` |

#### Pruebas de conectividad hacia el exterior

```bash
ping -c 4 8.8.8.8
```

```text
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=113 time=45.5 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=113 time=50.6 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=113 time=46.5 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=113 time=50.3 ms
```

> **Nota:** el registro del `ping -c 4 google.com` quedó con la misma salida del ping a `8.8.8.8` (copiada dos veces), por lo que no se muestra una salida propia para el nombre de dominio.

`traceroute 8.8.8.8` (resumen): el destino se alcanza en 12 saltos. El primer salto es el gateway de WSL (`172.31.32.1`, ~0.5 ms), el segundo es el router doméstico (`192.168.1.1`, ~2 ms) y luego se atraviesan redes del proveedor hasta `dns.google (8.8.8.8)` con ~46–48 ms. Los saltos 10 y 11 no responden (`* * *`), lo cual es habitual en routers que no contestan a las sondas.

`mtr -r -c 5 8.8.8.8` (resumen): 13 saltos y 0 % de pérdida en el destino final. Solo el salto 8 (`190.129.248.3`) muestra 20 % de pérdida, que no se propaga a los saltos siguientes, por lo que no indica un problema real de conectividad (limitación de respuestas ICMP del propio router). Latencia promedio al destino: ~50 ms.

#### Pregunta: si `ping 8.8.8.8` funciona pero `ping google.com` no, ¿qué se investigaría primero?

**Respuesta:** el servicio de **DNS** (resolución de nombres). Si el ping por IP funciona, la conectividad de capa de red y la salida a Internet están operativas; lo que falla es la traducción del nombre `google.com` a una dirección IP. Habría que revisar el servidor DNS configurado (por ejemplo en `/etc/resolv.conf`) y probar la resolución con una herramienta como `nslookup` o `dig`.

---

## 4. Direccionamiento

| Elemento | Valor |
|---|---|
| Namespace 1 | `hostA` |
| Namespace 2 | `hostB` |
| Interfaz de hostA | `vethA` |
| Interfaz de hostB | `vethB` |
| Dirección hostA | `10.10.1.1/30` |
| Dirección hostB | `10.10.1.2/30` |

### Actividad 2 — Análisis de la subred `10.10.1.0/30`

| Parámetro | Valor |
|---|---|
| Dirección de red | `10.10.1.0` |
| Direcciones utilizables | `10.10.1.1` y `10.10.1.2` |
| Dirección de broadcast | `10.10.1.3` |
| Cantidad de direcciones utilizables | 2 |

**Explicación:** con un prefijo `/30`, los primeros 30 bits identifican la red y solo quedan 2 bits para hosts (4 direcciones en total, de las cuales 2 son utilizables, pues una es de red y otra de broadcast). Tanto `10.10.1.1` como `10.10.1.2` comparten los mismos 30 bits de red (`10.10.1.0`) y la misma máscara (`255.255.255.252`), por lo que pertenecen a la misma subred y pueden comunicarse directamente por el mismo enlace, sin necesidad de un router.

---

## 5. Construcción

### 5.1 Creación de los namespaces

```bash
sudo ip netns add hostA
sudo ip netns add hostB
ip netns list
```

Resultado:

```text
hostB
hostA
```

**Explicación:** un *network namespace* es una característica del kernel de Linux que proporciona una instancia aislada y virtualizada de la pila de red del sistema operativo. Cada namespace tiene sus propias interfaces, direcciones, tabla de rutas y tabla de vecinos, independientes de las del sistema principal y de los demás namespaces.

### 5.2 Creación del enlace virtual (par `veth`)

```bash
sudo ip link add vethA type veth peer name vethB
ip link
```

Resultado (fragmento):

```text
3: vethB@vethA: <BROADCAST,MULTICAST,M-DOWN> ... state DOWN ...
4: vethA@vethB: <BROADCAST,MULTICAST,M-DOWN> ... state DOWN ...
```

**Pregunta: ¿por qué las dos interfaces aparecen inicialmente en el espacio de red principal?**
Porque al crear el par, el kernel instancia ambos extremos en el namespace por defecto (el principal), desde donde se ejecuta el comando. Después hay que moverlos manualmente. Un par `veth` funciona como un cable virtual: lo que entra por un extremo sale por el otro.

### 5.3 Asignación de cada extremo a su namespace

```bash
sudo ip link set vethA netns hostA
sudo ip link set vethB netns hostB
ip link
```

En el espacio principal solo quedan `lo` y `eth0`. Al consultar dentro de cada namespace:

```bash
sudo ip netns exec hostA ip link
sudo ip netns exec hostB ip link
```

```text
# hostA
1: lo: <LOOPBACK> ... state DOWN ...
4: vethA@if3: <BROADCAST,MULTICAST> ... state DOWN ... link-netns hostB

# hostB
1: lo: <LOOPBACK> ... state DOWN ...
3: vethB@if4: <BROADCAST,MULTICAST> ... state DOWN ... link-netns hostA
```

**Preguntas:**

1. *¿Por qué `vethA` y `vethB` dejaron de aparecer en el `ip link` del espacio principal?*
   Porque una interfaz de red solo puede pertenecer a un único namespace a la vez. Al moverlas, dejaron de existir en el espacio principal.
2. *¿Dónde existe ahora cada interfaz?*
   `vethA` existe en `hostA` y `vethB` existe en `hostB`. Además, el campo `link-netns` muestra en qué namespace está su extremo opuesto.

### 5.4 Configuración de direcciones IPv4

```bash
sudo ip netns exec hostA ip addr add 10.10.1.1/30 dev vethA
sudo ip netns exec hostB ip addr add 10.10.1.2/30 dev vethB
```

> **Observación:** en el registro, la ejecución de estos comandos devolvió `Error: ipv4: Address already assigned`, lo que indica que las direcciones ya estaban asignadas (probablemente por una ejecución repetida del comando). La verificación posterior confirma que la configuración final es la correcta.

Verificación:

```bash
sudo ip netns exec hostA ip addr
sudo ip netns exec hostB ip addr
```

```text
# hostA
4: vethA@if3: <BROADCAST,MULTICAST> ... state DOWN ...
    inet 10.10.1.1/30 scope global vethA

# hostB
3: vethB@if4: <BROADCAST,MULTICAST> ... state DOWN ...
    inet 10.10.1.2/30 scope global vethB
```

### 5.5 Puesta en servicio de las interfaces

Tras configurar las direcciones, las interfaces seguían en estado `DOWN`. Se habilitaron con:

```bash
sudo ip netns exec hostA ip link set lo up
sudo ip netns exec hostA ip link set vethA up

sudo ip netns exec hostB ip link set lo up
sudo ip netns exec hostB ip link set vethB up
```

Estado resultante:

```text
# hostA
1: lo: <LOOPBACK,UP,LOWER_UP> ... state UNKNOWN ...
4: vethA@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> ... state UP ...

# hostB
1: lo: <LOOPBACK,UP,LOWER_UP> ... state UNKNOWN ...
3: vethB@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> ... state UP ...
```

**Pregunta: ¿qué diferencia hay entre una interfaz existente, una interfaz direccionada y una interfaz en estado `UP`?**

- **Existente:** la interfaz está creada en el namespace, pero no tiene dirección IP ni está activa.
- **Direccionada:** tiene una dirección IP asignada, pero puede seguir en estado `DOWN` y, por tanto, no transmite ni recibe tráfico (así estaban `vethA` y `vethB` en el paso 5.4).
- **En estado `UP`:** está administrativamente activa y con enlace operativo (`LOWER_UP`); solo entonces puede comunicarse. Para que la comunicación funcione hacen falta las tres condiciones: que exista, que tenga la dirección correcta y que esté `UP`.

---

## 6. Verificación

### Interfaces

Dentro de cada namespace existen `lo` y la interfaz `veth` correspondiente, ambas en estado `UP` (ver sección 5.5). Las interfaces `vethA` y `vethB` ya no aparecen en el espacio de red principal.

### Direcciones

| Namespace | Interfaz | Dirección |
|---|---|---|
| `hostA` | `vethA` | `10.10.1.1/30` |
| `hostB` | `vethB` | `10.10.1.2/30` |

### Rutas

```bash
sudo ip netns exec hostA ip route
sudo ip netns exec hostB ip route
```

```text
# hostA
10.10.1.0/30 dev vethA proto kernel scope link src 10.10.1.1

# hostB
10.10.1.0/30 dev vethB proto kernel scope link src 10.10.1.2
```

#### Actividad 3 — Interpretación de las rutas (sin agregar rutas manuales)

1. **Ruta en `hostA`:** `10.10.1.0/30 dev vethA proto kernel scope link src 10.10.1.1`.
2. **Ruta en `hostB`:** `10.10.1.0/30 dev vethB proto kernel scope link src 10.10.1.2`.
3. **Por qué apareció:** al asignar una IP a una interfaz y ponerla en estado `UP`, Linux crea automáticamente una *ruta conectada* (`proto kernel`) hacia la red a la que pertenece esa dirección.
4. **Interfaz utilizada:** de `hostA` hacia `hostB` se usa `vethA`; de `hostB` hacia `hostA` se usa `vethB`.
5. **¿Es necesario un gateway?** **No.** Ambos nodos están en la misma red `10.10.1.0/30`, es decir, el destino es alcanzable directamente por el enlace (`scope link`). Un gateway solo es necesario para llegar a redes distintas.

### Vecinos

```bash
sudo ip netns exec hostA ip neigh
sudo ip netns exec hostB ip neigh
```

```text
10.10.1.2 dev vethA lladdr 16:1a:93:17:33:d4 REACHABLE
10.10.1.1 dev vethB lladdr 0e:4b:5c:98:6e:b2 REACHABLE
```

#### Actividad 4 — Tabla de vecinos

| Nodo | IP del vecino | MAC observada | Estado |
|---|---|---|---|
| `hostA` | `10.10.1.2` | `16:1a:93:17:33:d4` | `REACHABLE` |
| `hostB` | `10.10.1.1` | `0e:4b:5c:98:6e:b2` | `REACHABLE` |

Las MACs coinciden con las de `vethB` y `vethA` mostradas en `ip link`.

**¿Por qué una comunicación IPv4 en el mismo enlace necesita una dirección de capa 2?** Porque las tramas Ethernet se entregan según direcciones MAC, no IP. Para enviar un paquete IP a un nodo del mismo enlace, el emisor debe encapsularlo en una trama cuya dirección destino sea la MAC del vecino. ARP resuelve esa correspondencia IP → MAC, y el resultado se guarda en la tabla de vecinos.

### Conectividad

```bash
sudo ip netns exec hostA ping -c 4 10.10.1.2
sudo ip netns exec hostB ping -c 4 10.10.1.1
```

```text
# Desde hostA
4 packets transmitted, 4 received, 0% packet loss, time 3038ms
rtt min/avg/max/mdev = 0.064/0.528/1.891/0.786 ms

# Desde hostB
4 packets transmitted, 4 received, 0% packet loss, time 3079ms
rtt min/avg/max/mdev = 0.072/0.110/0.206/0.055 ms
```

Conectividad bidireccional confirmada, con 0 % de pérdida y `ttl=64` (los paquetes no atraviesan ningún router). El primer paquete tarda más (1.89 ms en hostA) porque incluye la resolución ARP inicial.

---

## 7. Captura y análisis de tráfico

En una terminal se inició la captura en `hostA`:

```bash
sudo ip netns exec hostA tcpdump -n -i vethA
```

Desde otra terminal, se generó tráfico desde `hostB`:

```bash
sudo ip netns exec hostB ping -c 4 10.10.1.1
```

(resultado del ping: 4 enviados, 4 recibidos, 0 % de pérdida).

### Tráfico observado

**Al iniciar la captura** (mensajes IPv6 generados por las propias interfaces al activarse):

```text
11:47:42.044971 IP6 fe80::c4b:5cff:fe98:6eb2 > ff02::2: ICMP6, router solicitation, length 16
11:47:42.045035 IP6 fe80::141a:93ff:fe17:33d4 > ff02::2: ICMP6, router solicitation, length 16
```

**ICMP Echo Request / Echo Reply:**

```text
11:49:35.340208 IP 10.10.1.2 > 10.10.1.1: ICMP echo request, id 952, seq 1, length 64
11:49:35.340239 IP 10.10.1.1 > 10.10.1.2: ICMP echo reply, id 952, seq 1, length 64
11:49:36.349010 IP 10.10.1.2 > 10.10.1.1: ICMP echo request, id 952, seq 2, length 64
11:49:36.349038 IP 10.10.1.1 > 10.10.1.2: ICMP echo reply, id 952, seq 2, length 64
(... seq 3 y seq 4 con el mismo patrón)
```

**ARP:**

```text
11:49:40.573000 ARP, Request who-has 10.10.1.2 tell 10.10.1.1, length 28
11:49:40.573078 ARP, Request who-has 10.10.1.1 tell 10.10.1.2, length 28
11:49:40.573089 ARP, Reply 10.10.1.1 is-at 0e:4b:5c:98:6e:b2, length 28
11:49:40.573103 ARP, Reply 10.10.1.2 is-at 16:1a:93:17:33:d4, length 28
```

### Actividad 5 — Explicación de la secuencia

1. Al activarse las interfaces aparecen `router solicitation` de IPv6 (`ff02::2`), tráfico propio de la autoconfiguración IPv6, ajeno al ping IPv4.
2. Los ICMP Echo Request (de `10.10.1.2`) y los Echo Reply (de `10.10.1.1`) se alternan en pares, con un segundo de diferencia entre cada secuencia (`seq 1` a `seq 4`, mismo `id 952`).
3. **No se ve ARP antes del ping**, porque ambos nodos ya conocían la MAC del otro (entradas `REACHABLE` generadas por las pruebas de la sección 6). Por eso el tráfico ICMP se envía directamente.
4. El intercambio ARP aparece unos segundos **después** del ping: es la revalidación periódica de la entrada de vecinos, con la que Linux comprueba que la IP sigue asociada a esa MAC.

---

## 8. Falla y diagnóstico

### Actividad 6

**1. Predicción previa**
Al desactivar `vethB` en `hostB`, `hostA` no tendría conectividad hacia `10.10.1.2` y se perderían todos los paquetes enviados con `ping`, debido a la interrupción del enlace.

**2. Síntoma observado**

```bash
sudo ip netns exec hostB ip link set vethB down
sudo ip netns exec hostA ping -c 4 10.10.1.2
```

```text
PING 10.10.1.2 (10.10.1.2) 56(84) bytes of data.
From 10.10.1.1 icmp_seq=1 Destination Host Unreachable
--- 10.10.1.2 ping statistics ---
4 packets transmitted, 0 received, +1 errors, 100% packet loss, time 3074ms
```

El ping falló con 100 % de pérdida y el mensaje `Destination Host Unreachable`, generado por el propio `hostA` (`From 10.10.1.1`).

**3. Comandos utilizados para diagnosticar**

```bash
sudo ip netns exec hostA ip link
sudo ip netns exec hostA ip addr
sudo ip netns exec hostA ip route
sudo ip netns exec hostA ip neigh

sudo ip netns exec hostB ip link
sudo ip netns exec hostB ip addr
sudo ip netns exec hostB ip route
```

**4. Causa de la falla**

| Comando | Evidencia |
|---|---|
| `ip link` / `ip addr` en `hostA` | `vethA` con `<NO-CARRIER,...,UP>` y `state DOWN`: está habilitada pero sin portadora (el otro extremo está caído). La dirección `10.10.1.1/30` sigue configurada. |
| `ip route` en `hostA` | `10.10.1.0/30 dev vethA ... linkdown`: la ruta existe pero está marcada como sin enlace. |
| `ip neigh` en `hostA` | `10.10.1.2 dev vethA FAILED`: no se pudo resolver la MAC del vecino. |
| `ip link` / `ip addr` en `hostB` | `vethB` en estado `DOWN` (sin `UP`) y sin ruta conectada: **causa raíz**. |

La configuración de direcciones y rutas era correcta; el fallo estaba en la **capa de enlace**: `vethB` estaba administrativamente caída, por lo que `hostA` no podía resolver por ARP la MAC de `10.10.1.2` y no se entregaba ningún paquete.

**5. Acción correctiva**

```bash
sudo ip netns exec hostB ip link set vethB up
```

**6. Prueba de recuperación**

```bash
sudo ip netns exec hostA ping -c 4 10.10.1.2
```

```text
4 packets transmitted, 4 received, 0% packet loss, time 3075ms
rtt min/avg/max/mdev = 0.075/0.081/0.094/0.007 ms
```

Tras restaurar la interfaz, `vethA` volvió a `state UP`, la ruta perdió la marca `linkdown` y la entrada de vecinos pasó de `FAILED` a `10.10.1.2 dev vethA lladdr 16:1a:93:17:33:d4 REACHABLE`. La conectividad se recuperó sin necesidad de recrear la topología.

---

## 9. Uso de OpenCode

OpenCode se utilizó como apoyo durante el laboratorio, principalmente en las secciones de actividades. La evidencia de estas consultas se conserva como capturas de pantalla (fotos) que se adjuntan en la carpeta `evidencias/`.

---

## 10. Conclusiones

- Se construyó una red virtual aislada con dos *network namespaces* (`hostA` y `hostB`) interconectados mediante un par `veth`, y se verificó la conectividad IPv4 bidireccional con 0 % de pérdida.
- Se comprobó que para que una interfaz funcione deben cumplirse tres condiciones: existir en el namespace, tener la dirección correcta y estar en estado `UP`.
- Linux generó automáticamente la ruta conectada `10.10.1.0/30` al asignar y activar las direcciones, por lo que no fue necesario configurar un gateway: ambos nodos están en la misma subred.
- La tabla de vecinos y la captura de tráfico mostraron el papel de ARP en la resolución IP → MAC y la secuencia ICMP Echo Request / Echo Reply.
- La falla controlada (`vethB` caída) se diagnosticó de forma ordenada, observando `NO-CARRIER`, `linkdown` y `FAILED` en `hostA`, y se corrigió reactivando la interfaz sin destruir la topología.
- El laboratorio se realizó en diferentes sesiones por falta de tiempo; aun así, los datos tomados en distintos momentos fueron coherentes.

---

## Limpieza del entorno

Una vez revisadas las evidencias, la topología puede eliminarse con:

```bash
sudo ip netns del hostA
sudo ip netns del hostB
ip netns list
```
