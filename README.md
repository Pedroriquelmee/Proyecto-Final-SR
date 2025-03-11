# Práctica SR

Creado por: Pedro José Riquelme Guerrero  
Fecha: 05/01/2024

## ÍNDICE
1. [Configuración del router](#configuración-del-router)
2. [Configuración DHCP](#configuración-dhcp)
3. [DHCP Relay](#dhcp-relay)
4. [Failover](#failover)
5. [DNS](#dns)
6. [DNS2](#dns2)
7. [FTP](#ftp)
8. [Conclusión](#conclusión)

---

## Configuración del router

### ROUTER1
- **Adaptadores**
  - 1. Adaptador puente: `192.168.35.95`
  - 2. Red interna: `172.16.95.1`

Para configurar el router empezaremos cambiando el `hosts` y el `hostname`:

```bash
sudo nano /etc/hosts
sudo nano /etc/hostname
```

Una vez hecho esto reiniciamos el Debian y configuramos la IP del router:

```bash
# Puente
auto enp0s3
iface enp0s3 inet static
  address 192.168.35.95
  netmask 255.255.255.0
  network 192.168.35.0
  gateway 192.168.35.1

# redAsir
auto enp0s8
iface enp0s8 inet static
  address 172.16.95.254
  netmask 255.255.255.0
  network 172.16.95.0
```

Una vez configurada la IP, configuraremos el archivo `/etc/init.d/reenvioiptables.sh`:

```bash
#!/bin/sh
# Script de inicio
### BEGIN INIT INFO
# Provides: reenvioiptables.sh
# Required-Start: $all
# Required-Stop: $all
# Default-Start: 2 3 4 5
# Default-Stop: 0 1 6
# Short-Description: Firewall para red interna.
# Description: Daemon para hacer funcionar la maquina como firewall usando iptables.
### END INIT INFO

# BORRA LAS REGLAS QUE HAYA
iptables -F
iptables -X
iptables -Z
iptables -t nat -F

# POLITICA POR DEFECTO
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -t nat -P PREROUTING ACCEPT
iptables -t nat -P POSTROUTING ACCEPT

# ACEPTAMOS ACCESO LOCALHOST
iptables -A INPUT -i lo -j ACCEPT

# PERMITIMOS ACCESO DESDE LA RED LOCAL
iptables -A INPUT -s 172.16.95.0/24 -j ACCEPT

# ENMASCARAMOS RED LOCAL
iptables -t nat -A POSTROUTING -s 172.16.95.0/24 -o enp0s3 -j MASQUERADE

# ACTIVAMOS BIT FORWARD
echo 1 > /proc/sys/net/ipv4/ip_forward

# PERMITIMOS ACCESO AL PUERTO 22 (SSH)
iptables -A INPUT -s 0.0.0.0/0 -i enp0s3 -p tcp --dport 22 -j ACCEPT

# REDIRECCIONAMOS PUERTOS
iptables -t nat -A PREROUTING -i enp0s3 -p tcp --dport 2201 -j DNAT --to 172.16.95.2:22
iptables -t nat -A PREROUTING -i enp0s3 -p tcp --dport 3389 -j DNAT --to 172.16.95.3:3389
iptables -t nat -A PREROUTING -i ens33 -p udp --dport 3389 -j DNAT --to 172.16.95.3:3389

# CERRAMOS LOS PUERTOS BIEN CONOCIDOS
iptables -A INPUT -s 0.0.0.0/0 -i enp0s3 -p tcp --dport 1:1024 -j DROP
iptables -A INPUT -s 0.0.0.0/0 -i enp0s3 -p udp --dport 1:1024 -j DROP
```

Una vez configurado el `reenvioiptables`, editamos el archivo `/etc/systemd/system/reenvioiptables.service`:

```ini
[Unit]
Description=Firewall para red interna
After=network.target

[Service]
Type=oneshot
ExecStart=/etc/init.d/reenvioiptables.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

A continuación, nos iremos al repositorio `/etc/init.d` y ejecutaremos el siguiente comando:

```bash
sudo update-rc.d reenvioiptables.sh defaults
```

Una vez hecho, instalamos `iptables`:

```bash
sudo apt install iptables
```

Cuando ya hemos instalado `iptables`, le daremos permisos al archivo `reenvioiptables.sh` y lo ejecutaremos:

```bash
sudo chmod +x reenvioiptables.sh
sudo ./reenvioiptables.sh
sudo iptables -L
```

Una vez hecho todo, nos iremos al cliente y le pondremos la IP y la puerta de enlace correspondiente.

Para comprobar si todo funciona, deberíamos tener acceso a Internet. Hacemos cualquier búsqueda y debería funcionar. En caso de que no funcione, podemos probar a ejecutar lo siguiente:

```bash
sudo ip route add default via <IP DEL ROUTER>
sudo ip route add default via 172.16.95.254
```

---

## Configuración DHCP

### DHCP
- **Adaptadores de red**
  - Red Interna: `172.16.95.12`

Para la configuración del DHCP necesitaremos otra máquina Debian, la cual le configuraremos la IP.

Una vez configurada la IP y comprobado que tenemos Internet, instalamos el `isc-dhcp-server`:

```bash
sudo apt-get install isc-dhcp-server
```

Cambiaremos la configuración del fichero `/etc/default/isc-dhcp-server` y donde pone `INTERFACESv4` se quedará tal que:

```bash
INTERFACESv4="enss33"
INTERFACESv6=""
```

**IMPORTANTE**: Comando para verificar errores de sintaxis en el archivo DHCP:

```bash
dhcpd -t
```

Una vez configurado, modificamos el fichero `/etc/dhcp/dhcp.conf`:

```bash
subnet 172.16.95.0 netmask 255.255.255.0 {
  range 172.16.95.20 172.16.95.30;
  option subnet-mask 255.255.255.0;
  option routers 172.16.95.254;
  option domain-name-servers 8.8.8.8;
  default-lease-time 86400;
  max-lease-time 691200;
  min-lease-time 3600;
}
```

Comprobamos que todo funcione correctamente con:

```bash
sudo systemctl restart isc-dhcp-server.service
sudo systemctl start isc-dhcp-server.service
sudo systemctl status isc-dhcp-server.service
```

Una vez viendo que todo funciona correctamente, volveremos a modificar el archivo `/etc/dhcp/dhcpd.conf` y le meteremos debajo de lo que ya escribimos lo siguiente:

```bash
host pedroj {
  hardware ethernet 00:0c:29:76:b3:40;
  fixed-address 172.16.95.1;
}

host cliente-pedro {
  hardware ethernet 00:0C:29:F7:78:4F;
  fixed-address 172.16.95.8;
  option routers 172.16.95.254;
}
```

Ahora nos iremos tanto al router como al cliente y escribiremos:

```bash
auto ens37
iface ens37 inet dhcp
```

Y comprobaremos si nos da la IP que le hemos indicado anteriormente.

---

## DHCP Relay

### DHCP Relay
- **Router**
  - **Adaptadores**
    - Red Interna: `192.168.35.65`
    - Adaptador Puente: `172.16.65.254`
- **DHCP Relay**
  - **Adaptadores**
    - Red Interna: `172.16.65.13`
- **Cliente**
  - **Adaptadores**
    - Red Interna: `172.16.65.8`

Crearemos un router nuevo para hacer el relay. Esta vez la configuración del router va a ser la misma pero con -30 a mi IP, así que se quedaría como `192.168.35.65` y la `172.16.65.1`.

En el `iptables` tendremos que modificar las IP y poner las que tenemos actualmente.

Una vez configurado, ejecutaremos todo como hicimos con el primer router. Para el DHCP Relay le cambiaremos el hostname y le pondremos `dhcp65`.

Le pondremos la siguiente configuración al DHCP relay. Una vez configurada la IP del DHCP, instalaremos el `dhcp-relay-server`:

```bash
sudo apt install isc-dhcp-relay
```

Una vez instalado, nos empezará a pedir cosas y le pondremos las siguientes direcciones IP. Una vez configurado eso, nos iremos al `iptables` del router primero que configuramos y añadiremos las siguientes líneas:

```bash
iptables -t nat -A PREROUTING -i enp0s3 -p udp --dport 67 -j DNAT --to 172.16.95.12:67
ip route add 172.16.65.0/24 via 192.168.35.65 dev enp0s3
```

Ahora para el router 2 ponemos en el `iptables`:

```bash
# IP del DHCP relay
iptables -t nat -A PREROUTING -i enp0s3 -p tcp --dport 67 -j DNAT --to 172.16.65.13:67
ip route add 172.16.95.0/24 via 192.168.35.95 dev enp0s3
```

Comprobamos que está todo bien con:

```bash
route show
```

Nos iremos a `/etc/dhcp/dhcpd.conf` y le pondremos lo siguiente. Una vez configurado todo, si nos vamos al cliente del relay y todo está bien, nos deberá de dar conexión a Internet.

---

## Failover

### DHCP Failover
- **Adaptadores de red**
  - Red Interna: `172.16.95.13`
- **Cliente**
  - **Adaptadores de red**
    - Red Interna: `172.16.95.25`

Para configurar el Failover, nos iremos al primer DHCP que configuramos y modificaremos su directorio, comentando Relay y añadiendo lo siguiente:

```bash
# Comentar Relay
```

Comprobamos si funciona con un status. Ahora nos iremos a la clonación que hicimos del DHCP 1 para poder añadirle lo siguiente:

```bash
# Configuración del Failover
```

Una vez configuramos todo, abrimos el cliente del Failover y vemos si nos da la IP que tenemos dentro del rango establecido, y en mi caso, sí me la da.

---

## DNS

### DNS1
- **Adaptadores**
  - `172.16.95.2`

### DNS2
- **Adaptadores**
  - `172.16.95.3`

Empezaremos añadiendo en el fichero `dhcpd.conf` los DNS que vamos a usar con las IP que queremos usar.

Cambiaremos la configuración de los adaptadores de red para que cojan el DHCP. Si hacemos un `ip a`, nos dará la IP.

Cambiamos el hostname de la máquina para aclarar cuál es cuál. Comprobamos que nos sale el nombre que hemos puesto. Hacemos:

```bash
sudo apt install bind9
sudo apt install bind9-doc dnsutils resolvconf ufw python-ply-doc
```

Configuramos el archivo de ACL y opciones:

```bash
acl "trusted" {
  172.16.95.1; # firewall
  172.16.95.2; # DNS1
  172.16.95.3; # DNS2
  172.16.95.4; # CLIENTE
};

options {
  directory "/var/cache/bind"; # Directorio de caché
  recursion yes;   # Habilita la recursividad
  allow-recursion { trusted; }; # Permite que solo los clientes definidos en la ACL "trusted" realicen consultas
  listen-on { 172.16.95.2; }; # Escucha en las IPs de DNS1
  allow-transfer { none; };   # Desactiva las transferencias de zona por defecto
  forwarders {
    8.8.8.8;   # Servidor DNS de Google
    8.8.4.4;   # Otro servidor DNS de Google
  };
  dnssec-validation auto;
};
```

Creamos la zona:

```bash
sudo mkdir /etc/bind/zones
```

Y el archivo de zona:

```bash
$TTL   604800
alumno95.com.   IN   SOA   ns1.alumno95.com. admin.alumno95.com. (
  6   ; Serial
  604800   ; Refresh
  86400   ; Retry
  2419200   ; Expire
  604800 )   ; Negative Cache TTL

@   IN   NS   ns1.alumno95.com.
@   IN   NS   ns2.alumno95.com.

; name servers - A records
@   IN   A   172.16.95.2
ns1   IN   A   172.16.95.2
ns2   IN   A   172.16.95.3

; 172.16.95.0/24 - A records
interna1.alumno95.com.   IN   A   172.16.95.4
firewall.alumno95.com.   IN   A   172.16.95.1
```

Si hacemos un `nslookup` a `alumno95.com` o a `pedroj.com`, nos devolverá la información:

```bash
$TTL   604800
@   IN   SOA   ns1.alumno95.com. admin.alumno95.com. (
  9   ; Serial
  604800   ; Refresh
  86400   ; Retry
  2419200   ; Expire
  604800 )   ; Negative Cache TTL

; name servers - NS records
IN   NS   ns1.alumno95.com.
IN   NS   ns2.alumno95.com.

; 172.16.80.0/24 - PTR records
1   IN   PTR   firewall.alumno95.com.
2   IN   PTR   ns1.alumno95.com.
3   IN   PTR   ns2.alumno95.com.
4   IN   PTR   cliente1.alumno95.com.
```

Ejecutamos un check para confirmar que la configuración está bien:

```bash
sudo named-checkconf
sudo named-checkzone alumno95.com /etc/bind/zones/db.alumno95.com
sudo named-checkzone 95.16.172.in-addr.arpa /etc/bind/zones/db.172.16.95
```

Reseteamos el servicio y agregamos las reglas:

```bash
sudo systemctl restart bind9
sudo ufw allow Bind9
```

### DNS2

Ahora como en el anterior, le cambiaremos el hostname y haremos cambios en algunos archivos.

Configuramos el archivo de ACL y opciones:

```bash
acl "trusted" {
  172.16.95.1; # firewall
  172.16.95.2; # DNS1
  172.16.95.3; # DNS2
  172.16.95.4; # CLIENTE
};

options {
  directory "/var/cache/bind";
  recursion yes;
  allow-recursion { trusted; };
  listen-on { 172.16.95.3; };
  allow-transfer { none; };
};
```

Creamos la zona:

```bash
zone "alumno95.com" {
  type slave;
  file "db.alumno95.com";
  masters {172.16.95.2;};
};

zone "95.16.172.in-addr.arpa" {
  type master;
  file "/etc/bind/zones/db.172.16.95";
  allow-transfer { 172.16.95.3; };
};
```

Volvemos a hacer un check conf para confirmar que ha funcionado correctamente todo. Aquí añadiremos el apartado de zone:

```bash
zone "pedroj.com" {
  type master;
  file "/etc/bind/zones/db.pedroj.com";
  allow-transfer { 172.16.95.2; };
};
```

Creamos el archivo de zona:

```bash
$TTL   604800
pedroj.com.   IN   SOA   ns1.pedroj.com. admin.pedroj.com. (
  6   ; Serial
  604800   ; Refresh
  86400   ; Retry
  2419200   ; Expire
  604800 )   ; Negative Cache TTL

@   IN   NS   ns1.pedroj.com.
@   IN   NS   ns2.pedroj.com.

; name servers - A records
@   IN   A   172.16.95.3
debian1   IN   A   172.16.95.2
debian2   IN   A   172.16.95.3

; 172.16.95.0/24 - A records
interna1.pedroj.com.   IN   A   172.16.95.4
firewall.pedroj.com.   IN   A   172.16.95.1
```

Nos iremos al DNS1 y le agregaremos el zone que pusimos en el DNS2, pero haciéndolo al contrario:

```bash
zone "pedroj.com" {
  type slave;
  file "db.pedroj.com";
  allow-transfer { 172.16.95.3; };
};
```

Agregaremos lo siguiente al fichero del DHCP:

```bash
option domain-name "alumno95.com";
option domain-search "alumno95.com", "pedroj.com";
option domain-name-servers 172.16.95.2, 172.16.95.3;
```

### CLIENTE 1

```bash
host Cliente1 {
  hardware ethernet 08:00:27:01:b3:cb;
  fixed-address 172.16.95.4;
}
```

Si funciona correctamente, si nos vamos al cliente, nos hará ping:

```bash
ping ns1
ping ns2
sudo cat /etc/resolv.conf
```

---

## FTP

Agregaremos los valores para darle IP a través del DHCP:

```bash
# FTP
host FTP {
  hardware ethernet 08:00:27:06:22:11;
  fixed-address 172.16.95.5;
}
```

Cambiamos el hostname. En el DNS1, nos iremos al `/etc/bind/named.conf.options` y añadiremos:

```bash
172.16.95.5; # FTP
```

También añadiremos en la siguiente ruta:

```bash
sudo nano /etc/bind/zones/db.alumno95.com
ftp.alumno95.com   IN   A   172.16.95.5
```

En el directorio de la zona inversa, tendremos que añadir al final del todo la siguiente línea:

```bash
5   IN   PTR   ftp.alumno95.com   ; 172.16.95.5
```

En el DNS2, igual meteremos el `172.16.95.5; # FTP` en el `conf.options`.

En el DNS2, meteremos en la ruta `/etc/bind/zones/db.pedroj.com`:

```bash
ftpdeb.pedroj.com.   IN   A   172.16.95.5
```

Si nos vamos, hacemos un `cat /etc/hosts`, nos saldrá todo lo que hemos puesto. Instalaremos el servicio con:

```bash
sudo apt install vsftpd
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.orig
```

Haremos:

```bash
sudo ufw allow 20,21,990/tcp
```

Para añadir las reglas:

```bash
sudo ufw allow 40000,50000/tcp
```

Habilitamos el servicio y si hacemos un status, veremos que las rutas que hemos puesto saldrán. Añadiremos un usuario con una contraseña para poder configurarlo y así luego poder usarlo con el FileZilla.

```bash
sudo chown nobody:nogroup /home/krieger/ftp
```

Esto se hace para que el directorio sea accesible solo por un usuario con privilegios limitados, lo que puede ayudar a mejorar la seguridad del servidor FTP.

```bash
sudo chmod a-w /home/krieger/ftp
```

Este comando quita el permiso de escritura para todos los usuarios en el directorio `/home/krieger/ftp`. Esto significa que nadie podrá modificar el contenido del directorio, lo que es útil para protegerlo de cambios no autorizados.

```bash
sudo ls -la /home/krieger/ftp
```

Este comando lista todos los archivos y directorios dentro de `/home/krieger/ftp`, mostrando detalles como permisos, propietario, grupo, tamaño y fecha de modificación. Se utiliza para verificar que los cambios de propiedad y permisos se hayan aplicado correctamente.

Crearemos la carpeta `files` si no nos la ha creado sola:

```bash
sudo mkdir /home/krieger/ftp/files
sudo chown krieger:krieger /home/krieger/ftp/files
```

El comando cambia la propiedad y el grupo del directorio `/home/krieger/ftp/files` a `krieger`, lo que permite que el usuario `krieger` y cualquier miembro del grupo `krieger` gestionen el contenido de ese directorio.

Le daremos los permisos y si hacemos un `ls`, nos saldrá todo:

```bash
sudo chmod 755 /home/krieger
sudo ls -la /home/krieger/ftp
```

Nos iremos al directorio `/etc/vsftpd.conf` y tendremos que añadir lo siguiente (lo que no está documentado) y también tendremos que tener la configuración igual que la tengo yo.

Luego nos iremos al `iptables` del router y tendremos que añadir lo siguiente a él, yo lo he puesto al final de la línea:

```bash
# Redirigir el rango de puertos pasivos al servidor FTP
iptables -t nat -A PREROUTING -i enp0s3 -p tcp --dport 40000:50000 -j DNAT --to 172.16.95.5:40000-50000
iptables -A FORWARD -p tcp --dport 40000:50000 -d 172.16.95.5 -j ACCEPT

IPTABLES_MODULES="nf_conntrack_ftp ip_nat_ftp"

# Redirigir puerto 21 al servidor FTP en la red interna
iptables -t nat -A PREROUTING -i enp0s3 -p tcp --dport 21 -j DNAT --to 172.16.95.5:21
iptables -A FORWARD -p tcp --dport 21 -d 172.16.95.5 -j ACCEPT

# Redirigir puerto 20 al servidor FTP en la red interna
iptables -t nat -A PREROUTING -i enp0s3 -p tcp --dport 20 -j DNAT --to 172.16.95.5:20
iptables -A FORWARD -p tcp --dport 20 -d 172.16.95.5 -j ACCEPT

# Redirigir puerto 80 al servidor FTP en la red interna
iptables -t nat -A PREROUTING -i enp0s3 -p tcp --dport 80 -j DNAT --to 172.16.95.5:80
```

Luego, si nos vamos a nuestra máquina, instalaremos el cliente FileZilla. En la barra de arriba, tendremos que poner la IP de nuestro router, el usuario y contraseña del usuario que creamos anteriormente. Una vez dado a conexión, comprobaremos que efectivamente nos dejará subir archivos.

Luego nos iremos al FTP y si hacemos un `ls` en `/home/krieger/ftp/files`, veremos que efectivamente nos saldrá el documento que hemos subido.

---
## Conclusión


- Objetivo del Proyecto:
  - El proyecto se centró en la configuración de una red utilizando diversas tecnologías y servicios, incluyendo routers, DHCP, DNS, FTP, Failover y DHCP Relay, con el fin de establecer una infraestructura de red funcional, segura y resiliente.

- Configuración de Routers:
  - Se configuraron múltiples routers con adaptadores de red adecuados, asegurando la correcta asignación de direcciones IP y la implementación de reglas de firewall mediante iptables para proteger la red interna.

- Implementación de DHCP:
  - Se instaló y configuró un servidor DHCP para la asignación dinámica de direcciones IP a los dispositivos en la red, lo que facilitó la gestión de la red y mejoró la eficiencia en la asignación de recursos.

- Implementación de DHCP Relay:
  - Se configuró un servidor DHCP Relay para permitir que los clientes en diferentes subredes obtengan direcciones IP de un servidor DHCP centralizado. Esto facilitó la gestión de direcciones IP en entornos más complejos, donde múltiples subredes requieren acceso a un único servidor DHCP.
  - La configuración del relay incluyó la modificación de las reglas de iptables para redirigir el tráfico DHCP adecuadamente, asegurando que las solicitudes de los clientes fueran correctamente encaminadas al servidor DHCP.

- Configuración de Failover:
  - Se implementó un sistema de Failover para el servidor DHCP, lo que permitió que dos servidores DHCP trabajaran en conjunto para proporcionar direcciones IP a los clientes. Esto garantiza que, en caso de que uno de los servidores falle, el otro pueda asumir la responsabilidad sin interrupciones en el servicio.
  - La configuración del Failover incluyó la sincronización de las bases de datos de direcciones IP entre los dos servidores, asegurando que ambos tuvieran información actualizada y coherente sobre las asignaciones de IP.

- Configuración de DNS:
  - Se implementaron servidores DNS para resolver nombres de dominio, lo que permitió a los clientes acceder a recursos de red utilizando nombres legibles en lugar de direcciones IP, mejorando la usabilidad de la red.



- FTP y Transferencia de Archivos:
  - Se configuró un servidor FTP para permitir la transferencia de archivos entre los usuarios de la red, asegurando que los permisos y la seguridad estuvieran correctamente gestionados.

- Pruebas y Verificación:
  - Se realizaron pruebas exhaustivas para verificar la funcionalidad de cada componente de la red, asegurando que los dispositivos pudieran comunicarse entre sí y acceder a Internet sin problemas. Se comprobó que los clientes podían obtener direcciones IP de manera efectiva a través del relay y que, en caso de fallo de uno de los servidores DHCP, el otro continuaba proporcionando direcciones IP sin problemas.

- Desafíos y Soluciones:
  - Durante el proceso, se enfrentaron desafíos relacionados con la configuración de iptables y la integración de diferentes servicios. Estos se resolvieron mediante la documentación cuidadosa y la aplicación de buenas prácticas en la configuración de red.

- Resultados:
  - Al finalizar el proyecto, se logró una red operativa que cumplía con los requisitos establecidos, proporcionando conectividad, seguridad y facilidad de uso para los usuarios finales. La implementación del Failover y el DHCP Relay mejoró significativamente la disponibilidad y la gestión de la red, permitiendo una experiencia de usuario más fluida y confiable.

- Recomendaciones Futuras:
  - Se sugiere realizar un monitoreo continuo de la red y actualizar las configuraciones según sea necesario para adaptarse a cambios en la infraestructura o en las necesidades de los usuarios. Además, se recomienda la implementación de medidas adicionales de seguridad para proteger la red contra amenazas externas y realizar pruebas periódicas de Failover para garantizar que el sistema responda adecuadamente en situaciones de fallo.

En resumen, el proyecto fue exitoso en la creación de una red robusta y funcional, demostrando la importancia de una planificación y ejecución cuidadosa en la configuración de redes. La capacidad de mantener la continuidad del servicio y la eficiencia en la asignación de direcciones IP, junto con la implementación de tecnologías como Failover y DHCP Relay, son aspectos críticos que se lograron con éxito.
