# mendix for private cloud install

## 1. K8s cluster 환경 구성
* 하기 repo를 참고하여 구성 필요
* [k8s 구성 - https://github.com/mythe82/k8s-offline-setup/blob/main/k8s-online-install.md](https://github.com/mythe82/k8s-offline-setup/blob/main/k8s-online-install.md)

## 2. mendix 설치 관련 PKG master node에 upload
```bash
mythe82@k8s-controller-1:~$ pwd
/home/mythe82
mythe82@k8s-controller-1:~$ tar xvf mx.tar
mythe82@k8s-controller-1:~/mx$ cd mx/
mythe82@k8s-controller-1:~/mx$ tree -L 2
.
├── 1_mx_operator
│   ├── mendix-operator.tar
│   ├── mxpc-cli
│   └── mxpc-cli-2.19.0-linux-amd64.tar.gz
├── 2_PCLM
│   ├── PCLM_nodeport.yaml
│   ├── manifest.yaml
│   ├── mx-pclm-cli
│   ├── mx-pclm-cli-0.5.0-linux-amd64.tar.gz
│   └── pclm-image.tar
├── 3_tekton
│   ├── aip
│   ├── airgapped-image-package-0.0.2-linux-amd64.tar.gz
│   ├── helm-v3.16.1-linux-amd64.tar.gz
│   ├── linux-amd64
│   ├── pipeline
│   ├── standalone-cicd-v1.0.4
│   ├── standalone-cicd-v1.0.4.tar
│   ├── tekton
│   ├── tekton.tar
│   └── tekton_dashboard_image.tar
├── harbor
│   └── harbor
├── harbor-data-pv.yaml
├── metallb-config.yaml
├── minio
│   └── values.yaml
├── minio-data-pv.yaml
├── nexus
│   ├── cert
│   └── helm-charts
├── nexus-data-pv.yaml
├── nexus-ingress.yaml
├── nfs-storageclass.yaml
├── postgresql
│   └── postgresql
└── postgresql-data-pv.yaml

15 directories, 22 files
```
