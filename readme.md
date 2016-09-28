# usb writer for raspberry pi

позволяет в автоматическом режиме записывать указанный образ на блочное устройство, подключенное к usb-порту (в просторечии — «на флэшку») мини-компьютера *raspberry pi*, с (опциональной) индикацей процесса с помощью подключенных к портам *gpio* светодиодов.

## инструкция по применению

1. скачайте и запишите на sd-карту (размером минимум 8 гигабайт, но лучше — 16) образ, например, [raspbian](https://www.raspberrypi.org/downloads/raspbian/) в версии lite — этого достаточно для работы.
1. установите пакет *at*:

        $ sudo apt-get install --no-install-recommends at
 если интернет на устройстве недоступен, скачайте его по ссылке, которую можно получить, запустив прилагающийся скрипт `get.url`, и установите «вручную» (версия в вашем случае может отличаться в большую сторону):

        $ sudo dpkg -i at_3.1.16-1_armhf.deb
1. создайте каталог `/var/local/usbwriter` и (для удобства) сделайте его владельцем пользователя *pi*:

        $ sudo install -o pi -g pi -d /var/local/usbwriter
1. скопируйте в него все файлы и файл с вашим образом (см. ниже про содержимое файла `rules`)
1. скопируйте файл `99-usbwrite.rules` в каталог `/etc/udev/rules.d/` (после этого надо будет перезапустить либо *udev* (`$ sudo service udev restart`), либо всё устройство.

##файл `rules`

состоит из двух типов записей — про команды для записи и про соответствие индикаторов usb-портам. строки, начинающиеся с символа `#`, можно считать комментариями.

###команды для записи

формат:

    vid:pid ... команда

где `vid` и `pid` — это *vendor id* и *product id* для usb-устройства. пар `vid:pid` может быть несколько (разделённых пробеом) в одной строке — для всех этих устройств будет выполнена одна и та же команда. лидирующие нули как в `vid`, так и в `pid`, должны быть опущены (см. ниже про режим отладки).

пример:

    13fe:4200 sleep 1; cd $(dirname $0) && ./write.usb scratchlive2-ru.hdd $DEVNAME

для устройства с *vendor id=13fe* и *product id=4200* будет выполнена указанная команда, внутри которой есть запуск прилагаемоего файла `write.usb`, который и осуществляет запись указанного файла (`scratchlive2-ru.hdd`) на устройство, имя которого (например, `/dev/sda`) получено от *udev* в переменной `DEVNAME`. в этой строке и надо менять имя файла с образом.

###соответствие индикаторов usb-портам

формат:

    порт сброс установка количество gpio ...

- порт — это идентификатор порта, который получен от *udev* (см. ниже про режим отладки)
- сброс — какое значение писать в gpio-порт для сброса индикатора (обычно — `0`)
- установка — какое значение писать в gpio-порт для установки индикатора (обычно — `1`)
- количество — сколько индикаторов используется для отображения состояния данного usb-порта
- gpio — номера gpio-портов. их число должно соответствовать значению поля `количество`.

пример:

    1-1.2 0 1 3 6 7 13

здесь идентификатор порта — `1-1.2`, для сброса записывать `0`, для установки — `1`, количество индикаторов — `3`, номера gpio-портов этих индикаторов: `6`, `7`, `13`. ([раскладка gpio-портов](http://pinout.xyz/) для raspberry pi).

##режим отладки

если в каталоге `/var/local/usbwriter` создать файл `onlylog`, то вызов программы для записи не будет осуществляться, а все udev-события, генерируемые для блочных устройств, подключаемых (и отключаемых) к usb-портам, будут записываться в лог-файл (по умолчанию — `/var/log/messages` в виде:

> Sep 27 17:40:25 raspberrypi udev.monitor: onlylog: action=add devpath=/devices/platform/soc/3f980000.usb/usb1/1-1/1-1.2/1-1.2:1.0/host12/target12:0:0/12:0:0:0/block/sda vid:pid=13fe:4200 port=1-1.2 devname=/dev/sda

тут перечислены как полученные от *udev* переменные — `action`, `devpath`, `vid`, `pid`, `devname` — так и извлечённый из `devpath` идентификатор порта. если он пустой, то надо вручную подправить регулярное выражение в файле `udev.monitor` (строка, начинающаяся с `port=`).

касательно *vendor id* и *product id* (`vid:pid=`) — именно в таком виде (в примере — `13fe:4200`) эти идентификаторы и должны быть указаны в файле `rules`.

в режиме отладки, для удобства соотнесения индикаторов портам, при каждом полученном событии (`add`, `remove`, `change` и любом неопознанном) зажигаются последовательно (на пол-секунды) все индикаторы, указанные в файле `rules` для данного порта.

##индикация светодиодами (если подключены)

<table>
  <tr>
    <th>состояние</th>
    <th>один индикатор</th>
    <th colspan="2">два индикатора</th>
    <th colspan="3">три индикатора</th>
  </tr>
  <tr>
    <td></td>
    <td></td>
    <td>1</td>
    <td>2</td>
    <td>1</td>
    <td>2</td>
    <td>3</td>
  </tr>
  <tr>
    <td>ничего не вставлено</td>
    <td colspan="6">все погашены</td>
  </tr>
  <tr>
    <td>вставлена неопознанная флэшка</td>
    <td>мигнуть 1 раз, затем 0</td>
    <td>1</td>
    <td>0</td>
    <td>1</td>
    <td>0</td>
    <td>0</td>
  </tr>
  <tr>
    <td>началась запись</td>
    <td>мигнуть 3 раза, затем 0</td>
    <td>0</td>
    <td>1</td>
    <td>0</td>
    <td>1</td>
    <td>0</td>
  </tr>
  <tr>
    <td>запись завершена</td>
    <td>1</td>
    <td>1</td>
    <td>1</td>
    <td>0</td>
    <td>0</td>
    <td>1</td>
  </tr>
  <tr>
    <td>индикация записи</td>
    <td colspan="6">все инвертируются на 0.1с с интервалом 2с</td>
  </tr>
  <tr>
    <td>ошибка записи</td>
    <td colspan="6">все инвертируются на 0.5с с интервалом 0.5с</td>
  </tr>
</table>
