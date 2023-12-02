# Отчет по лабораторной работе №3 "Развертывание Netbox, сеть связи как источник правды в системе технического учета Netbox"
University: [ITMO University](https://itmo.ru/ru/)

Faculty: [FICT](https://fict.itmo.ru)

Course: [Network Programming](https://itmo-ict-faculty.github.io/network-programming/)

Year: 2023/2024

Group: K34212

Author: Kostenko Darina Alekseevna

Lab: Lab3

Date of create: 29.11.2023

Date of finished: ?

## Цель работы: 
с помощью Ansible и Netbox собрать всю возможную информацию об устройствах и сохранить их в отдельном файле.

## Ход работы:

### 1. Поднятие Netbox на VM

На ранее созданной виртуальной машине на платформе Yandex Compute Cloud поднимем Netbox.

Для этого необходимо выполнить следующие шаги:

- Установка и настройка PostgreSQL

Устанавливаем PostgreSQL.

```
sudo apt install -y postgresql
```

Подключаемся к PostgreSQL.

```
sudo -u postgres psql
```

Создаем базу данных и пользователя netbox

```
CREATE DATABASE netbox;
CREATE USER netbox WITH PASSWORD 'J5brHrAXFLQSif0K';
ALTER DATABASE netbox OWNER TO netbox;
```
Проверяем, что мы можем подключиться к базе данных.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab3/lab3-pics/check_psql.png)

- Установка Redis

Устанавливаем Redis.

```
sudo apt install -y redis-server
```

Проверяем, что сервис redis работает.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab3/lab3-pics/check_redis.png)

- Установка и настройка NetBox

Устанавливаем необходимые пакеты.

```
sudo apt install -y python3 python3-pip python3-venv python3-dev build-essential libxml2-dev libxslt1-dev libffi-dev libpq-dev libssl-dev zlib1g-dev
```

Создаем отдельную директорию для NetBox, переходим в нее и клонируем туда репозиторий NetBox.

```
sudo mkdir -p /opt/netbox/
cd /opt/netbox/
sudo git clone -b master --depth 1 https://github.com/netbox-community/netbox.git .
```

Создаем системного пользователя netbox.

```
sudo adduser --system --group netbox
sudo chown --recursive netbox /opt/netbox/netbox/media/
sudo chown --recursive netbox /opt/netbox/netbox/reports/
sudo chown --recursive netbox /opt/netbox/netbox/scripts/
```

Копируем конфигурационный файл

```
cd /opt/netbox/netbox/netbox/
sudo cp configuration_example.py configuration.py
```

Генерируем секретный ключ.

```
python3 ../generate_secret_key.py
```

Настраиваем конфигурацию. В файле configuration.py настраиваем параметры ALLOWED_HOSTS, DATABASE (вводим данные пользователя netbox), SECRET_KEY (вводим сгенерированный секретный ключ). Параметр REDIS остается без изменений.

```
ALLOWED_HOSTS = ['*']
DATABASE = {
    'NAME': 'netbox',               # Database name
    'USER': 'netbox',               # PostgreSQL username
    'PASSWORD': 'пароль', # PostgreSQL password
    'HOST': 'localhost',            # Database server
    'PORT': '',                     # Database port (leave blank for default)
    'CONN_MAX_AGE': 300,            # Max database connection age (seconds)
}

SECRET_KEY = 'сгенерированный_секретный_ключ'
```

Запускаем upgrade-скрипт, который создает виртуальную среду Python, устанавливает необходимые пакеты Python, запускает миграции схемы базы данных.

```
sudo /opt/netbox/upgrade.sh
```

Заходим в созданную виртуальную среду.

```
source /opt/netbox/venv/bin/activate
```

Создаем суперюзера, чтобы была возможность через него заходить в NetBox.

```
cd /opt/netbox/netbox
python3 manage.py createsuperuser
```

- Установка Gunicorn

Копируем конфигурационный файл gunicorn, необходимые файлы netbox и перезагружаем демона.

```
sudo cp /opt/netbox/contrib/gunicorn.py /opt/netbox/gunicorn.py
sudo cp -v /opt/netbox/contrib/*.service /etc/systemd/system/
sudo systemctl daemon-reload
```

Запускаем сервисы netbox и проверяем их статус.

```
sudo systemctl start netbox netbox-rq
sudo systemctl enable netbox netbox-rq
```

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab3/lab3-pics/nb_status.png)


- Установка nginx

Устанавливаем nginx.

```
sudo apt install -y nginx
```

Копируем конфигурационный файл nginx

```
sudo cp /opt/netbox/contrib/nginx.conf /etc/nginx/sites-available/netbox
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/netbox /etc/nginx/sites-enabled/netbox
```

Теперь NetBox доступен по адресу https://белый_ip-адрес_виртуальной_машины/, пользователь admin.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab3/lab3-pics/nb_webpage.png)


### 2. Заполнение информации о CHR в NetBox

Для добавления устройств были добавлены сайт (физическое расположение устройства), роль, производитель и тип устройств.

Далее была заполнена информация о двух CHR, добавлены их интерфейсы и ip-адреса. Для каждого CHR были заранее добавлены по еще одному интерфейсу и ip-адресу для выполнения 5 пункта работы.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab3/lab3-pics/nb_devices.png)

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab3/lab3-pics/nb_interfaces.png)

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab3/lab3-pics/nb_ips.png)

### 3. Сбор данных из NetBox

Используя Ansible и модуль netbox.netbox.nb_inventory, соберем данные об устройствах из Netbox.

Создадим файл netbox_inventory.yml.

```
---
plugin: netbox.netbox.nb_inventory
api_endpoint: https://158.160.48.189
token: токен
validate_certs: False
config_context: False
group_by:
  - device_roles
interfaces: 'True'
```

Токен создается в NetBox.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab3/lab3-pics/nb_token.png)

Всю информацию сохраняем в файл nb_inventory.yml.

```
ansible-inventory -v --list -i netbox_inventory.yml > nb_inventory.yml
```

В файле nb_inventory.yml теперь находится вся полученная из NetBox информация об устройствах. Этот файл будет использоваться в качестве inventory-файла. В этот файл были также перенесены переменные для подключения к роутерам. Полное содержание файла можно просмотреть [здесь](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab3/nb_inventory.yml).

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab3/lab3-pics/nb_inventory_part.png)

### 4. Настройка 2 CHR на основе данных из Netbox

Был написан playbook для изменения имени устройств и добавления IP-адреса на основе inventory-файла nb_inventory.yml.

```
- name: Routers Configuration
  hosts: device_roles_router
  tasks:
    - name: Set Devices Name
      community.routeros.command:
        commands:
          - /system identity set name="{{interfaces[0].device.name}}"
    - name: Set additional IP
      community.routeros.command:
        commands:
        - /interface bridge add name="{{interfaces[1].display}}"
        - /ip address add address="{{interfaces[1].ip_addresses[0].address}}" interface="{{interfaces[1].display}}"
```

Выполняем playbook.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab3/lab3-pics/playbook1.png)

Проверяем изменения на 2 CHR. На рисунке можно увидеть, что изменились имена устройств и добавлен интерфейс с ip-адресом. Наверху также можно увидеть успешные пинги до виртуальной машины.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab3/lab3-pics/CHRs.png)

### 5. Сбор данных с CHR в NetBox

Был написан playbook, позволяющий собрать серийный номер устройства и вносящий серийный номер в Netbox, на основе inventory-файла nb_inventory.yml.

```
- name: Get Serial Numbers To NetBox
  hosts: device_roles_router
  tasks:
    - name: Get Serial Number
      community.routeros.command:
        commands:
          - /system license print
      register: license_print
    - name: Get Name
      community.routeros.command:
        commands:
          - /system identity print
      register: identity_print
    - name: Add Serial Number to Netbox
      netbox_device:
        netbox_url: https://158.160.48.189
        netbox_token: токен
        data:
          name: "{{identity_print.stdout_lines[0][0].split(' ').1}}"
          serial: "{{license_print.stdout_lines[0][0].split(' ').1}}"
        state: present
        validate_certs: False
```

Выполняем playbook.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab3/lab3-pics/playbook2.png)

Проверяем данные о 2 CHR в NetBox - поле Serial Number заполнено.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab3/lab3-pics/nb_CHR1.png) 
![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab3/lab3-pics/nb_CHR2.png)

## Вывод:

при выполнении работы на виртуальной машине был поднят NetBox, в NetBox была внесена информация о 2 CHR. Были написаны playbook-и для настройки 2 CHR на основе данных из Netbox и сбора данных с CHR в NetBox. Были получены навыки по работе с NetBox и Ansible.
