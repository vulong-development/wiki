---
title: Node.js
layout: base-layout
---
# Установка и удаление

#### Для проверки версии node.js и npm введите соответственно
`node -v`

`npm -v`

### Для того чтобы проверить установлена ли node.js введите

`dpkg --get-selections | grep node` 

### Для того чтобы удалить node.js введите

`sudo apt purge nodejs`


## Установка Node.js в Node Version Manager (nvm)

` sudo apt install build-essential checkinstall`

Также нам понадобится libssl:

`sudo apt install libssl-dev`

Скачать и установить менеджер версий NVM можно с помощью следующей команды:

`wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.37.2/install.sh | bash`

После завершения установки вам понадобится перезапустить терминал. Или можно выполнить:

`source /etc/profile`

Затем смотрим список доступных версий Node js:

`nvm ls-remote`

Дальше можно устанавливать Node js в Ubuntu, при установке обязательно указывать версию

`nvm install 14.0`

Список установленных версий вы можете посмотреть выполнив:

`nvm list`

Дальше необходимо указать менеджеру какую версию нужно использовать:

`nvm use 14.0`

Чтобы удалить эту версию node js, ее нужно деактивировать:

`nvm deactivate 14.0`

Затем можно удалить:

`nvm uninstall 14.0`

## Установка Node.js из репозиториев Ubuntu
Для обновления репозитория введите команду:

`curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -`

### Установка node.js

`sudo apt install nodejs`

### Установка npm

`sudo apt-get install nodejs`

## BIN

`wget https://nodejs.org/dist/v13.14.0/node-v13.14.0-linux-x64.tar.xz`

`tar xJf node-v13.14.0-linux-x64.tar.xz --strip 1`

`rm node-v13.14.0-linux-x64.tar.xz`