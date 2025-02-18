### 서비스가 필요한 이유?

- Pod는 일시적. pod가 어떠한 이유로 내렸다가 다시 뜨면 IP 주소가 변함
- 따라서 클라이언트 파드가 서버 파드의 IP를 알기 어렵다.
- 또한 수평 스케일링으로 서버 파드가 여러 개가 될 경우에도 클라이언트는 단일 IP를 통해 접근할 수 있어야 한다.
- 따라서 쿠버네티스는 이런 문제를 해결하고자 서비스 리소스를 제공한다.

### 클라이언트 파드가 서비스의 IP 주소를 알 수 있는 방법?

1. 환경 변수를 통한 서비스 검색
    - 쿠버네티스는 새로운 파드가 시작될 때 해당 시점에 존재하는 모든 서비스에 대한 환경 변수를 자동으로 주입한다.
    - 환경변수 형식은 다음과 같다.
        
        ```yaml
        {서비스이름}_SERVICE_HOST - 서비스의 IP 주소
        {서비스이름}_SERVICE_PORT - 서비스의 포트
        ```
        
    - 예를 들어, 'database'라는 이름의 서비스가 있다면
        
        ```yaml
        DATABASE_SERVICE_HOST=10.0.0.11
        DATABASE_SERVICE_PORT=3306
        ```
        
    - 컨테이너 내부에서 env 명령어를 실행해 환경변수를 조회할 수 있다.
    - 주의할 점:
        - 환경 변수는 파드가 생성되는 시점에만 주입된다.
        - 파드가 실행된 후에 생성된 서비스는 환경 변수로 주입되지 않는다.
2. DNS를 통한 서비스 검색
    - 쿠버네티스는 클러스터 내부 DNS 서버를 실행하며, 모든 서비스에 대해 DNS 레코드를 자동으로 생성한다.
    - 쿠버네티스 내부 DNS 서버 : kube-system 네임스페이스에 kube-dns라는 이름의 파드와 서비스
    - 서비스 이름을 알고 있는 클라이언트 파드는 환경변수 대신 FQDN (정규화된 도메인 이름)으로 액세스 할 수 있다.
    - FQDN : `{서비스이름}.{네임스페이스}.svc.cluster.local`

**덧) 서비스 IP에 ping을 할 수 없는 이유**

- 서비스 IP의 특성
    - 서비스 IP는 실제 네트워크 인터페이스에 할당되지 않은 가상 IP임
    - 이 IP는 kube-proxy에 의해 관리되는 순수한 라우팅/포워딩 규칙으로만 존재
    - 특정 포트로의 TCP/UDP 연결만을 위해 설계됨
- ping이 안되는 이유
    - ping은 ICMP 프로토콜을 사용
    - kube-proxy는 TCP/UDP 트래픽만 처리하도록 설계되어 있고, ICMP 패킷은 처리하지 않음
    - 따라서 ICMP 요청(ping)은 서비스 IP에 도달할 수 없음
- curl이 되는 이유
    - curl은 TCP 기반의 HTTP 프로토콜을 사용
    - kube-proxy는 TCP 트래픽을 인식하고 적절한 파드로 포워딩
    - 서비스에 정의된 포트로의 TCP 연결은 정상적으로 동작
- telnet 요청도 TCP 기반이므로 동작함

### 서비스 오브젝트는 어디에 존재하는 거고, kube-proxy와 어떻게 상호작용하는가?

- 서비스 오브젝트는 etcd에 저장되는 클러스터 범위의 논리적인 객체임
- 즉, 물리적으로 특정 노드에 떠 있는 것이 아니라, kubernetes 클러스터 전체에서 존재하는 논리적인 개념
- 서비스 오브젝트와 kube-proxy의 상호작용
    1. 서비스 오브젝트는 etcd(클러스터 저장소)에 저장됨
        - `Service` 객체를 생성하면, Kubernetes API Server가 이를 `etcd`에 저장함.
        - `kubectl get svc` 명령어를 실행하면, API Server에서 `etcd`에 저장된 정보를 조회하는 것.
    2. kube-proxy가 서비스 오브젝트를 감시 (watch)
        - `kube-proxy`는 Kubernetes API Server에서 서비스와 엔드포인트 정보를 감시(watch) 함.
        - 즉, `kube-proxy`는 이 정보를 기반으로 노드에서 라우팅을 설정하는 역할을 함.
    3. kube-proxy가 각 노드에서 iptables/IPVS 규칙을 설정
        - `kube-proxy`는 API Server에서 받은 `Service` 및 `Endpoints` 정보를 이용해서, 각 노드에서 iptables/IPVS 규칙을 설정함.
        - 이 규칙을 통해, ClusterIP로 들어오는 트래픽이 적절한 Pod로 전달됨.
- 서비스와 kube-proxy의 트래픽 흐름 (ClusterIP 기준)
    - 사용자가 ClusterIP로 요청을 보냄 (`curl http://<service-ip>:<port>`)
    - 요청이 Kubernetes 노드의 kube-proxy로 들어감
    - kube-proxy가 iptables 또는 IPVS 규칙을 사용하여 실제 Pod의 IP로 변경(DNAT)하여 전달
    - Pod가 요청을 처리하고, 응답을 다시 클라이언트에게 보냄

- kube proxy란?
    - 클러스터의 각 노드에서 실행되는 네트워크 프록시로, 쿠버네티스의 서비스 개념의 구현부이다
    - 모든 노드마다 데몬셋으로 설치됨
    - 서비스 오브젝트의 변화를 감지하고 -> 해당 노드의 네트워크 룰 설정을 변경함

참고) 
- https://kubernetes.io/ko/docs/concepts/overview/components/
- https://kodekloud.com/blog/kube-proxy/


### 클러스터 외부에 있는 서비스 연결하는 방법

1. 서비스 endpoint를 수동으로 구성하여 외부 IP 주소로 요청이 가도록 한다.
    - 서비스 yaml 파일에 selector를 추가하면 endpoint 리소스가 자동으로 생성된다.
    <img width="517" alt="image" src="https://github.com/user-attachments/assets/e95a8fe2-f2f1-47c5-9221-ee7b6ebb1635" />

    - selector 없이 서비스를 구성하면, endpoint 리소스를 수동으로 구성할 수 있다. (이때 endpoint 이름 = 서비스 이름)
    <img width="517" alt="image" src="https://github.com/user-attachments/assets/f3335afb-04cf-45db-ade0-4b626cf09ddb" />

    
2. 외부 서비스를 위한 별칭 생성
    - 서비스 생성 시 type을 ExternalName으로 한다.
    <img width="517" alt="image" src="https://github.com/user-attachments/assets/ce10d6c2-90f8-452a-a3f7-69b466011a09" />

    - 예 : 외부 데이터베이스(MySQL)를 Kubernetes 내부에서 사용할 때
    
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: external-db
      namespace: default
    spec:
      type: ExternalName
      externalName: db.example.com
    ```
    
    - 이 `Service`를 조회하면 `db.example.com`이라는 CNAME을 반환함.
    - `kubectl get svc external-db` 실행 시 `db.example.com`을 확인할 수 있음.
    - 클러스터 내부에서 `external-db.default.svc.cluster.local`로 접근하면 자동으로 `db.example.com`으로 리다이렉트됨.

### 외부 클러스터에 서비스 노출하는 방법

1. Node Port 서비스
    - 노드 포트 서비스를 생성하면 쿠버네티스는 모든 노드에 특정 포트를 할당하고, 서비스를 구성하는 파드로 들어오는 연결을 전달한다.
    - 노드에 대한 방화벽 설정 필요
    - 특정 노드가 장애나면, 해당 노드로 요청을 보내는 클라이언트는 더 이상 서비스에 액세스할 수 없음
    - 따라서 모든 노드에 요청을 분산시키도록 노드 앞에 로드밸런서를 배치하는 것이 좋다.
  <img width="517" alt="image" src="https://github.com/user-attachments/assets/4147e279-1a17-4c33-927c-f6c9ef201dc7" />

    
2. 로드밸런서 서비스
    - 노드 포트 서비스의 확장
    - 로드밸런서는 공개적으로 액세스 가능한 IP 주소를 가지며 모든 연결을 서비스로 전달함.
    - 노드에 대한 방화벽 설정 필요 없음
    - 덧) curl 명령과 달리, 브라우저에서 keep-alive로 요청을 보낼 경우, 처음 요청을 보낸 pod로 계속 요청이 가게 됨 (세션 어피니티 설정이 None이더라도. keep-alive는 TCP 연결을 유지하기 때문)
    <img width="517" alt="image" src="https://github.com/user-attachments/assets/90818e4d-b39d-4865-9ccc-f561a38559b8" />


1. 인그레스
    - 로드밸런서 서비스는 자신의 공용 IP 주소를 가진 로드밸런서가 필요한 반면, 인그레스는 한 IP 주소로 여러 개의 서비스에 접근이 가능하도록 함
    - 클라이언트가 HTTP 요청을 인그레스에 보낼 때, 요청한 호스트와 경로에 따라 요청을 전달할 서비스가 결정된다.
    - TSL (SSL) 인증서 적용이 가능하다.


- 인그레스 컨트롤러와 인그레스 리소스
    - Ingress는 기본적으로 Ingress Controller가 있어야 동작함
    - Ingress 자체는 설정 파일일 뿐이고, 실제 트래픽을 처리하는 것은 **Ingress Controller**가 맡음
    - Ingress Controller : 실제 요청을 라우팅하는 컨트롤러 (예: Nginx Ingress, Traefik, HAProxy)
    - Ingress 리소스 (yaml) : 사용자가 설정하는 Ingress 객체

- 인그레스 동작 방식
    <img width="517" alt="image" src="https://github.com/user-attachments/assets/a1d4455d-8cd3-42a5-8f44-bd94e753de63" />

    1. 클라이언트는 먼저 kubia.example.com의 DNS 조회를 수행. DNS 서버(혹은 로컬 host 파일)는 인그레스 컨트롤러의 IP를 반환
    2. 클라이언트는 HTTP 요청을 인그레스 컨트롤러로 전송하고 host 헤더에서 kubia.example.com을 지정
    3. 컨트롤러는 해당 헤더에서 클라이언트가 액세스하려는 서비스를 결정하고 서비스와 관련된 **앤드포인트 오브젝트로 파드 IP 조회 후** 클라이언트 요청을 **파드에 전달**한다.
        
        즉, 요청을 서비스로 전달하지 않는다. 파드를 선택하는데만 사용한다.
