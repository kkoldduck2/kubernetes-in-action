> 만약 공격자가 API 서버에 접근하게 되면, 그들이 자신의 코드를 컨테이너 이미지로 말아서 파드에서 실행시키면 무엇이든 할 수 있을까?
컨테이너는 동일 노드에 떠있는 컨테이너와 분리되어 있지 않나?
> 

<aside>
💡

1. 어떻게 파드가 자신이 떠있는 노드의 자원에 접근할 수 있도록 하는지?
2. 사용자가 자신의 파드로 무엇이든지 할 수 없도록 어떻게 클러스터를 구성할 것인지?
3. 파드 간 네트워크를 어떻게 보호할 것인지?
</aside>

## 1. 파드에서 호스트 노드의 네임스페이스 사용

- 파드의 컨테이너는 일반적으로 **별도의 리눅스 네임스페이스**에서 실행됨 → 따라서 프로세스가 다른 컨테이너  or 노드의 기본 네임스페이스에서 실행 중인 프로세스와 격리됨

### 1) 파드에서 노드의 네트워크 네임스페이스 사용

- 특정 파드 (일반적으로 시스템 파드)는 호스트의 기본 네임스페이스에서 작동해야함 → 노드의 리소스와 장치를 읽고 조작하기 위해
- 파드 스펙에서 **hostNetwork 속성을 true로 설정**하면 됨
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: hostnetwork-example
    spec:
      hostNetwork: true     # ← 이 부분이 핵심!
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
    ```
    
- 이 경우 파드의 네트워크 인터페이스가 아니라 노드의 네트워크 인터페이스를 사용하게 됨
- 이는 파드가 **자체 IP 주소를 갖는게 아니라**, 포트를 바인드하는 프로세스가 실행될 경우 해당 프로세스는 **노드의 포트에 바인딩**된다는 것임
<img width="774" alt="image" src="https://github.com/user-attachments/assets/2f88fdf7-3c19-48b4-ac7b-89e1ea99af7d" />

### 2) 호스트 네트워크 네임스페이스를 사용하지 않고 호스트 포트에 바인딩

- 파드는 hostNetwork 옵션으로 노드의 기본 네임스페이스의 포트에 바인딩할 수 있지만, 여전히 고유한 네트워크 네임스페이스를 가짐 
→ spec.containers.ports 필드 안에 hostPort 속성을 사용해서 구현
- 서비스 NodePort와 hostPort를 사용하는 파드의 차이?
    - hostPort
        - 노드의 특정 포트로 들어오는 요청은 해당 노드에서 실행 중인 pod로 다이렉트하게 전달된다.
        - 노드 포트는 해당 파드를 실행하는 노드에만 바인딩됨
    - NodePort
        - 노드의 포트로 들어오는 요청은 다른 노드에 있을 수 있는 임의의 파드로 랜덤하게 전달됨.
        - 파드를 실행하지 않는 노드에서도 모든 노드의 포트를 바인딩
<img width="774" alt="image" src="https://github.com/user-attachments/assets/ac6cb0c2-fde5-4c86-b926-71f23ed50c14" />

- 파드가 특정 호스트 포트를 사용하는 경우, 노드별로 파드 인스턴스 한개씩 밖에 스케줄링 될 수 없음 (두 프로세스가 동일한 호스트에 바인딩될 수 없으므로)
    - 스케줄러는 파드를 스케줄할 때 이를 고려함. 따라서 여러 파드를 동일 노드에 스케줄링하지 않음
    - 만약 노드 3개가 있고 파드 레플리카 4개를 배포하려고 시도하면, 3개만 스케줄링되고 하나는 pending 상태가 됨
<img width="774" alt="image" src="https://github.com/user-attachments/assets/352af999-7639-4cfa-9f1d-0e2466c28166" />

### 3) 노드의 PID와 IPC 네임스페이스 사용

- hostNework 옵션과 유사한 파드 스펙 속성으로 hostPID와 hostIPC가 있다.
- 이를 true로 설정하면 파드의 컨테이너는 노드의 PID와 IPC 네임스페이스를 사용함
    
    → 컨테이너에서 실행중인 프로세스가 노드의 다른 프로세스를 보거나 IPC로 이들과 통신할 수 있도록 한다. 
    

## 2. 컨테이너의 보안 컨텍스트 구성

- securityContext 속성
- Pod나 컨테이너가 실행될 때의 **보안 설정(사용자/권한/파일 권한/특권 모드 등)** 을 정의하는 설정
- 보안 컨텍스트는 Pod 단위 또는 Container 단위로 설정할 수 있음
- 보안 컨텍스트를 지정하지 않고 파드 실행할 경우
    - 사용자 ID(uid) 0, 그룹 ID(gid) 0인 루트 사용자로 실행됨
<img width="837" alt="image" src="https://github.com/user-attachments/assets/b14618dc-85a5-467a-a694-36bf5e7c2c6d" />

### 1) 주요 항목 설명 (컨테이너 기준)

| 필드 | 설명 |
| --- | --- |
| `runAsUser` | 컨테이너가 실행될 때 사용할 **리눅스 사용자 ID (UID)** |
| `runAsGroup` | 컨테이너가 사용할 **리눅스 그룹 ID (GID)** |
| `runAsNonRoot` | **루트 계정이 아닌 사용자로만 실행 허용** 여부 (true 추천) |
| `privileged` | **컨테이너에 호스트 권한 전체 부여** (보안상 위험. 대부분 false) |
| `allowPrivilegeEscalation` | 권한 상승(sudo 등) 허용 여부 |
| `readOnlyRootFilesystem` | 루트 파일시스템을 읽기 전용으로 설정 |
| `capabilities` | 리눅스 커널의 capability 설정 (특정 권한만 부여/제거) |
| `seccompProfile` | 시스템 콜 필터링 프로파일 설정 (고급 보안 설정) |

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  containers:
  - name: secure-container
    image: nginx
    securityContext:
      runAsUser: 406          # Guest 사용자
      runAsGroup: 100        # Users
      runAsNonRoot: true
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop:
          - ALL
```

- UID 405 / GID 100 으로 실행
- 루트 권한 금지 (`runAsNonRoot: true`)
    - 컨테이너가 격리되어있다 한들, 컨테이너 안에서 UID 0 = 루트는 리눅스 커널 레벨에서 여전히 루트 계정
    - 특권 컨테이너 / 호스트 네트워크, 호스트 PID/IPC 사용 중 / 호스트의 디렉토리 (/var, /etc)를 마운트해서 쓰는 경우 → 컨테이너의 루트 권한이 호스트를 직접 건드릴 수 있는 위험이 생김
- 루트 파일시스템을 읽기 전용으로
- 권한 상승 차단
- 리눅스 Capabilities 모두 제거

- 특권 모드에서 파드 실행
    - 컨테이너가 리눅스 커널의 제약 없이 **호스트 시스템에 거의 직접 접근 가능한 상태**로 실행되는 모드
    - 예) kube-proxy 파드 : 서비스를 작동 시키려 노드의 iptables 규칙을 수정함
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: privileged-pod
    spec:
      containers:
      - name: privileged-container
        image: busybox
        command: ["sleep", "3600"]
        securityContext:
          privileged: true    # ← 특권 모드 설정
    ```
    

- 컨테이너에 개별 커널 기능 추가
    - 실제로 필요한 커널 기능만 액세스하도록 하는 기능
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: cap-example
    spec:
      containers:
      - name: my-container
        image: busybox
        command: ["sleep", "3600"]
        securityContext:
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]   # ← 이 부분!
            drop: ["ALL"]                   # 기본 권한 제거 후 필요한 것만 추가
    ```
    
    - `drop: ["ALL"]`: 기본 권한 다 제거 (보안 강화)
    - `add: ["NET_ADMIN", "NET_RAW"]`: 필요한 커널 기능만 추가
    - 위와 같은 설정에서 할 수 있는 것:
        
         - `CAP_NET_ADMIN` 없으면 안 됨. 위처럼 capability를 추가해야 이런 네트워크 작업이 가능
        
        ```yaml
        ip link add dummy0 type dummy   # 가상 네트워크 인터페이스 생성
        ```
        

- 컨테이너가 다른 사용자로 실행될 때 볼륨 공유 (파드 수준 보안 컨텍스트 옵션 설정)
    - 하나의 파드에서 실행되는 두 컨테이너가 runAsUser 옵션을 통해 두 명의 다른 사용자로 실행됨
    - 이 두 컨테이너가 볼륨으로 파일을 공유하는 경우 반드시 서로의 파일을 읽거나 쓸 수 있는 것은 아님
    - supplementalGroups 속성을 지정해 실행 중인 사용자 ID에 상관 없이 파일을 공유할 수 있음
    - supplementalGroups란? :
        - 컨테이너 안에서 실행되는 프로세스에 추가 그룹 ID (GID)를 부여하는 옵션
        - 컨테이너가 여러 그룹에 속해야 하거나, **공유 볼륨의 그룹 권한에 접근할 수 있게 하려는 경우** 사용
        - fsGroup, supplementalGroups 속성을 사용함
    - `fsGroup` vs `supplementalGroups` 차이?
    
    | 항목 | fsGroup | supplementalGroups |
    | --- | --- | --- |
    | 목적 | **볼륨(파일시스템)의 그룹 소유자 설정** | **컨테이너 프로세스가 소속될 추가 그룹 부여** |
    | 적용 대상 | 주로 **볼륨 mount 시 파일 GID 설정** | 프로세스의 **그룹 소속 추가 (권한 부여)** |
    | 공유 볼륨 접근 | O (퍼미션 자동 변경됨) | O (이미 존재하는 퍼미션 기반 접근) |
    - `fsGroup`은 파일의 그룹을 바꾸고,
    - `supplementalGroups`는 **컨테이너 프로세스에 그룹을 더 붙여주는 것**
    - 예) supplementalGroups 사용
        
        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: supplemental-groups-pod
        spec:
          securityContext:
            supplementalGroups: [2000, 3000]   # ← 추가 그룹 GID 부여
          containers:
          - name: app
            image: busybox
            command: ["sleep", "3600"]
            securityContext:
              runAsUser: 1000
          volumes:
          - name: my-volume
            emptyDir: {}
        ```
        
        - 이 Pod의 컨테이너 프로세스는 UID 1000 이지만,
        - GID 2000, 3000 도 함께 소속되어 있어서,
        - **볼륨에 그룹 소유자가 2000이더라도 접근 가능**
    - 예) fsGroup 사용
        
        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: fsgroup-example
        spec:
          securityContext:
            fsGroup: 2000           # ← 이 그룹(GID)이 볼륨 파일들의 그룹으로 설정됨
          containers:
          - name: my-container
            image: busybox
            command: ["sleep", "3600"]
            securityContext:
              runAsUser: 1000       # ← 컨테이너 프로세스는 UID 1000으로 실행
            volumeMounts:
            - name: my-volume
              mountPath: /data
          volumes:
          - name: my-volume
            emptyDir: {}
        ```
        
        - `runAsUser: 1000` → 컨테이너는 UID 1000으로 실행됨 (비루트 사용자)
        - `fsGroup: 2000` → 쿠버네티스가 `/data` 볼륨 디렉토리의 **그룹 소유자(GID)를 2000**으로 설정함. 즉, 쿠버네티스가 **볼륨의 모든 파일에 GID 2000을 설정**해주고, 동시에 **컨테이너 프로세스가 GID 2000도 함께 포함된 그룹**으로 실행되게 만들어줌
        - 결과적으로, `/data` 디렉토리의 파일들이 `GID 2000`으로 설정되고, **컨테이너 프로세스는 그 그룹으로 접근 가능**해짐
        

## 3. 파드 네트워크 격리
- NetworkPolicy 설정을 통해 파드 간의 네트워킹을 제한할 수 있다. (안될 수도 있음. 네트워킹 플러그인에 따라 다름.)
- ingress와 egress 규칙으로 설정할 수 있음

### 1) 네임스페이스에서 네트워크 격리 사용

- 아래와 같이 특정 네임스페이스에서 default-deny NetworkPolicy를 생성하면 모든 클라이언트가 해당 네임스페이스의 모든 파드에 연결할 수 없다.



### 2) 네임스페이스의 일부 클라이언트 파드만 서버 파드에 연결하도록 허용

- 클라이언트가 네임스페이스의 파드에 연결할 수 있게 하려면 파드에 연결할 수 있는 대상을 명시적으로 지정해야 함
- 예) 네임스페이스 foo에서 실행되는 PostgreSQL 데이터베이스 파드와 데이터베이스를 사용하는 웹 서버 파드가 있음. 다른 파드도 같은 네임스페이스에 있지만 데이터베이스에 연결하는 것을 원하지 않음.

<img width="832" alt="image" src="https://github.com/user-attachments/assets/3ef73578-9e81-4522-93ee-acfd45def954" />

- app=webserver 레이블이 있는 파드가 app=database 레이블이 있는 파드의 포트 5432에만 연결될 수 있음

<img width="832" alt="image" src="https://github.com/user-attachments/assets/00bf6935-88f3-4e10-92fd-bb7022f31de8" />

### 3) 쿠버네티스 네임스페이스 간 네트워크 격리

- 여러 테넌트가 동일한 쿠버네티스 클러스터를 사용하는 경우 → 각 테넌트는 여러 네임스페이스를 사용할 수 있음, 각 네임스페이스는 해당 테넌트를 지정하는 레이블이 존재
- 예) Manning 테넌트가 있고 그 안에 모든 네임스페이스는 tenant: manning 레이블이 지정되어 있음
    - 그중 하나의 네임스페이스 안에서 같은 테넌트 안에 있는 모든 네임스페이스의 파드가 사용할 수 있는 Shopping Cart 마이크로서비스를 실행함.
    - 다른 테넌트가 해당 마이크로서비스에 액세스하는 것을 원하지 않음

<img width="832" alt="image" src="https://github.com/user-attachments/assets/cf3259b8-8ffa-433b-93ca-b88066dcdfd8" />

### 4) 파드의 아웃바운드 트래픽 제한

- 이그레스 규칙으로 아웃바운드 트래픽을 제한할 수 있다.
<img width="832" alt="image" src="https://github.com/user-attachments/assets/d476c72e-484c-49bd-8df6-896dfc1f517d" />


---
