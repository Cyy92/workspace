# DCF - CLI 

[DCF](https://github.com/DigitalCompanion-KETI/DCFramwork)와 함께 사용될 CLI이다. 

CLI를 통해  지원되는 Runtime으로 function을 빌드하고 배포할 수 있다. 즉, 단순히 handler 파일을 작성하고, CLI는 이를 통해 Docker Image를 생성한다. 

## Get started: Install the CLI 

## Run the CLI 

> ### CLI 실행 순서 

> - ### Function 생성 

1. 지원되는 Runtime의 목록을 보여준다. Runtime이란 CLI에서 지원해주는 프로그래밍 언어이다.
  
    ```
    $ dcf runtime list
    ```

2. echo-service라는 function을 생성한다. function을 생성할 때 원하는 runtime을 flag를 통해 지정할 수 있다. 생성이 완료되면 현재 디렉토리에 config.yaml 파일과 echo-service라는 폴더가 만들어지고, 폴더 안에는 handler.go 파일과 Dockerfile이 생성된다.

    ```
    $ dcf function init echo-service -r go
    ```
    config.yaml : 사용자가 function을 정의하고자 할 때 규격이 되는 파일, 여러 개의 function을 정의할 수 있다.

    ```
      funtions:
        echo-service: // function 이름
	      runtime: //function runtime 정의
	      desc: ""
	      maintainer: ""
	      handler: // function이 실행되는 main 함수
		    dir: ./echo-service // handler.go 파일이 저장된 경로
			file: ""
			name: Handler // main 함수 이름
			image: dcf-repository:5000/echo-service
      dcf:
        gateway: localhost:32222
    ```

3. Dockerfile을 통해 Image를 생성하고 repository에 push 및 deploy
  
    ```
    $ dcf function create -f config.yaml --replace=false --update=true
    ``` 

> - ### Function 호출 

  ```
  $ dcf function call echo-service -g localhost:32222 
  ```  

> - ### Function 확인 

  생성된 function의 목록을 보여준다. 
  
  ```
  $ dcf function list 
  ``` 

  생성된 function에 대한 정보를 확인한다. 

  ```
  $ dcf function info echo-service -f config.yaml -g localhost:32222 
  ```

> - ### Function 삭제 

  생성된 function을 삭제한다. 

  ```
  $ dcf function delete -f config.yaml 
  ```
  
CLI 명령어에 대한 도움말은 다음의 Flag를 통해 실행할 수 있다. 

```
$ dcf function -h 
$ dcf function init -h 
$ dcf runtime -h 
$ dcf runtime list -h 
```

# gRPC Guide 

이 가이드는 gRPC 설치법 및 사용법에 대한 가이드이다. 이에 앞서, RPC란 네트워크 상 원격에 있는 서버의 서비스를 호출하는데 사용되는 프로토콜로 IDL(Interface Definition Language)로 인터페이스를 정의한 후 이에 해당하는 Skeleton과 Stub 코드, 즉 해당 프로그래밍 언어가 부를 수 있는 형태의 코드를 통해 프로그래밍 언어에서 호출해서 사용하는 방식이다. gRPC란 자바, C/C++ 뿐만 아니라 Python, Ruby, Go 등 다양한 언어들을 지원함으로써 서버 간 뿐만 아니라 클라이언트 어플리케이션이나 모바일 앱에서도 사용 가능한 RPC 프레임워크이다. 

## gRPC for Go 

### PREREQUISITES 
----------------- 

- Go 1.6 이상의 버전이 필요하다. 
- [GOPATH](https://golang.org/doc/code.html#GOPATH)가 설정되어야 한다. 

### Install gRPC 
---------------- 

```
$ go get –u google.golang.org/grpc 
$ go get -u golang.org/x/net 
```

### Install Protocol Buffers 
---------------------------- 

- [here](https://github.com/google/protobuf/releases)에서 버전과 플랫폼에 맞는 압축 파일의 링크를 복사하여 파일을 다운 받는다. 

```
$ wget https://github.com/protocolbuffers/protobuf/releases/download/v3.6.1/protoc-3.6.1-linux-x86_64.zip 
$ unzip protoc-3.6.1-linux-x86_64.zip -d protoc3 
$ sudo mv protoc3/bin/* /usr/local/bin/ 
$ export PATH=$PATH:/usr/local/bin 
$ sudo mv protoc3/include/* /usr/local/include/ 
```

### Install Protoc plugin 
------------------------- 

```
$ go get -u github.com/golang/protobuf/{proto,protoc-gen-go} 
$ export PATH=$PATH:$GOPATH/bin 
```

### Communicate with gateway 
---------------------------- 

1. Define gRPC service 

  - 파일의 확장자는 [.proto]이고 Service의 이름은 proto 파일의 이름과 같다. 

```
syntax = "proto3"; 
package pb; 
service gateway { 
	rpc Deploy(CreateFunctionRequest) returns (Message) {} 
	rpc Delete(CreateFunctionRequest) returns (Message) {} 
	rpc Update(CreateFunctionRequest) returns (Message) {} 
	// There are more rpc methods. 
} 

// Define message for service 
message Message { 
	string Msg = 1; 
} 
  
message CreateFunctionRequest { 
  	string Service = 1; 
	string Image = 2; 
	map<string, string> EnvVars = 3; 
	map<string, string> Labels = 4; 
	repeated string Secrets= 5; 
	FunctionResources Limits = 6; 
    // There are more variables that user can define 
} 

message FunctionResources { 
	string Memory = 1; 
	string CPU = 2; 
	string GPU = 3; 
} 

// There are more message that user can define 
```

2. Generate gRPC service 

```
$ mkdir –p {GOPATH}/src/pb 
$ protoc -I . \ 
                -I${GOPATH}/src/pb \ 
				        --go_out=plugins=grpc:. \ 
						        ${GOPATH}/src/pb/gateway.proto 
```

- 컴파일을 완료하면 컴파일할 때의 설정값인 ${GOPATH}/src/pb 경로에 gateway.pb.go 파일이 생성된다. 

3. Create gRPC client 

- Client 생성 관련 예제 코드는 [grpc-go/client](https://github.com/grpc/grpc-go/tree/master/examples/route_guide/client)에서 확인할 수 있다.

- Creating a stub 
  
  구현된 서비스 메소드를 호출하기 위해, 서버와 통신할 수 있는 gRPC 채널을 만들어야 한다. 이는 서버 주소와 포트 번호, 필요에 따라 인증 자격 요청을 설정하기 위한 옵션을 인자로 갖는 grpc.Dial() 메소드를 통해 만들 수 있다.
    
  ```go
  address := "localhost:32222" 
  
  conn, err := grpc.Dial(address, grpc.WithInsecure()) 
  if err != nil { 
	log.Fatalf("did not connect: %v", err) 
  }

  defer conn.Close() 
  ```

  gRPC 채널이 구축되면 RPC를 수행할 클라이언트 Stub이 필요하다. 이는 gateway.pb.go 파일에 생성된 NewGatewayClient 메소드를 통해 만들 수 있다.

  ```go
  client := pb.NewGatewayClient(conn) 
  ```

- Calling service methods 
  
  Stub까지 생성이 완료되면, 클라이언트 측에서 서비스 메소드를 호출한다. 

  ```go
  r, err := client.Invoke(ctx, &pb.InvokeServiceRequest{Service: "echo", Input: []byte("hello world")}) 

  if err != nil { 
	  log.Fatalf("could not invoke: %v\n", err) 
  } 

  fmt.Println(r.Msg) 
  ```

## gRPC for Python 

### PREREQUISITES 
----------------- 

- Install pip (version 9.0.1 ~) 

  gRPC Python은 Python 2.7 또는 Python 3.4 이상부터 지원 가능하다. 
  
  ```
  $ python -m pip install --upgrade pip 
  ```

  만약 root 계정이 아닌 경우, 다음과 같이 pip을 업그레이드 할 수 있다. 

  ```
  $ python -m pip install virtualenv 
  $ virtualenv venv 
  $ source venv/bin/activate 
  $ python -m pip install --upgrade pip 
  ```

### Install gRPC 
---------------- 

```
$ python -m pip install grpcio 
```

* OS X EL Capitan의 운영체제일 경우, 다음과 같은 오류가 발생한다. 

OSError: [Errno 1] Operation not permitted: 
'/tmp/pip-qwTLbI-uninstall/System/Library/Frameworks/Python.framework/Versions/2.7/Extras/lib/python/six-1.4.1-py2.7.egg-info' 
  
* 이 오류는 다음과 같이 해결할 수 있다. 
  
```
$ python -m pip install grpcio --ignore-installed 
```
  
### Install gRPC tools 
---------------------- 

* Python gRPC tool은 프로토콜 버퍼 컴파일러인 protoc와 proto 파일로부터 서버 / 클라이언트 코드를 생성하는 특별한 플러그인이 포함되어있다. 때문에 Golang gRPC에서처럼 따로 프로토콜 버퍼를 설치할 필요가 없다. 
  
```
$ python -m pip install grpcio-tools googleapis-common-protos 
```

### Communicate with gateway 
---------------------------- 

1. Define gRPC service 

* service를 정의한 proto 파일은 앞서 `gRPC for Go`에서 정의한 파일과 동일하다. 

2. Generate gRPC service 

```
$ mkdir –p {GOPATH}/src/pb 
$ python -m grpc_tools.protoc –I${GOPATH}/src/pb \ 
   --python_out=. --grpc_python_out=. ${GOPATH}/src/pb/gateway.proto 
```

3. Create gRPC client 

- Client 생성 관련 예제 코드는 [grpc-python/client](https://github.com/grpc/grpc/tree/master/examples/python/route_guide)에서 확인할 수 있다. 

- Creating a stub 

  * 서비스 메소드를 호출하기 위해 Stub을 먼저 생성해야한다. 


  ```python
  channel = grpc.insecure_channel1(`*serverAddr`) 

  stub = gateway_pb2_grpc.GatewayStub(channel) 
  ```

- Calling service methods 

  * Stub까지 생성이 완료되면 클라이언트 측에서 서비스 메소드를 호출한다. 


  ```python
  r = stub.Invoke(servicerequest) 

  servicerequest = gateway_pb2.Invoke(Service=”echo”, Input=”hello world”.encode()) 

  print(r.Msg) 
  ```

