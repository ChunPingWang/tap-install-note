# Tanzu Application Platform 簡易安裝筆記 
### (以 Ubuntu 與 minikube 為例)

### Ubuntu 修改 open file 上限 (須 reboot OS)
```gherkin=
# sudo vim /etc/security/limit.comf，加入下列兩個參數
* soft nofile 65536
* hard nofile 65536
```
### 環境變數
```gherkin=
export TAP_VERSION=1.3.4

export TAP_NAMESPACE=tap-install

export INSTALL_REGISTRY_USERNAME=[自行輸入]

export INSTALL_REGISTRY_PASSWORD=[自行輸入]

export INSTALL_REGISTRY_HOSTNAME=reg.microservice.tw

export INSTALL_REPO=tanzu-application-platform

export TAP_DEV_NAMESPACE=default
```
### minikube 啟動
```gherkin=
minikube start --kubernetes-version='1.22.' --cpus='10' --memory='60g' --insecure-registry="reg.microservice.tw"
```
> 開啟另外一個 terminal 執行，啟動 LoadBalancer
```gherkin=
minikube tunnel
```

#### 鏡像倉庫準備(強烈建議，安裝 TAP 時使用個人的鏡像倉庫，因為 VMware Tanzu Registry 不保證連線品質，會造成不明原因安裝失敗)
---
###須先在private registry建立repo

```gherkin=
tanzu-cluster-essentials # 存放 Tanzu Cluster Essentials 用

tanzu-application-platform # 存放 Tanzu Packages 與 TBS full dependency 用

i7g12 #部署應用時， Tanzu Buidl Service 需要用的 repo

tap-apps #部署應用時，ootb_supply_chain_basic 需要用的 repo

```
###安裝 Taznu cli 與 Carvel 工具
> 從 https://network.pivotal.io/ 下載 VMware Tanzu Application Platform -> tanzu cli
```gherkin=
tar -xvf tanzu-framework-linux-amd64.tar -C $HOME/tanzu

export TANZU_CLI_NO_INIT=true

cd $HOME/tanzu
export VERSION=v0.25.0
sudo install cli/core/$VERSION/tanzu-core-linux_amd64 /usr/local/bin/tanzu

tanzu plugin install --local cli all

tanzu plugin list
```
> 從 https://network.pivotal.io/ 下載 Cluster Essentials for VMware Tanzu -> Carvel cli
```gherkin=
mkdir $HOME/tanzu-cluster-essentials
tar -xvf DOWNLOADED-CLUSTER-ESSENTIALS-BUNDLE -C $HOME/tanzu-cluster-essentials

sudo cp $HOME/tanzu-cluster-essentials/imgpkg /usr/local/bin

sudo cp $HOME/tanzu-cluster-essentials/kapp /usr/local/bin

sudo cp $HOME/tanzu-cluster-essentials/kbld /usr/local/bin

sudo cp $HOME/tanzu-cluster-essentials/ytt /usr/local/bin
```

### 下載 Tanzu Cluster Essentials、TAP packages、TBS full dependency 並上傳至 private registry (一次性工作)
> 登入 registry.tanzu.vmware.com 與 reg.microservice.tw (private registry)
```gherkin=
docker login registry.tanzu.vmware.com

docker login $INSTALL_REGISTRY_HOSTNAME
```
> Tanzu Cluster Essentials
```gherkin=
# 下載
imgpkg copy \
    -b registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:54bf611711923dccd7c7f10603c846782b90644d48f1cb570b43a082d18e23b9 \
    --to-tar cluster-essentials-bundle-1.3.0.tar \
    --include-non-distributable-layers
# 上傳    
imgpkg copy \
    --tar cluster-essentials-bundle-1.3.0.tar \
    --to-repo $INSTALL_REGISTRY_HOSTNAME/tanzu-cluster-essentials/cluster-essentials-bundle \
    --include-non-distributable-layers \
    --registry-ca-cert-path microservice.tw.crt    
```
> TAP packages
```gherkin=
# 下載
imgpkg copy   -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.3.3   --to-tar tap-packages-1.3.3.tar   --include-non-distributable-layers
# 上傳
imgpkg copy   --tar tap-packages-1.3.3.tar   --to-repo $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REPO/tap-packages   --include-non-distributable-layers
```
>TBS full dependency
```gherkin=
tanzu package available list buildservice.tanzu.vmware.com --namespace tap-install
# 下載
imgpkg copy -b $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REPO/full-tbs-deps-package-repo:1.7.4 \
  --to-tar=tbs-full-deps-1.7.4.tar
# 上傳
imgpkg copy --tar tbs-full-deps-1.7.4.tar \
  --to-repo=$INSTALL_REGISTRY_HOSTNAME/$INSTALL_REPO/full-tbs-deps-package-repo

```

### 安裝 Tanzu Cluster Essentials
```gherkin=
cd ~/tanzu-cluster-essentials

kubectl create ns kapp-controller

kubectl create secret generic kapp-controller-config \
  --namespace kapp-controller \
  --from-file caCerts=./microservice.tw.crt

INSTALL_BUNDLE=$INSTALL_REGISTRY_HOSTNAME/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:54bf611711923dccd7c7f10603c846782b90644d48f1cb570b43a082d18e23b9 \
  INSTALL_REGISTRY_HOSTNAME=$INSTALL_REGISTRY_HOSTNAME \
  INSTALL_REGISTRY_USERNAME=$INSTALL_REGISTRY_USERNAME \
  INSTALL_REGISTRY_PASSWORD=$INSTALL_REGISTRY_PASSWORD \
  ./install.sh
```
### 安裝 TAP Packages
> tap-values.yaml 範例
> * 關注點 kp_default_repository: 'reg.microservice.tw/i7g12/build-service'，private registry 須建立repo (eg.此設定檔為 i7g12) 
```gherkin=
shared:
  ingress_domain: "microservice.tw"
  image_registry:
    project_path: "reg.microservice.tw/tap-install"
    username: [自行輸入]
    password: [自行輸入]
  ca_cert_data: |
    -----BEGIN CERTIFICATE-----
    # microservice.tw.crt 內容
    -----END CERTIFICATE-----

ceip_policy_disclosed: true

profile: full

excluded_packages:
- policy.apps.tanzu.vmware.com

buildservice:
  kp_default_repository: 'reg.microservice.tw/i7g12/build-service' 
  kp_default_repository_username: [自行輸入]
  kp_default_repository_password: [自行輸入]
  tanzunet_username: [自行輸入]
  tanzunet_password: [自行輸入]
  enable_automatic_dependency_updates: true
  descriptor_name: full

supply_chain: basic

ootb_supply_chain_basic:
  registry:
    server: "reg.microservice.tw"
    repository: "tap-apps"
  gitops:
    ssh_secret: ""

learningcenter:
  ingressDomain: learning-center.microservices.tw

tap_gui:
  service_type: ClusterIP
  app_config:
    app:
      baseUrl: http://tap-gui.microservice.tw
    catalog:
      locations:
        - type: url
          target: https://github.com/ChunPingWang/tap-gui/blob/main/catalog-info.yaml
    auth:
      environment: development
      providers:
        github:
          development:
            clientId: [clientId] #由 GitHub 上取得
            clientSecret: [clientSecret] #由 GitHub 上取得

metadata_store:
  ns_for_export_app_cert: "*"
  app_service_type: ClusterIP

scanning:
  metadataStore:
    url: ""

contour:
  envoy:
    service:
      type: LoadBalancer

accelerator:
  domain: accelerator.microservices.tw
  ingress:
    include: true
  server:
    service_type: ClusterIP

cnrs:
  domain_name: cnr.microservices.tw
  domain_template: '{{.Name}}-{{.Namespace}}.{{.Domain}}'

grype:
  namespace: default
  targetImagePullSecret: registry-credentials
```
> 開始安裝
```gherkin=
kubectl create namespace $TAP_INSTALL

tanzu secret registry add tap-registry \
  --username $INSTALL_REGISTRY_USERNAME \
  --password $INSTALL_REGISTRY_PASSWORD \
  --server $INSTALL_REGISTRY_HOSTNAME \
  --namespace $TAP_NAMESPACE \
  --export-to-all-namespaces \
  --yes
  
tanzu package repository add tanzu-tap-repository \
  --url $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REPO/tap-packages:$TAP_VERSION \
  --namespace $TAP_NAMESPACE
  
tanzu package repository get tanzu-tap-repository --namespace $TAP_NAMESPACE

tanzu package available list --namespace $TAP_NAMESPACE

#安裝
tanzu package install tap -p tap.tanzu.vmware.com -v $TAP_VERSION \
  --values-file tap-values.yaml \
  --poll-timeout 45m \
  --namespace $TAP_NAMESPACE

#修改
tanzu package installed update tap -p tap.tanzu.vmware.com -v $TAP_VERSION \
  --values-file tap-values.yaml \
  --poll-timeout 45m \
  --namespace $TAP_NAMESPACE

#查詢
tanzu package installed list --namespace $TAP_NAMESPACE
#個別套件安裝細節
kubectl describe packageinstall/buildservice -n $TAP_NAMESPACE

#刪除
tanzu package installed delete tap  --namespace $TAP_NAMESPACE
```

### 安裝 Tanzu Build Service Full Dependency
```gherkin=
tanzu package repository add tbs-full-deps-repository \
  --url $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REPO/full-tbs-deps-package-repo:1.7.4 \
  --namespace tap-install

tanzu package repository get tbs-full-deps-repository --namespace $TAP_NAMESPACE 

tanzu package install full-tbs-deps -p full-tbs-deps.tanzu.vmware.com -v 1.7.4 -n $TAP_NAMESPACE

```

### 啟動 TAP-GUI
```gherkin=
kubectl get svc -A
```
> 輸出範例如下
```
stacks-operator-system      controller-manager-metrics-service                            ClusterIP      10.111.185.42    <none>          8443/TCP                                  44m
tanzu-system-ingress        contour                                                       ClusterIP      10.110.95.244    <none>          8001/TCP                                  43m
tanzu-system-ingress        envoy                                                         LoadBalancer   10.97.105.137    10.97.105.137   80:31241/TCP,443:32687/TCP                43m
tap-gui                     server                                                        ClusterIP      10.105.42.249    <none>          7000/TCP                                  38m
tekton-pipelines            tekton-pipelines-controller                                   ClusterIP      10.106.142.108   <none>          9090/TCP,8008/TCP,8080/TCP                44m
```
> 取得 envoy ip，將 "envoy ip     tap-gui.[domain name]" 寫入 /etc/hosts，本例為 tap-gui.microservice.tw 
```gherkin=
sudo vim /etc/hosts
```
> 結果如下所示：
```
10.0.0.44	    reg.microservice.tw
10.97.105.137  	tap-gui.microservice.tw
```
> 開啟 browser 輸入 http://tap-gui.[domain name]，開啟網頁並透過 github 驗證

### 應用部署
> 建立單一使用者的相關參數
```gherkin=
cat <<EOF | kubectl -n default apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: tap-registry
  annotations:
    secretgen.carvel.dev/image-pull-secret: ""
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: e30K
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
secrets:
  - name: registry-credentials
imagePullSecrets:
  - name: registry-credentials
  - name: tap-registry
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default-permit-deliverable
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: deliverable
subjects:
  - kind: ServiceAccount
    name: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default-permit-workload
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: workload
subjects:
  - kind: ServiceAccount
    name: default
EOF
```
> 開始部署
```gherkin=
tanzu secret registry add registry-credentials \
  --server $INSTALL_REGISTRY_HOSTNAME \
  --username $INSTALL_REGISTRY_USERNAME \
  --password $INSTALL_REGISTRY_PASSWORD \
  --namespace $TAP_DEV_NAMESPACE
  

#一般流程
# Create the workload 
tanzu apps workload create tanzu-java-web-app \
  --git-repo https://github.com/sample-accelerators/tanzu-java-web-app \
  --git-branch main \
  --label apps.tanzu.vmware.com/workload-type=web \
  --label app.kubernetes.io/part-of=tanzu-java-web-app \
  --label tanzu.app.live.view=true \
  --label tanzu.app.live.view.application.name=tanzu-java-web-app \
  --annotation autoscaling.knative.dev/minScale=1 \
  --namespace $TAP_DEV_NAMESPACE \
  --yes

```
> 取得 workload log
```gherkin=
tanzu apps workload tail tanzu-java-web-app --timestamp --since 1h
```
> 取得 workload 資訊
```gherkin=
tanzu apps workload get tanzu-java-web-app
```
> 刪除 workload
```gherkin=
# 刪除應用 
tanzu apps workload delete tanzu-java-web-app

```
