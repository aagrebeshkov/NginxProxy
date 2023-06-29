## Nginx HTTP load balancer
Цель: Cоздание балансировщика.

### Создание Web сервиса
Создание web-страницы на двух серверах под OS Ubuntu.

Установка Apache2:
```bash
sudo apt install -y apache2
```
<br>

Создание web-страницы в директории /var/www/html:

| Сервер | Файл  | Содержимое            | Адрес |
| ------------- | ------------- |-----------------------| ------------- |
| node01 | index.html | \<h1>Your request got to the server node01!\</h1> | http://192.168.50.2/ |
| node01 | app/index.html | \<h1>Your request got to the server node01!\</h1> | http://192.168.50.2/app |
| node02 | index.html | \<h1>Your request got to the server node02!\</h1> | http://192.168.50.3/ |
| node02 | app/index.html | \<h1>Your request got to the server node02!\</h1> | http://192.168.50.3/app |
<br>

### Установка и настройка Nging Proxy в Docker
Установка Nginx Proxy в контейнере docker на втором сервере под OS CentOS.

Содержимое docker-compose.yml:
```bash
version: '2'
services:
  web:
    image: nginx
    restart: always
    environment:
      TZ: Europe/Moscow
    volumes:
      - ./config/:/etc/nginx/conf.d
    ports:
      - 8080:8080
    healthcheck:
      test: ["CMD-SHELL", "service nginx status"]
      interval: 10s
      timeout: 10s
      retries: 10
```
<br>

Nginx необходимо настроить следующим образом:
При открытии web-страницы первые 5 запросов будут направлены на сервер node01, а шестой запро на node02.
При открытии web-страницы по адресу http://nginx:8080/ запросы будут перенаправлены на Apache2 на порт 80.
При открытии web-страницы по адресу http://nginx:8080/app запросы будут перенаправлены на Apache2 на адресс 80/app.
<br>

Содержимое nginx.conf остается не измененным.
Изменяем соджержимое default.conf (по пути config/default.conf на хостовой машине):
```bash
upstream inform_hosts {
  # weight - 5 запросов на сервер 192.168.50.2 и один запрос на 192.168.50.3
  # max_fails — количество неудачных попыток, после которых будем считать сервер недоступным.
  # fail_timeout — время, в течение которого сервер нужно считать недоступным и не отправлять на него запросы.
  # max_conns — максимальное число подключений, при превышении которого запросы на бэкенд не будут поступать. По умолчанию равно 0 (безлимитно).

  server 192.168.50.2 weight=5 max_fails=1 fail_timeout=10 max_conns=5;
  server 192.168.50.3 weight=1;
}

server {
  listen 8080;
  server_name _;

  location / {
    proxy_pass http://inform_hosts;
    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }

  location /app {
    proxy_pass http://inform_hosts/app/index.html;
    proxy_redirect off;
    proxy_http_version 1.1;
  }
}
```
<br>

Запуск контейнера с Nginx:
```bash
docker-compose up -d
```
<br>

Результат: <br>
Сервис на node01 доступен на прямую по адресам: http://192.168.50.2/ и http://192.168.50.2/app/ <br>
Сервис на node02 доступен на прямую по адресам: http://192.168.50.3/ и http://192.168.50.3/app/ <br>
Сервис доступен через балансировщик по адресам: http://192.168.1.123:8080/ и http://192.168.1.123:8080/app