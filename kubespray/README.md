# Параметры кластера в файлах kubespray

## group_vars/k8s_cluster/k8s-cluster.yml

* **kube_version**: v1.22.5 - выбираем версию кластера кубернетес.
* **kube_network_plugin**: calico - выбираем драйвер сети кластера.
* **kube_proxy_mode**: ipvs - режим iptables не ставим.
* **kube_proxy_strict_arp**: true - нужен для работы MetallB(который нужен для ingress LoadBalancer)
* **cluster_name**: cluster.local - имя кластера. Используется в качестве корневого домена во внутреннем DNS сервере.
* **enable_nodelocaldns**: true - включаем кеширующие DNS сервера на каждой ноде кластера.
* **container_manager**: containerd - определяем систему контейнеризации.
* **k8s_image_pull_policy**: IfNotPresent - политика загрузки образов системных контейнеров кластера.
* **kubeconfig_localhost**: true - : создаст контекст для управления кластером в inventory/mycluster/artifacts/admin.conf
* **auto_renew_certificates**: true - Автоматический перевыпуск сертификатов для кубернетес control plane. 
  Без необходимости увеличения версии кластера.
  
## group_vars/etcd.yml

* **etcd_memory_limit**: "0" - не ограничиваем etcd в оперативке.

## group_vars/k8s_cluster/addons.yml

Раскоментираем все что связано с ingress

* **ingress_nginx_enabled**: true
* **ingress_nginx_host_network**: false
* **ingress_publish_status_address**: ""
* **ingress_nginx_nodeselector**:
*  **kubernetes.io/os**: "linux"
* **ingress_nginx_tolerations**:
*  **- key**: "node-role.kubernetes.io/master"
*    **operator**: "Equal"
*    **value**: ""
*    **effect**: "NoSchedule"
*  **- key**: "node-role.kubernetes.io/control-plane"
*    **operator**: "Equal"
*    **value**: ""
*    **effect**: "NoSchedule"
* **ingress_nginx_namespace**: "ingress-nginx"
* **ingress_nginx_insecure_port**: 80
* **ingress_nginx_secure_port**: 443
* **ingress_nginx_configmap**:
*  **map-hash-bucket-size**: "128"
*  **ssl-protocols**: "TLSv1.2 TLSv1.3"
* **ingress_nginx_configmap_tcp_services**:
*  **9000**: "default/example-go:8080"
* **ingress_nginx_configmap_udp_services**:
*  **53**: "kube-system/coredns:53"
* **ingress_nginx_extra_args**:
*  - --default-ssl-certificate=default/foo-tls
* **ingress_nginx_termination_grace_period_seconds**: 300
* **ingress_nginx_class**: nginx

Включим metallb чтобы он смог выдать ip ингрессу

* **metallb_enabled**: true
* **metallb_speaker_enabled**: true
* **metallb_ip_range**:
  - "192.168.1.90/32"
