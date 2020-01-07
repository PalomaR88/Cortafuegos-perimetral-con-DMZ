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
ACCEPT     tcp  --  172.22.0.0/16        0.0.0.0/0            tcp dpt:2222 state NEW,ESTABLISHED
ACCEPT     tcp  --  172.23.0.0/16        0.0.0.0/0            tcp dpt:2222 state NEW,ESTABLISHED
ACCEPT     tcp  --  192.168.100.0/24     0.0.0.0/0            tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     tcp  --  192.168.200.0/24     0.0.0.0/0            tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0           
REJECT     icmp --  192.168.100.0/24     0.0.0.0/0            icmptype 8 reject-with icmp-port-unreachable
ACCEPT     icmp --  192.168.200.0/24     0.0.0.0/0            icmptype 8
ACCEPT     icmp --  192.168.100.0/24     0.0.0.0/0            icmptype 0
ACCEPT     icmp --  192.168.200.0/24     0.0.0.0/0            icmptype 0
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 0

Chain FORWARD (policy DROP)
target     prot opt source               destination         
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:22 state ESTABLISHED
ACCEPT     icmp --  192.168.200.0/24     0.0.0.0/0            icmptype 8
ACCEPT     icmp --  0.0.0.0/0            192.168.200.0/24     icmptype 0
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:22 state ESTABLISHED
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 8
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 0
ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            udp dpt:53 state NEW,ESTABLISHED
ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            udp spt:53 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 80 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            multiport sports 80 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 443 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            multiport sports 443 state ESTABLISHED
ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            udp dpt:53 state NEW,ESTABLISHED
ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            udp spt:53 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:80 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:443 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:443 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:25 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:25 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:3306 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:3306 state ESTABLISHED

Chain OUTPUT (policy DROP)
target     prot opt source               destination         
ACCEPT     tcp  --  0.0.0.0/0            172.22.0.0/16        tcp spt:22 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            172.23.0.0/16        tcp spt:22 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            192.168.100.0/24     tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            192.168.200.0/24     tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            172.22.0.0/16        tcp spt:2222 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            172.23.0.0/16        tcp spt:2222 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            192.168.100.0/24     tcp spt:22 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            192.168.200.0/24     tcp spt:22 state ESTABLISHED
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     icmp --  0.0.0.0/0            192.168.100.0/24     icmptype 0
ACCEPT     icmp --  0.0.0.0/0            192.168.100.0/24     state RELATED
ACCEPT     icmp --  0.0.0.0/0            192.168.200.0/24     icmptype 0
ACCEPT     icmp --  0.0.0.0/0            192.168.100.0/24     icmptype 8
ACCEPT     icmp --  0.0.0.0/0            192.168.200.0/24     icmptype 8
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 8
~~~

Se guardan las reglas en un fichero que hay que crear:
~~~
root@router-fw:/home/debian# mkdir /etc/iptables
root@router-fw:/home/debian# touch /etc/iptables/rules.v4
root@router-fw:/home/debian# iptables-save > /etc/iptables/rules.v4
~~~

Se crea el fichero **/etc/systemd/system/restart-iptables.service** con la siguientes líneas:
~~~
[Unit]
Description=restaurar las iptables
After=networking.service

[Service]
ExecStart=/usr/local/bin/restart-iptables.sh

[Install]
WantedBy=multi-user.target
~~~

Se hace un script, **/usr/local/bin/restart-iptables.sh**, con los siguientes permisos:
~~~
root@router-fw:/home/debian# chmod 744 /usr/local/bin/restart-iptables.sh 
~~~

Y con el siguiente contenido:
~~~
#!/bin/bash
echo "Restaurando reglas de iptables..."
iptables-restore < /etc/iptables/rules.v4
echo "Hecho"
~~~

Para que el servicio iptables que se está configurando se inicie al inicio del sistema se utiliza el siguiente comando:
~~~
root@router-fw:/home/debian# systemctl enable restart-iptables.service
Created symlink /etc/systemd/system/multi-user.target.wants/restart-iptables.service → /etc/systemd/system/restart-iptables.service.
~~~

Y se inicia el servicio:
~~~
root@router-fw:/home/debian# systemctl start restart-iptables.service
~~~

> Comprobación:
- Se reinicia:
~~~
root@router-fw:/home/debian# reboot
root@router-fw:/home/debian# Connection to 172.22.201.64 closed by remote host.
Connection to 172.22.201.64 closed.
paloma@coatlicue:~/DISCO2/CICLO II/SEGURIDAD/Cortafuegos-perimetral-con-DMZ$ ssh -A -p 2222 debian@172.22.201.64
Linux router-fw 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Jan  5 19:36:28 2020 from 172.23.0.22
debian@router-fw:~$ sudo iptables -n -L
Chain INPUT (policy DROP)
target     prot opt source               destination         
ACCEPT     tcp  --  172.22.0.0/16        0.0.0.0/0            tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     tcp  --  172.23.0.0/16        0.0.0.0/0            tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     tcp  --  192.168.100.0/24     0.0.0.0/0            tcp spt:22 state ESTABLISHED
ACCEPT     tcp  --  192.168.200.0/24     0.0.0.0/0            tcp spt:22 state ESTABLISHED
ACCEPT     tcp  --  172.22.0.0/16        0.0.0.0/0            tcp dpt:2222 state NEW,ESTABLISHED
ACCEPT     tcp  --  172.23.0.0/16        0.0.0.0/0            tcp dpt:2222 state NEW,ESTABLISHED
ACCEPT     tcp  --  192.168.100.0/24     0.0.0.0/0            tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     tcp  --  192.168.200.0/24     0.0.0.0/0            tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0           
REJECT     icmp --  192.168.100.0/24     0.0.0.0/0            icmptype 8 reject-with icmp-port-unreachable
ACCEPT     icmp --  192.168.200.0/24     0.0.0.0/0            icmptype 8
ACCEPT     icmp --  192.168.100.0/24     0.0.0.0/0            icmptype 0
ACCEPT     icmp --  192.168.200.0/24     0.0.0.0/0            icmptype 0
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 0

Chain FORWARD (policy DROP)
target     prot opt source               destination         
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:22 state ESTABLISHED
ACCEPT     icmp --  192.168.200.0/24     0.0.0.0/0            icmptype 8
ACCEPT     icmp --  0.0.0.0/0            192.168.200.0/24     icmptype 0
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:22 state ESTABLISHED
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 8
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 0
ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            udp dpt:53 state NEW,ESTABLISHED
ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            udp spt:53 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 80 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            multiport sports 80 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 443 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            multiport sports 443 state ESTABLISHED
ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            udp dpt:53 state NEW,ESTABLISHED
ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            udp spt:53 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:80 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:443 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:443 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:25 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:25 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:3306 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:3306 state ESTABLISHED

Chain OUTPUT (policy DROP)
target     prot opt source               destination         
ACCEPT     tcp  --  0.0.0.0/0            172.22.0.0/16        tcp spt:22 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            172.23.0.0/16        tcp spt:22 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            192.168.100.0/24     tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            192.168.200.0/24     tcp dpt:22 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            172.22.0.0/16        tcp spt:2222 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            172.23.0.0/16        tcp spt:2222 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            192.168.100.0/24     tcp spt:22 state ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            192.168.200.0/24     tcp spt:22 state ESTABLISHED
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     icmp --  0.0.0.0/0            192.168.100.0/24     icmptype 0
ACCEPT     icmp --  0.0.0.0/0            192.168.100.0/24     state RELATED
ACCEPT     icmp --  0.0.0.0/0            192.168.200.0/24     icmptype 0
ACCEPT     icmp --  0.0.0.0/0            192.168.100.0/24     icmptype 8
ACCEPT     icmp --  0.0.0.0/0            192.168.200.0/24     icmptype 8
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 8
debian@router-fw:~$ 
~~~


##### 5. El cortafuego debe cumplir al menos estas reglas:
> Se debe indicar pruebas de funcionamiento de todos las reglas.
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
Se configura la conexión con la LAN:
~~~
iptables -A INPUT -s 192.168.100.10/24 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -d 192.168.100.10/24 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
~~~

Se configura la conexión con la DMZ:
~~~
iptables -A INPUT -s 192.168.200.10/24 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -d 192.168.200.10/24 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
~~~

> Comprobación LAN
~~~
debian@router-fw:~$ ssh -A debian@192.168.100.10 
Linux lan 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Dec 13 11:57:36 2019 from 192.168.100.2
debian@lan:~$ 
debian@lan:~$ ssh debian@192.168.100.2
Linux router-fw 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Jan  4 17:35:13 2020 from 172.23.0.22
~~~

> Comprobación DMZ
~~~
debian@router-fw:~$ ssh -A debian@192.168.200.10 
Linux dmz 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Dec 13 11:11:25 2019 from 192.168.200.2
debian@dmz:~$ ssh debian@192.168.200.2
The authenticity of host '192.168.200.2 (192.168.200.2)' can't be established.
ECDSA key fingerprint is SHA256:rUm8t8gIDopOZ7Zhm779D24LlxFnQlm774wN1Gq0aWQ.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.200.2' (ECDSA) to the list of known hosts.
Linux router-fw 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sat Jan  4 17:35:52 2020 from 192.168.100.10
debian@router-fw:~$ 
~~~


- La máquina router-fw debe tener permitido el tráfico para la interfaz loopback.

~~~
iptables -A INPUT -i lo -p icmp -j ACCEPT
iptables -A OUTPUT -o lo -p icmp -j ACCEPT
~~~

> Comprobación
~~~
debian@router-fw:~$ ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.114 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.067 ms
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
debian@lan:~$ ping 192.168.100.2
PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.
From 192.168.100.2 icmp_seq=1 Destination Port Unreachable
From 192.168.100.2 icmp_seq=2 Destination Port Unreachable
~~~

> Comprobación DMZ
~~~
debian@dmz:~$ ping 192.168.200.2 
PING 192.168.200.2 (192.168.200.2) 56(84) bytes of data.
64 bytes from 192.168.200.2: icmp_seq=1 ttl=64 time=0.424 ms
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
debian@router-fw:~$ ping 192.168.100.10
PING 192.168.100.10 (192.168.100.10) 56(84) bytes of data.
64 bytes from 192.168.100.10: icmp_seq=1 ttl=64 time=1.71 ms
64 bytes from 192.168.100.10: icmp_seq=2 ttl=64 time=0.978 ms
^C
--- 192.168.100.10 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.978/1.346/1.714/0.368 ms

~~~

> Comprobación DMZ
~~~
debian@dmz:~$ ping 192.168.200.10
PING 192.168.200.10 (192.168.200.10) 56(84) bytes of data.
64 bytes from 192.168.200.10: icmp_seq=1 ttl=64 time=0.036 ms
64 bytes from 192.168.200.10: icmp_seq=2 ttl=64 time=0.067 ms
~~~

> Comprobación con el exterior
~~~
debian@router-fw:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=51 time=42.8 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=51 time=43.1 ms
~~~


- Desde la máquina DMZ se puede hacer ping y conexión ssh a la máquina LAN.

Configuración para hacer ssh desde la DMZ a la LAN:
~~~
iptables -A FORWARD -i eth2 -o eth1 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth1 -o eth2 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
~~~

Configuración para hacer ping desde la DMZ a la LAN:
~~~
iptables -A FORWARD -i eth2 -o eth1 -s 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A FORWARD -i eth1 -o eth2 -d 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
~~~


> Comprobación ping:
~~~
debian@dmz:~$ ping 192.168.100.10
PING 192.168.100.10 (192.168.100.10) 56(84) bytes of data.
64 bytes from 192.168.100.10: icmp_seq=1 ttl=63 time=2.40 ms
64 bytes from 192.168.100.10: icmp_seq=2 ttl=63 time=2.46 ms
~~~

> Comprobación ssh:
~~~
debian@dmz:~$ ssh 192.168.100.10
The authenticity of host '192.168.100.10 (192.168.100.10)' can't be established.
ECDSA key fingerprint is SHA256:t+oOUYzPINNAMvlxtwjZz8ypWKYgREQwAvEX6y4M3EU.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.100.10' (ECDSA) to the list of known hosts.
Linux lan 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Jan  5 18:12:46 2020 from 192.168.100.2
debian@lan:~$ 
~~~

- Desde la máquina LAN no se puede hacer ping, pero si se puede conectar por ssh a la máquina DMZ.
En este caso solo se debe configurar el acceso por ssh, ya que el ping está por defecto en DROP. Las reglas son:
~~~
iptables -A FORWARD -i eth1 -o eth2 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth1 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
~~~

> Comprobación:
~~~
debian@lan:~$ ping 192.168.100.2
PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.
From 192.168.100.2 icmp_seq=1 Destination Port Unreachable
From 192.168.100.2 icmp_seq=2 Destination Port Unreachable
^C
--- 192.168.100.2 ping statistics ---
2 packets transmitted, 0 received, +2 errors, 100% packet loss, time 2ms

debian@lan:~$ ssh 192.168.200.10
The authenticity of host '192.168.200.10 (192.168.200.10)' can't be established.
ECDSA key fingerprint is SHA256:mv6eLKhag1jO/uDrIKEidKeFRnOtRgFrq1tf5bum0c8.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.200.10' (ECDSA) to the list of known hosts.
Linux dmz 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Jan  5 18:12:06 2020 from 192.168.200.2
debian@dmz:~$ 
~~~


- Configura la máquina router-fw para que las máquinas LAN y DMZ puedan acceder al exterior.
~~~
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.200.0/24 -o eth0 -j MASQUERADE
~~~

- La máquina LAN se le permite hacer ping al exterior.
Configuración para que la LAN tengo permitido el ping al exterior:
~~~
iptables -A FORWARD -i eth1 -o eth0 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
~~~

> Comprobación:
~~~
debian@lan:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=50 time=43.9 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=50 time=43.5 ms
~~~

- La máquina LAN puede navegar.
Configuración para activar el DNS:
~~~
iptables -A FORWARD -i eth1 -o eth0 -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT
~~~

Configuración para activar HTTP:
~~~
iptables -A FORWARD -i eth1 -o eth0 -p tcp -m multiport --dports 80 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -p tcp -m multiport --sports 80 -m state --state ESTABLISHED -j ACCEPT
~~~

Configuración para activar HTTPS:
~~~
iptables -A FORWARD -i eth1 -o eth0 -p tcp -m multiport --dports 443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -p tcp -m multiport --sports 443 -m state --state ESTABLISHED -j ACCEPT
~~~

> Comprobación descargando un paquete:
~~~
debian@lan:~$ sudo apt install alien
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  autoconf automake autopoint autotools-dev binutils binutils-common
  binutils-x86-64-linux-gnu build-essential cpp cpp-8 debhelper
  debugedit dh-autoreconf dh-strip-nondeterminism dirmngr dpkg-dev dwz
  fakeroot g++ g++-8 gcc gcc-8 gettext gnupg gnupg-l10n gnupg-utils gpg
  gpg-agent gpg-wks-client gpg-wks-...
~~~

- La máquina DMZ puede navegar. Instala un servidor web, un servidor ftp y un servidor de correos.
Se configura el DNS:
~~~
iptables -A FORWARD -i eth2 -o eth0 -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth2 -p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT
~~~

Se configura HTTP:
~~~
iptables -A FORWARD -i eth2 -o eth0 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth2 -p tcp --sport 80 -m state --state ESTABLISHED -j ACCEPT
~~~

Y se configura el HTTPS:
~~~
iptables -A FORWARD -i eth2 -o eth0 -p tcp --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth2 -p tcp --sport 443 -m state --state ESTABLISHED -j ACCEPT
~~~

> Comprobación:
~~~
debian@dmz:~$ sudo apt install apache2 postfix proftpd
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Note, selecting 'proftpd-basic' instead of 'proftpd'
The following additional packages will be installed:
  apache2-bin apache2-data apache2-utils libapr1 libaprutil1
  libaprutil1-dbd-sqlite3 libaprutil1-ldap libbrotli1 libcurl4
  libgdbm-compat4 libgdbm6 libhiredis0.14 libjansson4 libldap-2.4-2
  libldap-common liblua5.2-0 libmemcached11 libmemcachedutil2
  libnghttp2-14 libperl5.28 librtmp1 libsasl2-2 libsasl2-modules
  libsasl2-modules-db libssh2-1 perl perl-modules-5.28 proftpd-doc
  ssl-cert
Suggested packages:
  apache2-doc apache2-suexec-pristine | apache2-suexec-custom
  www-browser libsasl2-modules-gssapi-mit
  | libsasl2-modules-gssapi-heimdal libsasl2-modules-ldap
  libsasl2-modules-otp libsasl2-modules-sql per...
~~~

- Configura la máquina router-fw para que los servicios web y ftp sean accesibles desde el exterior.


- El servidor web y el servidor ftp deben ser accesible desde la LAN y desde el exterior.

- El servidor de correos sólo debe ser accesible desde la LAN.
En la instalación del paquete Postfix se selecciona la opción **Internet Site**:
~~~
 ┌──────────────────────┤ Postfix Configuration ├──────────────────────┐
 │ Please select the mail server configuration type that best meets    │ 
 │ your needs.                                                         │ 
 │                                                                     │ 
 │  No configuration:                                                  │ 
 │   Should be chosen to leave the current configuration unchanged.    │ 
 │  Internet site:                                                     │ 
 │   Mail is sent and received directly using SMTP.                    │ 
 │  Internet with smarthost:                                           │ 
 │   Mail is received directly using SMTP or by running a utility      │ 
 │ such                                                                │ 
 │   as fetchmail. Outgoing mail is sent using a smarthost.            │ 
 │  Satellite system:                                                  │ 
 │   All mail is sent to another machine, called a 'smarthost', for    │ 
 │ delivery.                                                           │ 
 │  Local only:                                                        │ 
 │   The only delivered mail is the mail for local users. There is no  │ 
 │ network.                                                            │ 
 │                                                                     │ 
 │ General type of mail configuration:                                 │ 
 │                                                                     │ 
 │                      No configuration                               │ 
 │                      Internet Site                                  │ 
 │                      Internet with smarthost                        │ 
 │                      Satellite system                               │ 
 │                      Local only                                     │ 
 │                                                                     │ 
 │                                                                     │ 
 │                  <Ok>                      <Cancel>                 │ 
 │                                                                     │ 
 └─────────────────────────────────────────────────────────────────────┘ 
~~~

A continuación, se configura el fichero **/etc/postfix/main.cf** para permitir las redes correspondientes:
~~~
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 192.168.100.0/24
~~~

Reglas:
~~~
iptables -A FORWARD -i eth1 -o eth2 -p tcp --dport 25 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth1 -p tcp --sport 25 -m state --state ESTABLISHED -j ACCEPT
~~~

> Comprobación:
~~~
debian@lan:~$ telnet 192.168.200.10 25
Trying 192.168.200.10...
Connected to 192.168.200.10.
Escape character is '^]'.
220 dmz.novalocal ESMTP Postfix (Debian/GNU)
~~~


- En la máquina LAN instala un servidor mysql. A este servidor sólo se puede acceder desde la DMZ.
Instalación del servidor mariaDB en la máquina LAN:
~~~
debian@lan:~$ sudo apt install mariadb-server
~~~

Instalación del cliente de mariaDB en la máquina DMZ:
~~~
debian@dmz:~$ sudo apt install mariadb-client
~~~

Se crea una base de datos, un usuario y se otorgan los permisos adecuados:
~~~
debian@lan:~$ sudo mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 47
Server version: 10.3.18-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> create database dmz;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> create user user_dmz@"192.168.200.10" identified by "user_dmz";
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> grant all privileges on dmz.* to user_dmz identified by "user_dmz";
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> flush privileges;
Query OK, 0 rows affected (0.001 sec)

~~~

Se configura el fichero **/etc/mysql/mariadb.conf.d/50-server.cnf**:
~~~
bind-address            = 0.0.0.0
~~~

Y se reinicia el servicio:
~~~
debian@lan:~$ sudo systemctl restart mariadb.service 
~~~

Las reglas son:
~~~
iptables -A FORWARD -i eth2 -o eth1 -p tcp --dport 3306 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth1 -o eth2 -p tcp --sport 3306 -m state --state ESTABLISHED -j ACCEPT
~~~

> Comprobación:
~~~
debian@dmz:~$ sudo mysql -u user_dmz -p -h 192.168.100.10
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 36
Server version: 10.3.18-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> use dmz
Database changed
MariaDB [dmz]> 
~~~


##### 6. Si crees que necesitas más reglas de las que nos han indicado, describe porque pueden ser necesarias.

##### 7. MEJORA: Utiliza nuevas cadenas para clasificar el tráfico.

##### 8. MEJORA: Consruye el cortafuego utilizando nftables.

Para construir reglas nftables se puede utiliza el comando **iptables-translate**. Este comando devuelve la regla introducida traducida:
~~~
sudo iptables-translate <regla>
~~~

De esta forma, por pantalla aparecerá la regla traducida:
~~~
root@router-fw:/home/debian# iptables-translate -A FORWARD -i eth2 -o eth1 -p tcp --dport 3306 -m state --state NEW,ESTABLISHED -j ACCEPT
nft add rule ip filter FORWARD iifname "eth2" oifname "eth1" tcp dport 3306 ct state new,established  counter accept
~~~

Pero el comando anterior tiene dos fallos que hay que tener en cuenta a la hora de realizar el script:
- ip en nftables es inet.
- FORWARD en nftables es forward en minúscula.

Para solventar estos fallos se va a realizar el script, **iptables_translates.sh**, que se describe a continuación que crea un fichero con la reglas:
~~~
#!/bin/sh

# Borrar ficheros creados
sudo rm /home/debian/fichnft1 2> /dev/null
sudo rm /home/debian/fichnft2 2> /dev/null
sudo rm /home/debian/fichnft3 2> /dev/null

# Guardar reglas iptables en fichnft1
sudo iptables -S > /home/debian/fichnft1
sudo iptables -t nat -S >> /home/debian/fichnft1

# Recorrer fichnft1 y añadir comando
while read i
do
  sudo iptables-translate $i >> /home/debian/fichnft2
done < fichnft1

# Modificar ip por inet
sed 's/ ip /inet/g' /home/debian/fichnft2 > /home/debian/fichnft3
# Convertir todo en minúscula
tr A-Z a-z < /home/debian/fichnft3 > /home/debian/nftables.txt

# Borrar ficheros creados
sudo rm /home/debian/fichnft1 2> /dev/null
sudo rm /home/debian/fichnft2 2> /dev/null
sudo rm /home/debian/fichnft3 2> /dev/null
~~~

Se ejecuta:
~~~
debian@router-fw:~$ bash iptables_translates.sh 
~~~

El resultado es un fichero con la reglas implementadas:
~~~
debian@router-fw:~$ cat nftables.txt 
nft nft nft nft add ruleinetfilter inputinetsaddr 172.22.0.0/16 tcp dport 22 ct state new,established  counter accept
nft add ruleinetfilter inputinetsaddr 172.23.0.0/16 tcp dport 22 ct state new,established  counter accept
nft add ruleinetfilter input iifname "eth1"inetsaddr 192.168.100.0/24 tcp sport 22 ct state established  counter accept
nft add ruleinetfilter input iifname "eth2"inetsaddr 192.168.200.0/24 tcp sport 22 ct state established  counter accept
nft add ruleinetfilter inputinetsaddr 172.22.0.0/16 tcp dport 2222 ct state new,established  counter accept
nft add ruleinetfilter inputinetsaddr 172.23.0.0/16 tcp dport 2222 ct state new,established  counter accept
nft add ruleinetfilter inputinetsaddr 192.168.100.0/24 tcp dport 22 ct state new,established  counter accept
nft add ruleinetfilter inputinetsaddr 192.168.200.0/24 tcp dport 22 ct state new,established  counter accept
nft add ruleinetfilter input iifname "lo"inetprotocol icmp counter accept
nft add ruleinetfilter input iifname "eth1"inetsaddr 192.168.100.0/24 icmp type echo-request counter reject
nft add ruleinetfilter input iifname "eth2"inetsaddr 192.168.200.0/24 icmp type echo-request counter accept
nft add ruleinetfilter input iifname "eth1"inetsaddr 192.168.100.0/24 icmp type echo-reply counter accept
nft add ruleinetfilter input iifname "eth2"inetsaddr 192.168.200.0/24 icmp type echo-reply counter accept
nft add ruleinetfilter input iifname "eth0" icmp type echo-reply counter accept
nft add ruleinetfilter forward iifname "eth2" oifname "eth1" tcp dport 22 ct state new,established  counter accept
nft add ruleinetfilter forward iifname "eth1" oifname "eth2" tcp sport 22 ct state established  counter accept
nft add ruleinetfilter forward iifname "eth2" oifname "eth1"inetsaddr 192.168.200.0/24 icmp type echo-request counter accept
nft add ruleinetfilter forward iifname "eth1" oifname "eth2"inetdaddr 192.168.200.0/24 icmp type echo-reply counter accept
nft add ruleinetfilter forward iifname "eth1" oifname "eth2" tcp dport 22 ct state new,established  counter accept
nft add ruleinetfilter forward iifname "eth2" oifname "eth1" tcp sport 22 ct state established  counter accept
nft add ruleinetfilter forward iifname "eth1" oifname "eth0" icmp type echo-request counter accept
nft add ruleinetfilter forward iifname "eth0" oifname "eth1" icmp type echo-reply counter accept
nft add ruleinetfilter forward iifname "eth1" oifname "eth0" udp dport 53 ct state new,established  counter accept
nft add ruleinetfilter forward iifname "eth0" oifname "eth1" udp sport 53 ct state established  counter accept
nft add ruleinetfilter forward iifname "eth1" oifname "eth0"inetprotocol tcp tcp dport 80 ct state new,established  counter accept
nft add ruleinetfilter forward iifname "eth0" oifname "eth1"inetprotocol tcp tcp sport 80 ct state established  counter accept
nft add ruleinetfilter forward iifname "eth1" oifname "eth0"inetprotocol tcp tcp dport 443 ct state new,established  counter accept
nft add ruleinetfilter forward iifname "eth0" oifname "eth1"inetprotocol tcp tcp sport 443 ct state established  counter accept
nft add ruleinetfilter forward iifname "eth2" oifname "eth0" udp dport 53 ct state new,established  counter accept
nft add ruleinetfilter forward iifname "eth0" oifname "eth2" udp sport 53 ct state established  counter accept
nft add ruleinetfilter forward iifname "eth2" oifname "eth0" tcp dport 80 ct state new,established  counter accept
nft add ruleinetfilter forward iifname "eth0" oifname "eth2" tcp sport 80 ct state established  counter accept
nft add ruleinetfilter forward iifname "eth2" oifname "eth0" tcp dport 443 ct state new,established  counter accept
nft add ruleinetfilter forward iifname "eth0" oifname "eth2" tcp sport 443 ct state established  counter accept
nft add ruleinetfilter forward iifname "eth1" oifname "eth2" tcp dport 25 ct state new,established  counter accept
nft add ruleinetfilter forward iifname "eth2" oifname "eth1" tcp sport 25 ct state established  counter accept
nft add ruleinetfilter forward iifname "eth2" oifname "eth1" tcp dport 3306 ct state new,established  counter accept
nft add ruleinetfilter forward iifname "eth1" oifname "eth2" tcp sport 3306 ct state established  counter accept
nft add ruleinetfilter outputinetdaddr 172.22.0.0/16 tcp sport 22 ct state established  counter accept
nft add ruleinetfilter outputinetdaddr 172.23.0.0/16 tcp sport 22 ct state established  counter accept
nft add ruleinetfilter output oifname "eth1"inetdaddr 192.168.100.0/24 tcp dport 22 ct state new,established  counter accept
nft add ruleinetfilter output oifname "eth2"inetdaddr 192.168.200.0/24 tcp dport 22 ct state new,established  counter accept
nft add ruleinetfilter outputinetdaddr 172.22.0.0/16 tcp sport 2222 ct state established  counter accept
nft add ruleinetfilter outputinetdaddr 172.23.0.0/16 tcp sport 2222 ct state established  counter accept
nft add ruleinetfilter outputinetdaddr 192.168.100.0/24 tcp sport 22 ct state established  counter accept
nft add ruleinetfilter outputinetdaddr 192.168.200.0/24 tcp sport 22 ct state established  counter accept
nft add ruleinetfilter output oifname "lo"inetprotocol icmp counter accept
nft add ruleinetfilter output oifname "eth1"inetdaddr 192.168.100.0/24 icmp type echo-reply counter accept
nft add ruleinetfilter output oifname "eth1"inetprotocol icmpinetdaddr 192.168.100.0/24 ct state related  counter accept
nft add ruleinetfilter output oifname "eth2"inetdaddr 192.168.200.0/24 icmp type echo-reply counter accept
nft add ruleinetfilter output oifname "eth1"inetdaddr 192.168.100.0/24 icmp type echo-request counter accept
nft add ruleinetfilter output oifname "eth2"inetdaddr 192.168.200.0/24 icmp type echo-request counter accept
nft add ruleinetfilter output oifname "eth0" icmp type echo-request counter accept
nft nft nft nft nft add ruleinetfilter preroutinginetsaddr 172.23.0.0/16 tcp dport 22 counter dnat to 127.0.0.1
nft add ruleinetfilter preroutinginetsaddr 172.22.0.0/16 tcp dport 22 counter dnat to 127.0.0.1
nft add ruleinetfilter preroutinginetsaddr 172.23.0.0/16 tcp dport 2222 counter redirect to :22
nft add ruleinetfilter preroutinginetsaddr 172.22.0.0/16 tcp dport 2222 counter redirect to :22
nft add ruleinetfilter postrouting oifname "eth0"inetsaddr 192.168.100.0/24 counter masquerade 
nft add ruleinetfilter postrouting oifname "eth0"inetsaddr 192.168.200.0/24 counter masquerade 
~~~
