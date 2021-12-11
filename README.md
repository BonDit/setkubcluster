# Установка кластера

На примере CentOS 7 и Debian 10.

## 00 - подготовительные действия

Все действия производятся с упраляющего сервера gw.home(192.168.1.10)

Ставим ansible

    apt install python-pip python-setuptools -y
    python -m pip install ansible

Сгенерировать, если его ещё нет, ssh ключ:

    ssh-keygen

Установить ssh ключ на машины кластера.

    ssh-copy-id kube-1
    ssh-copy-id kube-2
    ssh-copy-id kube-3

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
    cp -r sample cluster
    cd cluster
    cp -f ../../../setkubcluster/kubespray/cluster/inventory.ini .
    vim inventory.ini

Параметры смотрим [тут](kubespray/README.md).

Переходим в корень kubespray и запускаем установку кластера. Для этого в системе должны быть установлены интерпретатор 
python и pip.

    cd ~/kubespray
    pip install -r requirements.txt
    ansible-playbook -i inventory/cluster/inventory.ini cluster.yml

После установки на ноде kube-1 можно посмотреть состояние кластера.

    kubectl get nodes -o wide
    kubectl get pods --all-namespaces

Для просмотра контейнеров на ноде, вместо docker следует использовать crictl

    crictl ps

    crictl pods

## 02 - Настройка NFS сервера для Persistent Volumes

Заходим на управляющий сервер

    ssh gw.home

Создаем каталог nfs

    mkdir -p /mnt/nfs

Даем права на каталог

    chmod -R 777 /mnt/nfs
    Для Debian chown -R nobody:nogroup /mnt/nfs
    Для Centos chown -R nfsnobody:nfsnobody /mnt/nfs

Правим файл экспорта

    echo '/mnt/nfs *(rw,sync,no_root_squash)' >> /etc/exports

Применяем конфигурацию

    exportfs -ra

Запускаем сервис nfs

    для Centos:
    systemctl restart rpcbind && systemctl enable rpcbind
    systemctl restart nfs && systemctl enable nfs
    для Debian:
    apt install -y nfs-kernel-server
    systemctl enable nfs-kernel-server.service
    systemctl start nfs-kernel-server.service

Запускаем nfs на нодах для Centos

    ssh kube-1
    systemctl start nfs && systemctl enable nfs
 
    ssh kube-2
    systemctl start nfs && systemctl enable nfs

    ssh kube-3
    systemctl start nfs && systemctl enable nfs
    
Монтируем nfs папку так же на всех нодах

    mkdir -p /mnt/nfs
    mount -t nfs 192.168.1.10:/mnt/nfs /mnt/nfs
    echo '192.168.1.10:/mnt/nfs /mnt/nfs nfs user,rw,auto 0 0' >> /etc/fstab

# 03 - Ставим helm и nfs-provisioner

    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh

    helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
    helm repo update
    helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=192.168.1.10 --set nfs.path=/mnt/nfs

# 04 - Запускаем statefulset wordpress c базой mysql

Делаем все так же с мастер ноды, зайдем в $HOME, скопируем папку с нашего сервера
из которой будем деплоить wordpress, и зайдем в нее

    ssh kube-master-1.home
    scp -r bondit@gw.home:/home/bondit/setkubcluster/site .
    cd site

Скопировали деплойменты для wordpress и mysql и секрет с паролем от базы

Не забываем изменить ***server: 192.168.1.80*** в деплойментах на ip своей мастер ноды или на тот ip где вы подняли nfs сервер

vim kustomization.yaml [тут](site/kustomization.yaml)

vim mysql-deployment.yaml [тут](site/mysql-deployment.yaml)

vim wordpress-deployment.yaml [тут](site/wordpress-deployment.yaml)

Ну и запускаем

    kubectl apply -k ./

Проверяем все

    kubectl get secrets
    kubectl get pv
    kubectl get pvc
    kubectl get pods
    kubectl get svc
    kubectl get pods -o wide
    kubectl get pods --all-namespaces
    kubectl get pod -A
    kubectl describe node kube-master-1.home
    kubectl describe node kube-node-1.home
    kubectl describe node kube-node-2.home

Удаляем весь деплой

    kubectl delete -k ./
