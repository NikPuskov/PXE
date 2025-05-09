# PXE

Описание домашнего задания

1. Настроить загрузку по сети дистрибутива Ubuntu 24

2. Установка должна проходить из HTTP-репозитория.

3. Настроить автоматическую установку c помощью файла user-data

*4. Настроить автоматическую загрузку по сети дистрибутива Ubuntu 24 c использованием UEFI

Задания со звёздочкой выполняются по желанию

Формат сдачи ДЗ: vagrant + ansible

------------------------------------------------------------------------------------------

1. Настройка DHCP и TFTP-сервера

Подготовим Vagrantfile в котором будут описаны 2 виртуальные машины:

• pxeserver (хост к которому будут обращаться клиенты для установки ОС)

• pxeclient (хост, на котором будет проводиться установка)

Создаем виртуальные машины: `vagrant up`

Для того, чтобы клиент мог получить ip-адрес нам требуется DHCP-сервер. Для того, чтобы клиент мог получить файл pxelinux.0 нам потребуется TFTP-сервер. Утилита dnsmasq совмещает в себе сразу и DHCP и TFTP-сервер. Настраиваем сервер. Отключаем firewall:

`systemctl stop ufw`

`systemctl disable ufw`

Обновляем apt кэш и устанавливаем dnsmasq:

`apt update`

`apt install dnsmasq`

Создаём файл /etc/dnsmasq.d/pxe.conf и добавляем в него следующее содержимое:

![Image alt](https://github.com/NikPuskov/PXE/blob/main/pxe1.jpg)

Создаём каталоги для файлов TFTP-сервера: `mkdir -p /srv/tftp`

Cкачиваем файлы для сетевой установки Ubuntu 24.04.2 и распаковываем их в каталог /srv/tftp:

`wget https://releases.ubuntu.com/noble/ubuntu-24.04.2-netboot-amd64.tar.gz`

`tar -xzvf ubuntu-24.04-netboot-amd64.tar.gz  -C /srv/tftp`

В каталоге видим следующие файлы:

![Image alt](https://github.com/NikPuskov/PXE/blob/main/pxe2.jpg)

Перезапускаем службу dnsmasq: `systemctl restart dnsmasq`

-------------------------------------------------------------------------------------------

2. Настройка Web-сервера

Для того, чтобы отдавать файлы по HTTP нам потребуется настроенный веб-сервер.

Устанавливаем Web-сервер apache2: `apt install apache2`

Cоздаём каталог /srv/images в котором будут храниться iso-образы для установки по сети: `mkdir /srv/images`

Переходим в каталог /srv/images и скачиваем iso-образ ubuntu 24.04.2:

`wget https://releases.ubuntu.com/noble/ubuntu-24.04.2-live-server-amd64.iso`

Cоздаём файл /etc/apache2/sites-available/ks-server.conf и добавлем в него следующее содержимое:

![Image alt](https://github.com/NikPuskov/PXE/blob/main/pxe3.jpg)

Активируем конфигурацию ks-server в apache: `a2ensite ks-server.conf`

Вносим изменения в файл /srv/tftp/amd64/pxelinux.cfg/default:

![Image alt](https://github.com/NikPuskov/PXE/blob/main/pxe4.jpg)

В данном файле мы указываем что файлы linux и initrd будут забираться по tftp, а сам iso-образ ubuntu 24.04.2 будет скачиваться из нашего веб-сервера.

Из-за того, что образ достаточно большой (3.0G) и он сначала загружается в ОЗУ, необходимо указать размер ОЗУ до 4 гигабайт (root=/dev/ram0 ramdisk_size=4000000).

Перезагружаем web-сервер apache: ` systemctl restart apache2`

На данный момент, если мы запустим ВМ pxeclient, то увидим загрузку по PXE, загрузку iso-образа и откроется мастер установки ubuntu. Но так как на клиенте pxeclient используется 2 сетевых карты, загрузка может начаться не с intnet-интерфейса, а с nat-интерфейса. При этом мы увидим сообщение: "No route to host ...". Решение: временно отключить nat-интерфейс, либо перезагружать виртуальную машину до тех пор пока не начнётся загрузка с intnet-интерфейса. Есть еще параметр виртуальной машины VBox --nicbootprio, в котором можно указать приоритет для сетевой карты при загрузке с PXE. Но мне не удалось повлиять этим параметром на порядок выбора интерфейса. В конфигурации клиента указывал:

![Image alt](https://github.com/NikPuskov/PXE/blob/main/pxe5.jpg)

-----------------------------------------------------------------------------------------------

3. Настройка автоматической установки Ubuntu 24.04

Создаём каталог для файлов с автоматической установкой: `mkdir /srv/ks`

Cоздаём файл /srv/ks/user-data и добавляем в него следующее содержимое:

![Image alt](https://github.com/NikPuskov/PXE/blob/main/pxe6.jpg)

`Файл /srv/ks/user-data имеет синтаксис yaml, соответственно нужно соблюдать отступы`

Cоздаём файл с метаданными /srv/ks/meta-data: `touch /srv/ks/meta-data`

Файл с метаданными хранит дополнительную информацию о хосте, мы сейчас не будем добавлять дополнительную информацию.

В конфигурации веб-сервера добавим каталог /srv/ks идентично каталогу /srv/images: `nano /etc/apache2/sites-available/ks-server.conf`

![Image alt](https://github.com/NikPuskov/PXE/blob/main/pxe7.jpg)

В файле /srv/tftp/amd64/pxelinux.cfg/default добавляем параметры автоматической установки:

![Image alt](https://github.com/NikPuskov/PXE/blob/main/pxe8.jpg)

Перезапускаем службы dnsmasq и apache2:

`systemctl restart dnsmasq`

`systemctl restart apache2`

На этом настройка автоматической установки завершена. Теперь можно перезапустить ВМ pxeclient и мы должны увидеть автоматическую установку. После успешной установки выключаем ВМ и в её настройках ставим запуск ВМ из диска. После запуска нашей ВМ мы сможем залогиниться под пользователем otus.
