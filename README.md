# DCF - CLI 

[DCF](https://github.com/DigitalCompanion-KETI/DCFramwork)와 함께 사용될 CLI이다. 

CLI를 통해  지원되는 Runtime으로 function을 빌드하고 배포할 수 있다. 즉, 단순히 handler 파일을 작성하고, CLI는 이를 통해 Docker Image를 생성한다. 

## Get started: Install the CLI 

```
$ wget https://github.com/DigitalCompanion-KETI/DCFramework/releases/download/v0.1.0/dcf
$ chmod +x dcf
$ cp dcf /usr/bin
```

## Run the CLI 

> ### CLI 실행 순서 

> - ### Function 생성 

1. __지원되는 Runtime의 목록을 보여준다. Runtime이란 CLI에서 지원해주는 프로그래밍 언어이다.__
  
    ```
    $ dcf runtime list
    ```

2. __echo-service라는 function을 생성한다. function을 생성할 때 원하는 runtime을 flag를 통해 지정할 수 있다. 생성이 완료되면 현재 디렉토리에 config.yaml 파일과 echo-service라는 폴더가 만들어지고, 폴더 안에는 handler.go 파일과 Dockerfile이 생성된다.__

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

3. __Dockerfile을 통해 Image를 생성하고 repository에 push 및 deploy__
  
    ```
    $ dcf function create -f config.yaml --replace=false --update=true
    ``` 

> - ### Function 호출 

  ```
  $ dcf function call echo-service -g localhost:32222 
  ```  

> - ### Function 확인 
  
- 생성된 function이 Ready 상태인지 확인

  ```
  $ dcf function list 
  ``` 

- 생성된 function의 정보

  ```
  $ dcf function info echo-service -g localhost:32222 
  ```

> - ### Function 삭제 

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

- Go 1.6 이상의 버전이 필요하다. 
- [GOPATH](https://golang.org/doc/code.html#GOPATH)가 설정되어야 한다. 

### Install gRPC 

```
$ go get –u google.golang.org/grpc 
$ go get -u golang.org/x/net 
```

### Install Protocol Buffers 

- [here](https://github.com/google/protobuf/releases)에서 버전과 플랫폼에 맞는 압축 파일의 링크를 복사하여 파일을 다운 받는다. 

```
$ wget https://github.com/protocolbuffers/protobuf/releases/download/v3.6.1/protoc-3.6.1-linux-x86_64.zip 
$ unzip protoc-3.6.1-linux-x86_64.zip -d protoc3 
$ sudo mv protoc3/bin/* /usr/local/bin/ 
$ export PATH=$PATH:/usr/local/bin 
$ sudo mv protoc3/include/* /usr/local/include/ 
```

### Install Protoc plugin 

```
$ go get -u github.com/golang/protobuf/{proto,protoc-gen-go} 
$ export PATH=$PATH:$GOPATH/bin 
```

## gRPC 통신 

### 1. Define gRPC service 

  - 파일의 확장자는 [.proto]이고 Service의 이름은 proto 파일의 이름과 같다. 

```
$ mkdir -p ${GOPATH}/src/pb
$ vim {GOPATH}/src/pb/gateway.proto
```

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

### 2. Generate gRPC service 

``` 
$ protoc -I . \ 
  -I${GOPATH}/src/pb \ 
  --go_out=plugins=grpc:. \ 
  ${GOPATH}/src/pb/gateway.proto 
```

- 컴파일을 완료하면 컴파일할 때의 설정값인 ${GOPATH}/src/pb 경로에 gateway.pb.go 파일이 생성된다. 

### 3. Create gRPC client 

> ### For Beginning

- 구현된 서비스 메소드를 호출하기 위해, 서버와 통신할 수 있는 gRPC 채널을 만든다

- 채널이 구축되면 RPC를 수행할 클라이언트 Stub을 만든다

- 서비스 메소드를 호출한다

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/digitalcompanion-keti/dcf/pb"
	"google.golang.org/grpc"
)

func main() {
    // Creating a gRPC Channel
    address := "localhost:32222" 
  
    conn, err := grpc.Dial(address, grpc.WithInsecure()) 
    if err != nil { 
	  log.Fatalf("did not connect: %v", err) 
    }

    defer conn.Close()

    // Creating a stub
    client := pb.NewGatewayClient(conn)

    // Calling service method
    r, err := client.Invoke(ctx, &pb.InvokeServiceRequest{Service: "echo", Input: []byte("hello world")}) 
    if err != nil { 
	    log.Fatalf("could not invoke: %v\n", err) 
    } 

    fmt.Println(r.Msg)   
}
```

## gRPC for Python 

### PREREQUISITES 

- ### Install pip (version 9.0.1 ~) 

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
 
```
$ python -m pip install grpcio 
```

* OS X EL Capitan의 운영체제일 경우, 다음과 같은 오류가 발생한다. 

  $ OSError: [Errno 1] Operation not permitted: '/tmp/pip-qwTLbI-uninstall/System/Library/Frameworks/Python.framework/Versions/2.7/Extras/lib/python/six-1.4.1-py2.7.egg-info' 
  
* 이 오류는 다음과 같이 해결할 수 있다. 
  
```
$ python -m pip install grpcio --ignore-installed 
```
  
### Install gRPC tools 

* Python gRPC tool은 프로토콜 버퍼 컴파일러인 protoc와 proto 파일로부터 서버 / 클라이언트 코드를 생성하는 특별한 플러그인이 포함되어있다. 때문에 Golang gRPC에서처럼 따로 프로토콜 버퍼를 설치할 필요가 없다. 
  
```
$ python -m pip install grpcio-tools googleapis-common-protos 
```

## gRPC 통신 

### 1. Define gRPC service 

* service를 정의한 proto 파일은 앞서 `gRPC for Go`에서 정의한 파일과 동일하다. 

### 2. Generate gRPC service 

```
$ python -m grpc_tools.protoc -I${GOPATH}/src/pb --python_out=. --grpc_python_out=. ${GOPATH}/src/pb/gateway.proto 
```

* 다음의 오류가 발생할 수 있다.

```
Traceback (most recent call last):
  File "/usr/lib/python2.7/runpy.py", line 174, in _run_module_as_main
    "__main__", fname, loader, pkg_name)
  File "/usr/lib/python2.7/runpy.py", line 72, in _run_code
      exec code in run_globals
  File "/usr/local/lib/python2.7/dist-packages/grpc_tools/protoc.py", line 36, in <module>
    sys.exit(main(sys.argv + ['-I{}'.format(proto_include)]))
  File "/usr/local/lib/python2.7/dist-packages/grpc_tools/protoc.py", line 30, in main
    command_arguments = [argument.encode() for argument in command_arguments]
UnicodeDecodeError: 'ascii' codec can't decode byte 0xe2 in position 0: ordinal not in range(128)	
```

* 이 오류는 다음과 같이 해결할 수 있다. 

```
$ vi /usr/local/lib/python2.7/dist-packages/grpc_tools/protoc.py

import sys
reload(sys)
sys.setdefaultencoding('utf-8')
```

### 3. Create gRPC client 

> ### For Beginning

- 서비스 메소드를 호출하기 위한 Stub을 만든다

- 서비스 메소드를 호출한다.  

```python
import grpc

import gateway_pb2
import gateway_pb2_grpc

def run():
    with grpc.insecure_channel('localhost:32222') as channel:
        # Creating a stub
        stub = gateway_pb2_grpc.GatewayStub(channel)
        
        # Calling service methods
        r = stub.Invoke(servicerequest)
        servicerequest = gateway_pb2.Invoke(Service=”echo”, Input=”hello world”.encode())
        print(r.Msg)

if __name__ == '__main__':
	run()
```

## gRPC for C++

### PREREQUISITES

```
$ apt-get install build-essential autoconf libtool pkg-config
$ apt-get install libgflags-dev libgtest-dev
$ apt-get install clang libc++-dev
```

### Install gRPC

```
$ git clone -b v1.15.0 https://github.com/grpc/grpc
```

### Install Protocal Buffers

- 만약 'protoc' 컴파일러가 설치되어있지 않다면 다음과 같이 설치해준다. 'protoc'는 기본적으로 /usr/local/bin에 설치된다.

```
$ cd grpc/third_party
$ git clone https://github.com/google/protobuf.git
$ cd protobuf
$ git submodule update --init –recursive
$ ./autogen.sh
$ ./configure
$ make
$ make check
$ make install
$ ldconfig
```

### Install Protoc plugin

- grpc repository의 root에서 설치한다. 

```
$ cd ../../
$ git submodule update --init
$ make
$ make install
```

- make 명령어 실행 중 다음과 같은 오류가 발생할 수 있다.

```
error: ‘dynamic_init_dummy_src_2fproto_2fgrpc_2fstatus_2fstatus_2eproto’ defined but not used [-Werror=unused-variable]

static bool dynamic_init_dummy_src_2fproto_2fgrpc_2fstatus_2fstatus_2eproto = []() { AddDescriptors_src_2fproto_2fgrpc_2fstatus_2fstatus_2eproto(); return true; }();
```

- Solution:

```
static bool dynamic_init_dummy_src_2fproto_2fgrpc_2fstatus_2fstatus_2eproto __attribute__((unused)) = []() { AddDescriptors_src_2fproto_2fgrpc_2fstatus_2fstatus_2eproto(); return true; }();
```

## gRPC 통신

### 1. Generate gRPC service

- make 명령어를 사용하기 위해 Makefile을 만들어준다. PROTOS_PATH는 자신의 proto 파일의 경로를 적어준다.

```
$ vim ${GOPATH}/src/pb/Makefile
```

```
#
# Copyright 2015 gRPC authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

HOST_SYSTEM = $(shell uname | cut -f 1 -d_)
SYSTEM ?= $(HOST_SYSTEM)
CXX = g++
CPPFLAGS += `pkg-config --cflags protobuf grpc`
CXXFLAGS += -std=c++11
ifeq ($(SYSTEM),Darwin)
LDFLAGS += -L/usr/local/lib `pkg-config --libs protobuf grpc++`\
           -lgrpc++_reflection\
           -ldl
else
LDFLAGS += -L/usr/local/lib `pkg-config --libs protobuf grpc++`\
           -Wl,--no-as-needed -lgrpc++_reflection -Wl,--as-needed\
           -ldl
endif
PROTOC = protoc
GRPC_CPP_PLUGIN = grpc_cpp_plugin
GRPC_CPP_PLUGIN_PATH ?= `which $(GRPC_CPP_PLUGIN)`
											
PROTOS_PATH = ${GOPATH}/src/pb
												
vpath %.proto $(PROTOS_PATH)
												
all: system-check gateway_client gateway_server
												
gateway_client: gateway.pb.o gateway.pb.o gateway.o helper.o
    $(CXX) $^ $(LDFLAGS) -o $@
											
gateway_server: gateway.pb.o gateway.grpc.pb.o gateway_server.o helper.o
    $(CXX) $^ $(LDFLAGS) -o $@
														
%.grpc.pb.cc: %.proto
    $(PROTOC) -I $(PROTOS_PATH) --grpc_out=. --plugin=protoc-gen-grpc=$(GRPC_CPP_PLUGIN_PATH) $<
															
%.pb.cc: %.proto
    $(PROTOC) -I $(PROTOS_PATH) --cpp_out=. $<
																
clean:
    rm -f *.o *.pb.cc *.pb.h gateway_client gateway_server 

# The following is to test your system and ensure a smoother experience.
# They are by no means necessary to actually compile a grpc-enabled software.

PROTOC_CMD = which $(PROTOC)
PROTOC_CHECK_CMD = $(PROTOC) --version | grep -q libprotoc.3
PLUGIN_CHECK_CMD = which $(GRPC_CPP_PLUGIN)
HAS_PROTOC = $(shell $(PROTOC_CMD) > /dev/null && echo true || echo false)
ifeq ($(HAS_PROTOC),true)
HAS_VALID_PROTOC = $(shell $(PROTOC_CHECK_CMD) 2> /dev/null && echo true || echo false)
endif
HAS_PLUGIN = $(shell $(PLUGIN_CHECK_CMD) > /dev/null && echo true || echo false)

SYSTEM_OK = false
ifeq ($(HAS_VALID_PROTOC),true)
ifeq ($(HAS_PLUGIN),true)
SYSTEM_OK = true
endif
endif

system-check:
ifneq ($(HAS_VALID_PROTOC),true)
    @echo " DEPENDENCY ERROR"
    @echo
    @echo "You don't have protoc 3.0.0 installed in your path."
    @echo "Please install Google protocol buffers 3.0.0 and its compiler."
    @echo "You can find it here:"
    @echo
    @echo "   https://github.com/google/protobuf/releases/tag/v3.0.0"
    @echo
    @echo "Here is what I get when trying to evaluate your version of protoc:"
    @echo
    -$(PROTOC) --version
    @echo
    @echo
endif
ifneq ($(HAS_PLUGIN),true)
    @echo " DEPENDENCY ERROR"
    @echo
    @echo "You don't have the grpc c++ protobuf plugin installed in your path."
    @echo "Please install grpc. You can find it here:"
    @echo
    @echo "   https://github.com/grpc/grpc"
    @echo
    @echo "Here is what I get when trying to detect if you have the plugin:"
    @echo
    -which $(GRPC_CPP_PLUGIN)
    @echo
    @echo
endif
ifneq ($(SYSTEM_OK),true)
    @false
endif
```

### 2. Create gRPC client

> ### For Beginning

- 서비스 메소드를 호출하기 위한 Stub을 만든다

- 서비스 메소드를 호출한다.

```cpp
#include <iostream>
#include <memory>
#include <string>

#include <grpc/grpc.h>

#include "gateway.grpc.pb.h"

using grpc::Channel;
using grpc::ClientAsyncResponseReader;
using grpc::ClientContext;
using grpc::CompletionQueue;
using grpc::Status;
using pb::Message;
using pb::InvokeServiceRequest;
using pb::Gateway;

class GatewayClient {
 public:
  GatewayClient(std::shared_ptr<Channel> channel)
      : stub_(Gateway::NewStub(channel)) {}

  std::string Invoke(const std::string& service) {
    ClientContext context;
    Status status;
    CompletionQueue cq;

    InvokeServiceRequest request;
    request.set_Service(service);
    request.set_Input();
    Message msg;

    std::unique_ptr<ClientAsyncResponseReader<Message> > rpc(
        stub_->PrepareAsyncInvokeRaw(&context, request, &cq));

    if (status.ok()) {
      return msg.message();   
    } 
  

 private:
  std::unique_ptr<Gateway::Stub> stub_;
};

int main(int argc, char** argv) {
  GatewayClient gateway(
      grpc::CreateChannel("localhost:32222",
                         grpc::InsecureChannelCredentials()));
  std::string service("echo-service");
  std::
  std::string Msg = gateway.Invoke(service, );
  std::cout << Msg << std:endl;
  return 0;
}
```

