#Установка node_exporter
* Скачать покет node_exporter
```
 wget https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz
 ```
 * Разорхивировать и переместить в /usr/bin
 ```
 tar -zxvf <имя_архива>.tar.gz
 mv<имя_разархивированного_архива>/node_exporter /usr/bin/
 ```
* Создать пользователя
```
useradd -rs /bin/bash node_exporter
```
* Создать unit в /etc/systemd/system/
 ```
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/bin/node_exporter  --web.config.file=/etc/node_exporter/config.yml

[Install]
WantedBy=multi-user.target
```
* Создать дирректорию для node_exporter
```
mkdir -p /etc/node_exporter && mkdir -p /etc/node_exporter/ssl
```
* Сгенерировать ssl для node_exporter
```
cd /etc/node_exporter/ssl
openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout node_exporter.key -out node_exporter.crt -subj "/C=RU/ST=MOSCOW/L=MOSCOW/O=PROM/CN=localhost" -addext "subjectAltName = DNS:localhost"
```
* Создать файл с конфигом
```
tls_server_config:
  cert_file: /etc/node_exporter/ssl/node_exporter.crt
  key_file: /etc/node_exporter/ssl/node_exporter.key
basic_auth_users:
  prometheus: <зашифрованный_пароль>
```
* Скачать и сгенерировать пароль с помощью htpasswd и вставить его в пункт выше
```
htpasswd -nBC 10 "" | tr -d ':\n'; echo

Пример:
htpasswd -c .htpasswd testuser
New password:
Re-type new password:
Adding password for user testuser
```
* Дать права пользователю node_exporter на конфиг файлы
  ```
  cd /etc && chown -R node_exporter:node_exporter node_exporter/
  ```
* Скачать и дополнить конфиг для nginx
```
apt-get install nginx
################################################################
server {
    listen 19100 ssl;
    server_name _;
    ssl_certificate /etc/node_exporter/ssl/node_exporter.crt;
    ssl_certificate_key /etc/node_exporter/ssl/node_exporter.key;
    location / {
      proxy_pass https://localhost:9100;
      proxy_set_header Host $http_host;
    }
  }
  systemctl restart nginx.service
```
* Включение node_exporter
```
systemctl enable node_exporter.service
systemctl start node_exporter.service
```
* P.S Небходимо разрешить порт 19100 если включен сетевой фильтр. Затем перенести сертификат "node_exporter.crt" на сервер с prometheus.
```
- job_name: "GitLab"
    scheme: https
    basic_auth:
      username: prometheus
      password: "*************"
    tls_config:
      ca_file: /opt/prometheus/ssl/DockerHost/node_exporter.crt
      insecure_skip_verify: true
    static_configs:
    - targets: ['ip:19100']
      labels:
        instance: node-services
```
* Пример: https://nixhub.ru/posts/node-exporter-setup/
