## 7장 컨피그맵과 시크릿 : 애플리케이션 설정



### Parameter Argument 차이 

Parameter는 함수 혹은 메서드 정의에서 나열되는 변수명

Argument는 함수 혹은 메서드를 호출할 때, 전달 혹은 입력되는 실제 값입니다

| 단어      | 번역           | 의미                                 |
| :-------- | :------------- | :----------------------------------- |
| Parameter | 매개변수       | 함수와 메서드 입력 변수(Variable) 명 |
| Argument  | 전달인자, 인자 | 함수와 메서드의 입력 값(Value)       |

#### parameter(매개변수)

다음 cancat 함수 정의에서 str1과 str2는 parameter 입니다.

```py
def cancat(str1, str2):
  return a +" "+ b
```

#### argument(전달인자)

cancat 함수를 호출할 때, 입력값 “parameter”와 “argument”는 argument입니다.

```python
cancat("parameter", "argument")
```





---



### Shell Form 과 Exec Form 차이

Dockerfile에서 명령어를 실행할 때, **Shell Form**(쉘 형식)과 **Exec Form**(exec 형식)의 두 가지 방법이 있다.

이 둘의 차이를 이해하는 것은 Dockerfile 작성의 핵심 중 하나

| 특성                       | **Shell Form**                       | **Exec Form**                   |
| -------------------------- | ------------------------------------ | ------------------------------- |
| **구문**                   | `CMD echo "Hello, World!"`           | `CMD ["echo", "Hello, World!"]` |
| **실행 방식**              | `/bin/sh -c`를 통해 실행             | 명령어를 직접 실행              |
| **쉘 기능 사용 가능 여부** | 가능 (`&&`, `                        |                                 |
| **보안성**                 | 쉘 인젝션 가능성 있음                | 보안성이 높음                   |
| **성능**                   | 쉘 프로세스 생성으로 약간의 오버헤드 | 쉘이 없어 더 효율적             |
| **유연성**                 | 높은 유연성                          | 명령어 및 인수에 국한됨         |





---



### 시크릿은 민감정보이기때문에 디스크에 저장하지않고 메모리에 저장된다.

### ConfigMap 도 메모리에 저장 시킬 수 있음.



---



### 컨테이너 실행 파일과 인자 지정

- docker
  - `ENTRYPOINT`: 컨테이너 내부에서 실행되는 파일
  - `CMD`: 실행파일에 전달되는 인자
- K8S
  - `command`: 실행파일에 전달되는 인자
  - `args`: 컨테이너 내부에서 실행되는 파일



---



### Configmap volume의 파일권한

default로 configmap 볼륨의 파일 권한은 `644`로 설정된다.
변경을 원한다면 `volumes.configMap.defaultMode`에서 설정할 수 있다.



---



### 애플리케이션 재시작 없이 애플리케이션 설정 업데이트

환경 변수로 설정하는 경우, 컨테이너 실행 중 환경변수의 변경이 불가하다.
컨테이너 재시작 없이 설정의 변경이 필요한 경우, `configmap 볼륨`을 사용하자.
단, 볼륨을 참조하는 컨테이너 내부 프로세스는 볼륨의 변경을 감지하고 재로드할 수 있어야 한다.



---



### 파일을 원자적으로 업데이트할 수 있는 원리

configmap 볼륨의 내용을 변경하면, 파드에 마운트된 내용들 또한 변경된다.
이는 이러한 파일들이 `symbolic link`로 연결되어 있기 때문이다.



---



### secret이 base64로 저장되는 이유

secret으로 저장되는 단순 문자열의 경우 바로 저장이 가능하지만,
SSL 인증서와 같은 바이너리 파일의 경우 문자열로 저장이 불가능하다.
바이너리와 같은 데이터를 저장하기 위해 이를 base64를 한번 인코딩한 후, secret으로 저장하는 것이다.



---



### 전역 컨피그맵 설정 방법

 - configmap-replicator

   ```yaml
   # ConfigMap Replicator using Operator
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: configmap-replicator
   spec:
     template:
       spec:
         containers:
         - name: replicator
           image: configmap-replicator
           args:
           - --source-configmap=global-config
           - --source-namespace=default
   ```

   

 - kustomize

   ```yaml
   # kustomization.yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   
   resources:
   - global-config.yaml
   
   namespace: default
   ```

   

 - helm

   ```yaml
   # values.yaml
   global:
     config:
       key1: value1
       key2: value2
   
   # templates/configmap.yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: {{ .Release.Name }}-config
   data:
     {{- toYaml .Values.global.config | nindent 2 }}
   ```

   

출처 :

https://velog.io/@sjoh0704/K8S-Configmap-%EC%82%AC%EC%9A%A9%EC%8B%9C-%EC%95%8C%EB%A9%B4-%EC%A2%8B%EC%9D%80-%EA%B2%83%EB%93%A4

https://sundaland.tistory.com/556

http://taewan.kim/tip/argument_parameter/