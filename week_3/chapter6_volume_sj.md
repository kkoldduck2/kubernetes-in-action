### 1. 볼륨을 이용한 컨테이너 간 데이터 공유 (EmptyDir)

- 동일 파드에서 실행 중인 컨테이너간 파일을 공유할 때 유용함.
- pod가 삭제되면 emptyDir의 데이터도 함께 삭제됨

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

### 워커 노드 파일 시스템의 파일 접근 (HostPath)

<img width="717" alt="image" src="https://github.com/user-attachments/assets/5a18d1b7-e603-4a5f-a08d-312572577bce" />

- 일반적으로 대부분의 파드는 호스트 노드를 인식하지 못함 
→ 노드의 파일 시스템에 있는 어떤 파일에도 접근하면 안됨
- 그러나 어떤 파드는 (주로 데몬셋) 노드 자체의 파일이나 디바이스에 직접 접근해야할 때가 있다.
- emptyDir과 달리 파드가 종료되어도 hostPath 볼륨의 콘텐츠는 삭제되지 않는다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
  - name: hostpath-demo-container
    image: nginx:latest
    volumeMounts:
    - name: hostpath-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: hostpath-volume
    hostPath:
      path: /data           # 노드의 /data 디렉터리를 사용
      type: DirectoryOrCreate  # 디렉터리가 없으면 자동으로 생성

```

### 2. 퍼시스턴트 스토리지 사용

- 파드에서 실행 중인 애플리케이션이 **디스크에 데이터를 유지**해야 하고
- 파드가 다른 노드로 **재스케줄링된 경우에도 동일한 데이터를 사용**해야 한다면
- 어떤 클러스터 노드에서도 접근이 필요하기 때문에 NAS (Network-Attached Storage) 유형에 저장되어야 한다.
- 예제) GCE 퍼시스턴트 디스크 볼륨을 사용하는 파드 생성하기
    - mongodb라고 이름 붙여진 GCE 퍼시스턴트 디스크 생성 후 아래 명령 수행하여 해당 볼륨을 마운트하는 파드 생성
    <img width="717" alt="image" src="https://github.com/user-attachments/assets/2fbc4a4b-e2ce-4e5e-8e0a-dca905532df6" />

- 예제) NFS 볼륨 사용하기
    <img width="717" alt="image" src="https://github.com/user-attachments/assets/25b6c014-0afb-4c68-a4aa-9445fe87a425" />


### 3. PV와 PVC

- 등장 배경
    - 위와 같은 퍼시스턴트 볼륨 유형은 파드 개발자가 실제 네트워크 스토리지 인프라스트럭처에 관한 지식을 갖추고 있어야 함.
    - 즉, NFS 기반의 볼륨을 생성하려면 개발자는 NFS 익스포트가 위치하는 실제 서버를 알아야 한다. → 개발자는 이런거 모르고 싶다.
    - 개발자와 인프라 관리자의 역할 분리
        <img width="717" alt="image" src="https://github.com/user-attachments/assets/7ede1a3d-cb38-4e10-8b22-f35225109276" />

        - **개발자**
            - 파드가 필요한 만큼의 스토리 용량을 요청(“5Gi 필요합니다”)
            - 접근 모드(읽기, 쓰기 등)를 지정한 ‘PVC’를 생성
            - 그러면 쿠버네티스가 이에 맞는 PV를 알아서 연결해줌
        - **인프라 관리자**
            - 클러스터에 실제 물리 또는 논리 스토리지를 미리 준비해두고, 이를 ‘PV’라는 형태로 등록해둠.
            - 또는 스토리지 클래스를 통해 자동으로 PV가 만들어지도록 설정함
    - 결과적으로 개발자는 **어떤 물리 디스크나 NFS 서버를 써야 하는지** 같은 세부 사항을 몰라도 됨
    - **인프라 관리자는 스토리지를 안전하게 제공하고 관리**하는 데만 집중할 수 있게 됨
    - 이는 마치 개발자가 쿠버네티스에서 CPU, 메모리를 요청하듯이 **스토리지 또한 추상화된 인터페이스**로 요청하는 것과 같은 원리

- 덧) PVC, Pod와 달리 PV는 네임스페이스에 속하지 않는다.
<img width="717" alt="image" src="https://github.com/user-attachments/assets/91b1b0e1-bf67-4b4c-b0b0-f5c90100d37c" />

- 예) PVC 생성
    
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: example-pvc
    spec:
      storageClassName: standard   # 사용하고자 하는 StorageClass 이름 (클러스터 환경마다 다를 수 있음)
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
    ```
    
- 해당 PVC 를 mount하는 pod 생성
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: example-pod
    spec:
      containers:
      - name: example-container
        image: nginx:latest
        volumeMounts:
        - name: example-pvc-volume
          mountPath: /usr/share/nginx/html
      volumes:
      - name: example-pvc-volume
        persistentVolumeClaim:
          claimName: example-pvc
    ```
    

### 4. PV 재사용 정책

- Retain:
    - PVC가 삭제되어도 PV와 데이터는 유지됨 (Released 상태)
    - 클러스터 관리자가 볼륨을 완전히 비우지 않으면 새로운 클레임에 바인딩할 수 없다.
    - 수동으로 데이터를 삭제하고 PV를 재사용할 수 있음
    - 기본값은 Retain
- Delete:
    - PVC가 삭제되면 PV도 자동으로 삭제됨
    - 클라우드 환경에서 주로 사용
- Recycle (Deprecated):
    - 볼륨의 컨텐츠를 삭제하고 볼륨이 다시 클레임될 수 있도록 볼륨을 사용가능하게 한다. (Available 상태)
    - 더 이상 사용을 권장하지 않음
    - 동적 프로비저닝 사용을 권장

### 5. PV 동적 프로비저닝

- 클러스터 관리자가 PV 생성하는 대신 PV 프로비저너를 배포하고 사용자가 선택 가능한 PV 타입을 하나 이상의 StorageClass 오브젝트로 정의
- PVC 생성 시 자동으로 PV를 생성해주는 기능
- StorageClass를 통해 프로비저닝 방식을 정의한다.
- 수동으로 PV를 생성할 필요가 없어서 관리가 편리하다.
- 관리자가 많은 PV를 미리 프로비저닝하는 대신 하나 혹은 그 이상의 스토리지 클래스를 정의하면 시스템은 누군가 PVC을 통해 요청 시 새로운 PV를 생성한다.
- 장점: PV가 부족할일이 없다. (스토리지 용량이 부족할수는 있음)
- StorageClass 정의 예시
    
    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: standard
    provisioner: kubernetes.io/aws-ebs
    parameters:
      type: gp2
      encrypted: "true"
    reclaimPolicy: Delete
    allowVolumeExpansion: true
    volumeBindingMode: WaitForFirstConsumer
    ```
    
- PVC 생성 예시
    
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: dynamic-pvc
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
      storageClassName: standard
    ```
