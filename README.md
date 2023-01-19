# tap-install-note (以 Ubuntu 與 minikube 為例)

### Ubuntu 修改 open file 上限
```gherkin=
# sudo vim /etc/security/limit.comf，加入下列兩個參數
* soft nofile 65536
* hard nofile 65536
```
### minikube 啟動
```gherkin=

minikube start --kubernetes-version='1.22.8' --cpus='10' --memory='60g' --insecure-registry="reg.microservice.tw"

```

#### 鏡像倉庫準備(強烈建議，安裝 TAP 時使用個人的鏡像倉庫，因為 VMware Tanzu Registry 不保證連線品質，會造成不明原因安裝失敗)
---
###須先在private registry建立repo

```gherkin=
tanzu-application-platform

tanzu-cluster-essentials
```

### 下載 Tanzu Cluster Essentials、TAP packages、TBS full dependency 並上傳至 private registry (一次性工作)
> 登入 registry.tanzu.vmware.com 與 reg.microservice.tw (private registry)
```gherkin=
docker login registry.tanzu.vmware.com

docker login reg.microservice.tw
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
    --to-repo harbor.example.com/tanzu-cluster-essentials/cluster-essentials-bundle \
    --include-non-distributable-layers \
    --registry-ca-cert-path microservice.tw.crt    
```
> TAP packages
```gherkin=
# 下載
imgpkg copy   -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.3.3   --to-tar tap-packages-1.3.3.tar   --include-non-distributable-layers
# 上傳
imgpkg copy   --tar tap-packages-1.3.3.tar   --to-repo reg.microservice.tw/tanzu-application-platform/tap-packages   --include-non-distributable-layers
```
>TBS full dependency
```gherkin=
tanzu package available list buildservice.tanzu.vmware.com --namespace tap-install
# 下載
imgpkg copy -b registry.tanzu.vmware.com/tanzu-application-platform/full-tbs-deps-package-repo:1.7.4 \
  --to-tar=tbs-full-deps-1.7.4.tar
# 上傳
imgpkg copy --tar tbs-full-deps-1.9.0.tar \
  --to-repo=reg.microservice.tw/tanzu-application-platform/tbs-full-deps

```

### 安裝 Tanzu Cluster Essentials
```gherkin=
cd ~/tanzu-cluster-essentials

kubectl create ns kapp-controller

kubectl create secret generic kapp-controller-config \
  --namespace kapp-controller \
  --from-file caCerts=./microservice.tw.crt

INSTALL_BUNDLE=reg.microservice.tw/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:54bf611711923dccd7c7f10603c846782b90644d48f1cb570b43a082d18e23b9 \
  INSTALL_REGISTRY_HOSTNAME='reg.microservice.tw' \
  INSTALL_REGISTRY_USERNAME=[自行輸入] \
  INSTALL_REGISTRY_PASSWORD=[自行輸入] \
  ./install.sh
```
### 安裝 TAP Package
```gherkin=
kubectl create namespace tap-install

tanzu secret registry add tap-registry \
  --username [自行輸入] \
  --password [自行輸入] \
  --server reg.microservice.tw \
  --namespace tap-install \
  --export-to-all-namespaces \
  --yes
  
tanzu package repository add tanzu-tap-repository \
  --url reg.microservice.tw/tanzu-application-platform/tap-packages:$TAP_VERSION \
  --namespace tap-install
  
tanzu package repository get tanzu-tap-repository --namespace tap-install

tanzu package available list --namespace tap-install

#安裝
tanzu package install tap -p tap.tanzu.vmware.com -v $TAP_VERSION \
  --values-file tap-values.yaml \
  --poll-timeout 45m \
  --namespace tap-install

#修改
tanzu package installed update tap -p tap.tanzu.vmware.com -v $TAP_VERSION \
  --values-file tap-values.yaml \
  --poll-timeout 45m \
  --namespace tap-install

#查詢
tanzu package installed list --namespace $TAP_NAMESPACE
#個別套件安裝細節
kubectl describe packageinstall/buildservice -n tap-install

#刪除
tanzu package installed delete tap  --namespace tap-install
```

### 安裝 Tanzu Build Service Full Dependency
```gherkin=
tanzu package repository add tbs-full-deps-repository \
  --url reg.microservice.tw/tanzu-application-platform/full-tbs-deps-package-repo:1.7.4 \
  --namespace tap-install

tanzu package repository get tbs-full-deps-repository --namespace $TAP_NAMESPACE 

tanzu package install full-tbs-deps -p full-tbs-deps.tanzu.vmware.com -v 1.7.4 -n tap-install

```
