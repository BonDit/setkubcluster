# Установка кластера

На примере CentOS 7.9 и Debian 10 и 11.

## 00 - подготовительные действия

Все действия производятся с упраляющего сервера gw.home(192.168.1.10)

Ставим ansible

    Для Debian
    apt install python-pip python-setuptools -y
    python -m pip install ansible
    Для Centos
    yum install python3-pip python3-setuptools -y
    pip3 install --upgrade pip
    pip3 install ansible

Сгенерировать, если его ещё нет, ssh ключ:

    ssh-keygen

Установить ssh ключ на машины кластера.

    ssh-copy-id 192.168.1.80
    ssh-copy-id 192.168.1.85
    ssh-copy-id 192.168.1.86

## 01 - Kubespray

    cd ~/
    git clone https://github.com/kubernetes-sigs/kubespray.git
    cd kubespray
    git checkout tags/v2.18.0
    cp -rfp inventory/sample inventory/cluster
    declare -a IPS=(192.168.1.80 192.168.1.85 192.168.1.86)
    CONFIG_FILE=inventory/cluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

Параметры смотрим [тут](kubespray/README.md).

Переходим в корень kubespray и запускаем установку кластера. Для этого в системе должны быть установлены интерпретатор 
python и pip.

    cd ~/kubespray
    pip install -r requirements.txt или pip3 install -r requirements.txt
    ansible-playbook -i inventory/cluster/hosts.yaml --become --become-user=root cluster.yml
    
Затем скопируем context для управления кластером 

    cp -f inventory/mycluster/artifacts/admin.conf /root/.kube/config

Так же можно скопировать конфиг(он же context) (cat .kube/config) и перенести на gw.home(192.168.1.10) по тому же пути,
не забыв заменить server: https://127.0.0.1:6443 на ip мастер ноды(192.168.1.80). И мы сможем
упроавлять кластером с gw.home(192.168.1.10)

После установки можно посмотреть состояние кластера.

    kubectl get nodes -o wide
    kubectl get pods --all-namespaces

Для просмотра контейнеров на ноде, вместо docker следует использовать crictl

    crictl ps

    crictl pods

## 02 - Настройка NFS сервера для Persistent Volumes

Заходим на управляющий сервер

    ssh 192.168.1.80

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

Запускаем сервис nfs на управляющем сервере

    для Centos:
    systemctl restart rpcbind && systemctl enable rpcbind
    systemctl restart nfs && systemctl enable nfs
    для Debian:
    apt install -y nfs-kernel-server
    systemctl enable nfs-kernel-server.service
    systemctl start nfs-kernel-server.service
    
Монтируем nfs папку так же на всех нодах

    Для дебиан ставим клиент apt install -y nfs-common
    mkdir -p /mnt/nfs
    mount -t nfs 192.168.1.10:/mnt/nfs /mnt/nfs
    echo '192.168.1.10:/mnt/nfs /mnt/nfs nfs user,rw,auto 0 0' >> /etc/fstab

# 03 - Ставим helm и nfs-provisioner через helm

Делаем все так же с мастер ноды или с управляющего сервера тут как удобнее

    ssh kube-1.home
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh

    helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
    helm repo update
    helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=192.168.1.10 --set nfs.path=/mnt/nfs

# 04 - Запускаем wordpress c базой mysql через helm

Делаем все так же с мастер ноды

    ssh kube-1.home
    helm repo add wordpress-bitnami https://charts.bitnami.com/bitnami
    helm repo update
    helm search repo wordpress | grep bitnami/wordpress
    helm install wordpress wordpress-bitnami/wordpress \
        --set global.storageClass=nfs-client \
        --set wordpressUsername=admin \
        --set wordpressPassword=password \
        --set mariadb.auth.rootPassword=secretpassword
