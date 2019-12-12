# Cortafuegos-FTP
## Modo activo
El servidor FTP tiene dos puertos abiertos (21 de control y 20 de datos).
El cliente a su vez tiene el puerto (1036 de datos y el 1035 de control). Pero estos puertos son aleatorios.
Las conecciones que se realizan son:
1. -> Cliente del 1035 al servidor 21. PORT 
2. <- Servidor 21 al cliente 1035. ACK 
3. <- Servidor 20 al cliente 1036. Envia datos 
4. -> Cliente 1036 al servidor 20. ACK 

Entre el punto 2 y el 2 puede suceder que se encuentre un cortafuegos que no permita la comunicación del segundo paquete del servidor ftp.

## Modo pasivo
1. -> Cliente desde 1035 al 21 del servidor. PASV
2. <- Servidor desde 21 al 1035. Respuesta a un puerto mayor al 1023.
3. -> Cliente desde 1036 al servidor 2040. el destino es aleatorio.
4. <- Servidor desde 2040 al cliente 1036. ACK

## Módulo de iptables para solucionar este problema. 
#### Práctica: Cortafuegos perimetral con DMZ
**Esquema de red**
Vamos a utilizar tres máquinas en openstack, que vamos a crear con la receta heat: [escenario3.yaml](https://fp.josedomingo.org/seguridadgs/u03/escenario3.yaml). La receta heat ha deshabilitado el cortafuego que nos ofrece openstack (todos los puertos de todos los protocolos están abiertos). Una máquina (que tiene asignada una IP flotante) hará de cortafuegos, otra será una máquina de la red interna 192.168.100.0/24 y la tercera será un servidor en la DMZ donde iremos instalando distintos servicios y estará en la red 192.168.200.0/24.

**Ejercicios**

Configurar un cortafuegos perimetral en la máquina **router-fw** teniendo en cuenta los siguientes puntos:
##### 1. Política por defecto DROP para las cadenas INPUT, FORWARD y OUTPUT.

Se configuran las reglas de ssh antes de las reglas por defecto:

~~~
iptables -A INPUT -s 172.22.0.0/16 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -d 172.22.0.0/16 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
iptables -A INPUT -s 172.23.0.0/16 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -d 172.23.0.0/16 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
~~~

~~~
root@router-fw:/home/debian# iptables -A INPUT -s 172.22.0.0/16 -p tcp --dport 22 -j ACCEPT
root@router-fw:/home/debian# iptables -A OUTPUT -d 172.22.0.0/16 -p tcp --sport 22 -j ACCEPT
root@router-fw:/home/debian# iptables -A INPUT -s 172.23.0.0/16 -p tcp --dport 22 -j ACCEPT
root@router-fw:/home/debian# iptables -A OUTPUT -d 172.23.0.0/16 -p tcp --sport 22 -j ACCEPT
~~~

Y se configuran las reglas por defecto:
~~~
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP
~~~

~~~
root@router-fw:/home/debian# iptables -P INPUT DROP
root@router-fw:/home/debian# iptables -P OUTPUT DROP
root@router-fw:/home/debian# iptables -P FORWARD DROP
~~~

Y las reglas de ssh para conectarse a los clientes:
~~~
iptables -A OUTPUT -p tcp -o eth1 -d 192.168.100.0/24 --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT -p tcp -i eth1 -s 192.168.100.0/24 --sport 22 -m state --state ESTABLISHED -j ACCEPT
iptables -A OUTPUT -p tcp -o eth2 -d 192.168.200.0/24 --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT -p tcp -i eth2 -s 192.168.200.0/24 --sport 22 -m state --state ESTABLISHED -j ACCEPT
~~~

~~~
root@router-fw:/home/debian# iptables -A OUTPUT -p tcp -o eth1 -d 192.168.100.0/24 --dport 22 -j ACCEPT
root@router-fw:/home/debian# iptables -A INPUT -p tcp -i eth1 -s 192.168.100.0/24 --sport 22 -j ACCEPT
root@router-fw:/home/debian# iptables -A OUTPUT -p tcp -o eth2 -d 192.168.200.0/24 --dport 22 -j ACCEPT
root@router-fw:/home/debian# iptables -A INPUT -p tcp -i eth2 -s 192.168.200.0/24 --sport 22 -j ACCEPT
~~~


##### 2. Se pueden usar las extensiones que queremos adecuadas, pero al menos debe implementarse seguimiento de la conexión.


##### 3. Debemos implementar que el cortafuego funcione después de un reinicio de la máquina.

Las reglas que hay actualmente son:
~~~
root@router-fw:/home/debian# iptables -L -n --line-numbers
Chain INPUT (policy DROP)
num  target     prot opt source               destination         
1    ACCEPT     tcp  --  172.22.0.0/16        0.0.0.0/0            tcp dpt:22
2    ACCEPT     tcp  --  172.23.0.0/16        0.0.0.0/0            tcp dpt:22
3    ACCEPT     tcp  --  192.168.100.0/24     0.0.0.0/0            tcp spt:22
4    ACCEPT     tcp  --  192.168.200.0/24     0.0.0.0/0            tcp spt:22

Chain FORWARD (policy DROP)
num  target     prot opt source               destination         
1    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:22 state NEW,ESTABLISHED
2    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:22 state ESTABLISHED

Chain OUTPUT (policy DROP)
num  target     prot opt source               destination         
1    ACCEPT     tcp  --  0.0.0.0/0            172.22.0.0/16        tcp spt:22
2    ACCEPT     tcp  --  0.0.0.0/0            172.23.0.0/16        tcp spt:22
3    ACCEPT     tcp  --  0.0.0.0/0            192.168.100.0/24     tcp dpt:22
4    ACCEPT     tcp  --  0.0.0.0/0            192.168.200.0/24     tcp dpt:22
~~~

Se guardan las reglas:
~~~
iptables-save > /etc/iptables/rules.v4
~~~

Y se hace un script
https://docs.google.com/presentation/d/e/2PACX-1vSDP6RNlDWZV2JnBES3u-IPGf4_F8TNYzOjKiESMAcWxS741Ise6kGCmTawhfAn0Q34qB8MHSJ2DRdu/pub?start=false&loop=false&delayms=3000#slide=id.g6b936b7496_0_18

Se activa el forward de forma permanente habilitando la siguiente línea de **/etc/sysctl.conf**
~~~
# Uncomment the next line to enable packet forwarding for IPv4
net.ipv4.ip_forward=1
~~~

Se habilita el bit de forward:
~~~
root@router-fw:/home/debian# echo 1 > /proc/sys/net/ipv4/ip_forward
~~~

Y se ejecuta el siguiente comando:
~~~
root@router-fw:/home/debian# sysctl -p /etc/sysctl.conf
net.ipv4.ip_forward = 1
~~~

Comprobación. Se reinicia la máquina

##### 4. Debes indicar pruebas de funcionamiento de todos las reglas.

##### 5. El cortafuego debe cumplir al menos estas reglas:
- La máquina router-fw tiene un servidor ssh escuchando por el puerto 22, pero al acceder desde el exterior habrá que conectar al puerto 2222.

iptables -I OUTPUT -p tcp -s 172.22.0.0/16 --dport 2222 -j ACCEPT
iptables -I INPUT -p tcp -d 172.22.0.0/16 --sport 2222 -j ACCEPT

iptables -I OUTPUT -p tcp -s 172.23.0.0/16 --dport 2222 -j ACCEPT
iptables -I INPUT -p tcp -d 172.23.0.0/16 --sport 2222 -j ACCEPT


iptables -A OUTPUT -p tcp -o eth1 -d 192.168.100.0/24 --dport 22 -j ACCEPT

- Desde la LAN y la DMZ se debe permitir la conexión ssh por el puerto 22 al la máquina router-fw.
- La máquina router-fw debe tener permitido el tráfico para la interfaz loopback.
- A la máquina router-fw se le puede hacer ping desde la DMZ, pero desde la LAN se le debe rechazar la conexión (REJECT).
- La máquina router-fw puede hacer ping a la LAN, la DMZ y al exterior.
- Desde la máquina DMZ se puede hacer ping y conexión ssh a la máquina LAN.
- Desde la máquina LAN no se puede hacer ping, pero si se puede conectar por ssh a la máquina DMZ.
- Configura la máquina router-fw para que las máquinas LAN y DMZ puedan acceder al exterior.
- La máquina LAN se le permite hacer ping al exterior.
- La máquina LAN puede navegar.
- La máquina DMZ puede navegar. Instala un servidor web, un servidor ftp y un servidor de correos.
- Configura la máquina router-fw para que los servicios web y ftp sean accesibles desde el exterior.
- El servidor web y el servidor ftp deben ser accesible desde la LAN y desde el exterior.
- El servidor de correos sólo debe ser accesible desde la LAN.
- En la máquina LAN instala un servidor mysql. A este servidor sólo se puede acceder desde la DMZ.

##### 6. Si crees que necesitas más reglas de las que nos han indicado, describe porque pueden ser necesarias.

##### 7. MEJORA: Utiliza nuevas cadenas para clasificar el tráfico.

##### 8. MEJORA: Consruye el cortafuego utilizando nftables.

