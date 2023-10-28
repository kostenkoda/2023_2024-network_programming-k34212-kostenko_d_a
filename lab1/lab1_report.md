# Отчет по лабораторной работе №1 "Установка CHR и Ansible, настройка VPN"
University: [ITMO University](https://itmo.ru/ru/)

Faculty: [FICT](https://fict.itmo.ru)

Course: [Network Programming](https://itmo-ict-faculty.github.io/network-programming/)

Year: 2023/2024

Group: K34212

Author: Kostenko Darina Alekseevna

Lab: Lab1

Date of create: 23.10.2023

Date of finished: 26.10.2023

**Цель работы:** развертывание виртуальной машины на базе платформы Yandex Compute Cloud с установленной системой контроля конфигураций Ansible и установка CHR в VirtualBox.

**Ход работы:**

1. Схема связи
   
В результате выполнения лабораторной работы будет получена схема связи следующего вида.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab1/lab1-pics/diagram.png)

2. Развертывание виртуальной машины на платформе Yandex Compute Cloud

На платформе Yandex Compute Cloud с использованием гранта была развернута виртуальная машина Ubuntu 22.04.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab1/lab1-pics/vm_ubuntu.png)

На виртуальной машине были установлены python3 и Ansible.

```
sudo apt install python3-pip
sudo pip3 install ansible
```

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab1/lab1-pics/vm_check.png)

3. Установка CHR на VirtualBox

С официального сайта MiktoTik был установлен образ диска CHR, версия 	7.11.2 Stable, и WinBox.
В VirtualBox была добавлена виртуальная машина CHR.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab1/lab1-pics/CHR.png)

Запускаем CHR и фиксируем ip-адрес. 

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab1/lab1-pics/CHR-ip.png)

Теперь к CHR можно подключиться в WinBox.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab1/lab1-pics/winbox.png)


4. Создание VPN-сервера

На виртуальной машине Ubuntu создадим свой WireGuard сервер для организации VPN туннеля между сервером автоматизации, где был установлена система контроля конфигураций Ansible, и локальным CHR.
Для этого устнавливаем WireGuard.
```
sudo apt-get install wireguard
```

Были созданы приватный и публичный ключей сервера.

```
wg genkey | sudo tee /etc/wireguard/wg0-server-private.key | wg pubkey | sudo tee /etc/wireguard/wg0-server-public.key
```

Были созданы приватный и публичный ключей клиента.

```
wg genkey | sudo tee /etc/wireguard/wg0-client-private.key | wg pubkey | sudo tee /etc/wireguard/wg0-client-public.key
```

Был создан конфигурационный файл сервера /etc/wireguard/wg0.conf. Здесь указывается ip-адрес интерфейса Wireguard на сервере (вритуальная машина) и на клиенте (CHR). 

```
[Interface]
Address = 10.2.0.1/24
SaveConfig = true
ListenPort = 51820
PrivateKey = [приватный ключ сервера]

[Peer]
PublicKey = [публичный ключ клиента]
AllowedIPs = 10.2.0.2/32
```

В файле /etc/sysctl.conf разрешаем прохождение трафика между интерфейсами.

```
net.ipv4.ip_forward = 1
```

Добавляем правила маршрутизации для пересылки трафика через WireGuard.

```
sudo iptables -A FORWARD -i wg0 -j ACCEPT
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

Запускаем сервер.

```
sudo systemctl start wg-quick@wg0
```

Проверяем статус сервера.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab1/lab1-pics/server_check.png)

5. Настройка WireGuard клиента на RouterOS

Для настройки WireGuard клиента на CHR был добавлен интерфейс wireguard1, назначен его ip-адрес, настроен пир и добавлено правило в firewall (разрешен трафик WireGuard). 

```
/interface wireguard add listen-port=51820 mtu=1420 name=wireguard1
/interface wireguard peers add allowed-address=10.2.0.1/24  endpoint-address=158.160.44.208
                               endpoint-port=51820 interface=wireguard1 persistent-keepalive=10s
                               public-key="публичный ключ сервера"
/ip address add address=10.2.0.2/24 interface=wireguard1 network=10.2.0.0
/ip firewall filter add action=accept chain=input dst-port=51820 int-interface=wireguard1
                        protocol=udp
```

6. Проверка IP-связности

С CHR пропингуем сервер.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab1/lab1-pics/ping-to-server.png)

С сервера пропингуем CHR.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab1/lab1-pics/ping-to-CHR.png)

Связь есть.


**Вывод:** в ходе выполнения лабораторной работы были получены навыки по поднятию WireGuard сервера и VPN туннеля между сервером и клиентом.
