# mendix private cloud install

## 1. K8s cluster 환경 구성
* 하기 repo를 참고하여 구성 필요

[k8s 구성 - https://github.com/mythe82/k8s-offline-setup/blob/main/k8s-online-install.md](https://github.com/mythe82/k8s-offline-setup/blob/main/k8s-online-install.md)

```bash
mythe82@k8s-controller-1:~$ source ~/kubespray-venv/bin/activate
(kubespray-venv) mythe82@k8s-controller-1:~$ cd kubespray/contrib/offline/
(kubespray-venv) mythe82@k8s-controller-1:~/kubespray/contrib/offline$ ./generate_list.sh -i inventory/mycluster/inventory.ini
(kubespray-venv) mythe82@k8s-controller-1:~/kubespray/contrib/offline$ cd temp/
(kubespray-venv) mythe82@k8s-controller-1:~/kubespray/contrib/offline/temp$ ls
files.list  files.list.template  images.list  images.list.template
```

### 1.2. 이미지 수집
이미지 리스트 파일을 기반으로 오프라인 이미지를 다운로드합니다.

```bash
(kubespray-venv) mythe82@k8s-controller-1:~/kubespray/contrib/offline/temp$ IMAGES_FROM_FILE=temp/images.list
(kubespray-venv) mythe82@k8s-controller-1:~/kubespray/contrib/offline/temp$ cd ..
(kubespray-venv) mythe82@k8s-controller-1:~/kubespray/contrib/offline$ ./manage-offline-container-images.sh create
(kubespray-venv) mythe82@k8s-controller-1:~/kubespray/contrib/offline$ ls -ltr
total 546652
-rwxrwxr-x 1 mythe82 mythe82      1406 May 22 06:41 manage-offline-files.sh
-rwxrwxr-x 1 mythe82 mythe82      7625 May 22 06:41 manage-offline-container-images.sh
-rw-rw-r-- 1 mythe82 mythe82       575 May 22 06:41 generate_list.yml
-rwxrwxr-x 1 mythe82 mythe82      1329 May 22 06:41 generate_list.sh
-rw-rw-r-- 1 mythe82 mythe82        44 May 22 06:41 docker-daemon.json
-rw-rw-r-- 1 mythe82 mythe82      2912 May 22 06:41 README.md
-rwxrwxr-x 1 mythe82 mythe82      2469 May 22 06:41 upload2artifactory.py
-rw-rw-r-- 1 mythe82 mythe82       189 May 22 06:41 registries.conf
-rw-rw-r-- 1 mythe82 mythe82      1186 May 22 06:41 nginx.conf
drwxrwxr-x 2 mythe82 mythe82      4096 May 22 08:17 temp
-rw-rw-r-- 1 mythe82 mythe82 559721189 May 22 08:27 container-images.tar.gz
```

## 2. 오프라인용 파일 수집
파일 리스트에 따라 오프라인 설치에 필요한 파일을 다운로드하고, nginx 컨테이너로 파일 서버를 만듭니다.

```bash
(kubespray-venv) mythe82@k8s-controller-1:~/kubespray/contrib/offline$ ./manage-offline-files.sh
(kubespray-venv) mythe82@k8s-controller-1:~/kubespray/contrib/offline$ ls -ltr
total 1448628
-rwxrwxr-x 1 mythe82 mythe82      1406 May 22 06:41 manage-offline-files.sh
-rwxrwxr-x 1 mythe82 mythe82      7625 May 22 06:41 manage-offline-container-images.sh
-rw-rw-r-- 1 mythe82 mythe82       575 May 22 06:41 generate_list.yml
-rwxrwxr-x 1 mythe82 mythe82      1329 May 22 06:41 generate_list.sh
-rw-rw-r-- 1 mythe82 mythe82        44 May 22 06:41 docker-daemon.json
-rw-rw-r-- 1 mythe82 mythe82      2912 May 22 06:41 README.md
-rwxrwxr-x 1 mythe82 mythe82      2469 May 22 06:41 upload2artifactory.py
-rw-rw-r-- 1 mythe82 mythe82       189 May 22 06:41 registries.conf
-rw-rw-r-- 1 mythe82 mythe82      1186 May 22 06:41 nginx.conf
drwxrwxr-x 2 mythe82 mythe82      4096 May 22 08:17 temp
-rw-rw-r-- 1 mythe82 mythe82 559721189 May 22 08:27 container-images.tar.gz
drwxrwxr-x 6 mythe82 mythe82      4096 May 22 08:37 offline-files
-rw-rw-r-- 1 mythe82 mythe82 923613797 May 22 08:38 offline-files.tar.gz
```

## 3. infra 구성
### 3.1. VM 준비
GCP GCE 2대 생성하여 VM 구성
* k8s-controller-1
  - 머신 유형 e2-medium (vCPU 2개, 메모리 4GB)
  - 200GB 표준 영구 디스크
  - 이미지 ubuntu-2204-jammy-v20250508

* k8s-worker-1
  - 머신 유형 e2-medium (vCPU 2개, 메모리 4GB)
  - 100GB 표준 영구 디스크
  - 이미지 ubuntu-2204-jammy-v20250508

 * GCP VPC 생성
   
Cloud Shell 터미널에서 명령어 실행
```Cloud Shell
mythe82@cloudshell:~ (malee-457606)$ gcloud compute networks create kubernetes-the-kubespray-way --subnet-mode custom
Created [https://www.googleapis.com/compute/v1/projects/malee-457606/global/networks/kubernetes-the-kubespray-way].
NAME: kubernetes-the-kubespray-way
SUBNET_MODE: CUSTOM
BGP_ROUTING_MODE: REGIONAL
IPV4_RANGE: 
GATEWAY_IPV4: 
INTERNAL_IPV6_RANGE: 
```
