# (CI & K3S 배포)

<aside>
💡

제목 : 개발 한 소스를 기반으로 CI 구성하고 K3S에 배포 해보기

- Rest API 까지 개발 한 소스에 swagger 기능 추가하여  GitHub 에 프로젝트 생성을 하고 Push 한다. ( swagger는 3.x 버전 사용 )
    - swagger는 Actuator 와 서비스 로직 부분 분리.
- GitHub Action으로 CI 진행한다
    - Docker file 생성 ( Maven 빌드와 도커 이미지 생성  부분 분리 )
        
        - 도커 이미지 생성시에는 JDK 17 Temurin 버전 사용
        
        - Multi Platform 지원
        
    - Github 에 도커 이미지 생성
- k3s 해당 이미지 배포 .
    - swagger 접속 가능하도록 node port 오픈
    - swagger ui 에서 비지니스 서비스 테스트
</aside>

### 1. 소스에 Swagger 추가 및 Push(Swagger 3.x 버전 사용)

소스 URL : https://github.com/kt-cloudnative/springboot_jpa_2

- Swagger 추가하기
    
    ```
    <dependency>
    			<groupId>org.springdoc</groupId>
    			<artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    			<version>2.6.0</version>
    </dependency>
    ```
    
    **추가 후 발생되는 라이브러리 취약점**
    
    ![image.png](%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A6(CI%20&%20K3S%20%E1%84%87%E1%85%A2%E1%84%91%E1%85%A9)%2012007f6b54aa80de86ded866f61b561d/image.png)
    
    ![image.png](%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A6(CI%20&%20K3S%20%E1%84%87%E1%85%A2%E1%84%91%E1%85%A9)%2012007f6b54aa80de86ded866f61b561d/image%201.png)
    
    **IntelliJ에서 알려준대로 11.0.0-m21버전으로 업그레이드 진행해야하는데, 어떻게 할 수 있지?**
    
    ![image.png](%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A6(CI%20&%20K3S%20%E1%84%87%E1%85%A2%E1%84%91%E1%85%A9)%2012007f6b54aa80de86ded866f61b561d/image%202.png)
    
    ⇒ [http://localhost:8080/swagger-ui/index.html](http://localhost:8080/swagger-ui/index.html) 로 접속하기.
    
    Actuator(모니터링 및 관리 도구)와 , 서비스로직(Mustache를 리턴하는)은 분리되어 있는 것을 확인할 수 있음 
    

### 2. Github Action으로 CI 진행하기

- DockerFile 생성하기( Maven 빌드와 도커 이미지 생성  부분 분리 )
    - Maven 빌드 후 도커 이미지로 생성하기.
        
        ```
        #멀티스테이지 빌드
        FROM maven:3.8.5-openjdk-17 AS builder
        
        # 작업 디렉토리 설정
        WORKDIR /app
        
        # pom.xml과 소스 파일을 복사
        COPY pom.xml .
        COPY src ./src
        
        # Maven 빌드 실행
        RUN mvn clean package -DskipTests
        
        # 최종 이미지 생성 단계
        FROM eclipse-temurin:17-jdk AS final
        
        # 작업 디렉토리 설정
        WORKDIR /app
        
        # 빌드된 jar 파일을 복사
        COPY --from=builder /app/target/*.jar app.jar
        
        # 애플리케이션 실행
        ENTRYPOINT ["java", "-jar", "app.jar"]
        
        ```
        
        - 도커 이미지 빌드 후 실행 확인
            
            ![image.png](%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A6(CI%20&%20K3S%20%E1%84%87%E1%85%A2%E1%84%91%E1%85%A9)%2012007f6b54aa80de86ded866f61b561d/image%203.png)
            
        
- Github에 도커 이미지 올리기
    
    Github Action WorkFlow
    
    ```
    name: Publish Docker GitHub image
    
    # This workflow uses actions that are not certified by GitHub.
    # They are provided by a third-party and are governed by
    # separate terms of service, privacy policy, and support
    # documentation.
    
    on:      
      workflow_dispatch:
        inputs:
          name:
            description: "Docker TAG"
            required: true
            default: "master"
    
    env:
      REGISTRY: ghcr.io
      # github.repository as <account>/<repo>
      IMAGE_NAME: ${{ github.repository }}
    
    jobs:
      build:
    
        runs-on: ubuntu-latest
        permissions:
          contents: read
          packages: write
          id-token: write
    
        steps:
          - name: Checkout repository
            uses: actions/checkout@v3
    
          # Install the cosign tool except on PR
          # https://github.com/sigstore/cosign-installer
          - name: Install cosign
            if: github.event_name != 'pull_request'
            uses: sigstore/cosign-installer@v3.7.0 #sigstore/cosign-installer@d6a3abf1bdea83574e28d40543793018b6035605
            #with:
            #  cosign-release: 'v1.7.1'
    
          # Workaround: https://github.com/docker/build-push-action/issues/461
          - name: Setup Docker buildx
            uses: docker/setup-buildx-action@79abd3f86f79a9d68a23c75a09a9a85889262adf
    
          # Login against a Docker registry except on PR
          # https://github.com/docker/login-action
          - name: Log into registry ${{ env.REGISTRY }}
            if: github.event_name != 'pull_request'
            uses: docker/login-action@28218f9b04b4f3f62068d7b6ce6ca5b26e35336c
            with:
              registry: ${{ env.REGISTRY }}
              username: ${{ github.actor }}
              password: ${{ secrets.GITHUB_TOKEN }}
    
          # Extract metadata (tags, labels) for Docker
          # https://github.com/docker/metadata-action
          - name: Extract Docker metadata
            id: meta
            uses: docker/metadata-action@98669ae865ea3cffbcbaa878cf57c20bbf1c6c38
            with:
              images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
              tags: ${{ github.event.inputs.name }}
    
          # Build and push Docker image with Buildx (don't push on PR)
          # https://github.com/docker/build-push-action
          - name: Build and push Docker image
            id: build-and-push
            uses: docker/build-push-action@ac9327eae2b366085ac7f6a2d02df8aa8ead720a
            with:
              context: .
              push: ${{ github.event_name != 'pull_request' }}
              tags: ${{ steps.meta.outputs.tags }}
              labels: ${{ steps.meta.outputs.labels }}
    ```
    
    ![image.png](%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A6(CI%20&%20K3S%20%E1%84%87%E1%85%A2%E1%84%91%E1%85%A9)%2012007f6b54aa80de86ded866f61b561d/image%204.png)
    
    ![image.png](%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A6(CI%20&%20K3S%20%E1%84%87%E1%85%A2%E1%84%91%E1%85%A9)%2012007f6b54aa80de86ded866f61b561d/image%205.png)
    

### K3S에 해당 이미지 배포

이미지를 k3s 클러스터에 배포하기 위한 Deployment 파일 작성

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ciandk3s
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ciandk3s
  template:
    metadata:
      labels:
        app: ciandk3s
    spec:
      containers:
      - name: ciandk3
        image: ghcr.io/hwan-koo/ciandk3s:master
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
```

배포된 파드가 외부에서 접속할 수 있도록 NodePort를 정의하기 위해 service.yaml 배포
또는 deployment expose 해주기

```
apiVersion: v1
kind: Service
metadata:
  name: ciandk3s
spec:
  type: NodePort
  selector:
    app: ciandk3s
  ports:
  - port: 8080

```

pod에서 스프링 애플리케이션 실행 확인이 되고 파드 내부에서도 정상 작동하나, 외부에서 통신이 안되고 있음.

![image.png](%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A6(CI%20&%20K3S%20%E1%84%87%E1%85%A2%E1%84%91%E1%85%A9)%2012007f6b54aa80de86ded866f61b561d/image%206.png)

⇒ 포트 포워딩을 통해 로컬호스트 접근 진행

![image.png](%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A6(CI%20&%20K3S%20%E1%84%87%E1%85%A2%E1%84%91%E1%85%A9)%2012007f6b54aa80de86ded866f61b561d/image%207.png)

**외부에서 Node와 통신할 수 있게 하려면 어떻게 해야하나?(NetworkPolicy 없음)**

![image.png](%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A6(CI%20&%20K3S%20%E1%84%87%E1%85%A2%E1%84%91%E1%85%A9)%2012007f6b54aa80de86ded866f61b561d/image%208.png)

![image.png](%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A6(CI%20&%20K3S%20%E1%84%87%E1%85%A2%E1%84%91%E1%85%A9)%2012007f6b54aa80de86ded866f61b561d/image%209.png)

⇒ curl 192.168.5.15:30861 안됌 .(timeout)

- RancherDesktop Setting 변경 → Administrative Access 허용

![image.png](%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A6(CI%20&%20K3S%20%E1%84%87%E1%85%A2%E1%84%91%E1%85%A9)%2012007f6b54aa80de86ded866f61b561d/image%2010.png)

⇒ 이제 외부에서도 접속이 가능해지고, Externernal IP도 생기게 되었다.
