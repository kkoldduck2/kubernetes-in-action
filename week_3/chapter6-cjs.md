## 6장 볼륨 : 컨테이너에 디스크 스토리지 연결



## PV, PVC 라이프사이클과 정책

PV와 PVC 간의 상호 작용은 다음 라이프사이클을 따른다. **프로비저닝** -> 바인딩 -> 사용 중 -> 반환



#### 정적 프로비저닝

**클러스터 관리자가 관련 스토리지 기술을 명세해 PV를 사전에 만드는 방식**이다. PVC의 storage class를 지정하지 않으면 정적으로 만든 PV를 사용할 수 있다. 보통 컴퓨팅 리소스가 한정된 온 프레미스 환경에서 활용한다.



#### 동적 프로비저닝

**PV, PVC를 활용하면 개발자가 내부 스토리지 기술을 몰라도 Persistent Storage를 사용할 수 있다는 장점**이 있다. 그러나 정적 프로비저닝의 경우 관리자가 실제 스토리지를 미리 PV로 다 만들어놓아야 한다는 관리 상 귀찮음이 있다.

인프라스터럭처 관련 처리는 클러스터 관리자만의영역이어야한다는게 쿠버네티스 사상.

K8s에서는 PV의 동적 프로비저닝을 통해 이 작업을 자동으로 수행할 수 있게 한다. 클러스터 관리자는 PV를 생성하는 대신에, PV 프로비저너를 배포하고 사용자가 선택 가능한 하나 이상의 storage class 오브젝트로 정의할 수 있다.

사용자는 PVC에서 PV가 아닌 storage class를 참조하면 프로비저너가 알아서 PV를 프로비저닝 해준다.





#### Retain 정책



PV와 바인딩된 PVC를 삭제하면 위와 같이 STATUS 항목이 Released로 변경되는데, 이 때 PV와 연결되어 있던 디스크 내부의 파일을 어떻게 처리할 것인지에 따라서 **(1)** **Retain, (2) Delete, (3) Recycle** 3가지로 나뉜다. 가장 직관적인 사용 방법은 **Retain**으로, PVC가 삭제되어 Released 상태가 되어도 실제 내부의 파일을 삭제하지 않는 정책이다. 

그렇지만 한 가지 알아둬야 할 점은 Released 상태의 PV는 다른 PVC에 의해 사용될 수 있는 상태는 아니며, 이는 해당 PV에 아직 데이터가 남아있기(Retain) 때문이라고 한다. PV와 연결되어 있던 디스크의 데이터를 계속 사용하고 싶으면 PV를 삭제한 뒤 동일한 PV 설정으로 다시 PV를 생성해줘야 한다. 



#### Delete 정책

**Delete 정책**은 PVC가 삭제되어 PV가 Released 상태가 되면 PV와 연결된 디스크 내부 자체를 삭제하는것이다. 예를 들어 AWS의 EBS에 연결된 PV를 삭제하면 PV에 연결되어 있던 EBS가 함께 삭제되며, 당연하게도 EBS에 저장되어 있던 데이터 또한 함께 삭제된다. 

이러한 클라우드 플랫폼 상에서는 디스크를 생성하고 PV를 직접 연결하는 작업보다는 Dynamic Provisioning을 주로 사용할텐데, **Storage Class의 Reclaim Policy는 기본적으로 Delete**로 설정된다. 그리고 Dynamic Provisioning에 의해 생성된 PV의 정책은 Storage Class의 Reclaim Policy를 상속받기 때문에, 별도의 설정을 하지 않았다면 Dynamic Provisioning의 Reclaim Policy는 자동으로 Delete로 설정된다. 

이 상태에서 PVC를 삭제하면 PV 및 디스크가 함께 삭제된다. 



#### PVC 삭제시 PV가 Pending 상태인 이유는?

* Retain Option 이므로 이미 데이터가있기 때문에 사용 할 수 없는 PV 이기 떄문.

* PVC와 연결된 리소스(Pod 또는 Finalizer)가 삭제되지 않았기 때문이고, Finalizer를 삭제하고 Pod를 삭제하는 방법으로 해결할 수 있다.

https://latte-12.tistory.com/entry/kubernetes-deletePVC 참고



사용자가 Pod에서 활성 사용 중인 PVC를 삭제하는 경우 PVC는 즉시 제거되지 않습니다. PVC 제거는 PVC가 더 이상 모든 Pod에서 활성으로 사용되지 않을 때까지 연기됩니다. 또한 관리자가 PVC에 바인딩된 PV를 삭제하는 경우 PV는 즉시 제거되지 않습니다. PV 제거는 PV가 더 이상 PVC에 바인딩되지 않을 때까지 연기됩니다.

PVC의 상태가 다음 `Terminating`과 같을 때 PVC가 보호되는 것을 확인할 수 있습니다.

---





## SC (StorageClass)

- 정의 : StorageClass는 동적 스토리지 프로비저닝을 관리하기 위한 설정 및 옵션을 제공하는 객체입니다. 스토리지 프로바이더와 함께 동적으로 PV를 프로비저닝하도록 지원합니다.

- 특징

  - StorageClass는 클러스터 관리자에 의해 설정되며 네임스페이스와 무관하게 사용됩니다.
  - 스토리지 클래스는 스토리지 프로바이더와 볼륨 프로비저닝을 위한 설정을 정의합니다. 예를 들어, 볼륨 타입, IOPS, 리플리케이션 수 등을 구성할 수 있습니다.
  - PVC를 생성할 때 사용자는 원하는 스토리지 클래스를 지정할 수 있으며, 해당 클래스에 따라 PV가 동적으로 프로비저닝됩니다. 따라서 먼저 StorageClass를 정의한 후 PVC를 요청하게 되면 SC가 그에 맞는 PV를 생성하여 binding 시켜줍니다.

- 정의 방법

  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: slow
  provisioner: kubernetes.io/no-provisioner
  volumeBindingMode: Immediate
  ```

- volumeBindingMode 종류

  - ***Immediate (즉시 바인딩)*** : Immediate 바인딩 모드에서는 `PV가 즉시 프로비저닝되어 요청된 PVC와 바인딩`됩니다. 이 모드는 PVC를 만들 때 PV가 없는 경우에도 PVC가 즉시 생성됩니다. 그러나 PV 프로비저닝이 지연될 수 있으므로 클러스터의 노드 및 스토리지 상태에 따라 PV가 사용 가능한 상태가 될 때까지 대기해야 할 수 있습니다.
  - ***WaitForFirstConsumer (첫 번째 사용자 대기)*** : WaitForFirstConsumer 바인딩 모드에서는 `PV가 PVC와 바인딩되기 전에 첫 번째 파드 또는 컨테이너가 PVC를 사용하기 위해 요청할 때까지 기다립니다.` 즉, PV는 PVC의 첫 번째 사용자가 요청한 후에만 프로비저닝됩니다. 이 모드는 PV 프로비저닝을 늦추지만, PV가 실제 사용되는지 확인할 수 있는 추가적인 안정성을 제공합니다.



---

Volume 설정시 EmptyDir로 설정하면서 Memory 에 저장 할 수 있음

---





출처 

https://velog.io/@hoonki/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4k8s-Persistent-Storage%EB%9E%80

https://bcho.tistory.com/1259

https://latte-12.tistory.com/entry/kubernetes-deletePVC