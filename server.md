# Настройка прокси-сервера на Ubuntu

Здесь даётся инструкция, актуальная для Ubuntu 22/24.
В итоге получится прокси, работающий на портах 443 (HTTPS) и 8080 (HTTP).

Трафик HTTPS прокси на порту 443 неотличим от трафика любого сайта, работающего по протоколу HTTPS.
Трафик HTTP прокси на порту 8080 не зашифрован и может быть просмотрен администратором. Нужен только для приложений, которые не поддерживают HTTPS прокси.

## 1. Зарегистрироваться на хостинге и создать сервер

Нужно выбрать хостинг для размещения сервера. Выбирать нужно в зависимости от требований к стране и удобству оплаты.
Примеры:

* [Hay Host](https://hayhost.am/?language=russian) (Армения)
* [gohost.kz](https://gohost.kz/) (Казахстан)
* [is*hosting](https://ishosting.io/ru) (Множество локаций)

Там нужно создать сервер на Ubuntu 22/24, мощная конфигурация не нужна.

В итоге сервер должен быть по ssh через определённый IP.

## 2. Зарегистрировать доменное имя

Регистрация домена нужна для корректного использования прокси в любых приложениях.
Это необходимо, так как нам будет нужен валидный HTTPS сертификат, выданный Let's Encrypt, а для его получения нужен собственный домен.
В случае, если есть желание и умение использовать самоподписанный сертификат, этот шаг можно пропустить.

Домен может быть вида `<рандомные буквы и цифры>.ru` или это может быть домен третьего уровня,
если уже имеется домен второго уровня: `<my-subdomain>.<my-domain>.ru`.

Дальше для простоты будет использоваться `<my-domain>.ru`

Зарегистрировать домен можно в специализированных регистраторах:

* [Рег.ру](https://reg.ru)
* [РуЦЕНТР](https://www.nic.ru/catalog/domains/)

Далее с помощью DNS нужно привязать `<my-domain>.ru` к серверу `<my-server-ip>`.
Регистраторы предоставляют свои DNS серверы, их можно использовать.

## 3. Обновить систему

Подключиться к серверу через SSH.
В случае если хостинг создаёт только пользователя root, нужно подключиться к серверу следующим образом:

```sh
ssh root@<my-domain>.ru
```

Если же хостинг создал пользователя, нужно подключиться к серверу следующим образом:

```sh
ssh myuser@<my-domain>.ru
```

Выполнить обновление:

```sh
sudo apt update        # Обновить списки пакетов
sudo apt dist-upgrade  # Обновить установленные в системе пакеты
```

## 4. Создать пользователя

**Этот шаг нужен, если хостинг создаёт только пользователя root.**

Если хостинг сразу создаёт пользователя, отличного от root, этот шаг можно пропустить.
Если нет – надо создать отдельного пользователя с sudo. Эти команды выполняются из сеанса пользователя root.

```sh
adduser myuser                   # Добавить пользователя в систему
usermod -aG sudo myuser          # Разрешить sudo для пользователя
visudo -f /etc/sudoers.d/myuser  # Создать и открыть для редактирования файл с параметрами sudo для пользователя
```

В файл `/etc/sudoers.d/myuser` нужно вставить строку:

```sudoers
myuser ALL=(ALL) NOPASSWD:ALL
```

Теперь нужно выйти из сеанса root и залогиниться под пользователем MyUser.

## 5. Настроить файрволл

```sh
sudo apt install ufw             # Установить файрвол
sudo ufw enable                  # Включить автозапуск сервиса
sudo ufw limit 22/tcp            # Разрешить входящие на порт 22 (SSH), но ограничить создание новых соединений на IP
sudo ufw allow 80/tcp            # Разрешить входящие на порт 80 (HTTP)
sudo ufw allow 8080/tcp          # Разрешить входящие на порт 8080 (HTTP Alt)
sudo ufw allow 443/tcp           # Разрешить входящие на порт 443 (HTTPS)
sudo ufw allow 8443/tcp          # Разрешить входящие на порт 8443, это специальный порт для Let's Encrypt
sudo ufw default deny incoming   # По умолчанию запретить входящие
sudo ufw default allow outgoing  # Разрешить все исходящие
sudo ufw status verbose          # Посмотреть, что получилось в итоге
```

## 6. Настроить SSH

**Только если ключей нет.** В случае если на *локальном компьютере* отсутствуют SSH ключи, нужно их создать:

```sh
ssh-keygen  # Создать новый ключ
```

На **локальном компьютере** выполнять команду копирования публичного ключа на прокси-сервер:

```sh
ssh-copy-id myuser@<my-domain>.ru  # Копировать публичный ключ на прокси-сервер
```

**На сервере** выполнить настройку ssh:

```sh
sudo vim /etc/ssh/sshd_config  # Открыть файл настроек ssh
```

В файле нужно раскомментировать или добавить следующие опции:

* `PermitRootLogin no` – запретить вход от root
* `PasswordAuthentication no` – запретить вход с паролем

Если они уже есть в таком виде, ничего делать не нужно.

После внесения правок, нужно перезагрузить конфигурацию ssh:

```sh
sudo systemctl reload ssh  # Перезагрузить конфигурацию ssh
```

## 7. Установить пакеты для прокси

Установить пакеты:

```sh
sudo apt install nginx libnginx-mod-stream squid certbot python3-certbot-nginx apache2-utils
```

Здесь:

* `nginx` – сервер для обработки входящих соединений, на нём будет терминироваться TLS, через него Let's Encrypt будет проверять и выдавать сертификаты
* `libnginx-mod-stream` – модуль для nginx, который позволяет пробрасывать соединения в squid в исходном виде
* `squid` – прокси-сервер, реализует ключевую логику
* `certbot` – приложение для работы с сертификатами Let's Encrypt
* `python3-certbot-nginx` – пакет для автоматизации выдачи сертификатов через nginx
* `apache2-utils` – пакет разных утилит для работы с Apache, нам нужен только для работы с хешами паролей

## 8. Настроить Squid

Нужно создать пользователя для аутентификации на прокси-сервере.

```sh
sudo htpasswd -c /etc/squid/users proxyuser
```

При необходимости использовать нескольких пользователей, нужно повторить этот шаг несколько раз.

Пароь желательно использовать максимально сильный, чтобы при брутфорсе его точно не подобрали.
Например, 32 символа с буквами в разном регистре, цифрами и спецсимволами.
Пароль должен генерироваться случайно. Генераторы паролей есть в менеджерах паролей или можно сгенерить тут: [genpas.narod.ru](https://genpas.narod.ru/).

Далее нужно сконфигурировать сам сервер:

```sh
sudo vim /etc/squid/squid.conf
```

Конфигурация должна быть такой:

```squid
http_port [::1]:3128
shutdown_lifetime 10 seconds
visible_hostname <my-domain>.ru

auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/users
auth_param basic realm Outworld

acl users proxy_auth REQUIRED
http_access allow users
http_access deny all

forwarded_for delete
via off
follow_x_forwarded_for deny all
request_header_access X-Forwarded-For deny all

cache deny all
cache_dir null /tmp
```

Далее нужно перезапустить squid:

```sh
sudo systemctl restart squid
```

## 9. Настроить Nginx

Тут будет несколько шагов.

### 9.1 Настроить локацию для статических файлов

Это нужно в первую очередь для работы с Let's Encrypt

```sh
sudo vim /etc/nginx/sites-enabled/default
```

Конфигурация default должна быть такой:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name <my-domain>.ru;

    location / {
        root /var/www/html;
    }
}
```

Тут certbot будет размещать файлы для идентификации домена,
чтобы выписывать сертификаты.

### 9.2 Настроить модуль проксирования соединений для HTTP

```sh
sudo vim /etc/nginx/nginx-squid-stream.conf
```

Конфигурация такая:

```nginx
stream {
    log_format proxy '$remote_addr [$time_local] '
                     '$protocol $status $bytes_sent $bytes_received '
                     '$session_time "$upstream_addr" '
                     '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';

    access_log /var/log/nginx/tcp-access.log proxy ;
    open_log_file_cache off;

    upstream _squid {
        server [::1]:3128;
    }

    server {
        listen 8080;

        proxy_pass _squid;

        include /etc/nginx/nginx-squid-https-include.conf*;
    }
}
```

```sh
sudo vim /etc/nginx/nginx.conf
```

Здесь нужно добавить строку:

```nginx
include /etc/nginx/nginx-squid-stream.conf;
```

Перезагрузить Nginx:

```sh
sudo systemctl reload nginx
```

### 9.3 Настроить Let's Encrypt

```sh
sudo certbot --nginx --domains "<my-domain>.ru" --non-interactive --agree-tos --https-port 8443 --email "<my-email>"
```

После успешного выполнения этого шага создадутся ключ и сертификат,
которые можно будет использовать в Nginx для HTTPS.

Также будет настроено автоматическое обновление сертификата.

### 9.4 Настроить модуль проксирования соединений для HTTPS

```sh
sudo vim /etc/nginx/nginx-squid-https-include.conf
```

В файл нужно прописать следующую конфигурацию:

```nginx
listen 443 ssl;

ssl_certificate /etc/letsencrypt/live/<my-domain>.ru/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/<my-domain>.ru/privkey.pem;
```

Перезагрузить Nginx:

```sh
sudo systemctl reload nginx
```

## 10. Настройка ротации логов

Все эти пакеты пишут логи, которые со временем могут занять всё место на диске.
Поэтому полезно поставить logrotate, которые подчищал бы всё это дело.

```sh
sudo apt install logrotate
```

Он автоматически привезёт конфиги для ротации основных логов, также туда добавляются конфиги для установленных пакетов.

Также нужно настроить ротацию системного журнала:

```sh
sudo vim /etc/systemd/journald.conf
```

В этом конфиге нужно раскомментировать, изменить или добавить следующие опции:

```ini
SystemMaxUse=500M
SystemMaxFileSize=100M
SystemMaxFiles=100
SystemKeepFree=100M
```

И выполнить перезапуск сервиса логирования:

```sh
sudo systemctl restart systemd-journald
```
