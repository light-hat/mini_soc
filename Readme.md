# SOC infrastructure

Ports to allow:

```
514/udp
1514/tcp
1515/tcp
55000/tcp
80/tcp
443/tcp
666/tcp (ssh)
```

## Проброс нужных портов на iptables

```bash
sudo iptables -A INPUT -p udp --dport 514 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 1514 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 1515 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 55000 -j ACCEPT

sudo service netfilter-persistent save
```

## Выгружаем docker tar образы на сервер

`#ЕбалиМыВашиСанкции`

Сохраняем нужные образы:

```bash
docker save -o siem-manager.tar wazuh/wazuh-manager:4.7.2
docker save -o siem-indexer.tar wazuh/wazuh-indexer:4.7.2
docker save -o siem-dashboard.tar wazuh/wazuh-dashboard:4.7.2
docker save -o thehive.tar strangebee/thehive:5.2
docker save -o cassandra.tar cassandra:4
docker save -o elasticsearch.tar docker.elastic.co/elasticsearch/elasticsearch:7.17.12
docker save -o minio.tar quay.io/minio/minio
docker save -o cortex.tar thehiveproject/cortex:3.1.7
# Пакуем их в архив images.zip
```

Отправляем на сервер через SCP:

```bash
scp -P 666 images.zip <user>@192.168.45.132:/home/<user>/images.zip
```

Импортируем образы уже на сервере:

```bash
unzip images.zip

sudo docker load --input siem-manager.tar
sudo docker load --input siem-indexer.tar
sudo docker load --input siem-dashboard.tar
sudo docker load --input thehive.tar
sudo docker load --input cassandra.tar
sudo docker load --input elasticsearch.tar
sudo docker load --input minio.tar
sudo docker load --input cortex.tar

rm *.tar *.zip
```

## Альтернатива: Pull

Нужен VPN. Грузится с хранилища минут 20. SIEM: 

```bash
sudo docker pull wazuh/wazuh-manager:4.7.2
```

```bash
sudo docker pull wazuh/wazuh-indexer:4.7.2
```

```bash
sudo docker pull wazuh/wazuh-dashboard:4.7.2
```

The Hive stack:

```bash
sudo docker pull strangebee/thehive:5.2
```

```bash
sudo docker pull cassandra:4
```

```bash
sudo docker pull docker.elastic.co/elasticsearch/elasticsearch:7.17.12
```

```bash
sudo docker pull quay.io/minio/minio
```

```bash
sudo docker pull thehiveproject/cortex:3.1.7
```

## Подготовка 

Клонируем репозиторий и пару настроек выставляем

```bash
git clone https://git.area51-lab.ru/architect/soc_server
cd soc_server/
sudo sysctl -w vm.max_map_count=262144
```

Генерим сертификаты для SIEM:

```bash
docker-compose -f generate-indexer-certs.yml run --rm generator
```

## Конфиг системы

Создаем файл .env.siem.manager:

```bash
vim .env.siem.manager
```

```yaml
INDEXER_URL=https://wazuh.indexer:9201
INDEXER_USERNAME=admin
INDEXER_PASSWORD=SecretPassword
FILEBEAT_SSL_VERIFICATION_MODE=full
SSL_CERTIFICATE_AUTHORITIES=/etc/ssl/root-ca.pem
SSL_CERTIFICATE=/etc/ssl/filebeat.pem
SSL_KEY=/etc/ssl/filebeat.key
API_USERNAME=wazuh-wui
API_PASSWORD=MyS3cr37P450r.*-
```

Создаем файл .env.siem.indexer:

```bash
vim .env.siem.indexer
```

```yaml
"OPENSEARCH_JAVA_OPTS=-Xms1024m -Xmx1024m"
INDEXER_USERNAME=admin
INDEXER_PASSWORD=SecretPassword
```

Создаем файл .env.siem.dashboard:

```bash
vim .env.siem.dashboard
```

```yaml
PATTERN="wazuh-alerts-*"
INDEXER_URL=https://wazuh.indexer:9201
INDEXER_USERNAME=admin
INDEXER_PASSWORD=SecretPassword
WAZUH_API_URL=https://wazuh.manager
DASHBOARD_USERNAME=kibanaserver
DASHBOARD_PASSWORD=kibanaserver
API_USERNAME=wazuh-wui
API_PASSWORD=MyS3cr37P450r.*-
```

Создаем файл .env.hive:

```bash
vim .env.hive
```

```yaml
JVM_OPTS="-Xms1024M -Xmx1024M"
```

Создаем файл .env.cassandra:

```bash
vim .env.cassandra
```

```yaml
MAX_HEAP_SIZE=1024M
HEAP_NEWSIZE=1024M
CASSANDRA_CLUSTER_NAME=TheHive
```

Создаем файл .env.elasticsearch:

```bash
vim .env.elasticsearch
```

```yaml
discovery.type=single-node
xpack.security.enabled=false
```

Создаем файл .env.minio:

```bash
vim .env.minio
```

```yaml
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
```

Создаем файл .env.cortex:

```bash
vim .env.cortex
```

```yaml
job_directory=/tmp/cortex-jobs
```

## Запуск стека

```bash
docker-compose up -d --build
```

## Конфигурируем Nginx

Открываем конфигурацию:

```bash
sudo rm /etc/nginx/sites-available/default
sudo vim /etc/nginx/sites-available/default
```

Сам конфиг:

```nginx
# ЭТО ПОКА ТЕСТОВАЯ ВЕРСИЯ! НЕ ГРУЗИТЬ В ПРОД!

upstream wazuh {
    server 127.0.0.1:5601;
}

upstream cortex {
    server 127.0.0.1:9001;
}

upstream thehive {
    server 127.0.0.1:9000;
}

# upstream misp {
#     server misp:9002;
# }

server {
    listen 80;
    #server_name siem.area51-lab.ru;
    return 301 https://$host$request_uri;
}

# server {
#     listen 80;
#     server_name cortex.area51-lab.ru;
#     return 301 https://$host$request_uri;
# }

# server {
#     listen 80;
#     server_name soar.area51-lab.ru;
#     return 301 https://$host$request_uri;
# }

# server {
#     listen 80;
#     server_name misp.area51-lab.ru;
#     return 301 https://$host$request_uri;
# }

server {
    listen 443 ssl;
    #server_name siem.area51-lab.ru;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS;
    ssl_certificate /etc/ssl/cert-cert.crt;
    ssl_certificate_key /etc/ssl/cert-key.key;

    client_body_buffer_size     32k;
    client_header_buffer_size   8k;
    large_client_header_buffers 8 64k;

    access_log /var/log/nginx/nginx-siem-access.log;
    error_log /var/log/nginx/nginx-siem-error.log;

    location / {
        proxy_pass https://wazuh;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }
}

server {
    listen 444 ssl; # !!!
    #server_name cortex.area51-lab.ru;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS;
    ssl_certificate /etc/ssl/cert-cert.crt;
    ssl_certificate_key /etc/ssl/cert-key.key;

    client_body_buffer_size     32k;
    client_header_buffer_size   8k;
    large_client_header_buffers 8 64k;

    access_log /var/log/nginx/nginx-cortex-access.log;
    error_log /var/log/nginx/nginx-cortex-error.log;

    location / {
        proxy_pass http://cortex;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }
}

server {
    listen 445 ssl; # !!!
    #server_name soar.area51-lab.ru;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS;
    ssl_certificate /etc/ssl/cert-cert.crt;
    ssl_certificate_key /etc/ssl/cert-key.key;

    client_body_buffer_size     32k;
    client_header_buffer_size   8k;
    large_client_header_buffers 8 64k;

    access_log /var/log/nginx/nginx-soar-access.log;
    error_log /var/log/nginx/nginx-soar-error.log;

    location / {
        proxy_pass http://thehive;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }
}

# server {
#     listen 443 ssl;
#     server_name misp.area51-lab.ru;

#     ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
#     ssl_prefer_server_ciphers on;
#     ssl_ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS;
#     ssl_certificate /etc/ssl/cert-cert.crt;
#     ssl_certificate_key /etc/ssl/cert-key.key;

#     client_body_buffer_size     32k;
#     client_header_buffer_size   8k;
#     large_client_header_buffers 8 64k;

#     access_log /var/log/nginx/nginx-misp-access.log;
#     error_log /var/log/nginx/nginx-misp-error.log;

#     location / {
#         proxy_pass http://misp;
#         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
#         proxy_set_header Host $host;
#         proxy_redirect off;
#     }
# }

```

Перезагружаем nginx:

```bash
sudo systemctl restart nginx
sudo systemctl status nginx
```

## Темная тема

`Management > Advanced Setting > "dashboard:defaultDarkTheme"`

## Кастомизация

https://documentation.wazuh.com/current/user-manual/wazuh-dashboard/custom-branding.html
