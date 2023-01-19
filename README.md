# tap-install-note

鏡像倉庫準備(強烈建議，安裝 TAP 時使用個人的鏡像倉庫，因為 VMware Tanzu Registry 不保證連線品質，會造成不明原因安裝失敗)
---
須先建立repo

```gherkin=
tanzu-application-platform

tanzu-cluster-essentials
```

下載 tanzu-cluster-essentials、TAP packages、TBS full dependency 並上傳至 private registry (一次性工作)
> 登入 registry.tanzu.vmware.com 與 reg.microservice.tw (private registry)
```gherkin=
docker login registry.tanzu.vmware.com

docker login reg.microservice.tw
```
> Tanzu Cluster Essentials
```gherkin=
imgpkg copy \
    -b registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:54bf611711923dccd7c7f10603c846782b90644d48f1cb570b43a082d18e23b9 \
    --to-tar cluster-essentials-bundle-1.3.0.tar \
    --include-non-distributable-layers
    
imgpkg copy \
    --tar cluster-essentials-bundle-1.3.0.tar \
    --to-repo harbor.example.com/tanzu-cluster-essentials/cluster-essentials-bundle \
    --include-non-distributable-layers \
    --registry-ca-cert-path microservice.tw.crt    
```
> TAP packages
```gherkin=
imgpkg copy   -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.3.3   --to-tar tap-packages-1.3.3.tar   --include-non-distributable-layers

imgpkg copy   --tar tap-packages-1.3.3.tar   --to-repo reg.microservice.tw/tanzu-application-platform/tap-packages   --include-non-distributable-layers
```
>TBS full dependency
```gherkin=
tanzu package available list buildservice.tanzu.vmware.com --namespace tap-install

imgpkg copy -b registry.tanzu.vmware.com/tanzu-application-platform/full-tbs-deps-package-repo:1.7.4   --to-tar=tbs-full-deps-1.7.4.tar
```
