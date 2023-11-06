# Отчет по лабораторной работе №2 "Развертывание дополнительного CHR, первый сценарий Ansible"
University: [ITMO University](https://itmo.ru/ru/)

Faculty: [FICT](https://fict.itmo.ru)

Course: [Network Programming](https://itmo-ict-faculty.github.io/network-programming/)

Year: 2023/2024

Group: K34212

Author: Kostenko Darina Alekseevna

Lab: Lab2

Date of create: 03.11.2023

Date of finished: ?

**Цель работы:** c помощью Ansible настроить несколько сетевых устройств и собрать информацию о них, правильно собрать файл Inventory.

**Ход работы:**

1. Установка 2-ого CHR

В VirtualBox была добавлена вторая виртуальная машина CHR.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab2/lab2-pics/CHRs.png)

3. Настройка VPN сервера для добавления 2-ого VPN клиента

Для настройки второго VPN клиента на VPN (WireGuard) сервере - виртуальной машине Ubuntu 22.04 на платформе Yandex Compute Cloud - была сгенерирована дополнительная пара ключей.

```
wg genkey | sudo tee /etc/wireguard/wg0-client2-private.key | wg pubkey | sudo tee /etc/wireguard/wg0-client2-public.key
```

Также на сервере был отредактирован конфигурационный файл /etc/wireguard/wg0.conf - был добавлен второй пир.

```
[Interface]
Address = 10.2.0.1/24
ListenPort = 51820
PrivateKey = [приватный ключ сервера]

[Peer]
PublicKey = [публичный ключ 1 клиента]
AllowedIPs = 10.2.0.2/32

[Peer]
PublicKey = [публичный ключ 2 клиента]
AllowedIPs = 10.2.0.3/32
```

3. Настройка 2-ого VPN клиента на 2-ом CHR

Для настройки WireGuard клиента на CHR2 был добавлен интерфейс wireguard1, назначен его ip-адрес, настроен пир и добавлено правило в firewall (разрешен трафик WireGuard).

```
/interface wireguard add listen-port=51820 mtu=1420 name=wireguard1
/interface wireguard peers add allowed-address=10.2.0.1/24  endpoint-address=158.160.53.43
                               endpoint-port=51820 interface=wireguard1 persistent-keepalive=10s
                               public-key="публичный ключ сервера"
/ip address add address=10.2.0.3/24 interface=wireguard1 network=10.2.0.0
/ip firewall filter add action=accept chain=input dst-port=51820 int-interface=wireguard1
                        protocol=udp
```

Настройка аналогична CHR1, отличны только приватный ключ клиента и ip-адрес интерфейса wireguard1.

4. Проверка связности

На WireGuard сервере перезапускаем службу WireGuard.

```
systemctl restart wg-quick@wg0
```

Проверяем доступность локальных CHR с сервера.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab2/lab2-pics/ping-to-CHRs.png)

Связь есть.

5. Настройка 2-х CHR с помощью Ansible

...


**Вывод:**
