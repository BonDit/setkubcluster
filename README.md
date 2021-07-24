# Установка кластера

На примере CentOS 7.

## 00 - подготовительные действия

Сгенерировать, если его ещё нет, ssh ключ:

    ssh-keygen

Установить ssh ключ на машины кластера.

    ssh-copy-id kube-master-1.home
    ssh-copy-id kube-node-1.home
    ssh-copy-id kube-node-2.home

Скачиваем плейбук для подготовки кластера

    cd ~/
    git clone https://github.com/BonDit/setkubcluster.git

Переходим в директорию setkubcluster и правим инвентори под себя

    cd setkubcluster
    vim hosts.txt

Проверяем подключение ansible к хостам:

    ansible-playbook ping.yaml

Если ping не проходит, ищем ошибки и исправляем.

Приводим настройки серверов кластера к одному виду:

    ansible-playbook prepare-hosts.yaml

## 01 - Kubespray

    cd ~/
    git clone https://github.com/kubernetes-sigs/kubespray.git
    cd kubespray/inventory
    cp sample cluster
    cd cluster

Параметры смотрим [тут](kubespray/README.md).

Переходим в корень kubespray и запускаем установку кластера. Для этого в системе должны быть установлены интерпретатор 
python и pip.

    pip install -r requirements.txt
    ansible-playbook -i inventory/cluster/inventory.ini cluster.yml

После установки на ноде master-1 можно посмотреть состояние кластера.

    kubectl get nodes -o wide

Для просмотра контейнеров на ноде, вместо docker следует использовать crictl

    crictl ps

    crictl pods

Установите в bash автодополнение для команд kubectl:

    source <(kubectl completion bash)
    echo "source <(kubectl completion bash)" >> ~/.bashrc

## 02 - Настройка NFS сервера для Persistent Volumes

Заходим на мастер ноду

    ssh kube-master-1.home

Создаем каталоги nfs

    mkdir -p /mnt/nfs
    mkdir -p /mnt/nfs/pv001
    mkdir -p /mnt/nfs/pv002

Даем права на каталоги

    chmod -R 777 /mnt/nfs /mnt/nfs/pv001 /mnt/nfs/pv002

Правим файл экспорта

    vim /etc/exports
    /mnt/nfs *(rw,no_root_squash)
    /mnt/nfs/pv001 *(rw,no_root_squash)
    /mnt/nfs/pv002 *(rw,no_root_squash)

Применяем конфигурацию

    exportfs -r

View вступает в силу

    exportfs

Запускаем rpcbind, сервис nfs

    systemctl restart rpcbind && systemctl enable rpcbind
    systemctl restart nfs && systemctl enable nfs

Запускаем nfs на всех остальных нодах

    systemctl start nfs && systemctl enable nfs

# 03 - Запускаем statefulset wordpress c базой mysql

Делаем все так же с мастер ноды, зайдем в $HOME, создадим папку
из которой будем деплоить wordpress, и зайдем в нее

    cd ~/
    mkdir site
    cd site

Создадщим деплойменты для wordpress и mysql и секрет с паролем от базы

    vim kustomization.yaml
secretGenerator:
- name: mysql-pass
  literals:
  - password=123456
resources:
  - mysql-deployment.yaml
  - wordpress-deployment.yaml

    vim mysql-deployment.yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv001
  labels:
    pv: nfs-pv001
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /mnt/nfs/pv001
    server: 192.168.1.80
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc001
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs
  selector:
    matchLabels:
      pv: nfs-pv001
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: nfs-pv001
          mountPath: /var/lib/mysql
      volumes:
      - name: nfs-pv001
        persistentVolumeClaim:
          claimName: nfs-pvc001

    vim wordpress-deployment.yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv002
  labels:
    pv: nfs-pv002
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /mnt/nfs/pv002
    server: 192.168.1.80
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc002
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs
  selector:
    matchLabels:
      pv: nfs-pv002
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.8-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: nfs-pv002
          mountPath: /var/www/html
      volumes:
      - name: nfs-pv002
        persistentVolumeClaim:
          claimName: nfs-pvc002
