# Instalación de un servidor Oracle 12c
En este punto se explica la instalación de Oracle Database 12c Release 2 (12.2.0.1.0) - Enterprise Edition en Debian Jessie (8.11.0).

### Configuración de la máquina servidor
Para la máquina servidor se utiliza el sistema operativo Debian Jessie (8.11.0) con 30 GB de disco duro y 2 GB de memoria RAM. 

Se recomienda usar los siguientes repositorios debian:
~~~
# deb cdrom:[Debian GNU/Linux 8.11.0 _Jessie_ - Official amd64 NETINST Bi$
#deb cdrom:[Debian GNU/Linux 8.11.0 _Jessie_ - Official amd64 NETINST Bin$
deb http://ftp.es.debian.org/debian/ jessie main contrib non-free
deb-src http://ftp.es.debian.org/debian/ jessie main contrib non-free
deb http://security.debian.org/ jessie/updates main contrib non-free
deb-src http://security.debian.org/ jessie/updates main contrib non-free
~~~

Una vez estén los repositorios correctos se actualizan los paquetes:
~~~
$ sudo apt-get update
~~~

### Creación de grupos, usuarios, directorios y enlaces
Para la instalación, en la máquina servidor, hay que añadir los grupos y el usuario siguientes:
~~~
$ sudo addgroup --system oinstall
Añadiendo el grupo `oinstall' (GID 126) ...
Hecho.
$ sudo addgroup --system dba
Añadiendo el grupo `dba' (GID 127) …
Hecho.
$ sudo adduser --system --ingroup oinstall -shell /bin/bash oracle
Añadiendo el usuario del sistema `oracle' (UID 119) ...
Añadiendo un nuevo usuario `oracle' (UID 119) con grupo `oinstall' ...
Creando el directorio personal `/home/oracle' ...
$ sudo adduser oracle dba
Añadiendo al usuario `oracle' al grupo `dba' ...
Añadiendo al usuario oracle al grupo dba
Hecho.
$ sudo passwd oracle
Introduzca la nueva contraseña de UNIX:
Vuelva a escribir la nueva contraseña de UNIX:
passwd: contraseña actualizada correctamente
~~~

Tras loguearse como el nuevo usuario Oracle, hay que crear los siguientes directorios y cambiar la propiedad de los mismo:
~~~
$ sudo mkdir -p /opt/oracle/product/12.2.0.1
$ sudo mkdir -p /opt/oraInventory
$ sudo chown -R oracle:dba /opt/ora*
~~~

Además, se crean los siguientes enlaces con la opción -s para que se creen enlaces simbólicos en lugar de enlaces duros. 
~~~
$ sudo ln -s /usr/bin/awk /bin/awk
$ sudo ln -s /usr/bin/basename /bin/basename
$ sudo ln -s /usr/bin/rpm /bin/rpm
$ sudo ln -s /usr/lib/x86_64-linux-gnu /usr/lib64
~~~

### Configuración en el fichero /etc/sysctl.d/local-oracle.conf
Para realizar la configuración del fichero /etc/sysctl.d/local-oracle.conf en primer lugar hay que saber los valores del sistema que se deben introducir. Para fs.file-max, fs.aio-max-nr, kernel.sem y kernel.shmmni se usan los siguientes comandos:
~~~
$ sudo sysctl -a | grep '^fs.file-max'
fs.file-max = 204094
$ sudo sysctl -a | grep '^fs.aio-max-nr'
fs.aio-max-nr = 65536
$ sudo sysctl -a | grep '^kernel.sem'
kernel.sem = 250    32000    32    128
kernel.sem_next_id = -1
$ sudo sysctl -a | grep '^kernel.shmmni'
kernel.shmmni = 4096
~~~

El valor de kernel.shmmax se consigue con:
~~~
$ sudo free -b | grep -Ei 'mem:' | awk '{print $2}'
2107785216
~~~

Para calcular el kernel.shmall se aplica la fórmula: (kernel.shmmax -1) / kernel.shmmni, que en nuestro caso da 514595.

Para ver el valor vm.hugetlb_shm_group (GID del grupo, dba) se utiliza:
~~~
$ cat /etc/group | grep dba
dba:x:127:oracle
~~~

Es recomendable cambiar el valor del número de páginas:
~~~
$ sudo sysctl -w vm.nr_hugepages=64
vm.nr_hugepages = 64
$ sudo sysctl -p
$ sudo sysctl -a | grep '^vm.nr_hugepages'
vm.nr_hugepages = 64
vm.nr_hugepages_mempolicy = 64
~~~

Una vez que se conocen todos los valores necesarios se agrega esta información en el fichero /etc/sysctl.d/local-oracle.conf:
~~~
## Valor del número máximo de manejadores de archivos. ##
fs.file-max = 204094
fs.aio-max-nr = 65536
## Valor de los parámetros de semáforo en el orden listado. ##
## semmsl, semmns, semopm, semmni ##
kernel.sem = 250 32000 32 128
## Valor de los tamaños de segmento de memoria compartida. ##
## (Oracle recomienda total de RAM -1 byte) 2GB ##
kernel.shmmax = 2107785216
kernel.shmall = 514595
kernel.shmmni = 4096
## Valor del rango de números de puerto. ##
net.ipv4.ip_local_port_range = 1024 65000
## Valor del número gid del grupo dba. ##
vm.hugetlb_shm_group = 127
## Valor del número de páginas de memoria. ##
vm.nr_hugepages = 64
~~~

Y se carga la configuración al sistema:
~~~
$ sudo sysctl -p /etc/sysctl.d/local-oracle.conf
~~~


### Configuración de otros ficheros
El fichero local-oracle.conf se debe crear y configurar por seguridad de la siguiente forma:
~~~
## Número máximo de procesos disponibles para un solo usuario. ##
oracle      	soft	nproc       	2047
oracle      	hard	nproc       	16384
## Número máximo de descriptores de archivo abiertos para un solo usuario. ##
oracle      	soft	nofile      	1024
oracle      	hard	nofile      	65536
## Cantidad de RAM para el uso de páginas de memoria. ##
oracle      	soft	memlock     	204800
oracle      	hard	memlock     	204800
~~~

También se deben configurar las variables de entornos en el fichero /etc/bash.bashrc:
~~~
## Nombre del equipo ##
export ORACLE_HOSTNAME=localhost
## Usuario con permiso en archivos Oracle. ##
export ORACLE_OWNER=oracle
## Directorio que almacenará los distintos servicios de Oracle. ##
export ORACLE_BASE=/opt/oracle
## Directorio que almacenará la base de datos Oracle. ##
export ORACLE_HOME=/opt/oracle/product/12.2.0.1/dbhome_1
## Nombre único de la base de datos. ##
export ORACLE_UNQNAME=oraname
## Identificador de servicio de escucha. ##
export ORACLE_SID=orasid
## Ruta a archivos binarios. ##
export PATH=$PATH:/opt/oracle/product/12.2.0.1/dbhome_1/bin

## Ruta a la biblioteca. ##
export LD_LIBRARY_PATH=/opt/oracle/product/12.2.0.1/dbhome_1/lib
## Idioma
export NLS_LANG='SPANISH_SPAIN.AL32UTF8'
~~~

Y se configura el idioma para la instalación en español:
~~~
$ sudo dpkg-reconfigure locales

 ┌────────────────────┤ Configuración de locales ├─────────────────────┐
 │ Las configuraciones regionales componen un sistema para cambiar 	   │
 │ entre varios idiomas, y permite a los usuarios utilizar su idioma,  │
 │ país, juego de caracteres, ordenación alfanumérica, etc.        	   │
 │                              	                                   │
 │ Por favor, elija las configuraciones regionales que desea generar.  │
 │ Se recomiendan las configuraciones regionales UTF-8, especialmente  │
 │ para instalaciones nuevas. Otros juegos de caracteres pueden   	   │
 │ resultar útiles por compatibilidad con sistemas y software      	   │
 │ antiguo.                                                            │
 │                                                                     │
 │ Seleccione las configuraciones regionales que desea generar:    	   │
 │ . . . 											                   │
 │    [*] es_ES.UTF-8 UTF-8                                    	↓      │
 │              	<Aceptar>             	<Cancelar>           	   │
 │                                                                     │
 └─────────────────────────────────────────────────────────────────────┘
~~~
~~~
 ┌────────────────────┤ Configuración de locales ├─────────────────────┐
 │ Muchos paquetes en Debian utilizan las configuraciones regionales   │
 │ para mostrar el texto en el idioma de los usuarios. Puede elegir	   │
 │ la opción predeterminada de entre las configuraciones regionales	   │
 │ que ha generado.                                                    │
 │                                                                     │
 │ Esto seleccionará el idioma predeterminado de todo el sistema. Si   │
 │ se trata de un sistema con varios usuarios en el que no todos   	   │
 │ hablan el idioma elegido, pueden tener problemas.               	   │
 │                                                                     │
 │ Configuración regional predeterminada para el entorno del sistema:  │
 │                                                                     │
 │                         	Ninguno                             	   │
 │                         	C.UTF-8                             	   │
 │                         	es_ES.UTF-8                         	   │
 │                                                                     │
 │              	<Aceptar> 	            <Cancelar>           	   │
 │                                                                     │
 └─────────────────────────────────────────────────────────────────────┘ 
~~~
~~~
Generating locales (this might take a while)...
  es_ES.UTF-8... done
Generation complete.
~~~

### Instalación de paquetes
En primer lugar, hay que instalar los siguientes paquetes:
    • Build-essential: para crear paquetes debian.
    • Binutils: proporciona las herramientas de desarrollo necesarias para desempaquetar.
    • Libcap-dev: implementa las interfaces del usuario para POSIX 1003.1 e disponibles en el kernel Linux.
    • Gcc: compilador de GNU C.
    • G++: compilador G++.
    • Libc6-dev: biblioteca de C de GNU.
    • Ksh: para automatizar en sistemas basados en Unix.
    • Libaio-dev: biblioteca que permite usar el sistema de llamadas de entrada y salida asíncronas del kernel.
    • Make: utilidad de la línea de comandos que permite descargar la última versión de otras herramientas del sistema. 
    • Libxi-dev: biblioteca de extensión para X11.
    • Libxau-dev: biblioteca de autorización de X11.
    • Libxtst-dev: biblioteca de extensión de grabación X11.
    • Libxcb1-dev: biblioteca que proporciona una interfaz para el protocolo del sistema de ventanas X. 
    • Sysstat: herramienta de rendimiento del sistema para Linux.
    • Rpm: para gestionar paquetes .rpm.
    • Xauth: es una herramienta para leer y manipular los archivos Xauthority.
    • Xorg: implementación de código abierto del sistema de ventanas X.
    • Unzip: desarchivador de archivos .zip. 
~~~
$ sudo apt-get -y install build-essential binutils libcap-dev gcc g++ libc6-dev ksh libaio-dev make libxi-dev libxtst-dev libxau-dev libxcb1-dev sysstat rpm xauth xorg unzip
~~~

Una vez descargados, desde la máquina anfitriona, se descomprime el .zip que contiene el paquete de Oracle y que con anterioridad se ha descargado de la página oficial de Oracle. A continuación,se envía a la máquina que actuará como servidor Oracle. 
~~~
$ cd Descargas/
$ unzip linuxx64_12201_database.zip
$ scp -r database/ oracle@192.168.43.6:
oracle@192.168.43.6's password: 
~~~

Se vuelve a la máquina servidora. Si se hace a través de ssh, con la opción -X que permite utilizar la interfaz gráfica de la máquina anfitriona. Importante: para ello hay que tener correctamente instalado el paquete X11forward. Y se comienza la instalación:
~~~
$ ssh -XC oracle@192.168.43.6
oracle@192.168.43.6's password:
$ database/runInstaller -IgnoreSysPreReqs -ignorePrereq
~~~

### Consola de instalación
La consola de instalación de Oracle es una herramienta sencilla, pero es muy importante que todos los pasos se realicen correctamente. En la primera pantalla hay que introducir un correo electrónico para recibir información sobre los problemas de seguridad de Oracle, es opcional. 


*
*
*
*
*
*
*
*
*
*********** Voy por la foto que hay detras de este renglon:
En cuanto a la edición que se instalará, existen de nuevo dos opciones, Entrerprise que es más exigente, es la que elegiremos, y Standard con necesidades más básicas. 

