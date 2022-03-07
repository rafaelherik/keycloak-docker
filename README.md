# oAuth Identity Provider using Keycloak



### Requirements
- Docker
- Docker Compose
- VSCode


### Files

idp-docker-compose.yml:

``` yml

version: '3'
volumes:
  mysql_data:
      driver: local
services:
  mysql:
      image: mysql:8.0.20
      volumes:
        - mysql_data:/var/lib/mysql
      environment:
        MYSQL_ROOT_PASSWORD: root
        MYSQL_DATABASE: keycloak
        MYSQL_USER: keycloak
        MYSQL_PASSWORD: <YOURPASSWORD>
      ports:
        - "3309:3306"
      networks:
        - keycloak-network
      depends_on:
        - nginx

  keycloak:
      image: jboss/keycloak:latest
      environment:
        DB_VENDOR: MYSQL
        DB_ADDR: mysql
        DB_DATABASE: keycloak
        DB_USER: keycloak
        DB_PASSWORD: <YOURPASSWORD>
        KEYCLOAK_USER: admin
        KEYCLOAK_PASSWORD: <YOURPASSWORD>
        PROXY_ADDRESS_FORWARDING: "true"
        REDIRECT_SOCKET: "proxy-https"
        KEYCLOAK_FRONTEND_URL: https://keycloak.obarrinha.me/auth
       
      ports:
        - "8080:8080"
      networks:
        - keycloak-network
      depends_on:
        - mysql
        - nginx
        
   nginx:
    image: nginx
    container_name: nginx
    restart: on-failure
    volumes:
      - ./conf:/etc/nginx/conf.d
      - ./certs:/etc/nginx/certs
    ports:
      - "80:80"
      - "443:443"
    networks:
        - keycloak-network

networks:
  key-net:    
      name: keycloak-network

```

nginx.conf (without ssl)

``` yml
upstream keycloak_backend {
  server keycloak:8080;
}
server {
    #listen 127.0.0.1;
    listen 80;
    server_name yourdomain.com;
    location /auth/ {
          proxy_pass "http://keycloak_backend/auth/";
          proxy_set_header   Host $host;
          proxy_set_header   X-Real-IP $remote_addr;
          proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header   X-Forwarded-Host $server_name;
    }
    location /auth/admin {
          proxy_pass "http://keycloak_backend/auth/admin";
          proxy_set_header   Host $host;
          proxy_set_header   X-Real-IP $remote_addr;
          proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header   X-Forwarded-Host $server_name;
    }
}
```


nginx.conf (with SSL)

``` yml
upstream keycloak_backend {
  server keycloak:8080;
}
server {
    #listen 127.0.0.1;
    listen 80;
    server_name yourdomain.com;
    location / {
        return 301 https://$host$request_uri;
    }
}
server {
    listen 443 ssl;
    server_name yourdomain.com;
    ssl_certificate /etc/nginx/certs/yourcertificate.crt;
    ssl_certificate_key /etc/nginx/certs/yourcertificate.key;
location /auth/ {
          proxy_pass "http://keycloak_backend/auth/";
          proxy_set_header   Host $host;
          proxy_set_header   X-Real-IP $remote_addr;
          proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header   X-Forwarded-Host $server_name;
    }
    location /auth/admin {
          proxy_pass "http://keycloak_backend/auth/admin";
          proxy_set_header   Host $host;
          proxy_set_header   X-Real-IP $remote_addr;
          proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header   X-Forwarded-Host $server_name;
    }
 
}
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA RC4 !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS";
```

with SSL configuration you need to pay atention on the certificates, you need to create the certificates and put them on the folder specified on docker-compose file:

```yml
 nginx:
    image: nginx
    container_name: nginx
    restart: on-failure
    volumes:
      - ./conf:/etc/nginx/conf.d 
      - ./certs:/etc/nginx/certs # IMPORTANT!!!!! THE FOLDER .\certs IS THE CERTIFICATES LOCATION 
    ports:
      - "80:80"
      - "443:443"
    networks:
        - keycloak-network

```


## Step by step

#### NGINX configurations

Basically on nginx.conf you need to describe the listen addresses and configure the domain, you can use the nginx on the same docker compose file of Keycloak and MySQL or put it on separate file. If you do it, you could add more services on the same docker service easily.


#### Creating the docker network

Before the start, you need to create the docker network that the services will use to communicate. If you have separate yml files to NGINX and Keycloak, you need to pay atention and use the same network.

```sh
docker network create keycloak-network
```
If you want another name, change it on the files too.


#### Up the containers

```
docker-compose -f idp-docker-compose.yml up -d
```




