### 1. **테인트(Taint)와 톨러레이션(Toleration)**

- **테인트(Taint)**: 노드에 제약을 설정하여, 특정 조건을 만족하지 않으면 파드가 해당 노드에 배치되지 않게 한다.
- **톨러레이션(Toleration)**: 파드가 특정 테인트를 허용할 수 있게 해주는 설정이다. 예를 들어, 파드가 "NoSchedule" 테인트가 설정된 노드에 배치될 수 있도록 허용하려면, 해당 파드가 "NoSchedule"을 톨러레이트(tolerate)하도록 설정해야 한다.
    - 예1) 노드A 에 `NoSchedule` 테인트를 추가한다고 가정해보자. 이제 이 노드에 파드를 배치하려면 해당 파드가 `NoSchedule` 테인트를 **허용**해야 한다. 즉, 노드A에 배치할 파드가 `NoSchedule`을 톨러레이트(tolerate)하지 않으면, 그 파드는 `node-1`에 배치되지 않는다.
    
    ```yaml
    # exampleKey=exampleValue 테인트를 가진 노드에 배치될 수 있는 파드의 예시
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-pod
    spec:
      tolerations:
      - key: "exampleKey"
        operator: "Equal"
        value: "exampleValue"
        effect: "NoSchedule"
    ```
    
    - 예2) 시스템 파드의 toleration이 master node의 taint와 일치하기 때문에 마스터 노드에 스케줄링 된다.
    
    <img width="783" alt="image" src="https://github.com/user-attachments/assets/0bb0f465-f4bd-4df7-acad-617b97962bd2" />

- **세 가지 유형의 테인트**:
    - **NoSchedule**: 해당 노드에 파드를 배치하지 않는다. (강력한 제약)
    - **Prefer-NoSchedule**: 파드가 해당 노드에 배치되지 않기를 "선호"하지만, 꼭 그렇지는 않다. (다소 유연한 제약)
    - **NoExecute**: 이미 해당 노드에 배치된 파드를 삭제하거나, 노드에 접근할 수 없을 때 파드를 이동시키도록 한다. 예를 들어, 노드가 준비되지 않거나 네트워크에 접근할 수 없을 때, 파드를 다시 다른 노드로 배치해야 할 때 사용된다.
    - NoScedule과 NoExecute 차이
        - **NoSchedule**: **새로 배치될 파드**만 해당 노드에 배치되지 않음. **기존 파드**에는 영향을 미치지 않음.
        - **NoExecute**: **새로운 파드**와 **기존 파드** 모두 영향을 받음. 기존 파드는 해당 노드에서 **종료되거나 이동**되며, 새 파드는 배치되지 않음.

### 2. **노드 어피니티(Node Affinity)**

- **노드 어피니티**는 파드를 특정 노드에 배치하도록 설정하는 방법이다.
- **필수 요구 사항(Must)**
    - 파드가 특정 노드에 배치되도록 강제로 요구할 수 있다. 예를 들어, 특정 라벨이 있는 노드에만 파드를 배치하도록 요구할 수 있다.
- **선호 사항(Preferred)**
    - 특정 노드에 배치되기를 "선호"하지만, 꼭 그 노드에 배치할 필요는 없다. 예를 들어, 고성능 CPU를 가진 노드에 배치하고 싶지만, 그런 노드가 부족하다면 다른 노드에 배치될 수도 있다.
- 예) GPU가 필요한 파드를 **GPU가 있는 노드**에만 배치하고 싶다고 가정해보자. 이 경우, GPU가 있는 노드를 `gpu=true`라는 라벨로 표시하고, 파드에는 해당 노드에 배치되도록 노드 어피니티를 설정할 수 있다.

```yaml
# 이 파드는 gpu=true 라벨을 가진 노드에서만 실행된다.
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: "gpu"
            operator: In
            values:
            - "true"
```

### 3. **파드 어피니티(Pod Affinity)**

- **파드 어피니티**는 같은 애플리케이션의 파드를 "같은 노드"에 배치하려는 규칙이다.
- **파드 레이블**을 사용해서 같은 서비스나 애플리케이션을 하나의 노드에 배치할 수 있다.
- 예를 들어, **웹 애플리케이션**의 파드들은 서로 가까운 노드에 배치되기를 원한다고 가정해보자. 이 경우 파드 어피니티를 사용하여 같은 레이블을 가진 다른 파드와 같은 노드에 배치하도록 설정할 수 있다.

```yaml
# 이 설정은 app=web-app 레이블을 가진 파드들이 같은 노드에 배치되도록 한다.
apiVersion: v1
kind: Pod
metadata:
  name: web-app-pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:  # 
        labelSelector:
          matchLabels:
            app: "web-app"
        topologyKey: "kubernetes.io/hostname"
```

- requiredDuringSchedulingIgnoredDuringExecution
    - **스케줄링 시 필수 조건**을 설정한다.
    - 이 조건을 만족해야만 **파드가 해당 노드에 배치**된다.
    - **실행 중인 파드에는 영향을 미치지 않음**. 즉, 이미 실행 중인 파드가 실행 중인 노드에서 변경되지 않는다.
- preferredDuringSchedulingIgnoredDuringExecution
    - **선호하는 조건**을 설정한다.
    - 이 조건을 만족하는 것이 좋지만, 만족하지 않아도 파드를 배치할 수 있다.

**topologyKey**:

- 파드가 배치될 위치를 결정할 때 중요한 역할을 한다. 일반적으로는 노드의 **호스트 이름**이나 **존(availability zone)** 등을 기준으로 설정할 수 있다.
- 이를 통해 **파드를 어떤 기준에 맞춰 배치할지** 결정할 수 있다. 예를 들어, **같은 노드에 배치**할지, 아니면 **같은 지역(zone)** 내에 배치할지를 설정할 수 있다.
- 예
    - `"kubernetes.io/hostname"`: **같은 노드**에 파드를 배치
    - `"failure-domain.beta.kubernetes.io/zone"`: **같은 가용 영역(zone)**에 파드를 배치
    - `"topology.kubernetes.io/region"`: **같은 지역**에 파드를 배치

### 4. **파드 안티-어피니티(Pod Anti-Affinity)**

- **파드 안티-어피니티**는 "특정 파드를 서로 멀리 배치"하는 규칙이다. 즉, 특정 파드가 다른 파드와 같은 노드에 배치되지 않도록 하는 설정이다.
- 예를 들어, 데이터베이스와 웹서버가 같은 노드에 배치되지 않도록 설정할 수 있다. 이 설정은 장애 발생 시 영향을 최소화하려는 목적이 있다.
- 예: 웹서버와 데이터베이스는 서로 다른 노드에 배치되도록 하라. (`app=db`와 `app=web` 파드를 다른 노드에 배치하도록 설정)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        labelSelector:
          matchLabels:
            app: "web"
        topologyKey: "kubernetes.io/hostname"

```

### 5. **필수 요구사항 vs 선호 사항**

- **필수 요구사항(Must)**: 해당 조건을 반드시 충족해야만 파드가 배치된다.
    - 예: 특정 노드 어피니티 요구사항을 만족해야 한다.
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: gpu-required-pod
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: "gpu"
                operator: In
                values:
                - "true"
    
    ```
    
- **선호 사항(Preferred)**: 해당 조건을 충족하면 좋지만, 충족되지 않아도 배치된다.
    - 예: 고성능 노드를 선호하지만, 꼭 그 노드에 배치할 필요는 없다.
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: cpu-preferred-pod
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: "cpu"
                operator: In
                values:
                - "high-performance"
    
    ```
    

### 요약:

- **테인트와 톨러레이션**을 사용하여 특정 노드에 파드를 배치하지 않거나, 허용할 수 있다.
- **노드 어피니티**는 특정 노드에 파드를 배치할 수 있도록 도와주고, **파드 어피니티**와 **파드 안티-어피니티**는 파드끼리의 관계를 기준으로 배치 규칙을 설정할 수 있다.
- **선호**와 **필수 요구사항**을 이용해 더 유연하게 배치 조건을 설정할 수 있다.

이렇게 하면 쿠버네티스에서 파드를 더 정교하게 관리하고, 자원 활용을 최적화할 수 있다.
