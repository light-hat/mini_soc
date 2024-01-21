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

TBC...

## Выгружаем docker tar образы на сервер

`#ЕбалиМыВашиСанкции`

TBC...

## Альтернатива: Pull

Нужен VPN. Грузится с хранилища минут 20.

TBC...

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

## Запуск

Конфиг. Создаем файл .env:

```yaml
TBC...
```

TBC...

Запуск стека:

```bash
docker-compose up -d --build
```

## Конфигурируем Nginx

TBC...

## Темная тема

`Management > Advanced Setting > "dashboard:defaultDarkTheme"`

## Кастомизация

https://documentation.wazuh.com/current/user-manual/wazuh-dashboard/custom-branding.html
