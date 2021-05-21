Documentación de la Instalación
================================
Arquitectura desplegada
--------------------------------
![Diagrama AWS](https://user-images.githubusercontent.com/47001432/119178036-ca4b5d80-ba32-11eb-9926-790178abdcde.png)
<br>
<br>

Requisitos
================================
> - Para la instalación se utilizaron los siguientes servicios de [**AWS**](https://console.aws.amazon.com/):
>   + VPC
>   + EFS
>   + RDS
>   + EC2
> - Como DNS se utilizó [**CloudFlare**](https://www.cloudflare.com/).
> - Para obtener el dominio se utilizó [**Freenom**](https://www.freenom.com/).
> - Para obtener el certificado se utilizó [**Let's Encrypt**](https://letsencrypt.org/) y [**Certbot**](https://certbot.eff.org/).



<br> <br>
Utilizando el servicio VPC
================================
Creación de la VPC
--------------------------------
+ En la sección `VPC` se hizo clic en `Create VPC` y se puso la siguiente configuración:
  > **Name:** Orgullo-VPC <br>
  > **IPv4 CIDR:** 172.16.0.0


Creación de las subredes
--------------------------------
* En la sección `Subnets` se crearon 4 subredes con la siguiente configuración: <br>
* Para las subredes privadas:
  > + Private subnet A: 172.16.1.0/24
  > + Private subnet B: 172.16.3.0/24

* Para las subredes públicas:
  > - Public subnet A: 172.16.2.0/24
  > - Public subnet B: 172.16.4.0/24


Creación de la Internet Gateway
---------------------------------
* En la sección `Internet Gateways` se creó una nueva y se vinculó con nuestra **VPC** creada anteriormente.

Creación de las route tables
--------------------------------
En la sección `Route tables` se creó una tabla por cada subred: <br>
* Luego de crear las tablas de enrutamiento de las subredes privadas se configuró en la sección `Routes` lo siguiente:
  > **Destination:** 0.0.0.0/0 <br>
  > **Target:** La **NAT Instance** de la subred pública de la zona de disponibilidad concreta. 
* En el caso de las tablas de subredes públicas:
  > **Destination:** 0.0.0.0/0 <br>
  > **Target:** La **Internet Gateway** creada anteriormente.


Creación de los grupos de seguridad
---------------------------------
En la sección `Security Groups` se crearon 4 grupos de seguridad vinculados con nuestra **VPC** con las siguientes configuraciones de Inbound Rules:
* SG-Bastion
  > Type | Protocol | Port range | Source
  > ------------ | ------------- | ------------ | ------------
  > SSH | TCP | 22 | Custom
  > > En `Source` se configuraron las IPs de los miembros del equipo, los cuales están autorizados a manipular la VPC.
* SG-WEB
  > Type | Protocol | Port range | Source
  > ------------ | ------------- | ------------ | ------------
  > HTTP | TCP | 80 | Anywhere
  > HTTPS | TCP | 443 | Anywhere
  > NFS | TCP | 2049 | Anywhere
  > SSH | TCP | 22 | SG-Bastion
  > > Para `SSH` se configuró el `source` como `SG-Bastion` para permitir las conexiones SSH solo si provienen desde el **Bastion Host** de la zona de disponibilidad.
* SG-NATInstance
  > Type | Protocol | Port range | Source
  > ------------ | ------------- | ------------ | ------------
  > HTTP | TCP | 80 | 172.16.1.0/24
  > HTTP | TCP | 80 | 172.16.3.0/24
  > HTTPS | TCP | 443 | 172.16.1.0/24
  > HTTPS | TCP | 443 | 172.16.3.0/24
  > SSH | TCP | 22 | SG-Bastion
  > > Para `HTTP` y `HTTPS` se configuró el `source` como cada subred privada para permitir la conexión de las instancias que estén allí.
* SG-RDS-DB
  > Type | Protocol | Port range | Source
  > ------------ | ------------- | ------------ | ------------
  > MySQL/Aurora | TCP | 3306 | SG-WEB
  > > Para `MySQL/Aurora` se configuró el `source` como `SG-WEB` para permitir las conexiones de los servidores web con la base de datos.



<br> <br>
Utilizando el servicio EFS
================================
Con este servicio se crea el Elastic File System para tener alamacenamiento compartido entre las instancias del web server. <br>

Creación del EFS
--------------------------------
* En la sección `File systems` se hizo clic en `Create file system` con la siguiente configuración:
  > **Name:** Orgullo-WP-EFS <br>
  > **VPC:** Orgullo-VPC (La **VPC** creada anteriormente) <br>
  > **Availability and Durability:** Regional <br>
* En `Availability and Durability` se configura de forma regional para compartir el almacenamiento entre varias zonas de disponibilidad.


Configuración de los mount target
--------------------------------
* Una vez creado el EFS, entramos a su configuración y en la sección `Network` configuramos los siguientes puntos de montaje:
  > Availability zone | Subnet ID | Security groups
  > ------------ | ------------- | ------------
  > Availability zone A | Private subnet A ID | SG-WEB
  > Availability zone B | Private subnet B ID | SG-WEB
* Una vez se hicieron estas configuraciones, guardamos las IP generadas automáticamente para luego montarlas en el Web Server.



<br> <br>
Utilizando el servicio RDS
================================
Con este servicio se crea la base de datos que va a utilizar el Web Server. <br>

Creación y configuración de la base de datos
--------------------------------
* En la sección `Databases` se hizo clic en `Create database` con la siguiente configuración:
  > - **Database creation method:** Standard 
  > - **Motor:** MySQL 
  > - **Template:** Free tier 
  > - **Settings:**
  >   + Aquí configuramos nuestras credenciales. 
  > - **Storage:** 
  >   + **Type:** SSD 
  >   + **Allocated storage:** 20 GB
  > - **Connectivity:**
  >   + **VPC:** Orgullo-VPC
  >   + **Public access:** No 
  >   + **Security group:** SG-RDS-DB
* Aunque se asignaron 20 GB la base de datos tiene almacenamiento escalable, de esta forma el almacenamiento aumenta a medida que el Web Server lo necesita.



<br> <br>
Utilizando el servicio EC2
================================
Con este servicio se crean las diferentes instancias necesarias, como lo son los `Bastion Host`, las `NAT Instance` y el `Web Server`. Además, se crea el `Load Balancer` con su grupo de destino y el `Auto Scaling Group` con su configuración de lanzamiento. <br>

Creación de los Bastion Host
--------------------------------
Para la creación de cada Bastion Host, en la sección `Instances` se hizo clic en `Launch instances` y se configuró cada uno de la siguiente manera:
  * Como **AMI** se utilizó `Amazon Linux 2`.
  * En los detalles de la instancia se configuraron los siguientes parámetros:
    - **Network:** Orgullo-VPC
    - **Subnet:** Public Subnet A / Public Subnet B (Poniendo la necesaria dependiendo del Bastion Host)
    - **Auto-assign public IP:** Enable (Dado que se encuentra en una subred pública)
  * En los grupos de seguridad se configuró `SG-Bastion`
 
Creación de las NAT Instance
--------------------------------
Para la creación de cada NAT Instance, en la sección `Instances` se hizo clic en `Launch instances` y se configuró cada una de la siguiente manera:
  * Como **AMI** se utilizó `Amazon Linux AMI 2018.03.0.20181116 x86_64 VPC HVM ebs` (Una AMI con todos los elementos necesarios para que la instancia funcione como NAT Instance).
  * En los detalles de la instancia se configuraron los siguientes parámetros:
    - **Network:** Orgullo-VPC
    - **Subnet:** Public Subnet A / Public Subnet B (Poniendo la necesaria dependiendo de la NAT Instance)
    - **Auto-assign public IP:** Enable
  * En los grupos de seguridad se configuró `SG-NAT-Instance`
  * Una vez creada la instancia se deshabilitó el `source/destination checking` para permitir que la instancia reciba y envíe tráfico sin problemas.

Creación del Web Server
--------------------------------
Para la creación del Web Server, en la sección `Instances` se hizo clic en `Launch instances` y se configuró de la siguiente manera:
  * Como **AMI** se utilizó `Amazon Linux 2`.
  * En los detalles de la instancia se configuraron los siguientes parámetros:
    - **Network:** Orgullo-VPC
    - **Subnet:** Private Subnet A
    - **Auto-assign public IP:** Disable (Dado que se encuentra en una subred privada)
  * En los grupos de seguridad se configuró `SG-WEB`

Configuración del Web Server
--------------------------------
Luego de conectarnos haciendo `SSH` a través del `Bastion Host A` empezamos con la configuración. <br>
Lo primero es instalar las herramientas necesarias para hacer el montaje del EFS y crear los directorios donde se va a montar: <br>
```
sudo yum install -y amazon-efs-utils
sudo mkdir -p /mnt/efs/wordpress
```

Lo siguiente es agregar los puntos de montaje obtenidos anteriormente en `/ect/fstab` para que se monten automáticamente cada vez que la instancia inicia, cambiando `<IP-ADDRESS-AZ-X>` por el punto de montaje de la zona de disponibilidad concreta:
```
<IP-ADDRESS-AZ-A>:/ /mnt/efs/wordpress nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport,_netdev 0 0
<IP-ADDRESS-AZ-B>:/ /mnt/efs/wordpress nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport,_netdev 0 0
```

La AMI `Amazon Linux 2` tiene un problema con el montaje automático de múltiples puntos, por lo cual se crea el siguiente archivo en `/etc/systemd/system/mount-nfs-sequentially.service`:
```
[Unit]
Description=Workaround for mounting NFS file systems sequentially at boot time
After=remote-fs.target

[Service]
Type=oneshot
ExecStart=/bin/mount -avt nfs4
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Luego de esto se ejecutan los siguientes comandos para utilizar el script anterior y reiniciar la instancia para que queden los puntos de montaje establecidos: <br>
```
sudo systemctl daemon-reload
sudo systemctl enable mount-nfs-sequentially.service
echo "<IP-ADDRESS-AZ-A> fs-b965cf0d.efs.us-east-1.amazonaws.com" | sudo tee -a /etc/hosts
echo "<IP-ADDRESS-AZ-B> fs-b965cf0d.efs.us-east-1.amazonaws.com" | sudo tee -a /etc/hosts
sudo reboot
```

Una vez montado el EFS instalamos e hicimos las configuraciones necesarias para utilizar docker y docker-compose: <br>
```
sudo amazon-linux-extras install docker -y
sudo yum install git -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -a -G docker ec2-user
sudo curl -L https://github.com/docker/compose/releases/download/1.27.4/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
newgrp docker
```

Luego creamos el archivo docker-compose en `/home/ec2-user/docker-compose.yml` con el siguiente contenido:
```
version: '3.1'
services:
  wordpress:
    image: wordpress
    container_name: orgullo_wordpress
    restart: always
    ports:
      - 443:443
      - 80:80
    environment:
      WORDPRESS_DB_HOST: <RDS-ENDPOINT>
      WORDPRESS_DB_USER: <DB-USER>
      WORDPRESS_DB_PASSWORD: <DB-PASSWORD>
      WORDPRESS_DB_NAME: <DB-NAME>
    volumes:
      - /mnt/efs/wordpress:/var/www/html
      
  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - /etc/letsencrypt/:/etc/letsencrypt
      - /mnt/efs/wordpress:/var/www/html
    command: certonly --webroot --webroot-path=/var/www/html --email <MAIL> --agree-tos --no-eff-email --staging -d <DOMINIO> -d www.<DOMINIO>
```

El `.yml` creado permite iniciar el wordpress y obtener el certificado de `Let's encrypt` para nuestro dominio utilizando `Certbot`. Para esto, ejecutamos el siguiente comando:
```
docker-compose up -d
```

Una vez se ejecuta el `.yml` guardamos los archivos `.pem` que quedaron almacenados en `/etc/letsencrypt` y la instalación y montaje del Web Server terminó.

Creación de la AMI de nuestro Web Server
-------------------------------
Una vez el Web Server está montado se selecciona la instancia y haciendo clic en `Actions -> Image and templates -> Create image` creamos una AMI de nuestro Web Server para ser utilizado posteriormente en el `Auto Scaling Group`. <br>

Creación del balanceador de carga
-------------------------------
Para la creación del balanceador de carga accedemos a la sección `Load Balancers` y hacemos clic en `Create Load Balancer` para configurarlo de la siguiente manera: <br>
  * Utilizamos `Application Load Balancer` para nuestro despliegue.
  * **Basic configuration:**
    - **Name:** Orgullo-ELB
    - **Scheme:** Internet-facing
    - **IP address type:** IPv4
  * **Listeners:**
    - HTTP en puerto 80
    - HTTPS en puerto 443 con el certificado `Let's encrypt` importado
  * **Availability zones:**
    - **VPC:** Orgullo-VPC
    - **Availability Zones:**
      > AZ-A: Public Subnet A <br>
      > AZ-B: Public Subnet B
  * **Security group:** SG-WEB
  * **Target group:**
    - **Name:** Orgullo-TG

Luego de estas configuraciones el balanceador de carga y el grupo de destino están listos.

<br>
Creación de la configuración de lanzamiento
-------------------------------
En la sección `Launch Configurations` se hizo clic en `Create Launch Configuration` con los siguientes parámetros: <br>
  * **Launch Configuration Name:** Orgullo-LaunchConfig
  * **Monitoring:**
    - Se selecciona la opción `Enable EC2 instance detailed monitoring within Cloud Watch` 
  * **Security group:** SG-WEB
Al crearla, esta es la configuración que va a utilizar el `Auto Scaling Group` en cada instancia del Web Server que lance. <br>

Creación del Auto Scaling Group
-------------------------------
En la sección `Launch Configurations` se seleccionó la configuración de lanzamiento creada anteriormente y en el menú `Actions` se seleccionó `Create Auto Scaling Group` y se configuró de la siguiente manera:
  * **Name:** Orgullo - Auto Scaling Group
  * **Network:**
    - **Network:** Orgullo-VPC
    - **Subnet:**
      > Private Subnet A <br>
      > Private Subnet B
  * **Advanced options:**
    - Seleccionar `Attach to an existing load balancer`
    - **Target Group:** Orgullo-TG
    - Se selecciona la opción `Enable group metrics collection within CloudWatch`
  * **Group size:**
    - **Desired:** 2 
    - **Minimum:** 2 
    - **Maximum:** 3
  * **Scaling policies:**
    - **Name:** Orgullo-TargetTrackingPolicy
    - **Metric type:** Average CPU Utilization
    - **Target value:** 60



<br> <br>
Configurando CloudFlare
================================
Certificado Let's encrypt
--------------------------
Luego de obtener el [certificado](https://crt.sh/?q=orgullo2.ml), Cloudflare lo configura automáticamente.

Configuración DNS
--------------------------
Luego de solicitar el dominio en [**Freenom**](https://www.freenom.com/) accedemos a Cloudflare para configurar el DNS con los siguientes registros: <br>
![image](https://user-images.githubusercontent.com/47001432/119207065-98ea8600-ba62-11eb-9359-d89b51df17b3.png)

Se configura un registro `CNAME` para redirigir el tráfico al `Load balancer` creado en AWS. <br>
Se configura un segundo registro `CNAME` para redirigir los accesos al dominio utilizando `www`.



<br> <br>
Configurando Freenom
================================
Una vez se configura [**CloudFlare**](https://www.cloudflare.com/) se accede al panel de freenom para configurar los nameservers que provee [**CloudFlare**](https://www.cloudflare.com/) <br>
![image](https://user-images.githubusercontent.com/47001432/119207270-45c50300-ba63-11eb-8c8e-540a568a567f.png)

Con esta configuración el tráfico es controlado por [**CloudFlare**](https://www.cloudflare.com/) mientras que [**Freenom**](https://www.freenom.com/) provee el dominio.





















