### Роли, которые используются

- Установка и удаление пакетов: `ansible.builtin.apt`
- Управление сервисами: `ansible.builtin.systemd`
- Управление файлами и каталогами на хосте: `ansible.builtin.file`
- Рендер файлов конфигурации (для шаблонизации): `ansible.builtin.template`
- Копирование файлов на хост: `ansible.builtin.copy`
- Управление пользователями: `ansible.builtin.user`
- Настройка фаервола (`ufw`): `community.general.ufw`
- Выполнение специфических команд: `ansible.builtin.command` или `ansible.builtin.shell`

### Механизмы Ansible

---

Получить результат задачи (`register`):
```yaml
- name: Check nginx
  ansible.builtin.command:
    cmd: nginx -t
    register: nginx_test
```

Использование:
```yaml
- debug:
    var: nginx_test
```

---

Условия (`when`)

```yaml
- name: Install Docker
  ansible.builtin.apt:
    name: docker.io
    state: present
  when: ansible_os_family == "Debian"
```

---

Циклы (`loop`)

```yaml
- name: Create site directories
  ansible.builtin.file:
    path: "{{ item.root }}"
    state: directory
  loop: "{{ sites }}"
```

---

Перезапуск сервисов после изменения конфигурации (`notify` / `handlers`):
```yaml
- name: Deploy nginx config
  ansible.builtin.template:
    src: site.conf.j2
    dest: /etc/nginx/sites-enabled/example.com.conf
  notify: Reload nginx
```
