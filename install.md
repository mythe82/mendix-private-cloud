# mendix for private cloud install

## 1. K8s cluster 환경 구성
* 하기 repo를 참고하여 구성 필요

[k8s 구성 - https://github.com/mythe82/k8s-offline-setup/blob/main/k8s-online-install.md](https://github.com/mythe82/k8s-offline-setup/blob/main/k8s-online-install.md)

### 1.1. Helm 설치
```bash
mythe82@k8s-controller-1:~$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
mythe82@k8s-controller-1:~$ chmod 700 get_helm.sh
mythe82@k8s-controller-1:~$ ./get_helm.sh -v v3.16.1
Downloading https://get.helm.sh/helm-v3.16.1-linux-amd64.tar.gz
Verifying checksum... Done.
Preparing to install helm into /usr/local/bin
helm installed into /usr/local/bin/helm

mythe82@k8s-controller-1:~$ helm version
version.BuildInfo{Version:"v3.16.1", GitCommit:"5a5449dc42be07001fd5771d56429132984ab3ab", GitTreeState:"clean", GoVersion:"go1.22.7"}
```

### 1.2. NFS 구성
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
mythe82@k8s-controller-1:~$ mkdir test && cd test
mythe82@k8s-controller-1:~/test$ cat <<EOF > nfs-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: Immediate
EOF
```

```bash
mythe82@k8s-controller-1:~/test$ kubectl get storageclasses.storage.k8s.io 
No resources found

mythe82@k8s-controller-1:~/test$ kubectl apply -f nfs-storageclass.yaml
storageclass.storage.k8s.io/nfs-storage created

mythe82@k8s-controller-1:~/test$ k get storageclasses.storage.k8s.io 
NAME          PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-storage   kubernetes.io/no-provisioner   Delete          Immediate           false                  4s
```

* PersistentVolume 및 PersistentVolumeClaim 생성 test
```bash
mythe82@k8s-controller-1:~/test$ k get pv,pvc
No resources found

mythe82@k8s-controller-1:~/test$ cat <<EOF | kubectl apply -f -
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
mythe82@k8s-controller-1:~/test$ cat <<EOF | kubectl apply -f -
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

mythe82@k8s-controller-1:~/test$ kubectl get pv,pvc
NAME                           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/nfs-test-pv   1Gi        RWX            Retain           Bound    default/nfs-test-pvc   nfs-storage    <unset>                          2m13s

NAME                                 STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/nfs-test-pvc   Bound    nfs-test-pv   1Gi        RWX            nfs-storage    <unset>                 55s
```

* pod / pvc 테스트
```bash
mythe82@k8s-controller-1:~/test$ cat <<EOF | kubectl delete -f -
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

mythe82@k8s-controller-1:~/test$ k get po -o wide
NAME           READY   STATUS    RESTARTS   AGE     IP               NODE           NOMINATED NODE   READINESS GATES
nfs-test-pod   1/1     Running   0          3m13s   10.233.110.135   k8s-worker-1   <none>           <none>
```

```bash
# Pod 내부에서 NFS 마운트 확인
mythe82@k8s-controller-1:~/test$ kubectl exec -it nfs-test-pod -- /bin/sh
/ # echo "Hello NFS!" > /mnt/k8s-nfs/testfile.txt
/ # cat /mnt/k8s-nfs/testfile.txt
Hello NFS!
/ # exit

# NFS 서버에서도 파일 확인
mythe82@k8s-controller-1:~/test$ ls /mnt/k8s-nfs
testfile.txt

mythe82@k8s-controller-1:~/test$ cat /mnt/k8s-nfs/testfile.txt
Hello NFS!

# 테스트가 완료되면 리소스를 정리
mythe82@k8s-controller-1:~/test$ kubectl delete pod nfs-test-pod
pod "nfs-test-pod" deleted

mythe82@k8s-controller-1:~/test$ kubectl delete pvc nfs-test-pvc
persistentvolumeclaim "nfs-test-pvc" deleted

mythe82@k8s-controller-1:~/test$ kubectl delete pv nfs-test-pv
persistentvolume "nfs-test-pv" deleted
```

### 1.3 SSL 인증서 구성
* cert-manager 설치
```bash
mythe82@k8s-controller-1:~$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.crds.yaml
mythe82@k8s-controller-1:~$ helm repo add jetstack https://charts.jetstack.io
mythe82@k8s-controller-1:~$ helm repo update
mythe82@k8s-controller-1:~$ helm repo list
mythe82@k8s-controller-1:~$ helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace
```
* cluster 내부용 CA 발급 및 Secret 생성
```bash
# rootCA 키와 인증서 생성
mythe82@k8s-controller-1:~$ openssl genrsa -out rootCA.key 2048
mythe82@k8s-controller-1:~$ openssl req -x509 -new -nodes -key rootCA.key -subj "/CN=k8sInternalCA" -days 3650 -out rootCA.crt

# Kubernetes에 Secret으로 등록
mythe82@k8s-controller-1:~$ kubectl create secret tls k8s-ca-secret \
  --cert=rootCA.crt \
  --key=rootCA.key \
  -n cert-manager
```
* ClusterIssuer 생성
```bash
mythe82@k8s-controller-1:~$ cat <<EOF > clusterissuer-ca.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: k8s-internal-ca
spec:
  ca:
    secretName: k8s-ca-secret
EOF

mythe82@k8s-controller-1:~$ kubectl apply -f clusterissuer-ca.yaml
```
* Wildcard 인증서 생성
```bash
mythe82@k8s-controller-1:~$ cat <<EOF > wildcard-cert.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-cert
  namespace: default
spec:
  secretName: wildcard-tls
  issuerRef:
    name: k8s-internal-ca
    kind: ClusterIssuer
  commonName: '*.malee.mds'
  dnsNames:
    - '*.malee.mds'
  duration: 87600h
  renewBefore: 720h
EOF

mythe82@k8s-controller-1:~$ kubectl apply -f wildcard-cert.yaml
mythe82@k8s-controller-1:~$ kubectl describe certificate wildcard-cert -n default
```

### 1.4. Nginx ingress controller 설치 (online)
```bash
# NGINX Ingress Controller를 배포할 namespace 생성
mythe82@k8s-controller-1:/mnt$ kubectl create namespace ingress-nginx

# Helm Chart 저장소 추가
mythe82@k8s-controller-1:/mnt$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
mythe82@k8s-controller-1:/mnt$ helm repo list
mythe82@k8s-controller-1:/mnt$ helm repo update

# NGINX Ingress Controller 설치
mythe82@k8s-controller-1:/mnt$ helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443
```
* 추가적으로 Helm Chart의 설정값을 커스터마이징하려면 아래 명령을 사용하여 설정값 파일을 생성하고 수정할 수 있습니다:
```bash
mythe82@k8s-controller-1:~$ helm show values ingress-nginx/ingress-nginx > helm-values.yaml
```

```bash
# 설치 확인
mythe82@k8s-controller-1:~$ kubectl get all -n ingress-nginx
NAME                                            READY   STATUS    RESTARTS   AGE
pod/ingress-nginx-controller-6885cfc548-75h9g   1/1     Running   0          4m55s

NAME                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.233.41.210   <none>        80:30080/TCP,443:30443/TCP   4m55s
service/ingress-nginx-controller-admission   ClusterIP   10.233.12.54    <none>        443/TCP                      4m55s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           4m55s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-6885cfc548   1         1         1       4m55s
```

* 간단한 nginx APP 생성
```bash
mythe82@k8s-controller-1:~$ cd test
mythe82@k8s-controller-1:~/test$ cat <<EOF > nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  labels:
    app: my-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
spec:
  selector:
    app: my-nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF
```

* Ingress 리소스 생성 test
```bash
mythe82@k8s-controller-1:~/test$ cat <<EOF > nginx-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-nginx-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - malee.mds
      secretName: wildcard-tls
  rules:
    - host: malee.mds
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-nginx
                port:
                  number: 80
EOF
```

* 리소스 적용
```bash
mythe82@k8s-controller-1:~/test$ kubectl apply -f nginx-deploy.yaml
mythe82@k8s-controller-1:~/test$ kubectl apply -f nginx-ingress.yaml
mythe82@k8s-controller-1:~/test$ sudo vi /etc/hosts
35.216.9.4      malee.mds

mythe82@k8s-controller-1:~/test$ curl -k https://malee.mds:30443

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

* 참고 - NGINX Ingress Controller 제거
```bash
helm uninstall ingress-nginx --namespace ingress-nginx
kubectl delete namespace ingress-nginx
```

## ~~1.7. Harbor 설치 (online)~~

```bash
root@cp-k8s:~/mx# cd harbor/
root@cp-k8s:~/mx/harbor# helm repo add harbor https://helm.goharbor.io

# 압축파일 다운로드 및 해제
root@cp-k8s:~/mx/harbor# helm fetch harbor/harbor --untar

# namespace 생성
root@cp-k8s:~/mx/harbor# kubectl create ns harbor
```

```yaml
root@cp-k8s:~/mx/harbor# cd harbor/
root@cp-k8s:~/mx/harbor/harbor# vi values.yaml

expose:
  # Set how to expose the service. Set the type as "ingress", "clusterIP", "nodePort" or "loadBalancer"
  # and fill the information in the corresponding section
  type: ingress
  tls:
    # Enable TLS or not.
    # Delete the "ssl-redirect" annotations in "expose.ingress.annotations" when TLS is disabled and "expose.type" is "ingress"
    # Note: if the "expose.type" is "ingress" and TLS is disabled,
    # the port must be included in the command when pulling/pushing images.
    # Refer to https://github.com/goharbor/harbor/issues/5291 for details.
    enabled: true
    # The source of the tls certificate. Set as "auto", "secret"
    # or "none" and fill the information in the corresponding section
    # 1) auto: generate the tls certificate automatically
    # 2) secret: read the tls certificate from the specified secret.
    # The tls certificate can be generated manually or by cert manager
    # 3) none: configure no tls certificate for the ingress. If the default
    # tls certificate is configured in the ingress controller, choose this option
    certSource: auto
    auto:
      # The common name used to generate the certificate, it's necessary
      # when the type isn't "ingress"
      commonName: ""
    secret:
      # The name of secret which contains keys named:
      # "tls.crt" - the certificate
      # "tls.key" - the private key
      secretName: ""
  ingress:
    hosts:
      core: harbor.mxtest.com
    # set to the type of ingress controller if it has specific requirements.
    # leave as `default` for most ingress controllers.
    # set to `gce` if using the GCE ingress controller
    # set to `ncp` if using the NCP (NSX-T Container Plugin) ingress controller
    # set to `alb` if using the ALB ingress controller
    # set to `f5-bigip` if using the F5 BIG-IP ingress controller
    controller: default
    ## Allow .Capabilities.KubeVersion.Version to be overridden while creating ingress
    kubeVersionOverride: ""
    className: ""
    
externalURL: https://harbor.mxtest.com

persistence:
  enabled: true
  # Setting it to "keep" to avoid removing PVCs during a helm delete
  # operation. Leaving it empty will delete PVCs after the chart deleted
  # (this does not apply for PVCs that are created for internal database
  # and redis components, i.e. they are never deleted automatically)
  resourcePolicy: "keep"
  persistentVolumeClaim:
    registry:
      # Use the existing PVC which must be created manually before bound,
      # and specify the "subPath" if the PVC is shared with other components
      existingClaim: ""
      # Specify the "storageClass" used to provision the volume. Or the default
      # StorageClass will be used (the default).
      # Set it to "-" to disable dynamic provisioning
      storageClass: "nfs-storage"
      subPath: ""
      accessMode: ReadWriteMany
      size: 5Gi
      annotations: {}
    jobservice:
      jobLog:
        existingClaim: ""
        storageClass: "nfs-storage"
        subPath: ""
        accessMode: ReadWriteMany
        size: 1Gi
        annotations: {}
    # If external database is used, the following settings for database will
    # be ignored
    database:
      existingClaim: ""
      storageClass: "nfs-storage"
      subPath: ""
      accessMode: ReadWriteMany
      size: 1Gi
      annotations: {}
    # If external Redis is used, the following settings for Redis will
    # be ignored
    redis:
      existingClaim: ""
      storageClass: "nfs-storage"
      subPath: ""
      accessMode: ReadWriteMany
      size: 1Gi
      annotations: {}
   trivy:
      existingClaim: ""
      storageClass: "nfs-storage"
      subPath: ""
      accessMode: ReadWriteMany
      size: 2Gi
      annotations: {}
```



