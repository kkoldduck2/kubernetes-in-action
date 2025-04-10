## 18장 : 쿠버네티스 확장

## 1. 사용자 정의 API 오브젝트

### 1.1 CustomResourceDefinition (CRD)

**개요:**

- CRD는 쿠버네티스에서 새로운 리소스 유형을 정의하기 위한 오브젝트 (이전에는 ThirdPartyResource라고 불림)
- CRD를 게시하면 JSON, YAML 등의 매니페스트로 사용자 정의 리소스 인스턴스 생성 가능
- 리소스가 실제 작업을 수행하려면 해당 컨트롤러 배포가 필수적

**생성 방법:**

1. CRD 오브젝트를 API 서버에 게시

   * CRD는 `Website`라는 새로운 리소스 타입을 정의하며, `extensions.example.com` API 그룹에 속하고 버전은 `v1`
     이 리소스는 `spec.gitRepo` 속성을 가질 수 있고  문자열 타입이며 yaml apply 후 `Website` 타입의 리소스를 생성

   ```yaml
   #crd.yaml
   apiVersion: apiextensions.k8s.io/v1
   kind: CustomResourceDefinition
   metadata:
     name: websites.extensions.example.com
   spec:
     scope: Namespaced
     group: extensions.example.com
     versions:
       - name: v1
         served: true
         storage: true
         schema:
           openAPIV3Schema:
             type: object
             properties:
               spec:
                 type: object
                 properties:
                   gitRepo:
                     type: string
     names:
       kind: Website
       singular: website
       plural: websites
   ```

   

2. CRD apply 및 확인

   ```sh
   $ kubectl apply -f crd.yaml
   customresourcedefinition.apiextensions.k8s.io/websites.extensions.example.com created
   
   
   $ kubectl get crd
   NAME                              CREATED AT
   websites.extensions.example.com   2025-04-09T13:17:01Z
   ```

   

3. 사용자 정의 리소스 매니페스트 정의 (예: imaginary-kubia-website.yaml)

   ```yaml
   apiVersion: "extensions.example.com/v1"
   kind: Website
   metadata:
     name: kubia #서비스와 파드에 사용되는 웹사이트 이름
   spec:
     gitRepo: https://github.com/luksa/kubia-website-example.git
   ```

   

4. Kind Website apply 및 확인
   ```sh
   $ kubectl apply -f test.yaml
   website.extensions.example.com/kubia created
   
   $ kubectl get Website
   NAME    AGE
   kubia   61s
   ```

   

**주요 특징:**

- 최신 버전에서는 apiVersion, versions 항목이 필수
- 스키마 검증을 위한 OpenAPIV3Schema 정의 필요
- 네임스페이스 범위 지정 가능



---



### 1.2 사용자 정의 컨트롤러

**기능:**

- API 서버를 감시하다가 사용자 정의 리소스(CRD) 생성 시 사전 정의된 작업 수행
- 예: 웹사이트 리소스 생성 시 서비스와 웹서버 파드(디플로이먼트) 생성

**컨트롤러의 주요 동작:**

1. 사용자 정의 리소스 오브젝트 감시
2. API 서버의 감시 이벤트 수신 (ADDED, DELETED 등)
3. 이벤트 내 정보 추출 및 처리
4. 필요한 리소스 생성 (디플로이먼트, 서비스 등)
5. 리소스 삭제 시 관련 리소스 정리

**컨트롤러 배포 절차:**

1. 필요한 서비스 어카운트 생성
2. RBAC이 활성화된 경우 클러스터롤바인딩 생성
3. 디플로이먼트로 컨트롤러 배포

생성방법:

1. ServiceAccount 생성

```bash
$ kubectl create serviceaccount website-controller
serviceaccount/website-controller created
```

2. RBAC 권한 설정

클러스터 내 RBAC이 활성화된 경우, 컨트롤러가 감시 작업이나 디플로이먼트 생성을 할 수 있도록 클러스터롤바인딩을 생성합니다:

```bash
$ ubectl create clusterrolebinding website-controller --clusterrole=cluster-admin \
  --serviceaccount=default:website-controller
clusterrolebinding.rbac.authorization.k8s.io/website-controller created
```

3. 컨트롤러 디플로이먼트 정의

`website-controller.yaml` 파일을 생성합니다:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: website-controller
  labels:
    app: website-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      app: website-controller
  template:
    metadata:
      labels:
        app: website-controller
    spec:
      serviceAccountName: website-controller
      containers:
      - name: main
        image: luksa/website-controller
      - name: proxy
        image: luksa/kubectl-proxy:1.6.2
```

이 디플로이먼트는 다음 두 컨테이너를 포함합니다:

- `main`: 실제 컨트롤러 로직이 있는 컨테이너
- `proxy`: kubectl proxy 프로세스를 실행해 API 서버와의 연결을 돕는 사이드카 컨테이너

4. 컨트롤러 배포

정의한 디플로이먼트를 적용합니다:

```bash
$ kubectl apply -f website-controller.yaml
deployment.apps/website-controller created
```

5. 작동 확인

컨트롤러 배포 후, 실제 Website 리소스를 생성하여 컨트롤러가 제대로 작동하는지 확인할 수 있습니다:

```bash
# Website 리소스 생성
$ kubectl apply -f imaginary-kubia-website.yaml

# 컨트롤러 로그 확인
$ kubectl logs -l app=website-controller -c main

2025/04/09 13:34:21 Received watch event: DELETED: kubia: https://github.com/luksa/kubia-website-example.git
2025/04/09 13:34:21 Deleting services with name kubia-website in namespace default
2025/04/09 13:34:21 response Status: 200 OK
2025/04/09 13:34:21 Deleting deployments with name kubia-website in namespace default
2025/04/09 13:34:21 response Status: 404 Not Found
2025/04/09 13:34:25 Received watch event: ADDED: kubia: https://github.com/luksa/kubia-website-example.git
2025/04/09 13:34:25 Creating services with name kubia-website in namespace default
2025/04/09 13:34:25 response Status: 201 Created
2025/04/09 13:34:25 Creating deployments with name kubia-website in namespace default
2025/04/09 13:34:25 response Status: 404 Not Found

# 생성된 리소스 확인
$ kubectl get deploy,svc,pod |grep website
deployment.apps/website-controller   1/1     1            1           3m58s
service/kubia-website    NodePort       10.106.142.13    <none>        80:32425/TCP                 39s
pod/website-controller-56f5bcdf69-mhvdm   2/2     Running   1 (3m53s ago)   3m58s
```

컨트롤러의 주요 동작 과정

1. 컨트롤러는 API 서버를 감시하며 Website 오브젝트 변경을 감지합니다.
2. 새 Website 오브젝트가 생성되면 ADDED 이벤트를 수신합니다.
3. 컨트롤러는 오브젝트에서 이름, 깃레포 URL 등 정보를 추출합니다.
4. 추출한 정보를 바탕으로 디플로이먼트와 서비스 매니페스트를 작성합니다.
5. API 서버에 이 매니페스트를 게시하여 실제 리소스를 생성합니다.
6. Website 오브젝트가 삭제되면 DELETED 이벤트를 수신하고 관련 리소스를 정리합니다.

이 과정을 통해 사용자 정의 리소스에 대응하는 컨트롤러를 만들고 배포할 수 있습니다.



---



### 1.3 사용자 정의 오브젝트 유효성 검증

- 최신 버전에서는 CRD에 스키마 지정이 필수로, OpenAPI v3 스키마를 통해 유효성 검증
- Validation Rules 기능이 활성화된 경우 x-kubernetes-validations 사용 가능
- 추가 검증은 admission webhook을 통해 가능



---



### 1.4 사용자 정의 API 서버

**API 서버 애그리게이션:**

- 사용자 정의 API 서버를 기본 쿠버네티스 API 서버와 통합하는 기능
- 여러 애그리게이션 API 서버가 단일 경로로 노출됨
- 리소스 저장 방식:
  1. 자체 ETCD 인스턴스 사용
  2. 주 서버 ETCD 저장소에 CRD로 저장

**생성 방법:**

1. APIService 리소스를 통해 사용자 정의 API 서버 등록
2. 사용자 정의 클라이언트 생성 (kubectl 또는 사용자 정의 CLI 도구)



**메트릭 서버**: 클러스터 내 노드와 파드의 리소스 사용량 메트릭을 제공합니다.

- API 그룹: `metrics.k8s.io`
- 사용 예: `kubectl top pods`, HPA(HorizontalPodAutoscaler)

**커스텀 메트릭 API**: 애플리케이션별 메트릭을 제공합니다.

- API 그룹: `custom.metrics.k8s.io`
- 사용 예: 비즈니스 메트릭 기반 스케일링

**서비스 카탈로그 API 서버**: 외부 서비스와의 통합을 위한 API를 제공합니다.

- API 그룹: `servicecatalog.k8s.io`
- 리소스: ClusterServiceBroker, ServiceInstance 등



---





## 2. 서비스 카탈로그를 통한 셀프 서비스

### 2.1 서비스 카탈로그 소개

- 클라우드 서비스(AWS RDS, Azure CosmosDB 등)나 관리형 소프트웨어를 쿠버네티스 애플리케이션에 쉽게 연결
- 복잡한 리소스 생성 과정을 추상화하여 사용자가 직접 파드, 서비스, 컨피그맵 등을 다루지 않고도 서비스 인스턴스 프로비저닝 가능
- Open Service Broker API를 사용하여 외부 서비스와 통신

#### 서비스 카탈로그 주요 리소스

1. **ClusterServiceBroker**
   - 서비스를 프로비저닝할 수 있는 외부 시스템(서비스 브로커)을 정의
   - 클러스터 관리자가 서비스 브로커를 등록하면 사용 가능한 서비스 목록을 가져옴
2. **ClusterServiceClass**
   - 프로비저닝 가능한 서비스 유형을 정의
   - 예: MySQL 데이터베이스, RabbitMQ 메시징 서비스 등
3. **ClusterServicePlan**
   - 서비스의 구체적인 구성 옵션(티어, 사양 등)을 정의
   - 예: 무료 티어, 프리미엄 티어, SSD 스토리지 옵션 등
4. **ServiceInstance**
   - 실제로 프로비저닝된 서비스의 인스턴스
   - 사용자가 특정 서비스를 사용하기 위해 직접 생성
5. **ServiceBinding**
   - 파드와 서비스 인스턴스 간의 연결을 정의
   - 서비스 접근에 필요한 자격 증명을 시크릿으로 생성하여 파드에 제공





### 2.2 서비스 카탈로그 API 서버 및 컨트롤러 매니저

- 서비스 카탈로그 API 서버: 리소스 게시 및 생성
- ETCD 스토리지: 리소스 저장
- 컨트롤러 매니저: 컨트롤러 실행, API 서버와 통신하여 외부 서비스 브로커와 연동





### 2.3 ServiceBroker와 OpenServiceBroker API

**OpenServiceBroker API 주요 기능:**

- 서비스 목록 검색
- 서비스 인스턴스 프로비저닝/디프로비저닝
- 서비스 인스턴스 업데이트
- 서비스 인스턴스 바인딩 설정/해제

### 2.4 프로비저닝과 서비스 사용

**ServiceInstance 프로비저닝:**

- 특정 서비스 클래스와 플랜을 지정하여 서비스 인스턴스 생성
- 파라미터를 전달하여 브로커가 실제 프로비저닝 수행

**서비스 인스턴스 바인딩:**

- 프로비저닝된 서비스 인스턴스에 연결하기 위한 바인딩 리소스 생성
- 바인딩을 통해 생성된 시크릿을 파드에 마운트하여 서비스 사용

### 2.5 바인딩 해제와 프로비저닝 해제

- 바인딩 삭제 시 관련 시크릿 삭제 및 브로커에 바인딩 해제 요청
- 서비스 인스턴스 삭제 시 브로커에 프로비저닝 해제 요청

### 2.6 예시: 데이터베이스 서비스 프로비저닝

```yaml
# 데이터베이스 서비스 브로커 등록
apiVersion: servicecatalog.k8s.io/v1beta1
kind: ClusterServiceBroker
metadata:
  name: database-broker
spec:
  url: http://database-osbapi.myorganization.org

---
# 데이터베이스 인스턴스 프로비저닝
apiVersion: servicecatalog.k8s.io/v1beta1
kind: ServiceInstance
metadata:
  name: my-postgres-db
  namespace: default
spec:
  clusterServiceClassExternalName: postgres-database
  clusterServicePlanExternalName: free
  parameters:
    init-db-args: --data-checksums

---
# 서비스 인스턴스 바인딩
apiVersion: servicecatalog.k8s.io/v1beta1
kind: ServiceBinding
metadata:
  name: my-postgres-db-binding
  namespace: default
spec:
  instanceRef:
    name: my-postgres-db
  secretName: postgres-secret
```

서비스 카탈로그는 클라우드 서비스 사용을 단순화하여 쿠버네티스 애플리케이션과 외부 서비스 간의 통합을 쉽게 만들어 주는 중요한 확장 기능입니다.



## 3. 쿠버네티스 기반 플랫폼

### 3.1 레드햇 오픈시프트 컨테이너 플랫폼

#### **주요 기능:**

- CI 솔루션 통합, 이미지 빌드 및 배포 자동화
- 향상된 보안의 멀티 테넌트 클러스터 구성
- 사용자 및 그룹 관리 기능



#### 오픈시프트의 추가 리소스:

오픈시프트는 무료 버전(OKD - Origin Kubernetes Distribution)에서도 다음 추가 리소스를 제공합니다:



##### 1. 사용자/그룹/프로젝트 관리

오픈시프트는 쿠버네티스 기본 RBAC에 더하여 강화된 사용자 관리 기능을 제공합니다.

**주요 기능:**

- 통합 사용자 인증 시스템 (LDAP, OAuth 등 지원)
- 세분화된 권한 관리와 역할 기반 접근 제어
- 프로젝트(네임스페이스) 수준의 리소스 분리와 격리

**예시 명령어:**

```bash
# 프로젝트 생성
oc new-project my-project

# 사용자에게 프로젝트 접근 권한 부여
oc policy add-role-to-user edit username -n my-project
```



##### 2. 애플리케이션 템플릿

템플릿은 여러 쿠버네티스 리소스를 파라미터화된 형태로 정의할 수 있게 해줍니다.

**주요 기능:**

- 복잡한 애플리케이션 배포를 단순화
- 파라미터를 통한 커스터마이징 지원
- 사전 정의된 템플릿 카탈로그 제공

**템플릿 예시:**

```yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: redis-template
  annotations:
    description: "Redis 데이터베이스 서비스"
objects:
- apiVersion: v1
  kind: Service
  metadata:
    name: ${DATABASE_SERVICE_NAME}
  spec:
    ports:
    - port: 6379
    selector:
      name: ${DATABASE_SERVICE_NAME}
- apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: ${DATABASE_SERVICE_NAME}
  spec:
    replicas: 1
    selector:
      matchLabels:
        name: ${DATABASE_SERVICE_NAME}
    template:
      metadata:
        labels:
          name: ${DATABASE_SERVICE_NAME}
      spec:
        containers:
        - name: redis
          image: redis:${REDIS_VERSION}
          ports:
          - containerPort: 6379
parameters:
- name: DATABASE_SERVICE_NAME
  description: 데이터베이스 서비스 이름
  value: redis
- name: REDIS_VERSION
  description: Redis 버전
  value: "6.0"
```



##### 3. BuildConfig

소스 코드에서 컨테이너 이미지를 자동으로 빌드하는 CI/CD 기능을 제공합니다.

**주요 기능:**

- 깃 레포지토리 변경 감지 및 자동 빌드
- Source-to-Image(S2I) 지원으로 소스 코드에서 직접 이미지 생성
- 다양한 빌드 전략 지원: Docker, S2I, 커스텀

**BuildConfig 예시:**

```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: sample-app
spec:
  source:
    git:
      uri: https://github.com/sclorg/nodejs-ex.git
  strategy:
    type: Source
    sourceStrategy:
      from:
        kind: ImageStreamTag
        name: nodejs:14
        namespace: openshift
  output:
    to:
      kind: ImageStreamTag
      name: sample-app:latest
  triggers:
  - type: GitHub
    github:
      secret: secret101
  - type: ConfigChange
```



##### 4. DeploymentConfig

새로운 이미지가 빌드되면 자동으로 배포를 트리거하는 리소스입니다.

**주요 기능:**

- 새 이미지 변경 감지 및 자동 롤아웃
- 배포 전/후 훅 지원으로 마이그레이션 등 작업 자동화
- 다양한 배포 전략(롤링, 재생성 등) 지원

**DeploymentConfig 예시:**

```yaml
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  name: sample-app
spec:
  replicas: 2
  selector:
    app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: sample-app:latest
        ports:
        - containerPort: 8080
  triggers:
  - type: ConfigChange
  - type: ImageChange
    imageChangeParams:
      automatic: true
      containerNames:
      - sample-app
      from:
        kind: ImageStreamTag
        name: sample-app:latest
  strategy:
    type: Rolling
```



##### 5. Route

외부 트래픽을 서비스로 라우팅하는 기능을 제공합니다.

**주요 기능:**

- 인그레스와 유사하지만 더 많은 기능 제공
- TLS 종료 지원
- 트래픽 분할 및 가중치 기반 라우팅
- 고급 라우팅 정책

**Route 예시:**

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: sample-app
spec:
  host: app.example.com
  to:
    kind: Service
    name: sample-app
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
  alternateBackends:
  - kind: Service
    name: sample-app-canary
    weight: 10
```



