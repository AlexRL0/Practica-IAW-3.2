# Practica-IAW-3.2- Implantación de Moodle en Amazon Web Services (AWS) mediante Ansible

## Pasos previos a la instalación

**para la instalación vamos a tener que crear la siguiente estructura de directorios y archivos:**

![](imagenes/directorio-3.1.png)

**Una vez creados los archivos y directorios, procedemos a la instalación de las herramientas.**

## Implantación de Moodle en Amazon Web Services (AWS) mediante Ansible

## 1. Creamos un nombre de dominio para nuestro servidor con la web `no-ip`

### Paso 1. Crear el nombre del dominio 

Para crear el nombre de dominio en la página, tenemos que registrarnos y dirigirnos al sitio que muestra la captura y darle a **crear nombre de host**.

![](imagenes/no-ip-3.1.png)

Una vez dado al recuadro de crear el nombre, nos sale un asistente para poner el nombre de host que queramos, el dominio que queremos que lleve, la dirección ip de la máquina o pc que queremos que tenga ese nombre de host y el tipo de registro que queremos que lleve, en nuestro caso es el **DNS Host A**

Hecho esto ya podemos automatizar la Implantación de Moodle en Amazon Web Services (AWS) mediante Ansible

## 1.1 Infraestructura necesaria

La infraestructura necesaria está compuesta de tres instancias EC2, un nodo de control para ejecutar los playbooks de ansible al resto de máquinas, un frontal donde se desplegará moodle (Frontend) y un backend donde tendremos la base de datos para moodle y el usuario (Back-End).

* Nodo de control.
* Frontal con el moodle desplegado.
* Servidor de Base de Datos MySQL. (Backend)

## 1.2 Aspectos importantes a tener en cuenta

Para poder utilizar ansible debemos instalarlo con los siguientes comandos:

```bash
sudo apt update
sudo apt install ansible -y
```

Para que el nodo de control pueda ver las otras máquinas sin problemas, hay que crear una carpeta llamada `inventory` y dentro de ella un archivo llamado `inventory` con el siguiente contenido:

```yaml
[frontend]
18.215.192.38

[backend]
50.17.188.169

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=/home/ubuntu/Practica-IAW-3.1/vockey.pem
ansible_ssh_common_args='-o StrictHostKeyChecking=accept-new'
```

> [!IMPORTANT]  
> `[frontend]:` Especifica el grupo de hosts que son parte del front-end de la infraestructura. Aquí, 18.215.192.38 es la IP del servidor frontend.  
> `[backend]:` Especifica el grupo de hosts que son parte del back-end de la infraestructura. Aquí, 50.17.188.169 es la IP del servidor backend.  
> `[all:vars]:` Define variables globales que se aplican a todos los hosts.  
> `ansible_user=ubuntu:` El nombre de usuario que se usará para conectarse a los hosts.  
> `ansible_ssh_private_key_file=/home/ubuntu/Practica-IAW-3.1/vockey.pem:` Ruta al archivo de clave privada SSH para autenticarse en los hosts.  
> `ansible_ssh_common_args='-o StrictHostKeyChecking=accept-new':` Argumentos adicionales de SSH, en este caso, para aceptar automáticamente nuevas claves de host sin preguntar.

También es importante darle los permisos necesarios a la clave `vockey.pem` para que pueda leer el archivo. Eso se puede hacer mediante chmod de la siguiente manera:

```bash
sudo chmod 400 vockey.pem
```

Para ejecutar los playbooks de ansible se hace de la siguiente forma:

```bash
sudo ansible-playbooks -i /ruta/al/inventario /ruta/al/archivo/yaml
```

## 2. Creación del archivo `install_lamp_frontend` para la instalación de apache en el frontend.

### Paso 1. Actualizamos los repositorios

```yaml
---
- name: Playbook para instalar la pila LAMP en el FrontEnd
  hosts: frontend
  become: yes

  tasks:

    - name: Actualizar los repositorios
      apt:
        update_cache: yes
```

### Paso 2. Instalamos el servidor web Apache

```yaml
    - name: Instalar el servidor web Apache
      apt:
        name: apache2
        state: present
```

### Paso 3. Instalamos PHP y los módulos necesarios

```yaml
    - name: Instalar PHP y los módulos necesarios
      apt: 
        name:
          - php
          - php-mysql
          - libapache2-mod-php
          - php-bcmath
          - php-curl
          - php-gd
          - php-imagick
          - php-intl
          - php-memcached
          - php-mbstring
          - php-dom
          - php-zip
          - php-cli
          - php-xml
          - php-soap
        state: present
```

### Paso 4. Modificamos la configuración de PHP

```yaml
    - name: Modificamos el valor max_input_vars de PHP
      replace: 
        path: /etc/php/8.3/apache2/php.ini
        regexp: ;max_input_vars = 1000
        replace: max_input_vars = 5000

    - name: Modificamos el valor de memory_limit de PHP
      replace: 
        path: /etc/php/8.3/apache2/php.ini
        regexp: memory_limit = 128M
        replace: memory_limit = 256M

    - name: Modificamos el valor de post_max_size de PHP
      replace: 
        path: /etc/php/8.3/apache2/php.ini
        regexp: post_max_size = 8M
        replace: post_max_size = 128M

    - name: Modificamos el valor de upload_max_filesize de PHP
      replace:
        path: /etc/php/8.3/apache2/php.ini
        regexp: upload_max_filesize = 2M
        replace: upload_max_filesize = 128M
```

### Paso 5. Configuramos Apache y creamos los directorios necesarios

```yaml
    - name: Copiar el archivo de configuración de Apache
      copy:
        src: ../templates/000-default.conf
        dest: /etc/apache2/sites-available/
        mode: 0755

    - name: Crear directorio /var/www/html/moodle
      file:
        path: /var/www/html/moodle
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'

    - name: Crear directorio /var/www/moodledata
      file:
        path: /var/www/moodledata
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'
```

### Paso 6. Habilitamos el módulo rewrite de Apache y reiniciamos Apache

```yaml
    - name: Habilitar el módulo rewrite de Apache
      apache2_module:
        name: rewrite
        state: present

    - name: Reiniciar el servidor web Apache
      service:
        name: apache2
        state: restarted
```

## 2. Creación del archivo `install_lamp_backend` para la instalación de mysql y creacion de la base de datos y el usuario en el backend

### Paso 1. Incluimos el archivo de variables

```yaml
---
- name: Playbook para instalar la pila LAMP en el Backend
  hosts: backend
  become: yes

  vars_files:
    - ../vars/variables.yml
```

### Paso 2. Actualizamos los repositorios

```yaml
  tasks:

    - name: Actualizar los repositorios
      apt:
        update_cache: yes
```

### Paso 3. Instalamos el sistema gestor de bases de datos MySQL

```yaml
    - name: Instalar el sistema gestor de bases de datos MySQL
      apt:
        name: mysql-server
        state: present
```

### Paso 4. Instalamos el modulo de pymysql

```yaml
    - name: Instalamos el módulo de pymysql
      apt:
        name: python3-pymysql
        state: present
```

### Paso 5. Creamos la base de datos para moodle

```yaml
    - name: Crear la base de datos para moodle
      mysql_db:
        name: "{{ moodle.db_name }}"
        encoding: utf8mb4
        collation: utf8mb4_unicode_ci
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock
```

> [!IMPORTANT]  
>Las variables que irán apareciendo durante los pasos están todas definidas en el archivo variables.yml y en variables_example.yml se pueden ver las variables que hay que tener definidas.

### Paso 6. Borramos el usuario para la base de datos de moodle si existe

```yaml
    - name: Borra el usuario si existe
      mysql_user:
        name: "{{ moodle.db_user }}"
        host: "{{ moodle.ip_cliente_mysql }}"
        state: absent
        login_unix_socket: /var/run/mysqld/mysqld.sock
```

### Paso 7. Creamos el usuario para la base de datos de moodle

```yaml
    - name: Crear el usuario para la base de datos de moodle
      mysql_user:
        name: "{{ moodle.db_user }}"
        host: "{{ moodle.ip_cliente_mysql }}"
        password: "{{ moodle.db_password }}"
        priv: "{{ moodle.db_name }}.*:ALL"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock
```

## Paso 8. Cambiamos el bind-address en el archivo de configuración de MySQL

```yaml
    - name: Cambiar bind-address en mysqld.cnf
      replace:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: '^bind-address.*'
        replace: 'bind-address = "{{ moodle.ip_cliente_mysql }}"'
```

### Paso 9. Reiniciamos los privilegios del usuario y el servicio MySQL

```yaml
    - name: Reiniciamos privilegios del usuario
      mysql_db:
        name: "{{ moodle.db_name }}"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock

    - name: Reiniciamos el servicio de base de datos
      service:
        name: mysql
        state: restarted
```

## 3. Creación del archivo `setup_letsencrypt_https` para la implementación del certificado de la página web

### Paso 1. Incluimos el archivo de variables

```yaml
---
- name: Playbook para configurar HTTPS
  hosts: frontend
  become: yes

  vars_files:
    - ../vars/variables.yml
```

### Paso 2. Desinstalamos instalaciones previas de Certbot

```yaml
  tasks:

    - name: Desinstalar instalaciones previas de Certbot
      apt:
        name: certbot
        state: absent
```

### Paso 3. Instalamos Certbot con snap

```yaml
    - name: Instalar Certbot con snap
      snap:
        name: certbot
        classic: yes
        state: present
```

### Paso 4. Solicitamos y configuramos el certificado SSL/TLS a Let's Encrypt con Certbot

```yaml
    - name: Solicitar y configurar certificado SSL/TLS a Let's Encrypt con certbot
      command:
        certbot --apache \
        -m {{ certbot.le_email }} \
        --agree-tos \
        --no-eff-email \
        --non-interactive \
        -d {{ certbot.le_domain }}
```

## 4. Creación del archivo `deploy` para la implementación del certificado de la página web

### Paso 1. Incluimos el archivo de variables

```yaml
---
- name: Playbook para configurar HTTPS
  hosts: frontend
  become: yes

  vars_files:
    - ../vars/variables.yml
```

### Paso 2. Limpiamos descargar previos y directorios para que no haya duplicados

```yaml
  tasks:
    - name: Limpiar descargas previas de Moodle
      file:
        path: /tmp/moodle*
        state: absent
      ignore_errors: true

    - name: Limpiar directorio /var/www/html
      file:
        path: /var/www/html/*
        state: absent
      ignore_errors: true

    - name: Limpiar directorio /var/www/moodledata
      file:
        path: /var/www/moodledata/*
        state: absent
      ignore_errors: true
```

### Paso 3. Descargamos el código fuente de moodle

```yaml
    - name: Descargar el código fuente de Moodle
      get_url:
        url: https://download.moodle.org/download.php/direct/stable405/moodle-latest-405.tgz
        dest: /tmp/moodle-latest-405.tgz
```

### Paso 4. Extraemos moodle y lo movemos al directorio deseado

```yaml
    - name: Extraer Moodle
      unarchive:
        src: /tmp/moodle-latest-405.tgz
        dest: /tmp
        remote_src: yes

    - name: Mover Moodle al directorio deseado
      command:
        cmd: mv /tmp/moodle {{ moodle.directory }}
```

### Paso 5. Configuramos permisos para los directorios moodle y moodledata

```yaml
    - name: Configurar permisos para el directorio moodle
      file:
        path: "{{ moodle.directory }}"
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'
        recurse: yes

    - name: Configurar permisos para el directorio moodledata
      file:
        path: "{{ moodle.data_directory }}"
        state: directory
        owner: www-data
        group: www-data
        mode: '0777'
        recurse: yes
```

### Paso 6. Hacemos una copia del archivo config-dist.php

```yaml
    - name: Copiar archivo config-dist.php
      copy:
        src: "{{ moodle.directory }}/config-dist.php"
        dest: "{{ moodle.directory }}/config.php"
        remote_src: yes
```

### Paso 7. Ejecutamos la instalación de Moodle

```yaml
    - name: Ejecutar instalación de Moodle
      command: >
        sudo -u www-data php /var/www/html/moodle/admin/cli/install.php
        --lang=es
        --wwwroot=https://{{moodle.domain}}
        --dataroot={{moodle.data_directory}}
        --dbtype=mysqli
        --dbhost={{moodle.db_host}}
        --dbname={{moodle.db_name}}
        --dbuser={{moodle.db_user}}
        --dbpass={{moodle.db_password}}
        --fullname="Moodle"
        --shortname="ARL"
        --summary="celia"
        --adminuser=admin
        --adminpass="Admin01%"
        --adminemail="admin@gmail.com"
        --non-interactive
        --agree-license
```

### Paso 8. Reiniciamos Apache

```yaml
    - name: Reiniciar apache
      service:
        name: apache2
        state: restarted
```

## Comprobación de que Moodle funciona

![](imagenes/moodle-3.1.png)