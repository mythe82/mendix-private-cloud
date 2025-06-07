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






