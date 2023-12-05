# Отчет по лабораторной работе №4 "Базовая 'коммутация' и туннелирование используя язык программирования P4"
University: [ITMO University](https://itmo.ru/ru/)

Faculty: [FICT](https://fict.itmo.ru)

Course: [Network Programming](https://itmo-ict-faculty.github.io/network-programming/)

Year: 2023/2024

Group: K34212

Author: Kostenko Darina Alekseevna

Lab: Lab4

Date of create: 04.12.2023

Date of finished: ?

## Цель работы: 
Изучить синтаксис языка программирования P4 и выполнить 2 задания обучающих задания от Open network foundation для ознакомления на практике с P4.

## Ход работы:

### 0. Подготовка

Перед выполнением работы был склонирован репозиторий [p4lang/tutorials](https://github.com/p4lang/tutorials). Vagrant и VirtualBox были установлены на компьютер ранее.

Был выполнен переход в папку vm-ubuntu-20.04 и развернута тестовая среда с помощью Vagrant.

```
cd vm-ubuntu-20.04
vagrant up
```

В результате установки была создана виртуальная машина с аккаунтами vagrant/vagrant и p4/p4.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/vm.png)



### 1. Implementing Basic Forwarding

На виртуальной машине был выполнен вход под аккаунтом p4. 

Для этого упражнения будет использоваться следующая топология сети:

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/pod-topo.png)

Откроем папку ```p4\tutorials\exercises\basic```. Здесь представлен файл basic.p4 - каркас программы на P4, который изначально отбрасывает все пакеты. Задача — расширить программу для корректного перенаправления пакетов IPv4.

Сначала скомпилируем неполный файл basic.p4 и создадим коммутатор в Mininet, чтобы протестировать его поведение. Для этого в терминале выполним:

```
make run
```

После перехода в среду Mininet проверим связность между хостами.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/t1_precheck.png)

Так как умолчанию все коммутаторы сбрасывают все приходящие к ним пакеты, пинг не успешен ни с одного коммутатора.

Выйдем из среды Mininer (```exit```) и остановим его

```
make stop
```

Также удалим все файлы pcaps, файлы сборки и логи.

```
make clean
```

Теперь дополним файл basic.p4. В нём необходимо дописать функции MyParser, MyIngress и MyDeparser.

Функция MyParser распаковывает Ethernet-заголовок и в случае, если тип пакета соответствует IPv4, происходит распаковка IPv4-заголовка. Здесь используется уже готовая функция extract, распаковывающая пакет.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/t1_MyParser.png)

Функция MyIngress является контроллером, отвечающим за обработку пакетов на этапе входа. Здесь прописываем изменение входящего пакета - изменение порта, адреса источника на свой адрес, установка нового получателя и уменьшение TTL на 1. В конце также проверяем, что заголовок ipv4 валидный.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/t1_MyIngress1.png)

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/t1_MyIngress2.png)

Функция MyDeparser отвечает за порядок вставки полей при сборке заголовков пакета обратно. Здесь используется функция emit, отвечающая за добавление заголовков

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/t1_MyDeparser.png)

Полное содержание файла basic.p4 можно просмотреть [здесь](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/basic.p4).

Скомпилируем полученный файл и проверим связь в сети.

```
make run
```

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/t1_check.png)

После проверки так же выйдем из среды Mininet (exit), остановим его и удалим файлы.

```
make stop
make clean
```

### 2. Implementing Basic Tunneling



## Вывод:

