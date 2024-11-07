# Para correr la maquina

Levantar la maquina servidorWeb

```
vagrant up
vagrant ssh servidorWeb
```

## Para correr la aplicacion web

```
cd /home/vagrant/webapp
export FLASK_APP=run.py
/usr/local/bin/flask run --host=0.0.0.0
```

## Instalacion y configuracion de docker

````
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done

sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.asc
````

Agregar el repositorio a apt sources:

````
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $USER
````

Verificar que se instalo docker:

`````
sudo systemctl status docker
`````

Descargar imagen de apache hhtpd:

``````
sudo docker search apache
docker pull httpd
``````

Configuracion para docker compose:

``````
vim ~/.vimrc

``````
En este archivo se agrega lo siguiente:

``````
" Configuracion para trabajar con archivos yaml
au! BufNewFile,BufReadPost *.{yaml,yml} set filetype=yaml foldmethod=indent
autocmd FileType yaml setlocal ts=2 sts=2 sw=2 expandtab
``````

Crear directorio de trabajo con Dockerfile y docker-compose

A la misma altura del directorio webapp, ejecutar los siguientes comandos para crear los archivos docker:

``````
vagrant@servidorWeb:~$ pwd
/home/vagrant
sudo touch Dockerfile
sudo touch docker-compose
``````


## Creacion de autofirmado de SSL

``````
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout localhost.key -out localhost.crt -subj "/C=Col/ST=Valle del cauca/L=Cali/O=UAO/CN=localhost"
``````

NOTA: Mover los archivos de certificado creados (localhost.crt y localhost.key) al mismo directorio del Dockerfile

Crear archivo de configuracion de host virtual en el mismo directorio del Dockerfile llamado "my-httpd-vhosts.conf"

``````
my-httpd-vhosts.conf
``````
Debe contener lo siguiente

``````
WSGIScriptAlias / /var/www/webapp/application.wsgi
DocumentRoot /var/www/webapp

<VirtualHost *:80>
    # Sin ServerName, se usará la IP
    Redirect permanent / https://192.168.60.3:443/
</VirtualHost>

<VirtualHost *:443>
    <Directory /var/www/webapp/>
    Order deny,allow
    Allow from all
    </Directory>
    SSLEngine on
    SSLCertificateFile "/usr/local/apache2/conf/ssl/localhost.crt"
    SSLCertificateKeyFile "/usr/local/apache2/conf/ssl/localhost.key"
</VirtualHost>

``````

En la carpeta webapp, crear el archivo application.wsgi:

``````
#!/usr/bin/python
import sys
sys.path.insert(0,"/var/www/webapp/")
from web.views import app as application
``````

Dockerfile:

``````
# Usa la imagen oficial de Python como base
FROM python:3.11-slim

# Instala Apache y las herramientas necesarias para compilar mod_wsgi
RUN apt-get update && \
    apt-get install -y apache2 build-essential python3-dev apache2-dev pkg-config libmariadb-dev libapache2-mod-wsgi-py3 && \
    apt-get clean

# Instala mod_wsgi desde PyPI, que es compatible con Python 3
RUN pip install mod_wsgi

# Crea un entorno virtual en el contenedor
RUN python3 -m venv /opt/venv

# Asegura que los comandos se ejecuten dentro del entorno virtual
ENV PATH="/opt/venv/bin:$PATH"

# Copia tu aplicación al contenedor
COPY ./webapp /var/www/webapp

# Copia el archivo de configuración de Apache SSL
COPY ./my-httpd-vhosts.conf /etc/apache2/sites-available/my-ssl.conf

# Habilita el módulo SSL y el sitio
RUN a2enmod ssl && \
    a2enmod proxy && \
    a2enmod proxy_http && \
    a2ensite my-ssl.conf

# Instala las dependencias de Python dentro del entorno virtual
COPY requirements.txt /tmp/requirements.txt
RUN pip install --no-cache-dir -r /tmp/requirements.txt

# Copia los certificados SSL
COPY localhost.crt /usr/local/apache2/conf/ssl/localhost.crt
COPY localhost.key /usr/local/apache2/conf/ssl/localhost.key

# Expone el puerto en el que Apache escucha
EXPOSE 443

# Inicia Apache cuando el contenedor arranque
CMD ["apache2ctl", "-D", "FOREGROUND"]

``````
El archivo docker-compose.yml tiene lo siguiente:

``````
version: '3.8'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: webapp_container
    ports:
      - "8443:443"
    depends_on:
      - db
    environment:
      - MYSQL_HOST=db
      - MYSQL_USER=root
      - MYSQL_PASSWORD=root
      - MYSQL_DB=myflaskapp

  db:
    image: mysql:8.0
    container_name: mysql_container
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: myflaskapp
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
      - mysql_data:/var/lib/mysql  # Añade un volumen para persistir los datos de MySQL
    ports:
      - "3307:3306"

volumes:
  mysql_data:  # Define un volumen persistente para MySQL
``````

Tambien crear en el mismo directorio del Dockerfile, un archivo requirements.txt:

``````
sudo vim requirements.txt
``````

Debe tener lo siguiente:

``````
Flask==2.3.3
flask-cors
Flask-MySQLdb
Flask-SQLAlchemy
``````

El arbol de directorios debe estar asi:

``````
vagrant@servidorWeb:~$ tree
.
├── docker-compose.yml
├── Dockerfile
├── init.sql
├── localhost.crt
├── localhost.key
├── my-httpd-vhosts.conf
├── requirements.txt
└── webapp
    ├── application.wsgi
    ├── config.py
    ├── __pycache__
    │   ├── config.cpython-310.pyc
    │   └── run.cpython-310.pyc
    ├── run.py
    ├── users
    │   ├── controllers
    │   │   ├── __pycache__
    │   │   │   └── user_controller.cpython-310.pyc
    │   │   └── user_controller.py
    │   └── models
    │       ├── db.py
    │       ├── __pycache__
    │       │   ├── db.cpython-310.pyc
    │       │   └── user_model.cpython-310.pyc
    │       └── user_model.py
    └── web
        ├── __pycache__
        │   └── views.cpython-310.pyc
        ├── static
        │   └── script.js
        ├── templates
        │   ├── edit.html
        │   └── index.html
        └── views.py
``````

Construir imagen a partir del Dockerfile:

``````
docker compose build
``````
Levantar los contenedores de los servicios descritos en el docker-compose.yml:

``````
docker compose up
``````
Para ver la imagen construida:

``````
docker images
``````

Loguearse en docker:

``````
sudo docker login
``````

Crear un repositorio en Docker hub para el siguiente paso

Pushear la imagen a Docker hub:

Intercambiar tag local-image con el nombre de la imagen que se construyo con el comando docker compose build, y new-repo con el nombre del repositorio de Docker Hub

``````
docker tag local-image:tagname new-repo:tagname
docker push new-repo:tagname
``````