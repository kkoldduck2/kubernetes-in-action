## 1. 인증 이해

- API 서버를 하나 이상의 인증 플러그인으로 구성할 수 있다. (인가 플러그인도 마찬가지)
- API 서버가 요청을 받음 → 인증 플러그인 목록을 거치면서 요청 전달 → 각각의 인증 플러그인이 요청을 검사해서 보낸 사람이 누구인지 밝혀냄 → 누군인지 처음으로 추출한 플러그인은 **사용자 이름, 사용자 ID, 클라이언트가 속한 그룹**을 API 서버 코어에 반환 → API 서버는 나머지 인증 플러그인의 호출을 중지하고, 인가 단계를 진행

### **1) 사용자와 그룹**

**인증 플러그인은 인증된 사용자의 username과 group(s)를 반환**한다. 

→ 이 정보는 쿠버네티스에 저장되지 않음

→ 이 정보를 바탕으로 사용자가 작업을 수행할 권한이 있는지 여부가 체크됨

- 사용자
    - 쿠버네티스는 API 서버에 접속하는 두 종류의 클라이언트를 구분함. 인증 플러그인을 사용해 인증됨.
        - 실제 사람 (사용자) : SSO와 같은 외부 시스템에 의해 관리돼야 함.
        - 파드 (파드 내부에서 실행되는 애플리케이션) : 서비스 어카운트 라는 매커니즘을 사용하며, 클러스터에 **서비스 어카운트 리소스로 생성되고 저장**된다.
- 그룹
    - 휴먼 사용자 + 서비스 어카운트는 하나 이상의 그룹에 속할 수 있다.
    - (authentication 플러그인은 groups, username, userId를 반환함. )
    - 그룹은 여러 사용자들에게 한번에 권한을 부여하기 위해 사용됨.
    - 플러그인이 반환하는 그룹은 그냥 그룹 이름을 나타내는 문자열임. 근데 buit-in 그룹은 특별한 의미를 가진다.
        - system:anauthenticated 그룹 - 어떤 인증 플러그인에도 클라이언트를 인증할 수 없는 요청에 사용됨
        - system:authenticated 그룹 - 성공적으로 인증된 사용자에게 자동으로 할당됨
        - system:serviceaccounts 그룹 - 시스템의 모든 서비스어카운트를 포함함
        - system:serviceaccounts:<namespace> - 특정 네임스페이스의 모든 서비스 어카운트를 포함함

### **2) 서비스 어카운트**

- **서비스 어카운트 소개**
    - 클라이언트가 API 서버에서 작업을 수행하기 위해서는 자신을 인증해야함.
    - 이를 위해 시크릿 볼륨으로 각 컨테이너의 파일시스템에 마운트된 /var/run/secrets/kubernetes.io/serviceaccount/token 파일의 내용을 전송해 파드를 인증함.
    - 이 파일이 의미하는 것은?
        - 모든 파드는 **파드에서 실행 중인 애플리케이션의 아이덴티티를 나타내는 서비스 어카운트**와 연계되어 있음
        - 이 토큰 파일은 **서비스어카운트의 인증 토큰**을 갖고 있다.
        - 즉, 모든 파드는 이 토큰 파일에 있는 서비스어카운트 인증 토큰을 이용 → 해당 파드에서 실행중인 애플리케이션의 아이덴티티를 증명한다.
    - 애플리케이션이 이 토큰을 이용해서 API 서버에 접속 → 인증 플러그인이 서비스 어카운트를 인증하고, 서비스 어카운트의 사용자 이름을 API 서버 코어로 전달
        
        ```yaml
        # 서비스 어카운트의 사용자 이름 형식
        system:serviceaccount:<namespace>:<serivce account name>
        ```
        
    - API 서버는 이 사용자 이름을 인가 플러그인에 전달 → 인가 플러그인은 애플리케이션이 수행하려는 작업을 서비스어카운트에서 수행할 수 있는지를 결정함.
<img width="870" alt="image" src="https://github.com/user-attachments/assets/0189745e-f58e-40ee-ab6a-340e7d95a85d" />


- **서비스어카운트 리소스**
    - 서비스어카운트는 파드, 시크릿, 컨피그맵 등과 같은 리소스
    - 개별 네임스페이스로 범위가 지정됨
    - 각 네임스페이스마다 default 서비스어카운트가 자동으로 생성된다 (이것이 그동안 파드가 사용한 서비스어카운트다). 필요할 경우 서비스어카운트 추가 가능
    - 각 파드는 오직 하나의 서비스와 연결됨, 여러 파드가 같은 서비스어카운트를 사용할 수 있음
    - 파드는 같은 네임스페이스의 서비스어카운트만 사용할 수 있음
    
    <img width="870" alt="image" src="https://github.com/user-attachments/assets/64698d14-ed0b-4230-8244-e8b29aa5ebc7" />

    - 서비스어카운트가 인가와 어떻게 밀접하게 연계돼 있는가?
        - Pod manifest에 서비스어카운트의 이름을 지정해 파드에 서비스어카운트를 할당할 수 있음
        - 만약 명시적으로 할당하지 않을 경우, 파드는 네임스페이스에 있는 default 서비스 어카운트를 사용함
        - 파드에 서로 다른 서비스 어카운트를 할당함으로써 **각 파드가 액세스할 수 있는 리소스를 제어**할 수 있다.
        - API 서버가 인증 토큰을 이용해 클라이언트를 인증한 다음, (클러스터 관리자가 구성한) 시스템 전체의 인가 플러그인에서 관련 서비스어카운트가 요청된 작업을 수행할 수 있는지 여부를 결정한다.
        - 사용 가능한 인가 플러그인 중 하나는 역할 기반 액세스 제어(RBAC) 플러그인이다.

- default 서비스 어카운트를 사용하지 않고 새로 생성하는 이유?
    - 클러스터 보안 때문
    - 파드가 꼭 필요한 권한만 갖도록 RBAC 설정 가능

- 서비스어카운트 생성 및 파드 할당
    - 서비스어카운트 생성
    
    ```bash
    kubectl create serviceaccount <서비스어카운트_이름>
    kubectl create serviceaccount my-sa # default 네임스페이스에 my-sa라는 서비스어카운트 생성
    
    # 다른 네임스페이스 생성
    kubectl create serviceaccount my-sa -n my-namespace
    ```
    
    - 파드에 서비스어카운트 연결
    
    ```yaml
    # 이 pod는 실행 시 my-sa를 통해 API Server와 통신하게 됨
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-app-pod
    spec:
      serviceAccountName: my-sa   # ← 여기 연결
      containers:
      - name: my-container
        image: nginx
    ```
    

## 2. 역할 기반 액세스 제어로 클러스터 보안

- 서비스 어카운트에 권한 부여 (RBAC)
    - 서비스 어카운트를 만들었다고 자동으로 권한이 생기지는 않음
    - 필요한 역할 (Role/ClusterRole)을 할당해줘야 함
    - 예) `my-sa`가 **Pod를 조회(get/list/watch)** 할 수 있도록 Role과 RoleBinding 생성:
    - Role 생성
    
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: pod-reader
    rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "list", "watch"]
    ```
    
    - RoleBinding (serviceaccount와 role을 바인딩)
    
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: pod-reader-binding
    subjects:
    - kind: ServiceAccount
      name: my-sa
      namespace: default   # SA가 속한 네임스페이스
    roleRef:
      kind: Role
      name: pod-reader
      apiGroup: rbac.authorization.k8s.io
    ```
    
    - `my-sa`는 **해당 네임스페이스의 Pod 정보만 읽을 수 있는 권한**을 갖게됨

### 클러스터롤과 클러스터롤바인딩

- ClusterRole과 Role 차이
    - Role → 특정 네임스페이스의 pod만 접근
    - ClusterRole → 클러스터 전체의 node 리소스를 조회하거나, 모든 네임스페이스의 pod를 조회

| 항목 | **Role** | **ClusterRole** |
| --- | --- | --- |
| 적용 범위 | 특정 **네임스페이스 내부** | **클러스터 전체 범위** 또는 모든 네임스페이스 |
| 사용 목적 | 네임스페이스 리소스 제어 | - 클러스터 리소스 (Node, PersistentVolume 등) - 모든 네임스페이스 리소스 제어 |
| 바인딩 대상 | RoleBinding | **RoleBinding or ClusterRoleBinding 둘 다 가능** |
- 클러스터롤
    - 모든 네임스페이스의 pod를 읽을 수 있는 권한
    
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: cluster-pod-reader
    rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "list", "watch"]
    ```
    
- 클러스터롤 바인딩
    - my-sa 서비스어카운트에게 ClusterRole의 권한을 부여
    - 즉, my-sa는 모든 네임스페이스의 pod를 get/list/watch 할 수 있음
    
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: read-pods-global-binding
    subjects:
    - kind: ServiceAccount
      name: my-sa
      namespace: default
    roleRef:
      kind: ClusterRole
      name: cluster-pod-reader
      apiGroup: rbac.authorization.k8s.io
    
    ```
    

- 정리

| 개념 | 설명 |
| --- | --- |
| ClusterRole | 클러스터 전역에서 사용할 수 있는 권한 정의 |
| ClusterRoleBinding | 특정 사용자/SA/그룹에게 ClusterRole을 부여 |
| Role | 특정 네임스페이스 내 권한만 정의 |
| RoleBinding | 특정 네임스페이스 내에서 권한을 부여 |
