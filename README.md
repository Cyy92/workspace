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

minikube까지 설치가 완료 되어서 Cluster가 구성되었으면 API 단위 응용 기술을 배포할 수 있는 환경을 구성해야 한다. 

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

### 2. Build Docker Image

> Note
>
> minikube를 통해서 cluster를 구성하는 경우 호스트 컴퓨터에 Docker Registry를 만들고 그 안에 image를 push할 필요가 없다. 단순히 Docker Image에 
> 'latest' 이외의 태그, 즉 image의 버전을 지정하여 build 하고 지정한 태그를 이용해 image를 가져온다. image의 버전을 지정하지 않으면 컨테이너 실 > 행 시, 자동으로 `:latest` 라는 태그가 지정되고, `imagePullPolicy: Always`에 의해 `ErrImagePull` 에러가 발생한다. 

- namespace에서 실행할 컨테이너를 위한 image를 Dockerfile을 통해 각각 build하여야 한다. Dockerfile이란 image의 설정 정보를 갖고 있는 파일이고,  Dockerfile은 gateway, watcher의 서버 코드가 있는 경로에 있어야 한다.  

```
$ vim /go/src/github.com/keti-openfx/openfx-gateway/Dockerfile
```

```
FROM golang:1.10.1 as builder

RUN mkdir -p /go/src/github.com/keti-openfx/openfx-gateway

WORKDIR /go/src/github.com/keti-openfx/openfx-gateway

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build --ldflags "-s -w" \
        -a -installsuffix cgo -o fxgateway .


FROM alpine:3.7

RUN addgroup -S app \
        && adduser -S -g app app \
        && apk --no-cache add \
        ca-certificates
WORKDIR /home/app

EXPOSE 10000

COPY --from=builder /go/src/github.com/keti-openfx/openfx-gateway    .
RUN chown -R app:app ./

USER app

CMD ["./fxgateway"]
```

- 다음의 명령을 통해 image를 build 한다.

```
$ sudo docker build -t 10.0.0.180:5000/fxgateway:0.1.0 .
```

---

```
$ vim /go/src/github.com/keti-openfx/openfx-watcher/go/Dockerfile
```

```
# Argrumnets for FROM
ARG GO_VERSION=1.9.7

# Build fxwatcher
FROM golang:${GO_VERSION}

RUN mkdir -p ${GOPATH}/src/github.com/keti-openfx/openfx-watcher/go
WORKDIR ${GOPATH}/src/github.com/keti-openfx/openfx-watcher/go

COPY . .

RUN gofmt -l -d $(find . -type f -name '*.go' -not -path "./vendor/*")
RUN go build -o fxwatcher .

RUN curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh

CMD ["./fxwatcher"]
```

- 다음의 명령을 통해 image를 build 한다.

```
$ sudo docker build -t 10.0.0.180:5000/fxwatcher:0.1.0-go .
```

---

```
$ vim /go/src/github.com/keti-openfx/Functions/sqrt-go/Dockerfile
```

```
#Argrumnets for FROM
ARG REGISTRY=10.0.0.180:5000
ARG GO_VERSION=1.9.7
ARG WATCHER_VERSION=0.1.0

# Get fxwatcher - if fxwatcher is uploaded on github, remove this line.
FROM ${REGISTRY}/fxwatcher:${WATCHER_VERSION}-go as watcher

ARG handler_file=handler.go
ARG handler_name=Handler

ENV HANDLER_DIR=${GOPATH}/src/openfx/handler
ENV HANDLER_FILE=${GOPATH}/src/${HANDLER_DIR}/${handler_file}
ENV HANDLER_NAME=${handler_name}

RUN cp ${GOPATH}/src/github.com/keti-openfx/openfx-watcher/go/fxwatcher /fxwatcher

RUN chmod +x /fxwatcher

RUN mkdir -p ${HANDLER_DIR}
WORKDIR ${HANDLER_DIR}
COPY . .
RUN dep ensure
RUN go build -o ${HANDLER_FILE} -buildmode=plugin .

HEALTHCHECK --interval=2s CMD [ -e /tmp/.lock ] || exit 1

CMD ["/fxwatcher"]
```

- 다음의 명령을 통해 image를 build 한다.

```
$ sudo docker build -t 10.0.0.180:5000/sqrt-go:0.1.0 .
```

### 3. Create deployments in each namespace

- API 단위 응용 기술은 우선적으로 Gateway에 사용자 요청이 입력된다. 이 사용자 요청에 대한 입력 데이터는 Watcher에서 처리되고 그 결과 데이터를 다시 Gateway를 통해 사용자에게 반환하는 과정을 거친다. 때문에 생성해 놓았던 각각의 namespace에 컨테이너 image를 지정하여 Gateway Deployment, Watcher Deployment를 생성한다. Deployment를 생성하면 

> Deployment: 애플리케이션 인스턴스들의 생성과 업데이트를 지원하는 pod(하나 이상의 컨테이너들의 묶음)

```
$ kubectl run fxgateway --image=10.0.0.180:5000/fxgateway:0.1.0 --namespace=openfx
$ kubectl run sqrt-go --image=10.0.0.180:5000/sqrt-go:0.1.0 --namespace=openfx-fn
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

```
$ kubectl get deployment --namespace=openfx-fn
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
sqrt-go   1         1         1            1           1h

$ kubectl get pods --namespace=openfx-fn
NAME                       READY   STATUS    RESTARTS   AGE
sqrt-go-5fcf7966b5-t9x96   1/1     Running   0          1h
```

- Deployment가 생성되고 Pod이 Running 상태인지까지 확인하면 다음과 같이 실행중인 컨테이너를 확인하고 API 단위 응용 기술이 배포되었는지 확인한다.

```
$ sudo docker ps
$ sudo docker logs ${CONTAINER_ID}
kubernetest host: https://10.96.0.1:443
[fxgateway] service start
```
