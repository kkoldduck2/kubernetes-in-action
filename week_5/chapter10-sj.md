### 스테이트풀 셋과 레플리카셋 비교

- 레플리카셋 : Stateless 애플리케이션 → 인스턴스가 죽더라도 새로운 인스턴스를 만들 수 있고 사용자는 그 차이를 알아차리지 못함
- 스테이트풀셋 : Stateful 애플리케이션 → 새 인스턴스가 이전 인스턴스와 완전히 같은 상태와 아이텐티티를 가져야 한다. (교체되기 이전 파드와 동일한 이름, 네티워크 아이덴티티, 상태 그대로 되살아나야 함)
- 또한 스테이트풀셋은 다른 피어와 구별되는 자체의 볼륨 세트를 가진다.

### 안정적인 네트워크 아이덴티티 제공하기

- Stateless 파드 → 그들중 아무거나 하나 선택하면 됨
- Stateful 파드 → 각각 서로 다름 (서로 다른 상태를 가짐) → 그룹의 특정 파드에서 동작해야하는 경우가 존재
- **거버닝 헤드리스 서비스 :**
    - **각 파드에게 실제 네트워크 아이덴티티를 제공해야함**
    - **이 서비스를 통해 각 파드는 자체 DNS 엔트리를 가짐**
    - **따라서 호스트 이름을 통해 파드의 주소를 지정할 수 있다.**
    - 예) default 네임스페이스에 속하는 foo라는 이름의 거버닝 서비스가 있고 파드 이름이 A-0이라면, 이 파드는 a-0.foo.default.svc.cluster.local이라는 FQDN을 통해 접근할 수 있다. (레플리카셋으로 관리되는 파드에서는 불가능하다. )

### 스테이트풀셋 스케일링

- 스케일업
    - 새로운 인스턴스는 인덱스2를 부여받는다. (기존 인스턴스의 인덱스가 0과 1일 경우)
- 스케일 다운
    - 높은 수의 인덱스를 갖는 인스턴스부터 순차적으로 삭제
    - 인스턴스 하나라도 비정상인 경우 스케일 다운 작업을 허용하지 않는다. 인스턴스 하나가 정상적으로 동작하지 않는 시점에 인스턴스 하나를 스케일 다운하면 결과적으로 두 개의 클러스터 멤버를 잃게되기 때문

### 각 스테이트풀 인스턴스에 안정적인 전용 스토리지 제공하기

- Statefulset은 파드마다 고유한 볼륨(PVC)를 할당함
    - 그렇기 때문에 파드가 삭제되더라도 데이터는 유지됨 → 파드가 재시작되더라도 기존 데이터는 유지할 수 있음
- 스테이트풀셋은 각 파드에 할당되는 PVC를 복제하는 하나 이상의 볼륨 클레임 템플릿을 가질 수 있다
- 예)
    
    ```yaml
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: mysql
    spec:
      serviceName: "mysql"
      replicas: 3
      selector:
        matchLabels:
          app: mysql
      template:
        metadata:
          labels:
            app: mysql
        spec:
          containers:
          - name: mysql
            image: mysql:5.7
            ports:
            - containerPort: 3306
            volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
      volumeClaimTemplates:
      - metadata:
          name: mysql-data
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
    
    ```
    
    - `replicas: 3` → MySQL Pod 3개 실행
    - `serviceName: "mysql"` → Pod들이 이 서비스로 연결됨
    - `volumeClaimTemplates` → 각 Pod마다 개별적인 Persistent Volume Claim 할당
- 스테이트풀셋의 스케일 업과 다운에 따른 PVC 생성과 삭제
    - 스테이트풀셋을 하나 스케일 업하면 2개 이상의 API 오브젝트 (파드, 파드에서 참조하는 하나 이상의 PVC)가 생성됨
    - 하지만 스케일 다운할 때는 파드만 삭제하고 클레임은 남겨둔다.
    - PV를 해제 (release)하려면 PVC를 수동으로 삭제해야 함.
