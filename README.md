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
--generic-ip-address=84.201.158.149 \
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
