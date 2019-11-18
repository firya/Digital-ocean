# Настройка нескольких сайтов в разных контейнерах с одним nginx-proxy

> Суть этот туториала в том чтобы поднять отдельный контейнер с nginx-proxy, без спецефических параметров для каждого отдельного сайта и добавлять проекты постепенно с переменными окружения, при помощи которых будт автоматически подключаться домен и запрашиваться ssl сертификат через lets-encrypt

Для настройки понадобится удаленный сервер с ubuntu и установленными docker и docker-compose. Я настраивал на [digitalocean.com](https://m.do.co/c/ac2598e4bfc2). Первоначальную настройку dropleta можно сделать по инструкциям:

- [Initial Server Setup with Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04)
- [How To Install and Use Docker on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-18-04)
- [How To Install Docker Compose on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-install-docker-compose-on-ubuntu-18-04)

## Устновка единой Nginx-proxy для сайтов

После настройки сервера и установка docker и docker-compose можем приступать к настройке nnginx-proxy. В качестве образа будем исполдьзовать уже готовый jwilder/nginx-proxy и jrcs/letsencrypt-nginx-proxy-companion для создания сертифкатов (мы же не хотим делать сайт без https в 21 веке?).

Создадим папку nginx-proxy и перейдем в нее:

`mkdir nginx-proxy; cd nginx-proxy`

А так же создадим сеть в которой между собой будут общаться все контейнеры, и назовем ее так же `nginx-proxy`:

`docker network create nginx-proxy`

Внутри папки нам понядобятся 2 файла `proxy_settings.conf` и собственно `docker-compose.yml`. Создадим их при помощи консольного редактора `nano` предустановленного в Ubuntu.

> Небольшая справка по nano. Для того чтобы открыть или создать файл нужно написать в консоли `nano filename.ext`, где `filename` — это имя файла, а `ext` — его расширение. Проще всего редактировать файлы в нормальном текстовом редакторе, и вставлять содержимое. Для выхода из nano нужно нажать `ctrl+x`, потом `y` или `n` для сохранения или не сохранения изменений, а затем `enter`. Если у вас уже есть файл, но вам нужно внести много изменений можно сделать это в любом текстовом редакторе, а затем удалить файл и создать его заного `rm filename.ext; nano filename.ext`.

В файле `proxy_settings.conf` будут лежать параметры nginx, например максимально доступый размер файла:

`nano proxy_settings.conf`
`client_max_body_size 30m;`

а в файле `docker-compose.yml` настройки для сборки контейнера:

`nano docker-compose.yml`
```yml
version: '3.7'

networks:
  default:
    external:
      name: nginx-proxy

services:
  nginx-proxy:
    container_name: nginx-proxy
    image: jwilder/nginx-proxy
    restart: always
    ports:
      - 80:80
      - 443:443
    volumes:
      - certs:/etc/nginx/certs
      - vhost.d:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - ./proxy_settings.conf:/etc/nginx/conf.d/proxy_settings.conf

  nginx-proxy-letsencrypt:
    container_name: nginx-proxy-letsencrypt
    image: jrcs/letsencrypt-nginx-proxy-companion
    restart: always
    volumes: 
      - certs:/etc/nginx/certs
      - vhost.d:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - NGINX_PROXY_CONTAINER=nginx-proxy

volumes:
  certs:
  vhost.d:
  html:
```

Как видите в самом nginx не хранится никаких настроек доменов, мы будем их передавать в параметрах environment каждого контейнера.
Далее нужно поднять контейнер с nginx-proxy:

`docker-compose up -d`

где `-d` — это ключ означающий поднятие контейнера в фоне.

## Создадим сайт на wordpress используя docker-compose

Выйдем в корень папки пользователя и создадим папку `wordpress`:

```
cd
mkdir wordpress
cd wordpress
```

Для этого нам понадобится поднять 2 контейнера, один из них с самим Wordpress'ом, а второй с MySQL.

Создадим файл с параметрами окружения `.env`:

`nano .env`

```
MYSQL_DATABASE=wordpress
MYSQL_ROOT_PASSWORD=password
MYSQL_USER=wordpress
MYSQL_PASSWORD=password
VIRTUAL_HOST=example.com,www.example.com
LETSENCRYPT_HOST=example.com
LETSENCRYPT_EMAIL=youremail@domain.tld
```

Эти параметры будут использоваться в `docker-compose.yml`. С их помощью можно разделить разработку на локальной машине и в продакшене. Названия переменных говорят сами за яебя, поэтому не буду расписывать подробно, скажу лишь что `VIRTUAL_HOST` должен быть привязан к серверу, а `LETSENCRYPT_EMAIL` рабочим e-mail адресом.

Теперь создадим файл с настройками для `php.ini`:

`nano uploads.conf`
```
file_uploads = On
memory_limit = 500M
upload_max_filesize = 30M
post_max_size = 30M
max_execution_time = 600
```

Разрешение файла не играет роли, а такое название выбрано потому что в файле мы меняем только настройки лимитов по загружаемым файлам для wordpress.

Создадим docker-compose.yml:

`nano docker-compose.yml`

```yml
version: "3.7"

services:
  db:
    image: mysql:5.7
    container_name: db
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      - MYSQL_DATABASE=wordpress
      - MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD
      - MYSQL_USER=$MYSQL_USER
      - MYSQL_PASSWORD=$MYSQL_PASSWORD

  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    container_name: wordpress
    expose:
      - 80
      - 443
    volumes:
      - ./config/uploads.conf:/usr/local/etc/php/conf.d/uploads.ini
      - ./theme:/var/www/html/wp-content/themes/yourtheme
    restart: always
    environment:
      - VIRTUAL_HOST=$VIRTUAL_HOST
      - LETSENCRYPT_HOST=$LETSENCRYPT_HOST
      - LETSENCRYPT_EMAIL=$LETSENCRYPT_EMAIL
      - WORDPRESS_DB_HOST=db:3306
      - WORDPRESS_DB_USER=$MYSQL_USER
      - WORDPRESS_DB_PASSWORD=$MYSQL_PASSWORD

volumes:
  db_data:

networks:
  default:
    external:
      name: nginx-proxy
```

В этом файле нужно обратить внимение на несколько моментов:

* `container_name` для базы данных и wordpress, если вы собираетесь поднять несколько сайтов на одном хостинге позаботтесь, чтобы имена не повторялись
* все параметры в `environment` подставляются из файла `.env` котоыйр мы создали ранее
* в контейнере wordpress в разделе volumes:
  * `./config/uploads.conf:/usr/local/etc/php/conf.d/uploads.ini` - копирует файл с настройками для php.ini
  * `- ./theme:/var/www/html/wp-content/themes/yourtheme` - Связывает содержимое папки `theme` с папкой `yourtheme` внутри вордпресса для создание собственной темы (так же по аналогии вы можете создать другие папки и привязать их внутрь контейнера wordpress, например для плагинов)

Теперь создадим папку с пустой темой для wordpress'а, вы можете пропустить этот шаг:

```
mkdir theme
cd theme
nano 'index.php'
```

```html
<html>
<head>
    <?php wp_head(); ?>
</head>
<body>
    <!-- your content here -->

    <?php wp_footer(); ?>
</body>
</html>
```

`nano style.css`

```css
/*   
Theme Name: Your empty theme
Description: Your empty theme
Author: Your name
Version: 1.0
*/
```

Теперь можем поднять получившийся контейнер:

`docker-compose up -d`

Контейнер создан, теперь нужно разрешить вордпрессу создавать и удалять файлы, без этого не поолучится установить/удалить плагины, темы и файлы. Зайдем внутрь запущенного контейнера:

`docker exec -it maksimlebedev /bin/bash`

Изменим пользователя владельца папки `/var/www/` внутри контейнера:

`chown -R www-data:www-data /var/www/`

Выйдем из контейнера командой `exit`.

## Создадим контейнер с php для запуска простых сайтов

Выйдем в корень папки пользователя и создадим папку `simple_site`:

```
cd
mkdir simple_site
cd simple_site
```

Так же как и в случае с wordpress создадим файл переменных окружения `.enc`:

`nano .env`

```
VIRTUAL_HOST=example.com,www.example.com
LETSENCRYPT_HOST=example.com
LETSENCRYPT_EMAIL=youremail@domain.tld
```

В этот раз нам не понадобится база данных, поэтому перменных получится меньше
Теперь создадим `docker-compose.yml`:

```yml
version: '3.7'

services:
  php:
    image: php:7.2-apache
    container_name: php
    expose:
      - 80
      - 443
    volumes:
      - ./public:/var/www/html
    restart: always
    environment:
      - VIRTUAL_HOST=$VIRTUAL_HOST
      - LETSENCRYPT_HOST=$LETSENCRYPT_HOST
      - LETSENCRYPT_EMAIL=$LETSENCRYPT_EMAIL

networks:
  default:
    external:
      name: nginx-proxy
```

Стоит так же помнить, что нельзя создавать несколько контейнеров с одним именем.
В этом контейнере монтируется папка `public`, все файлы находящиеся в ней будут находиться в корне сайта в контейнере, поэтому давайте создадим и ее:

`mkdir public`

И поднимем этот контейнер:

`docker-compose up -d`