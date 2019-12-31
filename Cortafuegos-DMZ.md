# Cortafuegos-DMZ
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

Se limpian las tablas:
~~~
iptables -F
iptables -t nat -F
iptables -Z
iptables -t nat -Z
~~~

Se configuran las reglas de ssh antes de las reglas por defecto:

~~~
iptables -A INPUT -s 172.22.0.0/16 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -d 172.22.0.0/16 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
iptables -A INPUT -s 172.23.0.0/16 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -d 172.23.0.0/16 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
~~~

Y las reglas de ssh para conectarse a los clientes:
~~~
iptables -A OUTPUT -p tcp -o eth1 -d 192.168.100.0/24 --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT -p tcp -i eth1 -s 192.168.100.0/24 --sport 22 -m state --state ESTABLISHED -j ACCEPT
iptables -A OUTPUT -p tcp -o eth2 -d 192.168.200.0/24 --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT -p tcp -i eth2 -s 192.168.200.0/24 --sport 22 -m state --state ESTABLISHED -j ACCEPT
~~~

Y se configuran las reglas por defecto:
~~~
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP
~~~


> Se pueden usar las extensiones que queremos adecuadas, pero al menos debe implementarse seguimiento de la conexión.


##### 3. Implementar que el cortafuego funcione después de un reinicio de la máquina.

Las reglas que hay actualmente son:
~~~
root@router-fw:/home/debian# iptables -L -n 
Chain INPUT (policy DROP)
target     prot opt source               destination         
ACCEPT     tcp  --  172.22.0.0/16        0.0.0.0/0            tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     tcp  --  172.23.0.0/16        0.0.0.0/0            tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     tcp  --  192.168.100.0/24     0.0.0.0/0            tcp spt:22 state ESTABLISHED
ACCEPT     tcp  --  192.168.200.0/24     0.0.0.0/0            tcp spt:22 state ESTABLISHED

Chain FORWARD (policy DROP)
target     prot opt source               destination         

Chain OUTPUT (policy DROP)
target     prot opt source               destination         
ACCEPT     tcp  --  0.0.0.0/0            172.22.0.0/16        tcp spt:22 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            172.23.0.0/16        tcp spt:22 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            192.168.100.0/24     tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            192.168.200.0/24     tcp dpt:22 state NEW,ESTABLISHED
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

Y se ejecuta el siguiente comando para comprobar que se ha modificado correctamente:
~~~
root@router-fw:/home/debian# sysctl -p /etc/sysctl.conf
net.ipv4.ip_forward = 1
~~~

Comprobación. Se reinicia la máquina

> Se debe indicar pruebas de funcionamiento de todos las reglas.

##### 5. El cortafuego debe cumplir al menos estas reglas:
- La máquina router-fw tiene un servidor ssh escuchando por el puerto 22, pero al acceder desde el exterior habrá que conectar al puerto 2222.

Hay que redirigir el puerto 2222 al 22.

> Además, hay que activar el 'ip_forward'
~~~
echo 1 > /proc/sys/net/ipv4/ip_forward
~~~

Redirección:
~~~
iptables -t nat -I PREROUTING -p tcp -s 172.22.0.0/16 --dport 2222 -j REDIRECT --to-ports 22
iptables -t nat -I PREROUTING -p tcp -s 172.23.0.0/16 --dport 2222 -j REDIRECT --to-ports 22
~~~

Se configura la regla para que se pueda hacer conexión desde el puerto 2222 desde los rangos adecuados:
~~~
iptables -A INPUT -s 172.22.0.0/16 -p tcp -m tcp --dport 2222 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -d 172.22.0.0/16 -p tcp -m tcp --sport 2222 -m state --state ESTABLISHED -j ACCEPT
iptables -A INPUT -s 172.23.0.0/16 -p tcp -m tcp --dport 2222 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -d 172.23.0.0/16 -p tcp -m tcp --sport 2222 -m state --state ESTABLISHED -j ACCEPT
~~~


Se indica qué se redirecciona, en este caso, hacia loopback:
~~~
iptables -t nat -I PREROUTING -p tcp -s 172.22.0.0/16 --dport 22 -j DNAT --to-destination 127.0.0.1
iptables -t nat -I PREROUTING -p tcp -s 172.23.0.0/16 --dport 22 -j DNAT --to-destination 127.0.0.1
~~~


> Comprobación:
~~~
paloma@coatlicue:~/DISCO2/CICLO II/SEGURIDAD/Cortafuegos-perimetral-con-DMZ$ ssh -A -p 2222 debian@172.22.201.43
Linux router-fw 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Dec 31 17:34:37 2019 from 172.23.0.22
debian@router-fw:~$ exit
logout
Connection to 172.22.201.43 closed.
paloma@coatlicue:~/DISCO2/CICLO II/SEGURIDAD/Cortafuegos-perimetral-con-DMZ$ ssh -A -p 22 debian@172.22.201.43
ssh: connect to host 172.22.201.43 port 22: Connection timed out
~~~


- Desde la LAN y la DMZ se debe permitir la conexión ssh por el puerto 22 a la máquina router-fw.
Se configura la conexión con la lan:
~~~
iptables -A INPUT -s 192.168.100.0/24 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -d 192.168.100.0/24 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
~~~

Se configura la conexión con la DMZ:
~~~
sudo iptables -A INPUT -s 192.168.200.10/16 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -d 192.168.200.10/16 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
~~~

> Comprobación LAN
~~~
debian@router-fw:~$ ssh -A debian@192.168.100.10 [funciona]
debian@lan:~$ ssh debian@192.168.100.2 [funciona]

~~~

> Comprobación DMZ
~~~
debian@router-fw:~$ ssh -A debian@192.168.200.10 [funciona]
debian@dmz:~$ ssh debian@192.168.200.2 [funciona]
~~~



- La máquina router-fw debe tener permitido el tráfico para la interfaz loopback.

~~~
iptables -A INPUT -i lo -p icmp -j ACCEPT
iptables -A OUTPUT -o lo -p icmp -j ACCEPT
~~~

> Comprobación
~~~
debian@router-fw:~$ ping 127.0.0.1 [no funciona]

debian@router-fw:~$ sudo iptables -A INPUT -i lo -p icmp -j ACCEPT
debian@router-fw:~$ sudo iptables -A OUTPUT -o lo -p icmp -j ACCEPT

debian@router-fw:~$ ping 127.0.0.1 [funciona]
~~~

- A la máquina router-fw se le puede hacer ping desde la DMZ, pero desde la LAN se le debe rechazar la conexión (REJECT).

Configuración para que la máquina LAN pueda hacer ping al router:
~~~
iptables -A INPUT -i eth1 -s 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-request -j REJECT --reject-with icmp-port-unreachable
iptables -A OUTPUT -o eth1 -d 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
iptables -A OUTPUT -o eth1 -d 192.168.100.0/24 -p icmp -m state --state RELATED -j ACCEPT
~~~

Configuración para que la máquina DMZ pueda hacer ping al router:
~~~
iptables -A INPUT -i eth2 -s 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A OUTPUT -o eth2 -d 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
~~~

> Comprobación LAN
~~~
debian@lan:~$ ping 192.168.100.2 [no funciona]
~~~

> Comprobación DMZ
~~~
debian@dmz:~$ ping 192.168.200.2 [funciona]
~~~

- La máquina router-fw puede hacer ping a la LAN, la DMZ y al exterior.

Configuración para poder hacer ping a la LAN:
~~~
iptables -A OUTPUT -o eth1 -d 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -i eth1 -s 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
~~~

Configuración para poder hacer ping a la DMZ:
~~~
iptables -A OUTPUT -o eth2 -d 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -i eth2 -s 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
~~~

Configuración para poder hacer ping al exterior:
~~~
iptables -A OUTPUT -o eth0 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -i eth0 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
~~~

> Comprobación LAN
~~~
debian@router-fw:~$ ping 192.168.100.10 [funciona]
~~~

> Comprobación DMZ
~~~
debian@router-fw:~$ ping 192.168.200.10 [funciona]
~~~

> Comprobación con el exterior
~~~
debian@router-fw:~$ ping 1.1.1.1 [funciona]
~~~


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

