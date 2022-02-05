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
--generic-ip-address=51.250.14.115 \
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
