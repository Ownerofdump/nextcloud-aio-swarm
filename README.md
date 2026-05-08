# Nextcloud AIO Swarm

Данный репозиторий позволит развернуть вам Nextcloud AIO в Docker Swarm для повышения отказоустойчивости за счет выноса stateless-сервисов в горизонтальное масштабирование, а также для контроля обновлений в production-средах.

---

Включает в себя:
- Nextcloud
- Apache
- Imaginary
- Notify-Push
- Redis
- Postgres
- OnlyOffice Unlimited
- Talk

# Теория
В Nextcloud используется два типа сервисов - stateless и stateful. 
- Stateless (Apache, Nextcloud, Imaginary, Notify-Push) можно масштабировать при условии подключения их к единой точке обмена данных - NFS, БД и Redis.
- Stateful - сервисы которые не поддерживают простого масштабирования и требуют репликацию(Redis, Postgres)/другое решение(OnlyOffice Docs for Buisness), либо вовсе не способны к масштабированию из-за определенных проблем (Talk)

---

Тестовый стенд, для которого была написана данная инструкция (учтите, все сервера должны иметь доступ друг к другу для создания Docker-Overlay сети):

- CS - сервер на котором будут находится reverse-proxy, NFS и stateful-сервисы
- WS - сервера, на которых будут развернуты stateless-сервисы

| Имя сервера | SSD | HDD |
| ------ | ------ | ------ |
| CS | 60GB + 512GB | 8TB |
| WS-1 | 60GB | - |
| WS-2 | 60GB | - |
| WS-3 | 60GB | - |

---

Yaml файл запускает по одной реплике каждого сервиса. Это необходимо для предотвращения ситуации, где несколько запущенных Nextcloud-контейнеров устраивают гонку за файлы.

Логика использования:
1. Запустить в единичной реплике с закреплением последних версий образов для получения самого актуального Nextcloud
2. Проверить работоспособность
3. Маштабировать через docker scale

Теперь про обновления и установку. Цикл внутри образа работает по такому порядку:
1. Проверяет текущую версию, если её нет - начинает копирование самого инстанса Nextcloud из образа в volume
2. Если в блоке nextcloud-aio-nextcloud установлен флаг `INSTALL_LATEST_MAJOR=yes`, то он обновит версию выше, чем есть в образе. К примеру - в образе 22.0.8, сначала он установит её, а после - обновит до следующего мажорного релиза
3. При повторном запуске он проверяет лишь текущую версию с версией в образе, если версия в образе новее - обновляет

Исходя из этого, есть два варианта развертывания:
1. Использовать только версии в самих образах и обновлять их сменой версии образа
2. Использовать флаг `INSTALL_LATEST_MAJOR=yes` в блоке nextcloud-aio-nextcloud при установке и дополнительно выполнять ручное обновление до последней версии после смены версии образа с последующим отключением авто-обновлений

# Подготовка серверов
Перед началом развертывания, необходимо привести сервера к данному состоянию:

### CS
- Установить Docker, NFS-Server, Reverse-Proxy
- Создать на сервере Docker-Swarm кластер (`docker swarm init --advertise-addr 192.168.0.1`)
- Установить label `storage` в Docker Swarm для данного сервера
- Создать папки для Nextcloud на сервере CS:
```bash
mkdir -p /mnt/nextcloud_aio_database
mkdir -p /mnt/nextcloud_aio_database_dump
mkdir -p /mnt/nextcloud_aio_redis
mkdir -p /mnt/nextcloud_aio_onlyoffice
mkdir -p /mnt/nextcloud_aio_nextcloud
mkdir -p /mnt/nextcloud_aio_nextcloud_data
mkdir -p /mnt/nextcloud_aio_nextcloud_tmp
chown -R 999:999 /mnt/nextcloud_aio_database
chown -R 999:999 /mnt/nextcloud_aio_database_dump
chown -R 999:999 /mnt/nextcloud_aio_redis
chown -R 1000:1000 /mnt/nextcloud_aio_onlyoffice
chown -R 33:33 /mnt/nextcloud_aio_nextcloud
chown -R 33:33 /mnt/nextcloud_aio_nextcloud_data
chown -R 33:33 /mnt/nextcloud_aio_nextcloud_tmp
```
- Монтировать HDD в `/mnt/nextcloud_aio_nextcloud_data`
- Монтировать SSD в `/mnt/nextcloud_aio_database`, `/mnt/nextcloud_aio_database_dump`, `/mnt/nextcloud_aio_redis`, `/mnt/nextcloud_aio_onlyoffice`, `/mnt/nextcloud_aio_nextcloud`, `/mnt/nextcloud_aio_nextcloud_tmp`
- Настроить папку `/mnt` как NFS, с параметрами `rw,async,no_subtree_check,no_root_squash,crossmnt`

### WS
- Установить Docker и NFS-Client
- Подключить сервера к Docker Swarm кластеру (`docker swarm join --token <TOKEN> 192.168.0.1:2377`)
- Установить label `worker` в Docker Swarm для данных серверов (**выполняется с CS**: `docker node update --label-add type=worker WS-1`, `docker node update --label-add type=worker WS-2`, `docker node update --label-add type=worker WS-3` и так далее..)

---

Для удобного доступа во время развертывания и после завершения настройки вешего инстанса Nextcloud вам необходимо настроить reverse-proxy с балансировкой. Для настройки мы рекомендуем ознакомиться с данными материалами:
- [Конфигурации Nextcloud AIO](https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md)
- [Конфигурации OnlyOffice](https://github.com/ONLYOFFICE/document-server-proxy/)

Официальные конфигурации reverse-proxy не предусматривают балансировки. Для его включения необходимо редактировать конфиги, для Nextcloud:
1. Добавляем в начало конфигурации секцию `upstream`:
```
upstream nextcloud_swarm {
    ip_hash;
    keepalive 32;
    server 192.168.0.2:335 max_fails=3 fail_timeout=30s;
    server 192.168.0.3:335 max_fails=3 fail_timeout=30s;
    server 192.168.0.4:335 max_fails=3 fail_timeout=30s;
}
```

2. Заменяем `proxy_pass` в секции `location /`, учтите, что имя сервиса должно совпадать с секцией `upstream`:
```
location / {
        proxy_pass http://nextcloud_swarm;
}
```

Также, для OnlyOffice (**обратите внимания, без** `ip_hash;`):
1. Добавляем в начало конфигурации секцию `upstream`:
```
upstream onlyoffice {
    keepalive 32;
    server 192.168.0.2:335 max_fails=3 fail_timeout=30s;
    server 192.168.0.3:335 max_fails=3 fail_timeout=30s;
    server 192.168.0.4:335 max_fails=3 fail_timeout=30s;
}
```

2. Заменяем `proxy_pass` в секции `location /`, учтите, что имя сервиса должно совпадать с секцией `upstream`:
```
location / {
        proxy_pass http://onlyoffice;
}
```

# Запуск Nextcloud AIO

Сохраните файл nextcloud.yaml, обязательно измените в нем все значения для минимального запуска:
- [последние версии образов AIO](https://github.com/orgs/nextcloud-releases/packages) - `IMAGE VERSION`
- домены Nextcloud и OnlyOffice - `NEXTCLOUD DOMAIN`, `ONLYOFFICE DOMAIN`
- логин и пароль администратора - `ADMIN USERNAME`, `ADMIN PASSWORD`
- необходимые секреты - `ONLYOFFICE SECRET`, `POSTGRES SECRET`, `REDIS SECRET`, `TURN SECRET`, `SIGNALING SECRET`, `IMAGINARY SECRET`, `INTERNAL TALK SECRET`
- IP и пути к NFS - `PATH TO NFS`, `NFS IP`

Для генерации секретов используйте данные команды:
```bash
echo "ONLYOFFICE SECRET: $(openssl rand -hex 16)"
echo "POSTGRES SECRET: $(openssl rand -hex 8)"
echo "REDIS SECRET: $(openssl rand -hex 8)"
echo "TURN SECRET: $(openssl rand -hex 8)"
echo "SIGNALING SECRET: $(openssl rand -hex 8)"
echo "IMAGINARY SECRET: $(openssl rand -hex 8)"
echo "INTERNAL TALK SECRET: $(openssl rand -hex 8)"
```

После внимательной проверки и выполнения всех подготовительных операций, запустите Nextcloud для первоначальной инициализации в режиме одной реплики:
```bash
docker stack deploy -c nextcloud.yaml nextcloud-aio
```

Отслеживайте работу вашего кластера:
```bash
watch -n 1 'docker stack ps nextcloud-aio'
```

Мы должны увидеть что-то вроде:
```
NAME                                      IMAGE                                                         NODE      DESIRED STATE   CURRENT STATE          ERROR     PORTS
nextcloud-aio_nextcloud-aio-apache        ghcr.io/nextcloud-releases/aio-apache:latest                  WS-2      Running         Running 1 hours ago              *:335->335/udp,*:335->335/tcp
nextcloud-aio_nextcloud-aio-database      ghcr.io/nextcloud-releases/aio-postgresql:latest              CS        Running         Running 1 hours ago
nextcloud-aio_nextcloud-aio-imaginary     ghcr.io/nextcloud-releases/aio-imaginary:latest               WS-1      Running         Running 1 hours ago
nextcloud-aio_nextcloud-aio-nextcloud     ghcr.io/nextcloud-releases/aio-nextcloud:latest               WS-3      Running         Running 1 hours ago
nextcloud-aio_nextcloud-aio-notify-push   ghcr.io/nextcloud-releases/aio-notify-push:latest             WS-2      Running         Running 1 hours ago
nextcloud-aio_nextcloud-aio-onlyoffice    ghcr.io/thomisus/onlyoffice-documentserver-unlimited:latest   CS        Running         Running 1 hours ago             *:336->80/tcp
nextcloud-aio_nextcloud-aio-redis         ghcr.io/nextcloud-releases/aio-redis:latest                   CS        Running         Running 1 hours ago
nextcloud-aio_nextcloud-aio-talk          ghcr.io/nextcloud-releases/aio-talk:latest                    CS        Running         Running 1 hours ago             *:3478->3478/udp,*:3478->3478/tcp
```

Для более подробной информации об установке смотрите логи Nextcloud:
```bash
docker service logs -f nextcloud-aio_nextcloud-aio-nextcloud
```

Как только установка завершится, Nextcloud будет доступен для входа через прокси-сервер по домену. **В случае, если вы проводите установку с  `INSTALL_LATEST_MAJOR=yes`, выполните дополнительное отключение проверки обновлений самим Nextcloud**:
```bash
docker exec --user www-data -it $(docker ps -qf "name=nextcloud-aio-nextcloud") php occ config:system:set updatechecker --type=bool --value=false
```

# Масштабирование (CS)

После того, как вы убедились, что Nextcloud работает, приступаем к масштабированию. Docker Swarm запомнит данное состояние и будет стремиться к его поддержанию, пока вы не выполните`docker stack deploy`, после которого вернется развертывание по 1 реплике (**укажите количество реплик в количестве ваших WS-нод**):
```bash
docker service scale nextcloud-aio_nextcloud-aio-nextcloud=3 nextcloud-aio_nextcloud-aio-apache=3 nextcloud-aio_nextcloud-aio-notify-push=3 nextcloud-aio_nextcloud-aio-imaginary=3
```

Вы должны увидеть что-то вроде:
```
NAME                                      IMAGE                                                         NODE      DESIRED STATE   CURRENT STATE          ERROR     PORTS
nextcloud-aio_nextcloud-aio-apache        ghcr.io/nextcloud-releases/aio-apache:latest                  WS-2      Running         Running 7 hours ago              *:335->335/udp,*:335->335/tcp
nextcloud-aio_nextcloud-aio-apache        ghcr.io/nextcloud-releases/aio-apache:latest                  WS-1      Running         Running 8 hours ago              *:335->335/tcp,*:335->335/udp
nextcloud-aio_nextcloud-aio-apache        ghcr.io/nextcloud-releases/aio-apache:latest                  WS-3      Running         Running 7 hours ago              *:335->335/tcp,*:335->335/udp
nextcloud-aio_nextcloud-aio-database      ghcr.io/nextcloud-releases/aio-postgresql:latest              CS        Running         Running 27 hours ago
nextcloud-aio_nextcloud-aio-imaginary     ghcr.io/nextcloud-releases/aio-imaginary:latest               WS-2      Running         Running 7 hours ago
nextcloud-aio_nextcloud-aio-imaginary     ghcr.io/nextcloud-releases/aio-imaginary:latest               WS-1      Running         Running 8 hours ago
nextcloud-aio_nextcloud-aio-imaginary     ghcr.io/nextcloud-releases/aio-imaginary:latest               WS-3      Running         Running 7 hours ago
nextcloud-aio_nextcloud-aio-nextcloud     ghcr.io/nextcloud-releases/aio-nextcloud:latest               WS-2      Running         Running 2 hours ago
nextcloud-aio_nextcloud-aio-nextcloud     ghcr.io/nextcloud-releases/aio-nextcloud:latest               WS-1      Running         Running 2 hours ago
nextcloud-aio_nextcloud-aio-nextcloud     ghcr.io/nextcloud-releases/aio-nextcloud:latest               WS-3      Running         Running 2 hours ago
nextcloud-aio_nextcloud-aio-notify-push   ghcr.io/nextcloud-releases/aio-notify-push:latest             WS-2      Running         Running 7 hours ago
nextcloud-aio_nextcloud-aio-notify-push   ghcr.io/nextcloud-releases/aio-notify-push:latest             WS-1      Running         Running 8 hours ago
nextcloud-aio_nextcloud-aio-notify-push   ghcr.io/nextcloud-releases/aio-notify-push:latest             WS-3      Running         Running 7 hours ago
nextcloud-aio_nextcloud-aio-onlyoffice    ghcr.io/thomisus/onlyoffice-documentserver-unlimited:latest   CS        Running         Running 27 hours ago             *:336->80/tcp
nextcloud-aio_nextcloud-aio-redis         ghcr.io/nextcloud-releases/aio-redis:latest                   CS        Running         Running 27 hours ago
nextcloud-aio_nextcloud-aio-talk          ghcr.io/nextcloud-releases/aio-talk:latest                    CS        Running         Running 27 hours ago             *:3478->3478/udp,*:3478->3478/tcp
```

# Обновление (CS)
Кратко, процесс обновления представляет собой:
1. Указать в yaml файле [последние версии образов AIO](https://github.com/orgs/nextcloud-releases/packages) и запустить Nextcloud в исходном состоянии с 1 репликой
2. Применить изменения и проследить за обновлением
3. В случае использования `INSTALL_LATEST_MAJOR=yes` - выполнить дополнительное обновление вручную
4. Масштабировать

Узнать текущую версию Nextcloud вы можете данной командой (версия прописана в самом верху списка):
```bash
docker exec --user www-data -it $(docker ps -qf "name=nextcloud-aio-nextcloud") php occ -v
```

Измените версии AIO-образов в .yaml файле. Найти их вы можете [тут](https://github.com/orgs/nextcloud-releases/packages). Применяем изменения:
```bash
docker stack deploy -c nextcloud.yaml nextcloud-aio
```

Обновление происходит автоматически, проследить за этим вы можете данной командой:
```bash
docker service logs -f nextcloud-aio_nextcloud-aio-nextcloud
```

**Если при установке вы использовали флаг `INSTALL_LATEST_MAJOR=yes`, смена версии образа может не запустить автоматического обновления, т.к. версия Nextcloud в образах сильно отстает от мажорных обновлений. В таком случае, вам необходимо выполнить ручное обновление Nextcloud до последней версии:**
```bash
docker exec --user www-data -it $(docker ps -qf "name=nextcloud-aio-nextcloud") php /var/www/html/updater/updater.phar --no-interaction --no-backup
```

После того, как вы убедились, что обновление прошло успешно, масштабируйте (**укажите количество реплик в количестве ваших WS-нод**):
```bash
docker service scale nextcloud-aio_nextcloud-aio-nextcloud=3 nextcloud-aio_nextcloud-aio-apache=3 nextcloud-aio_nextcloud-aio-notify-push=3 nextcloud-aio_nextcloud-aio-imaginary=3
```

# Развертывание в нескольких сетях для их разгрузки (CS)

Вы можете развернуть данный инстанс с разделением на максимум 3 сети:
- Сеть А (192.168.0.*) - трафик Nginx -> Apache
- Сеть Б (192.168.1.*) - трафик Docker Overlay (общение Nextcloud контейнеров)
- Сеть В (192.168.2.*) - трафик Nextcloud -> NFS

**Для подобной реализации с всех нод должен быть доступ к всем сетям. Переход на данную схему рекомендуется выполнять после установки и проверки основного инстанса. Если развертывание производится с нуля и сразу в несколько подсетей - пожалуйста изучите основную установку, т.к. в ней описаны действия необходимые для обязательного отключения обновлений и их фиксации.**

Перед началом, удалите текущую установку Nextcloud (файлы сохраняются):
```bash
docker stack rm nextcloud-aio
```

Выйдите из текущего кластера, выполнив команду на всех нодах:
```bash
docker swarm leave --force
```

Повторно создайте кластер, изменив IP-адрес подсети:
```bash
docker swarm init --advertise-addr 192.168.1.1 # CS
docker swarm join --token <TOKEN> 192.168.1.1:2377 # WS
```

Для перевода вашего хранилища NFS редактируйте файл exports:
```bash
nano /etc/exports
```

Замените на новый IP-адрес подсети:
```bash
/mnt 192.168.2.0/24(rw,async,no_subtree_check,no_root_squash,crossmnt)
```

Примените изменения:
```bash
exportfs -a
```

После данных действий, вам необходимо изменить IP-адреса NFS в .yaml файле и запустить:
```bash
docker stack deploy -c nextcloud.yaml nextcloud-aio
```

После того, как вы убедились, что Nextcloud работает, приступаем к масштабированию. Docker Swarm запомнит данное состояние и будет стремиться к его поддержанию, пока вы не выполните`docker stack deploy`, после которого вернется развертывание по 1 реплике (**укажите количество реплик в количестве ваших WS-нод**):
```bash
docker service scale nextcloud-aio_nextcloud-aio-nextcloud=3 nextcloud-aio_nextcloud-aio-apache=3 nextcloud-aio_nextcloud-aio-notify-push=3 nextcloud-aio_nextcloud-aio-imaginary=3
```

# Отключение защиты от брутфорса и ratelimit (CS)
Добавьте в env сервиса Nextcloud в вашем yaml файле строку `- DISABLE_BRUTEFORCE_PROTECTION=yes`.

Примените изменения:
```bash
docker stack deploy -c nextcloud.yaml nextcloud-aio
```

После того, как вы убедились, что защита от брутфорса и ratelimit отключена, приступаем к масштабированию. Docker Swarm запомнит данное состояние и будет стремиться к его поддержанию, пока вы не выполните`docker stack deploy`, после которого вернется развертывание по 1 реплике (**укажите количество реплик в количестве ваших WS-нод**):
```bash
docker service scale nextcloud-aio_nextcloud-aio-nextcloud=3 nextcloud-aio_nextcloud-aio-apache=3 nextcloud-aio_nextcloud-aio-notify-push=3 nextcloud-aio_nextcloud-aio-imaginary=3
```
