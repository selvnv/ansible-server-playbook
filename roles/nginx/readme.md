#### Целевая структура каталогов

```
/services/apps/nginx/<instance>/
├── conf/
│   ├── nginx.conf                 # главный конфиг: user, pid, events, http { include ... }
│   ├── mime.types                 # общий для инстанса
│   ├── common/                    # общие переиспользуемые параметры (опционально для vhost)
│   │   ├── http_params.conf
│   │   └── ssl_params.conf
│   ├── conf.d/                    # server-блоки
│   │   ├── selfplace.ru
│   │   │   └── selfplace.ru.conf
│   │   ├── example.com
│   │   │   └── example.com.conf
│   │   └── unlogic.ru
│   │       └── unlogic.ru.conf
│   └── ssl/
│       ├── dhparams.pem           # Общий на инстанс (для всех вирт. хостов)
│       ├── live/                  # Let's Encrypt (certbot --config-dir)
│       │   ├── selfplace.ru/{fullchain.pem, privkey.pem}
│       │   └── example.com/{fullchain.pem, privkey.pem}
│       └── custom/                # сторонние сертификаты (с контроллера)
│           └── unlogic.ru/{fullchain.pem, privkey.pem}
│
├── logs/                          # симлинк на /services/logs/nginx/<instance>/
│
├── static/                        # webroot: по подкаталогу на vhost
│   ├── selfplace.ru/
│   │   └── index.html
│   └── unlogic.ru/
│       └── index.html
└── var/
    └── nginx.pid
```

#### Деплой

При запуске плейбука, для выполнения роли `nginx` необходимо передать значение переменной 
- `nginx_instance`
- `nginx_certbot_email`

#### Механизм ACME для выпуска и продления срока действия SSL/TLS сертификата

ACME (Automatic Certificate Management Environment) — это протокол, разработанный Let's Encrypt для автоматической выдачи и управления SSL-сертификатами


Допустим, мы хотим выпустить сертификат на домен `example.com`
- Certbot генерирует уникальный токен (случайную строку) и отправляет запрос в Let's Encrypt.

- Let's Encrypt говорит: «Помести файл по адресу: `http://example.com/.well-known/acme-challenge/<токен>` с содержимым: `<токен>.<отпечаток_ключа>»`

- Certbot создает этот файл в указанной директории на вашем сервере.

- Let's Encrypt отправляет HTTP-запрос на этот URL и проверяет, совпадает ли содержимое.

- Если всё совпало — выдача сертификата подтверждена.

#### Команда выпуска сертификата

```bash
certbot certonly --webroot \
    --config-dir /services/apps/nginx/<nginx_server_name>/conf/ssl \
    -w "<nginx_acme_root>" \
    -d "<nginx_server_name>" \
    -m "<nginx_certbot_email>" \
    --non-interactive \
    --agree-tos
```

| Фрагмент |	Значение |	Ваше значение |
|---|---|---|
| `certbot certonly`	| Подкоманда «только получить сертификат», без автоматической настройки конфигов веб-сервера |	— |
| `--webroot`	| Метод проверки домена. certbot пишет файл-токен в специальный каталог (задан ключом `-w`) и ждёт, что работающий веб-сервер отдаст его по HTTP |	— |
| `--config-dir`	| Куда certbot складывает результат своей работы, в т.ч. полученные сертификаты. |	`/services/apps/nginx/<nginx_server_name>/conf/ssl` |
| `-w <nginx_acme_root>` |	Каталог, в котором certbot размещает challenge-файлы: `<nginx_acme_root>/.well-known/acme-challenge/<token>`. Должен совпадать с тем, что отдаёт инстанс nginx, который обрабатывает ACME-запрос (в данном случае под все ACME-запросы запущен отдельный nginx инстанс) |	`/services/apps/nginx/<nginx_acme_instance>/static` |
| `-d <nginx_server_name>` |	Домен, на который выпускается сертификат |	`<nginx_server_name>` |
| `-m <email>` |	E-mail для аккаунта Let's Encrypt: сюда приходят уведомления об истечении, продлении сертификата |	<ваш e-mail> |
| `--non-interactive` |	Не задавать интерактивных вопросов — при любой проблеме просто падать с ошибкой (нужно для автономного Ansible) |	— |
| `--agree-tos` |	Согласие с Subscriber Agreement Let's Encrypt (обязательно) |	— |

#### Команда перевыпуска сертификата

```bash
certbot renew --config-dir {{ nginx_ssl }} --quiet --deploy-hook "systemctl reload nginx.{{ nginx_instance }}"
```

| Фрагмент команды | Значение |
|---|---|
| `certbot renew` |	Подкоманда продления: проверяет все сертификаты и продлевает только те, у которых истекает срок действия (обычно за 30 дней до истечения) |
| `--config-dir {{ nginx_ssl }}` | Путь к каталогу, в котором находятся артефакты `certbot`, подлежащие продлению renewal-конфиги, `live/`, `archive/`, `accounts/`. По умолчанию `certbot` смотрит в `/etc/letsencrypt`, но в данном случае кастомный путь. Без этого флага сертификаты для продления он не найдёт |
| `--quiet` |	Выводить в лог только ошибки |
| `--deploy-hook "systemctl reload nginx.{{ nginx_instance }}"` |	Команда, которую `certbot` выполняет после успешного продления. Здесь это мягкий `reload` инстанса nginx: nginx перечитывает конфиг и заново открывает файлы сертификата |

Почему reload, а не restart: systemctl reload отправляет мастер-процессу nginx сигнал перечитать конфиг/сертификаты без обрыва соединений — для обновления сертификата этого достаточно и это не создаёт даунтайма.
