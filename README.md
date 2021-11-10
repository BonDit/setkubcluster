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
    cp -r sample cluster
    cd cluster
    cp -f ../../../setkubcluster/kubespray/cluster/inventory.ini .

Параметры смотрим [тут](kubespray/README.md).

Переходим в корень kubespray и запускаем установку кластера. Для этого в системе должны быть установлены интерпретатор 
python и pip.

    cd ~/kubespray
    pip install -r requirements.txt
    ansible-playbook -i inventory/cluster/inventory.ini cluster.yml

После установки на ноде master-1 можно посмотреть состояние кластера.

    kubectl get nodes -o wide
    kubectl get pods --all-namespaces

Для просмотра контейнеров на ноде, вместо docker следует использовать crictl

    crictl ps

    crictl pods

## 02 - Настройка NFS сервера для Persistent Volumes

Заходим на мастер ноду

    ssh kube-master-1.home

Создаем каталоги nfs

    mkdir -p /mnt/nfs/pv001
    mkdir -p /mnt/nfs/pv002

Даем права на каталоги

    chmod -R 777 /mnt/nfs
    chown -R nobody:nogroup /mnt/nfs

Правим файл экспорта

    vim /etc/exports
    /mnt/nfs *(rw,sync,no_root_squash)
    /mnt/nfs/pv001 *(rw,sync,no_root_squash)
    /mnt/nfs/pv002 *(rw,sync,no_root_squash)

Применяем конфигурацию

    exportfs -r

View вступает в силу

    exportfs

Запускаем rpcbind, сервис nfs

    systemctl restart rpcbind && systemctl enable rpcbind
    systemctl restart nfs && systemctl enable nfs

Запускаем nfs на нодах

    ssh kube-node-1.home
    systemctl start nfs && systemctl enable nfs

    ssh kube-node-2.home
    systemctl start nfs && systemctl enable nfs

# 03 - Запускаем statefulset wordpress c базой mysql

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
