### 질문 1. 만약 동일한 기본 레이어를 기반으로 하는 두 개의 이미지에서 생성한 두 개의 컨테이너가 있을 때, 그 중 하나가 해당 파일을 덮어쓰면  동일한 파일을 바라보는 다른 컨테이너는 어떻게 되는가? (p. 57, p. 78)

다른 컨테이너는 해당 변경 사항을 볼 수 없다.

컨테이너 이미지 레이어가 읽기 전용이기 때문에 파일을 공유하더라도 여전히 서로 격리되어 있음. 

컨테이너가 실행될 때 이미지 레이어 위에 새로운 쓰기 가능한 레이어가 만들어진다. 컨테이너의 프로세스가 기본 레이어 중 하나에 있는 파일에 쓰면 전체 파일의 복사본의 최상위 레이어에 만들어지고 프로세스는 복사본에 쓴다. 

<img width="861" alt="image" src="https://github.com/user-attachments/assets/b2ba4de5-33dd-4347-a773-734b316d06a1" />


### 질문 2. 컨테이너와 파드의 차이는?

쿠버네티스는 컨테이너 단위로 다루지 않는다. 대신 파드 단위로 다룬다.

파드는 하나 이상의 컨테이너 그룹으로, 같은 워커 노드에서 같은 리눅스 네임스페이스로 함께 실행된다. 각 파드는 자체 IP, 호스트 이름, 프로세스 등이 있는 논리적으로 분리된 머신이다. 따라서 동일 파드에서 실행 중인 모든 컨테이너는 동일한 논리적인 머신에서 실행되는 것 처럼 보이지만, 다른 파드에서 실행 중인 컨테이너들은 같은 워커 노드에서 실행 중이더라도 다른 머신에서 실행 중인 것으로 나타난다.  

### 질문 3. 도커 이미지와 컨테이너의 차이는?

도커 이미지는:

- 응용 프로그램을 실행하는데 필요한 모든 것을 포함하는 읽기 전용 템플릿입니다
- 코드, 런타임 환경, 시스템 도구, 시스템 라이브러리 등을 포함합니다
- 변경이 불가능한(immutable) 상태입니다
- 마치 프로그램의 실행 파일과 같은 개념입니다

도커 컨테이너는:

- 도커 이미지의 실행 가능한 인스턴스입니다
- 이미지를 실행했을 때 메모리에서 동작하는 상태입니다
- 읽기-쓰기가 가능한 레이어가 추가됩니다
- 실행 중인 프로세스와 같은 개념입니다

비유하자면:

- 이미지는 실행 파일(.exe)과 같고, 컨테이너는 그 실행 파일을 실행한 프로세스와 같습니다
- 하나의 이미지로 여러 개의 컨테이너를 실행할 수 있습니다
- 각 컨테이너는 독립적인 환경을 가지며, 서로 영향을 주지 않습니다

이러한 차이점 때문에 도커를 사용할 때:

- 이미지는 빌드(build) 명령으로 생성합니다
- 컨테이너는 실행(run) 명령으로 생성합니다
