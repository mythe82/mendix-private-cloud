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

## 3. NFS 구성
```bash
# master node에 설치
mythe82@k8s-controller-1:~$ apt update
mythe82@k8s-controller-1:~$ sudo apt-get install -y nfs-common nfs-kernel-server rpcbind portmap

# NFS 서버에서 공유할 디렉터리를 생성
mythe82@k8s-controller-1:~$ cd /mnt
mythe82@k8s-controller-1:/mnt$ mkdir -p /mnt/k8s-nfs
mythe82@k8s-controller-1:/mnt$ chmod -R 777 k8s-nfs/
mythe82@k8s-controller-1:/mnt$ chown -R 1001.1001 k8s-nfs/

# 공유할 디렉터리 경로와 현재 사용하고 있는 서버 인스턴스의 서브넷 범위를 지정
mythe82@k8s-controller-1:/mnt$ echo '/mnt/k8s-nfs 10.178.0.0/24(rw,sync,no_root_squash,no_subtree_check)' | sudo tee -a /etc/exports
/mnt/k8s-nfs 10.178.0.0/24(rw,sync,no_root_squash,no_subtree_check)

# 설정을 적용하고 서비스를 재시작
mythe82@k8s-controller-1:/mnt$ sudo exportfs -a
mythe82@k8s-controller-1:/mnt$ sudo systemctl restart nfs-kernel-server

# worker node에 설치
mythe82@k8s-worker-1:~$ apt update
mythe82@k8s-worker-1:~$ sudo apt-get install -y nfs-common
mythe82@k8s-worker-1:~$ showmount -e 10.178.0.11
Export list for 10.178.0.11:
/mnt/k8s-nfs 10.178.0.0/24
```

* NFS storage class 생성
```bash
mythe82@k8s-controller-1:/mnt$ cd
mythe82@k8s-controller-1:~$ cat <<EOF > nfs-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: Immediate
EOF
```

```bash
mythe82@k8s-controller-1:~$ kubectl apply -f nfs-storageclass.yaml
mythe82@k8s-controller-1:~$ k get storageclasses.storage.k8s.io
NAME          PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-storage   kubernetes.io/no-provisioner   Delete          Immediate           false                  103s
```

* PersistentVolume 및 PersistentVolumeClaim 생성 test
```bash
mythe82@k8s-controller-1:~$ k get pv,pvc
No resources found

mythe82@k8s-controller-1:~$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-test-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-storage
  nfs:
    path: /mnt/k8s-nfs
    server: 10.178.0.11
EOF
```

```bash
mythe82@k8s-controller-1:~$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-test-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
  storageClassName: nfs-storage
EOF

mythe82@k8s-controller-1:~$ k get pv,pvc
NAME                           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/nfs-test-pv   1Gi        RWX            Retain           Bound    default/nfs-test-pvc   nfs-storage    <unset>                          48s

NAME                                 STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/nfs-test-pvc   Bound    nfs-test-pv   1Gi        RWX            nfs-storage    <unset>                 30s
```

* pod / pvc 테스트
```bash
mythe82@k8s-controller-1:~$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nfs-test-pod
spec:
  containers:
  - name: test-container
    image: busybox
    command:
      - sleep
      - "3600"
    volumeMounts:
    - name: nfs-storage
      mountPath: "/mnt/k8s-nfs"
  volumes:
  - name: nfs-storage
    persistentVolumeClaim:
      claimName: nfs-test-pvc
EOF

mythe82@k8s-controller-1:~$ k get po -o wide
NAME           READY   STATUS    RESTARTS   AGE     IP               NODE           NOMINATED NODE   READINESS GATES
nfs-test-pod   1/1     Running   0          3m13s   10.233.110.135   k8s-worker-1   <none>           <none>
```

```bash
# Pod 내부에서 NFS 마운트 확인
mythe82@k8s-controller-1:~$ kubectl exec -it nfs-test-pod -- /bin/sh
/ # echo "Hello NFS!" > /mnt/k8s-nfs/testfile.txt
/ # cat /mnt/k8s-nfs/testfile.txt
Hello NFS!
/ # exit

# NFS 서버에서도 파일 확인
mythe82@k8s-controller-1:~$ cat /mnt/k8s-nfs/testfile.txt
Hello NFS!

# 테스트가 완료되면 리소스를 정리
mythe82@k8s-controller-1:~$ kubectl delete pod nfs-test-pod
mythe82@k8s-controller-1:~$ kubectl delete pvc nfs-test-pvc
mythe82@k8s-controller-1:~$ kubectl delete pv nfs-test-pv
```

## 4. Helm 설치 (offline)
```bash
mythe82@k8s-controller-1:~$ cd mx/3_tekton/
mythe82@k8s-controller-1:~/mx/3_tekton$ tar -zxvf helm-v3.16.1-linux-amd64.tar.gz
mythe82@k8s-controller-1:~/mx/3_tekton$ sudo mv linux-amd64/helm /usr/local/bin/helm
mythe82@k8s-controller-1:~/mx/3_tekton$ helm version
```

## 5. Harbor 설치 (offline)
```bash
# online 환경에서 Harbor 릴리즈 패키지 다운로드
mythe82@k8s-controller-1:~$ HARBOR_VERSION="2.13.1"
mythe82@k8s-controller-1:~$ curl -L "https://github.com/goharbor/harbor/releases/download/v${HARBOR_VERSION}/harbor-offline-installer-v${HARBOR_VERSION}.tgz" -o "harbor-offline-installer-v${HARBOR_VERSION}.tgz"

# online 환경에서 Helm 차트 다운로드
mythe82@k8s-controller-1:~$ helm repo add harbor https://helm.goharbor.io
mythe82@k8s-controller-1:~$ helm repo update
mythe82@k8s-controller-1:~$ helm fetch harbor/harbor --version 1.17.1

# Harbor 이미지 로드
mythe82@k8s-controller-1:~$ tar xvfz harbor-offline-installer-v2.13.1.tgz
mythe82@k8s-controller-1:~$ tar xzvf harbor-1.17.1.tgz
mythe82@k8s-controller-1:~$ cd harbor/
mythe82@k8s-controller-1:~/harbor$ scp harbor.v2.13.1.tar.gz wk01:~/

# --- 각 노드에서 아래 명령어 실행 ---
# harbor 이미지 아카이브가 있는 곳으로 이동
# containerd의 k8s 네임스페이스로 이미지 로드 (필수)
mythe82@k8s-controller-1:~/harbor$ sudo nerdctl --namespace k8s.io load -i harbor.v2.13.1.tar.gz
mythe82@k8s-controller-1:~/harbor$ ssh wk01
mythe82@k8s-worker-1:~$ sudo nerdctl --namespace k8s.io load -i harbor.v2.13.1.tar.gz

# 로드 확인
mythe82@k8s-controller-1:~/harbor$ sudo nerdctl --namespace k8s.io images | grep "goharbor"
mythe82@k8s-worker-1:~$ sudo nerdctl --namespace k8s.io images | grep "goharbor"

goharbor/harbor-log                                    v2.13.1          0ceff3e80b06    About a minute ago    linux/amd64    171.6MB    166.2MB
goharbor/prepare                                       v2.13.1          4297b70787fd    About a minute ago    linux/amd64    226.7MB    213.7MB
goharbor/harbor-jobservice                             v2.13.1          08af37e4194d    About a minute ago    linux/amd64    179.6MB    175.9MB
goharbor/redis-photon                                  v2.13.1          7023398c8d85    About a minute ago    linux/amd64    173.4MB    168.4MB
goharbor/harbor-exporter                               v2.13.1          a09bffc17c4f    About a minute ago    linux/amd64    133.4MB    129.7MB
goharbor/harbor-registryctl                            v2.13.1          f606f750f05f    About a minute ago    linux/amd64    164.6MB    162.7MB
goharbor/harbor-db                                     v2.13.1          5501021458c2    About a minute ago    linux/amd64    288.6MB    278.5MB
goharbor/trivy-adapter-photon                          v2.13.1          d39f86d91af1    About a minute ago    linux/amd64    390.2MB    387.9MB
goharbor/registry-photon                               v2.13.1          9e1894bee0ff    About a minute ago    linux/amd64    88.74MB    86.82MB
goharbor/nginx-photon                                  v2.13.1          18413e62e4bb    About a minute ago    linux/amd64    158.1MB    153.4MB
goharbor/harbor-portal                                 <none>           d83ecad7f48b    2 weeks ago           linux/amd64    166.9MB    53.6MB
goharbor/harbor-portal                                 v2.13.1          9b31001ccde5    2 weeks ago           linux/amd64    166.9MB    162MB
goharbor/harbor-core                                   <none>           bed040aef107    2 weeks ago           linux/amd64    203.6MB    63.86MB
goharbor/harbor-core                                   v2.13.1          a22b58594cea    2 weeks ago           linux/amd64    203.6MB    199.6MB

mythe82@k8s-controller-1:~/harbor$ cp values.yaml my-harbor-values.yaml
mythe82@k8s-controller-1:~/harbor$ vi my-harbor-values.yaml
# 1. 외부 접속 주소 설정 (필수)
# DNS에 미리 등록하거나, NodePort 사용 시 노드 IP와 포트로 설정
expose:
  type: ingress # 또는 nodePort
  # ingress 사용 시 hosts 설정
  ingress:
    hosts:
      core: harbor.your-domain.com # 실제 사용할 도메인으로 변경
    # className: "nginx" # 사용하는 Ingress Controller에 맞게 설정

# 2. 이미지 Pull 정책 변경 (필수)
# 노드에 미리 로드한 로컬 이미지를 사용하도록 설정
imagePullPolicy: IfNotPresent

# 3. 이미지 저장소 이름 (수정 금지)
# 로드한 이미지 이름과 일치해야 하므로 기본값 'goharbor'를 유지
image:
  repository: goharbor

# 4. Harbor 관리자(admin) 비밀번호 설정 (강력 권장)
harborAdminPassword: "YourStrongPassword"

# 5. 스토리지 설정
# 사용하는 스토리지 클래스가 있다면 지정하는 것이 좋습니다.
persistence:
  enabled: true
  # resourcePolicy: "keep"
  # imageChartStorage:
  #   storageClass: "your-storage-class"
  # ...

mythe82@k8s-controller-1:~/harbor$ kubectl create namespace harbor

# 수정된 values.yaml로 Harbor의 모든 k8s 설정 파일을 다시 생성합니다.
mythe82@k8s-controller-1:~/harbor$ helm template harbor . -f my-harbor-values.yaml --namespace harbor > harbor-manifests.yaml

# 재생성된 설정 파일을 클러스터에 적용하여 기존 설정을 덮어씁니다.
mythe82@k8s-controller-1:~/harbor$ kubectl apply -f harbor-manifests.yaml
mythe82@k8s-controller-1:~/harbor$ kubectl get pvc -n harbor

# NFS 서버에서 실행
mythe82@k8s-controller-1:~/harbor$ mkdir -p /mnt/k8s-nfs/harbor/database
mythe82@k8s-controller-1:~/harbor$ mkdir -p /mnt/k8s-nfs/harbor/redis
mythe82@k8s-controller-1:~/harbor$ mkdir -p /mnt/k8s-nfs/harbor/registry
mythe82@k8s-controller-1:~/harbor$ mkdir -p /mnt/k8s-nfs/harbor/trivy
mythe82@k8s-controller-1:~/harbor$ mkdir -p /mnt/k8s-nfs/harbor/jobservice

# Kubernetes 노드들이 접근할 수 있도록 권한 설정
mythe82@k8s-controller-1:~/harbor$ sudo chmod -R 777 /mnt/k8s-nfs/harbor

# harbor-nfs-pv.yaml
mythe82@k8s-controller-1:~/harbor$ vi harbor-nfs-pv.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: harbor-database-pv
spec:
  capacity: { storage: 10Gi }
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-storage
  nfs:
    server: 10.178.0.11
    path: "/mnt/k8s-nfs/harbor/database"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: harbor-redis-pv
spec:
  capacity: { storage: 2Gi }
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-storage
  nfs:
    server: 10.178.0.11
    path: "/mnt/k8s-nfs/harbor/redis"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: harbor-registry-pv
spec:
  capacity: { storage: 20Gi }
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-storage
  nfs:
    server: 10.178.0.11
    path: "/mnt/k8s-nfs/harbor/registry"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: harbor-trivy-pv
spec:
  capacity: { storage: 10Gi }
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-storage
  nfs:
    server: 10.178.0.11
    path: "/mnt/k8s-nfs/harbor/trivy"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: harbor-jobservice-pv
spec:
  capacity: { storage: 2Gi }
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-storage
  nfs:
    server: 10.178.0.11
    path: "/mnt/k8s-nfs/harbor/jobservice"

mythe82@k8s-controller-1:~/harbor$ kubectl apply -f harbor-nfs-pv.yaml
mythe82@k8s-controller-1:~/harbor$ kubectl get pvc -n harbor

NAME                              STATUS   VOLUME                 CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
data-harbor-redis-0               Bound    harbor-trivy-pv        10Gi       RWO            nfs-storage    <unset>                 15m
data-harbor-trivy-0               Bound    harbor-registry-pv     20Gi       RWO            nfs-storage    <unset>                 15m
database-data-harbor-database-0   Bound    harbor-redis-pv        2Gi        RWO            nfs-storage    <unset>                 15m
harbor-jobservice                 Bound    harbor-jobservice-pv   2Gi        RWO            nfs-storage    <unset>                 15m
harbor-registry                   Bound    harbor-database-pv     10Gi       RWO            nfs-storage    <unset>                 15m
```




