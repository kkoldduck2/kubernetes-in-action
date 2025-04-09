### 사용자 정의 리소스 (Custom Resource)

- 쿠버네티스 API를 확장하여 만든 리소스
- 기본 제공되는 리소스(Pod, Deployment, Service ) 외에 사용자가 정의한 리소스.
- 기존 쿠버네티스 API와 동일한 방식으로 접근 가능(`kubectl` 사용)
- 선언적 API를 통해 관리됨
- 버전 관리, 유효성 검사 등 쿠버네티스 표준 기능 지원

### CustomResourceDefinition(CRD)

- CRD는 사용자 정의 리소스의 스키마를 정의하는 방법
- 새로운 종류의 리소스가 어떤 구조와 속성을 가질지 선언한다.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
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
                engine:
                  type: string
                version:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
      - db
```

### 사용자 정의 리소스 사용 예시

- CRD를 생성한 후, 해당 정의에 따른 사용자 정의 리소스를 생성할 수 있다.

```yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: my-postgres
spec:
  engine: postgres
  version: "13.4"
  replicas: 3
```

### 사용자 정의 컨트롤러

- 사용자 정의 리소스는 보통 사용자 정의 컨트롤러와 함께 사용된다.
- 컨트롤러는 사용자 정의 리소스의 변경사항을 감시하고 그에 따른 실제 작업을 수행한다.
- 예를 들어, 위의 Database 리소스가 생성되면:
    1. 컨트롤러가 이를 감지
    2. 실제 PostgreSQL 데이터베이스를 설정하는 Pod, Service 등을 생성
    3. 리소스 상태를 업데이트하여 진행 상황 보고
<img width="838" alt="image" src="https://github.com/user-attachments/assets/53e3f7ee-731b-4ff1-8c31-290278753220" />

- 사용자 정의 컨트롤러의 동작 이해
    - 사용자 정의 컨트롤러는 메인 컨테이너랑 API 서버로의 앰배서더 역할을 하는 사이드카 컨테이너로 구성됨.
    - 메인 컨테이너는 이 사이드카 컨테이너에서 실행 중인 kubectl proxy 프로세스에 연결해서 TLS 암호화와 인증을 모두 처리하면서 요청을 API 서버로 전달
    - API 서버는 사용자 정의 리소스 오브젝트의 모든 변경 사항에 대한 감시 이벤트를 보냄
    
    <img width="838" alt="image" src="https://github.com/user-attachments/assets/6cbaf3b7-570e-44da-8cdd-f26cd1bd32a6" />


### 사용자 정의 API 서버와 API 서버 애그리게이션

- 쿠버네티스에서 사용자 정의 오브젝트에 대한 지원을 추가하는 좋은 방법은 자체 API 서버를 구현하고 클라이언트와 직접 통신하는 것이다.
- API 서버 애그리게이션
    - 여러 API 서버를 단일 API 서버처럼 작동하도록 하는 기능
    - 주요 쿠버네티스 API 서버가 "애그리게이션 레이어(Aggregation Layer)"를 통해 확장 API 서버들에 요청을 라우팅한다.
    - 클라이언트 → 주 API 서버(애그리게이션 레이어) → 확장 API 서버
    - 일반적으로 각 API 서버는 자신의 리소스 저장을 담당한다. (자체 etcd 인스턴스를 실행하거나, 코어 API 서버에 CRD 인스턴스를 생성해 코어 API 서버의 etcd 저장소에 리소스를 저장할 수 있다)
    
    <img width="838" alt="image" src="https://github.com/user-attachments/assets/3bfb8092-80bc-4cb9-88bc-396c5d498373" />


### 서비스 카탈로그

- 쿠버네티스 클러스터에서 외부 관리형 소프트웨어 서비스(주로 클라우드 제공업체의 서비스)를 쉽게 사용할 수 있게 해주는 확장 API
- 왜 사용함?
    - 쿠버네티스 애플리케이션은 종종 데이터베이스, 메시징 큐, 캐시 등의 외부 서비스가 필요함. 근데 이런 서비스를 수동으로 프로비저닝하고 자격증명을 관리하는 것은 번거롭고 오류 발생이 쉬움
    - 예) 개발팀이 새 프로젝트에 MySQL 데이터베이스가 필요할 때, AWS RDS 콘솔에 로그인하고, DB를 생성하고, 자격 증명을 얻어서, 쿠버네티스 Secret을 수동으로 만들어야 합니다.
    - 서비스 카탈로그를 사용하면 YAML 파일 하나로 이 모든 과정을 자동화할 수 있습니다.
- 서비스 카탈로그는 클라우드 서비스 브로커(Cloud Service Broker) API와 쿠버네티스를 통합해 다음과 같은 워크플로우를 제공
    - 사용 가능한 서비스 목록 조회
    - 서비스 프로비저닝(생성)
    - 애플리케이션과 서비스 바인딩(연결)
    - 서비스 디프로비저닝(삭제)
- 주요 구성 요소
    - **서비스 브로커(Service Broker)**:
        - 서비스를 프로비저닝할 수 있는 (외부)시스템
    - **서비스 클래스(Service Class)**:
        - 브로커가 제공하는 프로비저닝할 수 있는 서비스 유형 정의
        - 예: "AWS RDS MySQL", "Google Cloud Pub/Sub"
    - **서비스 인스턴스(Service Instance)**:
        - 프로비저닝된 실제 서비스의 인스턴스
        - 예: "my-production-database"
    - **서비스 바인딩(Service Binding)**:
        - 쿠버네티스 애플리케이션(클라이언트 파드)과 서비스 인스턴스 바인딩
        - 연결 정보(자격 증명 등)를 Secret으로 제공
<img width="838" alt="image" src="https://github.com/user-attachments/assets/5800e829-e386-4aeb-ab31-c1675bfbc078" />

- 서비스 카탈로그 아키텍처
    - 서비스 카탈로그는 다음 3가지 구성요소로 구성된 분산 시스템
    <img width="838" alt="image" src="https://github.com/user-attachments/assets/dc2c0e45-dfcf-4b4f-a526-366696eb1658" />

    - 서비스 카탈로그 API 서버
    - etcd 스토리지
    - 모든 컨트롤러를 실행하는 컨트롤러 매니저
    
- 사용 예시
    - 서비스 브로커 등록
        
        ```yaml
        apiVersion: servicecatalog.k8s.io/v1beta1
        kind: ClusterServiceBroker
        metadata:
          name: cloud-broker
        spec:
          url: https://servicebroker.example.com
        ```
        
    - 서비스 인스턴스 생성
        
        ```yaml
        apiVersion: servicecatalog.k8s.io/v1beta1
        kind: ServiceInstance
        metadata:
          name: my-db-instance
          namespace: test-ns
        spec:
          clusterServiceClassExternalName: cloud-mysql
          clusterServicePlanExternalName: standard
          parameters:
            storageGB: 20
            version: "5.7"
        ```
        
    - 서비스 바인딩 생성
        
        ```yaml
        apiVersion: servicecatalog.k8s.io/v1beta1
        kind: ServiceBinding
        metadata:
          name: my-db-binding
          namespace: test-ns
        spec:
          instanceRef:
            name: my-db-instance
          secretName: my-db-credentials
        ```
        
    - 애플리케이션에서 바인딩 사용
        
        ```yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: my-app
        spec:
          template:
            spec:
              containers:
              - name: my-app
                env:
                - name: DB_CONNECTION_STRING
                  valueFrom:
                    secretKeyRef:
                      name: my-db-credentials
                      key: connectionString
        ```
