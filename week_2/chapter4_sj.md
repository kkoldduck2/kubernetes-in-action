## 1. Pod를 안정적으로 유지하기 : Kubelet의 컨테이너 크래시, Liveness probe  감지 및 재시작 메커니즘

- 컨테이너에 크래시가 발생하거나 라이브니스 프로브가 실패하는 경우, 쿠버네티스가 컨테이너를 재시작하여 컨테이너가 계속 실행되도록 한다.
- 작업 주체 : Kubelet
- **컨테이너 크래시**
    - 컨테이너가 비정상 종료되는 경우 (exit code가 0이 아닌 경우)
    - 컨테이너에서 크래시가 발생하는 경우 (Crash 발생 원인)
        
        
        | 원인 | 설명 | 종료 코드 (예시) |
        | --- | --- | --- |
        | **애플리케이션 코드 버그** | 코드 내부에서 예외(Exception)가 발생해 프로세스가 종료됨 | `1`, `2`, `255` |
        | **필수 파일/설정 누락** | 애플리케이션이 실행되기 위해 필요한 환경 변수, 설정 파일이 없을 때 | `1` |
        | **네트워크/DB 연결 실패** | 애플리케이션이 DB, 외부 API 등과 연결을 시도했으나 실패한 경우 | `1`, `2` |
        | **Out of Memory (OOMKilled)** | 컨테이너가 노드의 메모리를 너무 많이 사용해서 강제 종료됨 | `137` |
        | **SIGKILL/SIGTERM 신호 수신** | 컨테이너가 외부에서 `kill -9` 또는 `kill -15` 등의 신호를 받아 종료됨 | `137` (SIGKILL), `143` (SIGTERM) |
        | **무한 루프, 데드락** | 애플리케이션이 멈춰 응답하지 않음 (Liveness Probe 실패) | - |
        | **디스크 용량 부족** | 애플리케이션이 로그를 너무 많이 쓰거나, 파일을 저장하다가 디스크가 가득 참 | `1` |
        | **잘못된 이미지 또는 실행 파일 없음** | `docker run` 시점에 실행할 바이너리가 없거나, 컨테이너 이미지에 문제가 있음 | `127` |
        | **포트 충돌** | 다른 프로세스가 사용 중인 포트를 컨테이너가 사용하려고 시도함 | `1` |
- **라이브니스 프로브**
    - 프로세스의 크래시 없이도 애플리케이션 작동이 중지되는 경우, 외부에서 애플리케이션의 상태를 체크해야 함.
        
        예) 애플리케이션이 무한 루프나 교착 상태에 빠져서 응답하지 않는 상황
        
    - 파드의 스펙에 각 컨테이너의 라이브니스 프로브를 지정한다.
    - 쿠버네티스는 주기적으로 프로브를 실행하고 실패할 경우 컨테이너를 다시 시작한다.
    - 라이브니스 프로브에는 3가지 메커니즘이 있다.
        - HTTP GET 프로브
            - 지정한 IP 주소, 포트, 경로에 HTTP GET 요청을 수행한다.
            - 프로브가 응답을 수신하고 응답 코드가 2xx 혹은 3xx인 경우 프로브가 성공했다고 간주된다.
            
            ```yaml
            apiVersion: v1
            kind: Pod
            metadata:
              labels:
                test: liveness
              name: liveness-http
            spec:
              containers:
              - name: liveness
                image: registry.k8s.io/e2e-test-images/agnhost:2.40
                args:
                - liveness
                livenessProbe:
                  httpGet:
                    path: /healthz
                    port: 8080
                    httpHeaders:
                    - name: Custom-Header
                      value: Awesome
                  initialDelaySeconds: 3
                  periodSeconds: 3
            
            ```
            
        - TCP 소켓 프로브
            - 컨테이너의 지정된 포트에 TCP 연결을 시도한다.
            
            ```yaml
            apiVersion: v1
            kind: Pod
            metadata:
              name: goproxy
              labels:
                app: goproxy
            spec:
              containers:
              - name: goproxy
                image: registry.k8s.io/goproxy:0.1
                ports:
                - containerPort: 8080
                readinessProbe:
                  tcpSocket:
                    port: 8080
                  initialDelaySeconds: 15
                  periodSeconds: 10
                livenessProbe:
                  tcpSocket:
                    port: 8080
                  initialDelaySeconds: 15
                  periodSeconds: 10
            ```
            
        - Exec 프로브
            - 컨테이너 내의 임의의 명령을 실행하고 명령의 종료 상태 코드를 확인한다. 상태 코드가 0이면 프로브가 성공한 것이다. 모든 다른 코드는 실패로 간주된다.
    - liveness prove와 readiness probe의 차이
        - liveness prove
            - 목적 : 컨테이너가 **정상적으로 동작하고 있는지** 확인
            - 동작 방식 : 실패하면 **컨테이너를 재시작**
            - 예시 : 컨테이너가 멈추거나 응답하지 않는 경우
        - Readiness Probe
            - 목적: 컨테이너가 **트래픽을 받을 준비가 되었는지** 확인
            - 동작 방식 : 실패하면 **트래픽을 받지 않음 (Service의 엔드포인트에서 제거됨)**
            - 예시 : DB 연결이 아직 안 되었거나, 초기 로딩 중인 경우
            - 초기화 과정이 긴 애플리케이션 (예: DB 연결이 필요한 경우)에 유용함
            - 예 : TCP 기반 Readiness Probe
            
            ```yaml
            readinessProbe:
              tcpSocket:
                port: 5432  # PostgreSQL이 실행되는 포트
              initialDelaySeconds: 5  # 컨테이너 시작 후 5초 후부터 체크 시작
              periodSeconds: 10  # 10초마다 체크
              failureThreshold: 3  # 3번 실패하면 트래픽 전달 중단
            ```
            

- 덧) Kubelet이 컨테이너 크래시 및 라이브니스 프로브 감지하고 컨테이너 재기동하는 과정 :
    1. Kubelet은 컨테이너의 상태를 주기적으로 확인 (`kubectl get pods -o wide` 명령으로 확인 가능)
    2. 컨테이너가 **비정상 종료(exit code 0이 아님)** 또는 **Liveness Probe 실패** 등의 이유로 종료됨
    3. Kubelet이 이를 감지하고, **Pod의 `restartPolicy`를 확인**
        - `restartPolicy: Always` → 컨테이너를 즉시 재시작 (기본값)
        - `restartPolicy: OnFailure` → 비정상 종료일 경우에만 재시작
        - `restartPolicy: Never` → 재시작하지 않음
    4. 재시작 횟수가 `backoff limit`을 초과하면 **CrashLoopBackOff 상태**로 변경됨 (재시작 시도 간격 증가)

- 노드 자체에 크래시가 발생한 경우
    - 컨테이너의 프로세스에서 크래시가 발생하면 kubelet이 처리함
    - 노드에서 크래시가 발생하면 **컨트롤 플레인**이 노드 크래시로 중단된 모든 파드의 대체 파드를 생성함.

## 2. 레플리케이션 컨트롤러와 레플리카 셋

- 레플리케이션 컨트롤러
    - 지정된 개수(Replica)만큼 **파드를 유지**하도록 관리
    - 파드가 죽으면 새로운 파드를 생성
    - 너무 많은 파드가 있으면 초과된 파드를 삭제
    - **하지만, 셀렉터가 유연하지 않아서 특정 레이블 매칭만 가능** (기능이 제한됨)
    
    ```yaml
    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: my-app-rc
    spec:
      replicas: 3  # 항상 3개의 파드를 유지
      selector:
        app: my-app
      template:
        metadata:
          labels:
            app: my-app
        spec:
          containers:
            - name: my-container
              image: nginx
              ports:
                - containerPort: 80
    ```
    

- 레플리카 셋
    - **ReplicationController의 개선 버전**
    - **Set-Based Selector** 지원 → 더 유연한 라벨 매칭 가능
    - 대부분의 경우, **ReplicaSet 대신 Deployment를 사용**하는 게 일반적
    
    ```yaml
    apiVersion: apps/v1
    kind: ReplicaSet
    metadata:
      name: my-app-rs
    spec:
      replicas: 3  # 항상 3개의 파드를 유지
      selector:
        matchLabels:
          app: my-app
        matchExpressions:
          - { key: env, operator: In, values: [production, staging] }  # env=production 또는 env=staging만 선택
      template:
        metadata:
          labels:
            app: my-app
            env: production
        spec:
          containers:
            - name: my-container
              image: nginx
              ports:
                - containerPort: 80
    ```
