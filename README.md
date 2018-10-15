# DCF - CLI 

[DCF](https://github.com/DigitalCompanion-KETI/DCFramwork)와 함께 사용될 CLI이다. 

CLI를 통해  지원되는 Runtime으로 function을 빌드하고 배포할 수 있다. 즉, 단순히 handler 파일을 작성하고, CLI는 이를 통해 Docker Image를 생성한다. 

## Get started: Install the CLI 

```
$ wget https://github.com/DigitalCompanion-KETI/DCFramework/releases/download/v0.1.0/dcf
$ chmod +x dcf
```

## Run the CLI 

> ### CLI 실행 순서 

> - ### Function 생성 

1. __지원되는 Runtime의 목록을 보여준다.__ 

- Runtime: DCF에서 지원해주는 프로그래밍 언어
  
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


