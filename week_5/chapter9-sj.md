### Deployments란?

- 애플리케이션을 안정적으로 배포, 업데이트, 롤백할 수 있도록 도와주는 컨트롤러
- 원하는 상태를 선언하면 쿠버네티스가 이를 자동으로 유지해줌

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3  # 실행할 파드 개수
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app-container
          image: my-app:1.0  # 배포할 컨테이너 이미지
          ports:
            - containerPort: 80
```

- `replicas: 3` → 3개의 파드를 실행하도록 지정.
- `selector.matchLabels.app: my-app` → 어떤 파드를 관리할지 지정.
- `template` → 생성할 파드의 정의.
- `image: my-app:1.0` → 컨테이너 이미지 지정.

- **Deployment 구조**
    - 내부적으로 ReplicaSet을 관리하면서, Pod를 생성하고 업데이트함
    - 디플로이먼트 → ReplicaSet → Pod
    - 즉, 디플로이먼트가 원하는 상태를 정의 하면, 쿠버네티스는 ReplicaSet을 생성하여 그 상태를 유지

- **Deployment 주요 기능**
    - 애플리케이션 배포
        - 사용자가 정의한 개수만큼의 파드를 실행하고 유지함
    - 롤링 업데이트
        - 기존 파드를 하나씩 교체하면서 중단 없이 애플리케이션을 업데이트할 수 있음
    - 롤백
        - 새로운 버전에서 문제가 발생하면 이전 버전으로 쉽게 되돌릴 수 있음
    - 자동 복구
        - 장애가 발생한 파드를 자동으로 감지하고, 새로운 파드를 생성하여 서비스 지속성을 보장함

### ReplicationSet이 아니라 Deployment를 사용하는 이유?

- ReplicaSet 역시 Pod를 원하는 개수만큼 유지하고 관리하는 역할을 함.
- 업데이트(롤링 업데이트) 및 롤백을 위해서는 Deployment를 사용하는 것이 더 좋다.
    - ReplicaSet
        - Pod 개수를 유지하는 역할만 함. 업데이트(버전 변경) 기능이 없음
        - 업데이트하기 위해서는 기존 ReplicaSet을 삭제하고 새로운 ReplicaSet을 만들어야 함
        - 즉, 수동으로 관리해야함
    - Deployment
        - 롤링 업데이트와 롤백 기능 제공
        - 기존 파드를 하나씩 교체하면서 무중단 배포(롤링 업데이트) 가능
        - 문제가 발생하면 한 줄 명령어로 롤백(undo) 가능
    - 또한 이전 ReplicaSet을 자동으로 관리해줌
    - 새로운 버전을 배포할 때 기존 ReplicaSet을 필요에따라 유지하거나 삭제할 수 있음
    - 업데이트할 때마다 새로운 ReplicaSet을 생성해 이전 버전의 히스토리를 보존하므로 필요하면 롤백 가능
- 롤링 업데이트
    - 아래 명령어 실행하면, 쿠버네티스가 자동으로 기존 파드를 하나씩 새 버전으로 변경해줌
    
    ```bash
    kubectl set image deployment my-app my-app-container=my-app:2.0
    ```
    
- 롤백
    
    ```bash
    kubectl rollout undo deployment my-app
    ```
    

### ReplicaSet 이전 버전 관리 & 새로운 버전 배포 시 유지/삭제 기능

- 이전 ReplicaSet을 자동으로 관리해줌
- 새로운 버전을 배포할 때 기존 ReplicaSet을 필요에따라 유지하거나 삭제할 수 있음
- 업데이트할 때마다 새로운 ReplicaSet을 생성해 이전 버전의 히스토리를 보존하므로 필요하면 롤백 가능
1. ReplicaSet의 이전 버전 관리 (업데이트 히스토리 저장)
    1. 사용자가 새로운 버전의 컨테이너 이미지를 배포하면, Deployment는 새로운 ReplicaSet을 생성
    2. 기존 ReplicaSet의 Pod를 하나씩 삭제하면서 새로운 ReplicaSet의 Pod를 배포하는 방식(롤링 업데이트)으로 진행됨
    3. 이전 ReplicaSet은 그대로 유지됨 → 즉, 필요하면 언제든지 롤백 가능.

1. 새로운 버전을 배포할 때 기존 ReplicaSet을 필요에 따라 유지하거나 삭제
    - Deployment의 `revisionHistoryLimit` 필드를 설정하면 이전 버전의 ReplicaSet을 몇 개까지 보관할지 정할 수 있음.
    
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-app
    spec:
      replicas: 3
      revisionHistoryLimit: 2  # 이전 ReplicaSet을 최대 2개까지만 유지
      selector:
        matchLabels:
          app: my-app
      template:
        metadata:
          labels:
            app: my-app
        spec:
          containers:
          - name: my-app-container
            image: my-app:2.0
    ```
