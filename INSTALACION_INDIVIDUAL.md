Sitios web de los integrantes del equipo:
================================
Jhonatan Acevedo: [**www.jhonargullo.ml**](https://www.jhonargullo.ml/)  <br>
Andrew Pérez: [**www.andruorgullo.ml**](https://www.andruorgullo.ml/) <br>
Juan Escudero: [**www.junajorgullo.ml**](https://www.juanjorgullo.ml/)  <br>


Documentación de la Instalación
================================

Requisitos
================================
> - Para la instalación se utilizaron los siguientes servicios de [**AWS**](https://console.aws.amazon.com/):
>   + EC2
>   + Elastic IP
> - Para obtener el dominio se utilizó [**Freenom**](https://www.freenom.com/).
> - Para obtener el certificado se utilizó [**Let's Encrypt**](https://letsencrypt.org/) y [**Certbot**](https://certbot.eff.org/).


<br> <br>

Configuración en Freenom
================================
Se accede al panel de freenom para configurar sus nameservers y que redirijan al dominio solicitado <br>
![image](![xd](https://user-images.githubusercontent.com/47001381/119211350-2e901080-ba77-11eb-86dc-c160b36bdb25.png)




Utilizando el servicio EC2
================================
Con este servicio se crean la instancias necesaria para el servidor web donde se alojarán todas las dependencias para el sistema.

Creación del Web Server
--------------------------------
Para la creación del Web Server, en la sección `Instances` se hizo clic en `Launch instances` y se configuró de la siguiente manera:
  * Como **AMI** se utilizó `Ubuntu Server 20.04 LTS (HVM)`.

Configuración del Web Server
--------------------------------
Luego de conectarnos con `SSH` empezamos con la configuración. <br>

Instalación de Docker Engine y Docker Compose
--------------------------------
Para instalar estas herramientas se hicieron los pasos especificados en los siguiente sitios web:

[**Docker-Engine**](https://docs.docker.com/engine/install/ubuntu/) <br>
[**Docker-Compose**](https://docs.docker.com/compose/install/)


Configuración de Docker Compose
--------------------------------

Luego creamos el archivo docker-compose en `/home/ubuntu/wordpress/docker-compose.yml` con el siguiente contenido:
```
version: '3'

services:
  db:
    image: mysql:8.0
    container_name: db
    restart: unless-stopped
    env_file: .env
    environment:
      - MYSQL_DATABASE=wordpress
    volumes:
      - dbdata:/var/lib/mysql
    command: '--default-authentication-plugin=mysql_native_password'
    networks:
      - app-network

  wordpress:
    depends_on:
      - db
    image: wordpress:5.1.1-fpm-alpine
    container_name: wordpress
    restart: unless-stopped
    env_file: .env
    environment:
      - WORDPRESS_DB_HOST=db:3306
      - WORDPRESS_DB_USER=$MYSQL_USER
      - WORDPRESS_DB_PASSWORD=$MYSQL_PASSWORD
      - WORDPRESS_DB_NAME=wordpress
    volumes:
      - wordpress:/var/www/html
    networks:
      - app-network

  webserver:
    depends_on:
      - wordpress
    image: nginx:1.15.12-alpine
    container_name: webserver
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - wordpress:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
      - certbot-etc:/etc/letsencrypt
    networks:
      - app-network

  certbot:
    depends_on:
      - webserver
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot-etc:/etc/letsencrypt
      - wordpress:/var/www/html
    command: certonly --webroot --webroot-path=/var/www/html --email jsacevedoc@jhonargullo.ml --agree-tos --no-eff-email --staging -d jhonargullo.ml -d www.jhonargullo.ml

volumes:
  certbot-etc:
  wordpress:
  dbdata:

networks:
  app-network:
    driver: bridge
```
Este contenido nos va a componer e integrar contenedores con los servicios de MySQL como base de datos, Wordpress como CMS, Nginx como servidor web y Cerbot para generación de certificados con Let´s Encrypt


Creación del .env para la base de datos
--------------------------
Para que el contenedor con la imagen de MySQL funcione correctamente, se requiere especificar los parámetros de inicio en un archivo .env de la siguiente manera:

```
MYSQL_ROOT_PASSWORD=user
MYSQL_USER=admin
MYSQL_PASSWORD=teamorgullo123*
```


Configuración de Nginx
--------------------------
Luego de solicitar el certificado de `Let's encrypt` procedemos a configurar el servidor Nginx con las rutas, los puertos y los dominios en cuestión.

```
server {
        listen 80;
        listen [::]:80;

        server_name jhonargullo.ml www.jhonargullo.ml;

        index index.php index.html index.htm;

        root /var/www/html;

        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }

        location / {
                try_files $uri $uri/ /index.php$is_args$args;
        }

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass wordpress:9000;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param PATH_INFO $fastcgi_path_info;
        }

        location ~ /\.ht {
                deny all;
        }

        location = /favicon.ico {
                log_not_found off; access_log off;
        }
        location = /robots.txt {
                log_not_found off; access_log off; allow all;
        }
        location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
                expires max;
                log_not_found off;
        }


}
```

Composición y montaje de imágenes
--------------------------

El `.yml` creado permite iniciar el Wordpress, la base de datos, el Nginx y obtener el certificado de `Let's encrypt` para nuestro dominio utilizando `Certbot`. Para esto, ejecutamos el siguiente comando:
```
sudo docker-compose up
```

<br> <br>
Configuración del Nginx con el certificado
================================
Una vez se obtiene el certificado y se ponen en ejecución los servicios requeridos se modifica el archivo anterior de configuración de Nginx para que funcione con el certificado especificando la ruta de la `.pem` creada anteriormente. 
```
server {
        listen 80;
        listen [::]:80;

        server_name jhonargullo.ml www.jhonargullo.ml;

        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }

        location / {
                rewrite ^ https://$host$request_uri? permanent;
        }
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name jhonargullo.ml www.jhonargullo.ml;

        index index.php index.html index.htm;

        root /var/www/html;

        server_tokens off;

        ssl_certificate /etc/letsencrypt/live/jhonargullo.ml/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/jhonargullo.ml/privkey.pem;

        include /etc/nginx/conf.d/options-ssl-nginx.conf;

        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Referrer-Policy "no-referrer-when-downgrade" always;

        location / {
                try_files $uri $uri/ /index.php$is_args$args;
        }

        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass wordpress:9000;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param PATH_INFO $fastcgi_path_info;
        }

        location ~ /\.ht {
                deny all;
        }

        location = /favicon.ico {
                log_not_found off; access_log off;
        }
        location = /robots.txt {
                log_not_found off; access_log off; allow all;
        }
        location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
                expires max;
                log_not_found off;
        }


}
```

Adición de archivo para configuración de Nginx con SSL
--------------------------
Para descargar el archivo adicional de configuración necesario para que Nginx trabaje correctamente con el certificado generado se hace el siguiente comando el la ruta donde se encuentra la carpeta de configuración del Nginx.

```
sudo curl -sSLo options-ssl-nginx.conf https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf![image](https://user-images.githubusercontent.com/47001381/119210886-7f523a00-ba74-11eb-8aca-f459d2b6246e.png)
```
