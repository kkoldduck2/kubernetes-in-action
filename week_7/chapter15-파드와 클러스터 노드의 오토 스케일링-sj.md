### 1. 수평적 파드 오토스케일링

- 수평적 파드 오토스케일링
    - 컨트롤러가 관리하는 파드의 레플리카 수를 자동으로 조정하는 것
    - Horizontal 컨트롤러에 의해 수행됨
    - HorizontalPodAutoscaler (HPA) 리소스를 작성하여 관리함

- 오토 스케일링 프로세스 이해
    - 파드 메트릭 얻기
        
        <img width="690" alt="image" src="https://github.com/user-attachments/assets/cdaed39c-9c29-4053-9824-c2e90f7144f0" />

        - kubelet에 있는 cAdvisor 에이전트가 파드 메트릭 수집 → 힙스터에 의해 집계됨
        - HPA 컨트롤러는 힙스터에 REST를 통해 질의해 모든 파드의 메트릭을 가져옴 (~버전 1.6이전)
        - 버전 1.8) 컨트롤러 매니저에 —horizontal-pod-autoscaler-use-rest-client=true로 설정하고 실행하면 Autoscaler는 집계된 버전의 리소스 메트릭 API를 통해 모든 파드의 메트릭을 얻을 수 있음
        - 버전 1.9) 위 설정이 기본으로 동작함
    - 필요한 파드 수 계산
        - Autoscaler는 각 메트릭(아래 그림에서는 CPU 사용률, QPS (초당 질의 수))의 레플리카 수를 개별적으로 계산한 뒤 가장 높은 값을 취한다.
        
        <img width="614" alt="image" src="https://github.com/user-attachments/assets/c8f697f3-9783-47c0-8a98-5b42ccaaf2d7" />

    
    - 스케일링된 리소스의 레플리카 수 갱신
        - 리소스 오브젝트(ex. 레플리카셋)의 레플리카 개수 필드를 원하는 값으로 갱신해, 레플리카 셋 컨트롤러가 추가 파드를 생성하거나 초과한 파드를 삭제
        - Autoscaler 컨트롤러는 **스케일 서브 리소스(scale sub-resource)**를 통해 스케일 대상 리소스의 replicas 필드를 변경한다.
        - API 서버가 scale sub-resource 를 노출하는 한, Autoscaler는 모든 scalable한 리소스를 대상으로 동작할 수 있다.
            - 현재 노출되는 리소스 : 디플로이먼트, 레플리카셋, 레플리케이션컨트롤러, 스테이트풀셋
        
        <img width="614" alt="image" src="https://github.com/user-attachments/assets/7a7a9db5-3f17-49d2-a968-225218fe028d" />


- CPU 사용률 기반 스케일링
    - CPU 사용량이 100%에 도달하면 더 이상 요구에 대응할 수 없음 → 스케일 업 or 스케일 아웃
    - 따라서 CPU가 완전히 바쁜 상태에 도달하기 전 (80%)에 스케일 아웃을 수행하는 것이 좋다.
    - 그러나 파드가 80%의 CPU를 사용한다고 할때 이게 노드의 CPU 중 80%인지, 파드가 보장받은 CPU 중의 80%인지, 리소스 제한의 80%인지 명확하지 않다.
        
        → Pod가 CPU 요청을 통해 보장받은 CPU 사용량만이 중요하다.
        
        → Autoscaler는 파드의 실제 CPU 사용량과 CPU 요청을 비교함
        → 오토스케일링이 필요한 파드는 직간접적으로 LimitRange 오브젝트를 통해 CPU 요청을 설정해야 함
        
    - 예시
        
        ```yaml
        apiVersion: autoscaling/v2
        kind: HorizontalPodAutoscaler
        metadata:
          name: example-hpa
        spec:
          scaleTargetRef:
            apiVersion: apps/v1
            kind: Deployment
            name: example-deployment
          minReplicas: 1
          maxReplicas: 10
          metrics:
          - type: Resource
            resource:
              name: cpu
              target:
                type: Utilization
                averageUtilization: 50
        ```
        
        - Pod의 평균 CPU 사용률이 요청된 CPU의 50%를 초과하면 스케일 아웃
        - 사용률이 50% 미만으로 떨어지면 스케일 인
        - Pod 수는 항상 1(minReplicas)에서 10(maxReplicas) 사이로 유지
    

- 메모리 소비량에 기반을 둔 스케일링
    - 예시
        
        ```yaml
        apiVersion: autoscaling/v2
        kind: HorizontalPodAutoscaler
        metadata:
          name: memory-based-hpa
        spec:
          scaleTargetRef:
            apiVersion: apps/v1
            kind: Deployment
            name: example-deployment
          minReplicas: 1
          maxReplicas: 10
          metrics:
          - type: Resource
            resource:
              name: memory
              target:
                type: Utilization
                averageUtilization: 60
        ```
        
    - 주의사항:
        - 메모리는 비압축성 리소스→ 애플리케이션이 직접 메모리를 해제하는 것이 필요  (시스템이 못함)
        - **스케일 업 시 발생하는 상황**
            - 메모리 사용량이 높아지면 HPA가 새로운 Pod를 생성(스케일 업/아웃)
            - 이때 기존에 실행 중이던 Pod들(오래된 Pod)은 여전히 높은 메모리를 사용하고 있을 수 있음
            - 새 Pod가 추가되어도 **기존 Pod의 메모리는 자동으로 해제되지 않음**
        - **애플리케이션의 책임**
            - 쿠버네티스 시스템은 Pod 내부의 애플리케이션이 사용 중인 메모리를 강제로 해제할 수 없음
            - 애플리케이션이 명시적으로 메모리를 해제하는 로직을 구현해야함
        - 예
            - 어떤 애플리케이션이 메모리 캐시를 사용하고 있다고 가정합니다
            - 트래픽이 증가하면서 캐시가 커지고 메모리 사용량이 80%에 도달합니다
            - HPA가 Pod를 2개에서 4개로 스케일 업합니다
            - 새로운 Pod들이 생성되었지만, 기존 Pod들은 여전히 큰 캐시를 메모리에 유지하고 있습니다
            - 이 상황에서는 애플리케이션이 부하 감소를 감지하고 캐시 크기를 줄이는 로직이 필요합니다
        
- 기타 그리고 사용자 정의 메트릭 기반 스케일링
    - 외부 메트릭(External Metrics)
        - 외부 시스템(예: 클라우드 제공업체)에서 제공하는 메트릭을 사용
        
        ```yaml
        # 메시지 큐의 길이에 따라 스케일링
        apiVersion: autoscaling/v2
        kind: HorizontalPodAutoscaler
        metadata:
          name: queue-based-hpa
        spec:
          scaleTargetRef:
            apiVersion: apps/v1
            kind: Deployment
            name: queue-processor
          minReplicas: 1
          maxReplicas: 10
          metrics:
          - type: External
            external:
              metric:
                name: queue_messages_ready
                selector:
                  matchLabels:
                    queue: worker_tasks
              target:
                type: AverageValue
                averageValue: 30
        ```
        
    - 파드 메트릭(Pod Metrics)
        - 애플리케이션 자체에서 노출하는 메트릭을 기반으로 스케일링
        
        ```yaml
        # 초당 요청 수에 따라 스케일링
        apiVersion: autoscaling/v2
        kind: HorizontalPodAutoscaler
        metadata:
          name: app-request-hpa
        spec:
          scaleTargetRef:
            apiVersion: apps/v1
            kind: Deployment
            name: web-app
          minReplicas: 1
          maxReplicas: 10
          metrics:
          - type: Pods
            pods:
              metric:
                name: requests_per_second
              target:
                type: AverageValue
                averageValue: 1000
        ```
        
    - 객체 메트릭 (Object Metrics)
        - 쿠버네티스 객체(예: Ingress)에서 제공하는 메트릭을 기반으로 스케일링
        
        ```yaml
        apiVersion: autoscaling/v2
        kind: HorizontalPodAutoscaler
        metadata:
          name: ingress-requests-hpa
        spec:
          scaleTargetRef:
            apiVersion: apps/v1
            kind: Deployment
            name: web-app
          minReplicas: 1
          maxReplicas: 10
          metrics:
          - type: Object
            object:
              describedObject:
                apiVersion: networking.k8s.io/v1
                kind: Ingress
                name: main-ingress
              metric:
                name: requests_per_second
              target:
                type: Value
                value: 2000
        ```
        

### 2. 수직적 파드 오토스케일링

- 없다. 현재는 기존에 존재하는 파드의 리소스 요청이나 한계를 변경 불가

### 요약

- 파드의 자동 수평적 스케일링 설정은 **HorizontalPodAutoscaler** 오브젝트를 만들고 디플로이먼트, 레플리카셋 또는 레플리케이션컨트롤러를 가리키게 한 다음, 목표 CPU 사용률을 정의하기만 하면 된다.
- 수평적 파드 오토스케일러가 파드의 CPU 사용률에 기반해 스케일링 동작을 수행하도록 하는 것 외에도, **애플리케이션에서 제공하는 사용자 정의 메트릭**이나 클러스터에 배포된 **다른 오브젝트와 관계된 메트릭**을 이용하도록 설정할 수도 있다.
- 수직적 파드 오토스케일링은 아직 불가능하다
- 쿠버네티스 클러스터가 지원하는 클라우드 제공자 위에서 실행되는 경우 **클러스터 노드도 자동으로 확장**할 수 있다.
