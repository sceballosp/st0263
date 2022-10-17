# Información de la materia:
ST0263 - Tópicos Especiales en Telemática

# Estudiante:
Samuel Ceballos Posada, sceballosp@eafit.edu.co

# Profesor:
Edwin Nelson Montoya Munera, [emontoya@eafit.edu.co](mailto:emontoya@eafit.edu.co)

# 1. Breve descripción de la actividad
Desarrollar habilidades en el proceso de creación, despliegue y gestión de una aplicacion monolitica con balanceo y datos distribuidos, utilizando docker y aprender a asignar un certificado ssl válido a un dominio en Let's Encrypt.

## 1.1. Que aspectos cumplió o desarrolló de la actividad propuesta por el profesor
- Desplegar las cinco instancias requeridad (Balanceador, Wordpress 1 y 2, NFS y Base de datos).
- Asignar un dominio a la dirección ip del balanceador.
- Conseguir un certificado SSL válido para el dominio.

## 1.2. Que aspectos NO cumplió o desarrolló de la actividad propuesta por el profesor
Todo lo propuesto ha sido implementado.

# 2. información general de diseño
Se utilizaron contenedores Docker para la instalación de Wordpress, MySQL y Nginx.

# 3. Descripción del ambiente de desarrollo y técnico:

## Dirección IP y nombre de dominio
- IP elástica del balanceador: 34.171.169.6
- Dominio: sceballosp.tk
- Dominio con certificacion: https://lab4.sceballosp.tk

## Detalles técnicos
- GCP: servicio para desplegar las instancias.
- Docker: contenedor para desplegar Wordpress, MySQL y Nginx.
- Nginx: servidor web.
- Certbot y Let's Encrypt: servicio para asignar certificado SSL.

## Descripción y cómo se configura los parámetros del proyecto 
### Crear cinco máquinas virtuales en GCP (asignarle un nombre distinto a cada una):
- Acceder a la consola de GCP (console.cloud.google.com).
- Dar click en **Compute Engine**.
- Dar click en **Crear Instancia**.
- Configurar la instancia para que sea de tipo ec2-micro y habilitar HTTP y HTTPS.
- Dar click en **CREAR**.

![instancias](https://user-images.githubusercontent.com/60147093/196289565-2654ba20-4590-4b02-ba78-d252c140c066.PNG)

### Asignar IP elastica a las instancias creadas:
- Dar click en el menú de navegación.
- Dar click en **Red de VPC**.
- Dar click en **Direcciones IP**.
- Seleccionar la IP externa de cada instancia creada y dar click en **RESERVAR**.

![direccionesIP](https://user-images.githubusercontent.com/60147093/196289608-736ac9be-44a0-4006-9277-560483a3ee79.PNG)

### Configurar DNS en GCP:
- En la barra de búsqueda ingresar *Cloud DNS*.
- Dar click en **Crear Zona**.
- Agregar los registros **A** y **CNAME** para el dominio.

![dns](https://user-images.githubusercontent.com/60147093/196289610-9649eaef-e099-4de4-98a0-387c9c4b1f94.PNG)

### Configurar los nameservers en el proveedor del dominio (Freenom):
- Dar click en **Manage domain**
- Dar click en **Management tools** -> **Nameservers**
- Agregar los dominios NS que da GCP.

![Freenom](https://user-images.githubusercontent.com/60147093/190935004-0951fdce-8c41-4871-8dbc-ead82a190ff4.PNG)

## Configuración del Balanceador:
### Conectarse a la instancia por SSH:
- Acceder a la consola de GCP.
- Dar click en **Compute Engine**.
- Dar click en **VM Instances**.
- Dar click en el botón *SSH* de la instancia deseada. 

### Instalar certbot, let's encrypt y nginx:
Usar los siguientes comandos:
```
sudo apt update
sudo apt install python3-pip
sudo -H pip3 install certbot
sudo apt install letsencrypt -y
sudo apt install nginx -y
```
### Configurar nginx.conf

Entrar al archivo de configuración:
```
sudo vim /etc/nginx/nginx.conf
```

Borrar el contenido del archivo y reemplazarlo con lo siguiente:
```
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections  1024;  ## Default: 1024
}
http {
    server {
        listen  80 default_server;
        server_name _;
        location ~ /\.well-known/acme-challenge/ {
            allow all;
            root /var/www/letsencrypt;
            try_files $uri = 404;
            break;
        }
    }
}

```

Guardar la configuración de nginx:
```
sudo mkdir -p /var/www/letsencrypt
sudo nginx -t
sudo service nginx reload
```

### Pedir los certificados ssl
Ejecutar el siguiente comando para certificados específicos (ej. www):
```
sudo letsencrypt certonly -a webroot --webroot-path=/var/www/letsencrypt -m sceballosp@eafit.edu.co --agree-tos -d lab4.sceballosp.tk
```
Ejecutar el siguiente comando para wildcards:
```
sudo certbot --server https://acme-v02.api.letsencrypt.org/directory -d *.sceballosp.tk --manual --preferred-challenges dns-01 certonly
```
- **IMPORTANTE:** Al ejecutar el comando anterior hay un periodo de espera en el que se debe agregar un registro TXT en el DNS para que funcione. 

### Configuración de archivos:
Crear carpeta para los certificados:
```
mkdir -p nginx/ssl
```

Copiar los certificados a la carpeta creada anteriormente:
```
sudo su
cp /etc/letsencrypt/live/lab4.sceballosp.tk/* /home/sceballosp/nginx/ssl/
cp /etc/letsencrypt/live/sceballosp.tk/* /home/sceballosp/nginx/ssl/
```

Crear el archivo de configuración options-ssl-nginx.conf:
```
sudo nano /etc/letsencrypt/options-ssl-nginx.conf
```

En el archivo creado anteriormente copiar lo siguiente:
```
# This file contains important security parameters. If you modify this file
# manually, Certbot will be unable to automatically provide future security
# updates. Instead, Certbot will print and log an error message with a path to
# the up-to-date file that you will need to refer to when manually updating
# this file.
ssl_session_cache shared:le_nginx_SSL:10m;
ssl_session_timeout 1440m;
ssl_session_tickets off;
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;
ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
```

Copiar el archivo ***options-ssl-nginx.conf***:
```
cp /etc/letsencrypt/options-ssl-nginx.conf /home/sceballosp/nginx/ssl/
```

Acceder al directorio ***nginx/ssl*** y crear la llave ss-dhparams.pem:
```
cd nginx/ssl
openssl dhparam -out ssl-dhparams.pem 512
```

Correr los siguientes comandos:
```
DOMAIN='lab4.sceballosp.tk' bash -c 'cat /etc/letsencrypt/live/$DOMAIN/fullchain.pem /etc/letsencrypt/live/$DOMAIN/privkey.pem > /etc/letsencrypt/$DOMAIN.pem'
cp /etc/letsencrypt/live/sceballosp.tk/* /home/sceballosp/nginx/ssl/
exit
```

### Configuración de Docker
Instalar docker, docker-compose y git:
```
sudo apt install docker.io -y
sudo apt install docker-compose -y
sudo apt install git -y
```

Inicializar Docker:
```
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -a -G docker sceballosp
sudo reboot
```

Clonar repositorio con la configuración de Docker y copiar archivos:
```
git clone https://github.com/st0263eafit/st0263-2022-2.git
cd st0263-2022-2/docker-nginx-wordpress-ssl-letsencrypt
sudo cp docker-compose.yml /home/sceballosp/nginx
sudo cp nginx.conf /home/sceballosp/nginx
sudo cp ssl.conf /home/sceballosp/nginx
```

Detener nginx:
```
ps ax | grep nginx
netstat -an | grep 80
sudo systemctl disable nginx
sudo systemctl stop nginx
```

Borrar el contenido del archivo de configuracion ***nginx.conf*** y reemplazarlo con lo siguiente:
(asegurese de cambiar las direcciones IP privadas de sus instancias con wordpress)
```
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
events {
  worker_connections  1024;  ## Default: 1024
}
http {
  upstream loadbalancer{
    server 10.128.0.51:80 weight=5;
    server 10.128.0.52:80 weight=5;
  }
  server {
    listen 80;
    listen [::]:80;
    server_name _;
    rewrite ^ https://$host$request_uri permanent;
  }
  server {
    listen 443 ssl http2 default_server;
    listen [::]:443 ssl http2 default_server;
    server_name _;
    # enable subfolder method reverse proxy confs
    #include /config/nginx/proxy-confs/*.subfolder.conf;
    # all ssl related config moved to ssl.conf
    include /etc/nginx/ssl.conf;
    client_max_body_size 0;
    location / {
      proxy_pass http://loadbalancer;
      proxy_redirect off;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Host $host;
      proxy_set_header X-Forwarded-Server $host;
      proxy_set_header X-Forwarded-Proto $scheme;
    }
  }
}
```

Borrar el contenido del archivo ***docker-compose.yml*** y reemplazarlo con lo siguiente:
```
version: '3.1'
services:
  nginx:
    container_name: nginx
    image: nginx
    volumes:
    - ./nginx.conf:/etc/nginx/nginx.conf:ro
    - ./ssl:/etc/nginx/ssl
    - ./ssl.conf:/etc/nginx/ssl.conf
    ports:
    - 80:80
    - 443:443
```

Inicializar Docker:
```
docker-compose up --build -d
```

## Resultados
Ingresar a https://lab4.sceballosp.tk



Se puede ver el certificado SSL:


# 4. Información relevante

## Referencias:
- https://github.com/st0263eafit/st0263-2022-2/tree/main/docker-nginx-wordpress-ssl-letsencrypt
- https://www.letscloud.io/community/how-to-set-up-an-nginx-with-certbot-on-ubuntu
- https://geekrewind.com/setup-lets-encrypt-wildcard-on-ubuntu-20-04-18-04/