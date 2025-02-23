쿠버네티스 안에서 실행되는 애플리케이션에 설정 데이터를 전달하는 방법

- 컨테이너에 명령줄 인수 전달
- 각 컨테이너를 위한 사용자 정의 환경변수 지정
- 특수한 유형의 볼륨을 통해 설정 파일을 컨테이너에 마운트

# 1. ConfigMap

### 1. 컨테이너에 명령줄 인자 전달

쿠버네티스는 파드 컨테이너 정의에 지정된 실행 명령 대신 다른 실행파일을 실행하거나 다른 명령줄 인자를 사용해 실행하는 것이 가능하다. 

**1) 도커에서 명령어와 인자 정의**

- ENTRYPOINT
    - 컨테이너가 시작될 때 반드시 실행되어야 하는 기본 **명령어**를 정의
    - docker run 실행 시 추가되는 인자들은 ENTRYPOINT의 파라미터로 전달됨.
    - shell 형식 또는 exec 형식으로 작성할 수 있음.
        - shell : ENTRYPOINT node app.js → 메인 프로세스는 shell 프로세스.
        - exec 형식 : ENTRYPOINT [”node”, “app.js”]
- CMD
    - 컨테이너가 시작될 때 기본 명령어 혹은 ENTRYPOINT에 전달되는 파라미터를 정의
    - docker run 명령어에서 새로운 명령어를 지정하면 Dockerfile의 CMD는 무시됨
    - 컨테이너 실행 시 유연하게 다른 명령어로 오버라이드하고 싶을 때 사용함
    - 예: CMD [”npm”, “start”]
- 예: ENTRYPOINT와 CMD를 함께 사용하는 경우
    
    ```docker
    FROM ubuntu:20.04
    
    ENTRYPOINT ["ping"]
    CMD ["-c", "3", "google.com"]
    ```
    
    ```docker
    # 1. 기본 실행 (CMD에 정의된 파라미터 사용)
    docker run ping-image
    # 결과: google.com으로 3번 ping
    # ping -c 3 google.com과 동일
    
    # 2. CMD 파라미터 일부 오버라이드
    docker run ping-image -c 5 google.com
    # 결과: google.com으로 5번 ping
    # ping -c 5 google.com과 동일
    
    # 3. 다른 도메인으로 변경
    docker run ping-image -c 2 naver.com
    # 결과: naver.com으로 2번 ping
    # ping -c 2 naver.com과 동일
    ```
    

**2) 쿠버네티스에서 명령과 인자 재정의**

- 쿠버네티스에서 컨테이너를 정의할 때 command와 args 속성을 이용해 ENTRYPOINT와 CMD 둘 다 재정의 할 수 있다.
- 커스텀 스크립트 실행 예시

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-script-pod
spec:
  containers:
  - name: script-runner
    image: python:3.9
		# command는 Dockerfile의 ENTRYPOINT를 재정의
    command: ["/bin/bash"]
		# args는 Dockerfile의 CMD를 재정의
    args: ["-c", "while true; do python /app/check_health.py; sleep 60; done"]
```

### 2. 컨테이너의 환경변수 설정

- 컨테이너 명령이나 인자와 마찬가지로 환경변수 목록도 파드 생성 후에는 업데이트 할 수 없다.
- **파드 정의에 하드코딩된 값을 가져오는 것 → 프로덕션과 개발을 위해 서로 분리된 파드 정의가 필요 → configmap**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    env:
    - name: DB_HOST
      value: "localhost"
```

### 3. 컨피그맵으로 설정 분리

- 환경에 따라 다르거나 자주 변경되는 설정 옵션 → 애플리케이션 소스 코드와 별도로 유지하는 것이 좋다.
- 컨피그맵의 내용은 컨테이너의 환경변수 또는 볼륨 파일로 전달됨.

1) ConfigMap 생성

- yaml 파일로 생성

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database.url: "mysql:3306"
  api.url: "http://api.example.com"
  app.properties: |
    environment=production
    log.level=info
    max.connections=100
```

- 명령어로 생성

```yaml
# 리터럴 값으로 생성
kubectl create configmap app-config --from-literal=database.url=mysql:3306

# 파일로부터 생성
# kubectl을 실행한 디렉터리에서 app.properties 파일을 찾아 파일 내용을 
# 컨피그맵의 app.properties 키 값으로 저장한다. 
kubectl create configmap app-config --from-file=app.properties

# 디렉토리의 모든 파일로부터 생성
# 각 파일을 개별적으로 추가하는 대신, 디렉터리 안에 있는 모든 파일을 가져올 수도 있다. 
kubectl create configmap app-config --from-file=config-dir/
```

![image.png](attachment:e240f899-79cc-440d-a083-f8aaf7501877:image.png)

2) 컨피그맵을 주입하는 방법

- 환경변수로 주입하는 방법
    - 동작 방식
        - Pod 스펙(`spec.containers.env` 혹은 `envFrom`)에 ConfigMap을 연경해 컨테이너 기동 시점에 환경 변수가 설정됨
    - 특징
        - 애플리케이션에서 “환경 변수를 읽는 로직”만 있으면 됨.
        - 예) `os.getenv("MY_ENV")`, `System.getenv("MY_ENV")` 등.
    - 장점:
        - 간단한 설정값(포트, 호스트, 토큰 등)을 빠르게 전달하기 좋다.
        - 추가적인 파일 IO 없이 환경 변수를 바로 사용하므로 로직이 간단함
    - 단점 :
        - **큰(대용량) 설정**에는 부적합 (환경 변수는 길이 제한이 있을 수 있고, 가독성이 떨어짐).
        - 환경 변수값을 변경하려면 **Pod 재시작** 필요(일반적으로 런타임에 동적으로 바뀌기 어려움).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    env:
    - name: DATABASE_URL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database.url
    
    # 모든 설정을 환경변수로 가져오기
    envFrom:
    - configMapRef:
        name: app-config
```

- 볼륨으로 마운트하는 방법
    - 동작 방식
        - Pod 스펙(`spec.volumes`)에서 ConfigMap을 지정하고, 컨테이너에서 `spec.containers.volumeMounts`를 통해 **특정 경로**로 해당 ConfigMap 내용을 파일로 마운트
        - 예) `ConfigMap`에 있는 `key=value`가 실제 `/etc/config/key` 파일로 만들어지거나, 파일 형태(`.properties`, `.yml` 등)로 그대로 매핑됩니다.
    - 특징
        - 애플리케이션이 **파일을 읽는 방식**(`nginx.conf`, `application.properties` 등)으로 설정값을 로드
        - ConfigMap의 각 key-value는 컨테이너 내부에서 파일 형태로 나타남.
        - 예: app-config의 key가 `app.properties`라면 `/etc/config/app.properties` 파일로 자동 생성됨.
    - 장점 :
        - **파일 기반 설정**(Nginx 설정, Spring Boot 설정, SSL 인증서 등)을 그대로 유지할 수 있어 편리
        - 큰 설정값(장문의 YAML, JSON 등)에 적합
    - 단점 :
        - 애플리케이션이 해당 파일 경로에서 설정을 읽어오도록 만들어져 있어야 함 (환경 변수 방식보다 설정이 조금 복잡)
        - ConfigMap이 변경되어도, 해당 파일을 **실시간**으로 재로딩할지는 애플리케이션에 따라 다름(일반적으로는 Pod 재시작 혹은 파일 재로딩 로직 필요).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

- 차이 : 주입된 설정이 컨테이너 내부에서 어떤 형태로(환경 변수 vs 파일) 존재하느냐

3) 컨피그맵 업데이트

- 컨피그맵이 업데이트되면
    - 환경변수로 사용된 경우 : Pod를 재시작해야 반영됨
    - 볼륨으로 마운트된 경우 : 자동으로 갱신됨
- 환경 변수는 운영 체제 관점에서**프로세스가 시작될 때**설정되고, 일반적으로 프로세스가 도는 동안에는 바뀌지 않음
- 반면, **볼륨으로 마운트** 하는 경우엔 Kubelet이 주기적으로 ConfigMap 변경을 감지하여 **파일**을 업데이트해주는 기능이 있어서, 파일 내용은 컨테이너가 계속 살아 있는 동안에도 바뀔 수 있다.
- 주의! 이미 존재하는 디렉터리에 파일만 마운트(Subpath) 했을 때 업데이트가 되지 않음
- 쿠버네티스에서 **ConfigMap**(또는 Secret)을 “프로젝션 볼륨(Projection Volume)”으로 마운트할 때, 내부적으로는 심볼릭 링크(soft link)를 활용하여 원자적(atomic)으로 파일을 교체한다. 이 메커니즘이 바로 “자동 업데이트”가 가능하게 만드는 핵심 포인트인데, **subPath**를 사용하면 이 심볼릭 링크 교체 과정을 우회하게 되어 자동 갱신이 깨지게 된다.

---

+덧) subPath가 자동 갱신이나 원자성이 깨지는 이유

1. **원자적 교체(심볼릭 링크) 메커니즘**
    - ConfigMap이나 Secret을 **디렉터리 전체**로 마운트하면, Kubelet은 실제 데이터 디렉터리(예: `..2025_02_24_15_00_00.0123456`)와 그것을 가리키는 **심볼릭 링크(`..data`)**를 활용하여
    - *“이전 링크 → 새 링크”**를 **한 번에(원자적)** 교체합니다.
    - 이 과정을 통해 “파일 덮어쓰는 중간 단계”가 노출되지 않고, 바뀌는 순간 바로 새 데이터로 전환되는 것이죠.
2. **subPath가 왜 원자성을 깨뜨리는가?**
    - subPath로 마운트하면, 디렉터리 전체가 아니라 **특정 파일(혹은 특정 하위 디렉터리)**만 **직접** 바인드 마운트합니다.
    - 즉, 원래 `..data`(심볼릭 링크)로 연결되어야 할 지점이 **파일 단위**로 고정되어 버립니다.
    - 이후 Kubelet이 **새 디렉터리에 심볼릭 링크**를 바꿔도, subPath로 바인드된 파일은 여전히 **옛 경로(또는 옛 디렉터리)에 묶여** 있어서 **새 링크**를 보지 못합니다.

결과적으로,

- **디렉터리 전체를 마운트**하면 심볼릭 링크 교체가 “원자적 업데이트”를 보장하지만,
- **subPath**를 사용하면 그 링크 교체 과정을 **우회**하여 **원자성이 깨지고** 자동 갱신도 반영되지 않는 문제가 발생합니다.

명령어로 생성하든, 컨피그맵 볼륨을 사용하든, 모든 경우 데이터는 etcd에 저장된다. 

# 2. Secret

- 보안이 유지되어야 하는 자격증명과 개인 암호화 키와 같은 민감정보는 secret으로 관리
- 2가지 방식으로 사용 가능
    - 환경변수로 시크릿 항목을 컨테이너에 전달
    - 시크릿 항목을 볼륨 파일로 노출
- 쿠버네티스는 시크릿에 접근해야 하는 파드가 실행되고 있는 노드에만 개별 시크릿을 배포 → 시크릿을 안전하게 유지
- 노드 자체적으로 시크릿을 항상 메모리에만 저장되게 하고 물리 저장소에 기록되지 않도록 함
- 마스터 노드(etcd)에는 시크릿을 암호화되지 않은 형식(base64 인코딩)으로 저장함. 따라서 etcd 접근권을 얻으면 Secret 내용을 그대로 볼 수 있다.
- RBAC (Role-Based Access Control)
    - Secret을 읽을 수 있는 권한을 최소화해야 합니다.
    - 예를 들어, 특정 네임스페이스 내 Secret은 해당 네임스페이스의 관리자만 열람 가능하도록 설정.
- 파드가 기동되면, 그 파드가 사용하는 Secret 값을 컨테이너 내부에서 읽을 수 있으므로, 파드(또는 노드)에 대한 보안도 중요합니다.
