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

    git clone https://github.com/BonDit/setkubcluster.git

Переходим в директорию setkubcluster и правим инвентори под себя

    cd setkubcluster
    vim hosts.txt

Проверяем подключение ansible к хостам:

    ansible-playbook ping.yaml

Если ping не проходит, ищем ошибки и исправляем.

Приводим настройки серверов кластера к одному виду:

    ansible-playbook prepare-hosts.yaml

## Kubespray

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
