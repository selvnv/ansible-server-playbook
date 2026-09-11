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

- `acme` — nginx на порту 80, один на хост. Нужен для выпуска Let's Encrypt сертификатов.
- `web` — nginx на порту 443, один на хост. Содержит несколько виртуальных хостов (`selfplace.ru`, `unlogic.ru` и др.).

Структура каталогов инстанса:

```
/services/apps/nginx/<instance>/
|-- conf/
|   |-- nginx.conf
|   |-- common/                     # общие http/ssl-параметры
|   |   |-- http_params.conf
|   |   |-- ssl_params.conf
|   |   `-- mime.types
|   |-- conf.d/                     # server-блоки (по одному на vhost)
|   |   `-- <server_name>/<server_name>.conf
|   `-- ssl/
|       |-- dhparams.pem
|       |-- live/<server_name>/{fullchain.pem, privkey.pem}
|       `-- custom/<server_name>/{fullchain.pem, privkey.pem}
|-- static/<server_name>/...        # webroot по vhost
|-- var/nginx.pid
`-- logs -> /services/logs/nginx/<instance>/
```

Плюс systemd-юнит `/etc/systemd/system/nginx.<instance>.service`.

## Переменные для запуска

- `nginx_instance` — **обязательно**. Примеры: `acme`, `web`.
- `nginx_certbot_email` — обязательно при выпуске сертификата на домен через Let's Encrypt (нужно только для vhost'ов с `ssl.type: "letsencrypt"`; не нужно для `acme` и для vhost'ов с собственным сертификатом).
- Данные инстанса — в `roles/nginx/vars/<instance>.yml`: `nginx_instance_port` и `nginx_vhosts` (список vhost'ов: `server_name` + `ssl.type`, где `type` = `none` / `letsencrypt` / `custom`).
- Собственные сертификаты: `ssl.type: "custom"` + файлы `files/<host>/<instance>/ssl/<server_name>/{fullchain.pem,privkey.pem}`.
- Остальные роли используют значения по умолчанию (`defaults/`) и `group_vars`/`host_vars`; дополнительно ничего передавать не нужно.

## Запуск плейбука

При деплое инстансов nginx порядок важен: сначала `acme`, затем `web` (иначе на момент выпуска сертификата нет challenge-сервера).

```bash
# 1. Проверка синтаксиса
ansible-playbook --syntax-check playbooks/main.yml

# 2. ACME-инстанс (первым при первом деплое)
ansible-playbook -e nginx_instance=acme playbooks/main.yml

# 3. Web-инстанс (все vhost'ы из vars/web.yml, включая выпуск LE-сертификатов)
ansible-playbook -e nginx_instance=web -e nginx_certbot_email=admin@example.com playbooks/main.yml

# 4. Сухой прогон (без изменения хостов)
ansible-playbook --check -e nginx_instance=acme playbooks/main.yml
ansible-playbook --check -e nginx_instance=web playbooks/main.yml

# 5. Повторный запуск — идемпотентно; certbot продлевает сертификат при приближении к истечению
ansible-playbook -e nginx_instance=web playbooks/main.yml
```

При запуске под WSL Ansible может проигнорировать `ansible.cfg` в корне проекта. В этом случае нужно явно указать путь к конфигу в переменной окружения
```bash
export ANSIBLE_CONFIG=./ansible.cfg
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

- Создать ключ для подключения по SSH к хосту. Добавить приватную часть в `.ssh/` проекта, а публичную - в `authorized_keys` пользователя `ansible` на хосте

На хосте установить корректного владельца и права
```bash
sudo chown -R ansible:ansible /home/ansible/.ssh
sudo chmod 700 /home/ansible/.ssh
sudo chmod 600 /home/ansible/.ssh/authorized_keys
```

Перед запуском плейбука добавить сервер в `known_hosts`
```bash
ssh-keyscan -p <sshd_port> -H <server_ip> >> ~/.ssh/known_hosts
```

В случае с WSL могут возникнуть проблемы с правами доступа к SSH-ключу, особенно если ключ находится в файловой системе Windows (`/mnt/c`). В этом случае нужно скопировать приватный ключ в файловую систему WSL и установить корректные права на него:
```bash
cp .ssh/ansible_ed25519 ~/.ssh
sudo chmod 600 ~/.ssh/ansible_ed25519
```

- Указать пакеты, каталоги и правила — в `inventory/production/group_vars/all.yml`, `host_vars/<host>.yml`.


Реальные примеры запуска:
```bash
# Задать путь к конфигу через переменную окружения (для WSL)
export ANSIBLE_CONFIG=./ansible.cfg

# Базовая настройка, без nginx (по умолчанию)
ansible-playbook playbooks/main.yml

# ACME-инстанс для получения Let's Encrypt сертификатов на домены
ansible-playbook -e "playbook_role_nginx=true" -e "nginx_instance=acme" playbooks/main.yml

# Web-инстанс: все vhost'ы (selfplace.ru — letsencrypt, unlogic.ru — custom) из vars/web.yml
ansible-playbook -e "playbook_role_nginx=true" -e "nginx_instance=web" -e "nginx_certbot_email=<mail>" playbooks/main.yml
```
