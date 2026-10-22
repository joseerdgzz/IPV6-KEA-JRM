# IPv6 en la topología E: SLAAC, stateless y stateful con Kea

1. **SLAAC** (autoconfiguración). Todos los routers lo tienen activo por defecto. Probar clientes en distintas redes (distintos routers), snifar el proceso y buscar los bits O y M.
2. **Stateless** (sin control de estado). Hace falta un servidor DHCP y la autoconfiguración. Snifar y ver los bits O y M.
3. **Stateful** (con control de estado). Hace falta un servidor DHCP completo, tan completo como en IPv4. Snifar y ver los bits O y M.
4. Opciones a tener presentes: relay donde haga falta, y ¿qué pasa con el exporter?

---

## 1. Introducción

En IPv4 el equipo pedía todo por DHCP (el DORA). En IPv6 el primero que habla es el router: manda un RA (*Router Advertisement*) con el prefijo de la red y diciendo cómo tienen que configurarse los equipos. El cliente puede pedirlo con un RS (*Router Solicitation*). Los dos son mensajes ICMPv6:

| Mensaje | Tipo ICMPv6 | Quién → a quién |
|---|---|---|
| RS | 133 | Cliente → `ff02::2` (todos los routers) |
| RA | 134 | Router → `ff02::1` (todos los equipos) |
| NS | 135 | Cliente comprobando que su IP no la usa nadie (DAD) |

Dentro del RA van dos bits que deciden el modo:

- **M** (*Managed address configuration*): "pide la IP por DHCPv6".
- **O** (*Other configuration*): "pide el resto (DNS, dominio) por DHCPv6".

| Modo | M | O | A | IP | DNS y dominio | Kea |
|---|---|---|---|---|---|---|
| SLAAC | 0 | 0 | 1 | Se la monta el cliente | Nada | No hace falta |
| Stateless | 0 | 1 | 1 | Se la monta el cliente | Kea | Sí, sin pool |
| Stateful | 1 | 1 | 0 | Kea | Kea | Sí, con pool |

En DHCPv6 el equivalente al DORA es el **SARR**: **S**olicit, **A**dvertise, **R**equest y **R**eply. En stateless no hay SARR, solo dos mensajes: Information-request y Reply.

Lo de que "todos los routers lo tienen activo por defecto" no es verdad en mi topología. El Cisco sí manda RA de serie, pero el MikroTik no (en la práctica de DHCP le puse `advertise=no`), el FRR los trae apagados (`suppress-ra`) y en la Olive hay que configurarlos.

---

## 2. Cómo lo monté

Ya tenía IPv6 en todas las LAN desde la malla: `2007:28:17:10N::/64`, con el router en `::1`. Los Kea de IPv4 los dejé tal cual. Para IPv6 hice una imagen nueva y dos contenedores nuevos, así no tocaba lo que ya funcionaba.

| Nodo | LAN | IPv4 | IPv6 |
|---|---|---|---|
| kea6-1 | 101 (Letonia), Switch3 | 10.0.101.3 | 2007:28:17:101::3 |
| kea6-2 | 106 (Laos), Switch4 | 10.0.106.3 | 2007:28:17:106::3 |

Les puse también IPv4 para la API, el HA y el exporter, igual que los Kea de IPv4.

| Cliente | Para qué sirve |
|---|---|
| VPCS (PC3) | Solo SLAAC (`ip auto`). No tiene cliente DHCPv6 |
| cliente6-1 y cliente6-3 | Los tres modos. `dhcpcd` hace caso de los bits M y O |
| Windows | SLAAC, y DHCPv6 si hace falta |

### La imagen kea6

En `~/kea6`:

```dockerfile
FROM debian:bookworm-slim
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl ca-certificates gnupg apt-transport-https debian-archive-keyring \
 && curl -1sLf 'https://dl.cloudsmith.io/public/isc/kea-3-0/setup.deb.sh' | bash \
 && apt-get install -y --no-install-recommends isc-kea-dhcp6 isc-kea-hooks \
      isc-dhcp-relay iproute2 iputils-ping dnsutils netcat-openbsd tcpdump procps python3 \
 && rm -rf /var/lib/apt/lists/* \
 && mkdir -p /var/lib/kea /run/kea /etc/kea/compartido
CMD ["kea-dhcp6", "-c", "/etc/kea/compartido/kea6-stateless-1.conf"]
```

Es como la de Kea de IPv4, pero con `isc-kea-dhcp6`. Le metí también `tcpdump` y `dhcrelay`, que luego me sirvieron para buscar fallos.

### La imagen cliente6

En `~/kea6/cliente6`:

```dockerfile
FROM alpine:3.19
RUN apk add --no-cache dhcpcd tcpdump
CMD ["sh"]
```

```bash
cd ~/kea6
docker build -t kea6 .
docker build -t cliente6 cliente6
```

![Construcción de las imágenes kea6 y cliente6](capturas/01_build_imagenes.png)

### La configuración de Kea DHCPv6

Igual que en IPv4, con un script, `generar_kea6.py`, que saca los ficheros. Hay dos modos y uno por servidor:

```bash
cd ~/kea6
for m in stateless stateful; do for n in 1 2; do
  python3 generar_kea6.py --modo $m --este kea6-$n > compartido/kea6-$m-$n.conf
done; done
```

Salen cuatro ficheros: `kea6-stateless-1.conf`, `kea6-stateless-2.conf`, `kea6-stateful-1.conf` y `kea6-stateful-2.conf`.

Lo que cambia entre los dos modos es solo esto:

```json
{ "id": 103, "subnet": "2007:28:17:103::/64" }
```

en stateless (sin pool), y

```json
{ "id": 103, "subnet": "2007:28:17:103::/64",
  "pools": [ { "pool": "2007:28:17:103::1000-2007:28:17:103::1fff" } ] }
```

en stateful (con pool).

Lo demás es común:

- DNS `2007:28:17:102::1` (el MikroTik) y dominio `json.local`.
- Tiempos cortos (renovar a los 150 s, préstamo de 600 s) para ver la renovación.
- API por HTTP en el 8000.
- HA en load-balancing por el 8001, igual que en IPv4.

Hay tres cosas que en IPv4 no hacían falta y aquí sí (salieron como errores, apartado 7):

```json
"interfaces-config": {
  "interfaces": [ "eth0/2007:28:17:101::3" ],
  "service-sockets-max-retries": 10,
  "service-sockets-retry-wait-time": 3000
}
```

- `eth0/2007:28:17:101::3`: que Kea escuche también en su IPv6 global, que es donde mandan los relays.
- Los reintentos: para que Kea vuelva a intentar abrir el socket si la IPv6 todavía no está lista.
- En la subred de su propia LAN, `"interface": "eth0"`. Sino, Kea no sabe qué subred dar a los clientes que le llegan sin relay.

### Los nodos en GNS3

Edit config de kea6-1 (en kea6-2 se cambia 101 por 106):

```
auto eth0
iface eth0 inet static
	address 10.0.101.3
	netmask 255.255.255.0
	gateway 10.0.101.1
	up sysctl -w net.ipv6.conf.all.disable_ipv6=0
	up sysctl -w net.ipv6.conf.eth0.disable_ipv6=0
	up sysctl -w net.ipv6.conf.eth0.accept_ra=0
	up ip -6 addr add 2007:28:17:101::3/64 dev eth0
	up ip -6 route add default via 2007:28:17:101::1
```

En los contenedores el IPv6 viene apagado, y las dos primeras líneas `sysctl` lo encienden. `accept_ra=0` es para que el servidor no se autoconfigure con los RA de su router.

El volumen compartido lo hice igual que en Kea de IPv4: directorios persistentes `/etc/kea/compartido` y `/var/lib/kea`, y después cambié la carpeta de cada nodo por un enlace a `~/kea6/compartido`:

```bash
for id in $(docker ps -aq --filter ancestor=kea6); do
  docker inspect $id --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}' | grep compartido
done
D=~/GNS3/projects/Proyecto-JRM.4/project-files/docker
for d in "$D/1d820db8-772a-4b00-9d05-7a38acbaa720/etc/kea/compartido" "$D/fd54faf5-ee22-41d2-805a-33a8e2207928/etc/kea/compartido"; do
  sudo rm -rf "$d" && sudo ln -s ~/kea6/compartido "$d"
done
```

![Enlaces del volumen compartido de los kea6](capturas/02_volumen_compartido.png)

El Start command decide el modo:

| Modo | kea6-1 | kea6-2 |
|---|---|---|
| Stateless | `kea-dhcp6 -c /etc/kea/compartido/kea6-stateless-1.conf` | `kea-dhcp6 -c /etc/kea/compartido/kea6-stateless-2.conf` |
| Stateful | `kea-dhcp6 -c /etc/kea/compartido/kea6-stateful-1.conf` | `kea-dhcp6 -c /etc/kea/compartido/kea6-stateful-2.conf` |

Edit config de los cliente6:

```
auto eth0
iface eth0 inet manual
	up sysctl -w net.ipv6.conf.all.disable_ipv6=0
	up sysctl -w net.ipv6.conf.eth0.disable_ipv6=0
```

Con eso el kernel ya hace SLAAC solo. Para stateless y stateful hay que lanzar `dhcpcd -6 -d -B eth0`, porque el kernel de Linux nunca hace DHCPv6 por su cuenta.

---

## 3. SLAAC

En SLAAC no hace falta Kea. Solo hay que conseguir que el router mande el RA con el prefijo y sin M ni O.

En Wireshark capturé siempre en el cable del cliente, con este filtro:

```
icmpv6.type == 133 || icmpv6.type == 134 || icmpv6.type == 135
```

### 3.1 FRR (Luxemburgo, LAN 103) con el cliente6-1

El FRR trae los RA apagados. En vtysh:

```
conf t
interface eth6
 no ipv6 nd suppress-ra
 ipv6 nd ra-interval 10
 ipv6 nd prefix 2007:28:17:103::/64
end
write memory
```

- `no ipv6 nd suppress-ra` enciende los RA.
- `ra-interval 10` hace que mande uno cada 10 s (por defecto tarda mucho más), así se ven enseguida en Wireshark.

![RA activados en el FRR](capturas/03_frr_ra_slaac.png)

En la captura salía todo el proceso:

1. Los RA del FRR cada 10 s, a `ff02::1` (paquetes 1, 5 y 6).
2. Al arrancar el cliente, un NS desde `::` (el DAD de su link-local, paquete 8).
3. El RS del cliente a `ff02::2` (paquete 11).
4. El RA de respuesta del FRR, en el mismo instante (paquete 12).
5. Otro NS desde `::` (paquete 13): el DAD de la dirección global que se acababa de montar.

![Captura de SLAAC en la LAN del FRR](capturas/04_frr_slaac_captura.png)

En el RA:

- `Flags: 0x00`, o sea `Managed address configuration: Not set` y `Other configuration: Not set`.
- `Router lifetime: 30`: el FRR se anuncia como puerta de enlace durante 30 s y lo renueva con cada RA.
- El prefijo `2007:28:17:103::/64` con `Flag: 0xc0`, que son los bits L (*on-link*) y A (*autonomous*).

![Bits M y O del RA del FRR](capturas/05_frr_ra_flags.png)

![Router lifetime y prefijo del RA del FRR](capturas/06_frr_ra_prefijo.png)

En el cliente, la dirección se la monta con su MAC (formato EUI-64): `2007:28:17:103:42:aff:fef1:6700`.

![Dirección SLAAC del cliente6-1](capturas/07_cliente6-1_slaac.png)

Esta última es de después de hacer stateless y stateful, cuando volví a dejar el FRR en SLAAC (quitando los bits y el `no-autoconfig`):

![FRR de vuelta a SLAAC](capturas/30_frr_vuelta_slaac.png)

### 3.2 Olive (Liberia, LAN 104) con el cliente6-3

En las Olive los RA hay que activarlos a mano. Lo hice en las tres (Letonia con la 101, Liberia con la 104 y Laos con la 106):

```
configure
set protocols router-advertisement interface em5.0 prefix 2007:28:17:104::/64
set protocols router-advertisement interface em5.0 max-advertisement-interval 10
commit and-quit
```

![RA activados en Liberia](capturas/08_liberia_ra_config.png)

Para comprobarlo:

![show ipv6 router-advertisement en Liberia](capturas/09_liberia_show_ra.png)

Aquí tuve un fallo de cableado (error 12): el switch del cliente estaba en la `e6` de la Olive y no en la `e5`. Una vez arreglado, el cliente cogió su dirección y su ruta por defecto:

```
/ # ip -6 route
2007:28:17:104::/64 dev eth0  metric 256
default via fe80::eaa:caff:fe75:5 dev eth0  metric 1024
```

`fe80::eaa:caff:fe75:5` es la link-local de la `em5` de Liberia.

En la captura sale el proceso dos veces: al arrancar el cliente y al hacer `ip link set eth0 down` / `up`.

![Captura de SLAAC en la LAN de Liberia](capturas/11_olive_slaac_captura.png)

Me fijé en dos cosas:

- Los RA periódicos de la Olive no salen cada 10 s exactos, sino cada 8 o 9. JunOS elige un tiempo al azar por debajo del máximo, para que si hay varios routers no manden a la vez.
- El FRR contestaba al RS al instante, y la Olive tarda entre 0,3 y 0,5 s (del paquete 22 al 23). La norma dice que el router espere un tiempo aleatorio de hasta 0,5 s, y JunOS lo cumple.

Los NS del DAD van a `ff02::1:ff78:cd00`. Es la dirección *solicited-node*: `ff02::1:ff` más los tres últimos bytes de la IP.

El RA de la Olive (paquete 23), con `Flags: 0x00`, `Router lifetime 30`, su MAC (`0c:aa:ca:75:00:05`) y el prefijo `2007:28:17:104::/64`:

![RA de la Olive](capturas/12_olive_ra_flags.png)

### 3.3 Cisco (Libano, LAN 105) con el PC3

El Cisco ya manda RA por defecto, porque tiene `ipv6 unicast-routing` y una IPv6 en la interfaz. Solo me aseguré de que no llevaba los bits y bajé el intervalo:

```
conf t
interface FastEthernet5/0
 no ipv6 nd managed-config-flag
 no ipv6 nd other-config-flag
 ipv6 nd ra interval 10
end
show ipv6 interface FastEthernet5/0
```

Al final del `show`, el Cisco lo dice él mismo: **`Hosts use stateless autoconfig for addresses.`**

![show ipv6 interface en el Cisco](capturas/13_cisco_show_ipv6.png)

En el PC3:

```
ip auto
show ipv6
```

El VPCS se montó la dirección `2007:28:17:105:2050:79ff:fe66:6802/64`, que sale de su MAC `00:50:79:66:68:02` con `ff:fe` en medio y el segundo bit cambiado (`00` → `02`), y aprendió el router por su MAC (`ca:01:32:f0:00:8c`).

![PC3 con SLAAC](capturas/14_pc3_ip_auto.png)

La captura la hice en el cable R1 `f5/0` ↔ PC3 (aquí no hay switch). El RA del Cisco sale con `Flags: 0x00` y lleva una opción más que los otros, el MTU (1500). Además anuncia `Router lifetime: 1800`, cuando el FRR y la Olive anunciaban 30.

![Captura de SLAAC en la LAN del Cisco](capturas/15_cisco_slaac_captura.png)

### 3.4 MikroTik (Lituania, LAN 102) con el Windows

En la práctica de DHCP había puesto la IPv6 de la LAN con `advertise=no`, así que el MikroTik no mandaba RA:

```
/ipv6 address set [find address="2007:28:17:102::1/64"] advertise=yes
/ipv6 nd add interface=ether7 ra-interval=10s-20s managed-address-configuration=no other-configuration=no
```

La segunda vez que lo lancé dio `failure: configuration for this interface already exists`, porque la entrada ya la había creado el primer día. Con `/ipv6 nd print` se ve que hay dos: la de por defecto (`interface=all`, RA cada 3 a 10 minutos) y la mía de `ether7` (cada 10 a 20 s, sin M ni O).

![Configuración de ND en el MikroTik](capturas/16_mikrotik_nd.png)

En el Windows, para que mande un RS nuevo, desactivé y activé la tarjeta (cmd como administrador):

```
netsh interface set interface "Ethernet 2" disable
netsh interface set interface "Ethernet 2" enable
ipconfig /all
```

En la captura (cable Windows ↔ Switch2) sale el RA del MikroTik con `Flags: 0x00`, `Router lifetime 1800` y el prefijo `2007:28:17:102::/64`:

![Captura de SLAAC en la LAN del MikroTik](capturas/17_mikrotik_slaac_captura.png)

En el `ipconfig /all` del Windows salen cosas interesantes:

- **Dirección IPv6** `2007:28:17:102:24ce:d515:5f4c:9c5a`: es la de SLAAC. Windows no usa la MAC, se inventa el final.
- **Dirección IPv6 temporal** `2007:28:17:102:995c:864:e4bc:a95e`: otra de SLAAC, que va cambiando por privacidad.
- **Puerta de enlace** `fe80::e5a:eeff:fe2f:6%3`: la link-local del MikroTik, sacada del RA.
- Y además una Dirección IPv6 `2007:28:17:102::1000` con *Concesión obtenida*, Servidores DNS `2007:28:17:102::1` e IAID y DUID de cliente DHCPv6. Esa no es de SLAAC: se la dio Kea6 por DHCPv6, a través del relay del MikroTik (apartado 6). Los kea6 estaban en modo stateful y Windows, al reactivar la tarjeta, también pidió por DHCPv6. Así que el relay del MikroTik funciona.

![ipconfig /all del Windows](capturas/18_windows_ipconfig.png)

### 3.5 Comparación

| Router | Cliente | Dirección | Router lifetime |
|---|---|---|---|
| FRR | cliente6-1 (Alpine) | EUI-64 con la MAC (`…:42:aff:fef1:6700`) | 30 s |
| Olive | cliente6-3 (Alpine) | EUI-64 con la MAC | 30 s |
| Cisco | PC3 (VPCS) | EUI-64 con la MAC (`…:2050:79ff:fe66:6802`) | 1800 s |
| MikroTik | Windows | Aleatoria + temporal | 1800 s |

---

## 4. Stateless

Los stateless y stateful los probé en la LAN 103 (FRR), con el cliente6-1. En esa LAN no hay ningún kea6, así que además probaba el relay.

### En el router: el bit O

```
conf t
interface eth6
 ipv6 nd other-config-flag
end
write memory
```

![Bit O en el FRR](capturas/19_frr_other_flag.png)

Ahora el RA sale con `Flags: 0x40`: `0100 0000` en binario, M a 0 y O a 1.

### El relay DHCPv6 en el FRR

FRR no trae relay, así que usé `dhcrelay` (el mismo que en IPv4, pero con `-6`):

```sh
echo "nameserver 10.0.102.1" > /etc/resolv.conf
apk add dhcrelay
dhcrelay -6 -l eth6 -u 2007:28:17:101::3%eth0 -u 2007:28:17:106::3%eth5 &
```

- `-l eth6`: la interfaz de abajo, la de los clientes.
- `-u dirección%interfaz`: cada kea6 y por qué interfaz se llega a él.

Al principio lo lancé con `%eth0` en los dos, y eso me dio el error 3:

![Primer dhcrelay -6 con %eth0 en los dos](capturas/20_frr_dhcrelay6_eth0.png)

El de IPv4 sigue siendo otro proceso aparte:

```sh
dhcrelay -4 -id eth6 -iu eth0 -iu eth1 -iu eth2 -iu eth4 -iu eth5 10.0.102.2 10.0.101.2 10.0.106.2 &
```

### En Kea

Start command con `kea6-stateless-1.conf` y `kea6-stateless-2.conf`. Comprobé el HA desde el Ubuntu:

```bash
curl -s -u kea:Kea12345 -H "Content-Type: application/json" -d '{"command":"ha-heartbeat"}' http://10.0.106.3:8000/
```

```
"scopes": [ "kea6-2" ], "state": "load-balancing"
```

### En el cliente

```sh
dhcpcd -6 -d -B eth0
```

Antes de que funcionara, el cliente mandaba Information-request sin parar y nadie contestaba (errores 3 a 6):

![Information-request sin respuesta](capturas/22_error_info_request_sin_reply.png)

Cuando ya funcionaba, en la captura salía el Information-request y justo después el Reply (paquete 1857). Dentro del Reply:

- **Server Identifier**: el DUID del kea6 que contesta.
- **DNS recursive name server**: `2007:28:17:102::1`.
- **Domain Search List**: `json.local` (se ve al final del hexadecimal).

No hay ninguna dirección IP: Kea no da IP, solo el DNS y el dominio. La IP sigue siendo la de SLAAC.

![Reply de stateless con DNS y dominio](capturas/23_stateless_reply.png)

---

## 5. Stateful

### En el router: el bit M y el prefijo sin autoconfiguración

```
conf t
interface eth6
 ipv6 nd managed-config-flag
 ipv6 nd prefix 2007:28:17:103::/64 no-autoconfig
end
write memory
```

- `managed-config-flag` pone el bit M a 1.
- `no-autoconfig` pone el bit A del prefijo a 0, para que el cliente no se monte también su dirección SLAAC y solo tenga la de Kea.

El RA sale con `Flags: 0xc0` (M y O a 1) y en el prefijo `Autonomous: Not set`.

### En Kea y en el cliente

Start command con `kea6-stateful-1.conf` y `kea6-stateful-2.conf`, y en el cliente otra vez `dhcpcd -6 -d -B eth0`. Ahora `dhcpcd` decía `soliciting a DHCPv6 lease` (en stateless decía que pedía información).

| Nº | Mensaje | Qué dice |
|---|---|---|
| 2615 | Solicit | "¿Algún servidor me da una IP?" |
| 2616 | Advertise | "Te ofrezco la `2007:28:17:103::1000`" |
| 2617 | Request | "Vale, quiero esa" |
| 2618 | Reply | "Confirmado" |

`2007:28:17:103::1000` es la primera IP del pool.

En esa misma captura se ve toda la historia: arriba el stateless (Information-request y el Reply 1857), en medio un montón de Solicit sin respuesta (cuando los kea6 estaban caídos por el error 7) y al final el SARR.

![Stateless, Solicit sin respuesta y el SARR de stateful](capturas/24_stateful_sarr.png)

En el cliente, la dirección sale con /128, no con /64. DHCPv6 solo da la dirección, no la red. Que la `2007:28:17:103::/64` está "al lado" lo sabe por el RA (el bit L, *on-link*). Y la puerta de enlace también viene del RA.

El Windows de la LAN 102 también cogió una dirección del pool por stateful (la `2007:28:17:102::1000`, apartado 3.4), así que el stateful funcionó a través de dos relays distintos: el `dhcrelay` del FRR y el relay del MikroTik.

---

## 6. Los relays DHCPv6 en los demás routers

Para que stateless y stateful funcionen en todas las LAN, cada router que no tiene un kea6 en su red tiene que hacer de relay. Los configuré todos.

**Cisco (Libano):**

```
interface FastEthernet5/0
 ipv6 dhcp relay destination 2007:28:17:101::3
 ipv6 dhcp relay destination 2007:28:17:106::3
```

![Relay DHCPv6 en el Cisco](capturas/25_cisco_relay6.png)

**MikroTik (Lituania):**

```
/ipv6 dhcp-relay add name=relay6 interface=ether7 dhcp-server=2007:28:17:101::3,2007:28:17:106::3 link-address=2007:28:17:102::1 disabled=no
```

![Relay DHCPv6 en el MikroTik](capturas/26_mikrotik_relay6.png)

**Olive.** Con DHCPv4 la Olive no podía ser servidor, así que pensaba que el relay de DHCPv6 tampoco funcionaría. Pero el `commit` lo aceptó sin quejarse en las tres:

```
configure
set forwarding-options dhcp-relay dhcpv6 server-group KEA6 2007:28:17:106::3
set forwarding-options dhcp-relay dhcpv6 group LAN active-server-group KEA6
set forwarding-options dhcp-relay dhcpv6 group LAN interface em5.0
commit and-quit
```

Letonia solo manda a kea6-2, porque kea6-1 ya está en su LAN; Laos solo a kea6-1, y Liberia a los dos.

![Relay DHCPv6 en Letonia](capturas/27_letonia_relay6.png)

![Relay DHCPv6 en Liberia](capturas/28_liberia_relay6.png)

![Relay DHCPv6 en Laos](capturas/29_laos_relay6.png)

Los que he comprobado que funcionan de verdad son el del FRR (captura del SARR) y el del MikroTik (la `::1000` del Windows).

---

## 7. Errores que me salieron

**1. `dhcrelay: not found` en el FRR.** El FRR se había reiniciado y había perdido `dhcrelay`, porque está instalado en caliente y no en la imagen. Hubo que poner el DNS del MikroTik en `/etc/resolv.conf`, hacer `apk add dhcrelay` y lanzar los dos relays (el de IPv4 y el de IPv6).

**2. `send_packet6: Network unreachable` en el relay.** El relay recibía las peticiones (`Relaying Information-request ... going up`), pero no podía mandarlas. Había puesto `%eth0` en los dos servidores, y a la LAN 106 se va por `eth5`:

```
/ # ip -6 route get 2007:28:17:106::3
2007:28:17:106::3 from :: via fe80::eaf:c9ff:fee2:3 dev eth5 ...
```

Con `dhcrelay -6`, el `%interfaz` obliga a salir por ahí. Lo arreglé poniendo a cada servidor su interfaz: `%eth0` para la 101 y `%eth5` para la 106. En IPv4 no pasaba, porque `-iu` acepta varias interfaces. El problema es que si OSPF cambia el camino, el relay vuelve a fallar.

**3. Kea no escuchaba en su IPv6 global.** Kea DHCPv6, por defecto, solo escucha en la multicast `ff02::1:2` y en la link-local. Los relays mandan a la dirección global (`2007:28:17:101::3`), así que Kea no recogía nada. Se arregla poniendo la dirección en `interfaces`: `"eth0/2007:28:17:101::3"`.

**4. `Cannot assign requested address` al abrir el socket.** Este fue el más difícil de encontrar. Kea arrancaba, contestaba a la API y el HA estaba en `load-balancing`, pero `ss -uln` no enseñaba nada en el 547. Con `tcpdump` dentro del kea6-1 se veía que los `relay-fwd` del FRR sí llegaban. En el log:

```
DHCPSRV_OPEN_SOCKET_FAIL failed to open socket: ... Failed to bind socket 17 to 2007:28:17:101::3/port=547: Cannot assign requested address
DHCP6_OPEN_SOCKETS_FAILED maximum number of open service sockets attempts: 0, has been exhausted without success
DHCPSRV_NO_SOCKETS_OPEN no interface configured to listen to DHCP traffic
```

En IPv6, cuando se pone una dirección, el sistema hace el DAD (comprueba que nadie más la usa) y durante unos segundos la dirección está *tentative* y no se puede usar. El contenedor arrancaba, ponía la IPv6 y lanzaba Kea en el mismo momento, así que Kea intentaba abrir el socket antes de que acabara el DAD. Como por defecto no lo vuelve a intentar, se quedaba sin escuchar para siempre.

En el momento lo arreglé pidiéndole a Kea que recargara la configuración, cuando las direcciones ya estaban listas:

```bash
curl -s -u kea:Kea12345 -H "Content-Type: application/json" -d '{"command":"config-reload"}' http://10.0.101.3:8000/
```

Y para que no volviera a pasar, añadí los reintentos (`service-sockets-max-retries` y `service-sockets-retry-wait-time`). Después ya abría el socket él solo al arrancar:

```
UNCONN 0 0   [2007:28:17:101::3]:547          [::]:*
UNCONN 0 0   [fe80::42:54ff:fefa:f500]%eth0:547   [::]:*
UNCONN 0 0   [ff02::1:2]%eth0:547             [::]:*
```

**5. `failure: configuration for this interface already exists` en el MikroTik.** Al repetir el `/ipv6 nd add` de `ether7`. La entrada ya existía del primer día. Si hay que cambiarla, se usa `set` en vez de `add`: `/ipv6 nd set [find interface=ether7] ...`.

---

## 8. ¿Qué pasa con el exporter?

- **SLAAC:** el cliente se configura solo con el RA y no habla con ningún servidor. Kea no se entera, así que kea-exporter no puede ver esos equipos. Para saber qué equipos hay habría que sacar la tabla de vecinos IPv6 (NDP) de los routers, por ejemplo por SNMP.
- **Stateless:** Kea contesta, pero no apunta ninguna concesión, porque no da IP. Solo suben los contadores de Information-request.
- **Stateful:** sí hay concesiones, como en IPv4. Se ven con la API (`lease6-get-all`) y en el exporter.

Para meter los kea6 en el exporter hay que añadirlos a la variable `TARGETS` del nodo kea-exporter:

```
TARGETS=http://kea:Kea12345@10.0.101.2:8000 http://kea:Kea12345@10.0.106.2:8000 http://kea:Kea12345@10.0.101.3:8000 http://kea:Kea12345@10.0.106.3:8000
```

El exportador de ISC que hice para IPv4 no sirve aquí: solo lee el `dhcpd.leases` de IPv4.
