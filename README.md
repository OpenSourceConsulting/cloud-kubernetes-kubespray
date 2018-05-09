# 클라우드 솔루션 - Kubernetes 설치

kubespray를 이용하여 Kubernetes 클러스터를 구성합니다.

## Docker 이미지 생성

    # kubespray 다운로드
    curl -o ~/kubespray.tar.gz -fSL https://github.com/kubernetes-incubator/kubespray/archive/v2.5.0.tar.gz
    tar -xvzf ~/kubespray.tar.gz
    rm -f ~/kubespray.tar.gz

    # kubespray 설정 변경
    mv kubespray* cloud-kubernetes-kubespray
    cd cloud-kubernetes-kubespray
    cp -rfp inventory/sample inventory/cloud
    mkdir vars
    touch vars/override.yaml
    echo 'ansible-playbook -i /kubespray/inventory/cloud/hosts.ini /kubespray/cluster.yml -b \
     --extra-vars "@/kubespray/vars/override.yaml" \
     --private-key=/kubespray/keypair/id_rsa --user=centos' > scripts/install.sh
    sed -i -e '/^---/a\
    \' roles/kubernetes-apps/cluster_roles/tasks/main.yml
    sed -i -e '/^---/a\    seconds: 60' roles/kubernetes-apps/cluster_roles/tasks/main.yml
    sed -i -e '/^---/a\  pause:' roles/kubernetes-apps/cluster_roles/tasks/main.yml
    sed -i -e '/^---/a\- name: Kubernetes Apps | Wait for kube-apiserver' roles/kubernetes-apps/cluster_roles/tasks/main.yml

    # 이미지 생성
    docker build -t registry.opensourcelab.co.kr/cloud/cloud-kubernetes-kubespray:0.1 .
    docker push registry.opensourcelab.co.kr/cloud/cloud-kubernetes-kubespray:0.1
    docker rmi -f $(docker images | grep "cloud-kubernetes-kubespray" | awk '{print $3}')

## Kubernetes 설치 준비

    # key 생성
    cloud-kubernetes-setup에서 생성한 키 사용

    # 설정 file 생성
    mkdir -p ~/kubernetes/kubespray
    cat << EOF > ~/kubernetes/kubespray/hosts.ini
    [all]
    
    [kube-master]
    
    [kube-node]
    
    [etcd]
    
    [kube-ingress]
    
    [k8s-cluster:children]
    kube-master
    kube-node
    EOF

## Kubernetes 설치 (On AWS)

    cat << EOF > ~/kubernetes/kubespray/override.yaml
    bootstrap_os: centos
    bin_dir: /usr/local/sbin
    etcd_multiaccess: true
    apiserver_loadbalancer_domain_name: $(grep lb_domain_name ~/kubernetes/terraform.out | awk '{print $3}')
    loadbalancer_apiserver:
      port: 6443
    cloud_provider: aws
    
    kube_users:
      admin:
        pass: "password"
        role: admin
        groups:
          - system:masters
      developer:
        pass: "password"
        role: developer
    kube_basic_auth: true
    kube_network_plugin: flannel
    kube_service_addresses: 10.96.0.0/12
    kube_pods_subnet: 10.244.0.0/16
    kube_apiserver_insecure_port: 0 # (http)
    helm_enabled: true
    registry_enabled: true
    registry_namespace: kube-system
    registry_storage_class: default-storage
    registry_disk_size: 100Gi
    ingress_nginx_enabled: true
    ingress_nginx_host_network: true
    ingress_nginx_namespace: ingress-nginx
    ingress_nginx_insecure_port: 80
    ingress_nginx_secure_port: 443
    ingress_nginx_configmap:
      server-tokens: "False"
    
    override_system_hostname: false
    EOF
    
    # Kubernetes 설치
    docker run -it --rm \
     -v ~/kubernetes/keypair/id_rsa:/kubespray/keypair/id_rsa \
     -v ~/kubernetes/kubespray/hosts.ini:/kubespray/inventory/cloud/hosts.ini \
     -v ~/kubernetes/kubespray/override.yaml:/kubespray/vars/override.yaml \
     registry.opensourcelab.co.kr/cloud/cloud-kubernetes-kubespray:0.1 \
     sh -x scripts/install.sh

## Kubernetes 설치 (On OpenStack)

    cat << EOF > ~/kubernetes/kubespray/override.yaml
    bootstrap_os: centos
    bin_dir: /usr/local/sbin
    etcd_multiaccess: true
    apiserver_loadbalancer_domain_name: $(grep lb_domain_name ~/kubernetes/terraform.out | awk '{print $3}')
    loadbalancer_apiserver:
      address: $(grep lb_address ~/kubernetes/terraform.out | awk '{print $3}')
      port: 6443
    cloud_provider: openstack
    
    kube_users:
      admin:
        pass: "password"
        role: admin
        groups:
          - system:masters
      developer:
        pass: "password"
        role: developer
    kube_basic_auth: true
    kube_network_plugin: flannel
    kube_service_addresses: 10.96.0.0/12
    kube_pods_subnet: 10.244.0.0/16
    kube_apiserver_insecure_port: 0 # (http)
    helm_enabled: true
    registry_enabled: true
    registry_namespace: kube-system
    registry_storage_class: default-storage
    registry_disk_size: 100Gi
    ingress_nginx_enabled: true
    ingress_nginx_host_network: true
    ingress_nginx_namespace: ingress-nginx
    ingress_nginx_insecure_port: 80
    ingress_nginx_secure_port: 443
    ingress_nginx_configmap:
      server-tokens: "False"
    
    override_system_hostname: false
    EOF
    
    # Kubernetes 설치
    docker run -it --rm \
     -e OS_AUTH_URL=http://openstack:5000/v3 \
     -e OS_REGION_NAME=RegionOne \
     -e OS_USER_DOMAIN_NAME=Default \
     -e OS_TENANT_ID=tenant_id \
     -e OS_PROJECT_NAME=project_name \
     -e OS_USERNAME=username \
     -e OS_PASSWORD=password \
     -v ~/kubernetes/keypair/id_rsa:/kubespray/keypair/id_rsa \
     -v ~/kubernetes/kubespray/hosts.ini:/kubespray/inventory/cloud/hosts.ini \
     -v ~/kubernetes/kubespray/override.yaml:/kubespray/vars/override.yaml \
     registry.opensourcelab.co.kr/cloud/cloud-kubernetes-kubespray:0.1 \
     sh -x scripts/install.sh

## Kubernetes 설치 (On KVM)

    cat << EOF > ~/kubernetes/kubespray/override.yaml
    bootstrap_os: centos
    bin_dir: /usr/local/sbin
    etcd_multiaccess: true
    apiserver_loadbalancer_domain_name: $(grep lb_domain_name ~/kubernetes/terraform.out | awk '{print $3}')
    loadbalancer_apiserver:
      address: $(grep lb_address ~/kubernetes/terraform.out | awk '{print $3}')
      port: 6443
    
    kube_users:
      admin:
        pass: "password"
        role: admin
        groups:
          - system:masters
      developer:
        pass: "password"
        role: developer
    kube_basic_auth: true
    kube_network_plugin: flannel
    kube_service_addresses: 10.96.0.0/12
    kube_pods_subnet: 10.244.0.0/16
    kube_apiserver_insecure_port: 0 # (http)
    helm_enabled: true
    registry_enabled: true
    registry_namespace: kube-system
    registry_storage_class: default-storage
    registry_disk_size: 100Gi
    ingress_nginx_enabled: true
    ingress_nginx_host_network: true
    ingress_nginx_namespace: ingress-nginx
    ingress_nginx_insecure_port: 80
    ingress_nginx_secure_port: 443
    ingress_nginx_configmap:
      server-tokens: "False"
    
    override_system_hostname: false
    EOF
    
    # Kubernetes 설치
    docker run -it --rm \
     -v ~/kubernetes/keypair/id_rsa:/kubespray/keypair/id_rsa \
     -v ~/kubernetes/kubespray/hosts.ini:/kubespray/inventory/cloud/hosts.ini \
     -v ~/kubernetes/kubespray/override.yaml:/kubespray/vars/override.yaml \
     registry.opensourcelab.co.kr/cloud/cloud-kubernetes-kubespray:0.1 \
     sh -x scripts/install.sh
