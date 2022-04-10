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
<details><summary>ДЗ№27 Введение в Kubernetes #1.</summary>

- Создаем файлы манифестов deployment
- Создаем две ноды:
  - Master
```
yc compute instance create \
  --name master \
  --hostname master \
  --zone ru-central1-a \
  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
  --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2004-lts,size=40 \
  --memory 4 \
  --cores 4 \
  --ssh-key ~/.ssh/id_rsa.pub
```
  - Worker
```
yc compute instance create \
  --name worker \
  --hostname worker \
  --zone ru-central1-a \
  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
  --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2004-lts,size=40 \
  --memory 4 \
  --cores 4 \
  --ssh-key ~/.ssh/id_rsa.pub
```
- Подключаемся к master
```
ssh yc-user@51.250.73.153 #IP your master node
```
- Установка Docker
```
sudo apt-get remove docker docker-engine docker.io containerd runc
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```
- Установка kubeadm
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo apt install kubeadm=1.19.16-00 kubelet=1.19.16-00 kubectl=1.19.16-00
```
- Установка K8s control plane
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
- Запуск master
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
- Подключаемся к worker
```
ssh yc-user@51.250.67.16 #IP worker nodes
```
- Устанавливаем Docker и kubeadm по аналогии с master
- Установка CNI Calico
```
curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O       #Edit CALICO_IPV4POOL_CIDR
kubectl apply -f calico.yaml
```
- Проверка манифестов
```
kubectl apply -f vicha2_microservices/kubernetes/reddit/
kubectl get po
```
- Удаляем ВМ
```
yc compute instance delete master worker
```
</details>
  
<details><summary>ДЗ№28 Введение в Kubernetes #2.</summary>

- Запускаем minikube в docker
```
minikube start --driver=docker --cpus=4 --memory=8g
kubectl get nodes
kubectl config current-context
kubectl config get-contexts
```
- Пробрасываем порт и проверяем
```
kubectl apply -f ui-deployment.yml
kubectl port-forward <pod-name>9292:9292
```
- Подключаемся к кластеру Yandex
```
yc managed-kubernetes cluster get-credentials test-cluster --external
kubectl config current-context
kubectl apply -f dev-namespace.yml
kubectl -n dev apply -f .
kubectl get nodes -o wide
kubectl -n dev describe service ui
```
</details>

<details><summary>ДЗ№30 Ingress-контроллеры и сервисы в Kubernetes</summary>

- Подключаемся к кластеру Yandex
```
yc managed-kubernetes cluster get-credentials cluster-test --external --force
kubectl config current-context
kubectl apply -f dev-namespace.yml
kubectl -n dev apply -f .
kubectl -n dev get po
kubectl get nodes -o wide
```
- Loadbalancer
```
kubectl -n dev apply -f ui-service.yml
kubectl -n dev get svc ui
```
- Устанавливаем Ingress
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.34.1/deploy/static/provider/cloud/deploy.yaml
kubectl -n ingress-nginx get svc
```
- Secret
```
kubectl -n dev get ingress ui
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=51.250.71.0"
kubectl create secret tls ui-ingress --key tls.key --cert tls.crt -n dev
kubectl -n dev get secrets
kubectl -n dev apply -f ui-ingress.yml
```
- Задание со *
> Опишите создаваемый объект Secret в виде Kubernetes-манифеста.
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=51.250.71.0"
cat tls.crt | base64
cat tls.key | base64

---
apiVersion: v1
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUREVENDQWZXZ0F3SUJBZ0lVWCswZTRNdXRhRE9SY21SZm00djFkQi9tSnA4d0RRWUpLb1pJaHZjTkFRRUwKQlFBd0ZqRVVNQklHQTFVRUF3d0xOVEV1TWpVd0xqY3hMakF3SGhjTk1qSXdOREV3TVRNeU5qVXlXaGNOTWpNdwpOREV3TVRNeU5qVXlXakFXTVJRd0VnWURWUVFEREFzMU1TNHlOVEF1TnpFdU1EQ0NBU0l3RFFZSktvWklodmNOCkFRRUJCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFLclFJTkU4NjE0MXQ2Tk5Sd0NsMTJ2U0RpSVl0M1FEeHlBdkh5dnIKZ1loWkRTYUtFQVA3b2VCdmNWU3dZWnlreThtbU43RVFqenFZVE5QRUtYcVp2d0RKdkVCMkdyMXQ3VTlrZkNFbApiYUZnQU1sVUVLbTVJMXNyUTMreTR1SjhtQ01OVmpjWGlNUWlEd2FwYmtuWUhkQzdldEFEd0FqZ251NFB6Q3hvCk9oSTU5L2t3bEJ0a3VJU0RvUUpYVGFFL21tL0M1dWhVN1VmRU94ckI1c3Znazg4S0NUcVdCcmRMVGNUOE9YRncKRU5Rankyd0dtTU42c3lleXowcSsvU0NpbVpoNXZzTzkzbGphc2F0c1E0a21OT1V4ampwNnVhdk4xTHB0aExZRApWTkVGcUxvNGR5RW5hZE1weFVmMVVQMTZWb1dDNE5nSml3TUdEc2RnSU9uQzl1Y0NBd0VBQWFOVE1GRXdIUVlEClZSME9CQllFRk16ZlhjSm90VDZ2bkhmSlRYVnRDYWpHTCtsRU1COEdBMVVkSXdRWU1CYUFGTXpmWGNKb3RUNnYKbkhmSlRYVnRDYWpHTCtsRU1BOEdBMVVkRXdFQi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQgpBQ2F2eXZ5UzB3a0laZXJYNmFRd01FTG5jbTVjTmRzTkoraG05UmE0OTZlMDBlemZwUlorc0N0WjRXYzlaRUhLCklQUEJmaVNrR2xkcEdZMndVZmk3RThIdU5waDFLRkc1UWhqaFFBTkxycGRuT1dTOTJ1T0lOWEZaSE1zdnNLYy8KbmhxekJwTG1CdDl6azFSaWlGaHIvcHFDNkluNFlXTFdhNnVteDZkcXF0SnZ1NHVhMW01SndwZzNzK0pFODVjbQpRcEJMUDBjWVhkK0RQeUhMaXRxb3l2elkyWmRkbWNOcUhPU3ZTVDlZZHZjUDArNjBmL1YzUm4xU2ZyN1FLY0pYClNTcy9OVjRsM1VWblJrQUhDbURuWDNXNTh1RWZFRncxVHBzWEhtSHNsTHFnak1nUkVuTlVjTGdLQnhnWFpQMzcKc0xVNk1CNzZzalVhSE1zL1BIQTk1SFE9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2QUlCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQktZd2dnU2lBZ0VBQW9JQkFRQ3EwQ0RSUE90ZU5iZWoKVFVjQXBkZHIwZzRpR0xkMEE4Y2dMeDhyNjRHSVdRMG1paEFEKzZIZ2IzRlVzR0djcE12SnBqZXhFSTg2bUV6VAp4Q2w2bWI4QXlieEFkaHE5YmUxUFpId2hKVzJoWUFESlZCQ3B1U05iSzBOL3N1TGlmSmdqRFZZM0Y0akVJZzhHCnFXNUoyQjNRdTNyUUE4QUk0Sjd1RDh3c2FEb1NPZmY1TUpRYlpMaUVnNkVDVjAyaFA1cHZ3dWJvVk8xSHhEc2EKd2ViTDRKUFBDZ2s2bGdhM1MwM0UvRGx4Y0JEVUk4dHNCcGpEZXJNbnNzOUt2djBnb3BtWWViN0R2ZDVZMnJHcgpiRU9KSmpUbE1ZNDZlcm1yemRTNmJZUzJBMVRSQmFpNk9IY2hKMm5US2NWSDlWRDllbGFGZ3VEWUNZc0RCZzdICllDRHB3dmJuQWdNQkFBRUNnZ0VBUjkxN0FTMWxSV1RLVjAxckF3M0RQWnpKejNTZ3NwSG9WRlVmQTBaNVlCay8KWENpWUptVFhMV3NWdm5EYkVLR1JEOHo3LzJZZExLVHBKZXVSSEFEVmlJcFh4ck1wK3VybC9oSWoyM280enIxcQpkMG9FSExSRStOV1I5NGNXeC8xdHNNbXFyVkVjZkpCcnkveTY1eHlqSnEvS012eHc3Z3M3TXFPNDNqSVh4SlNvCjBJWjFFWGRqU09CNVdWVHo4Y3Z1UHY1OWIweUU2RHhDU3ZidWI0OEU1YklYTkRlTnhsVEE2MHowcVlwSC9YNXAKaGlHTUhYcE1RNjFYdkFjZ3dvSU4vL0E4M3FMZ1BNNlNWS0NHMnNlWmJHMGtmWUJDd1dJNTJ4b2VobFg2SDg5NAoxL1NhU3FWYStkemhkWTU0NTdLOEE2b0hJcyt5QzhiaEV5NGtnc3J4eVFLQmdRRGRKRVIrNEhwalN4bm9mYmdnClpwa2RHUnJ2SHdaZzE4aGhLdVA2M1R5MTJGZzZLSEtwYkEvNlhISVZJZUwxVGJTeWE4ZWszMkZJbTVCNithZFUKWkszRXRwZDlMSzFpMm9EQnVWTGN6UW50SmRDbVhqeENDUUEzOHI3MkRUSHBhbk5OT1hiR2ppTDQ5bFZlSWRhMgpyQVRGUmxOSEFGUnc1dXJmWmt1QWRIQlVKUUtCZ1FERnZQSitNMlpuTzgrRGFhOVBlMU96SnQ4ZEVMdUVxTXMwCmlGTWRqWkVubDRuVXUwUU9GdDh4cGswSUx2MjkwYjdqVlBKK05kOXlUemFWamh4dTdHQmd1K2lCcFFmU09QWU4Kejltb3ZDNDNSdGUxMGdydnIrZFl2L1VhbStPeVF2TjNkcmRKODNWSm15QUF6OHNjMzk0T280cUdmczBZSlhtUwpkbGNGMFRLTEd3S0JnRzFvakJyWnBMT0xiSDRCOVI3U285NHBsWkhJbjdjNkN3RkgzeE0yY2RybDlvQ1BrbXNQCjg3ZkNGUTh2Zk1Jd2Q3M3VaUS9GRkxSL2dyUFU0Rng0a3lCSDFoc3dCM2hvOGxybC9ZRVFVR0RyM0pieStJMFQKTnZCM1FOTXJKQTUvaEJ3bzJnTFNQNnM4OUc5bC9uelNEbW9ycVBmdnlkY3g1L0l2QWh2RGYrK2hBb0dBYytaTQp4L1crcHZHaXJ1YnFMNDhjdnh3Z21EdXZmWkVtRWZONXJBL0hMY3FmcWdYZFhOakJGNnZlNk5ZS09oRlBicFhpCjBHRXBTQ252MTNjRmFXcTVEdG4wN05CYkpqZm0ySytrWjBkdFcwNzFyb2VmaTEreUhRM2VUeXRpS2FFZWJUNHoKTG5BNXBkdjd4UjRHY2pVeFJhbEx6NHRSRVQ4ZDQ5L2pIL0MvVEZNQ2dZQlhXQW54THdTV3ZxWk81SVhwMExUbQpjaGFrOGtSRkpaSGhyU2VYS1licGY5WG96cVZMeks2dmJwMFNkUk45ejdNVEVsOVpxaEhFUE5JaXNwZGl2dXlTCi91eWtxQjQ5dUQyM1VOd0MzMkZBWm42Y2FhVHNhQ1RGT0hob3U2Mlc0OVRNWSsxRkxRS3ZoTHIyNVFXTnJLcEwKUHV0Y0liZnpKN214TTRGcER1SmdZZz09Ci0tLS0tRU5EIFBSSVZBVEUgS0VZLS0tLS0K
kind: Secret
metadata:
  name: ui-ingress
  namespace: dev
type: kubernetes.io/tls
```
- PersitentVolume
```
yc compute disk create \
--name k8s \
--size 4 \
--description "disk for k8s" \
--zone "ru-central1-a"

yc compute disk list
kubectl -n dev create -f pv.yml
kubectl -n dev create -f pvc.yml
```
- Удаляем кластер и диск
```
yc managed-kubernetes cluster delete --name cluster-test
yc compute disk delete --name k8s
```
</details>
