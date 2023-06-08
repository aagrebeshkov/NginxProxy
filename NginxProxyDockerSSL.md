## Nginx Proxy SSL
Цель: Cоздание Proxy сервера и настройка SSL для конечного сервиса на стороне Nginx.

### Создание Web сервиса
Создание web-страницы на первом сервере под OS Ubuntu.

Установка Apache2:
```bash
sudo apt install -y apache2
```
<br>

Создание web-страницы в директории /var/www/html:

| Файл  | Содержимое            | Адрес |
| ------------- |-----------------------| ------------- |
| index.html | \<h1>It worked!\</h1> | http://192.168.1.179/ |
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
    volumes:
      - ./config/:/etc/nginx/conf.d
      - ./keys:/etc/nginx/keys
    ports:
      - 8080:8080
      - 8443:8443
```
<br>

Nginx необходимо настроить следующим образом:
При открытии web-страницы по порту 8080 будет запрос будет перенаправлен на web-сервис без SSL.
При открытии web-страницы по порту 8443 будет запрос будет перенаправлен на web-сервис с использованием SSL.

<br>

Содержимое nginx.conf остается не измененным.
Изменяем соджержимое default.conf (по пути config/default.conf на хостовой машине):
```bash
upstream inform_hosts {
    server 192.168.1.179:80;
}

upstream inform_https_hosts {
    server 192.168.1.179:80;
}

server {
    listen 8080;
    server_name _;
    
    location / {
        client_max_body_size 200M;
        proxy_read_timeout 1200;
        proxy_connect_timeout 1200;
        proxy_pass http://inform_hosts;
        proxy_set_header Host 'apacheserver.local';
        proxy_set_header X-Real_IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}

server {
    listen 8443 ssl;
    server_name _;
    ssl_certificate keys/apacheserver.crt;
    ssl_certificate_key keys/key.key;

    location / {
        client_max_body_size 200M;
        proxy_read_timeout 1200;
        proxy_connect_timeout 1200;
        proxy_pass http://inform_https_hosts;
        proxy_set_header Host 'apacheserver.local';
        proxy_set_header X-Real_IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```
<br>

Для работы web-сервиса с использованием SSL необходимо сгенерировать SSL сертификаты. 
Создание самоподписного сертификата и ключа:
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout keys/key.key \
-out keys/apacheserver.crt \
-subj "/C=RU/ST=Moscow/L=Moscow/O=tp-test-03/OU=Org/CN=apacheserver.local"
```
<br>

В моем случае нет DNS, поэтому я прописал в hosts адрес apacheserver.local на серверах Proxy, Apache и Mac.
Перезапустил Apache и проверил работу сервиса: http://apacheserver.local/
<br>

Запуск контейнера с Nginx:
```bash
docker-compose up -d
```
<br>

Результат: <br>
Сервис доступен на прямую по адресам: http://192.168.1.179/ и http://apacheserver.local/ <br>
Сервис доступен через Proxy по адресам: http://192.168.1.123:8080/ и https://192.168.1.123:8443/
