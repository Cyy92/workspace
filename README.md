# Deploy OpenFx by minikube

다음은 minikube 개발환경에서 OpenFx를 배포하는 방법에 대한 가이드이다.

## Prerequisites

### Install Docker

- [Docker Homepage](https://docs.docker.com/install/linux/docker-ce/ubuntu/)에서 OS에 맞게 Docker 설치

### Install kubectl

> kubectl: kubernetes API를 사용하여 cluster와 통신하는 kubernetes CLI이다.

```
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
$ chmod +x kubectl
$ sudo mv kubectl /usr/local/bin
```

### Install minikube

> minikube: Kubernetes를 로컬에서 실항하게 해주는 툴이다.

```
$ curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
$ chmod +x minikube
$ sudo mv minikube /usr/local/bin

$ export MINIKUBE_WANTUPDATENOTIFICATION=false
$ export MINIKUBE_WANTREPORTERRORPROMPT=false
$ export MINIKUBE_HOME=$HOME
$ export CHANGE_MINIKUBE_NONE_USER=true
```

### Settings

- kubectl을 사용하기 위해서 다음과 같은 설정이 필요하다.

```
$ mkdir $HOME/.kube || true
$ touch $HOME/.kube/config
$ export KUBECONFIG=$HOME/.kube/config
```

## Get started: minikube

- Start minikube

```
$ sudo -E minikube start --vm-driver=none
```

- Stop minikube

```
$ sudo -E minikube stop
```

- Delete minikube

```
$ sudo -E minikube delete
```

## Deploy OpenFx

minikube까지 설치가 완료 되어서 Cluster가 구성되었으면 애플리케이션을 배포할 수 있는 환경을 구성해야 한다. 

### 1. Create namespaces

- namespace란 가상의 Cluster를 의미한다. namespace를 통해 사용자는 Cluster 자원을 공유할 수 있다. 

```
apiVersion: v1
kind: Namespace
metadata:
  name: openfx
---
apiVersion: v1
kind: Namespace
metadata:
  name: openfx-fn
```

> namespaces.yml: namespace를 생성하기 위한 설정 파일

- 이 설정 파일을 통해 다음과 같이 namespace를 생성한다.

```
$ kubectl apply -f ./namespaces.yml
```

- 생성이 제대로 되었는지 확인한다.

```
$ kubectl get namespaces
NAME               STATUS    AGE
default            Active    1d
kube-public        Active    1d
kube-system        Active    1d
openfx             Active    1d
openfx-fn          Active    1d
```

> Note
>
> 기본적으로 Kubernetes는 세 개의 초기 namespace로 시작한다.
> - default: 다른 namespace가 없는 개체의 기본 namespace
> 
> - kube-public: Kubernetes 시스템에 의해 생성된 객체의 namespace
> 
> - kube-system: 자동으로 만들어지는 namespace, 일부 자원을 전체 Cluster에서 공개적으로 표시하고 읽을 수 있어야 할 경우를 대비하여 예약되어 있다.

### 2. Create Deployment

> Deployment: 애플리케이션 인스턴스들의 생성과 업데이트를 지원하는 pod(하나 이상의 컨테이너들의 묶음)

- 다음의 명령을 통해 컨테이너 image를 지정하여 Deployment를 생성한다. 

```
$ kubectl run fxgateway --image=10.0.0.180:5000/fxgateway:0.1.0 --namespace=openfx
```

- Deployment 제대로 생성되었는지 확인한다. 

```
$ kubectl get deployment --namespace=openfx
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
fxgateway   1         1         1            1           1h

$ kubectl get pods --namespace=openfx
NAME                         READY   STATUS    RESTARTS   AGE
fxgateway-759965d6d8-qfv2x   1/1     Running   0          1h
```

- Deployment가 생성되고 Pod이 Running 상태인지까지 확인하면 다음과 같이 실행중인 컨테이너를 확인하고 애플리케이션이 배포되었는지 확인한다.

```
$ sudo docker ps
$ sudo docker logs ${CONTAINER_ID}
kubernetest host: https://10.96.0.1:443
[fxgateway] service start
```
