---
title: Удаление мусора
layout: base-layout
time: 14 июля 2024
---

# Удаление мусора:
---

<time>{{time}}</time>

Если вы никогда не задумывались о том, сколько же места реально занято на вашей машине Docker’ом, то можете быть неприятно удивлены выводом этой команды:

```
$ docker system df
```

Здесь отображено использование диска Docker’ом в различных разрезах:


- образы (images) – общий размер образов, которые были скачаны из хранилищ образов и построены в вашей системе;
- контейнеры (containers) – общий объем дискового пространства, используемый запущенными контейнерами (имеется ввиду общий объем слоев чтения-записи всех контейнеров);
- локальные тома (local volumes) – объем локальных хранилищ, примонтированных к контейнерам;
- кэш сборки (build cache) – временные файлы, сгенерированные процессом построения образов (при использовании инструмента BuildKit, доступного начиная с Docker версии 18.09).

Готов поспорить, что уже после этого простого перечисления вы горите желанием почистить диск от мусора и вернуть к жизни драгоценные гигабайты (прим. перев.: особенно, если за эти гигабайты вы ежемесячно перечисляете арендную плату).

### Использование диска контейнерами

Каждый раз при создании контейнера на хостовой машине в каталоге /var/lib/docker создается несколько файлов и каталогов, среди которых стоит отметить следующие:

- Каталог /var/lib/docker/containers/ID_контейнера – при использовании стандартного драйвера логгирования именно сюда сохраняются журналы событий в JSON-формате. Слишком подробные логи, а также логи, которые никто не читает и не обрабатывает иными способами, часто становятся причиной переполнения дисков.

- Каталог /var/lib/docker/overlay2 – содержит слои чтения-записи контейнеров (overlay2 – предпочитаемые драйвер в большинстве дистрибутивов Linux). Если контейнер сохраняет данные в своей файловой системе, то именно в этом каталоге они и будут размещены.

Давайте представим себе систему, на которой установлен девственно чистый Docker, ни разу не участвовавший в запуске контейнеров и сборке образов. Его отчет об использовании дискового пространства будет выглядеть так:

```
$ docker system df
TYPE           TOTAL      ACTIVE     SIZE       RECLAIMABLE
Images         0          0          0B         0B
Containers     0          0          0B         0B
Local Volumes  0          0          0B         0B
Build Cache    0          0          0B         0B
```

Запустим какой-нибудь контейнер, например, NGINX:

```
$ docker container run --name www -d -p 8000:80 nginx:1.16
```

Что происходит с диском:

- образы (images) занимают 126 Мб, это тот самый NGINX, который мы запустили в контейнере;
- контейнеры (containers) занимают смешные 2 байта.

```
$ docker system df
TYPE           TOTAL      ACTIVE     SIZE       RECLAIMABLE
Images         1          1          126M       0B (0%)
Containers     1          1          2B         0B (0%)
Local Volumes  0          0          0B         0B
Build Cache    0          0          0B         0B
```

Судя по выводу, у нас еще нет пространства, которое мы могли бы высвободить. Так как 2 байта это совершенно несерьезно, давайте представим, что наш NGINX неожиданно для всех написал куда-то 100 Мегабайт данных и создал внутри себя файл test.img именно такого размера.

```
$ docker exec -ti www \
  dd if=/dev/zero of=test.img bs=1024 count=0 seek=$[1024*100]
```

Снова исследуем использование дискового пространства на хосте. Мы увидим, что контейнер (containers) занимает там 100 Мегабайт.

```
$ docker system df
TYPE           TOTAL      ACTIVE     SIZE       RECLAIMABLE
Images         1          1          126M       0B (0%)
Containers     1          1          104.9MB    0B (0%)
Local Volumes  0          0          0B         0B
Build Cache    0          0          0B         0B
```

Думаю, ваш пытливый мозг уже задается вопросом, где же находится наш файл test.img. Давайте его поищем:

```
$ find /var/lib/docker -type f -name test.img
/var/lib/docker/overlay2/83f177...630078/merged/test.img
/var/lib/docker/overlay2/83f177...630078/diff/test.img
```

Не вдаваясь в подробности можно отметить, что файл test.img удобно расположился на уровне чтения-записи, управляемом драйвером overlay2. Если же мы остановим наш контейнер, то хост подскажет нам, что это место, в принципе, можно высвободить:

```
# Stopping the www container
$ docker stop www

# Visualizing the impact on the disk usage
$ docker system df
TYPE           TOTAL      ACTIVE     SIZE       RECLAIMABLE
Images         1          1          126M       0B (0%)
Containers     1          0          104.9MB    104.9MB (100%)
Local Volumes  0          0          0B         0B
Build Cache    0          0          0B         0B
```

Как мы можем это сделать? Удалением контейнера, которое повлечет за собой очистку соответствующего пространства на уровне чтения-записи.


С помощью следующей команды вы можете удалить все установленные контейнеры одним махом и очистить ваш диск от всех созданных ими на уровне чтения-записи файлов:

```
$ docker container prune
WARNING! This will remove all stopped containers.
Are you sure you want to continue? [y/N] y
Deleted Containers:
5e7f8e5097ace9ef5518ebf0c6fc2062ff024efb495f11ccc89df21ec9b4dcc2

Total reclaimed space: 104.9MB
```

Итак, мы высвободили 104,9 Мегабайта удалением контейнера. Но так как мы уже не используем скачанный ранее образ, то он тоже становится кандидатом на удаление и высвобождение наших ресурсов:

```
$ docker system df
TYPE           TOTAL      ACTIVE     SIZE       RECLAIMABLE
Images         1          0          126M       126M (100%)
Containers     0          0          0B         0B
Local Volumes  0          0          0B         0B
Build Cache    0          0          0B         0B
```

> Внимание: до тех пор, пока образ используется хотя бы одним контейнером, вы не сможете использовать этот трюк.

Субкоманда prune, которую мы использовали выше, дает эффект только на остановленных контейнерах. Если мы хотим удалить не только остановленные, но и запущенные контейнеры, следует использовать одну из этих команд:

```
# Historical command
$ docker rm -f $(docker ps –aq)

# More recent command
$ docker container rm -f $(docker container ls -aq)
```

> Заметки на полях: если при запуске контейнера использовать параметр --rm, то при его остановке будут высвобождено все дисковое пространство, которое он занимал.

### Использование диска образами

Несколько лет назад размер образа в несколько сотен мегабайт был совершенно нормальным: образ Ubuntu весил 600 Мегабайт, а образ Microsoft .Net – несколько Гигабайт. В те лохматые времена скачивание одного только образа могло нанести большой урон вашему свободному месту на диске, даже если вы расшаривали уровни между образами. Сегодня – хвала великим – образы весят намного меньше, но даже в этом случае можно быстро забить имеющиеся ресурсы, если не принимать некоторых мер предосторожности.


Есть несколько типов образов, которые напрямую не видны конечному пользователю

- intermediate образы, на основе которых собраны другие образы в – они не могут быть удалены, если вы используете контейнеры на базе этих самых «других» образов;

- dangling образы – это такие intermediate образы, на которые не ссылается ни один из запущенных контейнеров – они могут быть удалены.

- С помощью следующей команды вы можете проверить наличие в вашей системе dangling образов

```
$ docker image ls -f dangling=true
REPOSITORY  TAG      IMAGE ID         CREATED             SIZE
none      none   21e658fe5351     12 minutes ago      71.3MB
```

Удалить их можно следующим способом:

```
$ docker image rm $(docker image ls -f dangling=true -q)
```

Мы можем использовать также субкоманду prune:

```
$ docker image prune
WARNING! This will remove all dangling images.
Are you sure you want to continue? [y/N] y
Deleted Images:
deleted: sha256:143407a3cb7efa6e95761b8cd6cea25e3f41455be6d5e7cda
deleted: sha256:738010bda9dd34896bac9bbc77b2d60addd7738ad1a95e5cc
deleted: sha256:fa4f0194a1eb829523ecf3bad04b4a7bdce089c8361e2c347
deleted: sha256:c5041938bcb46f78bf2f2a7f0a0df0eea74c4555097cc9197
deleted: sha256:5945bb6e12888cf320828e0fd00728947104da82e3eb4452f

Total reclaimed space: 12.9kB
```

Если мы вдруг захотим удалить вообще все образы (а не только dangling) одной командой, то можно сделать так:

```
$ docker image rm $(docker image ls -q)
```

### Использование диска томами

Тома (volumes) применяются для хранения данных за пределами файловой системы контейнера. Например, если мы хотим сохранить результаты работы какого-либо приложения, чтобы использовать их как-то еще. Частым примером являются базы данных.


Давайте запустим контейнер MongoDB, примонтируем к нему внешний по отношению к контейнеру том, и восстановим из него бэкап базы данных (у нас он доступен в файле bck.json):

```
# Running a mongo container
$ docker run --name db -v $PWD:/tmp -p 27017:27017 -d mongo:4.0

# Importing an existing backup (from a huge bck.json file)
$ docker exec -ti db mongoimport \
  --db 'test' \
  --collection 'demo' \
  --file /tmp/bck.json \
  --jsonArray
```

Данные будут находиться на хостовой машине в каталоге /var/lib/docker/volumes. Но почему не на уровне чтения-записи контейнера? Потому что в Dockerfile образа MongoDB каталог /data/db (в котором MongoDB по умолчанию хранит свои данные) определен как том (volume).

> Заметки на полях: многие образы, в результате работы которых должны создаваться данные, используют тома (volumes) для сохранения этих самых данных.

Когда мы наиграемся с MongoDB и остановим (а может даже и удалим) контейнер, том не будет удален. Он продолжит занимать наше драгоценное дисковое пространство до тех пор, пока мы явно не удалим его такой командой:

```
$ docker volume rm $(docker volume ls -q)
```

Ну или мы можем использовать уже знакомую нам субкоманду prune

```
$ docker volume prune
WARNING! This will remove all local volumes not used by at least one container.
Are you sure you want to continue? [y/N] y
Deleted Volumes:
d50b6402eb75d09ec17a5f57df4ed7b520c448429f70725fc5707334e5ded4d5
8f7a16e1cf117cdfddb6a38d1f4f02b18d21a485b49037e2670753fa34d115fc
599c3dd48d529b2e105eec38537cd16dac1ae6f899a123e2a62ffac6168b2f5f
...
732e610e435c24f6acae827cd340a60ce4132387cfc512452994bc0728dd66df
9a3f39cc8bd0f9ce54dea3421193f752bda4b8846841b6d36f8ee24358a85bae
045a9b534259ec6c0318cb162b7b4fca75b553d4e86fc93faafd0e7c77c79799
c6283fe9f8d2ca105d30ecaad31868410e809aba0909b3e60d68a26e92a094da

Total reclaimed space: 25.82GB
luc@saturn:~$
```

### Использование диска для кэша сборки образов

Давайте соберем образ обычным способом, без использования BuildKit:

```
$ docker build -t app:1.0 .
```

Если мы проверим использование дискового пространства, то увидим, что место занимают только базовый образ (node:13-alpine) и конечный образ (app:1.0):

```
TYPE           TOTAL      ACTIVE     SIZE       RECLAIMABLE
Images         2          0          109.3MB    109.3MB (100%)
Containers     0          0          0B         0B
Local Volumes  0          0          0B         0B
Build Cache    0          0          0B         0B
```

Давайте соберем вторую версию нашего приложения, уже с использованием BuildKit. Для этого нам лишь необходимо установить переменную DOCKER_BUILDKIT в значение 1:

```
$ DOCKER_BUILDKIT=1 docker build -t app:2.0 
```

Если мы сейчас проверим использование диска, то увидим, что теперь там участвует кэш сборки (buid-cache):

```
$ docker system df
TYPE           TOTAL      ACTIVE     SIZE       RECLAIMABLE
Images         2          0          109.3MB    109.3MB (100%)
Containers     0          0          0B         0B
Local Volumes  0          0          0B         0B
Build Cache    11         0          8.949kB    8.949kB
```

Для его очистки воспользуемся следующей командой:

```
$ docker builder prune
WARNING! This will remove all dangling build cache.
Are you sure you want to continue? [y/N] y
Deleted build cache objects:
rffq7b06h9t09xe584rn4f91e
ztexgsz949ci8mx8p5tzgdzhe
3z9jeoqbbmj3eftltawvkiayi

Total reclaimed space: 8.949kB
```

### Очистить все!

Итак, мы рассмотрели очистку дискового пространства, занятого контейнерами, образами и томами. В этом нам помогает субкоманда prune. Но ее можно использовать и на системном уровне docker, и она очистит все, что только сможет:

```
$ docker system prune
WARNING! This will remove:
  - all stopped containers
  - all networks not used by at least one container
  - all dangling images
  - all dangling build cache

Are you sure you want to continue? [y/N]
```