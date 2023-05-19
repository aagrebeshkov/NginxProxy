## Nginx Proxy
Цель: Cоздание Proxy сервера, который в зависимости от введенного в адресной строке, открывает нужный сервис.

### Создание имитации сервисов
Создание трех web-страниц на первом сервере под OS Ubuntu с целью имитации сервисов.

Установка Apache2:
```bash
sudo apt install -y apache2
```
<br>

Создание 3 web-страниц в директории /var/www/html:

| Файл  | Содержимое            | Адрес |
| ------------- |-----------------------| ------------- |
| index.html | \<h1>It worked!\</h1> | http://192.168.1.179/ |
| port8082.html | \<h1>PORT 8082\</h1>  | http://192.168.1.179/port8082.html |
| port8083.html | \<h1>PORT 8083\</h1>  | http://192.168.1.179/port8083.html |
<br>

### Установка и настройка Nging Proxy
Установка Nginx Proxy в контейнере docker на втором сервере под OS CentOS.

Содержимое Dockerfile:
```bash
FROM nginx
COPY nginx.conf /etc/nginx/nginx.conf
```
<br>

Nginx необходимо настроить следующим образом:

| Адрес на Nginx Proxy | Адрес сервиса на первом сервере | Содержимое web-страници |
| ------------- |----------------------| ------------- |
| http://192.168.1.111:8080 | http://192.168.1.179 | It worked! |
| http://192.168.1.111:8080/8082olol | http://192.168.1.179/port8082.html | PORT 8082 |
| http://192.168.1.111:8080/8083olol | http://192.168.1.179/port8083.html | PORT 8083 |
<br>

Содержимое nginx.conf:
```bash
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen       8080;
        location / {
            proxy_pass http://192.168.1.179/;
        }

        location /8082olol {
            proxy_pass http://192.168.1.179/port8082.html;
        }

        location /8083olol {
            proxy_pass http://192.168.1.179/port8083.html;
        }

        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /usr/share/nginx/html/404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /usr/share/nginx/html/50x.html {
        }
    }
}
```
<br>

Сборка образа:
```bash
docker build -t nginx_proxy .
```
<br>

Запуск контейнера с Nginx в отдельном контейнере:
```bash
docker run --name nginx_proxy -d -p 8080:8080 nginx_proxy
```
<br>


Запуск контейнера с Nginx через docker-compose.
Содержимое docker-compose.yml:
```bash
version: '3.1'
services:
  nginx:
    image: nginx_proxy
    restart: always
    ports:
      - 8080:8080
```
<br>

```bash
docker-compose up -d
```
<br>


Результат:
При открытии адреса на Proxy запрос будет отправлен на первый сервер в соответствующую web-страницу. 
Для настоящих сервисов необходимо заменить значение параметров "location" и "proxy_pass" в nginx.conf.<br>
На первом сервере с Apache2 в access логе /var/log/apache2/other_vhosts_access.log можно увидеть соответствующие запросы.
