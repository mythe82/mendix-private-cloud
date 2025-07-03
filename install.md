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

## 5. Nexus 설치
```bash
mythe82@k8s-controller-1:~$ mkdir nexus
mythe82@k8s-controller-1:~$ cd nexus/
mythe82@k8s-controller-1:~/nexus$ git clone https://github.com/stevehipwell/helm-charts.git
```

```yaml
mythe82@k8s-controller-1:~/nexus$ vi ./helm-charts/charts/nexus3/values.yaml

# 아래 값을 찾아 변경
image:
  repository: docker.io/sonatype/nexus3
  tag: latest
  pullPolicy: IfNotPresent

persistence:
  enabled: true
  accessMode: ReadWriteOnce
  storageClass: nfs-storage
  size: 20Gi

# ingress 섹션을 비활성화하거나 제거합니다.
ingress:
  enabled: false # 이 값을 false로 설정하거나, ingress 섹션 자체를 제거합니다.

service:
  # -- Service type.
  type: NodePort # ClusterIP 대신 NodePort로 변경합니다.
  # -- Service port.
  port: 8081 # Nexus가 사용하는 기본 포트입니다.
  # -- NodePort service port.
  nodePort: 30081 # 외부에서 접근할 NodePort를 지정합니다. (30000-32767 범위 내에서 선택)
  # -- Annotations for the `Service`.
  annotations: {}
  # -- Additional labels for the `Service`.
  labels: {}
```

* PV 생성
```yaml
mythe82@k8s-controller-1:~/nexus$ sudo mkdir -p /mnt/k8s-nfs/nexus
mythe82@k8s-controller-1:~/nexus$ vi nexus-data-pv.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: nexus-data-pv
spec:
  capacity:
    storage: 20Gi # PVC의 요구사항에 맞게 조정
  accessModes:
    - ReadWriteOnce # PVC의 AccessModes에 맞게 설정 (PVC의 AccessModes 확인 필요)
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-storage # PVC의 StorageClass와 동일하게 설정
  nfs:
    path: /mnt/k8s-nfs/nexus # NFS 서버에서 공유하는 디렉터리
    server: 10.178.0.11 # NFS 서버의 IP 주소로 변경
```

* nexus 설치
```bash
mythe82@k8s-controller-1:~/nexus$ kubectl apply -f nexus-data-pv.yaml
mythe82@k8s-controller-1:~/nexus$ k get pv
mythe82@k8s-controller-1:~/nexus$ cd ./helm-charts/charts/nexus3
mythe82@k8s-controller-1:~/nexus/helm-charts/charts/nexus3$ helm install nexus . --namespace nexus --create-namespace
```

* Nexsus 대시보드 설정

```bash
# web-ui password 확인
mythe82@k8s-controller-1:~/nexus$ kubectl exec -it -n nexus nexus-nexus3-0 -- cat /nexus-data/admin.password
24bfcb3f-47c5-4350-af04-fe86ba9c8bdc
```
  - ID: admin
  - password: qwe1212!Q

* docker image를 위한 repository 구성

Repository를 생성하기 위해서는 Blob Store를 생성 후 Repository에 연결해야 한다.

***상단의 톱니바퀴 모양을 눌러 Administration 메뉴로 진입 → Blob Store 탭 → 상단의 파란색 Create Blob Store 버튼 클릭***

외부 Object Storage에 연결하지 않고 로컬 서버의 저장소를 이용할 것이므로 ***Type을 File로 지정한 뒤 해당 Blob Store 이름을 지정 → save 클릭***

Blob Store 설정 완료 후 Repository 생성

Local 환경의 Repository와 Docker Hub와 연결되는 Repository를 생성한다.

***Administration → Repository → Repositories** 항목 클릭 **→ 좌측 상단의 Create Repository 버튼을 클릭***

해당 버튼을 클릭하면 Nexus에서 제공하는 Repository Recipe 리스트가 나온다.

Recipe는 해당 저장소에 대한 Tempelate라고 생각하면 된다. Nexus는 apt 저장소부터 Maven, yum, docker 등 다양한 저장소에 대한 Recipe를 제공한다.

***Local Repository를 만들기 위해 docker (hosted) Recipe를 선택***

***Local Repository의 이름을 private-reg-local로 설정***

***HTTP 설정을 체크한 뒤 Port 5080 설정***

***Docker API v1을 지원하기 위해 Enable Docker V1 API 옵션을 활성화***

***생성할 Repository가 어떤 Blob Store를 사용할지 설정 앞 단계에서 생성했던 private-reg-local Blob Store를 사용***

***하단의 Create Repository 버튼을 클릭하여 Repository 생성***

***Repository가 생성되었다면 Docker Login 설정을 위해 Administration → Security  Realms 설정 → Docker Bearer Token Realm 활성화***

***해당 설정 후 우측 하단의 파란색 Save 버튼을 클릭하여 Docker Login 설정을 적용***

- Docker 로그인 및 이미지 테스트

private registry nexus는 기본적으로 인증되지 않은 저장소이며 TLS 통신을 하지 않아 docker daemon에서 해당 서버로의 접근을 차단하므로 미리 아래 옵션 설정 필요

```yaml
# containers 섹션에 5080 포트 추가
root@cp-k8s:~/mx# kubectl edit statefulset nexus-nexus3 -n nexus

        name: nexus3
        ports:
        - containerPort: 8081
          name: http
          protocol: TCP
        - containerPort: 5080
          name: nexus
          protocol: TCP

# 수정 후 StatefulSet 재시작
root@cp-k8s:~/mx# kubectl rollout restart statefulset nexus-nexus3 -n nexus
```

```yaml
# Service에 port: 5080 및 targetPort: 5080 추가
root@cp-k8s:~/mx# kubectl edit svc nexus-nexus3 -n nexus

  ports:
  - name: http
    port: 8081
    protocol: TCP
    targetPort: http
  - name: nexus
    port: 5080
    protocol: TCP
    targetPort: nexus
```

```yaml
# Ingress path /v2/ 5080 포트 라우팅 추가
root@cp-k8s:~/mx# kubectl edit ingress nexus-ingress -n nexus

spec:
  ingressClassName: nginx
  rules:
  - host: nexus.mxtest.com
    http:
      paths:
      - backend:
          service:
            name: nexus-nexus3
            port:
              number: 8081
        path: /
        pathType: Prefix
      - backend:
          service:
            name: nexus-nexus3
            port:
              number: 5080
        path: /v2/
        pathType: Prefix
status:
  loadBalancer:
    ingress:
    - ip: 192.168.56.20
```

```bash
# nexus cluster IP 확인
root@cp-k8s:~/mx# kubectl get svc -n nexus
```

```json
root@cp-k8s:~/mx# vi /etc/docker/daemon.json

{
        "insecure-registries": ["10.96.248.71:5080", "nexus.mxtest.com:5080"]
}
```

```bash
root@cp-k8s:~/mx# systemctl restart docker.service

# master/worker node 모두 적용
root@cp-k8s:~/mx# vi /etc/hosts
127.0.0.1 localhost
192.168.56.10 cp-k8s
192.168.56.11 w1-k8s
10.96.248.71 nexus.mxtest.com
```

nexus 대시보드 계정인 admin 이용

```bash
root@cp-k8s:~/mx# docker login http://nexus.mxtest.com:5080

Username: admin
Password:
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credential-stores

Login Succeeded
```

```bash
root@cp-k8s:~/mx# docker pull redis:latest
root@cp-k8s:~/mx# docker image tag redis:latest nexus.mxtest.com:5080/redis-test:1.0
root@cp-k8s:~/mx# docker push nexus.mxtest.com:5080/redis-test:1.0


