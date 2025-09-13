# PHP-Nginx-Socket — учебный стек на Docker (современная замена XAMPP/MAMP/Open Server)

Простая, воспроизводимая и «говорящая» среда для изучения PHP и его экосистемы. Стек собирается из контейнеров Docker и предназначен для локальных экспериментов.

Важно: этот проект предназначен исключительно для обучения, практики и ознакомления. Не используйте его в проде.

## Что внутри (архитектура)

Сервисы docker-compose.yml:
- PHP-FPM 8.4 — выполняет PHP, слушает Unix-сокет /var/run/php/php-fpm.sock, Xdebug установлен и управляется через переменные окружения.
- Nginx — отдаёт статику и проксирует .php в PHP-FPM через Unix-сокет (fastcgi_pass unix:/var/run/php/php-fpm.sock); доступен на http://localhost:80.
- PostgreSQL — СУБД на localhost:5432, данные в именованном томе postgres-data.
- pgAdmin 4 — веб-интерфейс PostgreSQL на http://localhost:8080.

Здоровье (healthchecks):
- PHP-FPM — проверка наличия сокета и/или FastCGI-ответа через сокет.
- Nginx — HTTP-запрос к http://localhost/.
- PostgreSQL — pg_isready.
- pgAdmin — HTTP-запрос к http://localhost:8080/.

Порядок старта: Nginx ожидает, когда PHP-FPM станет healthy.

## Структура репозитория (актуальная)
```
php-nginx-unix/
├── Makefile
├── README.md
├── config/
│   ├── nginx/
│   │   └── nginx.conf          # Конфиг Nginx (fastcgi_pass unix:/var/run/php/php-fpm.sock)
│   └── php/
│       ├── php.ini             # Dev-настройки + Xdebug через env
│       └── pool.d/
│           └── www.conf        # PHP-FPM pool: listen=/var/run/php/php-fpm.sock и права на сокет
├── docker/
│   └── php.Dockerfile          # Образ PHP-FPM 8.4 (Alpine) + расширения + Xdebug + Composer
├── docker-compose.yml          # Основной стек: PHP-FPM (Unix-сокет), Nginx, PostgreSQL, pgAdmin
├── docker-compose.xdebug.yml   # Оверлей для включения Xdebug (mode=start)
├── docs/
│   └── AI-CONTEXT.md           # Контекст/гайдлайны для AI (вариант с Unix-сокетом)
├── env/
│   └── .env.example            # Пример переменных окружения (скопируйте в env/.env)
└── public/                     # DocumentRoot (монтируется в Nginx и PHP-FPM)
    ├── index.html
    ├── index.php
    └── phpinfo.php
```


Примечания:
- Сокет PHP-FPM хранится в общем именованном томе (например, php-fpm-sock), смонтированном в оба контейнера по пути /var/run/php.
- Папки src/ и logs/ не обязательны для учебных целей — достаточно public/.

## Быстрый старт

Предпосылки:
- Docker 20.10+
- Docker Compose v2+

Шаги:
1) Клонируйте репозиторий и перейдите в каталог проекта.
2) Подготовьте переменные окружения:
    - mkdir -p env && cp env/.env.example env/.env
    - при необходимости отредактируйте пароли/имена БД и учётку pgAdmin.
3) Запустите стек:
    - make up (или docker compose up -d)
4) Проверьте доступность:
    - Web: http://localhost
    - pgAdmin: http://localhost:8080
    - PostgreSQL: localhost:5432

Полезные команды Makefile:
- make setup — создать env/.env из примера
- make up / make down / make restart — управление стеком
- make logs / make status — логи и статусы контейнеров
- make xdebug-up / make xdebug-down — запуск/остановка стека с включённым Xdebug

## Конфигурация

PHP (config/php/php.ini):
- error_reporting=E_ALL, display_errors=On — удобно учиться на ошибках
- memory_limit=256M, upload_max_filesize=20M, post_max_size=20M
- opcache включён; validate_timestamps=1 (код подхватывается сразу)
- Xdebug управляется через переменные окружения (см. раздел ниже)

PHP-FPM (config/php/pool.d/www.conf):
- listen = /var/run/php/php-fpm.sock
- listen.owner = www-data; listen.group = www-data; listen.mode = 0660
- pm = dynamic (значения по умолчанию подходят для dev)
- Убедитесь, что /var/run/php существует в контейнере и смонтирован из именованного тома

Nginx (config/nginx/nginx.conf):
- Отдаёт статику из /var/www/html
- Проксирует .php через Unix-сокет: fastcgi_pass unix:/var/run/php/php-fpm.sock
- index включает index.php
- Корректные fastcgi_param для SCRIPT_FILENAME и DOCUMENT_ROOT
- client_max_body_size согласуйте с upload_max_filesize и post_max_size

Docker-образ PHP (docker/php.Dockerfile):
- База: php:8.4-fpm-alpine
- Расширения: pdo, pdo_pgsql, pgsql, mbstring, xml, gd, bcmath, zip
- Установлены Xdebug (pecl), Composer; при необходимости — cgi-fcgi (для healthcheck)
- Порт FPM наружу не публикуется — взаимодействие только через Unix-сокет

## Переменные окружения (env/.env)

Минимальный набор:
- POSTGRES_USER — имя пользователя PostgreSQL
- POSTGRES_PASSWORD — пароль пользователя
- POSTGRES_DB — имя создаваемой БД
- PGADMIN_DEFAULT_EMAIL — логин для входа в pgAdmin
- PGADMIN_DEFAULT_PASSWORD — пароль для входа в pgAdmin
- Переменные Xdebug (см. далее)

## Xdebug: как включить

По умолчанию Xdebug установлен, но выключен. Включить можно двумя способами:

Вариант A: оверлейный compose-файл
- make xdebug-up
  (эквивалент docker compose -f docker-compose.yml -f docker-compose.xdebug.yml up -d)
- Внутри оверлея задаются XDEBUG_MODE=debug и XDEBUG_START=yes.

Вариант B: через env/.env
- Пропишите:
    - XDEBUG_MODE=debug
    - XDEBUG_START=yes
- Затем перезапустите только php-контейнер:
    - docker compose up -d --no-deps php
- IDE подключается по Xdebug 3 на порт 9003; client_host=host.docker.internal.

Примечание: транспорт Xdebug не зависит от способа связи Nginx↔FPM (Unix-сокет или TCP).

## Рабочие директории и монтирование

- public/ монтируется в /var/www/html одновременно в PHP-FPM и Nginx — изменения видны сразу.
- Общий том для сокета PHP-FPM (например, php-fpm-sock) монтируется в оба контейнера по пути /var/run/php.
- Конфиги монтируются только на чтение.
- Для PostgreSQL используется именованный том postgres-data (персистентные данные).

## Подключение к PostgreSQL из PHP (пример)

```php
<?php
$host = 'postgres';   // имя сервиса БД в docker-compose
$port = 5432;
$dbname = 'your-db-name';
$user = 'your-user';
$pass = 'your-user-password';

$dsn = "pgsql:host=$host;port=$port;dbname=$dbname";
$pdo = new PDO($dsn, $user, $pass, [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
]);
```


## Healthchecks

Рекомендуемые проверки:
- PHP-FPM:
    - Быстрая проверка сокета:
        - test -S /var/run/php/php-fpm.sock
    - При наличии cgi-fcgi и настроенном ping.path:
        - cgi-fcgi -bind -connect /var/run/php/php-fpm.sock -request "SCRIPT_FILENAME=/ping&REQUEST_METHOD=GET"
- Nginx: curl -fsS http://localhost/ или wget -q -O- http://localhost/
- PostgreSQL: pg_isready -U $POSTGRES_USER -h 127.0.0.1
- pgAdmin: HTTP-запрос к http://localhost:8080/

depends_on: Nginx ждёт healthy PHP-FPM.

## Решение проблем

502/404 на .php:
- Убедитесь, что fastcgi_pass указывает на unix:/var/run/php/php-fpm.sock.
- Проверьте fastcgi_param:
    - SCRIPT_FILENAME $document_root$fastcgi_script_name
    - DOCUMENT_ROOT $document_root
- Пути /var/www/html в Nginx и PHP-FPM должны совпадать.

Nginx не может подключиться к сокету:
- Проверьте права: listen.owner, listen.group, listen.mode = 0660.
- Убедитесь, что Nginx и PHP-FPM видят один и тот же путь /var/run/php и он смонтирован из общего тома.
- Проверьте существование файла сокета: test -S /var/run/php/php-fpm.sock в обоих контейнерах.

Ограничения загрузки файлов:
- Согласуйте client_max_body_size (Nginx) с upload_max_filesize и post_max_size (PHP).

Порты заняты:
- Поменяйте привязку портов в docker-compose.yml, например 8081:80 для Nginx, 5433:5432 для PostgreSQL.

Контейнеры не стартуют по порядку:
- Проверьте healthchecks: docker compose ps; Nginx должен зависеть от healthy PHP-FPM.

Xdebug не подключается:
- Убедитесь, что IDE слушает порт 9003.
- Проверьте XDEBUG_MODE/START и XDEBUG_CLIENT_HOST=host.docker.internal.

## Дисклеймер

Проект создан для обучения и экспериментов с PHP-стеком. Не предназначен для production-использования или оценки производительности.

—  
Для архитектурной шпаргалки и договорённостей см. docs/AI-CONTEXT.md (вариант с Unix-сокетом).