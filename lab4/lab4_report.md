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
Изучить синтаксис языка программирования P4 и выполнить 2 обучающих задания от Open network foundation для ознакомления на практике с P4.

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

Так как по умолчанию все коммутаторы отбрасывают все приходящие к ним пакеты, пинг не успешен ни с одного коммутатора.

Выйдем из среды Mininet (```exit```) и остановим его.

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

Функция MyIngress отвечает за обработку пакетов на этапе входа. Здесь прописываем изменение входящего пакета - изменение порта, адреса источника на свой адрес, установка нового получателя и уменьшение TTL на 1. В конце также проверяем, что заголовок ipv4 валидный.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/t1_MyIngress1.png)

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/t1_MyIngress2.png)

Функция MyDeparser отвечает за порядок вставки полей при сборке заголовков пакета обратно. Здесь используется функция emit, отвечающая за добавление заголовков.

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

В этом упражнении необходимо добавить поддержку базового протокола туннелирования к IP-маршрутизатору, который был реализован в предыдущем упражнении. Базовый коммутатор пересылает пакеты на основе IP-адреса назначения. Задача — определить новый тип заголовка для инкапсуляции IP-пакета и изменить код коммутатора таким образом, чтобы он принимал решение о порте назначения с использованием нового заголовка myTunnel. Новый тип заголовка будет содержать ID протокола, который указывает на тип инкапсулируемого пакета, вместе с ID назначения, который будет использоваться для маршрутизации.

В итоге должна получится следующая сеть:

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/topo.png)

Откроем папку ```p4\tutorials\exercises\basic_tunnel```. Здесь представлен файл basic_tunnel.p4 - стартовый код для этого упражнения, содержащий реализацию базового IP-маршрутизатора (1 упражнение). Полная реализация коммутатора basic_tunnel.p4 должна перенаправлять пакеты на основе содержимого специального заголовка инкапсуляции, а также выполнять обычную IP-пересылку, если заголовок инкапсуляции отсутствует в пакете.

Дополним файл basic_tunnel.p4. В нём необходимо дописать функции MyParser, MyIngress и MyDeparser.

В функции MyParser была добавлена распаковка нового заголовка myTunnel.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/t2_MyParser.png)

В функции MyIngress были добавлены: 
- новый action myTunnel_forward по аналогии с ipv4_forward, но в случае с туннелем нужно указать только порт.
- новая таблица по аналогии с ipv4_lpm, но переадресация туннельная
- проверка валидности заголовка myTunnel: если он валиден, то происходит его обработка, если нет - обрабатывается заголовок IPv4

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/t2_MyIngress1.png)

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/t2_MyIngress2.png)

В функции MyDeparser было прописано добавление заголовка myTunnel.

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/t2_MyDeparser.png)

Полное содержание файла basic_tunnel.p4 можно просмотреть [здесь](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/basic_tunnel.p4).

Скомпилируем полученный файл.
```
make run
```

Откроем два терминала узлов h1 и h2.
```
mininet> xterm h1 h2
```

Каждый хост включает в себя небольшой клиент и сервер обмена сообщениями на основе Python. В xterm узла h2 запустим сервер. 
```
./receive.py
```

Сначала проведем тест без использования туннелирования. В xterm узла h1 отправим сообщение на h2.
```
./send.py 10.0.2.2 "P4 is cool"
```

![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/t2_check1.png)

Пакет был получен на узле h2. В полученном пакете можно увидеть, что он состоит из заголовка Ethernet, IP-заголовка, TCP-заголовка и сообщения. Если изменить IP-адрес назначения при отправке сообщения, то сообщение не будет получено на узле h2.

Теперь проведем тест с использованием туннелирования. В xterm узла h1 отправим сообщение на h2 с указанием ID назначения.
```
./send.py 10.0.2.2 "P4 is cool" --dst_id 2
```
![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/t2_check2.png)

Пакет был получен на узле h2. В полученном пакете можно увидеть, что он состоит из заголовка Ethernet, заголовка myTunnel, IP-заголовка и сообщения. 

Теперь изменим IP-адрес назначения при отправке сообщения с узла h1, но оставим ID назначения = 2 (h2).
```
./send.py 10.0.3.3 "P4 is cool" --dst_id 2
```
![](https://github.com/kostenkoda/2023_2024-network_programming-k34212-kostenko_d_a/blob/main/lab4/lab4-pics/t2_check3.png)

Пакет был получен на узле h2, несмторя на то, что указан IP-адрес узла h3. Это происходит потому, что коммутатор не использует IP-заголовок для маршрутизации, когда в пакете есть заголовок MyTunnel.

После проверки так же выйдем из среды Mininet (```exit```), остановим его и удалим файлы.

```
make stop
make clean
```

## Вывод:

при выполнении лабораторной работы был изучен синтаксис языка программирования P4, были выполнены 2 обучающих задания Implementing Basic Forwarding и Implementing Basic Tunneling для ознакомления на практике с P4.
