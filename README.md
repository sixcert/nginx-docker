# HOW TO: การติดตั้ง SSL Certificates บน Nginx โดยใช้ Docker container


## Requirements
1. Docker Engine (Linux, Windows, AWS, Azure)
2. Docker Compose
2. Docker Images (Nginx) 
2. SSL Certificates file (.crt or .pem)
3. Private Key file (.key)


## Project directory layout

```
.nginx-docker/
|____docker-compose.yaml
|____nginx/
| |____ssl/
| | |____www_sixcert_co.key
| | |____www_sixcert_co.pem
| |____conf.d/
|   |____vhost-sixcert_co.conf
|____public/
  |____index.php

```

![project-dir-layout.png](https://www.sixcert.co/wp-content/uploads/2018/12/project-dir-layout.png)


## Getting Started
1. หลังจากที่ทำการติดตั้ง `Docker Engine` เรียบร้อยแล้วให้ทำการตรวจสอบความพร้อมใช้งานโดย run command

    ```shell
    $ docker --version
    Docker version 18.09.0, build 4d60db4
    ```

2. ให้ทำการติดตั้ง `docker-compose` เพิ่มเติมเพื่อใช้สำหรับจัดการและส่ัง run docker container ได้อย่างมีประสิทธิภาพและสะดวกยิ่งกว่าการใช้งาน command ผ่าน `docker engine` โดยตรง ตรวจสอบการติดตั้งโดย run command
    
    ```shell
    $ docker-compose --version
    docker-compose version 1.23.2, build 1110ad01
    ```


3. ให้ทำการสร้าง directory ต่างๆตาม [Project directory layout](#Project-directory-layout) ด้านบนภายในเครื่อง Host ที่ได้ทำการติดตั้ง docker-engine ไว้ประกอบด้วย
    1. **docker-compose.yaml** *(เก็บ config ต่าง container)*
    2. **nginx/ssl/** *(เก็บไฟล์ ssl certificates และ private key)*
    3. **nginx/conf.d/** *(เก็บไฟล์ nginx configuration)*
    4. **public/** *(เก็บไฟล์ข้อมูลต่างของเว็บไซต์เช่น .php)*


4. ทำการกำหนดค่า container ที่ต้องการใช้งานลงในไฟล์ `docker-compose.yaml` ดังตัวอย่าง

    ```yaml
    version: '2'
    services:

      # Nginx webserver
      nginx:
        image: nginx:latest
        restart: always
        volumes:
          - ./nginx/conf.d:/etc/nginx/conf.d:ro
          - ./nginx/ssl:/etc/nginx/ssl:ro
        ports:
          - "80:80"
          - "443:443"
        volumes_from:
          - php

      # PHP-FPM for compile .php file
      php:
        image: php:7.2-fpm-alpine
        restart: always
        volumes:
          - ./public:/var/www/html
    ```
    คำอธิบาย
    - กำหนด nginx container โดยใช้ image version ล่าสุดและทำการ map volumes `./nginx` จากเครื่อง host ที่ run docker-engine เข้าไปใน nginx container เพื่อสามารถอ่านไฟล์ configuration, ssl certificate และ private key จากภายใน container ได้
    - ในที่นี้เราได้ทำการเพิ่ม php-fpm container เพิ่มเติมขึ้นมาเพื่อให้เหมือนกับการใช้งานจริงมากยิ่งขึ้น โดยให้ทำการ map volumes `./public` เข้าไปใน container ของ php-fpm ให้อยู่ภายใต้ `/var/www/html/` เพื่อให้ php-fpm สามารถประมวลผลไฟล์ `.php` เว็บไซต์ของเราได้


5. นำไฟล์ ssl certificate (.pem or .crt) ที่ได้รับจาก [SSL Providers](https://www.sixcert.co) และ private key (.key) ไปวางไว้ที่โฟล์เดอร์ `./nginx/ssl/`


6. สร้างไฟล์ nginx configuration (virtual host) ตัวอย่างเช่น `vhost-<<domain>>.conf` และวางไว้ในโฟล์เดอร์ `./nginx/conf.d/` 
    
    - โดยกำหนด `server_name` เป็นชื่อโดเมนของเว็บไซต์ที่ต้องการ และกำหนด `ssl_certificate` และ `ssl_certificate_key` ให้ชี้ไปที่ไฟล์ certificate และ private key ภายใต้ `/etc/nginx/ssl/` ซึ่งจะเป็น path ภายใน container ที่ได้ทำการ map volumes ไว้จาก docker host ตามขั้นตอนที่ 4

      ```nginx
          ...
          server_name sixcert.co www.sixcert.co;

          ssl_certificate /etc/nginx/ssl/www_sixcert_co.pem;
          ssl_certificate_key /etc/nginx/ssl/www_sixcert_co.key;
          ...
      ```
    
    - ทำการ redirect จาก `http` ไปที่ `https` ดังนี้โดยกำหนด `server_name` และ
    `return 301 https://<<domain>>$request_uri;`
    
      ```nginx
      ...
      server {
          listen 80;
          listen [::]:80;
          server_name sixcert.co www.sixcert.co;
          return 301 https://www.sixcert.co$request_uri;
      }
      ```
    
    << Nginx Configuration>>

    ```nginx
    server {
        listen 443 ssl http2 default_server;
        listen [::]:443 ssl http2 default_server;
        server_name sixcert.co www.sixcert.co;

        ssl_certificate /etc/nginx/ssl/www_sixcert_co.pem;
        ssl_certificate_key /etc/nginx/ssl/www_sixcert_co.key;

        root /var/www/html;
        index index.php index.html;

        location / {
            try_files $uri $uri/ /index.php?$args;
        }

        location ~ \.php {
            fastcgi_index index.php;
            fastcgi_pass php:9000;

            include fastcgi_params;
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_param PATH_INFO $fastcgi_path_info;
            fastcgi_param PATH_TRANSLATED $document_root$fastcgi_path_info;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }
    }

    server {
        listen 80;
        listen [::]:80;
        server_name sixcert.co www.sixcert.co;
        return 301 https://www.sixcert.co$request_uri;
    }

    ```


5. ทำการ start nginx และ php container ดัวย command ดังต่อไปนี้

    ```shell
    $ docker-compose up -d
    Creating network "nginx-docker_default" with the default driver
    Creating nginx-docker_php_1 ... done
    Creating nginx-docker_nginx_1 ... done
    ```
    *หมายเหตุ: เครื่อง docker-engine ที่ยังไม่เคยทำการ download images มาก่อนให้รอ docker-engine ทำการ pull image จาก Docker Hub สักครู่*


8. ตรวจสอบสถานะ container ที่ได้สร้างไว้ดังนี้

    ```shell
    $ docker-compose ps
            Name                      Command              State                      Ports
    ----------------------------------------------------------------------------------------------------------
    nginx-docker_nginx_1   nginx -g daemon off;            Up      0.0.0.0:443->443/tcp, 0.0.0.0:80->80/tcp
    nginx-docker_php_1     docker-php-entrypoint php-fpm   Up      9000/tcp
    ```

    *ให้สังเกตุตรงคอลัมน์ `State` จะต้องแสดงเป็น `Up` ทุก container หากไม่แสดงให้ตรวจสอบ log ของ container โดย command`$ docker-compose logs <<container>>`*


9. ทดสอบเข้าเว็บไซต์ที่ได้ทำการสร้างผ่าน Web Browser

![test-access-website.png](https://www.sixcert.co/wp-content/uploads/2018/12/test-access-website.png)


10. ตรวจสอบการติดตั้ง SSL Certificate โดยการกดสัญลักษ์รูปกุญบน Address bar ของ Web Browser!
 
![verify-installed-ssl-certificate.png](https://www.sixcert.co/wp-content/uploads/2018/12/verify-installed-ssl-certificate.png)



## Links: 
1. [Download Docker](https://store.docker.com/search?type=edition&offering=community)
2. [Install Docker CE for CentOS \| Docker Documentation](https://docs.docker.com/install/linux/docker-ce/centos/)
3. [Install Docker CE for Ubuntu \| Docker Documentation](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
4. [Install Docker for Windows \| Docker Documentation](https://docs.docker.com/docker-for-windows/install/)
5. [Install Docker Compose \| Docker Documentation](https://docs.docker.com/compose/install/)
6. [SSL Certificates Provider | sixcert.co](https://www.sixcert.co/)