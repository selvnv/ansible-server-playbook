# Ansible playbook для развёртывания веб-сервера

Плейбук настраивает хост (Debian/Ubuntu): пакеты, SSH, файрвол и nginx, в т.ч. выпуск
сертификатов Let's Encrypt через отдельный acme-инстанс.

## Роли

| Роль | Что делает |
|---|---|
| `common` | Базовая настройка хоста: создание групп, пользователей, базовых каталогов и т.д. |
| `packages` | Установка / удаление / обновление apt-пакетов |
| `ssh` | Настройка конфигурации `ssh` |
| `firewall` | Настройка правил `ufw` |
| `nginx` | Развёртывание nginx-инстанса: создание структуры каталогов, выпуск сертификатов, размещение статики и т.д. |

## Роль `nginx`

Один прогон роли = развертывание одного инстанса. Имя инстанса задаётся переменной `nginx_instance`.

Типовые инстансы:

- `acme` — nginx на порту 80, один на хост. Нужен для выпуска  Let's Encrypt сертификатов.
- `<server_name>` — виртуальный хост на порту 443 (сертификат — через certbot или собственный).

Структура каталогов инстанса:

```
/services
|-- apps
|   `-- nginx/<instance>
|       |-- static/                 # корень сайта / webroot acme
|       |-- conf/
|       |   |-- nginx.conf
|       |   |-- mime.types
|       |   |-- common/             # общие для виртхостов http/ssl-параметры
|       |   |-- conf.d/             # server-блоки: 80.conf, 443.conf
|       |   `-- ssl/                # сертификаты
|       `-- var/                    # pid и пр.
`-- logs
    `-- nginx/<instance>            # error.log, access.log
```

Плюс systemd-юнит `/etc/systemd/system/nginx.<instance>.service`.

## Переменные для запуска

- `nginx_instance` — **обязательно**. Примеры: `acme`, `<server_name>`.
- `nginx_certbot_email` — обязательно при выпуске сертификата на домен через Let's Encrypt (не нужно для `acme` инстанса и не нужно, если используется собственный сертификат: `nginx_use_custom_cert: true`).
- Данные инстанса — в `roles/nginx/vars/<instance>.yml` (`nginx_server_name`, `nginx_include_conf_list`, переопределение дефолтных переменных `nginx_issue_certificates`, `nginx_use_custom_cert` и др.).
- Собственные сертификаты: `nginx_use_custom_cert: true` + файлы `files/<host>/<instance>/ssl/{cert.pem,key.pem}`.
- Остальные роли используют значения по умолчанию (`defaults/`) и `group_vars`/`host_vars`; дополнительно ничего передавать не нужно.

## Запуск плейбука

При деплое инстансов nginx порядок важен: сначала `acme`, затем другие инстансы (иначе на момент выпуска нет challenge-сервера).

```bash
# 1. Проверка синтаксиса
ansible-playbook --syntax-check playbooks/main.yml

# 2. ACME-инстанс (первым при первом деплое)
ansible-playbook -e nginx_instance=acme playbooks/main.yml

# 3. Сайт-инстанс с выпуском сертификата
ansible-playbook -e nginx_instance=<server_name> -e nginx_certbot_email=admin@example.com playbooks/main.yml

# 4. Сайт-инстанс с собственным сертификатом (без certbot)
ansible-playbook -e nginx_instance=<server_name> -e nginx_use_custom_cert=true playbooks/main.yml

# 5. Сухой прогон (без изменения хостов)
ansible-playbook --check -e nginx_instance=acme playbooks/main.yml
ansible-playbook --check -e nginx_instance=<server_name> playbooks/main.yml

# 6. Повторный запуск — идемпотентно; certbot продлевает сертификат при приближении к истечению
ansible-playbook -e nginx_instance=<server_name> playbooks/main.yml
```

Перед запуском:

- A-запись домена указывает на хост (нужно для получения сертификата на домен).
- Доступ по SSH (указать пользователя и порт в `inventory/production/hosts`, под которыми к хосту подключается Ansible) и настроить sudo без пароля для пользователя, под которым Ansible подключается к хосту (для работы `become: true` в плейбуке).

```bash
sudo useradd -m -s /bin/bash -c "User for setup host via Ansible" -G sudo ansible
```

```bash
sudo visudo
# Добавить ansible ALL=(ALL:ALL) NOPASSWD: ALL
```

- Указать пакеты, каталоги и правила — в `inventory/production/group_vars/all.yml`, `host_vars/<host>.yml`.
