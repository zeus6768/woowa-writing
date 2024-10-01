# 도커 컴포즈 살펴보기: 쉘스크립트에서 도커 컴포즈로 배포 프로세스를 전환하며

안녕하세요. 우아한테크코스 BE 6기 제우스입니다. [코드잽](https://code-zap.com)([Github](https://github.com/zeus6768/2024-code-zap))의 백엔드를 맡아 개발하고 있습니다. 데모데이마다 요구사항이 주어지며 프로젝트의 인프라 구조가 변화했습니다. 그중 배포 프로세스에서 리눅스 쉘 스크립트를 사용하다가 도커 컴포즈로 전환한 과정에 대해 이야기해보려 합니다.

데모데이의 요구사항은 아래와 같이 변화했습니다. 

> ### 2차: 개발(dev) 환경 구축
> - 개발 서버에 서비스 띄우기
> - CI와 쉘 스크립트 등을 활용한 배포 자동화
> ### 3차: 서비스 운영 환경 구축
> - 로깅 프레임워크 적용 등
> ### 4차: 프로덕션(prod) 환경 구축
> - 실 서버 도메인 연결, HTTPS 적용 등
> ### 5차: 서비스 가용성 개선
> - AWS AZ 분리 및 로드밸런서 적용 등

한 달 사이에 운영 서버를 추가로 배포해야 했고, 5차 데모데이에는 EC2 가용 영역을 두 개로 분리하는 요구사항도 추가되었습니다. 서로 다른 EC2 인스턴스에서 동일한 설정으로 배포해야 했습니다. 객체지향에서만 유연한 확장이 필요한 건 아니었습니다. 

## 쉘스크립트

개발 서버에 간단한 서비스를 띄우는 일은 간단한 쉘 스크립트로 가능했습니다. 

다음은 레벨2의 백엔드 인프라 수업 당시 주어진 배포 스크립트입니다. 

우분투 환경에서 동작하는 코드입니다.

```shell
# 자바 설치
wget -O- https://apt.corretto.aws/corretto.key | sudo apt-key add - 
sudo add-apt-repository 'deb https://apt.corretto.aws stable main'
sudo apt-get update; sudo apt-get install -y java-17-amazon-corretto-jdk

# 빌드
git clone https://github.com/woowacourse/spring-roomescape-payment.git
cd spring-roomescape-payment
./gradlew bootJar

# 서버 실행
cd build/libs
java -jar spring-roomescape-payment-0.0.1-SNAPSHOT.jar
nohup java -jar spring-roomescape-payment-0.0.1-SNAPSHOT.jar &
```

10줄 남짓의 코드로 배포할 수 있기 때문에, 이것도 그렇게 나쁜 방법으로 보이지는 않습니다. 

그러나 이 스크립트에는 아쉬운 점이 있습니다. 하나씩 짚어보겠습니다. 

### 자바 설치 스크립트

```shell
wget -O- https://apt.corretto.aws/corretto.key | sudo apt-key add - 
sudo add-apt-repository 'deb https://apt.corretto.aws stable main'
sudo apt-get update; sudo apt-get install -y java-17-amazon-corretto-jdk
```

먼저 자바 설치를 위한 스크립트는 운영체제마다 다릅니다. 우분투에서는 APT라는 패키지 관리자를 사용하지만, CentOS에서는 YUM을 사용합니다. 또 윈도우 사용자가 본인의 PC에서 비슷한 환경을 만들어 테스트 하고 싶다면 어떨까요? 

게다가, 이 스크립트는 매번 실행될 필요가 없습니다. 자바를 한 번만 설치하면 되니까요. 어떤 스크립트는 한 번만 실행하면 되고, 어떤 스크립트는 배포할 때마다 실행해야 된다는 것을 스크립트만 보고 알기는 어려울 것입니다.  

### 빌드 스크립트

```shell
git clone https://github.com/woowacourse/spring-roomescape-payment.git
cd spring-roomescape-payment
./gradlew bootJar
```

빌드 스크립트에서 깃 저장소를 clone 해서 빌드하고 있는데, CI 등으로 인해 빌드를 위한 서버를 별도로 관리하게 된다면 어떨까요? 해당 스크립트는 빌드된 파일을 다운로드만 하는 내용으로 바뀌어야 할 것입니다. 

실행하는 스크립트에도 아쉬운 점이 있습니다. 만약 `-Dspring.config.location` 옵션으로 설정 파일을 명시해준다면, 새로운 설정 파일이 추가될 때마다 스크립트도 변경해야 합니다. 또 새로 배포한 jar 파일을 실행할 때, 이전에 실행된 프로세스를 종료하고 새로 실행하고 싶으면 쉘 스크립트를 어떻게 변경해야 할까요? 무중단 배포를 하고 싶다면 어떻게 변경해야 할까요? 쉘스크립트를 모국어처럼 쓸 수 있다면 괜찮을지도 모릅니다. 

### 서버 실행 스크립트
```shell
cd build/libs
java -jar spring-roomescape-payment-0.0.1-SNAPSHOT.jar
nohup java -jar spring-roomescape-payment-0.0.1-SNAPSHOT.jar &
```

이렇게 요구사항이 바뀔 때마다 쉘 스크립트를 변경해 배포 프로세스를 재구성하는 일은 상당히 까다롭습니다. 설정 파일의 위치와 내용이 스크립트와 강하게 결합해서, 프로젝트가 발전할수록 관리 포인트가 무한히 늘어나기 때문입니다. 무엇보다 쉘스크립트는 스크립트 언어이기 때문에 스크립트를 작성하는 시점에 오류를 알 수 없습니다.

심지어 우리는 배포 프로세스를 문서화해서 동료들에게 설명해야 하죠. 동료에게 쉘 스크립트로 돌아가는 배포 프로세스를 인수인계하는 일은 꽤나 고될 것 같습니다.  

### 실제 배포에 사용된 스크립트

```shell
#!/usr/bin/env bash

source before-start.sh
source after-start.sh

function __nohup_run() {

	output_log_path="$ARTIFACT_DIR/spring-log/output-$IDLE_PROFILE.log"
	error_log_path="$ARTIFACT_DIR/spring-log/error-$IDLE_PROFILE.log"

	local -r jar_name=$(ls -tr $ARTIFACT_DIR/*.jar | grep "$PROJECT.*\.jar" | tail -n 1)
	local -r config_location="$ARTIFACT_DIR/application-$PROFILE_GROUP.yml,$ARTIFACT_DIR/application-db.yml,$ARTIFACT_DIR/application-swagger.yml"

	echo "> start.sh: Current idle port=$IDLE_PORT"
	echo "> start.sh: New jar name=$jar_name"
	echo "> start.sh: New profile=$IDLE_PROFILE"

	nohup java -jar \
		-Dspring.config.location=$config_location \
		-Dspring.profiles.active=$IDLE_PROFILE \
		-Duser.timezone=Asia/Seoul \
		$jar_name \
		1> $output_log_path \
		2> $error_log_path \
		&
}

function __verify() {
	for count in {1..5}
	do
		local pid=$(lsof -ti ":$IDLE_PORT")
		if [ -n "$pid" ]; then
			echo "> start.sh: New application $pid started."
			echo "> start.sh: Check $output_log_path to check logs."
			return 0
		fi
		sleep $count
	done
	echo "> start.sh: Verification failed."
	echo "> start.sh: Check $ABS_DIR/nohup.out or $output_log_path to check logs."
	exit 1
}

function main() {
	__nohup_run
	__verify
}

main
```

위의 스크립트는 실제 배포에 쓰였던 스크립트의 일부입니다. 첫 번째 함수는 배포된 jar 파일을 설정을 적용해 실행할 뿐인데, 17줄이나 됩니다. 두 번째 함수는 프로세스가 실행중인지 검증합니다. 

위 3개 함수 말고도 13개의 다른 함수가 `before-start.sh`, `after-start.sh`에 import 되어 사용되었습니다. 물론 저보다 실력 좋은 분이 스크립트를 작성했다면 더 간결하고 효율적으로 스크립트를 사용할 수 있었을 것입니다. 다만 팀원 누구나 접근할 수 있어야 하는 스크립트에 그렇게까지 많은 지식이 요구되지 않는 편이 팀의 생산성에 더 도움이 될 것입니다.   

## Nginx

배포를 위해서 쉘스크립트만 필요한 건 아니었습니다. Nginx와 Certbot을 조합해 SSL 인증서를 설정하고 있었습니다. 이를 위해서는 EC2 인스턴스에서 APT를 사용해 Nginx를 직접 설치하고 설정했습니다.

Nginx를 설치하면 설정 파일이 `/etc/nginx`에 위치합니다. 해당 경로는 `$HOME` 경로를 벗어나므로 매번 `sudo` 권한이 필요했습니다. 이것 또한 작업을 번거롭게 만드는 요인이었습니다.

또 nginx를 재실행하거나 설정을 적용하기 위해 아래의 명령어들을 외우거나 매번 검색해서 찾아야 했습니다.

```shell
sudo nginx -t                   # 설정 파일의 문법 검사
sudo nginx -s reload            # 설정 파일 로드
sudo systemctl nginx restart    # 프로세스 재시작
```


## 도커 컴포즈

도커 컴포즈를 사용하면 다음의 설정 파일이 서버 실행과 Nginx 설정 적용을 위한 스크립트를 대체합니다. 

```yml
name: code-zap

services:
  spring:
    image: amazoncorretto:17
    container_name: spring
    expose:
      - 8080
    environment:
      TZ: Asia/Seoul
    volumes:
      - ./spring:/app
    entrypoint: [
      "java", "-jar",
      "-Dspring.config.location=/app/application-prod.yml,/app/application-db/yml,/app/application-actuator.yml,/app/application-swagger.yml",
      "-Dspring.profiles.active=prod,db,actuator,swagger",
      "/app/zap.jar"
    ]

  nginx:
    image: nginx:1.27.1
    container_name: nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    environment:
      TZ: Asia/Seoul
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - /etc/letsencrypt:/etc/letsencrypt
      - /var/log/nginx:/var/log/nginx
  
```

얼핏 봐도 훨씬 간단합니다. 설정을 적용해 실행하고 싶다면 간단한 명령어만 입력하면 됩니다. 서비스명을 입력하지 않으면 모든 서비스가 실행됩니다. 

```shell
docker compose up -d {서비스명}
```

종료하고 싶을 때도 간단합니다. 서비스명을 입력하지 않으면 모든 컨테이너를 종료합니다. 

```shell
docker compose down {서비스명}
```

재실행도 가능합니다. 

```shell
docker compose restart
```

특정 서비스만 실행하거나 재실행하는 것도 물론 가능합니다. 여러 개의 서비스를 실행하고 싶다면 띄어쓰기로 구분합니다. 
```shell
docker compose up -d {서비스명1} {서비스명2} ...
docker compose restart {서비스명1} {서비스명2} ...
```

하나씩 살펴 보겠습니다. 

### name & services

```yml
name: code-zap

services:
  spring:
    ...
  nginx:
    ...
```

실행하려는 컴포즈의 이름은 `code-zap`이고, `spring`과 `nginx`라는 이름의 서비스를 실행한다는 설정입니다. 서비스 이름은 자유롭게 작성합니다. `spring`은 `server`로, `nginx`은 `proxy`라는 이름으로 바꿔도 아무런 문제가 없습니다. 

### image

```yml
image: amazoncorretto:17
```

이 설정은 위에서 봤던 자바 설치 스크립트를 대체합니다. 

```yml
wget -O- https://apt.corretto.aws/corretto.key | sudo apt-key add - 
sudo add-apt-repository 'deb https://apt.corretto.aws stable main'
sudo apt-get update; sudo apt-get install -y java-17-amazon-corretto-jdk
```

3줄에서 1줄로 줄었고, 의미를 파악하기도 쉬워졌습니다. 도커만 설치되어있다면 현재 패키지 관리자가 무엇이든 상관 없게 되었습니다. 이미 이미지를 pull 했다면 저장된 이미지를 사용합니다. 도커 허브에 저장된 이미지를 사용합니다. 

### container_name
```yml
container_name: spring
```

컨테이너의 이름을 명시합니다. 선택 옵션이므로, 설정하지 않으면 서비스명과 같게 자동으로 설정됩니다. 

### expose
```yml
expose:
  - 8080
```

🚨 `ports` 설정과 혼동하지 않게 주의합니다. `expose`는 같은 네트워크에 속한 다른 컨테이너가 해당 옵션이 설정된 컨테이너의 포트로 접근할 수 있게 허용하는 옵션입니다. `ports`는 호스트에서 컨테이너의 포트로 접근할 수 있게 허용하는 설정입니다. 

이것이 가능한 이유는, 컨테이너에 따로 네트워크 설정을 하지 않은 경우 같은 컴포즈에 속한 컨테이너들은 하나의 네트워크에 소속된 것과 같기 때문입니다. 

심지어 컴포즈는 컨테이너명을 호스트명으로 사용해 서로 통신할 수 있도록 자동으로 DNS를 설정합니다. 쉽게 말해, `nginx` 컨테이너에서 `http://spring:8080`으로 접속할 수 있게 됩니다.

### environment

```yml
environment:
  TZ: Asia/Seoul
```

해당 컨테이너에 환경변수를 설정합니다. 컨테이너 안에서 `export TZ="Asia/Seoul"`를 설정한 것과 같습니다.
`env_file` 옵션과 `.env` 파일을 함께 사용하는 방식으로 대체할 수 있습니다.  

### volumes

```yml
volumes:
  - ./spring:/app
```

볼륨 설정이 되지 않은 컨테이너는 종료했을 때 모든 데이터를 잃습니다. 이는 유연함과 효율을 제공하기도 하지만, 어떤 데이터는 영속될 필요가 있습니다. `volumes`는 그런 상황을 위해 필요한 옵션입니다. 

저는 호스트의 `{compose.yml 경로}/spring` 디렉토리에 스프링 서버 실행을 위한 jar 파일과 yml 설정 파일을 위치시켰습니다. 이 파일들은 도커 컨테이너에서 스프링 서버를 실행하기 위해 필요합니다. 그래서 컨테이너 내부의 `/app` 경로와 볼륨 설정을 해주었습니다. 이렇게 하면 컨테이너가 몇 번이고 꺼졌다 켜져도 서버 실행을 위해 필요한 파일들은 그대로 남아있게 됩니다. 

### entrypoint

```yml
    entrypoint: [
      "java", "-jar",
      "-Dspring.config.location=/app/application-prod.yml,/app/application-db/yml,/app/application-actuator.yml,/app/application-swagger.yml",
      "-Dspring.profiles.active=prod,db,actuator,swagger",
      "/app/zap.jar"
    ]
```

`entrypoint` 설정은 컨테이너가 실행되자마자 실행할 명령어를 명시합니다. 위의 설정은 컨테이너 내의 `/app` 디렉토리에 위치한 yml 설정 파일과 함께 jar 파일을 실행합니다. 쉘스크립트보다 가독성이 좋다는 것을 한 눈에 알 수 있습니다. 

```yml
entrypoint: /app/entrypoint.sh
```

이렇게 파일로 분리해서 실행할 수도 있습니다. 

## 결론

쉘 스크립트만을 사용해 배포할 때는 서버를 운영하는 데 필요한 응용프로그램들의 설치 방법이 OS마다 다르고, 변경에 필요한 리소스가 크다는 문제가 있었습니다. 

도커 컴포즈를 사용하면 요구사항이 변경되더라도 OS에 구애받지 않고 유연하게 대처할 수 있었고, 팀원들과 소통하기 위한 비용도 줄어들었습니다. 

이 글을 통해 도커 컴포즈를 왜 사용해야 할지, 어떻게 사용해야 할지 의문을 갖고 있던 분들에게 도움이 되었길 바랍니다. 

## 참고 자료

https://nginx.org/en/docs/

https://docs.docker.com/reference/compose-file/

https://hub.docker.com/
