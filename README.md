# **Instalación de servicios mediante Ansible**

------------


**Se cuenta con dos servidores linux, uno Debian con Ubuntu y un RedHat con Rocky.**

El servidor Ubuntu contará con el servicio MariaDB y el servidor Rocky con el servicio de webserver Tomcat mediante un contenedor, entre otras configuraciones.

### Stages

- Actualizar repositorio de los servidores

- Instalación de Podman y Pull de imagen Tomcat para ejecución del contenedor.

- Instalación del servicio HTTPD y hablitación de los puertos HTTP y HTTPS del servidor Rocky. Importación del fichero index.html desde el repositorio y la configuración del virtualhost.conf.

- Instalación de MariaDB en Ubuntu utilizando MYSQL SECURE INSTALLATION. Habilitación del puerto 3306/tcp.


------------

#### Actualización de repositorios

Mediante el playbook packages_installer.yml ejecutamos una serie de instrucciones para actualizar los paquetes de Ubuntu y Rocky.

#### Instalación de PODMAN, Pull de imagen Tomcat y ejecución del contenedor

En primera instancia instalamos podman con el playbook podman_installer.yml.

```
    yum:
      name: podman
      state: present
    notify: Restart server

   handlers:
  - name: Restart server
    reboot:

```
Una vez que se instale el servicio en el servidor rocky, mediante un handler se reinicia el mismo. Antes de ejecutar el handler que contiene el reboot, inicia y habilita el servicio.

```
 service:
      name: podman
      state: started
      enabled: true
```

Una vez levantado podman, se procede a ejecutar el playbook execute_podman.yml.
- Automáticamente se ingresa las credenciales para logearse a la registry, que en este caso es quay.io
- Realiza el Pull de la imagen Tomcat que está en quay.io/sambrence/tallerjulio2023/tomcat-demo.
- Ejecuta la imagen en un contenedor, que denomina como tomcat-demo. Expone los puertos que utilizará Tomcat para que pueda ser accedido por fuera del contenedor.
- Habilita reglas de SELinux.  Activa httpd_can_network_connect y httpd_can_network_relay.

#### Instalación de HTTPD y habilitación de puertos.

Mediante playbook llamado httpd_installer.yml instalamos el servicio y realizamos sus configuraciones.
- Instala el servicio HTTPD con la última versión.
- Habilita los puertos del serivicio firewalld correspondientes a http y https.
- Inicia y habilita el servicio de httpd.services.
- Importa una copía del "index.html" desde el repositorio de GIT, en /sharedfiles.
- Configura directorio del VirtualHost. 
- Incluye el directorio del hosts, pasándole la línea IncludeOptional a los ficheros terminados en .conf en /etc/httpd/vhosts.d/.
- Mediante un handler y notify reinicia el servicio httpd.

#### Instalación de MariaDB y MYSQL Secure Installation.

El playbook mariadb_pb.yml instala en el servidor Ubuntu el servicio de base de datos MariaDB utilizando MYSQL Secure Installation para aplicar mayor seguridad.

Declaramos un vars_files que incluye al dbpwdsecure.yml que contiene, de forma encriptada, la contraseña del usuario root del servicio mysql.
- Ejecuta apt_key para validar la integridad de los paquetes a descargar.
- Agrega el repositorio para la descarga del servicio MariaDB, posteriormente realiza un update del cache.
- Ejecuta la instalación del servidor del servicio.
- Ejecuta la instalación del cliente del servicio MariaDB.
- Se instala Python3.PIP y PyMySQL mediante pip.3.10.
- Inicia y habilita el servicio mariadb
- Habilita puerto de Ubuntu mediante el servicio UFW, el cual es 3306/tcp.
- Cambia la configuración en 50-server.conf para setear bind-address = 0.0.0.0, es decir, que permita las conexiones de red a la base de datos en general.
- Por último, si nada falló, reinicia el servicio mariadb para cargar los cambios.

###### MYSQL Secure Installation

- En priemra instancia actualiza la contraseña del usuario root, que está contenida encriptadamente en el fichero vault dbpwdsecure.yml. y que es citada mediante la variable rootpass_ws en el playbook.
- Se guarda el usuario root con los distintos hosts posibles definidos en el loop, es decir; 

          - 192.168.56.%
          - 127.0.0.1
          - 192.168.56.104

- Elimina el usuario anónimo registrado.

- Dado a que existe una base de datos llamada test por default, esta es eliminada.

- Finalmente el servicio MariaDB es reiniciado mediante un handler.
