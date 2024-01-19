# Tarea 04 · Despliegue de Aplicaciones Web
___
## Oliver Fabian Stetcu Stepanov
___
### Tarea DNS · Servicio de nombres de dominios
___
## Bind 9
* https://www.isc.org/bind/
* https://bind9.readthedocs.io/
* https://www.fpgenred.es/DNS/index.html

# Infraestructura

Reutilizaremos las MV de la práctica de ``ssh``. Dos MV dentro de una ``red NAT``:
* **Servidor**: con un Ubuntu server sin entorno gráfico.
    * Usuario: ``sergio``, contraseña: ``sergio``.
* **Casa**: con un Lubuntu con el entorno gráfico por defecto (LXQt).
    * Usuario: ``carmen``, contraseña: ``carmen``.

Desde el equipo **Casa** nos conectaremos al equipo **Servidor** mediante una conexión ``ssh`` autentificándonos mediante claves asimétricas ``ed25519``.

## Instalación y uso básico

1. Acceder al servidor:

Desde el equipo de **Casa** ejecutamos el siguiente comando:

```bash
ls -la .ssh
cd ~/.ssh
ssh -p 22 -i clave_trabajo sergio@10.0.2.8
```

Utilizo la clave generada en la Tarea 01 (Tarea SSH - SCP - Shell - VirtualBox), también reutilizada para la Tarea 03, la clave se llama "**clave_trabajo**" (se puede omitir poner el puerto "-p 22"):

Resultado:

![Conectar de "Casa" a "Servidor"](./img/01_dns.png)

![Conectar de "Casa" a "Servidor"](./img/02_dns.png)

> Casi toda la instalación y configuración la debemos hacer con privilegios de administrador podemos ejecutar ``sudo`` en todas las instrucciones o cambiar al usuario administrador ``sudo su``.

2. Instalar bind 9:

```bash
sudo apt update
sudo apt install bind9 bind9utils
```

Resultado:

![Actualizo dependencias](./img/03_dns.png)

![Instalo "bind 9"](./img/04_dns.png))

> La instalación crea el usuario ``bind`` que ejecuta el servicio dns denominado ``named``. Puedes comprobarlo mostrando el contenido del archivo ``/etc/passwd``.

3. Comprobar estado del servicio ``bind``:

```bash
sudo systemctl status bind9
```

Resultado:

![Compruebo el estado de "bind 9"](./img/05_dns.png)

> Muestra advertencias ya que aún no lo hemos configurado.

4. Con los siguientes comandos lo activaremos para que se inicie al arrancar el servidor y lo iniciaremos:

```bash
sudo systemctl enable bind9
sudo systemctl enable named.service
sudo systemctl start bind9
sudo systemctl restart bind9
sudo systemctl status bind9
```

Resultado:

![Activo el inicio de arranque del servidor](./img/06_dns.png)

![Activo el inicio de arranque del servidor](./img/07_dns.png)

Con el comando ``sudo systemctl enable bind9`` me da error, pero si le cambio el ``bind9`` por ``named.service`` sí me deja, luego le hago el **start** al servicio y muestro el estado del servicio. Comprobamos que se está habilitado e iniciado. A la hora de hacer el **restart** ya no me aparece la línea de warning en amarillo.

5. Reglas firewall:

```bash
sudo ufw enable
sudo ufw allow bind9
sudo ufw status
```

Resultado:

![Reglas de Firewall](./img/08_dns.png)

6. Probar desde el cliente qué puertos tiene abiertos el servidor, en nuestro ejemplo desde el equipo **Casa** ejecutaremos:

```bash
exit
sudo apt install nmap
nmap 10.0.2.8 -p 1-1024
```

Resultado:

![Muestro los puertos abiertos del "Servidor"](./img/09_dns.png)

> Por defecto el servicio DNS utiliza el puerto 53.

> Si no tienes instalada esta utilidad, instalalá con: ``sudo apt install nmap``. Esta comprobación también se puede hacer desde el propio servidor, pero es menos fiable que desde otro equipo ya que puede conectarse por localhost.

7. Comprobar en el equipo Servidor qué conexiones tiene abiertas:

```bash
ssh -p 22 -i clave_trabajo sergio@10.0.2.8
sudo ss -natp | grep named
sudo ss -naup | grep named
```

Resultado:

![Comprobar en el "Servidor" las conexiones abiertas](./img/10_dns.png)

## Archivos de configuración

1. El archivo principal de configuración del ``bind`` es: ``/etc/bind/named.conf``. En él vemos que hace referencia a otros tres archivos de configuración:

* ``/etc/bind/named.conf.options``: hace referencia al archivo de configuración que posee
opciones genéricas.
* ``/etc/bind/named.conf.local``: hace referencia al archivo de configuración para opciones
particulares.
* ``/etc/bind/named.conf.default-zones``: hace referencia al archivo de configuración de
zonas.

Abre el archivo y muesta estas referencias.

```bash
sudo nano /etc/bind/named.conf
```

Resultado:

![Muestro las referencias a los archivos de configuración](./img/11_dns.png)

## Verificar archivos de configuración

1. Puedes realizar una verificación de los ficheros de configuración y de zona por posibles fallos mediante los comandos ``named-checkconf`` y ``named-checkzone`` respectivamente. Estos comandos suelen ejecutarse con la siguiente sintaxis:

* ``named-checkconf [-p] {filename}``: Comprueba la sintaxis, pero no la semántica de un
fichero de configuración named. El fichero se analiza y comprueba por errores de sintaxis,
junto con todos los archivos incluidos en él.

  * Parámetros:

    * El parámetro ``-p``: imprime la salida de named.conf y los ficheros incluidos en forma
    canónica si no fueron detectados errores.

    * ``filename``: es el nombre del archivo de configuración que se desea comprobar. Si no se
    especifica, por defecto es ``/etc/named.conf``.

* ``named-checkzone {zonename} {filename}``: Comprueba la sintaxis y la integridad de un
archivo de zona. Realiza las mismas comprobaciones que hace named al cargar una zona. Esto
hace que sea útil para comprobar los archivos de zona antes de configurarlos en un servidor de
nombres.

    * Parámetros:

        * ``zonename`` es el nombre de dominio de la zona que se comprueba.

        * ``ilename`` es el nombre del archivo de zona.

Prueba a verificar los siguientes archivos:

Verificar el fichero de configuración:

```bash
named-checkconf -p /etc/bind/named.conf
```

Resultado:

![Verifico el archivo de configuración](./img/12_dns.png)

Verificar el dominio de zona "ejemplo.com" en el archivo de zona:

```bash
named-checkzone ejemplo.com /etc/bind//db.local
```

Resultado:

![Verifico el dominio de zona "ejemplo.com" en el archivo de zona](./img/13_dns.png)

## Configurar servidor DNS
### Configurar Reenviadores (forwarders)

1. Primero indicar que cuando se ejecute ``bind`` lo haga solo sobre IPv4. Editar el archivo ``named`` y añadir ``-4``.

```bash
sudo nano /etc/default/named
```

Añadir "``OPTIONS="-u bind -4``".

Resultado:

![Edito el archivo "named" para que ejecute "bind" solo en IPv4](./img/14_dns.png)

2. Añadir al bloque ``options`` del archivo ``named.conf.options`` los reenviadores e indicar que no se validen las conexiones seguras DNS con las instrucciones siguientes:

```bash
sudo nano /etc/bind/named.conf.options
```

Añadir ``"forwarders {
 1.1.1.1;
 8.8.8.8;
};``" y "``dnssec-validation no;"``.

Resultado:

![Edito el archivo "named.conf.options" para no se validen las conexiones seguras DNS](./img/15_dns.png)

3. Verificar el archivo anterior:

```bash
sudo named-checkconf /etc/bind/named.conf.options
```

Resultado:

![Verifico el archivo "named.conf.options"](./img/16_dns.png)

* 1. Verificar el archivo de configuración principal ``named.conf``:

```bash
sudo named-checkconf
```

Resultado:

![Verifico el archivo "named.conf"](./img/17_dns.png)

4. Reiniciar el servicio ``bind``:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

Resultado:

![Reinicio el servicio "bind" y muestro el estado](./img/18_dns.png)

5. Los clientes ya pueden resolver **direcciones externas** manejadas por los reenviadores (forwarders):

Desde **Servidor**.

```bash
dig @localhost iesaguadulce.es
o
dig @10.0.2.8 iesaguadulce.es
o
dig iesaguadulce.es
```

Resultado:

![Resolver "direcciones externas" manejadas por reenviadores](./img/19_dns.png)

![Resolver "direcciones externas" manejadas por reenviadores](./img/20_dns.png)

Desde **Casa**.

```bash
exit
dig @localhost iesaguadulce.es
o
dig @10.0.2.8 iesaguadulce.es
o
dig iesaguadulce.es
```

Resultado:

![Resolver "direcciones externas" manejadas por reenviadores](./img/21_dns.png)

![Resolver "direcciones externas" manejadas por reenviadores](./img/22_dns.png)

Ejecuto el comando anterior dos veces en cada equipo y comparar el tiempo de respuesta (Query
time):

Vuelvo a ejecutarlo desde **Servidor**:

![Resolver "direcciones externas" manejadas por reenviadores](./img/23_dns.png)
![Resolver "direcciones externas" manejadas por reenviadores](./img/24_dns.png)

Vuelvo a ejecutarlo desde **Casa**:

![Resolver "direcciones externas" manejadas por reenviadores](./img/25_dns.png)

# Configurar zonas

1. Editar el archivo ``named.conf.local``

```bash
sudo nano `/etc/bind/named.conf.local`
```

Añado una zona con el formato **tuapellido.local**, por ejemplo: **garcia.local, lopez.local**. En este ejemplo usaremos despliegue.local. Esta zona será de la red NAT creada en VirtualBox **10.0.2.0/24**. Crea también su zona inversa.

Resultado:

![Edito el archivo "named.conf.local"](./img/26_dns.png)

2. Verificar el archivo anterior:

```bash
sudo named-checkconf /etc/bind/named.conf.local
```

Resultado:

![Verifico el archivo "named.conf.local"](./img/27_dns.png)

3. Crear la carpeta y los archivos de zonas (directa e inversa):

```bash
sudo mkdir /etc/bind/zones
```

Resultado:

![Creo la carpeta y los archivos de zonas directa e inversa](./img/28_dns.png)

4. Crear el archivo de la zona directa desde la plantilla ``db.local``:

```bash
sudo cp /etc/bind/db.local /etc/bind/zones/db.stetcu.local
```

Resultado:

![Creo el archivo de la zona directa](./img/29_dns.png)

Editar la zona directa y añadir los equipos **casa** y **servidor**. Crea tambien un alias para servidor con el nombre que quieras, en el ejemplo **server**:

```bash
sudo nano /etc/bind/zones/db.stetcu.local
```

Resultado:

![Edito la zona directa y añado los equipos](./img/30_dns.png)

5. Verificar el archivo anterior:

```bash
sudo named-checkzone despliegue.local /etc/bind/zones/db.despliegue.local
```

Resultado:

![Verifico el archivo anterior](./img/31_dns.png)

6. Crear el archivo de la zona inversa desde la plantilla ``db.local`` o desde el archivo de zona recién creado:

```bash
sudo cp /etc/bind/db.local /etc/bind/zones/db.2.0.10
o
sudo cp /etc/bind/zones/db.stetcu.local /etc/bind/zones/db.2.0.10
```

Resultado:

![Creo el archivo de zona inversa](./img/32_dns.png)

Editar el archivo de la zona inversa:

```bash
sudo nano /etc/bind/zones/db.2.0.10
```

Resultado:

![Edito el archivo de zona inversa](./img/33_dns.png)

7. Verificar el archivo anterior:

```bash
sudo named-checkzone 2.0.10.in-addr.arpa /etc/bind/zones/db.2.0.10
```

Resultado:

![Verifico el archivo de zona inversa](./img/34_dns.png)

8. Reiniciar el servicio ``bind``:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

Resultado:

![Reinicio el servicio "bind" y muestro el estado](./img/35_dns.png)

9. Los clientes ya pueden resolver direcciones de la **zona recién creada**.

Desde **Servidor**.

```bash
dig @localhost servidor.stetcu.local
dig @localhost casa.stetcu.local
dig @localhost server.stetcu.local
o
dig @10.0.2.8 servidor.stetcu.local
dig @10.0.2.8 casa.stetcu.local
dig @10.0.2.8 server.stetcu.local
```

Resultado:

![Resuelvo direcciones en la zona recién creada](./img/36_dns.png)

![Resuelvo direcciones en la zona recién creada](./img/37_dns.png)

![Resuelvo direcciones en la zona recién creada](./img/38_dns.png)

Desde **Casa**.

```bash
exit
dig @10.0.2.8 servidor.stetcu.local
dig @10.0.2.8 casa.stetcu.local
dig @10.0.2.8 server.stetcu.local
```

Resultado:

![Resuelvo direcciones en la zona recién creada](./img/39_dns.png)

![Resuelvo direcciones en la zona recién creada](./img/40_dns.png)

![Resuelvo direcciones en la zona recién creada](./img/41_dns.png)

10. [OPCIONAL] Si queremos hacer resoluciones de nombres sin tener que escribir todo el nombre
(formato FQDN) y usar por ejemplo: ``host casa`` o ``host servidor`` necesitamos establecer el
dominio de búsqueda local. Como no tenemos configurado un servidor ``DHCP`` para recibir las
direcciones de los servidores de nombres y el dominio de búsqueda local, configuraremos a mano
estos datos en el archivo temporal ``/etc/resolv.conf`` tanto en **Servidor** como en **Casa**. Editar los archivos ``/etc/resolv.conf`` en ambos equipos y añadir las primera y última línea:

Desde **Casa**.

```bash
sudo nano /etc/resolv.conf
```

Resultado:

![Parte opcional](./img/42_dns.png)

Desde **Servidor**.

Resultado:

![Parte opcional](./img/43_dns.png)

> El servidor de nombres (nameserver) y el dominio de búsqueda local (search) se puede
configurar en un DHCP de modo que se reciban estos datos junto con la configuración de red (IP, máscara de red y gateway).

11. [OPCIONAL] Ya funciona la resolución usando el dominio de búsqueda local:

```bash
host iesaguadulce.es
host casa.despliegue.local
host casa
host servidor
host servidor.despliegue.local
host server
host server.despliegue.local
```

Resultado:

![Parte opcional 2](./img/44_dns.png)

> El archivo ``/etc/resolv.conf`` es dinámico y ejecutar algunos comando como ``dig`` hacen que
se vuelva a actualizar con sus valores iniciales.

## Oliver Fabian Stetcu Stepanov
