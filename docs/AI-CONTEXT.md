AI-CONTEXT — руководящие принципы для AI Assistant и Junie (проект PHP-FPM + Nginx по Unix-сокету + PostgreSQL + pgAdmin)

1) Назначение документа
- Зафиксировать цели и рамки проекта.
- Синхронизировать подходы AI Assistant и Junie, чтобы не сбиваться с курса.
- Служить источником правды при принятии технических решений в стеке Nginx ↔ PHP-FPM по Unix-сокету.

2) Цель проекта
- Собрать учебный стек на Docker: Nginx (reverse proxy и статика), PHP-FPM 8.4 (FastCGI по Unix-сокету), PostgreSQL, pgAdmin.
- Сохранить удобство dev-опыта: Makefile, Xdebug-оверлей, healthchecks, «говорящие» конфиги и простая документация.

3) Итоговая архитектура (целевое состояние)
- PHP-FPM:
    - Выполняет PHP.
    - Слушает Unix-сокет: /var/run/php/php-fpm.sock.
    - Сокет хранится в общем томе, доступном Nginx и PHP-FPM.
- Nginx:
    - Отдаёт статику из DocumentRoot.
    - Проксирует .php в PHP-FPM через Unix-сокет (fastcgi_pass unix:/var/run/php/php-fpm.sock).
- PostgreSQL и pgAdmin:
    - PostgreSQL доступен на 5432, данные в именованном томе.
    - pgAdmin — веб-интерфейс для управления PostgreSQL.
- Внешние порты:
    - Nginx: 80 → localhost:80.
    - pgAdmin: 80 контейнера → localhost:8080.
    - PostgreSQL: 5432 → localhost:5432.

4) Ключевые решения и ограничения
- Соединение Nginx ↔ PHP-FPM: строго через Unix-сокет, доступный обоим контейнерам по одному и тому же пути.
- Путь сокета согласован: /var/run/php/php-fpm.sock.
- Простота и обучаемость в приоритете над прод-хардненингом.
- Проект не для продакшена; безопасности достаточно на уровне локальной среды.

5) Правила для AI Assistant и Junie
- Язык общения и документации: русский.
- Бэкенд-стек: PHP 8.4 (FPM), Nginx (stable), PostgreSQL (актуальная LTS/стабильная, например 16), pgAdmin 4, Docker Compose v2+.
- Не менять структуру проекта без отдельного обсуждения.
- Предпочитать минимальные и точечные изменения с чёткими комментариями и контекстом.
- Для правок кода/конфигов использовать формат фрагментов с контекстом и пометкой «// ... existing code ...».
- Не превращать проект в прод: без сложных CI/CD и enterprise-практик.

6) Именование и структура репозитория
- Имя репозитория (предложение): php-nginx-unix-socket.
- Базовая структура:
    - config/
        - nginx/nginx.conf (fastcgi_pass unix:/var/run/php/php-fpm.sock)
        - php/php.ini (dev-настройки, Xdebug через env)
        - php/pool.d/www.conf (listen = /var/run/php/php-fpm.sock; права на сокет)
    - docker/
        - php.Dockerfile (база для php-fpm)
    - env/
        - .env.example (переменные PostgreSQL, pgAdmin, Xdebug и т.д.)
    - public/ (DocumentRoot)
    - docker-compose.yml (Nginx, PHP-FPM, PostgreSQL, pgAdmin, общий том для сокета)
    - docker-compose.xdebug.yml (оверлей для Xdebug)
    - Makefile (команды управления стеком)
    - README.md (инструкции по запуску и конфигурации)
    - docs/AI-CONTEXT.md (этот файл)

7) План реализации (пошагово)
- Шаг 1: Инициализировать репозиторий и базовую структуру (см. выше).
- Шаг 2: Настроить PHP-FPM:
    - listen = /var/run/php/php-fpm.sock
    - listen.owner = www-data; listen.group = www-data (или группа nginx, если требуется), listen.mode = 0660
    - pm = dynamic/static по умолчанию для dev.
- Шаг 3: Настроить Nginx:
    - Статика: root /var/www/html.
    - .php: fastcgi_pass unix:/var/run/php/php-fpm.sock; корректные fastcgi_param SCRIPT_FILENAME и DOCUMENT_ROOT.
    - Включить index.php в index.
- Шаг 4: Общий том для сокета:
    - Создать именованный том (например, php-fpm-sock) и смонтировать его в оба контейнера по одному и тому же пути /var/run/php.
- Шаг 5: Настроить PostgreSQL и том для данных.
- Шаг 6: Настроить pgAdmin:
    - Публиковать на 8080.
    - ENV: PGADMIN_DEFAULT_EMAIL, PGADMIN_DEFAULT_PASSWORD.
- Шаг 7: Healthchecks:
    - PHP-FPM: проверка наличия сокета и ответа FPM (см. пункт 12).
    - Nginx: HTTP GET http://localhost/.
    - PostgreSQL: pg_isready -U $POSTGRES_USER -h 127.0.0.1.
    - pgAdmin: HTTP GET http://localhost:8080.
- Шаг 8: depends_on:
    - Nginx ждёт healthy PHP-FPM.
    - pgAdmin может стартовать независимо; подключение к БД задаётся вручную из UI.
- Шаг 9: Запуск и проверка:
    - make up → http://localhost, phpinfo, тест .php.
    - Проверить логи и здоровье контейнеров.
- Шаг 10: README.md:
    - Описать архитектуру, переменные окружения, быстрый старт.
- Шаг 11: Xdebug:
    - Проверить подключение к IDE (порт 9003).

8) Требования к конфигурации PHP-FPM (целевое)
- listen: /var/run/php/php-fpm.sock (внутри контейнера).
- listen.owner = www-data; listen.group = www-data (или nginx); listen.mode = 0660.
- user/group: www-data (или по умолчанию образа).
- Разрешить нужные расширения: pdo, pdo_pgsql, pgsql, mbstring, xml, gd, bcmath, zip и т.п.
- Папка для сокета (/var/run/php) должна существовать и быть доступной по правам/группе.
- Не публиковать никаких портов PHP-FPM наружу; взаимодействие — только через сокет.

9) Требования к конфигурации Nginx (целевое)
- root: /var/www/html.
- index: index.php index.html.
- fastcgi_pass: unix:/var/run/php/php-fpm.sock.
- Обязательные fastcgi_param:
    - SCRIPT_FILENAME $document_root$fastcgi_script_name
    - DOCUMENT_ROOT $document_root
- Включить стандартные fastcgi_params и fastcgi_buffers/timeout для dev по умолчанию.
- client_max_body_size согласовать с PHP upload_max_filesize/post_max_size.

10) PostgreSQL и pgAdmin
- PostgreSQL:
    - Порт 5432 → localhost:5432.
    - Данные: именованный том (например, postgres-data).
    - ENV: POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB.
- pgAdmin:
    - Порт 80 контейнера → localhost:8080.
    - ENV: PGADMIN_DEFAULT_EMAIL, PGADMIN_DEFAULT_PASSWORD.
    - Подключение к БД — через UI: host=postgres, port=5432.

11) Общие тома и монтирование
- Код (public/) монтируется в Nginx и PHP-FPM по одному и тому же пути (обычно /var/www/html) — изменения видны сразу.
- Сокет PHP-FPM:
    - Именованный том (например, php-fpm-sock) монтируется в оба контейнера в /var/run/php.
    - Путь сокета в обоих контейнерах одинаков: /var/run/php/php-fpm.sock.
- Конфиги монтируются только на чтение.
- Данные PostgreSQL — в отдельном томе (persist).

12) Healthchecks (целевое)
- PHP-FPM:
    - Быстрый тест существования сокета: test -S /var/run/php/php-fpm.sock.
    - При наличии cgi-fcgi можно выполнить: cgi-fcgi -bind -connect /var/run/php/php-fpm.sock -request "SCRIPT_FILENAME=/ping&REQUEST_METHOD=GET" при настроенном ping.path (опционально).
- Nginx: curl -fsS http://localhost/ или wget -q -O- http://localhost/.
- PostgreSQL: pg_isready -U $POSTGRES_USER -h 127.0.0.1.
- pgAdmin: HTTP-запрос к http://localhost:8080/.
- depends_on: Nginx ждёт healthy PHP-FPM.

13) Makefile и команды
- Сохранить привычные команды: setup, up, down, restart, logs, status.
- xdebug-up / xdebug-down — запуск с оверлеем Xdebug.
- clean / clean-all — удаление контейнеров/томов (с предупреждениями).

14) Xdebug (заметки)
- Версия Xdebug 3; порт клиента IDE — 9003.
- Управление через ENV (XDEBUG_MODE, XDEBUG_START, XDEBUG_CLIENT_HOST=host.docker.internal).
- Транспорт Xdebug не зависит от способа связи Nginx↔FPM (Unix-сокет vs TCP).

15) Тестирование и критерии приёмки
- http://localhost открывается; статика и .php отдаются.
- phpinfo() отражает fpm и xdebug (если включён).
- Логи Nginx и PHP-FPM без критических ошибок.
- Healthchecks всех сервисов OK.
- pgAdmin доступен на http://localhost:8080; можно подключиться к PostgreSQL (host=postgres).
- В public/ изменения видны сразу.

16) Не-цели
- Продакшен-хардненинг, SSL/TLS, балансировка, CDN.
- Сложная оркестрация (Swarm/Kubernetes).
- Тюнинг производительности под высокую нагрузку.

17) Безопасность (в рамках учебного проекта)
- Не хранить реальные секреты в репозитории.
- .env — локально; .env.example — шаблон без секретов.
- Порты открывать только локально; не публиковать наружу без необходимости.
- Пароли в примерах — слабые и для dev, меняйте локально.

18) Версионирование и совместимость
- PHP 8.4, Nginx stable, PostgreSQL 16 (или актуальная стабильная), pgAdmin 4.
- Docker Compose v2+.
- По возможности использовать alpine-образы, держать образы компактными.
- Согласованность путей между Nginx и PHP-FPM (DOCUMENT_ROOT, SCRIPT_FILENAME, путь к сокету).

19) Потенциальные риски и их обработка
- Несовпадение пути сокета между контейнерами → 502 на .php.
    - Согласовать путь и общий том: /var/run/php/php-fpm.sock в обоих контейнерах.
- Права на сокет недостаточны → Nginx не может подключиться.
    - Настроить listen.owner, listen.group, listen.mode = 0660, согласовать группы процессов (www-data/nginx) или использовать chgrp на точку монтирования.
- Неправильный SCRIPT_FILENAME/DOCUMENT_ROOT → 404/502 на .php.
    - Сверить fastcgi_param и пути в обоих контейнерах.
- Несоответствие client_max_body_size и upload_max_filesize/post_max_size → ошибки загрузки.
    - Согласовать значения в Nginx и PHP.
- pgAdmin не подключается к БД:
    - Проверить сеть docker, host=postgres, port=5432, креды env.
- Xdebug не подключается:
    - Проверить порт 9003, XDEBUG_MODE/START, client_host=host.docker.internal.

20) Что подготовить перед реализацией
- Утвердить пути: DocumentRoot=/var/www/html одинаково в Nginx и PHP-FPM.
- Убедиться, что fastcgi_pass будет указывать на Unix-сокет: unix:/var/run/php/php-fpm.sock.
- Определить минимальный набор PHP-расширений (включая pdo_pgsql/pgsql).
- Определить переменные окружения в .env.example:
    - POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB
    - PGADMIN_DEFAULT_EMAIL, PGADMIN_DEFAULT_PASSWORD
    - XDEBUG_MODE, XDEBUG_START, XDEBUG_CLIENT_HOST
- Спланировать общий том для сокета (php-fpm-sock) и права доступа к нему.
- Спланировать healthchecks и depends_on.

Конец документа.

Если нужно, подготовлю заготовки docker-compose.yml, nginx.conf и pool www.conf с правильными путями и правами под этот вариант.