Instalacion de Ansible + Semaphore
==================================
Instalación y configuracion de Ansible
--------------------------------------
Para instalar Ansible, tal como lo muestra su documentación se deben seguir los siguientes comandos:

Se instalan las dependencias:
::
   $ sudo apt-get -y install software-properties-common
   $ sudo apt-get -y install python-software-properties

Se añade el repositorio del paquete:
::
   $ sudo apt-add-repository -y ppa:ansible/ansible

Se actualizan las listas de descargas de paquetes:
::
   $ sudo apt-get -y update

Se instala Ansible:
::
   $ sudo apt-get -y install ansible

Luego, se instala otra dependendencia posterior a Ansible:
::
   $ sudo apt-get -y install python-passlib

Una vez instalado ansible, para evitar que Ansible genere advertencias por los certificados de los host sobre los que trabajará, se desactiva el siguiente parametro del archivo de configuracion de Ansible que está en ``/etc/ansible/ansible.cfg`` usando ``nano`` o cualquier otro editor. Solo es necesario descomentar la linea:
::
  host_key_checking: False

Instalacion de Semaphore
------------------------
Para la instalacipon de Semaphore, es necesario tener en ejecución MySQL (se instalará la version 5.7).

Preparacion
~~~~~~~~~~~
Si existe una instalación de MySQL y no se recuerda algun parametro (como la contraseña de usuario) se recomienda reinstalar el servicio haciendo lo siguiente.

Removemos todos los paquetes que coincidan con los siguientes parametros
::
   $ sudo apt-get purge mysql-server mysql-client mysql-common mysql-server-core-* mysql-client-core-*

Se elimina el directorio de MySQL
::
   $ sudo rm -rf /etc/mysql /var/lib/mysql

Se elimina cualquier dependencia innecesaria:
::
   $ sudo apt-get autoremove

Se elimina del cache paquetes .deb con versiones anteriores a los de los programas que tienes instalados como MySQL.
::
   $ sudo apt-get autoclean

Instalacion de MySQL
~~~~~~~~~~~~~~~~~~~~
Se actualizan los paquetes y se instala MySQL server.
::
   $ sudo apt-get -y update
   $ sudo apt-get -y install mysql-server

Una vez instalado, se configura la contraseña del usuario root a traves del siguiente comando:
::
   $ sudo mysql_secure_installation

Preguntará si deseamos usar una contraseña, a lo que responderemos sí:
::
   Press y|Y for Yes, any other key for No: y

Luego elegimos una política de validacion de contraseñas. Se recomienda la política STRONG u opcion = 2.
::
   Please enter 0 = LOW, 1 = MEDIUM and 2 = STRONG: 2

Se ingresa la contraseña que cumpla con los requisitos. En este ejemplo se usó “12345678”.

Luego, preguntará una serie de opciones. Se recomienda responder ``y`` a todo.
 #. Remover usuarios anonimos: ``y``
 #. No permitir inicio de sesion de usuario root de forma remota: ``y``
 #. Remover base de datos de prueba llamada test: ``y``
 #. Recargar privilegios sobre las tablas: ``y``

Verificación de la instalación
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Se verifica si el servicio de MySQL esta activo:
::
  $ service mysql status
  ● mysql.service - MySQL Community Server
   Loaded: loaded (/lib/systemd/system/mysql.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2018-11-22 11:35:30 -05; 2min 36s ago
 Main PID: 31500 (mysqld)
    Tasks: 28 (limit: 4915)
   CGroup: /system.slice/mysql.service
           └─31500 /usr/sbin/mysqld --daemonize --pid-file=/run/mysqld/mysqld.pid

Si no está ejecutandose, se inicia: 
::
  $ sudo service mysql start

Ahora probemos las credenciales de acceso a la base de datos. Digitamos el comand e ingresamos la clave que definimos anteriormente.
::
  $ sudo mysqladmin -p -u root version
  Enter password:
  mysqladmin  Ver 8.42 Distrib 5.7.24, for Linux on x86_64
  Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

  Oracle is a registered trademark of Oracle Corporation and/or its affiliates. Other names may be trademarks of their respective owners.

Podemos verificar ademas el puerto por el cual el servicio de MySQL se comunica, a traves del siguiente comando:
::
  $ sudo netstat -tlpn
  Conexiones activas de Internet (solo servidores)
  Proto  Recib Enviad Dirección local         Dirección remota       Estado       PID/Program name    
  tcp        0      0 127.0.0.53:53           0.0.0.0:*               ESCUCHAR    16686/systemd-resol 
  tcp        0      0 127.0.0.1:631           0.0.0.0:*               ESCUCHAR    12912/cupsd         
  tcp        0      0 127.0.0.1:3306          0.0.0.0:*               ESCUCHAR    31500/mysqld        
  tcp6       0      0 ::1:631                 :::*                    ESCUCHAR    12912/cupsd         


El puerto por defecto de MySQL: ``127.0.0.1:3306``.

Preparación de MySQL para Semaphore
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Base de datos para Semaphore
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Se recomienda crear una base de datos para Semaphore. 

Para esto, ingresamos a mysql con el usuario root:
::
  $ sudo mysql -u root -h localhost -p

Ingresamos la clave y accedemos. Una vez allí, listamos las bases existentes:
::
  mysql> show databases;
    +------------------------------------+
    | Database                           |
    +------------------------------------+
    | information_schema                 |
    | mysql                              |
    | performance_schema                 |
    | sys                                |
    +------------------------------------+

Creamos la base de datos
::
  mysql> create database semaphore;
  Query OK, 1 row affected (0.00 sec)

Verificamos que se creó.
::
  mysql> show databases;
    +------------------------------------+
    | Database                           |
    +------------------------------------+
    | information_schema                 |
    | mysql                              |
    | performance_schema                 |
    | semaphore                          |
    | sys                                |
    +------------------------------------+

Salimos de MySQL y reiniciamos el servicio para activar los cambios.
::
  mysql> exit
  Bye
  $ sudo service mysql restart

Usuario para Semaphore
^^^^^^^^^^^^^^^^^^^^^^
Por cuestiones de protección y seguridad de MySQL, el ingreso desde Semaphore con el usuario root puede que genere problemas y deniegue el acceso del servicio. Por tanto, se recomienda crear un usuario con los privilegios de root, como se indica a continuación.

Accedemos a MySQL.
::
  $ sudo mysql -u root

Nos aseguramos que estamos en la base de datos mysql
::
  mysql> USE mysql;
  Database changed

Creamos un usuario ‘semaphore’ con la contraseña ``Supersecure123_``
::
  mysql> CREATE USER 'semaphore'@'localhost' IDENTIFIED BY 'Supersecure123_';
  Query OK, 0 rows affected (0.00 sec)

Le otorgamos al usuario todos los privilegios
::
  mysql> GRANT ALL PRIVILEGES ON *.* TO 'semaphore'@'localhost';
  Query OK, 0 rows affected (0.00 sec)

Definimos como plugin de autenticación del usuario ‘semaphore’ la contraseña nativa de mysql:
::
  mysql> UPDATE user SET plugin='mysql_native_password' WHERE User='semaphore';
  Query OK, 1 row affected (0.00 sec)
  Rows matched: 1  Changed: 1  Warnings: 0

  mysql> FLUSH PRIVILEGES;
  Query OK, 0 rows affected (0.00 sec)

De nuevo salimos de MySQL y reiniciamos el servicio.
::
  mysql> exit
  Bye
  $ sudo service mysql restart

Instalacion de Semaphore
~~~~~~~~~~~~~~~~~~~~~~~~
La instalación de Semaphore debe realizarse descargando el paquete de instalación desde los repositorios segun la arquitectura y el SO. Para ubuntu, cuya arquitectura es amd64, el enlace de descarga (hasta la fecha 22/11/2018) es: https://github.com/ansible-semaphore/semaphore/releases/download/v2.5.1/semaphore_2.5.1_linux_amd64.deb

Usamos wget para descargar el paquete. Si no está instalado, siga los comandos:
::
  $ sudo apt-get install -y wget
  $ wget https://github.com/ansible-semaphore/semaphore/releases/download/v2.5.1/semaphore_2.5.1_linux_amd64.deb

Una vez descargado, se procede a instalarse de la siguiente forma:
::
  $ sudo apt install ./semaphore_2.5.1_linux_amd64.deb

Configuración de Semaphore
~~~~~~~~~~~~~~~~~~~~~~~~~~
Antes de ejecutarse, se recomienda crear el siguiente directorio ``/opt/data/semaphore``. Para esto, nos dirigimos a la raiz:
::
  $ cd /
Entramos en ``opt/`` y creamos el directorio ``data/``:
::
  $ cd opt
  $ mkdir data

Entramos en ``data/`` y creamos el directorio ``semaphore/``
::
  $ cd data
  $ mkdir semaphore

Entramos al directorio ``semaphore/`` y vemos la ruta
::
  $ cd semaphore
  $ pwd
  /opt/data/semaphore

La ruta anterior será clave en pasos siguientes de la configuración de Semaphore.

Ahora, nos dirigimos al directorio de nuestra preferencia para continuar el proceso. Una vez allí, colocamos el comando que iniciará la configuración de Semaphore para su despliegue.
::
  $ semaphore -setup
 Hello! You will now be guided through a setup to:

 1. Set up configuration for a MySQL/MariaDB database
 2. Set up a path for your playbooks (auto-created)
 3. Run database Migrations
 4. Set up initial semaphore user & password

A continuacion se describirán los valores en cada parámetro. Donde no se indiquen valores, se da ``[enter]`` aceptando el valor por defecto.

En este parametro se indica la IP y puerto de MySQL, que en este caso es en local y el puerto por defecto.
::
  > DB Hostname (default 127.0.0.1:3306): [enter]

Luego, el usuario creado en MySQL para Semaphore.
::
  > DB User (default root): semaphore

La contraseña del usuario ‘semaphore’:
::
  > DB Password: Supersecure123_

El nombre de la base de datos para Semaphore, como se había definido antes.
::
  > DB Name (default semaphore): [enter]

Se indica la ruta creada con anteriorirdad.
::
  > Playbook path (default /tmp/semaphore): /opt/data/semaphore
  > Web root URL (optional, example http://localhost:8010/): [enter]
 > Enable email alerts (y/n, default n): [enter]
 > Enable telegram alerts (y/n, default n): [enter]
 > Enable LDAP authentication (y/n, default n): [enter]

 Generated configuration:
 {
     "mysql": {
         "host": "127.0.0.1:3306",
         "user": "semaphore",
         "pass": "Supersecure123_",
         "name": "semaphore"
     },
     "port": "",
     "tmp_path": "/opt/data/semaphore",
     "cookie_hash": "ETZ4IjSjRaMRJxJHcRkY719iIqxdRXyAF8lCuE+EjmA=",
     "cookie_encryption": "+vtXZBVFZUJVjJE8a3nhv8xAuogu8aXYMJ5WQE0MszQ=",
     "email_sender": "",
     "email_host": "",
     "email_port": "",
     "web_host": "",
     "ldap_binddn": "",
     "ldap_bindpassword": "",
     "ldap_server": "",
     "ldap_searchdn": "",
     "ldap_searchfilter": "",
     "ldap_mappings": {
         "dn": "",
         "mail": "",
         "uid": "",
         "cn": ""
     },
     "telegram_chat": "",
     "telegram_token": "",
     "concurrency_mode": "",
     "max_parallel_tasks": 0,
     "email_alert": false,
     "telegram_alert": false,
     "ldap_enable": false,
     "ldap_needtls": false
 }

Confirmada la información ingresada, validamos y continuamos el proceso marcando ``yes``:
::
  > Is this correct? (yes/no): yes

El siguiente directorio es la ruta donde estarán alojados los archivos de salida que Semaphore genere. Se puede indicar la ruta de su preferencia.
::
  > Config output directory (default /home/labredes-15/Documentos/semaphore_instalacion):

Una vez hecho lo anterior, se comenzará con la configuración y migración de las bases de datos a MySQL
::
  Running: mkdir -p /home/labredes-15/Documentos/semaphore_instalacion..
  Configuration written to /home/labredes-15/Documentos/semaphore_instalacion/config.json..
  Pinging db..

  Running DB Migrations..
  Checking DB migrations
  Creating migrations table
  Executing migration v0.0.0 (at 2018-11-22 12:45:31.210497156 -0500 -05 m=+47.147041162)...
 [11/11]
  Executing migration v1.0.0 (at 2018-11-22 12:45:31.939999179 -0500 -05 m=+47.876543161)...
  [7/7]
  Executing migration v1.1.0 (at 2018-11-22 12:45:32.670561977 -0500 -05 m=+48.607105974)...
  [1/1]
  Executing migration v1.2.0 (at 2018-11-22 12:45:32.829611754 -0500 -05 m=+48.766155755)...
  [1/1]
  Executing migration v1.3.0 (at 2018-11-22 12:45:32.897156599 -0500 -05 m=+48.833700597)...
  [3/3]
  Executing migration v1.4.0 (at 2018-11-22 12:45:33.228799723 -0500 -05 m=+49.165343704)...
  [2/2]
  Executing migration v1.5.0 (at 2018-11-22 12:45:33.415970512 -0500 -05 m=+49.352514494)...
  [1/1]
  Executing migration v0.1.0 (at 2018-11-22 12:45:33.491666899 -0500 -05 m=+49.428210889)...
  [6/6]
  Executing migration v1.6.0 (at 2018-11-22 12:45:33.694602638 -0500 -05 m=+49.631146620)...
  [4/4]
  Executing migration v1.7.0 (at 2018-11-22 12:45:34.137515442 -0500 -05 m=+50.074059424)...
  [1/1]
  Executing migration v1.8.0 (at 2018-11-22 12:45:34.247516946 -0500 -05 m=+50.184060928)...
  [2/2]
  Executing migration v1.9.0 (at 2018-11-22 12:45:34.357351367 -0500 -05 m=+50.293895362)...
  [2/2]
  Executing migration v2.2.1 (at 2018-11-22 12:45:34.496512651 -0500 -05 m=+50.433056633)...
  [2/2]
  Executing migration v2.3.0 (at 2018-11-22 12:45:34.694145835 -0500 -05 m=+50.630689822)...
  [3/3]
  Executing migration v2.3.1 (at 2018-11-22 12:45:35.00776799 -0500 -05 m=+50.944311971)...
  [1/1]
  Executing migration v2.3.2 (at 2018-11-22 12:45:35.029190089 -0500 -05 m=+50.965734087)...
  [1/1]
  Executing migration v2.4.0 (at 2018-11-22 12:45:35.054947161 -0500 -05 m=+50.991491143)...
  [1/1]
  Executing migration v2.5.0 (at 2018-11-22 12:45:35.156617357 -0500 -05 m=+51.093161339)...
  [1/1]
  Migrations Finished

Cuando finalice, se indicará ahora el usuario, email y clave de acceso a la interfaz de Semaphore.
::
 > Username: myuser
 > Email: kpedrozag@gmail.com
 > Your name: kevin
 > Password: pass

 You are all setup kevin!
 Re-launch this program pointing to the configuration file

  ./semaphore -config /home/labredes-15/Documentos/semaphore_instalacion/config.json

 To run as daemon:

  nohup ./semaphore -config /home/labredes-15/Documentos/semaphore_instalacion/config.json &

 You can login with kpedrozag@gmail.com or myuser.

Finalizado lo anterior, ya se ha concluido con todo el proceso. Ahora es posible acceder a la interfaz web. Para esto, vamos a la ruta indicada de los archivos de salida, donde estará el archivo ``config.json`` que permite el despliegue del servicio.

Lo iniciamos con el comando:
::
  $ semaphore -config config.json

Lo cual generará lo siguiente:
::
  Using config file: config.json
  Semaphore v2.5.1
  Port :3000
  MySQL semaphore@127.0.0.1:3306 semaphore Tmp Path (projects home) /opt/data/semaphore
  Checking DB migrations
 403  12:48:00       6.043µs |   GET     /api/user

Ahora, desde el navegador y entrando a ``http://localhost:3000`` entraremos al Login de la interfaz web. Se ingresan las credenciales, que en este caso es el nombre de usuario ``myuser`` y la clave ``pass``.

Más información en:
 * TheDumbTechGuy (2018). Install Ansible Semaphore on Ubuntu. Retrieved from https://gist.github.com/thedumbtechguy/0fae1ae931042829b73426630f3cd168
 * Zetacu. (2018). ERROR 1698 (28000): Access denied for user 'root'@'localhost'. Retrieved from https://stackoverflow.com/questions/39281594/error-1698-28000-access-denied-for-user-rootlocalhost