# vicha2_microservices
vicha2 microservices repository
<details><summary>ДЗ№16 Docker контейнеры. Docker под капотом.</summary>

- Create VM
```
yc compute instance create \
--name docker-host \
--zone ru-central1-a \
--network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
--create-boot-disk image-folder-id=standard-images,image-family=ubuntu-1804-lts,size=15 \
--ssh-key ~/.ssh/id_rsa.pub
```
- Docker-machine
```
docker-machine create \
--driver generic \
--generic-ip-address=62.84.118.194 \
--generic-ssh-user yc-user \
--generic-ssh-key ~/.ssh/id_rsa \
docker-host

docker-machine ls
eval $(docker-machine env docker-host)
```
- Сборка образа
```
docker build -t reddit:latest .
```
- Запуск и проверка приложения
```
docker run --name reddit -d --network=host reddit:latest
docker ps
http://<your public IP>:9292/
```
- Docker HUB PUSH
```
docker login
docker images
docker tag reddit:latest vicha2/otus-reddit:1.0
docker images
docker push vicha2/otus-reddit:1.0
```
- Проверка
```
eval $(docker-machine env -u)
docker images
docker run --rm --name reddit -d -p 9292:9292 vicha2/otus-reddit:1.0
docker ps
http://<your local IP>:9292/
```
- Удаление docker-machine
```
docker-machine rm docker-host -y
yc compute instance delete docker-host
```
</details>

<details><summary>ДЗ№17 Docker образы. Микросервисы.</summary>

- Create VM
```
yc compute instance create \
--name docker-host \
--zone ru-central1-a \
--network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
--create-boot-disk image-folder-id=standard-images,image-family=ubuntu-1804-lts,size=15 \
--ssh-key ~/.ssh/id_rsa.pub
```
- RUN Docker-machine
```
docker-machine create \
--driver generic \
--generic-ip-address=<your public IP> \
--generic-ssh-user yc-user \
--generic-ssh-key ~/.ssh/id_rsa \
docker-host

docker-machine ls
eval $(docker-machine env docker-host)
```
- Сборка образов
```
docker build -t vicha2/post:1.0 ./post-py/
docker build -t vicha2/comment:1.0 ./comment/
docker build -t vicha2/ui:1.0 ./ui/
```
- Создаем сеть для приложения
```
docker network create reddit
```
- Запуск контейнеров
```
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db mongo:latest
docker run -d --network=reddit --network-alias=post vicha2/post:1.0
docker run -d --network=reddit --network-alias=comment vicha2/comment:1.0
docker run -d --network=reddit -p 9292:9292 vicha2/ui:1.0
```
- Задание со *
```
docker kill $(docker ps -q)
docker run -d --network=reddit --network-alias=post_db_2 --network-alias=comment_db_2 mongo:latest
docker run -d -e POST_DATABASE_HOST=post_db_2 --network=reddit --network-alias=post_2 vicha2/post:1.0
docker run -d -e COMMENT_DATABASE_HOST=comment_db_2 --network=reddit --network-alias=comment_2 vicha2/comment:1.0
docker run -d -e POST_SERVICE_HOST=post_2 -e COMMENT_SERVICE_HOST=comment_2 --network=reddit -p 9292:9292 vicha2/ui:1.0
```
- Пересобираем образ UI
```
docker build -t vicha2/ui:2.0 ./ui/    # from ubuntu
```
- Перезапускаем контейнеры с новой версией
```
docker kill $(docker ps -q)
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db mongo:latest
docker run -d --network=reddit --network-alias=post vicha2/post:1.0
docker run -d --network=reddit --network-alias=comment vicha2/comment:1.0
docker run -d --network=reddit -p 9292:9292 vicha2/ui:2.0
```
- Создаем volume и переподключаемся
```
docker volume create reddit_db
docker kill $(docker ps -q)
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db -v reddit_db:/data/db mongo:latest
docker run -d --network=reddit --network-alias=post vicha2/post:1.0
docker run -d --network=reddit --network-alias=comment vicha2/comment:1.0
docker run -d --network=reddit -p 9292:9292 vicha2/ui:2.0
```
- Удаление docker-machine
```
docker-machine rm docker-host -y
eval $(docker-machine env -u)
yc compute instance delete docker-host
```
</details>
<details><summary>ДЗ№18 Сетевое взаимодействие Docker контейнеров. Docker Compose. Тестирование образов.</summary>

- Создаем ВМ и подключаемся через docker-machime (см. предыдущее занятие)
- None network driver
```
docker run -it --rm --network none joffotron/docker-net-tools -c ifconfig
```
- Host network driver
```
docker run -it --rm --network host joffotron/docker-net-tools -c ifconfig
```
> Сравните вывод команды с : docker-machine ssh docker-host ifconfig

На docker-machine ip хостовой машины
- Запуск несколько раз:
```
docker run --network host -d nginx
```
> Каков результат? Что выдал docker ps? Как думаете почему?

Работать будет только первый контейнер, т.к. 80 порт на хостовой машине будет занят им.
- Bridge network driver
```
docker network create reddit --driver bridge
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db mongo:latest
docker run -d --network=reddit --network-alias=post vicha2/post:1.0
docker run -d --network=reddit --network-alias=comment vicha2/comment:1.0
docker run -d --network=reddit -p 9292:9292 vicha2/ui:1.0
```
## Docker compose
```
export USERNAME=vicha2
docker-compose up -d
docker-compose ps
```
```
version: '3.3'
services:
  post_db:
    image: mongo:3.2
    volumes:
      - post_db:/data/db
    networks:
      back_net:
        aliases:
          - comment_db
          - post_db
  ui:
    build: ./ui
    image: ${USERNAME}/ui:${TAG}
    ports:
      - ${UIPORT}:${UIPORT}/tcp
    networks:
      - front_net
  post:
    build: ./post-py
    image: ${USERNAME}/post:${TAG}
    networks:
      back_net:
        aliases:
          - post
      front_net:
        aliases:
          - post
  comment:
    build: ./comment
    image: ${USERNAME}/comment:${TAG}
    networks:
      back_net:
        aliases:
          - comment
      front_net:
        aliases:
          - comment

volumes:
  post_db:

networks:
  back_net:
  front_net:

```
> Узнайте как образуется базовое имя проекта. Можно
ли его задать? Если можно то как?

Базовое имя образуется из имени папки с проектом src_******
Изменить можно добавив параметр -p, --project-name NAME

- Задание со *

docker-compose.override.yml
```
version: '3.3'
services:
  ui:
    command: puma --debug -w 2
  comment:
    command: puma --debug -w 2
```
```
docker-compose -f docker-compose.yml -f docker-compose.override.yml up -d
```

    Name                  Command             State                    Ports                  
----------------------------------------------------------------------------------------------
src_comment_1   puma --debug -w 2             Up                                              
src_post_1      python3 post_app.py           Up                                              
src_post_db_1   docker-entrypoint.sh mongod   Up      27017/tcp                               
src_ui_1        puma --debug -w 2             Up      0.0.0.0:9292->9292/tcp,:::9292->9292/tcp

- Удаление docker-machine
```
docker-machine rm docker-host -y
eval $(docker-machine env -u)
yc compute instance delete docker-host
```
</details>
<details><summary>ДЗ№22 Введение в мониторинг. Модели и принципы работы систем мониторинга.</summary>

- Создаем ВМ
```
yc compute instance create \
  --name docker-host \
  --zone ru-central1-a \
  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
  --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-1804-lts,size=15 \
  --ssh-key ~/.ssh/id_rsa.pub
```
- Create Docker VM
```
docker-machine create \
  --driver generic \
  --generic-ip-address=<your Public IP> \
  --generic-ssh-user yc-user \
  --generic-ssh-key ~/.ssh/id_rsa \
  docker-host
eval $(docker-machine env docker-host)
```
- Запуск Prometheus
```
docker run --rm -p 9090:9090 -d --name prometheus prom/prometheus
```
- Создаем образ
```
export USER_NAME=vicha2
docker build -t $USER_NAME/prometheus .
```
- Docker HUB
```
https://hub.docker.com/repository/docker/vicha2/prometheus
https://hub.docker.com/repository/docker/vicha2/post
https://hub.docker.com/repository/docker/vicha2/comment
https://hub.docker.com/repository/docker/vicha2/ui
```
</details>
<details><summary>ДЗ№25 Применение системы логирования в инфраструктуре на основе Docker .</summary>

- Создаем ВМ
```
yc compute instance create \
  --name logging \
  --zone ru-central1-a \
  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
  --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-1804-lts,size=30 \
  --memory 8 \
  --ssh-key ~/.ssh/id_rsa.pub
```
- Create Docker VM
```
docker-machine create \
  --driver generic \
  --generic-ip-address=51.250.73.131 \
  --generic-ssh-user yc-user \
  --generic-ssh-key ~/.ssh/id_rsa \
  logging
eval $(docker-machine env logging)
```
- Запускаем приложение
```
docker-compose -f docker-compose-logging.yml up -d
docker-compose up -d
```



- Удаление docker-machine
```
docker-machine rm logging -y
eval $(docker-machine env -u)
yc compute instance delete logging
```

</details>
