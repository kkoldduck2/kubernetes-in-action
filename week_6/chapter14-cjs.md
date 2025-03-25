## 14장 : 파드의 컴퓨팅 리소스 관리



### Schedule 우선순위 방식

#### LeastRequestedPriority (최소 요청 우선순위)

- **목적**: 노드 간 리소스를 효율적으로 분산
- 특징
  - 기본값 (`kubectl describe deploy kube-scheduler -n kube-system` 명령어로 확인가능)
  - 리소스 요청량이 가장 적은 노드에 파드를 배치
  - 클러스터의 전체 리소스를 고르게 활용
  - CPU와 메모리 요청량을 기반으로 노드 선택

#### MostRequestedPriority (최대 요청 우선순위)

- **목적**: 노드 활용도를 최대화
- 특징
  - 리소스 요청량이 가장 많은 노드에 파드 집중
  - 클러스터의 노드 수를 최소화
  - 리소스 활용률이 높은 노드 선호



---



## CPU 시간 공유(Time-sharing)

####  컴퓨터 운영 체제가 여러 사용자나 프로그램이 동시에 CPU를 사용할 수 있도록 하는 방식.

* 여러 프로그램을 동시에 실행하는 것처럼 보이게함

* CPU가 아주 짧은 시간 동안 각 프로그램에 번갈아 할당

* 사용자는 자신의 프로그램이 독점적으로 CPU를 사용하는 것처럼 느낌

운영 체제는 시간 공유 개념을 사용하여 여러 프로그램이 동시에 실행될 수 있도록 합니다.





---



## 왜 Limit CPU를 초과하여 사용하는가 

**CPU 제한이 초과되는 이유와 방식:**

- CPU는 압축 가능한(compressible) 리소스로 간주됩니다.
  - CPU Compressible 메커니즘
    - 스로틀링을 통해 CPU 사용량 제한 가능
    - 전체 시스템 중단 없이 리소스 조절
    - 성능 일부 감소는 있지만, 시스템 안정성 유지
- 컨테이너가 CPU limit을 초과하면, 쿠버네티스는 해당 컨테이너의 CPU 시간을 제한하려고 시도합니다.
- 그러나 CPU 사용량이 순간적으로 급증하는 경우(burst), 실제로 limit을 초과하는 CPU를 사용할 수 있습니다.
- 이는 노드에 여유 CPU 자원이 있을 때 특히 그렇습니다.

**메모리와의 차이점:**

- 반면에 메모리 limits는 **하드 제한(hard limit)**입니다.
- 컨테이너가 메모리 limit을 초과하려고 하면, OOM(Out Of Memory) Killer에 의해 컨테이너가 종료될 수 있습니다.

**CPU 제한을 더 엄격하게 적용하고 싶다면:**

* CFS(Completely Fair Scheduler) 쿼터를 사용하는 방법
* ResourceQuota를 통한 네임스페이스 수준의 제한 설정
* LimitRange를 통한 기본 제한 설정

정리하자면, 쿠버네티스에서 CPU limits는 완전히 엄격한 제한이 아니라 CPU 시간의 비율을 조절하는 방식으로 작동하므로, 순간적으로는 설정한 제한을 초과할 수 있습니다



---



## 리소스 Request 미지정시 Limit과 동일한 값으로 설정???

* **v1.8 이전**: `limits` 설정 시 `requests`가 자동으로 동일하게 설정

* **v1.8 이후**: 명시적으로 `requests`를 설정하지 않으면 `0`으로 처리



---



## Limit 합계는 Node 보다 클 수 있다... 언제든 OverCommited 될 수 있음

CPU는 오버커밋되면 속도가 느려지다 Eviced발생, Memory가 오버커밋되면 바로 시스템이 종료된다.

```
메모리/CPU 압박 → 
  - QoS 클래스 확인
  - 리소스 사용량 분석
  - 파드 퇴거 결정
```





---



## LimitRange

네임스페이스 단위로 설정가능하며 개별 파드 및 Configmap 등 **각 리소스 에 대한 최소,최대,기본 리소스 요청 제한**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: comprehensive-limits
  namespace: default
spec:
  limits:
  # Container 리소스 제한
  - type: Container
    defaultRequest:
      cpu: 100m
      memory: 256Mi
    default:
      cpu: 200m
      memory: 512Mi
    min:
      cpu: 50m
      memory: 128Mi
    max:
      cpu: 1
      memory: 2Gi

  # Pod 리소스 제한
  - type: Pod
    max:
      cpu: 2
      memory: 4Gi

  # PersistentVolumeClaim 스토리지 제한
  - type: PersistentVolumeClaim
    min:
      storage: 1Gi
    max:
      storage: 10Gi

  # ConfigMap 크기 제한
  - type: ConfigMap
    max:
      storage: 10Mi

  # Secret 크기 제한
  - type: Secret
    max:
      storage: 1Mi

  # EmptyDir 임시 볼륨 제한
  - type: EmptyDir
    max:
      storage: 5Gi

  # Ephemeral Storage 제한
  - type: EphemeralStorage
    min:
      storage: 500Mi
    max:
      storage: 5Gi
```



---



## ResourceQuota

네임스페이스 단위로 설정 가능하며 사용 가능한 **총 리소스 제한** 

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: comprehensive-quota
  namespace: default
spec:
  hard:
    # 컴퓨팅 리소스 제한
    requests.cpu: "4"
    limits.cpu: "8"
    requests.memory: 8Gi
    limits.memory: 16Gi

    # 스토리지 리소스 제한
    requests.storage: 50Gi
    persistentvolumeclaims: "10"

    # 오브젝트 개수 제한
    pods: "20"
    services: "10"
    secrets: "50"
    configmaps: "30"
    replicationcontrollers: "20"
    deployments.apps: "10"
    statefulsets.apps: "5"
    jobs.batch: "10"
    cronjobs.batch: "5"

    # 특정 리소스 타입 제한
    services.loadbalancers: "2"
    services.nodeports: "3"

    # 노드포트 서비스 제한
    nodeports: "3"
```

