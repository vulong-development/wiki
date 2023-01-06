
### Установка с помощью apt-get

```
# wget -O YANDEX-DISK-KEY.GPG http://repo.yandex.ru/yandex-disk/YANDEX-DISK-KEY.GPG

# apt-key add YANDEX-DISK-KEY.GPG

# echo "deb http://repo.yandex.ru/yandex-disk/deb/ stable main" >> /etc/apt/sources.list.d/yandex-disk.list

# apt-get update

# apt-get install yandex-disk
```

### Мастер начальной настройки

Вы можете выполнить начальную настройку клиента с помощью команды

```
yandex-disk setup
```

- Введите название каталога для хранения локальной копии Диска. Если вы оставите название пустым, в домашнем каталоге будет создана папка Yandex.Disk.
- Укажите, использовать ли прокси-сервер (y/n).
- Укажите, запускать ли клиент при старте системы (y/n)

---

После того как мастер завершит работу, в каталоге ~/.config/yandex-disk будет создан файл конфигурации config.cfg.

<sub>Пример файла config.cfg</sub>

```
# Путь к файлу с данными авторизации
auth="/home/user/.config/yandex-disk/passwd"

# Каталог для хранения локальной копии Диска.
dir="/home/user/myDisk"

# Не синхронизировать указанные каталоги.
#exclude-dirs="exclude/dir1,exclude/dir2,path/to/another/exclude/dir"

# Указать прокси-сервер. Примеры:
#proxy=https,127.0.0.1,80
#proxy=https,127.0.0.1,80,login,password
#proxy=https,127.0.0.1,443
#proxy=socks4,my.proxy.local,1080,login,password
#proxy=socks5,my.another.proxy.local,1081
#proxy=auto
#proxy=no
```
