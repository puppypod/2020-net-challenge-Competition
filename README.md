# 2020 NET 챌린지 시즌7

## 본 설명은 과학기술정보통신부 주최, KOREN연구협력포럼 및 한국정보화진흥원 주관으로 진행된 NET 챌린지 캠프 시즌7의 연구개발 과제입니다.

대학교 : 숭실대학교 전자정보공학부 IT융합전공

팀명 : K.F.C

지도교수 : 김영한

K.F.C 팀 구성원 : 박재욱, 선훈식, 황태관, 조의진

과제 소개 
- 쿠버네티스 환경에서 멀티 클라우드 인프라 구축을 위한 멀티 사이트 클러스터링과 통합 관제 시스템을 구현한 모델을 제시한다. 이를 엣지 클라우드와 접목시켜 KOREN망에서 서울, 광주에 공장과 엣지를 운영하고, 대전에 코어 클라우드를 운영하는 가상의 기업의 스마트 팩토리 사고 방지 및 대응 서비스를 제시한다.

과제 성과
- 다양한 지역에 위치한 클라우드를 오버레이 네트워크로 연결하여 하나의 클러스터 구축
- 여러 지역에 분산되어 있는 인프라를 한 곳에서 관리 및 모니터링 가능
- 클라우드에 문제가 생겨도 쿠버네티스를 통해 해당 클라우드에서 돌아가는 서비스 컨테이너들이 자동적으로 빠르게 다른 곳으로 이동
- 공장 디바이스에서 보내는 data를 통해 빠르게 판단 및 조치하여 사고를 빠르게 방지 및 조치 가능
- 실제 공장의 많은 디바이스들이 한꺼번에 보내는 대용량 data들을 KOREN을 통해 빠르게 수집 및 저장 가능
- 공장 관리자는 한 웹페이지를 통해서 공장 디바이스들의 데이터, 공장의 상태 등을 어디서든 쉽게 확인 가능

## kubernetes cluster 구축 (IPIP,vxlan)

- 지금부터 마스터 노드와 워커노드, Kubernetes, calico의 vxlan mode로 클러스터를 구축하는 법을 알아보겠습니다.

    - 우선 각 노드를 간단하게 설명하자면, 크게 서울, 대전, 광주에 위치한 오픈스택 VM들을 노드로 사용하였습니다.
        
        - 대전 1: kfc-dj (103.22.222.245) => master node
        - 대전 2: kfc-dj2 ( 103.22.222.246) => worker node 1 ( backend)
        - 대전 3: kfc-dj3 ( 103.22.222.247) => worker node 2 ( backend) 
        - 서울 1: master (116.89.189.54) => worker node 3( frontend) <이름만 master고 역할은 worker…이름을 바꾸기 번거로워서 그대로 사용했습니다. 
        - 서울 2: worker-1 (116.89.189.21)=> worker node 4( frontend)
        - 광주 1: ssu-nc-in-gist-1 (116.89.189.210) => worker node 5 (frontend)
    
    - 서울은 116.89.189.0/24 , 대전은 103.22.222.0/24 , 광주는 116.89.189.0/24 로 서로 다른 subnet 대역을 가지고 있습니다. 이제부터 서로 다른 subnet에 있는 노드들로 쿠버네티스 클러스터를 구축해보겠습니다.
    
        1. **Kubernetes 설치**
    
            - Kubernetes 설치를 위해서는 우선 먼저 docker를 설치해야합니다. 왜냐하면 kubernetes는 컨테이너들을 관리해주는 도구이기에 기본적으로 컨테이너가 돌아갈 수 있는 엔진이 설치되어야 하고 이 역할을 docker가 해주기 때문입니다(정확히 말하면 docker engine). 컨테이너들을 master, worker node 모두에서 돌아가기 때문에 클러스터에 포함시킬 모든 node에는 설치해주도록 합니다.

                ![1](https://user-images.githubusercontent.com/47939832/111863979-eafb2680-89a1-11eb-9c72-e58e6f0bdd4d.png)

                ![2](https://user-images.githubusercontent.com/47939832/111863980-ec2c5380-89a1-11eb-989e-3c2a9ad02ea3.png)

            - 이 명령어가 아니라 sudo apt-get install docker 로 진행해도 상관없습니다. 다만, docker.io만다운받는 것은 이건 제 생각이지만 쿠버네티스에서 컨테이너를 돌리기 위해 docker engine만 필요해서 최소한 필요한 부분만 다운받는 것이지 않을까 싶습니다…
            - 다운 받은 후에는 다음 명령어를 통해 docker version을 확인 및 제대로 설치가 되었는지 확인할 수 있습니다. 



            - 다음 명령어를 수행하면 다음과 같은 출력이 나옵니다.
            - 
                ![3](https://user-images.githubusercontent.com/47939832/111863981-ec2c5380-89a1-11eb-97e3-f4f90f6abfaa.png)

                ![4](https://user-images.githubusercontent.com/47939832/111863983-ecc4ea00-89a1-11eb-8a0e-dc3179e50445.png)

            - 혹은 다른 명령어를 통해 더 자세하게 볼 수 있습니다. 다만 쿠버네티스 클러스터를 구축할 때는 크게 필요가 없을 수 있습니다.
                ![5](https://user-images.githubusercontent.com/47939832/111863984-ecc4ea00-89a1-11eb-8a7e-60435eedca97.png)

                ![6](https://user-images.githubusercontent.com/47939832/111863985-ed5d8080-89a1-11eb-9615-1698adbcc3b5.png)

            - Docker 를 설치했으면 docker를 enable시켜 줍니다

                ![7](https://user-images.githubusercontent.com/47939832/111863986-ed5d8080-89a1-11eb-8659-1865d549d342.png)

                이런식으로 아무런 결과가 출력되지 않아야 정상입니다.

            - 이후에 Kubernetes gpg key를 추가해줍니다. 이떄 curl 명령어를 사용하는데 만약 curl 명령어가 없다면 sudo apt install curl 명령어를 먼저 수행해 다운받으시길 바랍니다. 다음 명령어를 수행하신 후 ‘OK’라는 출력이 나오면 정상적으로 작동이 된 것입니다.

                ![8](https://user-images.githubusercontent.com/47939832/111863987-edf61700-89a1-11eb-9de0-615dfdacc66e.png)

                이후 Xenial Kubernetes repository를 추가해줍니다.

                ![9](https://user-images.githubusercontent.com/47939832/111863988-edf61700-89a1-11eb-9e9a-4bd2ae941599.png)

            - 여기까지 진행이 되었으면 이제 kubeadm을 설치합니다. Kubeadm는 kubernetes에서 클러스터를 구축하기 위한 명령어 도구를 지원해줍니다. 기본적으로 kubeadm, kubespray 등.. 여러 명령어 도구가 있지만 여기서는 가장 기본적인 kubeadm을 사용하도록 하겠습니다. 항상 새로운 패키지를 install 할 때는 습관적으로 sudo apt-get update명령어를 수행해 한 번 update해주는 것이 좋습니다.

                ![91](https://user-images.githubusercontent.com/47939832/111863989-ee8ead80-89a1-11eb-955d-c71f562310d8.png)

                앞에서 docker 설치시와 마찬가지로 다음 명령어를 수행해서 kubeadm 설치를 확인할 수 있습니다.

                ![92](https://user-images.githubusercontent.com/47939832/111863991-ee8ead80-89a1-11eb-82bd-1df260d1fd15.png)
        2. **Kubernetes deploy**

## 쿠버네티스 환경 설정 과정 中 Calico 세팅 방법

