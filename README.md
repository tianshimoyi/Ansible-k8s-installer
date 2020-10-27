# ansible脚本部署

## 1. 实现私有仓库的部署，etcd数据库使用tls的部署

### 1.1 准备工作-部署节点(准备资源)

1. `部署节点`需要自行预先安装以下软件，如果必要的话可以自己预先准备安装包或者是`虚拟机的镜像`（当然部署节点可以是你的本机，只要网络通即可）
  - ansible
  - curl

2. 准备私有仓库部署所需的资源
  - 创建文件夹 /tmp/caas4-component-images 用于存放镜像

    ```console
    $ mkdir  /tmp/caas4-component-images
    ```
  
  - 将镜像拷贝到 /tmp/caas4-component-images

  - 将 rpms 文件夹拷贝到 /tmp 下

  - 将 registry 镜像打包并放到 /tmp 下

3. 在文档为了方便会使用一下节点设置

    ```console
      在/tmp/hosts.txt下添加如下配置 ip  hostname
      10.0.0.101  etcd01
      10.0.0.102  etcd02
      10.0.0.103  etcd03
    ```

4. 在部署节点`准备`cfssl和cfssljson`二进制包 和etcd文件

    ```console
    $ tar -xzvf cfss.tar.gz    #将cfssl 和cfssljson工具压缩包进行解压
    $ cp cfssl  cfssljson /tmp/  #将cfssl 和cfssljson拷贝到 /tmp/文件下
    $ tar -xzvf etcd-v3.4.10-linux-amd64.tar.gz  将etcd压缩包进行解压
    $ cd etcd-v3.4.10-linux-amd64
    $ cp etcd etcdctl /tmp/etcd_tls_resources/  #将etcd etcdctl拷贝到/tmp/etcd_tls_resources/文件夹下
    ```

4. 确保三台主机的`2379，2380`端口都没被占用

### 1.2 准备工作-部署节点(修改配置文件)

1. 修改`inventory_ansible.example`配置文件

    ```console
      打开`inventory_ansible.example`文件修改配置
      [all:vars]
      ##部署私有仓库
      #部署节点镜像的父文件夹路径
      base_path="/tmp/caas4-component-images"
      #镜像名
      imagesname="caas4-component-images-20200812.tgz"
      #镜像路径
      image_path="{{ base_path+'/'+imagesname}}"
      #部署目标节点的镜像存放路径
      target_location="/root/private_warehouse/"
      #部署节点rpm包路径
      rpm_path=/tmp/rpms/
      #部署目标节点rpm包路径
      rpm_target_location=/root/private_warehouse/
      #部署节点registry镜像包路径
      registry_path=/tmp/registry.tar
      #目标节点registry镜像包路径
      target_registry_location=/root/private_warehouse/

      ##部署3节点的etcd数据库使用tls
      #basic section starts
      project_template_path="/etc/etsi-api-deployment"

      #部署节点etcd存放路径
      etcd_path="/tmp/etcd_tls_resources/etcd"

      #部署节点etcdctl存放路径
      etcdctl_path="/tmp/etcd_tls_resources/etcdctl"

      #部署目标节点的证书生成路径
      localhost_tls_dir="/etc/etsi-api-tls-dir"

      #部署节点cfssl存放路径
      cfssl_path="/tmp/cfssl"

      #部署节点cfssljson存放路径
      cfssljson_path="/tmp/cfssljson"

      #tls sign section starts
      tls_c="CHINA"
      tls_l="Shanghai"
      tls_o="99cloud"
      tls_st="Shanghai"
      tls_ou="CAAS"

      #部署目标节点的详细信息
      ##注意，若是减少或增加节点，请在/roles/etcd_tls/tasks/tls_localhost_preparetion.yaml 和 /roles/etcd_tls/templates/etcd-systemd-template.j2进行相应的修改
      etcd_node1_ip="10.0.0.101"  #第一台部署节点的ip地址
      etcd_node1_hostname="etcd01" #第一台部署节点的hostname
      etcd_node1_domain=""
      etcd_node1_fqdn="{{ etcd_node1_hostname + '.' + etcd_node1_domain }}"
      etcd_node1_address_set="{{ etcd_node1_ip + ',' + etcd_node1_hostname + '.' + etcd_node1_domain + ',' + etcd_node1_hostname }}"
      etcd_node2_ip="10.0.0.102"   #第二台部署节点的ip地址
      etcd_node2_hostname="etcd02"  #第二台部署节点的ip地址
      etcd_node2_domain=""
      etcd_node2_fqdn="{{ etcd_node2_hostname + '.' + etcd_node2_domain }}"
      etcd_node2_address_set="{{ etcd_node2_ip + ',' + etcd_node2_hostname + '.' + etcd_node2_domain + ',' + etcd_node2_hostname }}"
      etcd_node3_ip="10.0.0.103"    #第三台部署节点的ip地址
      etcd_node3_hostname="etcd03"   #第三台部署节点的ip地址
      etcd_node3_domain=""
      etcd_node3_fqdn="{{ etcd_node3_hostname + '.' + etcd_node3_domain }}"
      etcd_node3_address_set="{{ etcd_node3_ip + ',' + etcd_node3_hostname + '.' + etcd_node3_domain + ',' + etcd_node3_hostname }}"
      #配置/etc/hosts信息的hosts文件地址
      hosts_path="/tmp/hosts.txt"
      #部署私有仓库目标节点hostname
      [private_warehouse]
      etcd01
      etcd02
      etcd03
    #部署etcd数据库使用tls目标节点hostname
      [etcd]
      etcd01
      etcd02
      etcd03

      [tls]     #在本机上进行签名证书，再分发到不同的节点
      localhost    ansible_connection=local

    ```

## 2. 实现将server,client的二进制文件复制到目标节点，并进行配置。而后将server和client二进制文件分别包装为systemd

### 2.1 准备工作-部署节点

1. `部署节点`需要自行预先安装以下软件，如果必要的话可以自己预先准备安装包或者是`虚拟机的镜像`（当然部署节点可以是你的本机，只要网络通即可）
    - ansible
    
2. 创建/tmp/server_client_resources/x86 和/tmp/server_client_resources/arm文件夹用于存放x86或arm环境下的server和client二进制文件

    ```console
    $ mkdir -p {/tmp/server_client_resources/x86,/tmp/server_client_resources/arm}
    $ tar xzvf server_client.tar.gz #解压包含server和client的二进制文件的压缩包
    $ mv server client /tmp/server_client_resources/x86  #x86环境下
    $ mv client server /tmp/server_client_resources/arm  #arm环境下
    ```

3. 修改 `inventory_ansible.example` 配置文件

    ```console
    #配置sever client二进制文件变量
    #offline   # true | false 目前只实现了 online
    offline="false"
    #server二进制文件地址
    server_path="/tmp/server_client_resources/x86/server"   #x86
    #server_path="/tmp/server_client_resources/arm/server"  #arm
    ##client二进制文件地址
    #x86
    client_path="/tmp/server_client_resources/x86/client"
    #arm
    #client_path="/tmp/server_client_resources/arm/client"

    signal_port=9090
    message_queue_port=9889

    #配置server client主机
    #server主机的hostname和ip
    [server]
    localhost    ansible_connection=local ip=172.20.149.79

    #clinet主机的hostname
    [client]
    etcd01
    etcd02
    etcd03
    ```

## 3. 运行

### 3.1 运行

1. 运行 cd到工作目录下面然后执行ansible脚本 [path]为你的工作目录

    ```console
    $ cd [path]
    $ ansible-playbook -i inventory_ansible.example ansible.yaml
    ```

### 3.2 验证结果

1. 验证部署`私有仓库`

   ```consloe
   在任意节点上运行
   $ curl http://127.0.0.1:5000/v2/_catalog
   出现以下结果
   {"repositories":["caas4/alertmanager","caas4/busybox","caas4/cephcsi","caas4/cephfs-provisioner","caas4/cert-manager-cainjector","caas4/cert-manager-controller","caas4/cert-manager-webhook","caas4/cinder-csi-plugin","caas4/citadel","caas4/configmap-reload","caas4/console","caas4/controller","caas4/crunchy-postgres-ha","caas4/csi-attacher","caas4/csi-node-driver-registrar","caas4/csi-provisioner","caas4/csi-resizer","caas4/csi-snapshotter","caas4/curl","caas4/elasticsearch","caas4/es-index-rotator","caas4/etcd","caas4/examples-bookinfo-ratings-v1","caas4/examples-bookinfo-reviews-v1","caas4/examples-bookinfo-reviews-v2","caas4/examples-bookinfo-reviews-v3","caas4/fluentd-elasticsearch","caas4/galley","caas4/grafana","caas4/k8s-sidecar","caas4/kiali","caas4/kibana-oss","caas4/kubectl","caas4/log-pilot","caas4/middle-platform-api","caas4/middle-platform-api-gateway","caas4/middle-platform-auth","caas4/node-exporter","caas4/openldap","caas4/pgo-apiserver","caas4/pgo-backrest-repo","caas4/pgo-client","caas4/pgo-deployer","caas4/pgo-event","caas4/phpldapadmin","caas4/pilot","caas4/postgres-operator","caas4/prometheus","caas4/proxyv2","caas4/sidecar_injector","caas4/speaker","caas4/tiller","calico/cni","calico/kube-controllers","calico/pod2daemon-flexvol","istio/proxyv2","jettech/kube-webhook-certgen","k8s.gcr.io/kube-apiserver","k8s.gcr.io/kube-controller-manager","k8s.gcr.io/kube-proxy","k8s.gcr.io/kube-scheduler","k8s.gcr.io/pause","quay.io/kubernetes-ingress-controller/nginx-ingress-controller","registry","us.gcr.io/k8s-artifacts-prod/ingress-nginx/controller"]}
   ```

2. 验证3节点的`etcd数据库使用tls`

    ```console
     在部署节点执行以下命令
     $ ansible all -a 'etcdctl --cacert=/etc/etsi-api-deployment/etcd/etcd-ca.crt --cert=/etc/etsi-api-deployment/etcd//server.crt --key=/etc/etsi-api-deployment/etcd/server.key --endpoints="https://etcd01:2379,https://etcd02:2379,https://etcd03:2379" endpoint health'

     结果：
     etcd02 | CHANGED | rc=0 >>
     https://etcd03:2379 is healthy: successfully committed proposal: took = 13.437608ms
     https://etcd02:2379 is healthy: successfully committed proposal: took = 12.286761ms
     https://etcd01:2379 is healthy: successfully committed proposal: took = 13.679098ms
     etcd03 | CHANGED | rc=0 >>
     https://etcd01:2379 is healthy: successfully committed proposal: took = 13.774736ms
     https://etcd03:2379 is healthy: successfully committed proposal: took = 14.866842ms
     https://etcd02:2379 is healthy: successfully committed proposal: took = 14.324932ms
     etcd01 | CHANGED | rc=0 >>
     https://etcd03:2379 is healthy: successfully committed proposal: took = 13.013918ms
     https://etcd01:2379 is healthy: successfully committed proposal: took = 12.778537ms
     https://etcd02:2379 is healthy: successfully committed proposal: took = 13.607148ms

    ```
