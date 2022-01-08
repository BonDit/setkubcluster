# Параметры кластера в файлах kubespray

Перейдите в директорию inventory.

Скопируйте пример конфигурации кластера sample в cluster.

Перейдите в директорию cluster и отредактируйте файл inventory.ini. [Пример файла](cluster/inventory.ini).

Остальные параметры будем изменять в файлах в директории group_vars.

## group_vars/k8s_cluster/k8s-cluster.yml

* **kube_version**: v1.22.4 - выбираем версию кластера кубернетес.
* **kube_network_plugin**: calico - выбираем драйвер сети кластера.
* **kube_proxy_mode**: ipvs - режим iptables не ставим.
* **kube_proxy_strict_arp**: true - нужен для работы MetallB(который нужен для ingress LoadBalancer)
* **cluster_name**: cluster.local - имя кластера. Используется в качестве корневого домена во внутреннем DNS сервере.
* **enable_nodelocaldns**: true - включаем кеширующие DNS сервера на каждой ноде кластера.
* **container_manager**: _containerd_ - определяем систему контейнеризации.
* **k8s_image_pull_policy**: IfNotPresent - политика загрузки образов системных контейнеров кластера.
* **kubeconfig_localhost**: true - : создаст контекст для управления кластером в inventory/mycluster/artifacts/admin.conf
* **auto_renew_certificates**: true - Автоматический перевыпуск сертификатов для кубернетес control plane. 
  Без необходимости увеличения версии кластера.
  



