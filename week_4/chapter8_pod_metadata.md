### Downward API로 컨테이너에 정보 전달

- Downward API란?
    - Kubernetes에서 제공하는 기능
    - 컨테이너가 자신이 속한 **Pod의 메타데이터**(예: Pod 이름, 네임스페이스, 라벨 등)나 **리소스 정보**(예: CPU/메모리 제한치와 같은 리소스 설정)를 간단히 조회할 수 있도록 해준다.
    - 이를 통해 컨테이너 내부에서 자기 자신(또는 자신이 속한 Pod)에 대한 정보를 손쉽게 참고있다.
- Downward API는 환경 변수 혹은 파일로 파드의 메타데이타(스펙 또는 상태값)이 채워지도록 한다.
- 즉, 파드 자체의 메타데이터를 해당 파드 내에서 실행 중인 프로세스에 노출할 수 있다.
<img width="860" alt="image" src="https://github.com/user-attachments/assets/1398b72c-466f-4ea8-a941-46941294e2c4" />

- 필요한 이유
    - **자기 참조(self-referential) 정보 활용**
        - 애플리케이션 로깅 시 Pod 이름이나 Namespace, 혹은 특정 라벨 같은 메타데이터를 포함해야 할 때 Downward API를 통해 쉽게 가져와 로깅 정보에 넣을 수 있다.
    - **Pod 스펙에 따른 동적 동작**
        - 컨테이너가 구동되는 환경(리소스 제한, 어플리케이션 버전, 라벨에 지정된 설정값 등)에 따라 다른 동작을 해야 할 때
    - **환경 분리/확장성 용이**
        - 네임스페이스나 라벨 등이 다를 경우 이를 Downward API로 받아와 설정이나 로직에 반영할 수 있으므로, 환경이 달라져도 일관된 컨테이너 이미지를 사용해 개발/운영 환경을 유연하게 구성할 수 있습니다.
- 예: 환경 변수로 전달
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: downward-api-example
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name     # 현재 Pod 이름
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace     # 현재 Pod의 네임스페이스
    ```
    

- 예: Volume 마운트로 전달하기
    - 환경변수로 파드의 레이블이나 어노테이션을 노출할 수 없음 → downward API 볼륨 사용해야함
    - `/etc/config/pod_labels` 파일에 Pod에 설정된 라벨들이 key-value 형태로 저장
    - `/etc/config/cpu_limit` 파일에 CPU 리소스 제한치가 저장
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: downward-api-volume-example
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        volumeMounts:
        - name: downward
          mountPath: /etc/downward   # DownwardAPI 볼륨은 /etc/downward에 마운트됨
      volumes:
      - name: downward
        downwardAPI:
          items:
          - path: "pod_labels"     # 파드의 label은 /etc/downward/pod_labels 파일에 기록
            fieldRef:
              fieldPath: metadata.labels
          - path: "cpu_limit"
            resourceFieldRef:
              containerName: myapp
              resource: limits.cpu
    ```
    

### 쿠버네티스 REST API

- Downward API는 단지 파드 자체의 메타데이터와 모든 파드의 데이터 중 일부만 노출함
- 그 외 애플리케이션에서 클러스터에 정의된 다른 파드나 리소스에 관한 더 많은 정보가 필요할 경우 쿠버네티스 API 서버와 통신해서 가져와야 한다.
- kubectl cluster-info 를 통해 API 서버 URL을 얻을 수 있음. 그러나 서버는 HTTPS를 사용 → 인증 필요
- **kubectl proxy 명령**을 통해 프록시로 서버와 통신할 수 있다.
    - kubectl은 API 서버와 통신에 필요한 모든 것(URL, 인증토큰 등..)을 이미 알고 있으므로 다른 인자를 전달할 필요가 없다.
    - 일반적으로 `~/.kube/config` 파일에 API 서버 주소, 인증 정보(토큰, 인증서 등), 네임스페이스, Context 등이 설정되어 있음
    - kubectl proxy 명령 수행 후, localhost:8001로 호출하면 API 서버와 통신할 수 있다.

```bash
# API 버전 목록 조회
curl http://localhost:8001/

# Pod 리소스 조회 (코어v1)
curl http://localhost:8001/api/v1/pods

# 디플로이먼트(Deployment) 조회 (apps/v1)
curl http://localhost:8001/apis/apps/v1/deployments
```

- Pod 내에서 API 서버와 통신
    - pod 내에는 kubectl 명령 없음.
    - 파드 내부에서 API 서버와 통신하기 위해서는 아래 3가지를 처리해야함.
    1. API 서버 위치 찾기
        - 각 서비스에 대해 환경변수가 구성되어 있음 (pod 실행 시 주입)
        - 또한 각 서비스마다 DNS엔트리가 있음 → 환경변수 조회 할 필요 없이 curl에서 https://kubernetes를 가리키기만 하면 됨. (kubernetes 서비스는 default 네임스페이스에 존재)
    
    ```bash
    # 각 서비스에 대해 환경변수가 구성되어 있음 (pod 실행 시 주입)
    # 아래 명령으로 API 서버의 ip와 포트를 
    $ env | grep KBUERNETES_SERVICE
    KUBERNETES_SERVICE_PORT=443
    KUBERNETES_SERVICE_HOST=10.0.0.1
    KUBERNETES_SERVICE_PORT_HTTPS=443
    ```
    
    1. 진짜 API 서버와 통신하고 있는지 확인
        - 파드 내부에서 ServiceAccount 토큰과 CA 인증서를 사용해 API 서버에 직접 요청해볼 수 있음
        - 즉, 서버의 인증서가 CA로 서명됐는지 확인해본다.
        - 모든 Pod에는 ServiceAccount가 붙어있고, `/var/run/secrets/kubernetes.io/serviceaccount` 경로에 CA와 토큰이 기본 마운트된다.
        
        ```bash
        $ ls /var/run/secrets/kubernetes.io/serviceaccount/
        ca.crt    namespace   token
        ```
        
        ```bash
        # Pod 내부에서 실행
        # 이때 TOKEN 없이 실행하면 unauthorize 떨어짐. 3번에서 설명
        TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
        CA_CERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        
        curl -sS \
          --cacert "$CA_CERT" \
          --header "Authorization: Bearer $TOKEN" \
          https://kubernetes.default.svc.cluster.local/version
        
        ```
        
    
    1. API 서버로 인증
        - 서버에서 인증을 통과해야 클러스터에 배포된 API 오브젝트를 읽고, 업데이트와 삭제를 할 수 있음.
        - 인증하기 위해선 인증 토큰 필요
        - 시크릿 볼륨의 token 파일에 저장됨.

### 앰배서더 컨테이너를 이용한 API 서버 통신 간소화

- API 서버와 직접 통신하는 대신 HTTP로 앰배서더에 연결하고 앰배서더 프록시에 API 서버에 대한 HTTPS 연결을 처리하도록 하는 것.
- 파드의 모든 컨테이너는 동일한 루프백 네트워크 인터페이스를 공유하므로 애플리케이션은 localhost의 포트로 프록시에 액세스 가능

<img width="859" alt="image" src="https://github.com/user-attachments/assets/230ff970-5aff-4fbc-ba9d-d21dbb3a8386" />
