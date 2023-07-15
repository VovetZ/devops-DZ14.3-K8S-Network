# devops-DZ14.3-K8S-Network
# Домашнее задание к занятию «Как работает сеть в K8s»

### Цель задания

Настроить сетевую политику доступа к подам.

### Чеклист готовности к домашнему заданию

1. Кластер K8s с установленным сетевым плагином Calico.

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Документация Calico](https://www.tigera.io/project-calico/).
2. [Network Policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/).
3. [About Network Policy](https://docs.projectcalico.org/about/about-network-policy).

-----

### Задание 1. Создать сетевую политику или несколько политик для обеспечения доступа

1. Создать deployment'ы приложений frontend, backend и cache и соответсвующие сервисы.
2. В качестве образа использовать network-multitool.
3. Разместить поды в namespace App.
4. Создать политики, чтобы обеспечить доступ frontend -> backend -> cache. Другие виды подключений должны быть запрещены.
5. Продемонстрировать, что трафик разрешён и запрещён.

### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

-----

# Ответ

### Подготовка

1. Зарегистрировался в [Яндекс Облаке](console.cloud.yandex.ru)
2. Создал платёжный аккаунт с промо-кодом  
3. Установил Yandex Cloud (CLI) `yc`  

```bash
vk@vkvm:~/14.3$ curl -sSL https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash
Downloading yc 0.108.1
  ....
Yandex Cloud CLI 0.108.1 linux/amd64
To complete installation, start a new shell (exec -l $SHELL) or type 'source "/home/vk/.bashrc"' in the current one
vk@vkvm:~/14.3$ source "/home/vk/.bashrc"
```  

5. Настроил `yc` (OAuth токен взял [по адресу](https://oauth.yandex.ru/authorize?response_type=token&client_id=1a6990aa636648e9b2ef855fa7bec2fb)), проверил настройки

```bash
vk@vkvm:~/14.3$ yc init
Welcome! This command will take you through the configuration process.
......
vk@vkvm:~/14.3$ yc config list
token: y0_AgAAAAA.....
cloud-id: b1ggidd5mp363gkiouv2
folder-id: b1g12il7uurh00pne60k
compute-default-zone: ru-central1-a
```

7. Установил токен и параметры в переменные окружения  

```bash
vk@vkvm:~/14.3$ export YC_TOKEN=$(yc iam create-token)
vk@vkvm:~/14.3$ export YC_CLOUD_ID=$(yc config get cloud-id)
vk@vkvm:~/14.3$ export YC_FOLDER_ID=$(yc config get folder-id)
vk@vkvm:~/14.3$ export YC_ZONE=$(yc config get compute-default-zone)
```

8. Сгенерировал SSH ключи на локальной машине  

```bash
ssh-keygen -f ~/.ssh/yandex
...
Your identification has been saved in /home/vk/.ssh/yandex
Your public key has been saved in /home/vk/.ssh/yandex.pub
...
```

9. Создал ресурсы в Яндекс Облаке

- 1 Master node (2 Gb RAM, 2 CPU, 18 Gb SSD)
- 3 Worker node (2 Gb RAM, 2 CPU, 18 Gb SSD)
- OS `Ubuntu 22.04`
- Публичный ключ (`/home/vk/.ssh/yandex.pub`)
- User `vkadm`

```bash
vk@vkvm:~/14.3$ yc compute instance list
vk@vkvm:~$ yc compute instance list
+----------------------+-----------+---------------+---------+----------------+---------------+
|          ID          |   NAME    |    ZONE ID    | STATUS  |  EXTERNAL IP   |  INTERNAL IP  |
+----------------------+-----------+---------------+---------+----------------+---------------+
| fhm1of85oe9hda0us9cs | worker-01 | ru-central1-a | RUNNING | 158.160.33.178 | 192.168.10.28 |
| fhm5q505jtavk87jdrrf | worker-03 | ru-central1-a | RUNNING | 158.160.61.184 | 192.168.10.37 |
| fhm99rcht4nr6laf2n4u | master-01 | ru-central1-a | RUNNING | 158.160.54.220 | 192.168.10.8  |
| fhmr23edrpda3bphopo1 | worker-02 | ru-central1-a | RUNNING | 158.160.55.142 | 192.168.10.14 |
+----------------------+-----------+---------------+---------+----------------+---------------+
```

10. Установил `kubectl` на локальной машине

```bash
apt-get update && apt-get install -y ca-certificates curl apt-transport-https
mkdir -p /etc/apt/keyrings
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubectl
apt-mark hold kubectl
```

11. Установил кластер с помощью `kubespray`

**На master node**

```bash
vk@vkvm:~$ ssh -i ~/.ssh/yandex vkadm@158.160.54.220 
vkadm@master-01:~$ sudo bash -i
root@master-01:~# apt-get update -y && apt-get install git pip -y
root@master-01:~# git clone https://github.com/kubernetes-sigs/kubespray
root@master-01:~# cd kubespray
root@master-01:~# pip3 install -r requirements.txt
root@master-01:~# cp -rfp inventory/sample inventory/mycluster
```

В переменной `IPS`  указал локальные IP нод, на первом месте - master node

```bash
declare -a IPS=(192.168.10.8 192.168.10.28 192.168.10.14 192.168.10.37)
```

Сгенерировал inventory для `Ansible`, отредактировал его

```bash
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
cat inventory/mycluster/hosts.yaml
```

```yml
all:
  hosts:
    node1:
      ansible_host: 192.168.10.8
      ip: 192.168.10.8
      access_ip: 192.168.10.8
    node2:
      ansible_host: 192.168.10.28
      ip: 192.168.10.28
      access_ip: 192.168.10.28
    node3:
      ansible_host: 192.168.10.14
      ip: 192.168.10.14
      access_ip: 192.168.10.14
    node4:
      ansible_host: 192.168.10.37
      ip: 192.168.10.37
      access_ip: 192.168.10.37
  children:
    kube_control_plane:
      hosts:
        node1:
    kube_node:
      hosts:
        node2:
        node3:
        node4:
    etcd:
      hosts:
        node1:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```

Отредактировал `inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml`, добавил имя и внешний адрес master node  в сертификат

```bash
root@master-01:/home/vkadm/kubespray# grep supplement inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
supplementary_addresses_in_ssl_keys: [158.160.54.220,  master-01.ru-central1-a.internal]
```

Скопировал закрытый ключ с локальной машины на master node

```bash
vk@vkvm:~$ rsync --rsync-path="sudo rsync" ~/.ssh/yandex vkadm@158.160.54.220:/root/.ssh/yandex
vk@vkvm:~$ ssh -i ~/.ssh/yandex vkadm@158.160.54.220
vkadm@master-01:~$ sudo bash -i
root@master-01:~# chmod 600 /root/.ssh/yandex
```

Применил конфигурацию Ansible

```bash
root@master-01:~# cd kubespray
root@master-01:~# ansible-playbook -i inventory/mycluster/hosts.yaml -u vkadm -b -v --private-key=/root/.ssh/yandex cluster.yml
```

Создал `kubeconfig` на master node

```bash
root@master-01:~# mkdir -p $HOME/.kube
root@master-01:~# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
root@master-01:~# chown $(id -u):$(id -g) $HOME/.kube/config
```

Сохранил kubeconfig из мастер ноды на локальную машину, заменил адрес мастер ноды на внешний IP

```bash
vk@vkvm:~$ mkdir -p $HOME/.kube
vk@vkvm:~$ rsync --rsync-path="sudo rsync" vkadm@158.160.54.220:/root/.kube/config $HOME/.kube/config
vk@vkvm:~$ sed -i 's/127\.0\.0\.1/158\.160\.54\.220/g' $HOME/.kube/config
vk@vkvm:~$ chown $(id -u):$(id -g) $HOME/.kube/config
```

Проверил состояние нод, убедился что **кластер готов**

```bash
vk@vkvm:~$ kubectl get nodes
NAME    STATUS   ROLES           AGE     VERSION
node1   Ready    control-plane   9m11s   v1.26.6
node2   Ready    <none>          8m1s    v1.26.6
node3   Ready    <none>          8m2s    v1.26.6
node4   Ready    <none>          8m2s    v1.26.6
```

# Задание 1

1. Подготовка **namespace**. Создал `namespace.yml`. Применил конфигурацию

```yml
apiVersion: v1
kind: Namespace
metadata:
  name: app
```

[Ссылка на namespace.yml](namespace.yml)

```bash
vk@vkvm:~/14_3$ kubectl create -f namespace.yml
namespace/app created
```

2. Подготовка **frontend**. Создал `deploy-front.yml`, `service-front.yml`. Применил конфигурацию

```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-front
  name: deploy-front
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend-multitool
          image: wbitt/network-multitool
```

[Ссылка на deploy-front.yml](deploy-front.yml)

```yml
---
apiVersion: v1
kind: Service
metadata:
  name: service-front
  namespace: app
spec:
  selector:
    app: frontend
  ports:
    - name: port-80
      port: 80
      protocol: TCP
      targetPort: 80
```

[Ссылка на service-front.yml](service-front.yml)

```bash
vk@vkvm:~/14_3$ kubectl create -f deploy-front.yml -f service-front.yml
deployment.apps/deploy-front created
service/service-front created
```

2. Подготовка **backend**. Создал `deploy-back.yml`, `service-back.yml`. Применил конфигурацию

```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deployment-back
  name: deployment-back
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend-multitool
          image: wbitt/network-multitool
```

[Ссылка на deploy-back.yml](deploy-back.yml)

```yml
---
apiVersion: v1
kind: Service
metadata:
  name: service-backend
  namespace: app
spec:
  selector:
    app: backend
  ports:
    - name: port-80
      port: 80
      protocol: TCP
      targetPort: 80
```

[Ссылка на service-back.yml](service-back.yml)

```bash
vk@vkvm:~/14_3$ kubectl create -f deploy-back.yml -f service-back.yml
deployment.apps/deployment-back created
service/service-backend created
```

3. Подготовка **cache**. Создал `deploy-cache.yml`, `service-cache.yml`. Применил конфигурацию

```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-cache
  name: deploy-cache
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cache
  template:
    metadata:
      labels:
        app: cache
    spec:
      containers:
        - name: cache-multitool
          image: wbitt/network-multitool
```

[Ссылка на deploy-cache.yml](deploy-cache.yml)

```yml
---
apiVersion: v1
kind: Service
metadata:
  name: service-cache
  namespace: app
spec:
  selector:
    app: cache
  ports:
    - name: port-80
      port: 80
      protocol: TCP
      targetPort: 80
```

[Ссылка на service-cache.yml](service-cache.yml)

```bash
vk@vkvm:~/14_3$ kubectl create -f deploy-cache.yml -f service-cache.yml
deployment.apps/deploy-cache created
service/service-cache created
```

4. Создание политик

- Проверил сетевой доступ между сервисами

```bash
vk@vkvm:~/14_3$ kubectl exec -it service/service-front -n app -- curl --silent -i service-backend.app.svc.cluster.local | grep Server
Server: nginx/1.20.2
vk@vkvm:~/14_3$ kubectl exec -it service/service-backend -n app -- curl --silent -i service-cache.app.svc.cluster.local | grep Server
Server: nginx/1.20.2
vk@vkvm:~/14_3$ kubectl exec -it service/service-front -n app -- curl --silent -i service-cache.app.svc.cluster.local | grep Server
Server: nginx/1.20.2
```

Убедился, что **сетевой доступ - полный**

- Создал файл `netpolicy-default.yml`. Политика, запрещающая подключения, не разрешённые специально

```yml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

[Ссылка на netpolicy-default.yml](netpolicy-default.yml)

- Создал `netpolicy-back.yml`. Политика, разрешающая подключения `frontend` -> `backend` по портам `80`, `443`

```yml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend-policy
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: frontend
      ports:
        - protocol: TCP
          port: 80
        - protocol: TCP
          port: 443
```

[Ссылка на netpolicy-back.yml](netpolicy-back.yml)

- Создал `netpolicy-cache.yml`. Политика, разрешающая подключения `backend` -> `cache` по портам `80`,`443`

```yml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-to-cache-policy
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: cache
  policyTypes:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: backend
      ports:
        - protocol: TCP
          port: 80
        - protocol: TCP
          port: 443
```

[Ссылка на netpolicy-cache.yml](netpolicy-cache.yml)

- Применил конфигурацию

```bash
vk@vkvm:~/14_3$ kubectl create -f netpolicy-default.yml -f netpolicy-back.yml -f netpolicy-cache.yml
networkpolicy.networking.k8s.io/default-deny-ingress created
networkpolicy.networking.k8s.io/frontend-to-backend-policy created
networkpolicy.networking.k8s.io/backend-to-cache-policy created
```

- Проверил сетевой доступ между сервисами

```bash
vk@vkvm:~/14_3$ kubectl exec -it service/service-front -n app -- curl --silent -i service-backend.app.svc.cluster.local | grep Server
Server: nginx/1.20.2
vk@vkvm:~/14_3$ kubectl exec -it service/service-backend -n app -- curl --silent -i service-cache.app.svc.cluster.local | grep Server
Server: nginx/1.20.2
vk@vkvm:~/14_3$ kubectl exec -it service/service-front -n app -- curl --silent -i service-cache.app.svc.cluster.local | grep Server
command terminated with exit code 28
```

Убедились, что **полный сетевой доступ пропал**.
    - Правило по умолчанию запрещает подключения, не разрешённые специально.
    - Среди разрешённых подключений нет подключения `frontend` -> `cache`, отсюда ошибка соединения. Вывод - **сетевые политики работают**.
