# PBC & 마이크로서비스 클러스터 배포 Guide

#### 해당 가이드는 PBC와 마이크로서비스를 클라우드 환경에서 배포 진행시 순차적으로 진행하지 않을 경우 특정 서비스를 여러번 패키징, 도커라이징, 클러스터 배포단계를 진행하는 것을 방지하고자 만들어진 가이드로 아래의 나와있는 순서에 맞게 서비스를 배포를 진행한다.

## 1. PBC 클러스터 배포

### 1-1. Backend배포

#### 선택한 PBC (ex.PaymentService, Review&Rating ...) 하위의 backend를 담당하는 마이크로 서비스에서 아래의 명령어로 패키징, 도커라이징, 클러스터 배포를 진행한다.

```
// 패키징
cd [MICROSERVICE_NAME]
mvn package -B -Dmaven.test.skip=true

// 도커라이징
az acr login --name [REGISTRY_NAME] // 최초 1회 진행 후, Timeout발생시 재진행
docker build -t [REGISTRY_NAME].azurecr.io/[MICROSERVICE_NAME]:[오늘날짜] .
docker push [REGISTRY_NAME].azurecr.io/[MICROSERVICE_NAME]:[오늘날짜]

//kubernetes/deployment.yml 19 Line image 스펙 수정

    spec:
      containers:
        - name: payment
          image: "[REGISTRY_NAME].azurecr.io/[MICROSERVICE_NAME]:[오늘날짜]"


// deployment.yaml, service.yaml 클러스터 배포
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```

### 1.2 Frontend 배포

#### 선택한 PBC 하위의 frontend 경로로 이동하여 아래의 명령어를 통해 패키징을 진행한다.
```
// 패키징
cd frontend
npm run build

// node_modules가 없을 경우
cd frontend
nvm install 14.19.0
npm install
npm run build
```

#### 이후, 생성된 build파일의 위치인 dist 하위에 폴더를 생성한 후 dist파일에 생성된 모든 파일을 방금 만든 폴더 하위로 이동한다.
#### 폴더명은 해당 PBC이름 + pbcfe로 구성한다. (ex. paymentpbcfe, reviewpbcfe ..)
![image](https://github.com/user-attachments/assets/53232ec5-5786-41a3-8e9a-0de8aaba5dc9)

#### 폴더 설정이 완료되었다면 아래의 가이드를 참조하여 도커라이징, 클러스터 배포를 진행한다. 이때, deployment.yml과 service.yaml의 스펙에 설정된 frontend이름들은 Root Frontend와의 중복을 회피하기 위해 dist 하위 생성한 폴더명으로 교체한다
```
// 도커라이징
docker build -t [REGISTRY_NAME].azurecr.io/[dist 하위 생성한 폴더명]:[오늘날짜] .
docker push [REGISTRY_NAME].azurecr.io/[dist 하위 생성한 폴더명]:[오늘날짜]

//kubernetes/deployment.yml 19 Line image 스펙 수정
spec:
  containers:
    - name: paymentpbcfe
      image: "[REGISTRY_NAME].azurecr.io/[dist 하위 생성한 폴더명]:[오늘날짜]"

// 클러스터 배포
kubectl apply -f kubernetes/deployment.yml
kubectl apply -f kubernetes/service.yaml
```

## 2. gateway 배포

### 2.1 route설정

#### gateway배포 전 gateway/src/main/resources/application.yml로 이동한 다음, 'profiles: docker' 하위에 위치한 gateway routing rule에 선택한 PBC들의 backend, frontend에 따른 룰 설정을 진행한다.
```
// pbc backend route 설정 예시
cloud:
    gateway:
      routes:
        - id: payment
          uri: http://payment:8080
          predicates:
            - Path=/payments/**,

---

// pbc frontend route 설정 예시
cloud:
    gateway:
      routes:
        - id: paymentpbcfe
          uri: http://paymentpbcfe:8080
          predicates:
            - Path=/paymentpbcfe/**, 
```
#### 위와 같이 fontend를 설정해야 이전에 진행한 pbc내부에서 사용되는 다양한 엔드포인트가 paymentpbcfe/ 엔드포인트 뒤로 들어오며, gateway를 경유하였을 때 요청이 정상적으로 진행된다.

### 2.2 Gateway 패키징, 도커라이징, 클러스터 배포

#### 사용되는 모든 PBC에 대한 route설정이 완료되었다면 아래의 가이드를 참고하여 패키징, 도커라이징, 클러스터 배포를 진행한다.
```
// 패키징
cd gateway
mvn package -B -Dmaven.test.skip=true

// 도커라이징
docker build -t [REGISTRY_NAME].azurecr.io/gateway:[날짜] .
docker push [REGISTRY_NAME].azurecr.io/gateway:[날짜]

//kubernetes/deployment.yml 19 Line image 스펙 수정
spec:
  containers:
    - name: gateway
      image: "[REGISTRY_NAME].azurecr.io/gateway:[오늘날짜]"

// 클러스터 배포
kubectl apply -f kubernetes/deployment.yml
kubectl apply -f kubernetes/service.yaml
```

## 3. Root Frontend 배포

### 3.1 index.html 수정

#### Root Frontend의 경우 사용하는 PBC Frontend를 Single SPA로 동작하기 위해 web-component로 등록하는 작업을 필요로 한다. 따라서, index.html에 다음과 같이 사용하는 pbc에 대해 설정한다.
```
//frontend/public/index.html에 아래 내용 복사후 수정.

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width,initial-scale=1.0">
    <title><%= htmlWebpackPlugin.options.title %></title>
    <link rel="stylesheet" href="https://fonts.googleapis.com/css?family=Roboto:100,300,400,500,700,900">
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@mdi/font@latest/css/materialdesignicons.min.css">
    <link href="https://fonts.googleapis.com/css2?family=Material+Symbols+Outlined:opsz,wght,FILL,GRAD@20,100,1,200" rel="stylesheet" />
    <link href="https://fonts.googleapis.com/css?family=Roboto:100,300,400,500,700,900" rel="stylesheet">
    <link href="https://cdn.jsdelivr.net/npm/@mdi/font@6.x/css/materialdesignicons.min.css" rel="stylesheet">
    <link href="https://cdn.jsdelivr.net/npm/vuetify@2.x/dist/vuetify.min.css" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/vue@2.x/dist/vue.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/vuetify@2.x/dist/vuetify.js"></script>
  </head>
  <body>
    <noscript>
      <strong>We're sorry but <%= htmlWebpackPlugin.options.title %> doesn't work properly without JavaScript enabled. Please enable it to continue.</strong>
    </noscript>
    <div id="app"></div>
    <!-- built files will be auto injected -->
    <script src="http://[Gateway External IP]:8080/[dist 하위에 생성한 폴더 이름]/[dist 하위에 생성됐던 app.js 이름]"></script>
  </body>
</html>

// <script> 예시
<script src="http://20.249.134.209:8080/paymentpbcfe/payment-system-app.js"></script>
```

### 3.2 패키징, 도커라이징, 클러스터 배포

#### 사용되는 모든 PBC에 대한 index.html설정이 완료되었다면 아래의 가이드를 참고하여 패키징, 도커라이징, 클러스터 배포를 진행한다.
```
// 패키징
cd frontend
npm run build

// node_modules가 없을 경우
cd frontend
nvm install 14.19.0
npm install
npm run build

// 도커라이징
docker build -t [REGISTRY_NAME].azurecr.io/frontend:[날짜] .
docker push [REGISTRY_NAME].azurecr.io/frontend:[날짜]

//kubernetes/deployment.yml 19 Line image 스펙 수정
spec:
  containers:
    - name: gateway
      image: "[REGISTRY_NAME].azurecr.io/frontend:[오늘날짜]"

// 클러스터 배포
kubectl apply -f kubernetes/deployment.yml
kubectl apply -f kubernetes/service.yaml
```
