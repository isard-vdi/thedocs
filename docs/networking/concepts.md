#Network linux commands



# Link layer

Show interfaces:

```
ip link show
```

Enable/disable an interface:

```
ip link set eth0 down
ip link set eth0 up
```

Change name or address to interface manually:

```
ip link set eth0 down
ip link set eth0 address 00:11:22:33:44:55
ip link set eth0 name lan
ip link set eth0 up
```

Show interface options physical options

```
ethtool eth0
```

Show speed of interface:
```
ethtool eth0 |grep Speed
```

Kernel module that use the interface:
```
ethtool -i eth0
```

Show statistics, error packets, collisions:
```
ethtool -S eth0
```

## Vlans

Add virtual interface with tagged vlan 101 to eth0:

```
ip link add link eth0 name vlan101 type vlan id 101
ip link set vlan101 up
```

## Bonding

```
ip link set eth1 down
ip link set eth2 down


modprobe -r bonding
modprobe bonding mode=4

# default xmit_hash_policy is 0 (hash with mac adress)
# modprobe bonding mode=4 xmit_hash_policy=2

ip link set eth1 master bond0
ip link set eth2 master bond0

ip link set eth1 up 
ip link set eth2 up

ip link set bond0 up

cat /proc/net/bonding/bond0

``` 

## Test spped network with iperf

Traditional tool is **iperf**, with speeds greater than 1Gbps new tool with more options **iperf3** is better

```
#Server
iperf -s 
#Client
iperf -c SERVER_IP

```

Some useful options:

```
#Server
iperf -s -i 2 -p 5002
#Client
iperf -p 5002 -c SERVER_IP

```

-t to increase duration of test
 
```
#Server
iperf -s -i 2
#Client
iperf -t 30 -c SERVER_IP

```

## Bridges

Create bridge:
```
brctl addif br0 usb0
```


Assign interfaces to bridge:

```
brctl addif br0 eth0
brctl addif br0 usb0
brctl addif br0 usb1
brctl delif br0 usb1
```

To show interfaces members of a bridge and table of ports and mac address:

```
brctl show
brctl show br0
brctl showmacs br0
```



## Capture packets

**tcpdump**: herramienta clásica para hacer capturas
```
tcpdump -w 08232010.pcap -i eth0
```

**tshark**: wireshark in command line, capture filters show [https://wiki.wireshark.org/CaptureFilters]

* **\[ -i \]** Input interface
* **\[ -w \]** Output file
* **\[ -r \]** Read options from file
* **\[ -c \]**  limit packets captured
* **\[ -a duration:X\]** seconds when stop capture
* **\[ -Y \]** Wireshark filters

Examples:

```
tshark -i enp3s0 -w /tmp/out.pcap -c 10
tshark -i enp3s0 -w /tmp/out.pcap -a duration:4
tshark -r /tmp/out.pcap -w /tmp/web.pcap -Y "http"

tshark -i enp3s0 -w /tmp/ping.pcap "host 8.8.8.8"
tshark -i enp3s0 -w /tmp/web.pcap "port 80"
```

To show in screen the packets with extended version (-x)
```
tshark -r /tmp/web.pcap
tshark -r /tmp/web.pcap -x 
tshark -r /tmp/web.pcap -x -Y "frame.number==10"
```

# Network Layer

Show ip address, short command of **ip address show**:
```
ip a

```

Show stats (tx,rx,error,dropped)

```
ip -statistics a
```

## Fixed ip address:

Delete all address of an interface (flush):

```
ip a f dev eth0
```

Asociar una ip a una interfaz (no olvidar la máscara):
```
ip a a 192.168.100.10/24 dev eth0
```

Se pueden asociar más de una ip a una interfaz y eliminar una en concreto:
```
ip a a 192.168.100.10/24 dev eth0
ip a a 192.168.200.11/27 dev eth0

ip a d 192.168.200.11/27 dev eth0

```

## Ips dinámicas:

Liberar la ip actual:
```
dhclient -r
```

Esto debería haber liberado la ip actual y el daemon debería haber finalizado. En caso de que no podamos pedir una nueva ip no nos queda otra más que matar ese proceso con:

```
killall dhclient
```

Para pedir una nueva ip en cualquier interface:
```
dhclient
```

Y en una interface en concreto:
```
dhclient eth0
```

Si quieres ver más detalle de la concesión de ip:
```
dhclient -v eth0
dhclient -v -lf /tmp/eth0.lease
cat /tmp/eth0.lease
```

## Tabla ARP

La orden tradicional arp ha quedado centralizada en la utilidad ip con la opción **ip neigh** 

Para ve la tabla de arp actual:

```
ip neigh
```

Borrar tabla de arp:
```
ip neigh flush all
```

Añadir entrada arp permanente:
```
ip neigh add 192.168.100.1 lladdr 00:11:22:33:44:55 dev enp3s0
```

Ver sólo las entradas reachable:
```
ip neigh show nud reachable
```

## Routing

Ver las rutas con **ip route show**:
```
ip r 
```

Para eliminar todas las rutas "ip route flush":
```
ip r f all
```

Al añadir una dirección ip a una interfaz, se añade una ruta directa
que informa que para ir a la red de esa dirección ip se va directamente
a través de la interfaz sin necesidad de pasar por ningún router:

```
$ ip r f all
$ ip a a 172.16.0.10/16 dev eth0
$ ip r s 
172.16.0.10/16  dev eth0 [...]
```

Añadir puerta de enlace por defecto:
```
ip r a default via 192.168.1.1
```

Eliminar puerta de enlace por defecto:
```
ip r d default
```

Añadir ruta estática:
```
ip r a 192.168.200.0/24 via 192.168.100.1
```

Activar bit de forwarding para que trabaje como router:
```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

## Resolución de nombres DNS

Se consulta la resolución en el fichero **/etc/hosts**, que contiene
una línea para el nombre localhost, se pueden añadir líneas para
resolver nombres sin utilizar un servidor dns:

```
$ cat /etc/hosts
127.0.0.1		localhost.localdomain localhost
```

Para resolver con servidores dns linux consulta el contenido del fichero /etc/resolv.conf
```
$ cat /etc/resolv.conf 
[...]
nameserver 192.168.1.1
```

Se consulta la línea que empieza con nameserver seguida de la ip del servidor dns que queremos usar.
Si al renovar la ip dinámica el servidor dhcp ofrece un servidor dns se sobreescribe este fichero

Para forzar la resolución de nombres a través de un servidor dns de forma manual:
```
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

## Enmascaramiento de ips

Enmascarar todo el tráfico que sale hacia internet por eth0

```
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

Consultar iptables:
```
iptables-save
iptables -t filter -L
iptables -t nat -L
```

## Pruebas de conectividad

Comprabar respuestas protocolo ARP:

```
arping 192.168.100.1
```

Hacer ping indefinidamente (para terminar ctrl + C)

```
ping 8.8.8.8
```


Limitar número de pings

```
ping -c 2 8.8.8.8
```

**fping**: Hacer pings a varios equipos de una red:

```
fping -g 192.168.100.100 192.168.100.110 -a -q
fping -g 192.168.100.0/24 -a -q

```

**nmap** 

Sondeo ping:
```
nmap -sP 192.168.100.0/24
```
Sondeo de puerto 22

## Traceroute y geolocalizar ips

**traceroute**: Herramienta histórica para descubrir rutas

```
traceroute 8.8.8.8
```

**mtr**: Si quieres más información, estadísticas y detalle de geolocalización de ips

```
dnf -y install mtr GeoIP GeoIP-GeoLite-data GeoIP-GeoLite-data-extra
mtr -n --report www.gencat.cat
geoiplookup 72.52.92.222

```

# Capa de transporte

## nmap y sondeo de puertos

Sondeo de puertos TCP típicos:
```
nmap -sS 192.168.100.1
```

Sondeo de un puerto concreto
```
nmap -sS -p 80 192.168.100.1
```

Sondeo de rango de puertos
```
nmap -sS -p 8080-8090 192.168.100.1
```

Sondeo de puertos aunque el ping no conteste, añadiendo la opción -PN
```
nmap -sS -PN -p 80 192.168.100.1
```

Sondeo de puerto UDP:
```
nmap -sU -p 53 192.168.100.1
```

## netstat

número de puertos tcp y udp escuchando con el programa asociado:

```
netstat -utlnp
```

Las opciones se pueden combinar sin importar el orden en que van las letras,
las típicas de netstat son:
* -u => puertos udp
* -t => puertos tcp
* -p => programa que está usando esa conexión y pid (sólo lo puedes ver si eres root)
* -n => muestra el número de puerto
* -a => todas las conexiones
* -l => puertos en escucha

Todas las conexiones tcp:
```
netstat -tanp 
```

## probar puertos con netcat

