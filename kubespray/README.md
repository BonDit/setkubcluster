# Параметры кластера в файлах kubespray

Перейдите в директорию inventory.

Скопируйте пример конфигурации кластера sample в cluster.

Перейдите в директорию cluster и отредактируйте файл inventory.ini. [Пример файла](cluster/inventory.ini).

Остальные параметры будем изменять в файлах в директории group_vars.

## group_vars/k8s_cluster/k8s-cluster.yml

* **kube_version**: v1.22.4 - выбираем версию кластера кубернетес.
* **kube_network_plugin**: calico - выбираем драйвер сети кластера.
* **kube_service_addresses**: 10.233.0.0/18 - диапазон адресов для сервисов кластера.
* **kube_pods_subnet**: 10.233.64.0/18 - диапазон адресов для подов кластера.
* **kube_network_node_prefix**: 24 - размер подсети подов на ноде кластера.
* **kube_apiserver_port**: 6443 - Порт API сервера кластера.
* **kube_proxy_mode**: ipvs - режим iptables не ставим.
* **kube_proxy_strict_arp**: true - нужен для работы MetallB(который нужен для ingress LoadBalancer)
* **cluster_name**: cluster.local - имя кластера. Используется в качестве корневого домена во внутреннем DNS сервере.
* **enable_nodelocaldns**: true - включаем кеширующие DNS сервера на каждой ноде кластера.
* **nodelocaldns_ip**: 169.254.25.10 - IP адрес кешируюшего DNS сервера на ноде.
* **container_manager**: _containerd_ - определяем систему контейнеризации.
* **k8s_image_pull_policy**: IfNotPresent - политика загрузки образов системных контейнеров кластера.
* **system_memory_reserved**: 512Mi - зарезервированная за Linux системой (приложениями) память.
* **system_cpu_reserved**: 500m - зарезервированное за Linux системой (приложениями) время процессора.
* **auto_renew_certificates**: true - Автоматический перевыпуск сертификатов для кубернетес control plane. 
  Без необходимости увеличения версии кластера.
  
## group_vars/k8s_cluster/k8s-net-calico.yml

* **calico_ipip_mode**: 'Always' - Определяем когда использовать IP in IP режим.

## group_vars/all/all.yml

* **etcd_kubeadm_enabled**: true - разрешаем установку и etcd средствами kubeadm.
* **loadbalancer_apiserver_type**: nginx - значение по умолчанию. Доступ к k8s API через loopback интерфейс ноды кластера.


