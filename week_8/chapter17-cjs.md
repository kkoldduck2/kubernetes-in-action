## 17장 : 어플리케이션 개발을 위한 모범 사례

### POD LifeCycle

1. **파드 생성 요청** - 사용자가 파드 생성을 요청하면 API 서버는 이를 etcd에 저장

2. **스케줄링(Scheduling)** - 스케줄러가 적합한 노드를 선택하여 파드를 할당

3. 초기화(Initialization)

    \- 노드의 kubelet이 파드 실행을 준비합니다:

   - 볼륨 초기화
   - Init 컨테이너 실행

4. 컨테이너 실행(Container Running)

    \- 주 컨테이너가 실행되고 다양한 프로브가 동작

   - 스타트업 프로브(Startup Probe)
   - 라이브니스 프로브(Liveness Probe)
   - 레디니스 프로브(Readiness Probe)
   - **라이프사이클 훅(PostStart, PreStop)**

5. 종료(Termination)

    \- 파드 종료 시 진행되는 과정

   - PreStop 훅 실행
   - SIGTERM 신호 전송
   - 종료 유예 기간 대기
   - SIGKILL 신호 전송
   - 리소스 정리

#### 파드 상태(Pod Phase)

파드의 라이프사이클 동안 `.status.phase` 필드는 다음과 같은 상태를 가질 수 있음

- **Pending**: 승인되었지만 아직 모든 컨테이너가 실행 준비되지 않음
- **Running**: 노드에 바인딩되고 최소 하나의 컨테이너가 실행 중
- **Succeeded**: 모든 컨테이너가 성공적으로 종료
- **Failed**: 최소 하나의 컨테이너가 실패로 종료
- **Unknown**: 파드 상태를 확인할 수 없음

---



### 파드 실행 순서 의존성 설정

책에선 Init-Contaienr / Readiness-Probe를 소개하고있음.



#### Helm 차트 훅(Hooks) 사용

```yaml
# 데이터베이스 차트
metadata:
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "-5"

# 애플리케이션 차트 
metadata:
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "5"
```



#### Operator 패턴 구현

```yaml
apiVersion: myapp.example.com/v1
kind: MyApplication
metadata:
  name: my-complex-app
spec:
  database:
    enabled: true
    size: 10Gi
  cache:
    enabled: true
    dependsOn: ["database"]
  backend:
    replicas: 3
    dependsOn: ["database", "cache"]
  frontend:
    replicas: 2
    dependsOn: ["backend"]
```



#### ArgoCD Sync-wave 기능 활용

```yaml
# 데이터베이스 매니페스트
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
```

#### 실제 활용 패턴

인프라 준비 → 핵심 서비스 → 애플리케이션 → 추가 서비스

- 웨이브 -2: 네임스페이스, CRD
- 웨이브 -1: PV, PVC, ConfigMap, Secret
- 웨이브 0: 데이터베이스, 캐시 등 핵심 서비스
- 웨이브 1: 기본 애플리케이션
- 웨이브 2: 모니터링, 추가 서비스

---



### 모니터링용 로그 파일 패스 고려하면 더 좋았을듯



---



### **terminationMessagePath**

쿠버네티스에서 컨테이너가 종료될 때 종료 상태 메시지를 저장하는 파일 경로를 지정하는 설정

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
  - name: main-container
    image: example-image
    terminationMessagePath: "/var/log/termination-message"
```



---



### **FallbackTologsOnError**

 쿠버네티스 컨테이너 상태 검사(Probe)에서 사용하며 원인을 몇줄 기록함

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
  - name: example-container
    image: example-image
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      fallbackToLogsOnError: true  # 이 옵션 활성화
      initialDelaySeconds: 10
      periodSeconds: 5
```



