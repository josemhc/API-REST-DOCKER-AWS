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

