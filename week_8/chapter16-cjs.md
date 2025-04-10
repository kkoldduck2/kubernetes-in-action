## 16장 : 고급 스케줄링



### 테인트(Taint)란?

테인트는 특정 노드에 설정하는 키-값 쌍으로, 해당 노드에 파드가 스케줄링되는 것을 제한 

테인트가 있는 노드는 해당 테인트를 허용(tolerate)하는 파드만 스케줄링할 수 있음.

테인트는 `key=value:effect` 형식으로 구성

---



### 톨러레이션(Toleration)이란?

톨러레이션은 파드에 적용되는 설정으로, 특정 테인트가 있는 노드에도 스케줄링될 수 있도록 허용. 톨러레이션은 테인트의 "예외 처리" 같은 역할.

default 톨러레이션

1. `node.kubernetes.io/not-ready`: 노드가 준비되지 않은 상태일 때 

   ```yaml
   - key: node.kubernetes.io/not-ready
     operator: Exists
     effect: NoExecute
     tolerationSeconds: 300
   ```

2. `node.kubernetes.io/unreachable`: 노드에 접근할 수 없을 때

   ```yaml
   - key: node.kubernetes.io/unreachable
     operator: Exists
     effect: NoExecute
     tolerationSeconds: 300
   ```

---



### 테인트의 효과(Effect)

- `NoSchedule`: 톨러레이션이 없는 파드는 해당 노드에 스케줄링되지 않음
- `PreferNoSchedule`: 가능하면 톨러레이션이 없는 파드를 스케줄링하지 않음 (소프트한 제약)
- `NoExecute`: 톨러레이션이 없는 파드는 스케줄링되지 않으며, 이미 실행 중인 파드도 제거됨



**마스터 노드 보호**: 쿠버네티스 마스터 노드는 일반적으로 `node-role.kubernetes.io/master:NoSchedule` 테인트가 있어 일반 워크로드가 마스터 노드에 스케줄링되는 것을 방지합니다.

**특수 하드웨어 노드 관리**: GPU가 있는 노드에 `special-hardware=gpu:NoSchedule` 테인트를 설정하고, GPU를 필요로 하는 파드에만 톨러레이션을 부여하여 특수 리소스 관리를 할 수 있습니다.

**노드 유지보수**: 노드 유지보수 전에 `node.kubernetes.io/not-ready:NoExecute` 테인트를 적용하여 새로운 파드 스케줄링을 방지하고 기존 파드를 다른 노드로 이동시킬 수 있습니다.



---



### 노드 어피니티(Node Affinity) 

노드 셀렉터(Node Selector)는 쿠버네티스에서 파드가 특정 노드에 스케줄링되도록 제어하는 메커니즘입니다. 이 두 기능은 목적은 비슷하지만 유연성과 표현력에서 차이가 있음

#### **필수 노드 어피니티 **

##### **requiredDuringSchedulingIgnoredDuringExecution (Hard Requirement): **파드 스케줄링 시 반드시 만족해야 하는 조건

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
            - nvme
  containers:
  - name: nginx
    image: nginx
```

이 파드는 `disktype`이 `ssd` 또는 `nvme`인 노드에만 스케줄링됩니다.

#### **선호 노드 어피니티 **

##### **preferredDuringSchedulingIgnoredDuringExecution (Soft Preference):** 가능하면 만족하는 것이 좋지만, 필수는 아닌 조건

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - zone1
      - weight: 2
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: nginx
    image: nginx
```

이 파드는 가능하면 `zone=zone1`인 노드와 `disktype=ssd`인 노드에 스케줄링되며, `disktype=ssd`에 더 높은 가중치(2)가 부여됩니다.





---



### 파드 어피니티

쿠버네티스에서 파드 간의 위치 관계를 정의하여 스케줄링을 제어하는 기능, 특정 파드들이 서로 가까이 또는 멀리 배치되도록 설정

`IgnoredDuringExecution` 부분은 파드가 이미 실행 중일 때는 조건이 변경되어도 파드를 재스케줄링하지 않음을 의미합니다.



#### topologyKey

`topologyKey`는 파드 어피니티/안티 어피니티 룰 적용 정의(노드의 레이블 키를 참조)

- `kubernetes.io/hostname`: 노드 레벨의 토폴로지
- `topology.kubernetes.io/zone`: 가용 영역(AZ) 레벨의 토폴로지
- `topology.kubernetes.io/region`: 리전 레벨의 토폴로지



#### 필수 파드 어피니티

**requiredDuringSchedulingIgnoredDuringExecution**: 반드시 충족해야 하는 조건 (Hard Requirement)

`app=frontend`인 파드와 동일한 노드에 `app=cache` 파드를 배치

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache
  labels:
    app: cache
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - frontend
        topologyKey: "kubernetes.io/hostname"
  containers:
  - name: redis
    image: redis
```



#### 선호도 기반 안티 어피니티

**preferredDuringSchedulingIgnoredDuringExecution**: 가능하면 충족하는 것이 좋지만, 필수는 아닌 조건 (Soft Preference)

가능하면 다른 노드에 배치하되, 불가능한 경우에는 같은 노드에도 배치 가능

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  replicas: 5
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - api
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: api-container
        image: api-image
```



---



