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

Во-первых, создаем inventory - файл /etc/ansible/hosts, который содержит информацию о хостах CHR1 и CHR2, которые необходимо настроить с помощью Ansible, и общие переменные.

```
[CHRs]
CHR1 ansible_host=10.2.0.2 router_id=1.1.1.1
CHR2 ansible_host=10.2.0.3 router_id=2.2.2.2

[CHRs:vars]
ansible_user=admin
ansible_password=admin
ansible_connection=network_cli
ansible_network_os=routeros
```

Проверим inventory.

```
ansible-inventory --list -y
```

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab2/lab2-pics/hosts.png)

Во-вторых, настраиваем ssh-соединения с Ubuntu на CHR1 и CHR2 для работы Ansible. Для этого создаем пару ключей и копируем публичный ключ на CHR1 и CHR2.
```
ssh-keygen
ssh admin@10.2.0.2 "/file print file=mykey; file set mykey contents=\"`cat ~/.ssh/id_rsa.pub`\";/user ssh-keys import public-key-file=mykey.txt;/ip ssh set always-allow-password-login=yes"
ssh admin@10.2.0.3 "/file print file=mykey; file set mykey contents=\"`cat ~/.ssh/id_rsa.pub`\";/user ssh-keys import public-key-file=mykey.txt;/ip ssh set always-allow-password-login=yes"

```

Проверим доступность хостов CHR1 и CHR2.

```
ansible -m ping all
```

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab2/lab2-pics/check_hosts.png)

В-третьих, создаем playbook - yaml файл, который будет содержать инструкции для выполнения задач на CHR1 и CHR2.

На 2-х CHR необходимо настроить логин/пароль, NTP Client, OSPF с указанием Router ID и собрать данные по OSPF топологии и полный конфиг устройств.

Полученный playbook:

```
- name: "playbook for lab2"
  hosts: CHRs

  tasks:
    - name: Set User&Password
      community.routeros.command:
        commands: "user add name=user1 password=user1 group=full"

  tasks:
    - name: Set NTP
      community.routeros.command:
        commands: "system ntp client set enabled=yes servers=8.8.8.8"

    - name: Set OSPF
      community.routeros.command:
        commands:
          - /interface bridge add name=Lo
          - /ip address add address="{{ router_id }}"/32 interface=Lo
          - /routing ospf instance add name=v2inst version=2 router-id="{{ router_id }}"
          - /routing ospf area add name=backbone_v2 area-id=0.0.0.0 instance=v2inst
          - /routing ospf interface-template add network=0.0.0.0/0 area=backbone_v2

    - name: Collect OSPF information
      community.routeros.command:
        commands: "/routing ospf neighbor print"
      register: ospf_info

    - name: Collect config
      community.routeros.facts:
        gather_subset:
          - config
      register: config_info

    - name: Get OSPF information
      debug:
        msg: "{{ ospf_info }}"

    - name: Get config
      debug:
        msg: "{{ config_info }}"
```

Запускаем playbook.

```
ansible-playbook -i /etc/ansible/hosts lab2_playbook.yml
```

После запускаться последовательно выводится информация о выполнении каждого задания.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab2/lab2-pics/playbook_playing.png)

Далее выводится информация по OSPF топологии и полный конфиг устройств, в конце - общая сводка.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab2/lab2-pics/playing_res.png)

6. Проверка настроек на CHR и их связности

После запуска и работы playbook на 2-х CHR добавились настройки.

CHR1:

```
/interface bridge add name=Lo
/routing ospf instance add disabled=no name=v2inst router-id=1.1.1.1
/routing ospf area add disabled=no instance=v2inst name=backbone_v2
/routing ospf interface-template add network=0.0.0.0/0 area=backbone_v2
/ip address add address=1.1.1.1 interface=Lo network=1.1.1.1
/routing ospf interface-template add area=backbone_v2 disabled=no network=0.0.0.0/0
```

CHR2:

```
/interface bridge add name=Lo
/routing ospf instance add disabled=no name=v2inst router-id=2.2.2.2
/routing ospf area add disabled=no instance=v2inst name=backbone_v2
/routing ospf interface-template add network=0.0.0.0/0 area=backbone_v2
/ip address add address=2.2.2.2 interface=Lo network=2.2.2.2
/routing ospf interface-template add area=backbone_v2 disabled=no network=0.0.0.0/0
```

Проверим работу OSPF на обоих CHR.

CHR1:

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab2/lab2-pics/ospf_neighbor_on_CHR1.png)

CHR2:

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab2/lab2-pics/ospf_neighbor_on_CHR2.png)

Проверим связность между CHR.

Пинг CHR2 с CHR1:

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab2/lab2-pics/ping_to_CHR2.png)

Пинг CHR1 с CHR2:

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab2/lab2-pics/ping_to_CHR1.png)

CHR так же пингуются через Lo.

**Вывод:** в ходе выполнения лабораторной работы были получены навыки по сборке файла Inventory, настройке нескольких сетевых устройств и сбора информацию о них с помощью Ansible.
