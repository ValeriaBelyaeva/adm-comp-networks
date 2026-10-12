# ЛР 2. Ansible + Caddy

**Задача:** установить Ansible и с его помощью развернуть веб-сервер Caddy.

## Состав
| Файл | Что это |
|---|---|
| `ansible.cfg`, `inventory/hosts` | Базовый конфиг и хосты |
| `file_playbook.yml` | Задание 1: создание → проверка → изменение → удаление файла |
| `caddy_deploy.yml` + `roles/caddy_deploy/` | Роль: установка Caddy, своя страница, шаблон Caddyfile |

## Ход работы

### Часть 1. Ansible
```bash
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py && python3 get-pip.py
python3 -m pip install ansible
ansible my_servers -m ping -c local
ansible my_servers -m setup -c local
```
![](screenshots/01-ping.png)

Ad-hoc команды:
```bash
ansible my_servers -c local -m shell -a 'echo test_file_content > $HOME/test.txt'
ansible my_servers -c local -m file -a 'path=$HOME/test.txt state=absent'
```
![](screenshots/02-adhoc.png)

### Часть 2. Установка Caddy
Роль создана через `ansible-galaxy init caddy_deploy`, шаги — в `roles/caddy_deploy/tasks/main.yml`.
```bash
ansible-playbook caddy_deploy.yml
systemctl status caddy
```
![](screenshots/03-caddy-install.png)

### Часть 3. Домен и Caddyfile
1. Регистрируем домен на duckdns.org на IP сервера, прописываем его в `roles/caddy_deploy/vars/main.yml` (`domain_name`).
2. Caddyfile генерируется из шаблона `templates/Caddyfile.j2`, после чего сервис перезагружается.
3. Открываем домен в браузере — страница отдаётся по HTTPS с сертификатом Let's Encrypt.
   ![](screenshots/04-https.png)

## Задание

### 1. Ad-hoc → плейбук
`file_playbook.yml`: создаёт файл, выводит содержимое, **меняет содержимое**, выводит снова, удаляет.
```bash
ansible-playbook file_playbook.yml
```
![](screenshots/05-file-playbook.png)

### 2a. Расширение конфига Caddy
- Своя страница `index.html` (Hello world) кладётся в `/var/www/html` отдельными шагами роли, `root` в шаблоне берётся из переменной `site_root`.
- Добавлен блок `header`: свой заголовок `X-Team`, защитные `X-Content-Type-Options`, `X-Frame-Options`, убран `Server`.

Проверка:
```bash
curl -I https://<домен>
```
![](screenshots/06-hello-world.png)
![](screenshots/07-headers.png)
