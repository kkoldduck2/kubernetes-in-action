## 13장 : 클러스터 노드와 네트워크보안

### hostNetwork

* 컨테이너가 호스트 노드의 네트워크 네임스페이스를 직접 공유
* 컨테이너의 포트가 호스트의 포트에 직접 바인딩
* Pod가 호스트의 네트워크 인터페이스와 IP 주소를 사용



---



### hostPort  / NodePort 차이

* hostPort는 단일 Pod와 노드에 대한 직접적인 바인딩
* NodePort는 클러스터 모든 워커노드에서 서비스에 접근할 수 있는 방법을 제공하는 Service 유형



---



### hostPID

* 컨테이너가 호스트 노드의 PID(프로세스 ID) 네임스페이스를 공유
* Pod 내의 컨테이너가 호스트의 모든 프로세스를 볼 수 있음



---



###  hostIPC

* 컨테이너가 호스트 노드의 IPC(프로세스 간 통신) 네임스페이스를 공유
* 호스트 프로세스와 직접 통신이 필요한 애플리케이션에서 사용



---



### SecuriyContext

```yaml
spec:
  securityContext: 
    fsGroup: 555 #프로세스가 볼륨에 파일을 생성할 때 사용
    supplementalGroups: [666,777] #사용자와 관련된 추가 그룹 ID 목록을 정의
  containers:
    securityContext: 
      runAsUser: 1111 # 컨테이너 ID 1111 지정
 
      capabilities: #	이 컨테이너에서는 파일 소유권을 변경 할 수 없다
        drop:
        - CHOWN
      readOnlyRootFilesystem: true # 컨테이너 파일시스템에 쓰기를 할 수 없다 
    volumeMounts:
    - name: my-volume # 하지만 마운트된 볼륨인 /volume에는 쓸 수 있다.
    
```

  

```sh
$ ls -l
-rw-r--r--- 1 1111 555 4 May 29 12:25 foo
```





---



### PodSecurityPolicy 

쿠버네티스 1.25부터 PodSecurityPolicy는 완전히 제거 Pod Security Admission(PSA)을 사용해야 합니다

클러스터 수준 리소스로 사용자가 파드에서 사용할 수 있거나 사용할 수 없는 보안 관련 기능 정의

PodSecurityPolicy 어드미션 컨트롤 플러그인으로 수행

- 호스트의 IPC,PID또는 네트워크 네임스페이스를 사용 할 수 있는지 여부
- 호스트포트 사용여부
- 사용자ID/ 파일시스템 그룹 여부
- Privileged Container 생성 여부
- 사용 할 수 있는 볼륨 유형



전역 PSP 적용 샘플

```yaml
# 1. 정의
apiVersion: policy/v1beta1  # PodSecurityPolicy의 API 버전
kind: PodSecurityPolicy  # 리소스 종류
metadata:
  name: restricted-global  # 정책 이름
spec:
  privileged: false  # 특권 컨테이너 실행 금지
  # 필수적인 기본 설정
  allowPrivilegeEscalation: false  # 권한 에스컬레이션 금지
  
  # 호스트 네임스페이스 제한
  hostNetwork: false  # 호스트 네트워크 사용 금지
  hostIPC: false  # 호스트 IPC 네임스페이스 사용 금지
  hostPID: false  # 호스트 PID 네임스페이스 사용 금지
  
  # 볼륨 및 파일시스템 제한
  volumes:  # 허용되는 볼륨 타입 목록
  - 'configMap'
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'downwardAPI'
  - 'persistentVolumeClaim'  # PVC만 허용
  
  fsGroup:  # 파일시스템 그룹 ID 범위
    rule: 'MustRunAs'  # 반드시 지정된 범위에서 실행
    ranges:
    - min: 1  # 최소 GID
      max: 65535  # 최대 GID
  
  # 사용자 및 그룹 설정
  runAsUser:
    rule: 'MustRunAsNonRoot'  # 루트 사용자로 실행 금지
  
  runAsGroup:
    rule: 'MustRunAs'  # 반드시 지정된 범위에서 실행
    ranges:
    - min: 1  # 최소 GID
      max: 65535  # 최대 GID
  
  supplementalGroups:  # 보조 그룹 ID 범위
    rule: 'MustRunAs'  # 반드시 지정된 범위에서 실행
    ranges:
    - min: 1  # 최소 GID
      max: 65535  # 최대 GID
  
  # 리눅스 capabilities 제한
  requiredDropCapabilities:  # 반드시 삭제해야 하는 capabilities
  - ALL  # 모든 capabilities 삭제
  defaultAddCapabilities : [] # 모든 컨테이너에 추가
  allowedCapabilities: []  # 추가 허용되는 capabilities (비어있음 = 없음)
  
  # SELinux 컨텍스트 제한
  seLinux:
    rule: 'RunAsAny'  # 모든 SELinux 컨텍스트 허용
  
  # 호스트 경로 마운트 제한
  allowedHostPaths: []  # 호스트 경로 마운트 없음
  
  # Seccomp 프로필 설정
  # (seccomp 설정은 주로 어노테이션을 통해 이루어짐)
  
  # ReadOnlyRootFilesystem은 PSP 1.11 버전부터 지원
  readOnlyRootFilesystem: false  # 루트 파일시스템의 읽기 전용 여부
  
  
---
# 2. ClusterRole 생성 - PSP 사용 권한
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: use-restricted-psp
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs: ['use']
  resourceNames: ['restricted-global']

---
# 3. 모든 인증된 사용자에게 PSP 사용 권한 부여 (전역 적용)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: all-authenticated-users-psp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: use-restricted-psp
subjects:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:authenticated # 'system:authenticated' 그룹은 모든 인증된 사용자를 포함

---
# 4. 시스템 서비스 계정을 위한 추가 ClusterRoleBinding (필요한 경우)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system-serviceaccounts-psp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: use-restricted-psp
subjects:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:serviceaccounts # 모든 SA
```



Namespace 단위로 제어

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```





### Pod Security Admission 전역 설정 (권장)

쿠버네티스 1.25부터 PSA 사용해야함.

```yaml
# Pod Security Admission을 사용하여 전역 설정하기
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
metadata:
  name: pod-security-configuration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    # 기본값 (모든 네임스페이스에 적용)
    defaults:
      enforce: "restricted"    # 위반하는 포드는 거부됨
      enforce-version: "latest"
      audit: "restricted"      # 위반 사항을 감사 로그에 기록
      audit-version: "latest"
      warn: "restricted"       # 위반 시 경고를 표시
      warn-version: "latest"
    
    # 특정 네임스페이스 제외 (필요한 경우)
    exemptions:
      # 네임스페이스 제외
      namespaces:
      - kube-system
      - cert-manager
```



**클러스터가 관리형 서비스인 경우 (EKS, GKE, AKS 등)**: 관리형 쿠버네티스 서비스를 사용하는 경우, 각 플랫폼에서 제공하는 방법으로 Pod Security Standards를 활성화해야 한다

- **GKE**: GKE에서는 'GKE Security Posture' 기능을 사용

- **EKS**: EKS 클러스터 생성/수정 시 PSA 활성화

- **AKS**: AKS 클러스터 보안 정책 업데이트

  



---

### PodSecurity / RBAC 차이

이 두 메커니즘은 상호 보완적이며, 완전한 보안을 위해 함께 사용해야 합니다

- **Pod Security**로 "어떤 종류의 파드가 실행될 수 있는지" 제어

  - 제한내용

    - 파드가 사용할 수 있는 보안 컨텍스트
    - 컨테이너 권한(privileged 모드 등)
    - 호스트 리소스 접근(네트워크, 파일시스템 등)
    - 컨테이너 사용자/그룹 설정
    - 볼륨 타입 제한

  - 작동 시점

    - 파드 생성 시점에 검증

    - Admission Controller로 작동

- **RBAC**로 "누가 파드를 생성할 수 있는지" 제어

  - 제한 내용
    - API 서버에 대한 접근 권한
    - 쿠버네티스 리소스(Pod, Service, ConfigMap 등)에 대한 작업 권한
    - 네임스페이스별 또는 클러스터 전체 접근 제어

  *  작동 시점

    - API 요청 시점에 인증/인가 검증

    - 리소스 CRUD 작업 전에 항상 검증

  *  적용 방식

    * Role/ClusterRole: 권한 정의
    * RoleBinding/ClusterRoleBinding: 사용자/그룹/서비스계정에 역할 할당

| 측면      | Pod Security                      | RBAC                                         |
| --------- | --------------------------------- | -------------------------------------------- |
| 목적      | 파드가 **어떻게** 실행되는지 제어 | **누가** 어떤 리소스에 접근할 수 있는지 제어 |
| 보호 대상 | 컨테이너 보안 설정                | API 리소스 접근                              |
| 적용 시점 | 파드 생성 시점                    | API 요청 시점                                |
| 적용 단위 | 파드/컨테이너                     | 사용자/그룹/서비스계정                       |
| 보안 계층 | 워크로드 보안                     | 인증 및 인가                                 |

------



### 파드 네트워크 격리



**CNI 호환성**: NetworkPolicy는 네트워크 플러그인이 지원해야 작동합니다.

- 지원 CNI: Calico, Cilium, Weave Net, Antrea 등
- 미지원 CNI: Flannel (기본 설정)

**정책 조합**: 여러 NetworkPolicy가 같은 파드에 적용될 경우, 모든 정책의 허용 규칙이 합쳐집니다(OR 조건).



**NetworkPolicy 주요 선택자**

* podSelector

  * 정책을 적용할 파드를 선택
  * 비어있는 경우(`{}`) 네임스페이스의 모든 파드에 적용

* namespaceSelector

  * 다른 네임스페이스의 파드를 선택
  * `{}`: 모든 네임스페이스

  - 특정 레이블: 해당 레이블이 있는 네임스페이스

- ipBlock

  - CIDR 표기법으로 IP 주소 범위를 지정합니다.

  - 클러스터 외부 리소스와의 통신에 유용

```yaml
apiVersion: networking.k8s.io/v1  # NetworkPolicy API 버전
kind: NetworkPolicy  # 리소스 타입
metadata:
  name: example-network-policy  # 네트워크 정책 이름
  namespace: default  # 적용할 네임스페이스
spec:
  podSelector:  # 이 정책을 적용할 파드 선택
    matchLabels:
      role: db  # label이 'role: db'인 파드에 적용
  policyTypes:  # 정책 타입 목록
  - Ingress  # 인그레스(들어오는) 트래픽 규칙
  - Egress   # 이그레스(나가는) 트래픽 규칙
  
  ingress:  # 들어오는 트래픽에 대한 규칙
  - from:  # 트래픽 소스 목록
    - ipBlock:  # IP 범위 기반 규칙
        cidr: 172.17.0.0/16  # 허용할 CIDR 범위
        except:  # 제외할 IP 범위
        - 172.17.1.0/24
    - namespaceSelector:  # 네임스페이스 기반 선택 , {}: 모든 네임스페이스
        matchLabels:
          project: myproject  # 'project: myproject' 레이블이 있는 네임스페이스의 모든 파드
    - podSelector:  # 파드 기반 선택
        matchLabels:
          role: frontend  # 'role: frontend' 레이블이 있는 파드
    ports:  # 허용할 포트 목록
    - protocol: TCP  # 프로토콜 (TCP 또는 UDP)
      port: 6379  # 허용할 포트 번호
  
  egress:  # 나가는 트래픽에 대한 규칙
  - to:  # 트래픽 목적지 목록
    - ipBlock:
        cidr: 10.0.0.0/24  # 허용할 목적지 IP 범위
    ports:
    - protocol: TCP
      port: 5978  # 허용할 목적지 포트
```



전체거부

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: restricted
spec:
  podSelector: {}  # 네임스페이스의 모든 파드에 적용
  policyTypes:
  - Ingress
  - Egress
  # from 또는 to 규칙이 없음 = 모든 트래픽 차단
```



---



### Zero Trust 네트워크 모델 (ZTN)

#### 📌 기본 개념

> *"신뢰하지 말고, 항상 검증하라(Never Trust, Always Verify)"*

Zero Trust는 기본적으로 모든 트래픽, 사용자, 기기를 신뢰하지 않고 지속적인 검증을 요구하는 보안 철학입니다.

#### 🔑 핵심 원칙

##### 1️⃣ 기본 불신 (Default Deny)

- 모든 접근 요청은 기본적으로 거부됨
- 명시적으로 허용된 트래픽만 통과 가능
- "차단이 기본값, 허용은 예외"

##### 2️⃣ 최소 권한 원칙

- 업무 수행에 필요한 최소한의 접근 권한만 부여
- 필요한 리소스에만, 필요한 시간 동안만 접근 허용
- "알 필요가 있는 경우에만" 접근 권한 부여

##### 3️⃣ 지속적인 검증

- 한 번 인증했다고 계속 신뢰하지 않음
- 모든 접근 시도마다 인증, 인가, 검증 실시
- 장치 상태, 위험 수준, 행동 패턴 계속 모니터링

##### 4️⃣ 마이크로 세분화 (Microsegmentation)

- 네트워크를 작은 보안 영역으로 분할
- 각 영역 간 엄격한 경계와 접근 제어 적용
- 침해 발생 시 측면 이동(lateral movement) 제한



#### 💻 쿠버네티스에서의 구현

##### 🔸 NetworkPolicy

- 마이크로서비스 간 통신을 세밀하게 제어
- 기본적으로 모든 통신 차단, 필요한 것만 허용
- 예시: 백엔드 서비스는 프론트엔드로부터만 특정 포트로 접근 허용

##### 🔸 서비스메시 (Service Mesh)

- Istio, Linkerd 등 활용
- 서비스 간 mTLS(상호 TLS) 적용으로 암호화 통신
- 트래픽에 대한 세밀한 정책과 인증 적용

##### 🔸 RBAC (역할 기반 접근 제어)

- 최소 권한 원칙에 따른 권한 부여
- 서비스계정, 사용자별 세밀한 권한 설정
- API 서버 접근 엄격하게 제한

##### 🔸 OPA/Gatekeeper

- 정책 기반 접근 제어 구현
- 리소스 생성/수정 시 자동 정책 검증
- 보안 정책 일관되게 적용

#### ✅ 이점

##### 🛡️ 보안 강화

- 경계 침해에도 내부 이동 제한으로 피해 최소화
- 지속적 검증으로 무단 접근 차단
- 내부자 위협에도 효과적 대응

##### 🌐 현대적 환경 적합성

- 클라우드, 하이브리드, 멀티클라우드 환경에 적합
- 원격 근무 환경에서 일관된 보안 제공
- 컨테이너, 마이크로서비스 아키텍처 보호에 효과적

##### 📊 가시성 향상

- 모든 네트워크 통신에 대한 상세한 로깅과 모니터링
- 이상 행동 탐지 용이
- 보안 인시던트 대응 역량 강화

##### 📜 규제 준수

- 세분화된 접근 제어로 규제 요구사항 충족
- 감사 및 증거 자료 확보 용이
- 데이터 보호 강화
