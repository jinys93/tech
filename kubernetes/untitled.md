Kubernetes
컨테이너: 어디에서나 실행할 수 있는 소형의 독립 운영체제.
이는 공용 repository 또는 개인 레포에서 호스팅 되는 일련의 명령에 따라 몇 초 만에 생성이 가능

쿠버네티스는 노드에서 여러 컨테이너를 관리하고 예약할 수 있음.

필요한 이유: 실제 프로덕션 에플리케이션은 여러 컨테이너에 걸쳐있으며 이러한 컨테이너는 여러 서버 호스트에 배포되어야 한다.
모델을 생성할 때, 한 개 모델을 한 서버에서 학습 시키는 것은 비교적 쉽다.
그러나, 수 많은 모델들을 여러 서버에 걸쳐서 코드를 배포하고 모델을 학습하는 작업은 그리 간단하지 않다.
- 물론, 깃으로 소스코드 관리하고 코드를 배포하고 원격으로 학습을 실행하면 쉽게 해결할 수도 있다.
혹은, 자체적인 모델 학습 서버 cluster를 구축하여 server-side 엔지니어링에 대해서는 더 이상 고민할 필요 없이 모델링에만 집중할 수 있다.
이런 경우라면 체계적인 모델링 플랫폼을 가지고 있다고 할 수 있다. 자체 학습 cluster를 구축하기는 어렵지만 체계적으로 모델 학습 서버들을 관리하고 운용하기 원한다면 쿠버네티스트는 괜찮은 선택이다.

 여러 서버를 하나로 묶어서 공통적으로 관리하고 컨테이너를 분산하여 실행해주는 cluster management system에 더 가깝다. 
모델 학습에 필요한 모든 소스코드 및 하이퍼 파라미터들은 마스터를 통해서만 정보를 주고 받고 나머지 서버들은 단순히 컨테이너화된 모델 학습 Job을 수행하는 executor로 활용할 수 있다.
이제는 더 이상 각 서버를 순회하면서 서버의 자원이 가용한지 확인하고 직접 모델 학습 스크립트를 실행 시킬 필요 없이 학습에 사용할 서버를 전부 쿠버네티스 클러스터에 연결하고 필요한 작업들을 마스터에 맡기면 마스터가 알아서 서버를 보살핀다.

모든 학습 서버들은 쿠버네티스 마스터에 연결되어 있고, 보통 여러대의 서버를 묶어서 마스터를 만든다.
각 서버는 마스터와 통신하며 필요한 정보를 전달(node 상태 정보 등)하고 마스터로 부터 정보를 받는다. (모델학습 실행 정보 등) 또한 각 학습 서버는 NFS로 연결이 되어 각 서버에서 실행한 결과를 한곳에 모이도록 구성되어 있다. 이를 통해 학습이 끝난 이후에 각 서버에 직접 접속하여 학습 결과 (모델파일, 성능 지표 등)를 확인하는 것이 아니라 NAS 서버로 접속하여 한번에 확인할 수 있다.

k8s를 이용한 모델 학습 방식의 장점
1. 매번 각 서버에 접속하여 학습 스크립트를 실행하고 모델링하는 작업이 사라진다.
- 기존의 방법은 직접 학습 서버에 들어가서 실행 명령을 내려줘야 한다. 반면, 쿠버네티스를 이용하면 모든 명령을 쿠버네티스 마스터로만 요쳥하면 되고 마스터에서 학습 프로세스를 관리한다. 또한, 학습이 어디까지 진행이 되었는지, 정상적으로 학습하고 잇는지 학습 로그를 통해서 확인해야 하는데 쿠버네티스를 이용한다면 마스터를 통해 한 곳에서 각 학습 진행상황을 모니터링할 수 있다.

2. 각 서버별 모델 실핼 환경(라이브러리 버전 등)을 동일하게 맞추는 작업이 사라진다.
- 쿠버네티스는 기본적으로 모든 프로세스를 컨테이너로 실행시키기 때문에 업데이트할 이미지만 바꾸면 모든 서버에서 동일하게 수정할 수 있다.(컨테이너를 사용하는 모든 시스템의 장점)

3. 모델 실험이 편리해진다.
- 새로운 모델을 실험할 때, 어느 서버에서 어떤 모델을 돌리는 지 확인하는 일은 중요하다. 서버 개수가 많아지면 모델 실험 파라미터들을 디비로 저장하는 로직을 개발하여 관리할 수도 있다. 쿠버네티스를 이용하면 조금 더 편리하게 모델 실험을 할 수 있다. hyper parameter set들을 여러 서버에 분산하여 관리할 필요 없이 마스터 서버에서만 들고 있으면 된다. 실행한 모델 학습 정보는 자동으로 쿠버네티스의 저장소에 메타정보로 남게 된다. 추가적인 노력없이 모델 실행 정보를 관리하게 된다.

4. 서버 자원 관리를 직접할 필요가 없다.
새로운 학습을 실행 시키기 위해서 가장 먼저, 놀고 있는 서버를 찾아야 한다. 그 다음, 찾은 서버중에 학습 시키고자 하는 모델의 성격과 가장 적합한 서버를 골라야 한다. 만약 현재 가용한 서버가 없다면 가장 빨리 끝날 것 같은 서버를 찾아 스케쥴을 등록해야 한다. 쿠버네티스를 이용하면 이 모든 것을 쿠버네티스가 관리해준다. 쿠버네티스에는 자체 스케쥴러가 있어, 컨테이너가 요쳥한 자원의 크기를 보고 적절하게 가용한 서버에 할달을 한다. 또한, 쿠버네티스의 매커니즘 중에 하나인 label을 이용하여 사용자가 지정한 서버들 중에서 고르게 할 수도 있다. 

5. 모델 학습 결과를 한곳에서 확인할 수 있다.
여러 서버에서 학습을 수행 하였을 때 또 다른 불편한 점을 들면, 학습 결과물을 한곳에서 확인하기 힘들다는 점이다. 쿠버네티스를 이용하면 손쉽게 모델 파일을 원격 중앙 저장소에 모을 수 있다. 
* 원격 저장소를 연결하는 기술은 쿠버네티스가 제공하는 기능으 ㄴ아니다.
단순히 NFS 프로토콜을 이용하여 원격 저장소에 마운트하는 것이 전부다. 하지만, 쿠버네티스가 제공하는 Persistent Volume Claim(pvc)이라는 추상화된 쿠버네티스 volume 인터페이스를 사용하게 되면 구체적인 원격 저장소 정보를 알 필요없이 손쉽게 컨테이너에 volume을 여녁ㄹ할 수 있다. 또한, pvc를 이용하면 사용자가 직접 NAS 서버에 저장소를 마운트할 필요없이 쿠버네티스가 필요할 때마다 연결해 준다. 마지막으로, pvc는 개별적으로 이름을 가지는데 사용자는 해다 ㅇ이름을 reference하여 전혀 다른 컨테이너에서 파일을 접근할 수도 있다.

6. 특정 서버에서 장애가 발생해도 크게 문제가 되지 않는다.
7. 한개 프로세스의 문제로 서버 전체에 장애가 발생하는것을 방지해 준다.
여러 모델을 한 서버에서 학습하는 경우, 문제가 되는 한개의 학습 프로세스로 인해 서버 전체가 영향을 받는 경우가 있다. 예를들면, 한개의 프로세스가 모든 GPU 자원을 점유한다던지 서버의 모든 메모리를 혼자서 차지하는 경우다. 이러한 문제는 엄격한 코드 리뷰를 통해 코드 레벨에서 문제가 되는 부분을 걸러낼 수 있지만 매번 엄격한 코드 리뷰를 하는 것은 현실적으로 힘들고 리뷰를 한 경우에도 미처 찾아내지 못한 코드가 문제를 일으킬 수 있다. 각 컨테이너마다 리소스 제한을 할 수 있기 때문에 한개 프로세스의 문제로 서버 전체가 망가지는 것을 방지해 준다. 이를 통해 조금 더 안정적으로 모델 핛급 운영을 할 수 있다.